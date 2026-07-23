# 从 RPC 到工作队列：Hermes Kanban Swarm 如何让 AI Agent 协作迈向生产级

> *From Function Calls to Work Queues: How Hermes Kanban Swarm Makes Multi-Agent Collaboration Production-Ready*

---

## 引言

你肯定经历过这种时刻：凌晨一点，你终于调好了 3 个并行 agent 的 prompt，一个负责调研竞品架构，一个梳理安全边界，一个写故障预案。敲下回车，合上笔记本，心想明早起来收成果。然后你醒了——笔记本休眠了，terminal 里三个进程全挂，context 窗口里的中间产物灰飞烟灭，连个 stack trace 都没留下。

这不是你的问题。这是 `delegate_task` 的问题。

`delegate_task` 的本质是一个**同步 RPC fork-join**：父进程 fork 出子进程，阻塞等待所有子进程返回，然后把结果拼起来塞回你的上下文。它无状态，无崩溃恢复，你合上笔记本的那一刻，整个编排树就断了。这在 demo 里很优雅，但在生产环境里是灾难。

Hermes Agent v0.16 给出了一个不同的答案：**Kanban Swarm**。它用持久化工作队列替代同步 RPC，用共享黑板替代上下文拼接，用七态状态机替代一次性 fork。你的 agent 不再是函数调用栈上的一次性帧，而是 SQLite 里一条可以断点续传的任务记录。

本文将从架构设计、源码结构和生产级工作流三个层面，拆解 Kanban Swarm 如何让 Hermes 从单次会话工具升级为持久化 AI 平台基础设施。

---

## 范式跃迁：`delegate_task` 是函数调用，Kanban 是工作队列

理解 Kanban Swarm 的关键，在于认识到它与 `delegate_task` 不是"升级"关系，而是**不同抽象层的工具**。

### delegate_task：同步 RPC 的优雅与脆弱

`delegate_task` 的设计哲学很纯粹：把 agent 当成一个可并行的函数。你调用它，传入 prompt 和 profile，它返回结果。如果同时调用 3 个，底层 fork 出 3 个 OS 进程，父进程阻塞等待所有子进程 exit，然后把 stdout 拼成结构化输出。

这个模型的问题出在"同步"和"无状态"两个约束上：

- **同步阻塞**：父进程必须存活到所有子进程返回。笔记本休眠、网络闪断、OOM——任何一个都能让整个编排树灰飞烟灭。
- **无状态**：子进程之间没有通信协议。Worker A 调研出的结论，Worker B 完全不知道——除非你把所有结果塞进父进程的 context，然后手动分发。
- **无审计**：进程退出后，中间推理链全部丢失。你想知道 agent 为什么得出那个结论？抱歉，stdout 已经被回收了。

### Kanban Swarm：持久化消息队列 + 状态机

Kanban Swarm 换了一个范式：**任务不是函数调用，是数据库里的一条记录**。

当你通过 `hermes kanban swarm` 启动一个 Swarm 时，编排器做的事不是 fork，而是往 SQLite 里写任务卡片。每个卡片带一个状态字段，从 `triage` → `todo` → `ready` → `running` → `done`，形成一个完整的生命周期。Dispatcher（运行在 gateway 内的调度循环）每 60 秒扫描一次看板，回收过期声明，提升就绪任务，原子性地声明并 spawn worker。

这带来了几个根本性的变化：

| 维度 | delegate_task | Kanban Swarm |
|------|--------------|-------------|
| 存活周期 | 父进程生命周期 | SQLite 持久化，跨会话存活 |
| 崩溃恢复 | 无，全部重来 | Dispatcher 检测超时声明，自动重新指派 |
| 人类介入 | 不可打断 | mid-turn `/kanban` 斜杠命令直接操作看板 |
| 审计追踪 | 无 | 每个 worker 的 claim/complete/block 事件全量记录 |
| 跨 agent 通信 | 无（需手动拼接） | `[swarm:blackboard]` 共享黑板，JSON comments 协议 |
| 隔离级别 | 共享父进程 | 每个 worker 是独立 OS 进程，独立 profile，独立记忆 |

关键洞察：**Kanban 并不替代 delegate_task**。实际上，每个 Kanban worker 内部仍然可以调用 `delegate_task` 做子任务分解。这是一个分层架构——Kanban 管"工作的持久化和调度"，delegate_task 管"单次执行的计算并发"。两者是共存关系，不是竞争关系。

> **"`delegate_task` is a function call; Kanban is a work queue."**

---

## 拓扑解剖：Swarm 的 DAG 结构

Kanban Swarm 的源码位于 `hermes_cli/kanban_swarm.py`，约 150 行。它的角色是一个**薄层编排器**——不引入新的调度服务，不依赖外部消息队列，所有逻辑都落在 SQLite 和 Hermes 已有的 gateway Dispatcher 上。

### DAG 拓扑

一个典型的 Swarm 拓扑如下：

```
planning root (立即完成)
  ├─ worker₁ ∥ worker₂ ∥ worker₃  (并行执行)
  └─ verifier (屏障，等待所有 worker)
       └─ synthesizer (等待 verifier)
```

这个 DAG 的语义很清晰：
- **Planning root**：由启动 Swarm 的 agent 直接完成，将目标分解为 N 个子任务卡片，写入看板。
- **Worker 层**：N 个 worker 并行执行。它们通过 `[swarm:blackboard] ` 前缀的 JSON comments 共享中间产物，互不阻塞。
- **Verifier**：屏障节点——等待**所有** worker 标记为 `done` 后才从 `blocked` 转为 `ready`。它审阅所有 worker 输出，标记通过或驳回。
- **Synthesizer**：等待 verifier 完成，将所有结果合并为最终交付物。

这种拓扑设计的精妙之处在于：**DAG 的结构不是硬编码在代码里的，而是通过看板卡片之间的依赖关系动态构建的**。Verifier 不"知道"worker 的存在——它只是在 `depends_on` 字段里声明了依赖，Dispatcher 看到所有前置任务 complete 后自动将其提升为 ready。

### SwarmWorkerSpec 数据结构

每个 worker 的规格由 `SwarmWorkerSpec` 定义：

```python
# 概念模型（非直接源码）
SwarmWorkerSpec:
  profile: str          # worker 使用的 Hermes profile（决定模型/provider）
  title: str            # 看板卡片标题
  body: str             # prompt / 任务描述
  skills: list[str]     # 加载的 skills 列表
  priority: int         # 优先级（数字越大越优先调度）
  max_runtime_seconds: int  # 最大运行时间，超时自动回收
```

`create_swarm()` 核心函数签名：

```python
def create_swarm(
    conn,                    # SQLite 连接
    goal: str,               # Swarm 目标
    workers: list[SwarmWorkerSpec],
    verifier_assignee: str,  # verifier 的 assignee 标识
    synthesizer_assignee: str,
    root_title: str,
    verifier_title: str,
    synthesizer_title: str,
    tenant: str,
    created_by: str,
    workspace_kind: str,     # 'scratch' | 'dir:<path>' | 'worktree'
    workspace_path: str | None,
    priority: int,
    idempotency_key: str | None,  # 幂等键，防止重复创建
) -> SwarmCreated:
    ...
```

返回值 `SwarmCreated` 包含：`root_id`、`worker_ids`（列表）、`verifier_id`、`synthesizer_id`。

### 共享黑板协议

Worker 之间的通信不走 context 拼接，而是通过看板卡片上的 comments。约定前缀 `[swarm:blackboard] ` 标记的 JSON comment 作为共享黑板：

```json
{
  "key": "security_findings",
  "value": "OAuth2 token refresh 存在 race condition，建议引入分布式锁",
  "confidence": 0.85
}
```

这是一个松耦合协议——任何 worker 可以读黑板上的任何 key，也可以写自己的 findings。Verifier 和 Synthesizer 直接消费黑板数据，不依赖 worker 的内部实现细节。

### CLI 示例

```bash
hermes kanban swarm "Design a multi-region failover plan for PostgreSQL" \
  --workers researcher,architect,sre \
  --verifier reviewer \
  --synthesizer writer
```

一行命令，生成 6 张看板卡片（1 root + 3 worker + 1 verifier + 1 synthesizer），写入 `~/.hermes/kanban.db`，然后 Dispatcher 开始按 DAG 拓扑逐层调度。

---

## Kanban 内核：SQLite 状态机 + Dispatcher 调度循环

如果说 Swarm 是 Kanban 的"宏编排"，那看板本身的内核就是它的"微引擎"。

### 持久化层

每个 Board 是一个独立的 SQLite 数据库，默认路径 `~/.hermes/kanban.db`。**多 Board 之间硬隔离**——每个项目一个独立 DB，不会出现任务泄露或命名冲突。Board = 隔离边界。

### 七态状态机

看板卡片的状态机是最核心的设计：

```
triage → todo → ready → running → blocked → done → archived
```

- **triage**：刚创建，尚未分配
- **todo**：已分配，等待调度
- **ready**：所有依赖满足，可被执行
- **running**：worker 已声明并正在执行
- **blocked**：被依赖阻塞，或达到失败上限
- **done**：完成
- **archived**：归档（软删除）

状态转换不是任意方向的。`running` 只能转到 `done` 或 `blocked`；`blocked` 可以转回 `ready`（依赖解除）或被人工 `archive`。`kanban.failure_limit` 默认 2 次——一个 worker 连续失败超过 2 次，自动转到 `blocked`，需要人工介入。

### Dispatcher 四步调度循环

Dispatcher 运行在 Hermes gateway 进程内部，以 60 秒为周期执行调度：

1. **回收过期声明**：扫描所有 `running` 状态且心跳超过阈值的卡片，将其状态回退到 `ready`，释放声明锁。
2. **恢复崩溃 worker**：对于状态回退的卡片，其对应的 OS 进程可能已死，Dispatcher 保证该任务可以被其他可用 worker 重新声明。
3. **提升 ready 任务**：检查 `blocked` 卡片的所有依赖是否都已完成，若是则提升为 `ready`。
4. **原子声明 + spawn worker**：对 `ready` 卡片执行原子性 claim（CAS 更新），成功则 spawn 新的 OS 子进程作为 worker。

关键设计决策：**Worker 是完整 OS 进程，不是协程**。这意味着：
- 每个 worker 有独立的 profile，可以走不同的 provider/model
- 每个 worker 有独立的记忆（持久化到自己的 session）
- 一个 worker OOM 不会影响其他 worker
- 进程级别的隔离，不是 async/await 的并发糖

### Worker 工具集与环境变量

每个 worker 进程启动时，注入以下环境变量：

```bash
HERMES_KANBAN_TASK        # 当前任务 ID
HERMES_KANBAN_DB          # 看板数据库路径
HERMES_KANBAN_BOARD       # Board 名称
HERMES_KANBAN_WORKSPACE   # 工作空间路径
HERMES_KANBAN_RUN_ID      # 本次运行 ID
HERMES_KANBAN_CLAIM_LOCK  # 声明锁 token
HERMES_PROFILE            # Worker 使用的 profile
HERMES_TENANT              # 租户标识
```

Worker 可用 7 个 kanban 工具：

- `kanban_show` — 查看当前任务详情和黑板内容
- `kanban_complete` — 标记任务完成，写入产出
- `kanban_block` — 标记任务阻塞（附带原因）
- `kanban_heartbeat` — 发送心跳，防止被 Dispatcher 回收
- `kanban_comment` — 写 comment（包括 `[swarm:blackboard]` 协议）
- `kanban_create` — 创建子任务卡片（worker 内进一步分解）
- `kanban_link` — 建立卡片之间的依赖关系

### 三种工作空间

- **scratch**：临时目录，Swarm 结束后自动清理
- **dir:\<path\>**：指定本地目录，多个 worker 共享文件系统
- **worktree**：基于 git worktree，每个 worker 在独立分支上工作，适合代码生成类任务

---

## 实战场景：五个生产级工作流

### 1. Research Triage（调研分流）

```
输入："调研 React Server Components 对 Next.js 项目迁移的影响"
  ├─ Researcher A: RSC 性能基准测试
  ├─ Researcher B: 现有项目兼容性分析
  ├─ Researcher C: 社区迁移案例收集
  └─ Verifier → Synthesizer: 综合调研报告
```

三个 researcher 并行跑，各自写黑板，verifier 交叉验证，synthesizer 生成最终报告。博客管线正是这种模式——这也是你现在读到这篇文章的生产方式。

### 2. 定时运维（Recurring Briefs）

```bash
hermes cron "daily ai news brief" --schedule "0 8 * * *" \
  --swarm "briefing-workers" --board "daily-briefs"
```

每天早上 8 点自动触发 Swarm，结果持久化到 `daily-briefs` 看板。积累一周就是一份完整的行业简报档案。

### 3. Digital Twins（持久化命名助手）

创建一个 profile 为 `code-review-buddy` 的持久化 worker，挂载在专用 Board 上。每当你 push 代码，它从看板上拿到最新 diff，积累跨会话的记忆——它记得你上次的代码风格偏好、常见的 lint 错误模式、甚至你偏爱用 `Result<T, E>` 而不是 `try-catch`。

### 4. 工程管线（Decompose → Parallel Worktrees → Review → PR）

```
planning: 将 feature spec 分解为 module-a, module-b, module-c
  ├─ worker-a (worktree: feature/module-a)
  ├─ worker-b (worktree: feature/module-b)
  └─ worker-c (worktree: feature/module-c)
       └─ verifier: 交叉 review
            └─ synthesizer: 合并 → 打开 PR
```

每个 worker 在独立 git worktree 上工作，互不冲突。Verifier 审阅所有 diff，通过后 synthesizer 合并并打开 PR。

### 5. Fleet Work（一个专家管 N 个对象）

一个 SRE 专家 worker，持久化 Board 上挂着几十张卡片——每张代表一个微服务的健康检查。Dispatcher 循环调度，专家逐张处理，把异常服务的诊断结果写到黑板上。不需要为每个服务起一个 agent，一个专家 + 一个队列 = 一个运维舰队。

---

## 结论

Kanban Swarm 解决的核心问题不是"如何让 agent 协作"，而是**"如何让 agent 协作可持久化、可恢复、可审计"**。

在 v0.16 之前，Hermes 是一个强大的单次会话工具——你打开它，完成任务，关闭它。Kanban Swarm 改变了这个定义：你的 agent 工作流现在有自己的数据库、状态机和生命周期。任务不是函数栈上的一次性帧，而是 SQLite 里一条可以跨会话、跨进程、跨崩溃持续存在的记录。Hermes 从 CLI copilot 变成了**持久化 AI 平台基础设施**。

v0.16（代号 "The Surface Release"，2026.6.5）的规模本身就是社区对这个方向投票的证据：**874 commits，542 merged PRs，1,962 files changed，170 位贡献者**。399 个 issue 关闭，其中 2 个 P0 和 62 个 P1。这不是一个小团队的实验——这是一个平台级功能。

Kanban Swarm 只是 v0.16 的亮点之一。同期的 Electron 桌面应用、Quick Setup via Nous Portal、`/undo` 命令、Simplified Chinese 翻译、NVIDIA/skills 加入 Skills Hub——这些都在指向同一个方向：Hermes 正在从开发者的玩具变成开发者的基础设施。

**现在就试**：

```bash
# 安装/升级 Hermes
pip install hermes-agent --upgrade

# 创建你的第一个 Board
hermes kanban board create "my-first-swarm"

# 跑一个 Swarm
hermes kanban swarm "Research the state of AI agent frameworks in 2026" \
  --workers researcher,analyst \
  --verifier reviewer \
  --synthesizer writer
```

让 agent 不再是你笔记本合上就消失的一次性进程，而是一条可以在 SQLite 里存活、恢复、演化的持久化工作流。

> **"The future of AI engineering isn't bigger models — it's better coordination."**
