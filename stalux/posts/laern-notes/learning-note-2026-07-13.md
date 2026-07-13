---
title: 学习笔记：多 Agent 协作模式与典型项目架构
abbrlink: learning-note-20260713
date: 2026-07-13T15:53:46
updated: 2026-07-13T15:53:46
tags:
  - 学习笔记
  - 学习
  - AI Agent
  - 多 Agent
  - 系统架构
categories:
  - 学习笔记
desc: 对多 Agent 协作的常见模式，以及 Cairn、Pentest-Swarm-AI、Multica 三种典型架构的系统整理。
---

# 学习笔记：多 Agent 协作模式与典型项目架构

今天主要学习了多 Agent 系统究竟如何协作，并结合 Cairn、Pentest-Swarm-AI 和 Multica 三个项目，比较了动态规划、事件驱动专家网络和层级主管三种不同架构。

这三个项目都会同时运行多个 Agent，但“有多个 Agent”只是表面现象。真正决定系统性质的是：谁负责提出下一步任务、任务如何分配、Agent 通过什么共享信息、系统如何验证结果，以及满足什么条件才会停止。

## 一、多 Agent 协作的基本问题

多 Agent 协作可以理解为：把一个复杂目标拆成若干任务，让多个 Agent 分别处理，再通过共享状态和调度机制把结果组合起来。

一个完整系统通常要回答以下问题：

1. 谁决定下一步做什么？
2. 任务由谁创建和分配？
3. Agent 如何交换信息？
4. 如何避免重复执行、冲突和死循环？
5. 如何判断结果是否可信？
6. 谁来判断整个任务已经完成？

常见组件可以概括为：

```text
用户目标
   ↓
Planner：拆解目标、决定方向
   ↓
Scheduler：选择 Agent、控制并发
   ↓
Worker Agent：执行具体任务
   ↓
Memory / Blackboard：保存共享状态
   ↓
Reviewer / Judge：验证结果
   ↓
Termination：判断是否结束
```

并不是每个系统都会把这些组件分别实现成独立 Agent。有的系统由一个 Manager Agent 同时负责规划和验收，有的把规划逻辑写在固定工作流里，也有的让每个 Agent 根据共享状态自行决定是否行动。

## 二、常见的多 Agent 协作模式

### 1. 主管分工模式

主管 Agent 负责拆解目标、分配任务、检查结果和决定是否继续：

```text
用户目标
   ↓
Manager Agent
   ├── Agent A：前端实现
   ├── Agent B：后端实现
   └── Agent C：测试验证
          ↓
      Manager 汇总
```

这种模式结构清楚，适合软件开发和办公协作。缺点是 Manager 容易成为瓶颈；如果 Manager 对目标理解错误，所有成员都可能沿着错误方向工作。

### 2. 固定流水线模式

任务按照预先设计的顺序执行：

```text
Research → Outline → Writer → Reviewer
```

它稳定、容易测试，适合步骤明确的重复任务。但它的灵活性有限，很多被称为“多 Agent”的系统，本质上只是多个大模型调用组成的流水线。

### 3. 共享黑板模式

所有 Agent 围绕同一个共享空间读写状态：

```text
Agent A ──写入──┐
Agent B ──读取──┼── 共享黑板
Agent C ──写入──┘
```

黑板中可以保存事实、任务、假设、错误和中间结果。Agent 不需要互相私聊，只要读取同一个共享状态即可。它适合动态加入 Agent、并行探索和保留完整协作记录。

难点是黑板会不断增长，必须处理重复、冲突、错误和过期信息。

### 4. 事件驱动模式

每个 Agent 订阅自己关心的事件：

```text
代码提交
   ├── Test Agent
   ├── Security Agent
   └── Review Agent

测试失败
   └── Fix Agent
```

事件一出现，对应 Agent 就自动工作。这种模式并发能力强、组件耦合较低，但必须防止 Agent 互相触发形成无限循环。

### 5. 自由讨论与竞争评审模式

多个 Agent 可以直接讨论、辩论，或者独立生成方案后交给 Judge Agent 选择：

```text
问题
 ├── Agent A → 方案 A
 ├── Agent B → 方案 B
 └── Agent C → 方案 C
               ↓
           Judge Agent
```

这种方式适合高不确定性的推理和方案比较，但成本较高，也不能保证 Judge 一定正确。

### 6. 混合模式

实际项目往往组合多种模式：

```text
Manager 拆解大目标
        ↓
任务写入共享看板
        ↓
专业 Agent 根据事件并行执行
        ↓
Reviewer 验证关键结果
        ↓
Manager 判断是否继续
```

因此，分析一个多 Agent 项目时，不能只问“它有几个 Agent”，而要看任务生成、共享状态、调度和终止机制。

## 三、Cairn：动态规划的通用探索模式

Cairn 把问题抽象成“起点明确、目标明确、路径未知”的状态空间搜索。渗透测试是它首先验证的场景，但这套模型也可以用于漏洞研究、CTF、数学证明等开放式问题。

### 核心数据结构

Cairn 的共享黑板由三个核心概念组成：

- `Fact`：已经确认的客观事实。
- `Intent`：准备探索但还没有结果的方向。
- `Hint`：人类随时加入的判断或提示。

一个项目还有两个特殊节点：

- `Origin`：已知起点。
- `Goal`：成功条件。

例如：

```text
Origin：目标是 10.0.0.8
Goal：获得 root shell
```

图会按照下面的方式生长：

```text
Fact：发现 8080 端口
   ↓
Intent：分析 8080 Web 服务
   ↓
Fact：发现文件上传接口
   ↓
Intent：检查上传文件能否执行
   ↓
Fact：成功获得 WebShell
```

### 三类任务

Cairn 将 Agent 工作收敛成三类：

1. `Bootstrap`：项目开始时先尝试直接解决整个问题。
2. `Reason`：读取完整图，判断目标是否完成，并提出新的 Intent。
3. `Explore`：领取一个 Intent，执行具体探索，最后写回一个 Fact。

协作循环是：

```text
读取完整事实图
      ↓
Reason 动态提出 Intent
      ↓
Dispatcher 分配给可用 Worker
      ↓
多个 Explore 并行执行
      ↓
写回新的 Fact
      ↓
再次 Reason
      ↓
直到已有事实足以证明 Goal 完成
```

### Cairn 中 Agent 的角色

Cairn 没有固定的“扫描 Agent”“利用 Agent”或“报告 Agent”。同一个 Claude Code、Codex 或 Pi Worker，这次可以执行 Reason，下一次可以执行 Explore。

因此，Cairn 的特点是：

- Agent 相对通用，可以互换。
- 下一步任务由 LLM 根据当前事实动态生成。
- Intent 显式记录从哪些 Fact 出发，并最终产生哪个 Fact。
- Agent 之间不直接聊天，通过事实图间接协作。

Dispatcher 仍然是集中控制面，负责 Worker 选择、Intent 认领、心跳、超时、容器和并发。也就是说，Cairn 在知识协作上采用黑板模式，在运行管理上采用集中调度。

### 优点与限制

优点：

- 适合未知路径问题，探索方向不需要预先写死。
- Fact 与 Intent 保留了清晰的因果链。
- 不同 Intent 可以并行执行。
- 更换 Agent 后端不会丢失共享状态。

限制：

- Reason 需要反复读取完整图，图很大时上下文成本较高。
- Fact 主要是自然语言，真实性仍依赖 Agent 判断。
- 错误 Fact 可能污染后续推理。
- 当前按单 Dispatcher 设计。
- “通用状态空间搜索”是方向，目前主要证明来自渗透测试场景。

## 四、Pentest-Swarm-AI：事件驱动的专家网络

Pentest-Swarm-AI 同样使用共享黑板，但它与 Cairn 的核心差异是：Agent 有固定专业角色，并通过 Finding 类型自动触发。

典型 Agent 包括：

- `Recon Agent`：资产发现、端口和 Web 侦察。
- `Classifier Agent`：分析漏洞、CVE 和错误配置。
- `Exploit Agent`：设计并验证攻击路径。
- `Report Agent`：汇总结果并生成报告。

### 协作过程

每个 Agent 声明自己关注哪些 Finding：

```text
TARGET_REGISTERED
       ↓
Recon Agent
       ↓
SUBDOMAIN / PORT_OPEN / HTTP_ENDPOINT / TECHNOLOGY
       ↓
Classifier Agent
       ↓
CVE_MATCH / MISCONFIGURATION
       ↓
Exploit Agent
       ↓
EXPLOIT_CHAIN / EXPLOIT_RESULT
```

Scheduler 不负责制定攻击计划，只负责订阅分发、并发、预算、限流和停止。真正的协作关系写在每个 Agent 的 Trigger 中。

### 信息素权重

每个 Finding 还带有信息素权重和半衰期。新鲜、高价值的发现权重较高，过期线索会逐渐衰减。

例如：

- 开放端口可以保持较长时间。
- 会话或 Token 的有效时间较短。
- Agent 错误只短暂保留关注度。
- Exploit Agent 只处理权重超过阈值的 CVE。

这使系统能够优先处理近期、高可信的发现，而不是让所有历史信息永久保持同等重要性。

### 它是否完全“涌现”

Pentest-Swarm-AI 没有中央 Planner，比固定流水线更加松耦合。但 Recon、Classifier、Exploit 的职责以及触发关系仍然由代码预先定义：

```text
Recon Finding → Classifier
CVE Match → Exploit
Campaign Complete → Report
```

所以更准确的描述是“基于共享黑板的事件驱动专家网络”，而不是完全自由的群体智能。

### 优点与限制

优点：

- 事件出现后立即响应，不需要反复调用全局 Planner。
- Agent 职责清晰，容易控制和测试。
- 类型化 Finding 比自然语言聊天更适合程序处理。
- 信息素衰减可以过滤陈旧、低价值线索。

限制：

- 新能力通常需要增加 Finding 类型、Agent 和 Trigger。
- 灵活性受到预定义领域模型限制。
- 默认协作路径仍然带有 Recon、Classify、Exploit 的业务顺序。
- 当前 Swarm Scheduler 仍标记为 alpha，默认运行路径使用内存黑板。

## 五、Multica：层级主管式的人机协作平台

Multica 与前两个项目的目标不同。它不是面向未知问题的自动搜索引擎，而是把 Coding Agent 变成可以被分配 Issue、发表评论、汇报阻塞和持续工作的团队成员。

它更像一个面向人类和 Agent 混合团队的任务操作系统。

### 基本协作单元

Multica 使用人类熟悉的协作对象：

- Workspace
- Project
- Issue
- Assignee
- Comment
- Status
- Sub-issue
- Dependency
- Activity Timeline

这些数据构成了广义共享黑板，但它保存的是团队工作状态，而不是 Cairn 那种 Fact/Intent 因果图。

### Squad 与 Leader

Multica 的多 Agent 协作主要通过 Squad 实现：

```text
              Squad Leader Agent
             /         |          \
       UI Agent    Test Agent    Human
```

当 Issue 分配给 Squad 时，系统不会自动同时唤醒所有成员，而是先路由给 Squad 的 `leader_id`。

Leader 会收到项目内置的协调协议，以及 Squad 成员名单、角色和 Skills。它需要：

1. 阅读 Issue 和验收条件。
2. 根据成员能力选择合适执行者。
3. 通过结构化 `@mention` 或子 Issue 委派任务。
4. 记录本轮判断是 `action`、`no_action` 还是 `failed`。
5. 委派后停止，不亲自继续实现。
6. 成员汇报后重新评估，再决定下一步。

协作过程可以表示为：

```text
人类把 Issue 分配给 Squad
          ↓
Multica 唤醒 Leader
          ↓
Leader 根据 Roster 和 Skills 分工
          ↓
通过结构化 @mention 唤醒成员 Agent
          ↓
成员执行并在 Issue 评论中汇报
          ↓
评论再次唤醒 Leader
          ↓
Leader 继续委派、升级给人类或结束任务
```

### Leader 是什么

Multica 自带 Leader 身份、路由机制和协调 Prompt，但不自带一个固定的系统 Leader Agent。

用户需要：

1. 先创建普通 Agent。
2. 创建 Squad。
3. 从已有 Agent 中选择一个作为 `leader_id`。
4. 添加其他 Agent 或人类成员。
5. 可选填写 Squad Instructions。

真正产生调度作用的是 `leader_id`，而不是成员列表中的 `role: leader` 文本。角色字段主要用于展示和给 Leader 提供上下文。

### Multica 的亮点

第一，Agent 是一等团队成员。Agent 和人类共用 Issue、评论、状态和任务看板，不需要另建一套 AI 工作流。

第二，Squad 是稳定路由层。外部只需要把任务交给 `Frontend Team`，内部成员变化由 Leader 处理。

第三，Leader 能看到成员挂载的 Skills，可以按能力而不是只按名字分工。

第四，同一个 `(Agent, Issue)` 可以恢复之前的 Session ID 和工作目录，使多轮任务保持上下文与文件状态。

第五，底层 Agent 后端中立，可以管理 Claude Code、Codex、Cursor Agent、OpenCode、Pi、Kimi、Copilot CLI 等多种工具。

第六，Autopilot 可以通过 Cron、Webhook 或手动方式自动创建并路由周期任务。

第七，工程控制比较完整，包括任务队列、claim、lease、并发限制、心跳、失败重试、取消、WebSocket 输出、工作目录隔离和权限控制。

### Multica 的限制

- Leader 是潜在瓶颈，成员反馈可能反复唤醒它。
- 分工质量依赖 Leader 是否遵守 Prompt 并正确使用 mention。
- Squad 成员不会自动 fan-out，`role` 也不会自动产生调度规则。
- Issue 评论不是严格的知识图，复杂任务的关键事实可能散落在讨论中。
- 它擅长管理 Coding Agent，而不是自动搜索一个模糊目标的未知解法。

## 六、三个项目的核心区别

| 项目 | 核心模式 | 谁决定下一步 | Agent 是否固定分工 | 共享载体 | 完成方式 |
| --- | --- | --- | --- | --- | --- |
| Cairn | 动态探索 + 共享事实图 | Reason Agent | 否，Worker 相对通用 | Fact、Intent、Hint | 判断 Fact 是否满足 Goal |
| Pentest-Swarm-AI | 事件驱动专家网络 | 各 Agent 的 Trigger | 是 | 类型化 Finding 和信息素 | Campaign Complete 信号 |
| Multica | 层级主管 + 任务看板 | Squad Leader 或人类 | 由用户配置 | Issue、Comment、Status | Issue 工作流和 Leader 判断 |

可以用三个比喻理解：

- Cairn 像探险队：定期看地图，动态决定下一步探索哪里。
- Pentest-Swarm-AI 像医院：某类检查结果出现后，自动送到对应科室。
- Multica 像软件公司：任务进入团队，由 Team Lead 分配给合适成员。

## 七、容易混淆的点

### 1. 多 Agent 不等于 Agent 群聊

工程系统通常更倾向使用任务、事件和结构化共享状态，而不是让 Agent 无限自由讨论。群聊容易消耗大量 Token，也容易产生重复、附和和无法收敛的问题。

### 2. 黑板模式不等于完全去中心化

Cairn 和 Pentest-Swarm-AI 都使用黑板，但仍有 Scheduler 或 Dispatcher 负责并发、预算和运行管理。共享知识可以去中心化，运行控制仍然可以集中。

### 3. 固定角色不一定等于固定流水线

Pentest-Swarm-AI 的 Agent 角色是固定的，但通过事件订阅工作，可以并发，也可以根据 Finding 形成反馈环。因此它比单向流水线更灵活，但仍保留预定义的领域结构。

### 4. Leader 角色不等于内置 Agent

Multica 内置的是 Leader 协议。具体哪个 Agent 当 Leader，需要用户创建 Agent 并配置 Squad 的 `leader_id`。

### 5. 多 Agent 不一定优于单 Agent

如果任务不能真正并行、状态没有有效共享、结果没有验证机制，那么多 Agent 可能只是增加成本和故障点。简单任务通常使用单 Agent 或固定工作流更合适。

## 八、实践中的选型建议

适合使用 Cairn 式动态探索的场景：

- 起点和目标明确，但中间路径未知。
- 每一步会产生可供后续利用的新事实。
- 需要并行尝试多个探索方向。
- 需要保留因果链和审计记录。

适合使用 Pentest-Swarm-AI 式事件专家网络的场景：

- 领域模型相对稳定。
- 输入和输出可以类型化。
- 不同专家的触发条件比较明确。
- 需要低延迟并行处理大量事件。

适合使用 Multica 式层级团队模式的场景：

- 人类和 Agent 共同参与长期项目。
- 工作主要通过 Issue、评论和状态管理。
- 需要管理多种 Coding Agent 和运行环境。
- 需要明确负责人、进度、权限和审计记录。

适合使用单 Agent 或固定流水线的场景：

- 步骤短且确定。
- 没有真正可并行的子任务。
- 共享状态和协作成本高于任务本身。
- 可以通过普通程序或工作流可靠解决。

## 九、总结

多 Agent 系统的本质不是“同时启动多个模型”，而是设计一套能够分工、共享、反馈、验证和停止的协作机制。

分析这类项目时，应该重点观察：

1. 任务是预先定义，还是运行时动态产生？
2. Agent 是通用 Worker，还是固定领域专家？
3. 信息通过对话、任务看板、事件还是事实图共享？
4. 是否有去重、租约、心跳、预算和重试机制？
5. 关键结果由谁验证？
6. 系统根据目标、状态、事件还是时间预算结束？

Cairn、Pentest-Swarm-AI 和 Multica 分别代表了三种很有辨识度的方向：通用问题探索、领域事件网络和人机团队管理。它们没有绝对的优劣，关键在于协作模型是否与问题结构匹配。
