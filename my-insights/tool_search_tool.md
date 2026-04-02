# ToolSearchTool 工作原理

`tool_search_tool` 在这个仓库里对应的是 `ToolSearchTool`，实际工具名是 `ToolSearch`。它不是“搜索代码”的工具，而是“搜索其他可用工具定义”的元工具，用来按需加载 deferred tools 的 schema，避免在每次请求里把所有工具的完整定义都塞进 prompt。

## 它解决的问题

当可用工具很多时，尤其是有大量 MCP 工具时，如果把所有工具的 description 和 input schema 都直接发送给模型，会带来明显的上下文开销。`ToolSearchTool` 的方案是：

- 普通常用工具直接暴露给模型
- MCP 工具和低频工具先延迟加载
- 模型需要时，先调用 `ToolSearch`
- `ToolSearch` 返回 `tool_reference`
- 下一轮请求再把这些工具的完整 schema 发送给模型

这样既保留了大量工具的可发现性，也减少了 token 消耗。

## 关键入口

- 工具实现：`src/tools/ToolSearchTool/ToolSearchTool.ts`
- deferred 规则与提示词：`src/tools/ToolSearchTool/prompt.ts`
- 启用判断与 discovered tools 提取：`src/utils/toolSearch.ts`
- API schema 构造：`src/utils/api.ts`
- 请求侧过滤逻辑：`src/services/api/claude.ts`
- 消息归一化与 `tool_reference` 清洗：`src/utils/messages.ts`
- 工具执行入口：`src/services/tools/toolExecution.ts`
- tool_use 调度：`src/query.ts`、`src/services/tools/toolOrchestration.ts`

## 完整流程

下面按一次真实时序来讲。

### 1. 启动时先把 `ToolSearchTool` 注册进基础工具集

在 `src/tools.ts` 中，`ToolSearchTool` 会按“乐观判断”加入工具列表：

- 条件：`isToolSearchEnabledOptimistic()`
- 含义：这里只判断“有没有可能启用”
- 注意：这一步只是注册工具，不会调用 `ToolSearchTool.call()`

也就是说，程序启动或构建工具列表时，只是让它“有资格”出现在某些请求里。

### 2. 系统先判断哪些工具属于 deferred tools

`isDeferredTool()` 决定一个工具是否需要先通过 `ToolSearch` 才能真正调用。

规则大致是：

- `tool.alwaysLoad === true`：永不延迟
- `tool.isMcp === true`：默认延迟
- `ToolSearch` 自己：永不延迟
- 某些必须首轮可见的特殊工具：永不延迟
- 其他工具：当 `tool.shouldDefer === true` 时延迟

这部分逻辑在 `src/tools/ToolSearchTool/prompt.ts`。

### 3. 每次发请求前，`claude.ts` 再做一次最终启用判断

真正是否启用 tool search，不在注册阶段决定，而是在 `src/services/api/claude.ts` 里每次请求前决定。

这里会调用 `isToolSearchEnabled()`，综合判断：

- `ENABLE_TOOL_SEARCH` 环境变量
- 当前模型是否支持 `tool_reference`
- `ToolSearchTool` 是否还在当前工具列表里
- `tst-auto` 模式下 deferred tools 的总体体积是否大到值得延迟加载

当前代码里，像某些 `haiku` 模型会被视为不支持 `tool_reference`。

### 4. 如果启用了 tool search，本轮并不会把所有 deferred tools 都发给模型

这是 `claude.ts` 里最关键的一步。

如果 `useToolSearch === true`，它会先收集所有 deferred tools 的名字，然后调用 `extractDiscoveredToolNames(messages)`，从历史消息里找出已经被 `ToolSearch` 发现过的工具。

最后，真正发给 API 的 `filteredTools` 只保留：

- 非 deferred 工具
- `ToolSearch` 自己
- 已经 discovered 的 deferred 工具

未被发现的 deferred tools 不会在当前请求中带上完整 schema。

### 5. `tool_reference` 在这里的角色不是“立刻展开”，而是“记录已发现工具”

`ToolSearchTool` 真正返回结果时，不是只返回文本，而是把匹配到的工具名映射成 `tool_reference` 块。

如果找到了工具，结果语义上类似：

- `tool_result`
- `content`
- `tool_reference(tool_name=...)`
- `tool_reference(tool_name=...)`

对应实现就在 `ToolSearchTool.ts` 的 `mapToolResultToToolResultBlockParam()`。

这些 `tool_reference` 会留在消息历史里，供后续请求使用。

### 6. `claude.ts` 会把 `tool_reference` 当成 discovered set 的来源

`extractDiscoveredToolNames(messages)` 会扫描历史消息，专门查找：

- user message 里的 `tool_result`
- 其中 array content 里的 `tool_reference`
- 提取每个 `tool_reference.tool_name`

于是上一轮 `ToolSearch` 找到过哪些工具，就会变成下一轮的 discovered tools 集合。

这就是“先搜索，再真正加载 schema”的核心连接点。

### 7. 请求发送前，消息归一化阶段还会专门处理 `tool_reference`

`claude.ts` 在真正发请求前会调用：

- `normalizeMessagesForAPI(messages, filteredTools)`

这一步和 `tool_reference` 有几层关系。

当 tool search 没启用时：

- user message 中的 `tool_reference` 会被剥掉
- assistant message 里的 `caller` 字段也会被剥掉
- 这样可以避免不支持该 beta 的模型/API 报 400

当 tool search 已启用时：

- 不会粗暴删除全部 `tool_reference`
- 只会删除那些指向“当前已不存在工具”的引用
- 典型场景是某个 MCP server 断开后，历史消息里还残留旧引用

此外，`normalizeMessagesForAPI()` 还会处理 `tool_reference` 所在消息附近的 text sibling，避免 server 在展开后形成对模型不友好的提示词尾部模式。

### 8. 真正发请求时，schema 上会通过 `defer_loading` 表示按需加载

`src/utils/api.ts` 中的 `toolToAPISchema()` 支持 `deferLoading` 参数。

如果某个工具本轮应该延迟加载，就会在发送给 API 的 schema 上附加：

```json
{
  "defer_loading": true
}
```

这表示模型知道“有这个工具”，但当前还没有它的完整调用 schema。

### 9. 只有当模型真的发出 `tool_use(name="ToolSearch")` 时，`ToolSearchTool.call()` 才会执行

这一步很重要：`ToolSearchTool.call()` 不是在注册工具时调用，也不是在构造 schema 时调用，而是在模型本轮输出了 `tool_use` 之后才执行。

完整链路是：

- `query.ts` 在流式 assistant message 中收集 `tool_use` block
- `toolOrchestration.ts` / `toolExecution.ts` 按工具名找到对应工具
- 最终调用 `tool.call(...)`

如果这个工具名正好是 `ToolSearch`，这里才会真正进入 `ToolSearchTool.call()`。

### 10. `ToolSearchTool.call()` 会返回匹配结果或 `tool_reference`

`ToolSearchTool.call()` 支持两类查询：

- 直接选择
  - `select:Read`
  - `select:Read,Edit,Grep`
- 关键词搜索
  - `notebook jupyter`
  - `+slack send`

其中 `+term` 表示必选词。

执行时它会：

- 从当前 `tools` 中筛出 deferred tools
- 检查是否有 MCP server 仍在连接中
- 处理 `select:` 直选
- 或执行关键词搜索
- 最后把结果包装成 `tool_result`

如果有匹配，则 `tool_result.content` 是 `tool_reference[]`；如果没有匹配，则返回普通文本。

### 11. 下一轮请求时，这些工具就会进入真正的工具 schema 列表

因为上一轮 `ToolSearch` 的结果已经把 `tool_reference` 写进了消息历史，所以下一轮 `claude.ts` 在调用 `extractDiscoveredToolNames(messages)` 时，就能读到这些工具名。

于是这些工具会从“仅延迟可见”变成“完整 schema 已发给模型”。

到这一步，模型才能像调用普通工具一样调用它们。

### 12. 如果模型跳过 `ToolSearch`，直接调用 deferred tool，会触发补救提示

`src/services/tools/toolExecution.ts` 中有一个补救逻辑。

如果满足：

- 某个工具是 deferred tool
- 它还不在 discovered set 中
- 模型却直接调用了它
- 结果导致参数校验失败

系统会追加提示，告诉模型：

- 这个工具的 schema 还没发到 API
- 先调用 `ToolSearch`
- 使用 `select:<tool_name>` 加载它
- 然后再重试

这样可以减少模型明明知道工具名、却因为不知道参数 schema 而持续报错的情况。

## `claude.ts` 对 `tool_reference` 的具体处理

很多人会以为 `tool_reference` 在 `claude.ts` 里会被“展开成完整 schema”。实际上不是。

`claude.ts` 里对 `tool_reference` 的职责主要是四件事：

### 1. 判断本轮是否允许它出现

如果本轮模型/API 不支持 tool search，就不会保留 `ToolSearchTool`，也不会让相关字段继续进 API 请求。

### 2. 从历史消息中提取已发现工具名

它通过 `extractDiscoveredToolNames(messages)` 扫描历史消息中的 `tool_reference`，得到 discovered set。

### 3. 用 discovered set 决定本轮哪些 deferred tools 真正带 schema

这一步发生在 `filteredTools` 过滤过程中。也就是说，`tool_reference` 本质上像一个“解锁记录”。

### 4. 在不兼容时清洗掉 `tool_reference`

如果本轮不支持 tool search，就在发请求前把 user message 中的 `tool_reference` 和 assistant message 中的 `caller` 清理掉，避免 API 报错。

一句话说，`claude.ts` 不负责“展开” `tool_reference`，而是负责：

- 允许或禁止它存在
- 从消息历史中读取它
- 把它转化成 discovered tools 集合
- 基于这个集合筛选出下一轮真正要发 schema 的工具

## ToolSearch 自身怎么搜索

关键词搜索在 `searchToolsWithKeywords()` 中实现，核心步骤是：

1. 精确匹配工具名
2. 如果查询像 `mcp__server`，按 MCP 前缀匹配
3. 将查询拆成 terms
4. 解析工具名
   - MCP 工具：`mcp__server__action`
   - 普通工具：如 `FileEditTool` 拆成 `file edit tool`
5. 对每个候选工具打分
   - 工具名精确部分匹配：高分
   - 工具名部分包含：中分
   - `searchHint` 命中：加分
   - 工具 prompt/description 命中：低一些的分
6. 排序后返回 top N

为了减少重复开销，它会缓存 deferred tools 的 description；当 deferred tool 集合变化时，缓存失效。

## 一句话总结

`ToolSearchTool` 是整个工具系统里的“按需发现和解锁 deferred tools 的入口”。

完整闭环是：

- 工具先被标记为 deferred
- 请求时只暴露 `ToolSearch` 和已发现工具
- 模型调用 `ToolSearch`
- `ToolSearch` 返回 `tool_reference`
- `claude.ts` 从历史消息中提取 discovered tools
- 下一轮再把这些工具的完整 schema 发给模型

它通过 `defer_loading + tool_reference + discovered-tool set` 这套机制，把“大量工具可用”和“上下文成本可控”这两件事兼顾起来。

## 实现补充

Anthropic 原生支持 `defer_loading + tool_reference`，因此可以直接把大量工具标记为按需加载。

如果目标模型平台不支持这套原生协议，就需要在客户端自己实现一层“工具发现与延迟加载”编排。一个常见做法是：

- 先在 system prompt 或额外上下文中只暴露 deferred tools 的轻量目录信息
- 提供一个专门的 tool search / tool select 工具，让模型先选出要用的工具
- 在下一轮请求里，把被选中的工具完整 schema 注入给模型
- 如果平台没有原生 tool calling，再额外自己实现参数解析与执行回传

这份仓库更接近 Anthropic 原生方案：

- deferred tools 先按需加载，而不是一开始就塞完整 schema
- `ToolSearch` 返回 `tool_reference`
- 后续请求再把已发现工具的完整 schema 发送给模型

参考：

- https://www.anthropic.com/engineering/advanced-tool-use

```json
{
  "tools": [
    {
      "type": "tool_search_tool_regex_20251119",
      "name": "tool_search_tool_regex"
    },
    {
      "name": "github.createPullRequest",
      "description": "Create a pull request",
      "input_schema": {},
      "defer_loading": true
    }
  ]
}
```
