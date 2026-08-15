---
title: "一条命令生成三层多智能体流水线：Hermes Agent 的 Kanban Swarm"
date: "2026-08-16T00:00:00+08:00"
tags: ["Hermes Agent", "Kanban", "多智能体", "AI 开发"]
author: "ndflj"
---

# 一条命令生成三层多智能体流水线：Hermes Agent 的 Kanban Swarm

你让一个 subagent 去调研竞品。它跑了四十分钟，中间结论都攒在上下文里。然后宿主进程崩了——子代理连同它脑子里的东西一起蒸发，你只剩一条报错日志。

这是 `delegate_task` 这类进程内 fork/join 模式的宿命：快，但脆。任务一交出去，命运就绑在父进程的生死上；跑完之前没人能看它一眼，跑完之后也留不下任何审计记录。

Hermes Agent 在 v0.15.0（2026-05-28，"The Velocity Release"，横跨 104 个 PR）给出了另一条路：`hermes kanban swarm`。一条命令把整条多代理流水线变成一张持久化的 SQLite 看板——任务、状态、交接、失败原因全部落库。进程死了，任务不死。到 v0.16.0（2026-06-05，"The Surface Release"），这条流水线又搬进了桌面端 dashboard：列视图、拖拽改状态、依赖编辑器、Nudge dispatcher 按钮。版本脉络一句话：swarm v1 在 v0.15 定型，v0.16 把它全面 GUI 化了。

## 进程内 subagent 的三种死法

用 `delegate_task` 编排多步工作流，有三个绕不开的坑：

1. 无持久化。任务状态存在父进程内存里。父进程重启，一切归零。跑了一个小时的调研说没就没。
2. 崩溃即丢失。fork/join 是同步 RPC 语义：子代理抛异常，父代理要么吞掉重试，要么整体失败，没有中间状态可以捞。
3. 无法人审。任务在跑的时候，人类完全插不上手：没有看板看进度，没有评论可留言，没有"暂停重派"这种操作。

这三个坑的共同根源：任务生命周期没有离开进程，没有变成数据。Kanban 做的第一件事，就是把任务从内存里搬出来，落成数据库里的一行。

## 看板不是任务队列，是状态机

Hermes Kanban 的核心是一块 SQLite 看板（`~/.hermes/kanban.db`），每个任务一行，生命周期是显式状态机：`todo → ready → running → done / blocked`。`task_links` 表记依赖边，`task_events` 表记每一次状态变迁，`task_runs` 表记每一次执行尝试。

为什么重要：状态机是"可恢复"的前提。进程随便崩，数据库里那行任务知道自己进行到哪一步了。dispatcher 回收它、重派它，都是从落库的状态继续，而不是从零开始。

在这块板上跑的是命名 profile——每个 profile 是独立的 agent 身份，有自己的 config、skills、memory、session，是完整的 OS 进程。任务在板上的流转，本质上是"一条记录 + 一个被指派的进程"。人类和 agent 看到同一块板，谁都能 comment、block、unblock。

## 一条命令生成三层拓扑

```bash
hermes kanban swarm "写一篇2000字技术博客" \
  --workers researcher:"调研":hermes-agent \
  --verifier reviewer \
  --synthesizer writer
```

worker 参数格式 `profile:标题[:技能,技能]`，可重复。命令一次性提交整棵任务图：

```
root（黑板卡，立即 complete，只当审计锚点）
  ├─ researcher worker 卡（ready，可并行）
  └─ verifier 卡（todo，等所有 worker done 才 promote）
       └─ synthesizer 卡（todo，等 verifier done 才 promote）
```

几个设计细节值得单独说。

依赖门机制。dispatcher 每个 tick 检查 `task_links`：只有"所有 parent 都 done"的卡才会从 `todo` promote 到 `ready`。worker 并行跑完，verifier 才被唤醒；verifier 用 metadata `{"gate": "pass"}` 显式通过门禁，synthesizer 才被唤醒。依赖是数据结构，不是写在自然语言里的约定。

verifier 门禁约定。verifier 卡的 body 写死了规则：证据充分才带 `{"gate": "pass"}` complete，否则必须 block 并列出缺什么。默认给 verifier 加载 `requesting-code-review` 技能、给 synthesizer 加载 `humanizer` 技能——技能钉在单张卡上，不用改 profile。

原子性与幂等。整棵图一次性提交，调度器要么看到完整拓扑，要么什么都看不到。`--idempotency-key` 命中已有 root 时，直接从黑板读拓扑复用，不重复建图。

## 黑板：刻意低科技的结构化评论

跨 worker 通信怎么做？设计者选了个意外朴素的方案：结构化 JSON 评论挂在 root 卡上，前缀 `[swarm:blackboard]`。

```json
[swarm:blackboard] {"key": "research", "value": {"task": "t_814b3e7b", "done": true}}
```

合并规则两条：同 key 后写覆盖先写；`_authors` 字段记录每个 key 的最终作者，方便溯源。

为什么重要：状态仍然留在 `task_comments`/`task_events` 表里，dashboard、notifier、slash command、dispatcher 不需要新组件就能继续工作。新造一个黑板服务，等于给系统加一个必须时刻在线的故障点；评论是看板本来就有的能力。

## dispatcher：每 60 秒一次的可靠性引擎

dispatcher 默认内嵌在 gateway 进程里（`dispatch_in_gateway: true`），不需要独立守护进程。每 60 秒一个 tick，做四件事：回收过期 claim、promote ready 任务、原子 claim、按 assignee 派生 profile 进程。

可靠性设计全部在 `task_events` 表留痕：

- claim 带 TTL（15 分钟）；PID 消失但 TTL 没到 → 记 `crashed`，可回收重派。
- 跑超 4 小时且最近 1 小时没有 `kanban_heartbeat` → `stale`，SIGTERM 后回 `ready` 重派，不记失败。
- 连续派生失败达 `failure_limit`（默认 2）→ 自动 `blocked`，防止 profile 不存在这类问题无限抖动。
- respawn guard：上次失败是 quota/auth/429，或 1 小时内刚成功过 → 本 tick 拒派。
- 正常退出但没调 complete/block → `protocol_violation`，连续 3 次自动 block。

多代理系统的故障不是"会不会发生"，而是"什么时候发生"。这套设计把每一种死法变成数据库里的一行事件，再给每种死法一条确定的恢复路径。

## 多 profile 编排：拆解的人不干活

swarm 里的角色分工是硬性的。orchestrator 只负责拆解、指派、链接，不亲自干活；worker 只做自己被指派的卡，不越权拆新任务。建议把 orchestrator profile 的 toolsets 限制到 board 操作（kanban/gateway/memory），从物理上杜绝它顺手执行实现任务。

成本策略也很直白：orchestrator 用前沿模型做拆解（token 消耗小），worker profile 用廉价模型执行（token 大头在 worker）；质量敏感的卡再用 per-task `--model/--provider` 覆盖。技能用 `--skill` 数组钉到单张卡上——翻译、代码评审这些专家技能不用改 profile 就能挂到具体任务。

还有一个容易忽略的点：worker 驱动看板用的是 `kanban_*` 工具集，不是 shell 里的 `hermes kanban` CLI。原因很实际——远程终端后端（Docker/Modal/SSH）里既没有 hermes 可执行文件，也没有挂载 kanban.db；工具走 agent 自身 Python 进程直达数据库。普通会话没有 `HERMES_KANBAN_TASK` 环境变量时，这套工具集根本不会加载，零 schema 占用。

## delegate_task vs Kanban

|  | delegate_task | Kanban |
|---|---|---|
| 语义 | 进程内 RPC，fork/join | 持久队列 + 状态机 |
| 身份 | 匿名子代理 | 命名 profile |
| 故障 | 崩溃即丢 | 回收、重派 |
| 审计 | 无 | SQLite 永久留痕 |
| 人审 | 无法介入 | comment / block / unblock |

## 这篇文章本身就是一个 swarm

最后交代一个事实：这篇博客就是一次 swarm 的执行结果，跑在 blog 板上。root 卡 `t_cc00edcc` 上挂着 `[swarm:blackboard]` 评论；researcher 调研并产出笔记（t_814b3e7b）；reviewer 做 verifier（t_309275ec），对照本地源码、发布说明、官方文档逐项核验后以 `{"gate": "pass"}` 通过门禁；我（writer）是 synthesizer，等门禁通过才动笔。

你看到的"三层拓扑 + 黑板 + 门禁"不是文档里的抽象概念，是正在发生的事。这套系统把人从"盯着 agent 干活"里解放出来，代价是把任务定义得足够清楚——清楚到能写进一张卡片的 body 里。
