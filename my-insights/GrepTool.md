# GrepTool 实现分析

## 1. 概览

`GrepTool` 是这个仓库里专门负责“按正则搜索文件内容”的工具，底层能力来自 `ripgrep`，主实现位于：

- `src/tools/GrepTool/GrepTool.ts`
- `src/tools/GrepTool/prompt.ts`
- `src/tools/GrepTool/UI.tsx`
- `src/utils/ripgrep.ts`

它不是简单地把 `rg` 包一层，而是把以下几类逻辑统一封装进了工具系统：

- 输入 schema 和参数语义
- 路径校验
- 文件系统读取权限检查
- ripgrep 参数拼装
- 结果裁剪、分页与相对路径化
- UI 展示和 transcript 搜索文本提取

在工具注册层，`GrepTool` 会和 `GlobTool` 一起加入基础工具集；但如果运行环境已经提供了内嵌搜索工具，则这两个专用工具会被跳过：

- `src/tools.ts`

## 2. 工具定义层

`GrepTool` 通过 `buildTool({...})` 构建，属于标准 `Tool` 体系的一员。它在定义层声明了几件关键事情：

- `name: 'Grep'`
- `searchHint: 'search file contents with regex (ripgrep)'`
- `maxResultSizeChars: 20_000`
- `strict: true`
- `isConcurrencySafe(): true`
- `isReadOnly(): true`
- `isSearchOrReadCommand(): { isSearch: true, isRead: false }`

这说明系统把它视为：

- 可并发执行的工具
- 只读工具
- 搜索型工具
- 有 20KB 的结果持久化阈值

`prompt.ts` 里对模型的约束也比较明确：

- 搜索任务“必须优先使用 `GrepTool`”
- 不要通过 `BashTool` 直接调用 `grep` / `rg`
- 支持 ripgrep 正则
- 多轮开放式检索应交给 `AgentTool`

这部分不是运行时逻辑，但会直接影响模型是否正确选用该工具。

## 3. 输入 schema 设计

`inputSchema` 使用 `zod` 定义，支持的核心参数有：

- `pattern`: 必填，ripgrep 正则
- `path`: 可选，搜索目录或单文件，默认当前工作目录
- `glob`: 可选，文件过滤模式
- `output_mode`: 可选，`content | files_with_matches | count`
- `-B` / `-A` / `-C` / `context`: 上下文行数
- `-n`: 是否显示行号，默认 `true`
- `-i`: 是否忽略大小写
- `type`: ripgrep 的 `--type`
- `head_limit`: 结果条数上限
- `offset`: 分页偏移
- `multiline`: 是否开启跨行匹配

几个实现细节值得注意：

### 3.1 默认输出模式

如果不传 `output_mode`，默认是 `files_with_matches`，也就是只返回命中的文件路径，不返回内容。

### 3.2 默认分页上限

如果不显式传 `head_limit`，工具会使用内部默认值 `250`。这不是 ripgrep 的限制，而是 `GrepTool` 自己为控制上下文膨胀加的一层裁剪。

特殊约定：

- `head_limit = 0` 表示不限制
- `offset` 用于跳过前 N 条结果

### 3.3 语义化布尔/数字

一些字段通过 `semanticBoolean` / `semanticNumber` 包装，这意味着工具层接受的输入比严格 JSON 更宽松，利于模型生成参数时保持稳定。

## 4. 执行前的校验与权限

### 4.1 路径存在性校验

`validateInput()` 只在显式传入 `path` 时执行：

1. 调用 `expandPath(path)` 转成绝对路径
2. 对 UNC 路径（`\\\\` 或 `//`）直接跳过文件系统探测
3. 对普通路径执行 `stat`
4. 如果不存在，返回带建议路径的错误信息

这里的意图很明确：

- 避免无意义搜索
- 给模型更友好的纠错提示
- 避免在 UNC 路径上做危险的文件系统访问

### 4.2 权限检查

`checkPermissions()` 最终走的是：

- `checkReadPermissionForTool(GrepTool, input, toolPermissionContext)`

权限检查依赖 `getPath()`：

- 有 `path` 时用传入路径
- 没有 `path` 时退回 `getCwd()`

也就是说，`GrepTool` 的权限语义是“读取某个路径下的内容”。

在 `filesystem.ts` 里，`checkReadPermissionForTool()` 会继续做：

1. UNC 路径拦截
2. 可疑 Windows 路径模式拦截
3. 读取 deny 规则检查
4. ask 规则检查
5. 其他 allow/继承规则判断

所以 `GrepTool` 的权限模型本质上和 `FileReadTool` 是同一套“读权限体系”，这也是权限 UI 里它和 `GlobTool`、`FileReadTool` 共用 `FilesystemPermissionRequest` 的原因。

### 4.3 权限规则与 pattern 匹配

`preparePermissionMatcher({ pattern })` 返回的是：

- 将权限规则里的 wildcard pattern 与用户输入的 `pattern` 做匹配

这说明权限系统不只关心路径，也可以对搜索模式本身做规则约束。

## 5. ripgrep 参数拼装

`call()` 是 `GrepTool` 的核心。它先把 `path` 展开为绝对路径，然后开始构建 ripgrep 参数。

### 5.1 固定参数

默认总会加上：

- `--hidden`
- `--max-columns 500`

并且自动排除版本控制目录：

- `.git`
- `.svn`
- `.hg`
- `.bzr`
- `.jj`
- `.sl`

这里有两个重要含义：

- 隐藏文件默认会被搜索
- 但 VCS 元数据目录会被主动排除，减少噪音

### 5.2 模式相关参数

根据输入追加：

- `multiline: true` -> `-U --multiline-dotall`
- `-i: true` -> `-i`
- `output_mode = files_with_matches` -> `-l`
- `output_mode = count` -> `-c`
- `output_mode = content && -n !== false` -> `-n`

上下文参数优先级是：

1. `context`
2. `-C`
3. `-B` / `-A`

### 5.3 pattern 的安全处理

如果 `pattern` 以 `-` 开头，不直接拼到参数列表，而是转成：

- `-e <pattern>`

这是为了避免 ripgrep 把搜索模式误判成命令行选项。

### 5.4 type 与 glob

`type` 直接映射到：

- `--type <type>`

`glob` 的处理稍微特别：

1. 先按空白切分
2. 对不带花括号的片段继续按逗号切分
3. 最终每个模式都拼成 `--glob <pattern>`

它试图兼容：

- `"*.js,*.ts"`
- `"*.{ts,tsx}"`

但这是一个相对“经验型”的拆分器，不是完整的 shell parser。

### 5.5 读取 deny 规则转成忽略模式

`GrepTool` 会读取文件系统权限中的“读 deny 规则”，再通过：

- `getFileReadIgnorePatterns()`
- `normalizePatternsToPath()`

把这些规则转成 ripgrep 的 `--glob !...` 排除项。

这是实现上非常关键的一层：  
即使 ripgrep 本身能扫到这些文件，`GrepTool` 也会尽量在搜索阶段就把被 deny 的内容隐藏掉，而不是等结果返回后再过滤。

### 5.6 插件缓存目录排除

另外还会调用：

- `getGlobExclusionsForPluginCache(absolutePath)`

用于屏蔽孤儿插件版本目录，避免搜索噪音。

## 6. ripgrep 执行层

真正执行搜索的是：

- `ripGrep(args, absolutePath, abortController.signal)`

对应实现位于 `src/utils/ripgrep.ts`。

这个封装比直接 `execFile('rg')` 更完整，主要做了几件事：

### 6.1 ripgrep 来源选择

运行时会优先尝试系统 `rg`；如果不可用，则退到项目内置版本；在 bundled 模式下还支持 embedded rg。

### 6.2 超时与缓冲区

默认超时：

- 普通平台 20 秒
- WSL 60 秒

最大缓冲区：

- 20MB

### 6.3 EAGAIN 自动重试

如果第一次执行出现资源不足类错误（如线程启动失败），会自动重试一次，并强制加上：

- `-j 1`

### 6.4 超时与部分结果

`ripGrep()` 并不总是“失败就抛错”：

- `exit code 1` 视为“没有匹配”，返回空数组
- 某些错误下如果已经拿到部分 stdout，会尽量返回部分结果
- 只有“超时且没有任何结果”时，才抛 `RipgrepTimeoutError`

这意味着 `GrepTool` 某些情况下可能拿到“部分结果但不是完整结果”。这是一个实用优先的设计。

## 7. 三种输出模式的后处理

### 7.1 `content`

`content` 模式下，ripgrep 的原始输出就是命中行文本。

后处理流程：

1. 先应用 `applyHeadLimit(results, head_limit, offset)`
2. 再把绝对路径转成相对路径
3. 拼成 `content` 字符串返回

输出对象大致是：

- `mode: 'content'`
- `numFiles: 0`
- `filenames: []`
- `content: string`
- `numLines: finalLines.length`
- 可选 `appliedLimit`
- 可选 `appliedOffset`

注意点：

- 它为了省 token，会把当前工作目录下的绝对路径转成相对路径
- `numFiles` 在这个模式下固定为 `0`，因为这里关心的是“行”，不是“文件”

### 7.2 `count`

`count` 模式下会给 ripgrep 加 `-c`，每行通常是：

- `文件路径:匹配次数`

后处理流程：

1. 先分页
2. 再把路径相对化
3. 解析每一行尾部的数字
4. 汇总出 `numMatches` 与 `numFiles`

这里有个实现语义需要特别说明：

- `numMatches` 和 `numFiles` 是基于“分页后的结果”计算出来的
- 如果命中文件很多，而 `head_limit` 截断了列表，汇总值并不是全局总数，而是当前页总数

这是当前实现的真实行为。

### 7.3 `files_with_matches`

这是默认模式，会给 ripgrep 加 `-l`，返回命中的文件路径列表。

后处理比前两种更复杂：

1. 对每个结果执行 `stat`
2. 取 `mtimeMs`
3. 按修改时间倒序排序
4. 修改时间相同时按文件名排序
5. 再应用 `head_limit` / `offset`
6. 最后把路径转成相对路径

所以默认模式不是“ripgrep 原始顺序”，而是“最近修改优先”。

这很适合代码探索场景，因为更可能先看到刚改过、也更相关的文件。

## 8. 分页与裁剪策略

分页逻辑由 `applyHeadLimit()` 完成，规则是：

- `limit === 0` -> 不限量
- `limit === undefined` -> 用默认值 `250`
- 只有真的发生截断时，才设置 `appliedLimit`

这个设计很细致，原因是：

- 工具结果里如果出现 `appliedLimit`，模型才能知道“后面还有结果，可以继续翻页”
- 如果没有截断，就不应该给模型造成“结果不完整”的假象

`formatLimitInfo()` 会把 `appliedLimit` / `appliedOffset` 格式化到最终 tool result 文本里，告诉模型当前结果是分页后的子集。

## 9. UI 与消息展示

`src/tools/GrepTool/UI.tsx` 负责展示层。

主要有三块：

### 9.1 工具使用消息

`renderToolUseMessage()` 会展示：

- `pattern`
- `path`（verbose 模式下显示完整路径，否则显示简化路径）

### 9.2 错误消息

`renderToolUseErrorMessage()` 会识别带 `tool_use_error` tag 的结果。

如果错误里包含“路径不存在”的提示，会把 UI 简化成：

- `File not found`

否则显示：

- `Error searching files`

### 9.3 结果消息

展示核心由 `SearchResultSummary` 组件完成：

- `content` 模式展示命中行数
- `count` 模式展示命中次数与文件数
- `files_with_matches` 模式展示文件数

在非 verbose 模式下，如果有结果，会出现 `Ctrl+O` 展开提示。

另外，`extractSearchText()` 会为 transcript 搜索提取可索引文本：

- `content` 模式索引 `content`
- 其他模式索引 `filenames.join('\n')`

## 10. 和系统其他模块的耦合点

### 10.1 工具注册

`src/tools.ts` 中，`GrepTool` 作为基础工具参与注册。

### 10.2 权限对话框

`src/components/permissions/PermissionRequest.tsx` 中，`GrepTool` 与：

- `GlobTool`
- `FileReadTool`

共享 `FilesystemPermissionRequest`。

### 10.3 Session 文件访问埋点

`src/utils/sessionFileAccessHooks.ts` 会解析 `GrepTool` 的输入：

- 看 `path`
- 看 `glob`

从而判断它是否访问了 session memory / transcript 文件，并记录埋点。

也就是说，`GrepTool` 不只是一个搜索工具，它还参与了会话级“记忆文件访问”观测链路。

## 11. 设计取舍与潜在问题

### 11.1 优点

- 基于 `ripgrep`，性能基础好
- 统一接入权限系统，而不是裸跑 shell
- 默认分页，能控制上下文膨胀
- 默认按最近修改时间排序，适合代码探索
- 自动屏蔽 VCS 目录、deny 规则文件和部分插件缓存噪音
- UI、tool result、transcript 索引三层行为比较一致

### 11.2 需要注意的行为

- `--hidden` 是默认开启的，因此搜索范围比很多人预期更广
- `files_with_matches` 模式返回的是“排序并截断后的文件列表”，不是 rg 原始顺序
- `count` 模式里的汇总值是“当前页汇总”，不是全局总计
- 默认 `head_limit=250`，如果模型没意识到这一点，可能误以为“只有这些结果”

### 11.3 一个明显的跨平台风险点

`content` 模式在相对路径化时，用的是“找到第一个冒号后切分”：

- `const colonIndex = line.indexOf(':')`

这在 Unix 路径上通常没问题，但在 Windows 上 ripgrep 输出可能是：

- `C:\path\file.ts:12:content`

此时第一个冒号来自盘符 `C:`，会导致路径截断错误。  
同文件里的 `count` 模式使用的是 `lastIndexOf(':')`，相对更稳健；`content` 模式这里存在潜在的 Windows 兼容性缺陷。

### 11.4 一个语义层面的限制

`files_with_matches` 模式返回的 `numFiles` 是“当前页文件数”，不是“所有命中文件总数”。  
如果以后要支持真正的总量统计，就需要在分页前单独保留一份总数。

## 12. 总结

`GrepTool` 的实现思路可以概括为：

1. 用 `zod` 定义一个适合模型调用的搜索接口
2. 把路径校验和读权限控制前置
3. 把 ripgrep 的复杂参数拼装收敛到工具层
4. 对结果做分页、排序、相对路径化和 UI 适配
5. 把搜索行为纳入整个 CLI 的权限、埋点和 transcript 体系

所以它的定位并不是“一个 rg wrapper”，而是“CLI 工具系统里的受控内容搜索能力”。  
如果后面要继续完善，我认为最值得优先关注的是两点：

- Windows 下 `content` 模式的路径解析
- `count` / `files_with_matches` 模式中“当前页统计”和“全量统计”的语义区分
