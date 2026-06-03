# Compact 实现详解

## 目录
1. [触发时机](#1-触发时机)
2. [入口：/compact 命令](#2-入口compact-命令)
3. [四条执行路径](#3-四条执行路径)
4. [核心函数：compactConversation](#4-核心函数compactconversation)
5. [摘要生成：streamCompactSummary](#5-摘要生成streamcompactsummary)
6. [PTL 重试机制](#6-ptl-重试机制)
7. [Post-Compact 重注入](#7-post-compact-重注入)
8. [partialCompactConversation](#8-partialcompactconversation)
9. [消息结构](#9-消息结构)
10. [关键常量与阈值](#10-关键常量与阈值)

---

## 1. 触发时机

Compact 有三种触发方式：

### 1.1 手动触发（/compact）

用户主动执行 `/compact [自定义指令]`，入口是 `src/commands/compact/compact.ts`。

### 1.2 自动触发（Auto-Compact）

文件：`src/services/compact/autoCompact.ts`

**触发条件**：每次 API 响应后，主循环检查 token 用量是否超过阈值。

```
有效上下文窗口 = 模型上下文窗口 - 20,000（为摘要输出预留）
自动 compact 阈值 = 有效上下文窗口 - 13,000（AUTOCOMPACT_BUFFER_TOKENS）
```

以 200K 上下文的 claude-sonnet-4-6 为例：
- 有效窗口 ≈ 180,000 tokens
- 自动 compact 阈值 ≈ 167,000 tokens
- 用量超过 167K → 自动触发 compact

**环境变量控制**：
```bash
DISABLE_COMPACT=1          # 完全禁用
DISABLE_AUTO_COMPACT=1     # 仅禁用自动，保留手动
CLAUDE_CODE_AUTO_COMPACT_WINDOW=xxx  # 覆盖上下文窗口大小
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=80   # 用百分比覆盖阈值（测试用）
```

**配置项**：`userConfig.autoCompactEnabled`（settings.json 中可配置）

**熔断机制**：连续失败 3 次后停止重试，避免无效 API 调用浪费（BQ 数据显示这可节省 ~250K API calls/day）。

### 1.3 用量警告阶段

在触发 compact 前，有两个预警阶段：

```
警告阈值 = 有效窗口 - 20,000    → 显示黄色警告
错误阈值 = 有效窗口 - 20,000    → 显示红色警告
阻塞阈值 = 有效窗口 - 3,000     → 阻止新消息提交（MANUAL_COMPACT_BUFFER_TOKENS）
```

### 1.4 不触发的场景

以下 querySource 会跳过自动 compact（防止递归）：
- `session_memory`：session memory compact 本身是 forked agent
- `compact`：compact forked agent 不触发自身
- `marble_origami`（内部）：ctx-agent 绕过，避免破坏主线程状态
- 启用了 REACTIVE_COMPACT 或 CONTEXT_COLLAPSE 功能开关时

---

## 2. 入口：/compact 命令

文件：`src/commands/compact/compact.ts`

```typescript
export const call: LocalCommandCall = async (args, context) => {
  // 1. 只处理最后一个 compact boundary 之后的消息
  messages = getMessagesAfterCompactBoundary(messages)

  const customInstructions = args.trim()

  // 2. 路径选择（见下节）
  if (!customInstructions) {
    // 尝试 Session Memory Compact
  }
  if (reactiveCompact?.isReactiveOnlyMode()) {
    // Reactive Compact 路径
  }
  // 传统 compact 路径
}
```

**getCacheSharingParams()**：构建 forked agent 所需的缓存共享参数，包含：
- 完整系统提示（system prompt + custom prompt + append prompt）
- 用户上下文（getUserContext）
- 系统上下文（getSystemContext）
- 工具列表、当前消息

---

## 3. 四条执行路径

```
/compact [指令]
     │
     ├─1─ 无自定义指令？
     │       └─ trySessionMemoryCompaction()
     │              ├─ 成功 → 写入 session memory 文件，返回
     │              └─ 失败 → 继续
     │
     ├─2─ isReactiveOnlyMode()？（功能开关 REACTIVE_COMPACT）
     │       └─ compactViaReactive()
     │              └─ reactiveCompactOnPromptTooLong()
     │
     └─3─ 传统路径（默认）
              ├─ microcompactMessages()    轻量预处理
              └─ compactConversation()     完整摘要压缩

自动触发时（autoCompactIfNeeded）：
     ├─1─ trySessionMemoryCompaction()    优先尝试
     └─2─ compactConversation(isAutoCompact=true)
```

### 路径 1：Session Memory Compact
- 将对话历史写入 session memory 文件（不生成摘要）
- 最轻量的方式，不调用模型，无 API 成本
- 仅在无自定义指令时可用

### 路径 2：Reactive Compact（功能开关）
- 专为处理 API 返回 prompt-too-long（413）错误设计
- 从尾部（最近的消息）开始逐步压缩，而非从头部截断
- 保留最新上下文，丢弃最早历史

### 路径 3：传统 Compact（默认）
先 `microcompactMessages()` 轻量预处理，再 `compactConversation()` 完整摘要。

`microcompactMessages()` 的工作：
- 去除内容完全重复的消息
- 截断过长的 tool_result 内容
- 剥离不必要的附件类型

---

## 4. 核心函数：compactConversation

文件：`src/services/compact/compact.ts`，函数签名：

```typescript
async function compactConversation(
  messages: Message[],
  context: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  suppressFollowUpQuestions: boolean,
  customInstructions?: string,
  isAutoCompact: boolean = false,
  recompactionInfo?: RecompactionInfo,
): Promise<CompactionResult>
```

**完整执行顺序**：

```
1. 计算压缩前 token 数（preCompactTokenCount）

2. 执行 PreCompact Hooks
   └─ hookResult.newCustomInstructions 会合并到 customInstructions

3. 构建摘要请求（getCompactPrompt(customInstructions)）

4. PTL 重试循环（for;;）
   └─ streamCompactSummary() → 生成摘要文本
      ├─ 成功：summary 不以 PROMPT_TOO_LONG 开头 → break
      └─ PTL：truncateHeadForPTLRetry() → 丢弃最旧消息组 → 重试

5. 清空状态
   ├─ context.readFileState.clear()
   └─ context.loadedNestedMemoryPaths?.clear()
   注意：故意不重置 sentSkillNames（见注释：节省 ~4K tokens/compact）

6. 并行生成 post-compact 附件
   ├─ createPostCompactFileAttachments()  重注入最近文件
   └─ createAsyncAgentAttachmentsIfNeeded()  异步 agent 状态

7. 附加其他附件
   ├─ createPlanAttachmentIfNeeded()       当前 plan 文件
   ├─ createPlanModeAttachmentIfNeeded()   plan mode 指令（如在 plan mode 中）
   ├─ createSkillAttachmentIfNeeded()      已调用的 skills
   ├─ getDeferredToolsDeltaAttachment()    工具 delta（重新宣告全量）
   ├─ getAgentListingDeltaAttachment()     agent 列表
   └─ getMcpInstructionsDeltaAttachment()  MCP 工具指令

8. 执行 SessionStart Hooks（source='compact'）
   └─ 重新触发会话初始化逻辑

9. 创建 compact 边界标记（CompactBoundaryMessage）
   └─ 记录 preCompactDiscoveredTools，确保 deferred tools 继续发送

10. 记录 tengu_compact 事件（含 cache 命中率、token 使用量等指标）

11. notifyCompaction()    重置 prompt cache break 检测基线
    markPostCompaction()  标记 compact 完成

12. reAppendSessionMetadata()  保持会话标题在 16KB tail window 内

13. 执行 PostCompact Hooks
    └─ postCompactHookResult.userDisplayMessage 与 PreCompact 消息合并

14. 返回 CompactionResult
```

**CompactionResult 结构**：

```typescript
interface CompactionResult {
  boundaryMarker: SystemMessage      // compact 边界标记
  summaryMessages: UserMessage[]     // 摘要消息（含 isCompactSummary=true）
  attachments: AttachmentMessage[]   // 重注入的文件/工具/skills 附件
  hookResults: HookResultMessage[]   // SessionStart Hook 执行结果
  messagesToKeep?: Message[]         // partial compact 时保留的消息
  userDisplayMessage?: string        // Hook 生成的用户可见消息
  preCompactTokenCount?: number
  postCompactTokenCount?: number     // 实际是 compact API 调用的总 token 数
  truePostCompactTokenCount?: number // compact 后上下文的真实大小估算
  compactionUsage?: TokenUsage       // API 用量明细
}
```

---

## 5. 摘要生成：streamCompactSummary

这是 compact 最核心也最复杂的函数，负责将长对话压缩为摘要文本。

### 5.1 消息预处理

发给 API 之前，消息经过三层过滤：

```typescript
normalizeMessagesForAPI(
  stripImagesFromMessages(          // 图像 → "[image]"，文档 → "[document]"
    stripReinjectedAttachments(     // 过滤会在 post-compact 重注入的附件
      getMessagesAfterCompactBoundary(messages)  // 只取最后一个 boundary 后的消息
    )
  )
)
```

**为什么剥离图像？**
- 图像本身不贡献摘要价值
- 图像 block 会使 compact 请求本身超出 token 限制（特别是 CCD 场景）
- 替换为占位符文本，摘要仍能知道图像曾经存在

**为什么过滤 skill_discovery/skill_listing 附件？**
- 这些附件会在 post-compact 时重新注入
- 提前过滤避免摘要器处理过时的 skill 建议（约节省 4K tokens）

### 5.2 路径 A：Forked Agent 缓存共享（优先）

```typescript
const result = await runForkedAgent({
  promptMessages: [summaryRequest],   // 只追加一条"请总结以上对话"
  cacheSafeParams,                     // 与主线程完全相同的系统提示参数
  canUseTool: createCompactCanUseTool(),  // 禁止所有工具调用
  querySource: 'compact',
  forkLabel: 'compact',
  maxTurns: 1,
  skipCacheWrite: true,               // 不写入缓存，避免污染主线程缓存
})
```

**缓存共享原理**：
- Forked agent 使用与主线程完全相同的 system prompt + tools + 消息前缀
- Anthropic API 的 prompt cache 基于前缀匹配：参数相同 → 命中缓存
- 缓存命中率通过 `tengu_compact_cache_sharing_success` 事件追踪
- BQ 实验（Jan 2026）显示：未共享路径 98% cache miss，消耗约 0.76% 全球 cache_creation token

**重要注意事项**：
```typescript
// 禁止在这里设置 maxOutputTokens！
// 设置会通过 Math.min(budget, maxOutputTokens-1) 修改 thinking config，
// 导致与主线程的 thinking config 不一致，破坏 prompt cache 命中
```

**工具限制**：
```typescript
function createCompactCanUseTool(): CanUseToolFn {
  return async () => ({
    behavior: 'deny',
    message: 'Tool use is not allowed during compaction',
    // 摘要器应该只产生文本，不调用任何工具
  })
}
```

**Abort 处理**：
- 使用主线程的 abortController，确保用户 Esc 能中断 compact
- 若 forked agent 以 `isApiErrorMessage` 返回（如 APIUserAbortError），视为失败并 fallback

### 5.3 路径 B：直接流式生成（fallback）

当 forked agent 路径失败（或被 feature flag 禁用）时，直接调用 `queryModelWithStreaming`。

**工具集选择**：
```typescript
const tools: Tool[] = useToolSearch
  ? uniqBy([FileReadTool, ToolSearchTool, ...mcpTools], 'name')
  : [FileReadTool]  // 大多数情况只注入 FileReadTool
```
- 只给摘要器最小权限（只读文件）
- 若启用了 tool search，额外注入 ToolSearchTool 和 MCP 工具

**流式参数**：
```typescript
{
  thinkingConfig: { type: 'disabled' },  // 禁用 extended thinking，节省 token
  maxOutputTokensOverride: Math.min(COMPACT_MAX_OUTPUT_TOKENS, modelMax),
  querySource: 'compact',
}
```

**流式重试**：
- `tengu_compact_streaming_retry` 功能开关控制（默认关闭）
- 最多 `MAX_COMPACT_STREAMING_RETRIES = 2` 次重试
- 重试间隔使用 `getRetryDelay(attempt)` 指数退避

### 5.4 Keep-Alive 机制

compact API 调用可能需要 5-10 秒，期间无消息流动。为避免远程会话 WebSocket 超时：

```typescript
const activityInterval = setInterval(() => {
  sendSessionActivitySignal()    // PUT /worker heartbeat
  context.setSDKStatus?.('compacting')  // 重新发送 compacting 状态
}, 30_000)
// try { ... } finally { clearInterval(activityInterval) }
```

---

## 6. PTL 重试机制

PTL = Prompt Too Long（API 返回 prompt-too-long 错误）

当 compact 请求本身超出 API token 限制时触发。最多重试 `MAX_PTL_RETRIES = 3` 次。

### 6.1 truncateHeadForPTLRetry 算法

```typescript
function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,  // 包含 token gap 信息
): Message[] | null {
  // 1. 移除上次重试插入的合成标记（防止它自己成为 group 0）
  const input = startsWithMarker ? messages.slice(1) : messages

  // 2. 按 API 轮次分组（一次用户请求 + 一次助手回复 = 一组）
  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null  // 无法再丢弃

  // 3. 计算需要丢弃多少组
  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  if (tokenGap !== undefined) {
    // 精确：累加最旧的组直到覆盖 gap
    dropCount = 累加直到 acc >= tokenGap
  } else {
    // 兜底：丢弃 20%（Vertex/Bedrock 错误格式无 gap 信息时）
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  // 4. 至少保留一组供摘要使用
  dropCount = Math.min(dropCount, groups.length - 1)

  // 5. 如果丢弃 group 0 后剩余以 assistant 消息开头（API 不接受），
  //    插入合成 user 标记：[earlier conversation truncated for compaction retry]
  if (sliced[0]?.type === 'assistant') {
    return [createUserMessage({ content: PTL_RETRY_MARKER, isMeta: true }), ...sliced]
  }
  return sliced
}
```

### 6.2 重试流程

```typescript
for (;;) {
  summaryResponse = await streamCompactSummary({ messages: messagesToSummarize, ... })
  summary = getAssistantMessageText(summaryResponse)

  if (!summary?.startsWith(PROMPT_TOO_LONG_ERROR_MESSAGE)) break  // 成功

  ptlAttempts++
  const truncated = ptlAttempts <= MAX_PTL_RETRIES
    ? truncateHeadForPTLRetry(messagesToSummarize, summaryResponse)
    : null

  if (!truncated) throw new Error(ERROR_MESSAGE_PROMPT_TOO_LONG)  // 超过重试次数

  // 同步更新 forkContextMessages，两条路径（forked agent 和直接流式）都使用截断后的消息
  messagesToSummarize = truncated
  retryCacheSafeParams = { ...retryCacheSafeParams, forkContextMessages: truncated }
}
```

---

## 7. Post-Compact 重注入

Compact 清空了上下文，需要将关键信息重新注入，让模型不会丢失重要状态。

### 7.1 文件重注入

```typescript
async function createPostCompactFileAttachments(
  readFileState: Record<string, { content: string; timestamp: number }>,
  maxFiles: number,
  preservedMessages: Message[] = [],  // partial compact 时已保留的消息
): Promise<AttachmentMessage[]>
```

**流程**：
1. 过滤掉已在 preservedMessages 中的文件（避免重复注入）
2. 过滤掉 plan 文件和所有 CLAUDE.md 内存文件（它们有专用的重注入通道）
3. 按 `timestamp` 降序排序（最近访问的优先）
4. 取前 `maxFiles = 5` 个
5. 对每个文件调用 `generateFileAttachment`（重新读取最新内容）
6. 累计 token 预算，超过 `POST_COMPACT_TOKEN_BUDGET = 50,000` 则丢弃

```
约束：最多 5 个文件，总计 ≤ 50K tokens，单文件 ≤ 5K tokens
```

### 7.2 Skills 重注入

```typescript
function createSkillAttachmentIfNeeded(agentId?: string): AttachmentMessage | null
```

- 按 `invokedAt` 降序排序（最近调用的优先）
- 每个 skill 内容截断至 `POST_COMPACT_MAX_TOKENS_PER_SKILL = 5,000` tokens
- 截断时保留文件头部（使用说明通常在开头）
- 截断标记：`[... skill content truncated; use Read on the skill path if needed]`
- 总预算 `POST_COMPACT_SKILLS_TOKEN_BUDGET = 25,000` tokens（约 5 个 skill）

### 7.3 其他重注入

| 附件类型 | 函数 | 说明 |
|---------|------|------|
| Plan 文件 | `createPlanAttachmentIfNeeded` | 当前任务计划 |
| Plan Mode 指令 | `createPlanModeAttachmentIfNeeded` | 在 plan mode 中时重注入指令 |
| 异步 Agent 状态 | `createAsyncAgentAttachmentsIfNeeded` | 后台运行或已完成的 agent |
| 工具 delta | `getDeferredToolsDeltaAttachment` | 重新宣告全量工具（diff against 空列表） |
| Agent 列表 delta | `getAgentListingDeltaAttachment` | 重新宣告可用 agents |
| MCP 工具 delta | `getMcpInstructionsDeltaAttachment` | 重新宣告 MCP 工具指令 |

**delta 重注入策略**：
- 全量 compact：diff against 空列表 `[]` → 输出完整的工具宣告
- partial compact：diff against `messagesToKeep` → 只宣告保留消息中未出现的工具

---

## 8. partialCompactConversation

除了全量 compact，还支持对指定位置进行部分压缩：

```typescript
async function partialCompactConversation(
  allMessages: Message[],
  pivotIndex: number,       // 分割点
  direction: 'from' | 'up_to' = 'from',
): Promise<CompactionResult>
```

### 8.1 'from' 方向（压缩尾部）
- 摘要：`allMessages[pivotIndex:]`
- 保留：`allMessages[:pivotIndex]`（保留头部，prompt cache 命中）
- 使用场景：压缩最近的一段工作，保留早期上下文

### 8.2 'up_to' 方向（压缩头部）
- 摘要：`allMessages[:pivotIndex]`
- 保留：`allMessages[pivotIndex:]`（保留尾部，最近上下文不变）
- 使用场景：压缩早期历史，保留最新内容
- 特殊处理：过滤保留消息中的旧 compact boundary 和旧摘要消息（防止嵌套摘要混乱）

### 8.3 缓存优化
```typescript
// 'up_to' 直接命中缓存（摘要的就是消息前缀）
apiMessages = direction === 'up_to' ? messagesToSummarize : allMessages
```

---

## 9. 消息结构

Compact 后的消息列表结构（由 `buildPostCompactMessages` 定义）：

```
[SystemCompactBoundaryMessage]   ← compact 边界标记（含元数据）
[UserMessage(isCompactSummary)]  ← 摘要内容（isVisibleInTranscriptOnly=true）
[messagesToKeep...]              ← partial compact 时保留的消息
[AttachmentMessage...]           ← 文件/skills/工具 附件
[HookResultMessage...]           ← SessionStart Hook 结果
```

**CompactBoundaryMessage 携带的元数据**：
- `trigger`: 'auto' | 'manual'
- `preCompactTokenCount`：压缩前 token 数
- `lastPreCompactMessageUuid`：最后一条原始消息的 UUID（用于历史导航）
- `preCompactDiscoveredTools`：已加载的 deferred tools 集合（压缩后继续发送）
- `preservedSegment`（partial compact）：`{headUuid, anchorUuid, tailUuid}`（用于消息链重建）

**摘要消息的特殊标记**：
- `isCompactSummary: true`：标识这是摘要
- `isVisibleInTranscriptOnly: true`：全量 compact 时不直接发给 API，只在 transcript 中显示
- `summarizeMetadata`（partial compact）：记录 `messagesSummarized`, `direction`, `userContext`

---

## 10. 关键常量与阈值

### 自动触发阈值
```typescript
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000  // 为摘要输出预留的 token
AUTOCOMPACT_BUFFER_TOKENS = 13_000      // 自动 compact 缓冲区
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000 // 警告阈值缓冲
ERROR_THRESHOLD_BUFFER_TOKENS = 20_000   // 错误阈值缓冲
MANUAL_COMPACT_BUFFER_TOKENS = 3_000     // 阻塞新消息的缓冲
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3 // 熔断阈值
```

### Post-Compact 重注入预算
```typescript
POST_COMPACT_MAX_FILES_TO_RESTORE = 5       // 最多恢复 5 个文件
POST_COMPACT_TOKEN_BUDGET = 50_000          // 文件总 token 预算
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000    // 单文件最大 token
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000   // Skills 总 token 预算
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000   // 单 skill 最大 token
```

### PTL 重试参数
```typescript
MAX_PTL_RETRIES = 3                        // 最多 3 次 PTL 重试
MAX_COMPACT_STREAMING_RETRIES = 2           // 流式生成最多 2 次重试
```

---

## 完整流程图

```
用户输入 / token 超阈值
         │
         ▼
  ┌──────────────────────────────────────────────┐
  │             触发判断                          │
  │  手动 /compact → commands/compact/compact.ts  │
  │  自动触发 → autoCompactIfNeeded()             │
  └──────────────────────────────────────────────┘
         │
         ▼ getMessagesAfterCompactBoundary()  只处理最后 boundary 后的消息
         │
  ┌──────┴───────────────────────────────────┐
  │ 路径 1：trySessionMemoryCompaction()       │
  │   无自定义指令 → 写 session memory 文件   │
  │   成功 → 返回 ✓                           │
  └──────┬───────────────────────────────────┘
         │ 失败
  ┌──────┴───────────────────────────────────┐
  │ 路径 2：reactiveCompactOnPromptTooLong()  │
  │   REACTIVE_COMPACT 功能开关              │
  │   从尾部逐步压缩                          │
  └──────┬───────────────────────────────────┘
         │ 未启用/失败
  ┌──────┴───────────────────────────────────────────────────────┐
  │ 路径 3：传统 compact                                          │
  │                                                              │
  │  microcompactMessages()  轻量预处理                           │
  │         │                                                    │
  │  compactConversation()                                       │
  │    1. executePreCompactHooks()                               │
  │    2. for(;;) streamCompactSummary()                         │
  │         ├─ A: runForkedAgent()        缓存共享（优先）         │
  │         │      └─ 系统提示与主线程相同 → 命中 prompt cache      │
  │         └─ B: queryModelWithStreaming()  直接流式（fallback）  │
  │                  └─ 工具集：[FileReadTool] 或 +ToolSearchTool │
  │    3. PTL 重试（for;;）                                       │
  │         └─ truncateHeadForPTLRetry()  丢弃最旧 API 轮次组     │
  │    4. readFileState.clear()  清空文件状态缓存                 │
  │    5. 并行生成附件（Promise.all）                             │
  │         ├─ createPostCompactFileAttachments()  最近 5 文件    │
  │         └─ createAsyncAgentAttachmentsIfNeeded()             │
  │    6. 附加 plan / plan_mode / skills / tools delta           │
  │    7. processSessionStartHooks('compact')                    │
  │    8. createCompactBoundaryMessage()                         │
  │    9. logEvent('tengu_compact', ...)                         │
  │   10. notifyCompaction() + markPostCompaction()              │
  │   11. reAppendSessionMetadata()                              │
  │   12. executePostCompactHooks()                              │
  │   13. return CompactionResult                                │
  └──────────────────────────────────────────────────────────────┘
         │
         ▼
  buildPostCompactMessages(result)
  ┌────────────────────────────────────────────┐
  │ [CompactBoundaryMarker]                    │
  │ [SummaryMessage(isCompactSummary=true)]    │
  │ [messagesToKeep...]  (partial compact 专用) │
  │ [FileAttachments...] (最近文件/plan/skills)  │
  │ [HookResultMessages...] (SessionStart 结果) │
  └────────────────────────────────────────────┘
```

---

## 设计要点

### 1. 缓存共享最大化降低成本
Forked agent 路径的核心价值：通过保持与主线程完全相同的系统提示参数，直接复用 Anthropic API 的 prompt cache。BQ 数据（2026-01）验证：98% 的 cache miss 集中在 ephemeral 环境（CCR/GHA/SDK），生产环境中大量命中缓存，节省了可观的 token 成本。

### 2. 两级安全退出
`streamCompactSummary` 的 fallback 设计确保 compact 不会因单一失败点而中断：forked agent 失败 → 直接流式 → 最多 2 次重试 → 最终抛出用户可见错误。

### 3. PTL 是紧急逃生通道
`truncateHeadForPTLRetry` 是"最后手段"（代码注释原文：last-resort escape hatch for CC-1180）。丢弃历史是有损的，但比用户永久卡死在超长对话中要好。Reactive compact 才是处理 PTL 的正确路径（从尾部裁剪，保留最新上下文）。

### 4. 状态清理的取舍
Compact 后只清空 `readFileState` 和 `loadedNestedMemoryPaths`，故意保留 `sentSkillNames`。原因：重注入完整 skill_listing 每次 compact 需要 ~4K tokens，而模型仍然知道 SkillTool 的存在（通过 invoked_skills 附件），没有必要全量重注入。

### 5. Token 预算的分层控制
Post-compact 重注入不是无限制的，每层都有精确预算：
- 文件：50K total / 5K per file / 最多 5 个
- Skills：25K total / 5K per skill
- 超预算的内容按优先级（最近访问/最近调用）截断，不是随机丢弃

### 6. 摘要消息对 API 不可见
`isVisibleInTranscriptOnly: true` 标记让摘要消息只出现在 transcript 文件中，不发给 Anthropic API。实际发给 API 的是重注入的附件，附件才是模型的上下文来源。这样设计是因为将长摘要作为 user message 发给 API 会干扰对话语义。
