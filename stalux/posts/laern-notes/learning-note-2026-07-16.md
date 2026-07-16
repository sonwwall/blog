---
title: 学习笔记：从 Agent 记忆到工程化工作流设计
abbrlink: learning-note-20260716
date: 2026-07-16T17:12:14
updated: 2026-07-16T17:12:14
tags:
  - 学习笔记
  - 学习
  - Agent
  - AI 工程
categories:
  - 学习笔记
desc: 从跨 Session 记忆、Agent Harness、Workflow、Skill 和机器门禁出发，系统整理可靠 Agent 的设计与使用方法。
---

# 学习笔记：从 Agent 记忆到工程化工作流设计

今天围绕 Agent 的使用与设计，重点学习了两篇文章：一篇讨论如何让 Agent 通过外部记忆跨 Session 积累经验，另一篇讨论如何使用 Workflow、Skill、Evidence、State 和机器门禁，把测试 Agent 接入真实工程流水线。

两篇文章关注的是不同层次的问题：

```text
第一篇：Agent 怎样记住过去的经验？
第二篇：怎样保证 Agent 按正确流程使用这些经验？
```

结合后续关于“开发项目时怎样使用 AI”“设计 Agent 时是否必须写 Workflow”“Codex 有没有 Workflow”的讨论，可以形成一套更完整的 Agent 工程方法：

> 模型负责理解与灵活决策，规则和知识负责提供上下文，Workflow 负责约束阶段，Tool 和 Gate 负责守住不可违反的底线，人负责批准高风险动作和长期规则。

## 一、基础概念：FRP 是什么

对话开始时还了解了 FRP。在开发和运维语境中，FRP 通常指 Fast Reverse Proxy，是一个常见的内网穿透工具。

它由两个主要组件组成：

- `frps`：运行在拥有公网 IP 的服务器上。
- `frpc`：运行在内网机器上，主动连接 `frps`。

典型链路是：

```text
外网用户
  ↓
公网服务器 frps
  ↓
内网机器 frpc
  ↓
本地 Web、SSH、NAS 等服务
```

FRP 与 Nginx 反向代理的主要区别是，FRP 更侧重将无法被公网直接访问的内网服务暴露出去；Nginx 通常用于代理已经能够互通的网络服务。

## 二、问题一：为什么一次性的 Agent Loop 不够

第一篇文章《意识 × Loop：让 Loop 跨 Session 自进化的最佳实践》讨论了一个常被忽略的问题：Agent 即使能在一次会话中自动执行任务，也不代表它能持续进步。

普通 Loop 通常是：

```text
接收任务
→ 调用工具
→ 生成结果
→ 自检
→ 修改
→ 完成
```

问题在于，会话结束后，这次任务中积累的经验可能随上下文一起消失。

例如，用户已经多次纠正 Agent：

- 关键数字必须提供来源。
- 不能只相信企业官方宣传稿。
- 定性判断要区分事实和推断。
- 不要在报告结尾追加空洞的“综合评价”。

如果新 Session 仍然需要重新说明这些要求，那么 Loop 只是一次性自动执行，并没有真正形成复利。

文章所谓的“跨 Session 自进化”，并不是重新训练模型或修改模型参数，而是：

```text
持久化外部规则和记忆
→ 新 Session 自动加载
→ 使用过程中继续纠错
→ 将稳定经验升级为长期规则
```

## 三、意识层的三个文件

文章使用三个文件承担不同职责。

### 1. AGENTS.md：长期硬规则

`AGENTS.md` 保存已经得到验证、后续任务必须遵守的规则，例如：

```text
任何关键数字都必须有来源。
缺少直接证据的判断必须标注为定性推断。
输出前必须执行指定测试。
不得删除失败测试来通过 CI。
不得执行任务范围之外的重构。
```

它类似于项目宪法、质量标准和验收清单。

规则应该少而硬。规则太多时，Agent 的注意力会被稀释，甚至出现规则互相冲突。比较合理的做法是只保留 10～20 条高频、长期、重要规则。

### 2. MEMORY.md：尚未固化的经验

`MEMORY.md` 保存当前工作中学到的经验，例如：

```text
这个文档 API 写入长文本时必须分批提交。
这个项目的集成测试依赖本地 PostgreSQL。
修改订单状态时还需要同步写入 Outbox Event。
```

它不是权威规则，而是规则候选。

经验经过多次验证后，应移动到更稳定的位置：

- 通用开发约束进入 `AGENTS.md`。
- 架构决策进入 ADR。
- 项目事实进入 `CONTEXT.md` 或正式文档。
- 可自动验证的行为变成回归测试。
- 已经失效的经验直接删除。

### 3. USER.md：个人判断偏好

`USER.md` 保存用户的个人偏好，例如：

```text
偏好短句和明确结论。
不喜欢“综合来看”“值得关注”等空洞表达。
不要使用客套话。
代码修改说明应先讲结果，再讲过程。
```

这些内容不是客观质量标准，而是 Judge 的主观偏好。

三类信息应该分开，因为它们具有不同优先级：

```text
客观规则
> 当前任务要求
> 用户表达偏好
```

## 四、经验如何形成跨 Session 的进化循环

完整过程可以表示为：

```text
新 Session 加载规则和记忆
        ↓
Agent 执行任务
        ↓
用户发现问题并纠正
        ↓
将教训记录为经验候选
        ↓
后续任务自动使用经验
        ↓
同类经验多次复现
        ↓
由人决定是否升级为长期规则
```

这里最重要的控制点是：Agent 可以提出规则候选，但不能根据单次任务自动修改自己的最高级规则。

如果把一个偶发现象错误概括为通用规律，它会污染之后所有任务。规则从“本次经验”升级为“以后都必须遵守”，应该经过人工判断、Code Review 和版本控制。

## 五、从记忆走向 Harness

第二篇文章《agent-next：把测试 Agent 接进流水线——工作流、门禁与 Lazy Load 的 Harness 实践》将问题推进到了工程层。

它关注的核心矛盾是：

> Agent 能写出测试用例，不代表团队敢直接使用。中间缺少的是可检查、可复现、可阻断副作用的 Harness。

Harness 可以理解为包在 Agent 外部的一套工作控制系统。

没有 Harness 时：

```text
用户：帮我回归这个 Bug
Agent：直接写用例 → 猜接口 → 执行请求 → 输出报告
```

此时无法确定：

- Agent 是否读过 Bug 单和修复代码。
- 是否分析了影响范围。
- API 地址是否来自真实请求。
- 是否跳过必要步骤。
- 是否擅自操作共享环境。
- 产物是否符合团队模板。
- 最终结论能否被复现。

有 Harness 后：

```text
选择任务类型
→ 确认当前阶段
→ 加载当前 Skill
→ 收集证据
→ 按模板生成产物
→ 机器校验
→ 阶段门禁
→ 进入下一阶段
```

Harness 的价值不是提高模型智商，而是限制模型的行动范围，并为每一步留下证据。

## 六、为什么“大 Skill + 长对话”容易失败

把完整流程塞进一个巨大 Prompt 或 Skill，常见问题包括：

1. 规则太长，Agent 只关注前半部分。
2. 不同类型任务混在一起，阶段顺序不清楚。
3. 用户一句“帮我验证”，Agent 就直接跳到执行阶段。
4. Agent 凭记忆手写产物，不使用正式模板。
5. 运行状态和正式交付物混在一起。
6. 文档只说“应该检查”，却没有程序真正阻止漏检。

因此，可靠 Agent 不应该只依赖一个万能 Prompt，而应把职责分层。

## 七、可靠 Agent 的五层 Harness

文章将 Harness 分为五层：

```text
入口层 → 流程层 → 技能层 → 证据层 → 机器层
```

### 1. 入口层：Router 负责选路

Router 判断任务属于哪种类型。

例如开发 Agent 可以有：

```text
feature-development
bug-fix
refactoring
release-validation
```

Router 只输出任务入口和初始阶段，不应该自己分析代码、编写实现、运行测试或生成报告。

```json
{
  "entry": "bug-fix",
  "initial_phase": "intake",
  "reason": "用户报告已有功能发生异常"
}
```

Router 越薄，职责越清晰。

### 2. 流程层：Workflow 规定阶段

不同任务可以共享 Skill，但阶段顺序不同。

Bug 修复可以设计为：

```text
Intake
→ Reproduction
→ Root Cause
→ Impact Analysis
→ Fix Plan
→ Implementation
→ Verification
→ Report
```

新功能开发可以设计为：

```text
Requirement
→ Acceptance Criteria
→ Architecture Impact
→ Implementation Plan
→ Implementation
→ Testing
→ Documentation
→ Delivery
```

每个阶段都应该定义：

- 目标。
- 输入。
- 允许执行的动作。
- 必须提供的证据。
- 产物。
- 进入下一阶段的条件。

### 3. 技能层：Skill 负责具体方法

Workflow 决定“现在做什么”，Skill 决定“具体怎么做”。

例如：

```text
skills/
├── reproduce-bug/
├── analyze-stacktrace/
├── inspect-database/
├── design-api/
├── write-unit-tests/
├── security-review/
└── generate-report/
```

Skill 应保持窄而专。不要创建一个包含需求、架构、编码、测试和发布全部内容的超大 Skill。

### 4. 证据层：区分 Knowledge 和 Evidence

Knowledge 解释“通常应该怎样”，例如：

- 领域术语。
- 订单状态机。
- API 规范。
- 权限模型。
- 指标计算口径。

Evidence 证明“本次任务实际发生了什么”，例如：

- 本次 Bug 日志。
- Git diff 和 Commit。
- 复现请求。
- 出错 SQL。
- 浏览器 Network 请求。
- 实际测试结果。

Agent 不能用通用 Knowledge 代替本次 Evidence。

如果 README 说接口通常是 `/api/orders`，但浏览器真实请求是 `/gateway/v2/orders`，本次任务必须以真实请求为准。

### 5. 机器层：Tool 和 Gate 强制执行

自然语言规则只能提醒 Agent，机器门禁才能真正阻止错误流程继续。

常见工具包括：

- `run_state.py`：维护任务状态。
- `phase_doc.py`：定位当前阶段文档。
- `copy_template.py`：从模板创建产物。
- `validate_artifact.py`：校验产物格式。
- `validate_run_state.py`：校验任务状态。
- `stage_gate.py`：判断能否进入下一阶段。

例如，进入 Implementation 前必须满足：

```text
问题已经成功复现。
根因具有代码或日志证据。
影响范围已经分析。
修复方案已经生成。
必要产物通过格式校验。
```

任何条件不满足，Gate 就应该拒绝推进。

## 八、State：任务状态不能只存在聊天里

长时任务不能依赖聊天上下文保存状态，而应该维护机器可读的状态文件：

```json
{
  "run_id": "bug-1649",
  "entry": "bug-fix",
  "phase": "root-cause",
  "completed_phases": [
    "intake",
    "reproduction"
  ],
  "evidence": [
    "logs/order-error.log",
    "src/order/service.go:142"
  ],
  "artifacts": [
    "outputs/bug-1649/reproduction.md"
  ],
  "confirmations": [],
  "status": "in_progress"
}
```

State 可以支持：

- 跨 Session 恢复。
- 更换模型后继续执行。
- 检查 Agent 是否跳阶段。
- 审计它读取过哪些证据。
- 失败后从当前阶段重试。

## 九、运行数据和正式产物必须分离

推荐目录：

```text
runs/       Agent 的状态和中间信息
outputs/    用户需要的正式产物
```

例如：

```text
runs/bug-1649/state.json
runs/bug-1649/debug-notes.md

outputs/bug-1649/root-cause.md
outputs/bug-1649/fix-plan.md
outputs/bug-1649/regression-report.md
```

二者混在一起，会让 Agent 和用户都难以判断哪个文件才是最终版本。

## 十、Lazy Load：每次只读取当前需要的内容

可靠 Agent 不应该在任务开始时加载所有 Workflow、Skill 和知识文档。

合理顺序是：

```text
1. AGENTS.md
2. Router
3. Workflow 索引
4. 当前 Phase
5. 当前 Phase 需要的 Skill
6. 当前任务相关 Knowledge
7. 当前任务 Evidence
```

如果当前处于 Root Cause 阶段，只需要读取：

```text
root-cause phase
analyze-stacktrace skill
相关代码
本次日志
```

不需要读取发布流程、测试报告 Skill 和 Deployment 阶段。

Lazy Load 的主要价值不是单纯节省 Token，而是减少无关规则对当前判断的干扰，让“此刻应该读什么”成为明确、可审计的动作。

## 十一、Agent 的副作用必须分级

Agent 的操作可以分为三个等级。

### 可以自动执行

- 阅读文件。
- 搜索代码。
- 运行本地只读命令。
- 修改任务范围内代码。
- 运行相关测试。
- 生成本地产物。

### 执行后报告

- 安装项目依赖。
- 运行耗时较长的完整测试。
- 启动本地服务。
- 创建临时测试数据。

### 必须提前确认

- 推送远程分支。
- 创建或合并 PR。
- 部署服务。
- 操作生产环境。
- 修改共享数据库。
- 删除重要数据。
- SSH 到远程服务器。
- 向外部系统写入消息或缺陷。

高风险确认也应该进入 State，便于审计谁在什么时候批准了什么动作。

## 十二、Codex 到底有没有 Workflow

Codex 没有要求每个项目都编写显式 `workflows/` 目录，但不能说它没有 Workflow。

Codex 通常具有一个隐式、动态的通用循环：

```text
理解任务
→ 阅读仓库规则
→ 检查相关代码
→ 必要时制定计划
→ 修改代码
→ 运行验证
→ 检查结果
→ 汇报
```

这个流程由模型的系统指令、工具权限、`AGENTS.md`、当前 Prompt、Skill 和任务计划共同决定。

因此，更准确的说法是：

> Codex 自带动态 Workflow，但默认没有为每种业务任务预设固定状态机。

### 动态 Workflow 的特点

优点：

- 灵活。
- 适合未知任务。
- 不需要提前列举所有情况。
- 日常开发成本低。

缺点：

- 可能漏步骤。
- 不同 Session 行为不完全一致。
- 很难证明某项检查真的执行过。
- 不适合高风险无人值守任务。

### 显式 Workflow 的特点

优点：

- 可预测。
- 可恢复。
- 可审计。
- 不容易跳步。
- 可以设置机器门禁。

缺点：

- 设计和维护成本较高。
- 过度严格会限制 Agent。
- 容易产生形式主义文档。

## 十三、什么时候需要显式 Workflow

并非所有 Agent 都需要完整状态机。关键问题是：

> 如果 Agent 漏掉某一步，后果是否严重？

可以采用分级策略：

| 任务情况 | 推荐方式 |
| --- | --- |
| 一次性、低风险、步骤未知 | Codex 动态规划 |
| 经常重复、风险一般 | Skill + 检查清单 |
| 长时、跨 Session | Workflow + State |
| 多 Agent 并行 | Workflow + State + 文件边界 |
| 部署、数据、安全操作 | Workflow + Gate + 人工确认 |
| 无人值守生产任务 | 完整状态机、审计、权限和回滚 |

普通任务，例如修改页面样式、添加简单字段、解释代码或修复明确的小 Bug，使用以下组合通常就够了：

```text
清晰需求
+ AGENTS.md
+ 验收条件
+ 测试命令
```

而数据库迁移、生产部署、安全漏洞修复、支付回归、批量数据修复等任务，不能只依赖模型“记得检查”，必须设计显式阶段和机器门禁。

## 十四、最合理的三层结构

不需要在“完全自由”和“完整状态机”之间二选一，可以使用三层结构。

### 第一层：模型动态规划

```text
理解 → 检查 → 实现 → 验证 → 汇报
```

模型负责灵活处理现场情况。

### 第二层：Skill 推荐流程

```text
skills/
├── diagnose-bug/
├── database-migration/
├── security-review/
└── release-check/
```

Skill 保存重复任务的稳定方法，但允许 Agent 根据实际情况调整。

### 第三层：机器 Gate

```text
测试未通过，不允许合并。
迁移没有回滚方案，不允许执行。
没有用户确认，不允许部署。
没有安全扫描，不允许发布。
```

三者的关系是：

```text
模型负责灵活决策
Skill 负责推荐方法
Gate 负责不可违反的底线
```

## 十五、开发项目时使用 AI 的最佳实践

对于普通项目开发，可以采用以下工作方式。

### 开始任务前

```text
1. 读取 AGENTS.md。
2. 阅读相关 CONTEXT、ADR 和代码。
3. 检查当前工作区状态。
4. 明确验收条件。
5. 明确不修改范围。
```

### 开发过程中

```text
1. 先复现问题或建立测试。
2. 实现最小改动。
3. 运行最相关测试。
4. 检查 diff 是否越界。
5. 再运行更大范围验证。
```

### 任务结束时

```text
1. 总结实际改动。
2. 列出运行过的验证。
3. 说明未验证内容和剩余风险。
4. 将新经验记录为候选。
5. 判断是否需要测试、ADR、文档或 Gate 固化。
```

对重复出现的问题，最好的沉淀路径通常是：

```text
踩坑
→ 理解根因
→ 添加回归测试
→ 必要时更新 Skill 或规则
→ 机器永久防止复发
```

## 十六、设计 Agent 的正确实施顺序

不要一开始就搭建完整 Agent 平台。更合理的顺序是：

```text
第一步：选择一个高频任务。
第二步：写出主要阶段。
第三步：定义每个阶段的输入、产物和 Gate。
第四步：建立 state.json。
第五步：加入模板和格式校验。
第六步：拆出两三个核心 Skill。
第七步：增加高风险操作确认。
第八步：使用真实任务验证。
第九步：离线改进 Workflow、Skill 和 Tool。
第十步：稳定后再增加其他 Workflow。
```

例如先完整实现 `bug-fix`，用三个真实 Bug 验证整个流程，再考虑新功能开发和发布验收。

## 十七、容易混淆和容易犯错的地方

### 1. Workflow 不是越详细越好

Workflow 应约束关键决策点，而不是记录每一个微小动作。

Bug 修复至少应确认：

- 是否复现。
- 根因证据是什么。
- 影响范围是什么。
- 是否增加回归测试。
- 验证是否通过。

至于先阅读 Controller 还是 Service，可以让 Agent 动态决定。

### 2. Memory 不是项目真相

Memory 可能过时，聊天记录更可能不完整。建议采用以下信息优先级：

```text
测试、编译器、Schema
> 当前代码行为
> ADR 和正式文档
> AGENTS.md
> MEMORY.md
> 当前聊天记录
```

### 3. Skill 不是最高级规则

Skill 负责方法，不能代替权限控制。必须执行的检查要进入 Tool 或 Gate。

### 4. Agent 自进化不能等于自动修改规则

推荐路径是：

```text
多次任务出现同类问题
→ 收集失败轨迹
→ 离线归纳共同原因
→ 人工审核
→ 修改 Skill、Workflow 或 Tool
→ 添加测试
→ 合并进 Git
```

### 5. AI 生成速度不等于开发效率

真正的效率需要考虑：

- 是否减少返工。
- 是否能够验证。
- 是否避免重复错误。
- 是否能够被其他人理解。
- 是否可以安全进入流水线。

## 十八、最终认识

今天最大的认识是，Agent 的可靠性不能只靠更强的模型或更长的 Prompt。

一个成熟 Agent 系统至少需要解决四个问题：

```text
知道什么：Knowledge、Context、Evidence
现在做什么：Router、Workflow、State
具体怎么做：Skill、Tool
怎样保证不出界：Test、Gate、Confirmation、Human Review
```

对于 Codex 这样的通用编码 Agent，不需要给每一个普通任务编写固定 Workflow。模型的动态规划适合处理开放、低风险、一次性的开发任务。

当任务变得重复、长时、高风险、多人协作或需要无人值守时，就应该逐步增加 Skill、State、显式 Workflow、机器 Gate 和人工确认。

整套方法可以浓缩为：

```text
AI 开发效果
= 清晰任务
× 可执行验证
× 有边界的自主权
× 持久化项目上下文
× 独立审查
× 人类最终判断
```

最有价值的知识沉淀，也不是把所有经验写进越来越长的 Memory，而是把稳定经验转化为测试、工具、门禁和经过审查的项目规则。
