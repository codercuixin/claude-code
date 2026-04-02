# 单次 Agent Loop 时序图

下面这张图对应的是 `src/query.ts` 里 `queryLoop()` 的“一次 iteration”主流程，也就是一次 `模型采样 -> 可能执行工具 -> 决定是否进入下一轮`。

```mermaid
sequenceDiagram
    autonumber
    participant U as "REPL / QueryEngine"
    participant Q as "queryLoop()"
    participant M as "callModel() / Claude API"
    participant S as "StreamingToolExecutor"
    participant T as "Tool Runner"
    participant A as "Attachments / Memory / Queue"

    U->>Q: 调用 query(...)，传入 messages / systemPrompt / toolUseContext
    Q->>Q: 读取 state<br/>裁剪 messagesForQuery<br/>做 compact / prefetch / budget 检查
    Q-->>U: stream_request_start

    Q->>M: 发起流式请求
    loop 流式事件
        M-->>Q: stream_event(content_block_start/delta/stop, message_delta...)
        alt 普通 text / thinking block 完成
            Q-->>U: assistant message
        else 收到 tool_use block
            Q->>Q: assistantMessages += block
            Q->>Q: toolUseBlocks += block
            Q->>Q: needsFollowUp = true
            opt 启用 streaming tool execution
                Q->>S: addTool(tool_use)
                S->>T: 立即启动工具
                opt 工具先于模型结束
                    T-->>S: progress / tool_result
                    S-->>Q: 已完成结果
                    Q-->>U: progress / user(tool_result)
                end
            end
        else 收到可恢复 API 错误
            Q->>Q: 先暂存，不立刻向上游暴露
        end
    end

    Q->>Q: 流结束，执行 post-sampling hooks

    alt 用户在流式阶段中断
        opt 有 streaming executor
            Q->>S: getRemainingResults()
            S-->>Q: synthetic tool_result / 剩余结果
            Q-->>U: user interruption / tool_result
        end
        Q-->>U: 返回 aborted_streaming
    else 没有 tool_use
        alt 命中 prompt-too-long / max-output-tokens 恢复
            Q->>Q: 构造 next state
            Q->>Q: continue 进入下一次 iteration
        else stop hooks 阻止继续
            Q-->>U: 返回 completed / stop_hook_prevented
        else token budget 要求继续
            Q->>Q: 注入 meta user message
            Q->>Q: continue 进入下一次 iteration
        else 正常结束
            Q-->>U: 返回 completed
        end
    else 有 tool_use
        alt 未启用 streaming tool execution
            Q->>T: runTools(toolUseBlocks)
            T-->>Q: progress / tool_result / context update
            Q-->>U: progress / user(tool_result)
        else 已启用 streaming tool execution
            Q->>S: getRemainingResults()
            S-->>Q: 剩余 progress / tool_result / context update
            Q-->>U: progress / user(tool_result)
        end

        alt 工具阶段被中断
            Q-->>U: user interruption
            Q-->>U: 返回 aborted_tools
        else hook 明确要求停止续转
            Q-->>U: 返回 hook_stopped
        else 继续组装下一轮上下文
            Q->>A: 注入 attachment / queue / memory / skill prefetch
            A-->>Q: attachment messages
            Q-->>U: attachment messages
            Q->>Q: next.messages = messages + assistant + toolResults
            Q->>Q: turnCount++
            Q->>Q: continue 进入下一次 iteration
        end
    end
```

可以把它压缩成一句话理解：

`queryLoop` 不是“一问一答”，而是一个内部小状态机：

`准备上下文 -> 流式采样 -> 收集 tool_use -> 执行工具 -> 把 tool_result 回灌消息历史 -> 再次采样，直到没有 follow-up 为止`

几个关键代码位置：

- 主循环和 `state` 在 `src/query.ts`
- 流式采样与 `needsFollowUp` 在 `src/query.ts`
- 无工具时的恢复/终止分支在 `src/query.ts`
- 工具执行与下一轮 state 构造在 `src/query.ts`
- streaming 工具执行器在 `src/services/tools/StreamingToolExecutor.ts`
- 普通工具编排在 `src/services/tools/toolOrchestration.ts`
