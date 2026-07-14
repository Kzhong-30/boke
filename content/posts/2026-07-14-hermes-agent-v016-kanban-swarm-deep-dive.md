---
title: "Hermes Agent v0.16 Kanban Swarm：多智能体协作的持久化任务编排引擎"
date: 2026-07-14
tags: ["Hermes Agent", "Kanban", "Multi-Agent", "AI Agent", "Nous Research", "Task Orchestration"]
author: "Kzhong"
description: "深度解析 Hermes Agent v0.16 Kanban Swarm —— 一个基于 SQLite 的持久化任务看板系统，让多个具名 AI Agent 配置协作完成复杂工作流。"
---

## 引言

2025–2026 年，AI Agent 领域的焦点从"单个 Agent 能做什么"快速转向"多个 Agent 如何协作"。市面上涌现了琳琅满目的多智能体框架——CrewAI、AutoGen、LangGraph——但大多数解决的是同一个问题："如何在一次会话中让多个模型对话？"

Hermes Agent v0.16（Surface Release）给出的回答完全不同。它引入的 **Kanban Swarm** 不是一个"智能体聊天室"，而是一个**持久化的、基于 SQLite 的分布式任务看板**，允许多个具名 Agent 配置（profiles）以异步、可靠的方式协作完成复杂工作流。

本文深入 Hermes Kanban 的架构设计、核心概念、实战模式，以及它在多智能体编排领域所做的独特取舍。

---

## 第一性原理：为什么 Kanban 而不是 delegate_task？

在理解 Kanban 之前，必须先理解它的对立面——`delegate_task`。

`delegate_task` 是 Hermes 内建的子智能体机制：父 Agent fork 一个子 Agent，把任务传给它，阻塞等待结果，拿到摘要后继续。这是一个**同步 RPC 调用**——快、简单，适合"查一下这个错误日志"或"帮我搜一段代码"这类短推理任务。

但当你试图用它编排复杂工作流时，问题就暴露了：

| 维度 | delegate_task | Kanban |
|------|--------------|--------|
| **持久性** | 子任务随父上下文压缩而丢失 | 重启后恢复，SQLite 持久化 |
| **身份** | 匿名子 Agent，无记忆 | 具名 profile，持久的记忆和技能 |
| **人在回路** | 不支持——父进程阻塞等待 | 任何节点可查看、评论、解阻塞 |
| **重试** | 失败即失败 | 崩溃后自动回收，断路器限次重试 |
| **审计** | 上下文压缩后丢失 | SQLite 行记录永久保留 |
| **协调模式** | 层级式（调用方→被调用方） | 对等式——任意 profile 读写任意任务 |

**一句话区分：`delegate_task` 是一个函数调用；Kanban 是一个工作队列，每一次交接都是一行任何 profile（或人）都能看到和编辑的记录。**

两者可以共存——一个 Kanban worker 在运行过程中完全可以内部调用 `delegate_task` 来完成子步骤。但跨 Agent 边界、需要持久性、需要人工介入的工作，Kanban 是不二之选。

---

## 核心架构：三个原语撑起一切

Hermes Kanban 的底层极其简洁，只有三个组件：

### 1. SQLite 数据库

所有任务存储在 `~/.hermes/kanban.db`（默认看板）或 `~/.hermes/kanban/boards/<slug>/kanban.db`（多看板）。每行记录一个任务的状态、赋值、父任务链接、运行时、心跳时间戳和完成摘要。

选择 SQLite 而非 PostgreSQL 或消息队列，是深思熟虑的决策：
- **零依赖**：Hermes 自身就能管理，无需外部基础设施
- **事务安全**：WAL 模式确保并发安全，dispatcher 的 compare-and-swap 不会出现竞态
- **便携**：看板是单个文件，可以复制、备份、传递给其他开发者

### 2. Dispatcher 循环

嵌入在 Hermes Gateway 进程中的一个循环，默认每 **60 秒** tick 一次（可通过 Dashboard 的 "Nudge dispatcher" 按钮立刻触发）。每个 tick 执行六个阶段：

1. **Reclaim stale claims** —— 检测崩溃的 worker（通过 `kill(pid, 0)` 探活），将任务重置为 `ready` 状态
2. **Close timed-out runs** —— 超过 `max_runtime_seconds` 的任务被标记为超时
3. **Auto-decompose triage** —— 如果启用，将 triage 状态的任务交给 orchestrator profile 分解
4. **Promote dependencies** —— 所有父任务已达 `done` 的子任务从 `todo` 提升到 `ready`
5. **Spawn new workers** —— 对每个 `ready` 状态的任务，spawn 对应的 profile 作为独立 OS 进程
6. **Circuit breaker** —— 连续失败超过 `failure_limit` 的任务自动阻塞，停止无意义的重试

### 3. 工具表面（Tool Surface）

Worker Agent 通过一套 `kanban_*` 工具与看板交互，无需 shell 到 CLI：

| 工具 | 用途 |
|------|------|
| `kanban_show` | 读取任务详情、父任务摘要、前置尝试、评论 |
| `kanban_list` | 按状态/赋值人列出任务 |
| `kanban_complete` | 完成任务，写入结构化的摘要和元数据 |
| `kanban_block` | 阻塞任务（需要人工介入时） |
| `kanban_heartbeat` | 发送心跳信号（防止被 dispatcher 回收） |
| `kanban_comment` | 添加评论 |
| `kanban_create` | 创建子任务 |
| `kanban_link` | 建立任务间依赖关系 |

关键设计细节：`kanban_show` 默认从环境变量 `$HERMES_KANBAN_TASK` 读取任务 ID，worker 甚至不需要知道自己的 ID。

---

## 七状态状态机

Kanban 的每个任务在生命周期中经历七个状态：

```
[*] → triage → todo → ready → running → done → archived
                              ↓
                           blocked → ready (unblock后)
```

- **Triage**：粗略想法，尚未分解。可由 orchestrator profile 自动分解为具体子任务
- **Todo**：已定义但等待依赖条件满足。所有父任务 `done` 后自动提升到 ready
- **Ready**：可被 dispatcher 分派给对应 profile 执行
- **Running**：worker 已认领并正在执行
- **Blocked**：worker 遇到障碍需要人工介入，或断路器触发
- **Done**：完成——通过 `kanban_complete` 写入总结和元数据
- **Archived**：归档隐藏

状态迁移大部分是**自动化**的：依赖引擎自动推进 todo→ready，dispatcher 自动分配 ready→running，崩溃自动 running→ready。人只需要在 blocked→ready 这一步做决策，以及在 triage→todo 阶段分解任务。

---

## 依赖 DAG：让管道自动推进

Kanban 最强大的特性之一是**依赖有向无环图（DAG）**。通过 `--parent` 标志，你可以显式声明任务之间的前驱关系：

```bash
# 设计 schema（无父任务，立即进入 ready）
SCHEMA=$(hermes kanban create "Design auth schema" \
  --assignee backend-dev --priority 2)

# 实现 API（等 schema 完成）
API=$(hermes kanban create "Implement auth API" \
  --assignee backend-dev --parent $SCHEMA)

# 写测试（等 API 完成）
hermes kanban create "Write auth integration tests" \
  --assignee qa-dev --parent $API
```

当 Schema 完成时，依赖引擎自动将 API 提升到 `ready`；API 完成时，Tests 提升。**无需任何手动协调代码**。

这同样适用于 fan-out 模式：多个并行的 research 任务各自完成后，自动触发 synthesis 任务。或者 PM 写 spec 与工程师实现同时进行，reviewer 等两者都完成才启动。

---

## 结构化交接：summary + metadata 是关键

一个常被低估但至关重要的设计：`kanban_complete` 接受两个结构化参数。

- **summary**：自由文本的完成总结
- **metadata**：JSON 对象，可包含 `changed_files`、`decisions`、`test_results` 等任意键值对

当下游 worker 调用 `kanban_show()` 读取任务时，返回的 `worker_context` 会自动包含所有已完成父任务的 summary 和 metadata。这意味着：

- 工程师写 Schema 时记录的字段设计决策，API worker 自动可见
- PM 写在 spec 里的验收标准，reviewer worker 结构性读取
- QA 记录的测试覆盖率和通过率，发布 worker 直接引用

这替代了"翻评论找上下文"的苦力活——每一次交接都自带完整的决策上下文。

---

## 四看板实战模式

官方教程展示了四个经典使用场景，每个解决一类常见的工作流需求：

### Story 1：独开发者推送功能

经典的单人流水线：Schema → API → Tests。三个任务排成依赖链，dispatcher 按序分派给 `backend-dev` profile。structured handoff 确保下游 worker 不需要重新翻阅上游的设计文档。

### Story 2：农场式并行执行

你需要同时对 20 个文件执行相同的操作——比如批量翻译或批量代码审查。不创建 20 个依赖链，而是创建 20 个同层级子任务，每个挂同一个父任务。dispatcher 在配置允许的并发范围内同时 spawn 多个 worker 并行执行。

### Story 3：角色管道 + 自动重试

Research → Writer → Reviewer → Publisher，每个步骤由不同 profile 执行。如果 Reviewer 拒绝了 Writer 的输出，dispatcher 不重新分配——Writer 收到包含 rejection 评论的任务卡片，修改后再次提交。重试次数由 `failure_limit` 控制，超过后 circuit breaker 自动阻塞。

### Story 4：断路器

当某个任务连续失败达到阈值，dispatcher 不再盲目重试，而是将任务自动设为 `blocked` 状态，等待人工介入排查根因。审计日志完整记录了每次尝试的开始时间、结束时间、退出码和产出摘要。

---

## v0.16 的新增特性

v0.16 Surface Release 对 Kanban 系统做了实质性的迭代优化：

### 多看板支持

你可以创建多个隔离的看板，每个对应不同的项目或领域：

```bash
hermes kanban boards create blog --name "Blog Pipeline" --icon 📝
hermes kanban --board blog create "Write post on Kanban" --assignee writer
```

看板级隔离是绝对的——worker 物理上无法看到其他看板上的任务。Dashboard 在看板切换时重新建立 WebSocket 连接，确保数据不会串流。

### 看板级 Dashboard UI

`hermes dashboard` 提供了完整的可视化看板界面：六列状态泳道、profile 视角切换（每个 profile 看到自己的任务流）、拖拽操作、实时 WebSocket 事件流。Triage 卡片上的 **✨ Specify** 和 **⚗ Decompose** 按钮让任务分解变成一键操作。

### 附件系统

任务可以携带文件附件（PDF、图片、源码文档），worker 启动时自动可见。附件存储在 `<hermes-home>/kanban/attachments/<task_id>/` 目录，worker 通过 `read_file` 工具直接读取。

### 事件日志系统

`hermes kanban tail <id>` 显示单个任务的完整事件链。`hermes kanban watch` 实时流式推送所有任务的状态变更事件。事件类型包括：

- `spawned` — worker 进程已启动
- `heartbeat` — worker 报告存活状态
- `reclaimed` — worker 崩溃后任务被回收
- `crashed` — worker 非正常退出
- `timed_out` — 任务超过最大执行时间
- `gave_up` — 断路器触发
- `protocol_violation` — worker 退出但未完成任务

---

## 与生态中共存

### delegate_task

**用 delegate_task 当**：父 Agent 需要快速推理答案再继续，无需人工介入，结果回到父上下文。

**用 Kanban 当**：工作跨 Agent 边界、需要持久化、可能需人工介入、可能由不同角色处理、或需要事后可追溯。

### Cron Jobs

Cron Job 可以触发 Kanban 任务创建——比如每天早 8 点创建一个 "生成日汇报" 的任务给 writer profile。

### Skills

Kanban worker 的 profile 可以加载不同的 skills 集，使同一个人格在不同任务下展现不同的专业技能。比如 `backend-dev` profile 加载 `engineering-senior-developer` skill，`researcher` profile 加载 `web_search` 和 `web_extract` 工具集。

---

## 局限性

### 单主机限制

Kanban 明确设计为单主机部署。`~/.hermes/kanban.db` 是本机 SQLite 文件，dispatcher 只能在本地 spawn worker。跨主机的共享看板目前不支持——没有原语能保证"主机 A 上的 worker 与主机 B 上的 worker"的协调一致性。如果你的场景需要多主机，每个主机运行独立的看板，用 `delegate_task` 或消息队列桥接。

### 无嵌套限制

Worker 可以调用 `delegate_task` 甚至创建 Kanban 子任务——但嵌套深度需要你自行控制。Hermes 的默认配置限制 delegation 的嵌套深度为 1（即 worker 不能再 delegate），通过修改 `delegation.max_spawn_depth` 配置可以放开，但需要谨慎。

---

## 配置参考

```yaml
# ~/.hermes/config.yaml 中与 Kanban 相关的配置

kanban:
  orchestrator_profile: "orchestrator"     # 负责分解 triage 任务的 profile
  default_assignee: "researcher"           # 分解未指定赋值者时的默认值
  auto_decompose: true                     # 是否自动分解 triage 任务
  auto_decompose_per_tick: 3              # 每次 tick 最多分解的任务数
  dispatch_stale_timeout_seconds: 120      # 心跳超时阈值
  failure_limit: 3                         # 断路器阈值
  dispatch_in_gateway: true                # 是否在 gateway 中嵌入 dispatcher

auxiliary:
  kanban_decomposer:                       # 分解器专用 LLM 槽
    provider: openrouter
    model: anthropic/claude-sonnet-4
  profile_describer:                       # 自动生成 profile 描述的 LLM
    provider: openrouter
    model: anthropic/claude-haiku-3.5
```

---

## 总结

Hermes Agent v0.16 的 Kanban Swarm 代表了一种务实的多智能体协作哲学：**不用"智能体聊天室"，而用持久化的任务队列**。每个 worker 是匿名的、有自己记忆和技能、运行在独立 OS 进程中的完整 Agent。每一次交接都是一条永久的、可审计的记录。依赖 DAG 自动推进工作流，断路器防止无意义的重试，结构化交接让下游 worker 自动获得上游的决策上下文。

对于正在构建多智能体系统的 AI 开发者，以下是你应该从本文带走的关键判断：

1. **Kanban 不是 delegate_task 的替代品**，而是互补品——函数调用给短任务，持久队列给长工作流
2. **结构化交接比上下文窗口更可靠**——`summary` + `metadata` 让管道中的每一步都有完整的决策上下文
3. **可观测性不是附加品，是核心功能**——`kanban tail` 和 `kanban watch` 让调试多智能体系统从玄学变成科学
4. **断路器不是失败，是优雅的失败管理**——自动阻塞和人工介入比无休止的重试更高效
5. **单机版不是缺陷，是设计选择**——SQLite 的便携性和零依赖对于个人开发者和中小团队来说是巨大优势

Kanban Swarm 系统已经在社区中被用来构建从自动代码审查、批量文档翻译到 18 个 Agent 并行调研的全自动工作流。它证明了多智能体系统不需要复杂的编排框架——只需要一个可靠的 SQLite 数据库、一个简单的状态机和一套清晰的工作交接协议。

---

*参考资源：*
- [Hermes Agent 官方文档 — Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Kanban 教程](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial)
- [Hermes Agent v0.16 Surface Release](https://hermes-agent.ai/blog/hermes-agent-v0-16-surface-release)
- [Hermes Agent 多智能体工作流说明](https://hermes-agent.ai/blog/hermes-agent-multi-agent)
- [The Hermes Kanban: A Complete Guide](https://magnus919.com/2026/05/the-hermes-kanban-a-complete-guide-to-multi-agent-task-orchestration)
- [Tonbi 的多智能体工作流模板](https://github.com/tonbistudio/hermes-multi-agent-workflow)
