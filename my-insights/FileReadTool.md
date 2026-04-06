# FileReadTool 实现分析

## 1. 入口与定位

- 主实现：`src/tools/FileReadTool/FileReadTool.ts`
- 提示词与工具名：`src/tools/FileReadTool/prompt.ts`
- 默认限制：`src/tools/FileReadTool/limits.ts`
- UI 展示：`src/tools/FileReadTool/UI.tsx`
- 文本分段读取：`src/utils/readFileInRange.ts`
- PDF 处理：`src/utils/pdf.ts`、`src/utils/pdfUtils.ts`
- Notebook 处理：`src/utils/notebook.ts`
- 读文件状态缓存：`src/utils/fileStateCache.ts`

`FileReadTool` 在 `src/tools.ts` 里注册，实际对外工具名是 `Read`。它的职责不只是“读文本文件”，还统一承载了：

- 普通文本文件读取
- 图片读取
- PDF 读取 / PDF 按页抽取
- Jupyter Notebook 读取
- 会话级读文件缓存与去重
- 与权限系统、附件系统、compact、后续编辑工具之间的衔接

## 2. 整体结构

实现可以分成四层：

1. `buildTool({...})` 定义工具元信息、schema、权限检查、UI、结果映射。
2. `validateInput()` 做“尽量不碰磁盘”的前置校验。
3. `call()` 做主流程编排：限制读取大小、去重、技能触发、错误兜底。
4. `callInner()` 按文件类型真正执行读取。

也就是说，这个工具不是一个简单的 `fs.readFile()` 包装，而是一个“文件读取编排器”。

## 3. 输入与输出协议

### 输入

输入 schema 很简单：

- `file_path: string`
- `offset?: number`
- `limit?: number`
- `pages?: string`

其中：

- `file_path` 要求是绝对路径
- `offset` / `limit` 面向文本文件，按“行”读取
- `pages` 面向 PDF，格式如 `"3"`、`"1-5"`、`"3-"`

### 输出

输出是一个按 `type` 区分的 discriminated union：

- `text`
- `image`
- `notebook`
- `pdf`
- `parts`
- `file_unchanged`

这很关键，因为 `Read` 并不总是返回字符串，而是按内容类型走不同序列化策略。

## 4. buildTool 层做了什么

`FileReadTool` 在 `buildTool()` 里声明了这些核心属性：

- `name: FILE_READ_TOOL_NAME`，也就是 `Read`
- `searchHint: 'read files, images, PDFs, notebooks'`
- `strict: true`
- `isConcurrencySafe() => true`
- `isReadOnly() => true`
- `isSearchOrReadCommand() => { isSearch: false, isRead: true }`

另外几个很重要的点：

- `backfillObservableInput()` 会先对 `file_path` 做 `expandPath()`，避免 `~`、相对路径、Windows 分隔符等绕过权限或观察逻辑。
- `checkPermissions()` 通过 `checkReadPermissionForTool()` 接入统一的文件系统权限系统。
- `preparePermissionMatcher()` 返回路径匹配函数，供 permission rule 使用。
- `extractSearchText()` 故意返回空串，因为 UI 不展示文件内容，只展示“读了什么”的摘要。

## 5. validateInput：前置校验策略

`validateInput()` 的特点是：优先做“纯字符串 / 路径级”判断，尽量在真正触盘前先挡掉危险输入。

它依次做了这些事：

### 5.1 PDF 页码参数校验

如果传了 `pages`，先用 `parsePDFPageRange()` 校验格式，并限制单次最多 `PDF_MAX_PAGES_PER_READ` 页。

### 5.2 deny 规则检查

将路径做 `expandPath()` 后，用 `matchingRuleForInput(..., 'read', 'deny')` 提前检查是否被权限规则禁止。

### 5.3 UNC 路径特殊处理

如果是 Windows UNC 路径（如 `\\\\server\\share` 或 `//server/share`），这里只返回 `result: true`，把真正的文件系统访问延后到权限确认之后，避免提前触发 NTLM 凭据泄露风险。

### 5.4 二进制扩展名拦截

如果扩展名被判断为二进制文件，则拒绝读取；但有两类明确例外：

- PDF
- 图片

换句话说，它不是禁止“所有非文本”，而是只允许自己明确支持的特殊格式。

### 5.5 特殊设备文件拦截

通过 `BLOCKED_DEVICE_PATHS` 拦掉可能导致无限输出或阻塞的设备文件，例如：

- `/dev/zero`
- `/dev/random`
- `/dev/stdin`
- `/dev/tty`
- `/dev/fd/0`
- `/proc/<pid>/fd/0-2`

这是一个非常典型的“CLI 工具级安全防御”。

## 6. call：主流程编排

`call()` 本身不直接读文件，它主要负责 orchestration。

### 6.1 读取限制

它从 `context.fileReadingLimits` 或 `getDefaultFileReadingLimits()` 里得到两个核心限制：

- `maxSizeBytes`
- `maxTokens`

默认值在 `limits.ts`：

- `maxSizeBytes = 256KB`（来自 `MAX_OUTPUT_SIZE`）
- `maxTokens = 25000`

`maxTokens` 的优先级是：

- 环境变量 `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS`
- GrowthBook 配置
- 硬编码默认值

而 `call()` 还允许 `ToolUseContext` 再覆盖一层。

### 6.2 去重：readFileState

这是 `FileReadTool` 最重要的实现细节之一。

`context.readFileState` 是一个 LRU cache，记录最近读过的文件内容与时间戳。`call()` 会尝试做如下优化：

- 如果之前已经读过同一个文件
- 不是 `isPartialView`
- 之前那次是 `Read` 写入的缓存（`offset !== undefined`）
- 本次 `offset` / `limit` 与上次完全一致
- 且当前磁盘 `mtime` 没变

那么直接返回：

- `type: 'file_unchanged'`

而不是重新把整段内容塞进模型上下文。

这个设计的目的很明确：降低重复文件内容造成的 token 浪费。也因此：

- 图片和 PDF 不参与这套 dedup
- `FileEditTool` / `FileWriteTool` 写入 `readFileState` 的条目不会误触发 dedup，因为它们写入时 `offset` 是 `undefined`

### 6.3 技能发现触发

如果不是 simple mode，`call()` 会基于文件路径触发：

- `discoverSkillDirsForPaths()`
- `addSkillDirectories()`
- `activateConditionalSkillsForPaths()`

这说明 `Read` 不只是 I/O 工具，它还承担了“用户触达某些文件后动态发现技能”的副作用入口。

### 6.4 ENOENT 的友好错误处理

`call()` 外层包了一层 `try/catch`，专门对 `ENOENT` 做增强：

- macOS 截图文件名里可能有普通空格 / 窄空格，尝试替代路径
- `findSimilarFile()`：同目录下同名异后缀建议
- `suggestPathUnderCwd()`：补全“漏掉 repo 目录”的路径猜测

所以最终用户拿到的不是原始 `ENOENT`，而是更接近交互产品的错误文案。

## 7. callInner：按类型分支读取

`callInner()` 才是实际的文件读取执行层。

### 7.1 Notebook 分支

条件：`ext === 'ipynb'`

处理流程：

1. `readNotebook(resolvedFilePath)` 解析 notebook
2. 将 cells 序列化成 JSON 文本
3. 先检查字节大小是否超过 `maxSizeBytes`
4. 再用 `validateContentTokens()` 检查 token 数
5. 写入 `readFileState`
6. 返回 `type: 'notebook'`

这里有两个值得注意的点：

- Notebook 的“模型侧结果”不是整份原始 `.ipynb` JSON，而是经过 `readNotebook()` 结构化后的 cell 列表。
- `mapToolResultToToolResultBlockParam()` 对 notebook 又会走 `mapNotebookCellsToToolResult()`，把 cell 内容与输出重新拼成适合模型消费的 text/image blocks。

换句话说，Notebook 读取是强语义化处理，不是原样透传。

### 7.2 图片分支

条件：扩展名属于 `png/jpg/jpeg/gif/webp`

处理函数是 `readImageWithTokenBudget()`，核心思想是“一次读入，多级压缩回退”：

1. 用 `getFsImplementation().readFileBytes()` 读取一次原始 buffer
2. 检测真实图片格式
3. 先尝试标准 resize/downsample
4. 估算 base64 后的 token 数
5. 如果超预算，再走更激进的压缩 `compressImageBufferWithTokenLimit()`
6. 再失败，就 fallback 到 `sharp(...).resize(400, 400).jpeg({ quality: 20 })`
7. 再失败，最后才退回原图

这个实现有几个优点：

- 避免重复读盘
- 优先保留视觉信息，再考虑 token 成本
- 尽量把“图片读入模型上下文”控制在 token budget 内

此外，图片读取后还会额外发一条 `meta` user message，内容来自 `createImageMetadataText()`，用于补充尺寸信息，便于后续坐标映射或视觉理解。

### 7.3 PDF 分支

条件：`isPDFExtension(ext)`

PDF 是这里最复杂的分支，分两种模式。

#### 模式 A：传了 `pages`

流程是：

1. `parsePDFPageRange()`
2. `extractPDFPages()` 调 `pdftoppm` 把页转成 JPEG
3. 返回 `type: 'parts'`
4. 再把抽出来的每页 JPG 读出来、缩放后，作为 image blocks 放进 `newMessages`

也就是说，按页读取 PDF 时，模型真正看到的是“页图片”而不是 PDF document block。

#### 模式 B：整份读取

流程是：

1. 先尝试 `getPDFPageCount()`
2. 如果页数超过阈值 `PDF_AT_MENTION_INLINE_THRESHOLD`，拒绝整份读取，要求必须传 `pages`
3. 如果当前模型不支持 PDF，直接报错并提示升级模型 / 安装 poppler
4. 否则调用 `readPDF()`，把 PDF 整体 base64 化
5. 返回 `type: 'pdf'`
6. 再通过 `newMessages` 给模型补一条 `document` block

这里可以看出作者对 PDF 做了“双轨处理”：

- 支持文档块的模型：直接喂 PDF document
- 不支持或太大的情况：转页图

### 7.4 文本分支

剩余情况都走文本分支。

流程：

1. 把用户的 1-based `offset` 转成内部 0-based `lineOffset`
2. 调 `readFileInRange()`
3. `validateContentTokens()`
4. 写入 `readFileState`
5. 触发 `fileReadListeners`
6. 返回 `type: 'text'`

其中 `readFileInRange()` 自己又分成两条路径：

- 小于 10MB 的普通文件：一次性 `readFile()` 后内存切行，走 fast path
- 大文件 / 非 regular file：`createReadStream()` 流式扫描换行，走 streaming path

这个实现的价值在于：

- 小文件读得快
- 大文件不会因为只想读某几行就把整份内容全堆进内存
- 还能统计 `totalLines`、`readBytes`、`mtimeMs`

## 8. 文本结果是怎么发给模型的

文本分支的 `Output` 只是原始内容和行号元数据。真正发给模型时，走的是 `mapToolResultToToolResultBlockParam()`。

它会做这些事：

### 8.1 行号格式化

调用 `formatFileLines()`，底层是 `addLineNumbers()`。

当前默认是紧凑格式：

```text
12\tconst x = 1
13\treturn x
```

如果 killswitch 打开，也可能退回旧的箭头格式。

### 8.2 自动记忆文件 freshness 提示

如果读的是 auto memory 文件，会通过 `memoryFileMtimes` 这个 `WeakMap` 在映射阶段插入 `memoryFreshnessNote()`。

注意这里没有把 “freshness” 直接放进 schema，而是用 side-channel 传递，只服务展示层和模型侧序列化。

### 8.3 网络安全提醒

大多数模型下，文本结果尾部还会追加一段 `CYBER_RISK_MITIGATION_REMINDER`：

- 可以分析 malware
- 但不能帮助增强或改造恶意代码

只有 `claude-opus-4-6` 被放在 exemption 列表里。

### 8.4 空文件 / offset 越界提示

如果内容为空，不会直接返回空字符串，而是生成系统提醒：

- 文件存在但为空
- 或者文件行数小于请求 offset

这能避免模型把“空内容”误认为工具异常。

## 9. UI 层与模型层是分离的

`src/tools/FileReadTool/UI.tsx` 的设计很克制：UI 只显示摘要，不显示具体内容。

例如：

- 文本：`Read 30 lines`
- 图片：`Read image (42KB)`
- PDF：`Read PDF (1.4MB)`
- 未变化：`Unchanged since last read`

这和模型侧完全不同。模型侧拿到的是：

- 文本正文 + 行号 + reminder
- 图片 block
- notebook 的多 block 内容
- PDF 的 document block 或页图片

所以 `FileReadTool` 明确把“给用户看的东西”和“给模型吃的东西”拆开了。

## 10. 与其他系统的关系

### 10.1 与 readFileState 的关系

`readFileState` 不只是给 `Read` 自己用，后续很多模块都依赖它：

- `FileEditTool` / `FileWriteTool` 会检查文件是否先被读取过
- compact 会把已读文件信息纳入上下文保留 / 恢复逻辑
- attachments / nested memory 会利用它做去重
- forked agent 会克隆这份缓存

所以 `Read` 实际上在维护“模型已经看过哪些文件”的会话事实。

### 10.2 与 analytics 的关系

它埋了不少事件：

- `tengu_file_read_limits_override`
- `tengu_file_read_dedup`
- `tengu_pdf_page_extraction`
- `tengu_session_file_read`

说明这个工具在产品层面是被重点观测的高频工具。

### 10.3 与 skill discovery 的关系

读文件时会触发动态技能目录发现与激活，这一点很容易在初看源码时忽略。

## 11. 设计亮点

我觉得这份实现有几个很鲜明的特点：

### 11.1 安全边界比较完整

- deny rule 提前校验
- UNC 延迟访问
- 设备文件黑名单
- 二进制类型白名单

这不是“读文件功能”，而是“可放进 agent 环境里的读文件功能”。

### 11.2 把 token 成本当一等公民

- 文本有 `maxSizeBytes + maxTokens` 双限制
- 图片有 token budget 驱动的压缩
- 重复读取有 dedup
- 行号格式还有紧凑开关

### 11.3 文件类型处理高度特化

- Notebook 不是文本读
- PDF 不是统一按文本处理
- 图片不是简单 base64 透传

这说明 `Read` 的目标不是“返回原始 bytes”，而是“为模型构造最合适的上下文材料”。

### 11.4 UI 与模型序列化分层清晰

用户界面只展示摘要，真正内容只进模型上下文。这使终端 UI 不会因为大文件而变得不可读。

## 12. 几个值得注意的细节

### 12.1 Prompt 写着“默认最多读 2000 行”，实现里并没有硬编码这个上限

`prompt.ts` 里有 `MAX_LINES_TO_READ = 2000`，并写进了提示词；但 `FileReadTool.call()` 自身默认是：

- `offset = 1`
- `limit = undefined`

真正限制默认输出的主要是：

- 字节上限 `maxSizeBytes`
- token 上限 `maxTokens`

`2000` 这个值目前主要还出现在附件读取等外围逻辑里，而不是 `FileReadTool` 的核心文本路径里直接强制执行。

这意味着“提示词契约”和“底层实现”之间存在一个轻微不完全一致。

### 12.2 dedup 只针对文本 / notebook 一类可缓存内容

图片和 PDF 不走 `file_unchanged` 逻辑，这样能避免多模态对象缓存语义过于复杂。

### 12.3 内部规范化路径与对外返回路径并不完全一致

内部很多操作基于 `resolvedFilePath` / `fullFilePath`，但不同分支对外返回的路径不完全一样：文本和 notebook 更偏向保留原请求路径，PDF/图片分支则更偏向返回处理后的结果对象。这说明它的“内部执行路径”和“外部展示路径”是刻意解耦的。

### 12.4 文本读取对大文件做了专门优化

`readFileInRange()` 的 streaming 路径只累计命中的行，不累计范围外内容，这对超大文件非常重要。

## 13. 一句话总结

`FileReadTool` 本质上不是一个“读取文件内容”的小工具，而是 Claude Code 里负责把本地文件安全、经济、结构化地转换成模型上下文的核心基础设施。它同时承担了权限前置校验、类型分流、token 控制、会话去重、附件/记忆衔接和 UI 摘要展示等多重职责，是整个工具链里非常关键的一层。
