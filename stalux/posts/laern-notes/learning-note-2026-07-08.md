---
title: 学习笔记：多 Agent 协作与托管式 Agent 平台
abbrlink: learning-note-20260708
date: 2026-07-08T18:19:16
updated: 2026-07-08T18:19:16
tags:
  - 学习笔记
  - 学习
  - AI Agent
  - 安全测试
categories:
  - 学习笔记
desc: 对多 Agent 协作、安全测试 Agent 流程和托管式编码 Agent 平台的系统整理。
---

# 学习笔记：多 Agent 协作与托管式 Agent 平台

今天主要围绕两个项目学习了多 Agent 系统的不同形态：一个是面向安全测试的 Pentest Swarm AI，另一个是面向软件工程团队的 Multica。前者关注多个安全 Agent 如何围绕目标协作完成侦察、分析、验证和报告；后者关注如何把不同编码 Agent 管理成可以被分配任务、汇报进度、持续工作的团队成员。

这两个项目都不是单纯的“调用大模型回答问题”，而是在大模型之外增加了任务状态、运行时、工具调用、权限边界和协作机制。它们体现了 Agent 应用从“单次对话助手”走向“可管理的执行系统”的趋势。

## Pentest Swarm AI 的核心定位

Pentest Swarm AI 是一个自动化渗透测试框架，目标是把多个安全测试 Agent 组织起来，在授权范围内完成安全测试流程。它不是重新实现所有安全扫描能力，而是把成熟的外部安全工具接入 Agent 流程。

可以把它理解成三层：

1. 底层是外部安全工具，例如子域名发现、端口扫描、HTTP 服务识别、模板化漏洞检测等。
2. 中间层是项目自己的工具适配器、scope 校验、黑板存储和调度逻辑。
3. 上层是不同职责的 Agent，例如侦察、分类、验证和报告。

所以这个项目的重点不在于“自己写了一个新的 nmap”，而在于“如何让 Agent 有组织地调用工具，并把工具结果变成可继续推理和协作的结构化信息”。

## 工具链的含义

在这个项目里，工具链主要指外部安全工具加上项目内部封装适配器的组合。

外部工具负责真实执行动作，例如：

- `subfinder` 发现子域名。
- `naabu` 扫描端口。
- `httpx` 识别 HTTP 服务。
- `nmap` 识别端口、服务和版本。
- `nuclei` 使用模板检测常见漏洞和错误配置。
- `katana` 爬取网页路径和端点。
- `dnsx` 做 DNS 解析和探测。
- `gau` 收集历史 URL。

项目本身负责把这些工具组织起来：

```text
调用外部工具
    ↓
解析工具输出
    ↓
转成统一 finding
    ↓
写入共享黑板
    ↓
触发后续 Agent
```

这样做的好处是安全工具可以继续复用成熟生态，Agent 系统则专注在调度、分析、判断和报告上。

## Scope 的作用

`scope` 是授权测试范围。渗透测试不能随意扩展目标，必须明确哪些资产允许测试，哪些资产不能碰。

例如：

```bash
scan example.com --scope example.com
```

这里的含义是：

- `target` 是从哪里开始测试。
- `scope` 是允许继续深入的边界。

如果侦察阶段发现 `api.example.com`、`admin.example.com`，通常还在 `example.com` 范围内，可以继续处理。如果发现第三方 CDN、支付平台、GitHub Pages、外部 SaaS 域名等，就应该被 scope 拦住，不能继续扫描或验证。

因此，scope 不是一个普通参数，而是安全边界。它约束 Agent 和工具链，避免自动化系统因为链接跳转、资产关联或误判而越界。

## 多 Agent 协作方式

Pentest Swarm AI 的多 Agent 协作方式接近“黑板模式”。多个 Agent 不一定直接互相聊天，而是围绕一个共享的 blackboard 读写结构化发现。

典型 finding 包括：

- `TARGET_REGISTERED`
- `SUBDOMAIN`
- `PORT_OPEN`
- `HTTP_ENDPOINT`
- `TECHNOLOGY`
- `CVE_MATCH`
- `MISCONFIGURATION`
- `EXPLOIT_CHAIN`
- `EXPLOIT_RESULT`

流程可以理解成：

```text
Recon Agent 发现资产
    ↓
写入 SUBDOMAIN / PORT_OPEN / HTTP_ENDPOINT
    ↓
Classifier Agent 读取发现并判断风险
    ↓
写入 CVE_MATCH / MISCONFIGURATION
    ↓
Exploit Agent 对高价值线索做验证
    ↓
写入 EXPLOIT_RESULT
    ↓
Report Agent 汇总证据并生成报告
```

这种协作方式类似一个专门为安全测试设计的 Teambox：每个 Agent 都在同一个共享空间里留下线索，其他 Agent 根据自己的职责继续推进。但它比普通协作空间更结构化，因为黑板里的内容不是聊天消息，而是机器可查询、可触发的 finding。

## 四类核心 Agent

这个项目里可以把核心 Agent 理解为四类。

### Recon Agent

Recon Agent 负责侦察。它调用子域名发现、端口扫描、HTTP 探测、路径爬取等工具，把原始攻击面收集出来。

它回答的问题是：

- 目标有哪些子域名？
- 哪些端口开放？
- 哪些服务是 HTTP 服务？
- 有哪些路径、接口、历史 URL？
- 服务可能使用了什么技术栈？

Recon 的输出是后续所有分析的基础。

### Classifier Agent

Classifier Agent 负责分类和风险判断。它读取侦察阶段发现的服务、端口、技术栈、路径和响应信息，判断这些信息是否可能对应安全风险。

它回答的问题是：

- 某个服务版本是否可能对应已知 CVE？
- 某个配置是否可能是错误配置？
- 某个接口是否值得进一步验证？
- 风险等级和置信度大概是多少？

它的输出通常是更高层的 finding，例如 `CVE_MATCH` 或 `MISCONFIGURATION`。

### Exploit Agent

Exploit Agent 容易被误解。这里的 exploit 不应该理解成无节制攻击，而应该理解成授权范围内的漏洞验证。

它负责确认某个风险是否真实存在，例如：

- 这个 CVE 是否真的可触发？
- 这个错误配置是否真的能造成信息泄露？
- 这个接口是否真的存在可复现的安全问题？
- 有哪些低破坏性的证据可以放进报告？

一个健康的 Exploit Agent 应该遵守 scope，并优先做低风险验证。它的价值不是“打穿目标”，而是把疑似风险变成可复现、可说明、可修复的证据。

### Report Agent

Report Agent 负责汇总黑板中的结果，把发现、证据、严重程度、复现路径和修复建议整理成报告。

它解决的问题是：

- 哪些 finding 是有效的？
- 哪些是重复或低置信度的？
- 哪些有验证证据？
- 应该如何描述影响和修复建议？

报告是安全测试流程的最终交付物。

## 并发与 Agent 实例

多 Agent 不等于每种 Agent 只有一个实例慢慢执行。更准确的理解是：

```text
Agent 类型：Recon / Classifier / Exploit / Report
Agent 任务：某类 Agent 针对某个 finding 的一次执行
Scheduler：控制触发条件、并发上限和执行预算
```

也就是说，项目可以只有四类核心 Agent，但同一类 Agent 可以并发处理多个 finding。例如黑板里出现很多 HTTP endpoint 时，Classifier Agent 可以并发分析不同 endpoint。这里并不是复制多个独立人格，而是同一种 Agent 逻辑被调度多次执行。

这种设计兼顾了两个点：

- 角色清晰：不同 Agent 类型负责不同阶段。
- 执行高效：同一类型可以并发处理多个任务。

## Pentest Swarm AI 的攻击测试流程

完整流程可以拆成七步。

第一步，输入目标和 scope。用户指定起点和授权边界。

第二步，目标注册到黑板。系统把目标写成初始 finding，触发侦察阶段。

第三步，Recon Agent 做资产发现。它发现子域名、端口、HTTP 服务、路径、技术栈等。

第四步，Classifier Agent 做风险判断。它把原始发现映射到 CVE、错误配置、敏感暴露等风险类型。

第五步，Exploit Agent 做漏洞验证。它只对高价值、高置信度、在 scope 内的 finding 做验证。

第六步，结果反馈回黑板。一次验证可能产生新的线索，从而触发新的分析或侦察。

第七步，Report Agent 生成报告。报告整合最终发现、证据、影响和修复建议。

这个流程不是单向流水线，而是带反馈循环的探索过程：

```text
发现 → 判断 → 验证 → 新发现 → 再判断 → 再验证 → 报告
```

## Multica 的核心定位

Multica 是一个托管式编码 Agent 平台。它不是自己实现一个新的 coding agent，而是管理各种已有的编码 Agent CLI，例如 Claude Code、Codex、Copilot CLI、Cursor Agent、OpenCode、Kimi、Kiro CLI 等。

它想解决的问题是：当团队里有很多 coding agent 时，如何像管理工程师一样管理它们。

传统使用 coding agent 往往是单次交互：

```text
打开终端
复制 prompt
等待 agent 执行
手动观察进度
手动整理结果
```

Multica 想把它变成团队协作流程：

```text
创建 issue
分配给 agent
agent 自动执行
平台追踪状态
agent 汇报进度和阻塞
结果沉淀成可复用技能
```

所以 Multica 更像是 AI Agent 时代的任务板、运行时管理平台和团队协作系统。

## Multica 的关键概念

### Agent as Teammate

Multica 把 Agent 当作团队成员，而不是一次性命令。Agent 可以有自己的身份，出现在任务板上，被分配 issue，发表评论，报告进度和阻塞。

这体现了一种重要变化：Agent 不只是“工具”，也可以是任务生命周期中的参与者。

### Runtime

Runtime 是实际执行 Agent 任务的计算环境。它可以是本地机器，也可以是云端机器。

Multica 的 daemon 会运行在 runtime 上，检测当前环境里有哪些 Agent CLI 可用。平台根据 runtime 能力决定哪些任务可以路由到哪里执行。

可以把 runtime 理解成 CI runner，只不过它执行的不是普通流水线任务，而是 AI Agent 任务。

### Daemon

Daemon 是连接平台和本地环境的桥。它负责：

- 连接 Multica 服务端。
- 注册当前机器为可用 runtime。
- 检测本机有哪些 Agent CLI。
- 接收分配过来的任务。
- 启动对应 Agent CLI 执行。
- 把进度和结果回传平台。

这让 Multica 能够统一管理不同机器上的 Agent 执行能力。

### Squads

Squads 是小队机制。多个 Agent 或人可以组成一个小队，由 leader agent 负责路由任务。

这解决的是规模化问题。当团队里 Agent 越来越多时，用户不应该总是纠结“这个任务分给哪个具体 Agent”，而可以分配给一个稳定的小队，例如 `FrontendTeam`、`InfraTeam`、`ReviewTeam`。

### Autopilots

Autopilots 是周期任务或自动触发任务。例如每天生成日报、每周做代码审计、定期检查依赖、接收 webhook 后创建任务等。

它把 Agent 从“被动接单”扩展到“按计划主动工作”。

### Reusable Skills

Reusable Skills 指把解决过的问题沉淀成团队可复用能力。比如部署流程、迁移经验、代码审查规则、项目约定等，都可以变成后续 Agent 能复用的技能。

这点很关键，因为 Agent 系统的长期价值不只是一次执行成功，而是团队知识能否持续复利。

## Multica 与 Pentest Swarm AI 的对比

两者都涉及多 Agent，但目标不同。

Pentest Swarm AI 面向安全测试，核心是围绕黑板的自动化探索：

```text
资产发现
风险分类
漏洞验证
报告生成
```

Multica 面向软件工程团队，核心是围绕 issue 的任务管理：

```text
任务创建
Agent 分配
执行跟踪
阻塞反馈
技能沉淀
```

前者更像安全测试领域的 swarm 系统，后者更像 coding agent 的团队管理平台。

它们共同说明了一个趋势：Agent 应用正在从“一个模型回答一个问题”变成“多个执行单元在一个系统里协作”。

## 容易混淆的点

第一，工具链不是项目全部自研的能力。工具链通常是外部工具和内部适配器的组合。项目通过封装和调度，把外部工具变成 Agent 能调用的能力。

第二，scope 不是扫描目标本身，而是授权边界。target 决定从哪里开始，scope 决定哪些发现可以继续深入。

第三，Exploit Agent 不等于恶意攻击者。在合规语境下，它更准确的职责是漏洞验证和证据收集。

第四，多 Agent 不一定意味着很多个大模型实例互相聊天。它也可以是多个角色围绕共享状态协作，每个角色由调度器按条件触发。

第五，Multica 不是新的代码模型，而是管理已有编码 Agent 的平台。它的价值在任务生命周期、运行时管理、协作可见性和技能沉淀。

## 实践理解

如果把 Agent 系统看成一个工程系统，不应该只关注“模型聪不聪明”，还要关注这些问题：

- 状态放在哪里？
- 任务如何被触发？
- 多个 Agent 如何避免重复劳动？
- 权限边界如何控制？
- 工具调用结果如何结构化？
- 失败和阻塞如何反馈？
- 成功经验如何沉淀？

Pentest Swarm AI 用黑板解决安全测试中的共享状态和协作触发问题。Multica 用 issue、runtime、daemon 和 workspace 解决编码 Agent 的任务管理问题。

这两个方向都说明，真正可用的 Agent 平台需要的不只是模型调用，还需要状态管理、执行环境、调度机制、权限控制和可观察性。

