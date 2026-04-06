# Coordinator 设计摘要

基于 [`src/coordinator/coordinatorMode.ts`](./src/coordinator/coordinatorMode.ts) 中 `getCoordinatorSystemPrompt()` 的提炼整理。

## 设计定位

Coordinator 不是直接干活的执行者，而是一个多 worker 协作系统里的指挥层。

它的核心职责只有四件事：
- 理解用户目标
- 把任务拆给合适的 worker
- 综合 worker 结果并做决策
- 把最新进展准确反馈给用户

一个关键约束是：Coordinator 负责“理解”和“决策”，Worker 负责“研究”、“实现”和“验证”。

## 核心工作流

这个设计把软件工程任务拆成四个阶段：

1. Research：并行派出多个 worker，从不同角度理解问题。
2. Synthesis：Coordinator 阅读结果，自己形成判断，并写出明确规格。
3. Implementation：把清晰的实现指令交给 worker 执行。
4. Verification：由独立 worker 验证代码是否真的可用。

其中最重要的一步不是分派，而是 `Synthesis`。  
Coordinator 不能把“理解研究结果”这件事继续外包出去，必须先消化，再下达具体指令。

## Prompt 设计原则

- Prompt 必须自包含，因为 worker 看不到主对话。
- 指令必须具体到文件路径、行号、错误信息和完成标准。
- 不允许写“based on your findings”这类懒委派表达。
- 继续已有 worker 还是新开 worker，取决于上下文重叠度，而不是固定偏好。
- 验证的目标是证明代码真的工作，不是简单确认“改动存在”。

这套原则的本质是：减少信息损耗，降低上下文污染，提高执行闭环质量。

## 这个设计的优点

### 1. 分层清晰

Coordinator 和 Worker 的职责边界非常明确，避免一个 agent 同时负责思考、执行、验收，导致角色混乱。

### 2. 强制中间综合

它要求 Coordinator 先理解研究结果，再生成实现规格。这一步能显著减少“研究很多，但落实很差”的问题。

### 3. 并行能力被充分利用

设计明确鼓励把研究类任务并行展开，让系统在定位问题时更快覆盖多个角度。

### 4. 对上下文污染有防护

通过区分 `continue` 和 `spawn fresh`，它避免错误思路、冗余探索和历史噪音持续污染后续任务。

### 5. 验证标准更像真实工程

它强调独立验证、检查 feature 开启状态、追查 typecheck 和测试失败原因，这比“跑一下测试”更接近真实团队流程。

### 6. 失败恢复成本低

当 worker 失败时，优先继续同一个 worker，可以复用它已有的错误上下文，减少重复解释。

### 7. 对用户沟通更稳定

它明确规定内部通知不是对话对象，Coordinator 只向用户汇报可靠进展，不编造结果，也不提前预测结论。

## 一句话总结

这套 Coordinator 设计的本质，不是“多开几个 agent”，而是用一个负责综合判断的中枢，把并行研究、精确委派、独立验证串成一个可控的软件工程闭环。
