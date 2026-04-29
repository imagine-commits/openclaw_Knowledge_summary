# Claude Code React Loop 拆解分析

> 源码来源: [oboard/claude-code-rev](https://github.com/oboard/claude-code-rev)  
> 对应源文件: `src/query.ts` (1729 行)  
> 分析日期: 2026-04-29

---

## 一、整体架构：三层循环

Claude Code 的核心是一个 **三层嵌套循环**，从外到内：

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: query()        — AsyncGenerator 协程, 产事件流     │
│  ├─────────────────────────────────────────────────────────┤│
│  │  Layer 2: queryLoop()  — while(true) 主循环, 状态机      ││
│  │  ├───────────────────────────────────────────────────┐  ││
│  │  │  Layer 3: callModel() — for await 流式 API 调用  │  ││
│  │  └───────────────────────────────────────────────────┘  ││
└─────────────────────────────────────────────────────────────┘
```

### 三层职责

| 层级 | 函数 | 循环类型 | 职责 |
|------|------|----------|------|
| L1 | `query()` | AsyncGenerator | 包装器, 处理命令生命周期通知 |
| L2 | `queryLoop()` | `while(true)` | 主状态机, 管理消息、压缩、工具执行、重试 |
| L3 | `deps.callModel()` | `for await` | 流式 API 调用, yield stream events |

---

## 二、queryLoop() 主循环结构

### 2.1 状态机 State 类型

```typescript
type State = {
  messages: Message[]                    // 对话历史
  toolUseContext: ToolUseContext         // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState | undefined  // 自动压缩状态
  maxOutputTokensRecoveryCount: number   // max_output_tokens 重试计数
  hasAttemptedReactiveCompact: boolean   // 是否已尝试 reactive compact
  maxOutputTokensOverride: number | undefined  // 输出 token 上限覆盖
  pendingToolUseSummary: Promise | undefined   // 工具使用摘要(异步)
  stopHookActive: boolean | undefined    // stop hooks 是否激活
  turnCount: number                      // 当前轮次
  transition: Continue | undefined      // 上一次循环继续原因
}
```

### 2.2 循环入口图

```
while (true) {
    ┌─ 状态解构 ──────────────────────────────────────────────┐
    │  let { toolUseContext } = state                         │
    │  const { messages, autoCompactTracking, ... } = state    │
    └────────────────────────────────────────────────────────┘
         │
         ▼
    ┌─ 预压缩处理 ───────────────────────────────────────────┐
    │  1. startRelevantMemoryPrefetch() 内存预取             │
    │  2. skillPrefetch.startSkillDiscoveryPrefetch() 技能发现│
    │  3. applyToolResultBudget() 工具结果预算               │
    │  4. snipCompactIfNeeded() 历史裁剪                      │
    │  5. microcompact() 微压缩                               │
    │  6. contextCollapse.applyCollapsesIfNeeded() 上下文折叠│
    │  7. autocompact() 自动压缩                             │
    └────────────────────────────────────────────────────────┘
         │
         ▼
    ┌─ 流式 API 调用 (for await deps.callModel) ─────────────┐
    │  • stream_request_start 事件                            │
    │  • assistant 消息流 (含 thinking/tool_use/text blocks)  │
    │  • tool_result 结果流                                   │
    │  • 错误 withheld (PTL/max_output_tokens/媒体错误)       │
    └────────────────────────────────────────────────────────┘
         │
         ▼
    ┌─ 错误恢复 (needsFollowUp === false 时) ─────────────────┐
    │  1. 413 PTL  → contextCollapse.recoverFromOverflow()   │
    │               → reactiveCompact.tryReactiveCompact()    │
    │  2. 媒体错误  → reactiveCompact.strip + retry            │
    │  3. max_output_tokens → 升级到 64k → 注 meta 消息重试   │
    │  4. stop hooks → blockingErrors → retry loop           │
    │  5. token_budget → nudge 或 early stop                  │
    └────────────────────────────────────────────────────────┘
         │
         ▼
    ┌─ 工具执行 ─────────────────────────────────────────────┐
    │  • streamingToolExecutor (流式) 或 runTools (批处理)     │
    │  • yield tool_result 消息                               │
    │  • generateToolUseSummary (Haiku, 非阻塞)               │
    └────────────────────────────────────────────────────────┘
         │
         ▼
    ┌─ 附件处理 (进入下一轮前) ───────────────────────────────┐
    │  • getCommandsByMaxPriority() 获取队列命令              │
    │  • getAttachmentMessages() 处理文件变更/MCP通知         │
    │  • filterDuplicateMemoryAttachments() 内存附件去重      │
    │  • collectSkillDiscoveryPrefetch() 技能发现结果         │
    └────────────────────────────────────────────────────────┘
         │
         ▼
    ┌─ 状态更新 & continue ──────────────────────────────────┐
    │  state = { messages: [...], toolUseContext, turnCount++│
    │           maxOutputTokensRecoveryCount: 0,              │
    │           hasAttemptedReactiveCompact: false, ... }     │
    └────────────────────────────────────────────────────────┘
         │
         ▼ (loop back to while(true))
```

---

## 三、核心压缩机制

### 3.1 压缩管道 (按执行顺序)

```
messagesForQuery (原始消息)
    │
    ▼
┌─────────────────┐
│ HISTORY_SNIP     │  裁剪历史消息, 释放 token
│ (snipCompact)   │
└────────┬────────┘
         ▼
┌─────────────────┐
│ MICROCOMPACT    │  工具结果缓存编辑 (cache_deleted_input_tokens)
└────────┬────────┘
         ▼
┌─────────────────┐
│ CONTEXT_COLLAPSE│  读时投影, 折叠存储提交日志
│ (实验特性)       │
└────────┬────────┘
         ▼
┌─────────────────┐
│ AUTOCOMPACT     │  主动压缩, 超阈值时触发
└────────┬────────┘
         ▼
 postCompactMessages (压缩后消息)
```

### 3.2 Reactive Compact 错误恢复

```typescript
// 1. 流式循环中 withheld 错误:
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true  // 不 yield, 继续累积到 assistantMessages
}

// 2. 流结束后检查:
if (isWithheld413) {
  // 优先: 泄掉 staged context-collapse
  const drained = contextCollapse.recoverFromOverflow(messagesForQuery)
  if (drained.committed > 0) {
    state = { ...state, transition: 'collapse_drain_retry' }
    continue  // ← 回到循环开头
  }
}

// 3. 其次: reactive compact
if (reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({...})
  if (compacted) {
    state = { ...state, transition: 'reactive_compact_retry' }
    continue
  }
}

// 4. 恢复失败: surface 错误, 执行 stop hooks, return
```

---

## 四、工具执行模式

### 4.1 两种执行器

| 模式 | 触发条件 | 行为 |
|------|----------|------|
| `streamingToolExecutor` | `config.gates.streamingToolExecution` | 流式: 工具边执行边 yield 结果 |
| `runTools` | 默认/旧模式 | 批处理: 等所有工具执行完再 yield |

### 4.2 流式工具执行流程

```typescript
// 创建 executor (在循环开始时)
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(tools, canUseTool, toolUseContext)
  : null

// 流式循环中: 边收 tool_use 边加入 executor
for (const toolBlock of msgToolUseBlocks) {
  streamingToolExecutor.addTool(toolBlock, message)
}

// 流结束后取已完成结果
for (const result of streamingToolExecutor.getCompletedResults()) {
  yield result.message  // 实时 yield 给上游
}

// 工具全部执行完后取剩余结果
for await (const update of streamingToolExecutor.getRemainingResults()) {
  yield update.message
}
```

---

## 五、关键 continue 点 (回到循环开头)

| # | 触发条件 | transition reason | 说明 |
|---|----------|-------------------|------|
| 1 | `collapse_drain_retry` | `contextCollapse.recoverFromOverflow()` 成功 | 泄掉 staged 折叠, 重试 API |
| 2 | `reactive_compact_retry` | `reactiveCompact.tryReactiveCompact()` 成功 | 压缩后重试 |
| 3 | `max_output_tokens_escalate` | `maxOutputTokensOverride === undefined` 且 capEnabled | 升级到 64k 再试 |
| 4 | `max_output_tokens_recovery` | `maxOutputTokensRecoveryCount < 3` | 注 meta 消息重试 |
| 5 | `stop_hook_blocking` | `stopHookResult.blockingErrors.length > 0` | stop hooks 阻塞, 重试 |
| 6 | `token_budget_continuation` | `checkTokenBudget() === 'continue'` | token 预算继续 |
| 7 | `next_turn` | 正常工具执行完毕 | 下一轮 |

---

## 六、关键设计决策

### 6.1 为什么用 `while(true)` 而不是递归?

- **内存安全**: 递归深度 = 轮次数, 长对话会爆栈
- **状态易变**: `state` 对象每次 `continue` 前整体替换, 避免闭包陷阱
- **Generator 语义**: `yield*` 穿透错误, `return` 处理清理

### 6.2 错误 withheld 机制

流式循环中遇到 `PTL`/`max_output_tokens` 错误时:
- **不 yield** 给上层 (避免 SDK 消费者终止会话)
- **继续累积** 到 `assistantMessages`
- **流结束后** 才处理恢复逻辑
- 这保证: 即使错误, 循环依然完整走完, 恢复逻辑始终能执行

### 6.3 `toolUseContext` 的特殊处理

```typescript
let { toolUseContext } = state  // ← let, 可重新赋值
// ...
toolUseContext = { ...toolUseContext, queryTracking }  // ← 重新赋值
// ...
toolUseContext = {
  ...toolUseContext,
  options: { ...toolUseContext.options, tools: refreshedTools }
}
```

其余状态用 `const` 解构, 保证 **read-only between continue sites**。

---

## 七、React Hook: useMainLoopModel

```typescript
// src/hooks/useMainLoopModel.ts
export function useMainLoopModel(): ModelName {
  const mainLoopModel = useAppState(s => s.mainLoopModel)
  const mainLoopModelForSession = useAppState(s => s.mainLoopModelForSession)

  // GrowthBook 刷新时强制重渲染 (alias 解析可能过期)
  const [, forceRerender] = useReducer(x => x + 1, 0)
  useEffect(() => onGrowthBookRefresh(forceRerender), [])

  const model = parseUserSpecifiedModel(
    mainLoopModelForSession ?? mainLoopModel ?? getDefaultMainLoopModelSetting()
  )
  return model
}
```

这是 React 侧获取当前 loop model 的 hook，与 query loop 的 `currentModel` 呼应。

---

## 八、文件速查

| 路径 | 作用 |
|------|------|
| `src/query.ts` | 主循环, 1729 行 |
| `src/hooks/useMainLoopModel.ts` | React hook 获取当前 model |
| `src/query/tokenBudget.ts` | token 预算管理 |
| `src/query/stopHooks.ts` | stop hooks 处理 |
| `src/services/compact/compact.ts` | 主压缩逻辑 |
| `src/services/compact/autoCompact.ts` | 自动压缩触发 |
| `src/services/compact/reactiveCompact.ts` | 响应式压缩(实验) |
| `src/services/compact/microcompact.ts` | 微压缩 |
| `src/services/tools/StreamingToolExecutor.ts` | 流式工具执行器 |
| `src/services/tools/toolOrchestration.ts` | 批处理工具执行 |
| `src/services/toolUseSummary/toolUseSummaryGenerator.ts` | Haiku 工具摘要 |
| `src/context/QueuedMessageContext.tsx` | React 队列消息上下文 |
| `src/context/notifications.tsx` | React 通知上下文 |
| `src/skills/bundled/loop.ts` | `/loop` 技能实现 |
