# BashTool 实现分析

## 1. BashTool 在系统里的位置

`BashTool` 是 CLI 里最核心的工具之一，负责执行 shell 命令。它在 [src/tools.ts](src/tools.ts) 中注册进全局工具列表，真正实现位于 [src/tools/BashTool/BashTool.tsx](src/tools/BashTool/BashTool.tsx)。

从调用链上看，大致是：

1. 模型产出一个 `Bash` 工具调用。
2. 工具框架根据 `inputSchema` 解析参数。
3. `checkPermissions()` 调用 `bashToolHasPermission()` 做权限判定。
4. 若允许执行，`call()` 进入 `runShellCommand()`。
5. 命令输出被整理成 UI 展示格式，以及返回给模型的 `tool_result`。

`BashTool` 并不是“把字符串交给 shell 执行”这么简单，而是把“命令执行”拆成了：

- 输入校验
- 只读/并发判断
- 权限规则匹配
- AST / legacy 安全检查
- 沙箱决策
- 前台 / 后台任务管理
- 输出裁剪、持久化、图片识别、语义化返回

## 2. 输入与输出模型

### 输入

输入 schema 定义在 `BashTool.tsx` 顶部，核心字段有：

- `command`: 要执行的 shell 命令
- `timeout`: 超时时间，默认来自 [src/tools/BashTool/prompt.ts](src/tools/BashTool/prompt.ts)
- `description`: 给用户看的简要说明
- `run_in_background`: 是否后台运行
- `dangerouslyDisableSandbox`: 是否强制关闭沙箱

另有一个内部字段：

- `_simulatedSedEdit`: 仅供权限弹窗确认后的 sed 预览回写使用，不暴露给模型

这一点很重要：作者专门把 `_simulatedSedEdit` 从模型可见 schema 里剔除了，避免模型借“看起来无害的 bash 命令”绕过 sed 编辑权限检查，直接落任意文件内容。

### 输出

输出 schema 也不只是 `stdout/stderr`：

- `stdout`
- `stderr`
- `interrupted`
- `isImage`
- `backgroundTaskId`
- `assistantAutoBackgrounded`
- `returnCodeInterpretation`
- `noOutputExpected`
- `persistedOutputPath`
- `persistedOutputSize`
- `structuredContent`

这里最容易误读的一点是：实际 shell 层的 `stderr` 基本被并进了 `stdout`。`stderr` 字段在最终 `Out` 中更多只承载 BashTool 自己补的附加提示，例如“cwd 被重置”。

## 3. Tool 对象本身做了什么

`BashTool` 通过 `buildTool()` 构造，核心能力有：

- `description()` / `prompt()`：告诉模型怎么使用 BashTool
- `isConcurrencySafe()`：只有只读命令才被视为并发安全
- `isReadOnly()`：依赖 [src/tools/BashTool/readOnlyValidation.ts](src/tools/BashTool/readOnlyValidation.ts)
- `validateInput()`：例如阻止长时间 `sleep N`
- `checkPermissions()`：进入 `bashToolHasPermission()`
- `call()`：真正执行命令
- `mapToolResultToToolResultBlockParam()`：把执行结果转换成模型能消费的 `tool_result`

它还实现了几类 UI 相关方法：

- `renderToolUseMessage`
- `renderToolUseProgressMessage`
- `renderToolUseQueuedMessage`
- `renderToolResultMessage`
- `renderToolUseErrorMessage`

这说明 BashTool 的实现不是纯后端逻辑，而是“执行层 + REPL 展示层”绑在一个工具对象里。

## 4. 只读判定与并发策略

只读判断在 [src/tools/BashTool/readOnlyValidation.ts](src/tools/BashTool/readOnlyValidation.ts)。

它做的事情比“命令名是不是 `cat/ls`”复杂很多：

- 有大量命令级 allowlist
- 对 flag 做安全校验
- 对 `git`、`gh`、`rg`、`fd`、`sed` 等命令做细粒度限制
- 识别路径类参数
- 防御 `--` 之后以 `-` 开头的真实路径
- 防御 `cd + git`、危险 `rm/rmdir` 路径等情况

`BashTool.isConcurrencySafe()` 直接依赖 `isReadOnly()`，因此这个工具的并发语义非常保守：只有明确只读时才允许并发安全执行。

## 5. 权限检查是 BashTool 的核心

权限检查的主入口是 [src/tools/BashTool/bashPermissions.ts](src/tools/BashTool/bashPermissions.ts) 里的 `bashToolHasPermission()`。这是整个 BashTool 最复杂、也最值得看的部分。

### 5.1 总体思路

它不是“一次 if/else”，而是多阶段决策：

1. 先做 AST 级 bash 安全解析
2. 再做 legacy 兼容解析
3. 看是否能被 sandbox auto-allow
4. 先查 exact deny/ask/allow 规则
5. 再跑 classifier 的 deny/ask 规则
6. 再处理管道/重定向/compound command
7. 再按 subcommand 逐个做权限判断
8. 汇总建议规则和最终 prompt 行为

### 5.2 AST 优先，legacy 兜底

作者希望优先使用 tree-sitter 解析 bash：

- `parseCommandRaw()`
- `parseForSecurityFromAst()`
- `checkSemantics()`

如果 AST 结果是：

- `simple`：说明命令结构足够清晰，可以继续细分 subcommand
- `too-complex`：直接要求用户确认
- `parse-unavailable`：退回 legacy 流程

这层设计的目标很明确：尽量不要用纯字符串 split 去理解复杂 shell，因为 bash 的 quoting、substitution、operator 很容易让字符串规则失真。

### 5.3 exact / prefix / wildcard 规则

权限规则匹配分成几类：

- exact match
- prefix match
- wildcard match

而且 deny / ask / allow 的优先级是清晰的：

1. deny
2. ask
3. allow

代码里还处理了很多安全细节：

- 允许剥离“安全 env var 前缀”再匹配规则
- 允许剥离 `timeout` / `nohup` / `nice` 等 wrapper 再匹配
- deny/ask 的 env var stripping 比 allow 更激进，避免绕过拒绝规则
- prefix/wildcard 默认不允许匹配 compound command，防止 `Bash(cd:*)` 误放行 `cd x && rm -rf y`

### 5.4 subcommand 级别判定

复杂命令最终会被拆成多个 subcommand，再分别走 `bashToolCheckPermission()` / `checkCommandAndSuggestRules()`。

这一层做的事情包括：

- 检查每个子命令是否被 deny
- 检查每个子命令是否需要 ask
- 单独校验原始命令上的输出重定向路径
- 汇总建议规则

这也是为什么它能对类似下面的命令做更细的处理：

```bash
cd src && npm test | tee output.log
```

而不是把整串命令粗暴地视为一个不可理解的字符串。

### 5.5 classifier 是“附加自动批准层”

`bashPermissions.ts` 里集成了 Bash classifier：

- deny classifier
- ask classifier
- allow classifier
- speculative classifier check

它的作用不是替代所有规则，而是在“本来要弹确认框”的情况下，尝试基于 prompt rule 做高置信度自动放行。

不过按仓库根目录的 `AGENTS.md` 描述，这个 reverse-engineered 版本里 `feature()` 在入口被 polyfill 成恒为 `false`。这意味着像 `feature('BASH_CLASSIFIER')`、`feature('TREE_SITTER_BASH_SHADOW')`、`feature('MONITOR_TOOL')`、`feature('KAIROS')` 之类的分支，在当前构建里大概率都不会启用。

所以：

- 这些代码仍然存在
- 设计意图值得分析
- 但在当前仓库运行态里，很多属于“保留下来的未启用逻辑”

## 6. 路径校验与危险命令防护

[src/tools/BashTool/pathValidation.ts](src/tools/BashTool/pathValidation.ts) 负责从命令参数中提取路径并校验。

它覆盖的场景很多：

- `cd`
- `ls`
- `find`
- `mkdir`
- `touch`
- `rm`
- `mv`
- `cp`
- `cat/head/tail`
- `grep/rg`
- `sed`
- `git`

这个文件的价值不只是“解析路径”，而是有很多安全修补：

- 处理 POSIX `--`
- 处理 `find -- -path`
- 处理看起来像 flag、但其实是路径的参数
- 对危险删除目标做强制 ask
- 校验输出重定向的目标路径

也就是说，哪怕子命令本身是允许的，像 `> /etc/passwd` 这样的重定向目标仍然会被单独检查。

## 7. 沙箱策略

沙箱决策在 [src/tools/BashTool/shouldUseSandbox.ts](src/tools/BashTool/shouldUseSandbox.ts)。

逻辑大致是：

1. 全局没启沙箱，就不使用
2. 用户显式要求 `dangerouslyDisableSandbox` 且策略允许，则不使用
3. 命令命中 `sandbox.excludedCommands`，则不使用
4. 否则默认使用

它支持对 compound command 逐段检查 excluded command，还会在匹配时剥掉部分 env var / wrapper，避免因为前缀包装导致排除规则失效。

## 8. 执行层：真正跑命令的过程

真正执行发生在 `call()` 和 `runShellCommand()`。

### 8.1 特殊分支：模拟 sed 编辑

如果输入带 `_simulatedSedEdit`，就不会真的跑 `sed`，而是直接走 `applySedEdit()`：

- 读取原文件
- 写入预览确认后的新内容
- 更新 file history
- 通知 VS Code 文件发生变化
- 更新 read cache

这套设计保证“用户在权限弹窗里看到的 diff”与“最终写入内容”一致。

### 8.2 正常执行

正常路径会调用 [src/utils/Shell.ts](src/utils/Shell.ts) 的 `exec()`，后者再去：

- 选择 shell provider
- 拼装命令
- 决定是否套 sandbox wrapper
- 创建 `ShellCommand`

`runShellCommand()` 本身是一个 async generator，它一边等待命令完成，一边按进度 yield：

- 最近输出
- 全量输出
- 行数 / 字节数
- 已运行秒数

这样 REPL 可以实时展示 shell 进度。

### 8.3 后台任务

`BashTool` 支持三类“后台化”：

- 用户显式传 `run_in_background`
- 超时后的 auto-background
- assistant mode 的 blocking budget 超时后自动后台化

相关能力依赖：

- `spawnShellTask`
- `registerForeground`
- `backgroundExistingForegroundTask`
- `TaskOutput.startPolling()`

不过同样需要注意：这个仓库里和 `KAIROS`、`MONITOR_TOOL` 相关的 feature flag 分支按当前 polyfill 多半不会触发。

## 9. 输出处理

输出处理是 BashTool 另一个很成熟的部分。

### 9.1 退出码语义化

[src/tools/BashTool/commandSemantics.ts](src/tools/BashTool/commandSemantics.ts) 不是简单地把非 0 当错误，而是按命令解释：

- `grep/rg`: `1` 表示没匹配到，不算错误
- `find`: `1` 表示部分目录不可访问
- `diff`: `1` 表示文件不同
- `test` / `[`：`1` 表示条件为假

这让 BashTool 返回给模型的结果更贴近真实语义。

### 9.2 大输出持久化

如果输出很大，BashTool 会：

- 保留一份截断预览给当前轮次
- 把完整输出复制到 `tool-results` 目录
- 在 `tool_result` 中放一个 `<persisted-output>` 风格的提示，让模型后续用 `FileRead` 去看

这避免了一次工具调用把上下文窗口塞爆。

### 9.3 图片输出

如果 stdout 是 `data:image/...;base64,...`：

- UI 会把它标记为图片输出
- 返回给模型时会构造成 image content block
- 还会在必要时重新读取落盘内容并做 resize/downsample

这意味着 BashTool 理论上可以承载一些“命令生成图片”的场景。

## 10. UI 与权限弹窗

### 10.1 执行态 UI

[src/tools/BashTool/UI.tsx](src/tools/BashTool/UI.tsx) 负责：

- 缩略显示命令
- 渲染 progress
- 显示排队中 / 运行中状态
- 显示后台快捷键提示

[src/tools/BashTool/BashToolResultMessage.tsx](src/tools/BashTool/BashToolResultMessage.tsx) 负责最终结果展示：

- stdout/stderr 渲染
- cwd reset 警告拆出显示
- 无输出时显示 `Done` / `(No output)`
- 后台任务显示 `Running in the background`

### 10.2 权限弹窗

[src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx](src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx) 是 BashTool 权限体验的重要组成部分。

它做了几件事：

- 对 sed in-place 编辑走专门的 diff 审批
- 显示 destructive warning
- 展示/编辑建议的 permission rule
- 支持 classifier 自动批准中的 UI 状态
- 支持用户把当前批准转成可持久化规则

其中 destructive warning 来自 [src/tools/BashTool/destructiveCommandWarning.ts](src/tools/BashTool/destructiveCommandWarning.ts)，但这只是信息提示，不参与真正的权限决策。

## 11. 测试透露出的设计重点

现有测试不多，但很能说明作者关注点：

- [src/tools/BashTool/__tests__/commandSemantics.test.ts](src/tools/BashTool/__tests__/commandSemantics.test.ts)
  - 验证退出码语义化解释
- [src/tools/BashTool/__tests__/destructiveCommandWarning.test.ts](src/tools/BashTool/__tests__/destructiveCommandWarning.test.ts)
  - 验证权限弹窗里的破坏性提示

从测试覆盖面可以看出，这个仓库当前更偏“恢复关键行为”，而不是把 BashTool 全部细节都做完整回归。

## 12. 对当前实现的总结

如果只用一句话概括：

`BashTool` 的本质是“带权限系统、沙箱系统、任务系统和 UI 渲染能力的 shell 执行框架”。

它的几个关键特点是：

- 工具对象职责很重，前后端逻辑都在一起
- 权限检查远比执行逻辑复杂
- 安全策略主要围绕 shell 解析不确定性、规则匹配绕过、路径写入风险展开
- 输出处理做得比较成熟，考虑了大输出、图片、后台任务和非标准退出码
- 在当前 reverse-engineered 版本里，很多 feature-flag 分支存在但不会真正启用

## 13. 我认为最值得关注的几个实现点

如果后续还要继续研究或裁剪 BashTool，我会优先看这几块：

1. `bashToolHasPermission()`
   这是整个 BashTool 的核心复杂度来源。
2. `readOnlyValidation.ts`
   它决定了什么命令可以无感放行、并发执行。
3. `pathValidation.ts`
   它是文件系统安全边界的重要组成部分。
4. `runShellCommand()`
   它把 shell 执行、进度轮询、后台任务粘合在一起。
5. `BashPermissionRequest.tsx`
   它决定用户最终看到怎样的权限体验。

## 14. 结合这个仓库现状的结论

结合仓库根目录 `AGENTS.md` 里的说明，这个反编译版本分析 BashTool 时有一个必须带上的前提：

- 代码里保留了很多 Anthropic 内部/feature-flag 分支
- 但当前构建下 `feature()` 被入口 polyfill 为恒 `false`

因此，分析时最好分清两层：

- “源码设计上 BashTool 支持什么”
- “当前这个仓库实际运行时会走到什么”

前者决定我们如何理解原始架构，后者决定我们后续要不要裁剪死代码、或者恢复某些能力。
