# Claude Code 模型交互核心代码解析

> 分析日期: 2026-05-27
> 核心文件: `src/services/api/claude.ts` (3,489 行)
> 辅助文件: `src/query.ts`, `src/services/api/withRetry.ts`, `src/services/api/client.ts`

---

## 📋 目录

- [概述](#概述)
- [核心文件](#核心文件)
- [数据流架构](#数据流架构)
- [关键函数详解](#关键函数详解)
- [流式处理机制](#流式处理机制)
- [重试与容错](#重试与容错)
- [多云平台支持](#多云平台支持)
- [性能优化](#性能优化)
- [完整调用链路](#完整调用链路)

---

## 概述

Claude Code 与 Claude API 的交互核心集中在 **`src/services/api/claude.ts`** 文件中，这是一个 **3,489 行**的复杂模块，负责:

✅ **流式/非流式** API 调用
✅ **重试与容错** (自动降级)
✅ **多云平台** 支持 (Anthropic/Bedrock/Vertex/Foundry)
✅ **工具调用** 编排 (Tool Use)
✅ **上下文管理** (Prompt Caching, Context Management)
✅ **错误处理** (429/529 限流、超时、网络错误)
✅ **性能监控** (TTFT, 流式延迟检测)

---

## 核心文件

### 1. `src/services/api/claude.ts` (3,489 行)

**主要功能模块**:

| 模块 | 功能 | 关键函数 |
|------|------|----------|
| **流式调用** | 实时流式响应 | `queryModelWithStreaming()` |
| **核心查询** | 内部查询逻辑 | `queryModel()` |
| **非流式降级** | 流失败时降级 | `executeNonStreamingRequest()` |
| **客户端工厂** | 多平台客户端 | `getAnthropicClient()` |
| **API 验证** | API Key 验证 | `verifyApiKey()` |
| **参数构建** | 请求参数组装 | `buildAPIParams()` |
| **流事件处理** | 解析流事件 | 内部 switch case (line 2044) |

### 2. `src/query.ts` (1,730 行)

**查询主循环**:

```typescript
// 主入口
export async function* query(...): AsyncGenerator<StreamEvent> {
  yield* queryLoop(...);
}

// 主循环
async function* queryLoop(...): AsyncGenerator<StreamEvent> {
  while (true) {
    // 1. 发送消息到 API
    for await (const event of queryModelWithStreaming(...)) {
      yield event;
    }

    // 2. 执行工具调用
    if (assistantMessage.stop_reason === 'tool_use') {
      await runTools(...);
    }

    // 3. 检查停止条件
    if (shouldStop) break;
  }
}
```

### 3. `src/services/api/withRetry.ts` (29 KB)

**重试逻辑**:

```typescript
export async function* withRetry<T>(
  clientFactory: () => Anthropic,
  fn: (client: Anthropic) => Promise<T>,
  options: RetryOptions
): AsyncGenerator<SystemMessage | T> {
  let attempt = 0;
  while (attempt < maxRetries) {
    try {
      const client = clientFactory();
      const result = await fn(client);
      return result;
    } catch (error) {
      // 429/529: 速率限制错误
      if (is429or529Error(error)) {
        yield createSystemMessage('Rate limit hit, retrying...');
        await exponentialBackoff(attempt);
        attempt++;
        continue;
      }

      // 其他错误: 抛出
      throw error;
    }
  }
}
```

### 4. `src/services/api/client.ts` (18 KB)

**客户端创建**:

```typescript
export function getAnthropicClient(options: ClientOptions): Anthropic {
  const provider = getAPIProvider();

  switch (provider) {
    case 'anthropic':
      return new Anthropic({ apiKey, baseURL, ... });

    case 'bedrock':
      return new AnthropicBedrock({ awsAccessKey, awsSecretKey, ... });

    case 'vertex':
      return new AnthropicVertex({ projectId, region, ... });

    case 'foundry':
      return new AnthropicFoundry({ apiKey, baseURL, ... });

    case 'azureOpenAI':
      // 特殊处理: 使用 OpenAI SDK
      return createAzureOpenAIClient(...);

    default:
      throw new Error(`Unknown provider: ${provider}`);
  }
}
```

---

## 数据流架构

```
┌────────────────────────────────────────────────────────────────┐
│                        用户输入                                  │
│                    (Terminal/Desktop/IM)                        │
└──────────────────────┬─────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────┐
│                    src/query.ts                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  query() → queryLoop()                                   │  │
│  │  - 消息累积                                               │  │
│  │  - 工具执行编排                                           │  │
│  │  - 上下文管理                                             │  │
│  └──────────────────────┬───────────────────────────────────┘  │
└────────────────────────┼──────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│            src/services/api/claude.ts                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  queryModelWithStreaming()                               │  │
│  │    ↓                                                     │  │
│  │  queryModel()                                            │  │
│  │    ↓                                                     │  │
│  │  buildAPIParams() - 构建请求参数                         │  │
│  │    ↓                                                     │  │
│  │  withRetry() - 重试包装                                  │  │
│  └──────────────────────┬───────────────────────────────────┘  │
└────────────────────────┼──────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│            src/services/api/client.ts                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  getAnthropicClient()                                    │  │
│  │    ↓                                                     │  │
│  │  Anthropic / AnthropicBedrock / AnthropicVertex         │  │
│  └──────────────────────┬───────────────────────────────────┘  │
└────────────────────────┼──────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│                  Anthropic API                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  anthropic.beta.messages.create({ stream: true })        │  │
│  │    ↓                                                     │  │
│  │  Stream<BetaRawMessageStreamEvent>                       │  │
│  └──────────────────────┬───────────────────────────────────┘  │
└────────────────────────┼──────────────────────────────────────┘
                         │
                         ▼ (流式返回)
┌────────────────────────────────────────────────────────────────┐
│            流事件处理 (claude.ts line 2005)                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  for await (const part of stream) {                      │  │
│  │    switch (part.type) {                                  │  │
│  │      case 'message_start': ...                           │  │
│  │      case 'content_block_start': ...                     │  │
│  │      case 'content_block_delta': ...                     │  │
│  │      case 'message_delta': ...                           │  │
│  │      case 'message_stop': ...                            │  │
│  │    }                                                     │  │
│  │    yield streamEvent;                                    │  │
│  │  }                                                       │  │
│  └──────────────────────┬───────────────────────────────────┘  │
└────────────────────────┼──────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│                    返回给用户                                    │
│              (实时流式显示/工具执行)                              │
└────────────────────────────────────────────────────────────────┘
```

---

## 关键函数详解

### 1. `queryModelWithStreaming()` - 主入口

**位置**: `src/services/api/claude.ts:755`

```typescript
export async function* queryModelWithStreaming({
  messages,         // 对话历史
  systemPrompt,     // 系统 Prompt
  thinkingConfig,   // 思考配置
  tools,            // 可用工具列表
  signal,           // 中断信号
  options,          // 其他选项
}: {
  messages: Message[];
  systemPrompt: SystemPrompt;
  thinkingConfig: ThinkingConfig;
  tools: Tools;
  signal: AbortSignal;
  options: Options;
}): AsyncGenerator<
  StreamEvent | AssistantMessage | SystemAPIErrorMessage,
  void
> {
  // 包装 VCR 录制/回放 (用于测试)
  return yield* withStreamingVCR(messages, async function* () {
    yield* queryModel(
      messages,
      systemPrompt,
      thinkingConfig,
      tools,
      signal,
      options,
    );
  });
}
```

**功能**:
- 对外暴露的主流式 API
- 包装 VCR 用于录制/回放 (测试模式)
- 委托给 `queryModel()` 执行实际查询

---

### 2. `queryModel()` - 核心查询逻辑

**位置**: `src/services/api/claude.ts:1020`

**签名**:

```typescript
async function* queryModel(
  messages: Message[],
  systemPrompt: SystemPrompt,
  thinkingConfig: ThinkingConfig,
  tools: Tools,
  signal: AbortSignal,
  options: Options,
): AsyncGenerator<
  StreamEvent | AssistantMessage | SystemAPIErrorMessage,
  void
>
```

**主要流程**:

```typescript
async function* queryModel(...) {
  // 1. Azure OpenAI 特殊处理
  if (getAPIProvider() === 'azureOpenAI') {
    const result = await requestAzureOpenAI(...);
    yield assistantMessage;
    return;
  }

  // 2. Off-switch 检查 (Opus 模型限流开关)
  if (!isSubscriber && isOpusModel && offSwitchActivated) {
    yield errorMessage('Service temporarily unavailable');
    return;
  }

  // 3. 获取前一个请求 ID (用于追踪)
  const previousRequestId = getPreviousRequestIdFromMessages(messages);

  // 4. 构建 Beta 头
  const betas = getMergedBetas(model, { isAgenticQuery });

  // 5. Advisor 模型配置 (内部工具)
  if (isAdvisorEnabled()) {
    betas.push(ADVISOR_BETA_HEADER);
    advisorModel = getAdvisorModel();
  }

  // 6. 构建工具 Schema
  const toolsArray = Object.values(tools);
  const apiTools = toolsArray.map(tool => toolToAPISchema(tool));

  // 7. 构建 API 请求参数
  const apiParams = buildAPIParams({
    messages,
    systemPrompt,
    thinkingConfig,
    tools: apiTools,
    model,
    betas,
    ...options,
  });

  // 8. 包装重试逻辑
  yield* withRetry(
    () => getAnthropicClient({ model, source: 'query' }),
    async (anthropic) => {
      // 9. 创建流式请求
      const stream = await anthropic.beta.messages.create({
        ...apiParams,
        stream: true,
      });

      // 10. 处理流事件 (核心循环)
      yield* processStream(stream);
    },
    { maxRetries: 10, model, thinkingConfig }
  );
}
```

---

### 3. 流式事件处理循环

**位置**: `src/services/api/claude.ts:2005`

这是**整个系统最核心的部分** - 实时解析 Claude API 的流式响应。

```typescript
// 初始化状态
let partialMessage: BetaMessage | null = null;
let contentBlocks: BetaContentBlock[] = [];
let usage: BetaUsage | null = null;
let ttftMs: number | null = null; // Time to First Token
let isFirstChunk = true;
let stallCount = 0;
let lastEventTime: number | null = null;

// 核心流式循环
for await (const part of stream) {
  // 重置空闲计时器
  resetStreamIdleTimer();
  const now = Date.now();

  // 检测流式延迟 (超过 30 秒为 stall)
  if (lastEventTime !== null) {
    const timeSinceLastEvent = now - lastEventTime;
    if (timeSinceLastEvent > 30_000) {
      stallCount++;
      logEvent('tengu_streaming_stall', {
        stall_duration_ms: timeSinceLastEvent,
        stall_count: stallCount,
      });
    }
  }
  lastEventTime = now;

  // 首个 chunk 标记
  if (isFirstChunk) {
    ttftMs = Date.now() - start;
    isFirstChunk = false;
  }

  // 解析事件类型
  switch (part.type) {
    case 'message_start': {
      // 消息开始: 初始化部分消息
      partialMessage = part.message;
      usage = updateUsage(usage, part.message?.usage);
      break;
    }

    case 'content_block_start': {
      // 内容块开始
      switch (part.content_block.type) {
        case 'tool_use':
          // 工具调用块
          contentBlocks[part.index] = {
            ...part.content_block,
            input: '', // 初始化空输入
          };
          break;

        case 'text':
          // 文本块
          contentBlocks[part.index] = {
            ...part.content_block,
            text: '', // 初始化空文本
          };
          break;

        case 'thinking':
          // 思考块 (Extended Thinking)
          contentBlocks[part.index] = {
            ...part.content_block,
            thinking: '',
          };
          break;

        case 'server_tool_use':
          // 服务端工具 (Advisor)
          contentBlocks[part.index] = {
            ...part.content_block,
            input: {} as { [key: string]: unknown },
          };
          if (part.content_block.name === 'advisor') {
            isAdvisorInProgress = true;
            logEvent('tengu_advisor_tool_call', { model, advisor_model: advisorModel });
          }
          break;
      }
      break;
    }

    case 'content_block_delta': {
      // 内容块增量更新
      const existingBlock = contentBlocks[part.index];
      if (!existingBlock) break;

      switch (part.delta.type) {
        case 'text_delta':
          // 文本增量
          if (existingBlock.type === 'text') {
            existingBlock.text += part.delta.text;
          }
          break;

        case 'thinking_delta':
          // 思考增量
          if (existingBlock.type === 'thinking') {
            existingBlock.thinking += part.delta.thinking;
          }
          break;

        case 'input_json_delta':
          // 工具输入增量 (JSON 字符串)
          if (existingBlock.type === 'tool_use') {
            existingBlock.input += part.delta.partial_json;
          }
          break;

        case 'server_tool_delta':
          // 服务端工具增量
          if (existingBlock.type === 'server_tool_use') {
            // 累积 JSON 片段
            serverToolInputBuffer += part.delta.partial_json;
          }
          break;
      }
      break;
    }

    case 'content_block_stop': {
      // 内容块结束
      const block = contentBlocks[part.index];
      if (block?.type === 'tool_use') {
        // 解析完整的工具输入 JSON
        try {
          block.input = JSON.parse(block.input as string);
        } catch (e) {
          logError('Failed to parse tool input JSON', e);
        }
      }
      break;
    }

    case 'message_delta': {
      // 消息增量 (stop_reason, usage 更新)
      if (part.delta.stop_reason) {
        stopReason = part.delta.stop_reason;
      }
      usage = updateUsage(usage, part.usage);
      break;
    }

    case 'message_stop': {
      // 消息结束
      const finalMessage: BetaMessage = {
        id: partialMessage?.id ?? randomUUID(),
        model: options.model,
        role: 'assistant',
        content: contentBlocks,
        stop_reason: stopReason ?? 'end_turn',
        usage: usage ?? { input_tokens: 0, output_tokens: 0 },
      };

      // 生成最终 AssistantMessage
      const assistantMessage: AssistantMessage = {
        type: 'assistant',
        message: finalMessage,
        uuid: randomUUID(),
        timestamp: new Date().toISOString(),
        requestId: streamRequestId,
        ttftMs,
      };

      yield assistantMessage;
      return; // 流结束
    }

    case 'error': {
      // 错误事件
      logError('Stream error', part.error);
      yield createSystemAPIErrorMessage(part.error);
      return;
    }
  }

  // 实时 yield 流事件 (用于 UI 更新)
  yield {
    type: 'stream_event',
    event: part,
    contentBlocks: [...contentBlocks], // 快照
  } as StreamEvent;
}
```

**流事件类型**:

| 事件类型 | 触发时机 | 数据内容 |
|---------|---------|---------|
| `message_start` | 消息开始 | `message.id`, `usage` |
| `content_block_start` | 内容块开始 | `index`, `content_block` (类型: text/tool_use/thinking) |
| `content_block_delta` | 内容块增量 | `index`, `delta` (文本片段/JSON 片段) |
| `content_block_stop` | 内容块结束 | `index` |
| `message_delta` | 消息元数据更新 | `stop_reason`, `usage` |
| `message_stop` | 消息结束 | (无数据, 触发最终消息生成) |
| `error` | 流错误 | `error` 对象 |

**性能监控**:

- **TTFT (Time to First Token)**: 从请求发出到首个 token 返回的时间
- **Stall Detection**: 检测流式传输中的停顿 (超过 30 秒)
- **Token 统计**: 实时累积 input_tokens / output_tokens

---

### 4. `buildAPIParams()` - 构建请求参数

**功能**: 将内部消息格式转换为 Anthropic API 格式。

```typescript
function buildAPIParams({
  messages,
  systemPrompt,
  thinkingConfig,
  tools,
  model,
  betas,
  maxTokens,
  temperature,
  toolChoice,
  ...options
}: BuildAPIParamsInput): BetaMessageStreamParams {
  // 1. 规范化消息 (转换为 API 格式)
  const apiMessages = normalizeMessagesForAPI(messages);

  // 2. 构建系统 Prompt (可能包含缓存标记)
  const systemBlocks = buildSystemPromptBlocks(systemPrompt, {
    enableCaching: true,
  });

  // 3. 构建工具列表
  const apiTools = tools.map(tool => ({
    name: tool.name,
    description: tool.description,
    input_schema: tool.input_schema,
  }));

  // 4. 思考配置
  let thinkingParams = {};
  if (thinkingConfig.type === 'enabled') {
    thinkingParams = {
      thinking: {
        type: 'enabled',
        budget_tokens: thinkingConfig.budgetTokens,
      },
    };
  }

  // 5. 组装最终参数
  return {
    model,
    max_tokens: maxTokens ?? getModelMaxOutputTokens(model).default,
    messages: apiMessages,
    system: systemBlocks,
    tools: apiTools.length > 0 ? apiTools : undefined,
    tool_choice: toolChoice,
    temperature: temperature ?? 1,
    betas: betas.length > 0 ? betas : undefined,
    metadata: getAPIMetadata(),
    stream: true, // 始终使用流式
    ...thinkingParams,
    ...getExtraBodyParams(), // Bedrock 额外参数
  };
}
```

**关键转换**:

1. **消息格式**: 内部 `Message[]` → API `MessageParam[]`
2. **系统 Prompt**: 字符串数组 → `{ type: 'text', text: string, cache_control?: ... }[]`
3. **工具列表**: 内部 `Tool[]` → API `BetaToolUnion[]`
4. **缓存标记**: 自动在长系统 Prompt 上添加 `cache_control: { type: 'ephemeral' }`

---

### 5. `withRetry()` - 重试包装器

**位置**: `src/services/api/withRetry.ts`

```typescript
export async function* withRetry<T>(
  clientFactory: () => Anthropic,
  fn: (client: Anthropic) => Promise<T>,
  options: {
    maxRetries: number;
    model: string;
    thinkingConfig: ThinkingConfig;
    fallbackModel?: string;
    signal?: AbortSignal;
  }
): AsyncGenerator<SystemMessage | T> {
  let attempt = 0;
  let consecutive529Errors = 0;
  const startTime = Date.now();

  while (attempt < options.maxRetries) {
    try {
      // 创建客户端
      const client = clientFactory();

      // 执行请求
      const result = await fn(client);

      // 成功返回
      return result;

    } catch (error) {
      attempt++;

      // 1. 用户中断 (AbortError)
      if (error instanceof APIUserAbortError) {
        throw error; // 不重试
      }

      // 2. 速率限制 (429 / 529)
      if (is429or529Error(error)) {
        consecutive529Errors++;

        // 提取 Retry-After 头
        const retryAfter = extractRetryAfter(error);
        const waitTime = retryAfter ?? calculateExponentialBackoff(attempt);

        // 通知用户
        yield createSystemMessage(
          `Rate limit reached. Retrying in ${waitTime}s... (${attempt}/${maxRetries})`
        );

        // 等待
        await sleep(waitTime * 1000, options.signal);

        // 连续 3 次 529 错误: 降级到 fallback 模型
        if (consecutive529Errors >= 3 && options.fallbackModel) {
          yield createSystemMessage(
            `Switching to ${options.fallbackModel} due to repeated rate limits`
          );
          // 更新 clientFactory 使用 fallback 模型
          // ...
        }

        continue; // 重试
      }

      // 3. 超时错误 (APIConnectionTimeoutError)
      if (error instanceof APIConnectionTimeoutError) {
        if (attempt < maxRetries) {
          yield createSystemMessage(
            `Request timed out. Retrying... (${attempt}/${maxRetries})`
          );
          await sleep(1000 * attempt, options.signal);
          continue;
        }
      }

      // 4. 其他 API 错误
      if (error instanceof APIError) {
        // 某些错误不重试 (如 401 Unauthorized)
        if (isNonRetryableError(error)) {
          throw new CannotRetryError(error);
        }

        // 重试
        if (attempt < maxRetries) {
          yield createSystemMessage(
            `API error: ${error.message}. Retrying... (${attempt}/${maxRetries})`
          );
          await sleep(1000 * attempt, options.signal);
          continue;
        }
      }

      // 5. 重试次数耗尽
      throw new CannotRetryError(error);
    }
  }

  throw new Error('Max retries exceeded');
}
```

**重试策略**:

| 错误类型 | 重试次数 | 等待策略 | 降级策略 |
|---------|---------|---------|---------|
| **429 (Rate Limit)** | 10 次 | Retry-After 头 / 指数退避 | 连续 3 次 → Fallback 模型 |
| **529 (Overloaded)** | 10 次 | 同上 | 连续 3 次 → Fallback 模型 |
| **Timeout** | 10 次 | 线性退避 (attempt * 1s) | 流式 → 非流式 |
| **Network Error** | 10 次 | 指数退避 | - |
| **401 / 403** | 0 次 | 不重试 | - |

**指数退避算法**:

```typescript
function calculateExponentialBackoff(attempt: number): number {
  // Base: 2^attempt 秒
  const baseDelay = Math.pow(2, attempt);

  // 加入随机抖动 (0.5x - 1.5x)
  const jitter = 0.5 + Math.random();

  // 上限 60 秒
  return Math.min(baseDelay * jitter, 60);
}
```

---

## 流式处理机制

### 流式 vs 非流式

| 特性 | 流式 (Streaming) | 非流式 (Non-streaming) |
|------|------------------|----------------------|
| **响应方式** | 实时增量返回 | 完整响应一次返回 |
| **用户体验** | 打字机效果 | 等待后一次性显示 |
| **超时控制** | 30 分钟 idle timeout | 10 分钟总超时 |
| **重试策略** | 失败后降级到非流式 | 失败后报错 |
| **Prompt Caching** | 支持 | 支持 |
| **思考显示** | 实时显示思考过程 | 不显示思考 |

### 流式降级机制

当流式请求失败时,自动降级到非流式:

```typescript
try {
  // 1. 尝试流式
  const stream = await anthropic.beta.messages.create({
    ...params,
    stream: true,
  });

  yield* processStream(stream);

} catch (error) {
  // 2. 流式失败,降级到非流式
  if (shouldFallbackToNonStreaming(error)) {
    logEvent('tengu_streaming_fallback', { error: error.message });

    yield createSystemMessage('Streaming failed, switching to non-streaming...');

    const response = await anthropic.beta.messages.create({
      ...params,
      stream: false, // 非流式
    });

    yield convertToAssistantMessage(response);
  } else {
    throw error;
  }
}
```

**降级触发条件**:

- 流初始化失败 (连接超时)
- 流中断 (网络波动)
- 流解析错误 (协议错误)

---

## 重试与容错

### 错误分类

```typescript
// 1. 可重试错误
const RETRYABLE_ERRORS = [
  'APIConnectionError',
  'APIConnectionTimeoutError',
  'InternalServerError',
  'RateLimitError', // 429
  'OverloadedError', // 529
];

// 2. 不可重试错误
const NON_RETRYABLE_ERRORS = [
  'AuthenticationError', // 401
  'PermissionDeniedError', // 403
  'NotFoundError', // 404
  'InvalidRequestError', // 400
];

// 3. 需要特殊处理的错误
const SPECIAL_ERRORS = {
  'PromptTooLongError': 'context_overflow', // 上下文超长
  'ToolValidationError': 'tool_error', // 工具参数错误
};
```

### 降级策略

```typescript
// 1. 模型降级
Opus 4.6 (529 x3) → Sonnet 4.6

// 2. 流式降级
Streaming (失败) → Non-streaming

// 3. 工具降级
Full Tool Schema (失败) → Simplified Schema

// 4. 上下文降级
Full Context (超长) → Auto Compact → Manual Compact
```

### 超时控制

| 场景 | 超时设置 | 说明 |
|------|---------|------|
| **流式请求** | 30 分钟 idle | 30 秒无数据则判定为 stall |
| **非流式请求** | 5 分钟 | 总超时时间 |
| **远程会话** | 2 分钟 | CCR 环境降低超时避免容器 kill |
| **API Key 验证** | 30 秒 | 快速验证 |

```typescript
// 流式空闲超时
const STREAM_IDLE_TIMEOUT_MS = 30 * 60 * 1000; // 30 分钟

// 非流式总超时
const NON_STREAMING_TIMEOUT_MS = 5 * 60 * 1000; // 5 分钟

// 远程会话超时
const REMOTE_TIMEOUT_MS = 2 * 60 * 1000; // 2 分钟
```

---

## 多云平台支持

### 支持的平台

| 平台 | SDK | 认证方式 | 特殊配置 |
|------|-----|---------|---------|
| **Anthropic** | `@anthropic-ai/sdk` | API Key | baseURL, Custom Headers |
| **AWS Bedrock** | `@anthropic-ai/bedrock-sdk` | AWS Credentials | Region, Cross-region inference |
| **Google Vertex AI** | `@anthropic-ai/vertex-sdk` | GCP Service Account | Project ID, Region |
| **Azure Foundry** | Custom adapter | API Key | Foundry Endpoint |
| **Azure OpenAI** | `openai` SDK | API Key | 特殊消息格式转换 |

### 客户端创建逻辑

```typescript
export function getAnthropicClient(options: ClientOptions): Anthropic {
  const provider = getAPIProvider(); // 从环境变量读取

  switch (provider) {
    case 'anthropic': {
      const apiKey = process.env.ANTHROPIC_API_KEY;
      const baseURL = process.env.ANTHROPIC_BASE_URL || 'https://api.anthropic.com';

      return new Anthropic({
        apiKey,
        baseURL,
        maxRetries: 0, // 自定义重试逻辑
        fetch: options.fetchOverride, // 代理支持
        defaultHeaders: {
          ...getAttributionHeader(),
          ...getExtraHeaders(),
        },
      });
    }

    case 'bedrock': {
      const awsAccessKey = process.env.AWS_ACCESS_KEY_ID;
      const awsSecretKey = process.env.AWS_SECRET_ACCESS_KEY;
      const awsRegion = process.env.AWS_REGION || 'us-east-1';

      return new AnthropicBedrock({
        awsAccessKey,
        awsSecretKey,
        awsRegion,
      });
    }

    case 'vertex': {
      const projectId = process.env.GCP_PROJECT_ID;
      const region = process.env.GCP_REGION || 'us-central1';

      return new AnthropicVertex({
        projectId,
        region,
        // 使用 Application Default Credentials
      });
    }

    case 'foundry': {
      const apiKey = process.env.AZURE_FOUNDRY_API_KEY;
      const baseURL = process.env.AZURE_FOUNDRY_ENDPOINT;

      return new AnthropicFoundry({
        apiKey,
        baseURL,
      });
    }

    case 'azureOpenAI': {
      // 特殊处理: 使用 OpenAI SDK
      const apiKey = process.env.AZURE_OPENAI_API_KEY;
      const endpoint = process.env.AZURE_OPENAI_ENDPOINT;
      const deployment = process.env.AZURE_OPENAI_DEPLOYMENT;

      return createAzureOpenAIClient({
        apiKey,
        endpoint,
        deployment,
        apiVersion: '2024-02-01',
      });
    }

    default:
      throw new Error(`Unsupported provider: ${provider}`);
  }
}
```

### 环境变量配置

```bash
# Anthropic API
export ANTHROPIC_API_KEY="sk-ant-..."
export ANTHROPIC_BASE_URL="https://api.anthropic.com" # 可选

# AWS Bedrock
export API_PROVIDER="bedrock"
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"

# Google Vertex AI
export API_PROVIDER="vertex"
export GCP_PROJECT_ID="my-project"
export GCP_REGION="us-central1"
# 或使用 GOOGLE_APPLICATION_CREDENTIALS

# Azure Foundry
export API_PROVIDER="foundry"
export AZURE_FOUNDRY_API_KEY="..."
export AZURE_FOUNDRY_ENDPOINT="https://..."

# Azure OpenAI
export API_PROVIDER="azureOpenAI"
export AZURE_OPENAI_API_KEY="..."
export AZURE_OPENAI_ENDPOINT="https://...openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT="gpt-4"
```

---

## 性能优化

### 1. Prompt Caching

自动在长系统 Prompt 上添加缓存标记:

```typescript
function buildSystemPromptBlocks(
  systemPrompt: SystemPrompt,
  options: { enableCaching: boolean }
): Array<{ type: 'text'; text: string; cache_control?: { type: 'ephemeral' } }> {
  const blocks = systemPrompt.map((text, index) => {
    const isLastBlock = index === systemPrompt.length - 1;

    // 最后一个块添加缓存标记
    if (isLastBlock && options.enableCaching) {
      return {
        type: 'text',
        text,
        cache_control: { type: 'ephemeral' },
      };
    }

    return {
      type: 'text',
      text,
    };
  });

  return blocks;
}
```

**缓存效果**:

- 📊 **Cache Hit Rate**: 70-90% (根据 BQ 数据)
- 💰 **成本节省**: 90% (缓存命中的 input tokens 只收 10%)
- ⚡ **延迟降低**: 30-50% (避免重新处理长 Prompt)

### 2. 连接复用

```typescript
// 全局客户端缓存
const clientCache = new Map<string, Anthropic>();

export function getAnthropicClient(options: ClientOptions): Anthropic {
  const cacheKey = `${options.model}-${options.source}`;

  // 复用已有客户端
  if (clientCache.has(cacheKey)) {
    return clientCache.get(cacheKey)!;
  }

  // 创建新客户端
  const client = createClient(options);
  clientCache.set(cacheKey, client);

  return client;
}
```

### 3. 流式性能监控

```typescript
// 监控指标
const metrics = {
  ttft: null,           // Time to First Token
  stallCount: 0,        // 流式停顿次数
  totalStallTime: 0,    // 总停顿时间
  throughput: 0,        // 吞吐量 (tokens/s)
};

// TTFT 记录
if (isFirstChunk) {
  metrics.ttft = Date.now() - requestStartTime;
  logEvent('tengu_ttft', { ttft_ms: metrics.ttft, model });
}

// Stall 检测
if (timeSinceLastEvent > 30_000) {
  metrics.stallCount++;
  metrics.totalStallTime += timeSinceLastEvent;
  logEvent('tengu_streaming_stall', {
    stall_duration_ms: timeSinceLastEvent,
    stall_count: metrics.stallCount,
  });
}

// 吞吐量计算
const duration = Date.now() - requestStartTime;
metrics.throughput = (outputTokens / duration) * 1000; // tokens/s
```

### 4. 批量工具调用

工具调用支持并行执行:

```typescript
// 并行执行多个工具
const toolResults = await Promise.all(
  toolUses.map(async (toolUse) => {
    try {
      const result = await executeTool(toolUse);
      return result;
    } catch (error) {
      return createToolErrorResult(toolUse.id, error);
    }
  })
);
```

---

## 完整调用链路

### 示例: 用户发送 "读取 README.md"

```
1. 用户输入: "读取 README.md"
   ↓
2. src/screens/REPL.tsx
   - 捕获用户输入
   - 创建 UserMessage
   ↓
3. src/query.ts: query()
   - 累积消息: [...history, userMessage]
   - 调用 queryLoop()
   ↓
4. src/query.ts: queryLoop()
   - 调用 queryModelWithStreaming()
   ↓
5. src/services/api/claude.ts: queryModelWithStreaming()
   - 包装 VCR (测试模式)
   - 调用 queryModel()
   ↓
6. src/services/api/claude.ts: queryModel()
   - 构建系统 Prompt
   - 构建工具列表 (包含 FileReadTool)
   - 规范化消息
   - 调用 buildAPIParams()
   ↓
7. src/services/api/claude.ts: buildAPIParams()
   - 转换消息格式
   - 添加缓存标记
   - 构建最终参数
   ↓
8. src/services/api/withRetry.ts: withRetry()
   - 包装重试逻辑
   - 调用 getAnthropicClient()
   ↓
9. src/services/api/client.ts: getAnthropicClient()
   - 根据 provider 创建客户端
   - 返回 Anthropic 实例
   ↓
10. Anthropic SDK: anthropic.beta.messages.create()
    - 发送 HTTP POST 到 https://api.anthropic.com/v1/messages
    - 请求体:
      {
        model: "claude-sonnet-4-6",
        max_tokens: 8192,
        messages: [
          { role: "user", content: "读取 README.md" }
        ],
        system: [...],
        tools: [
          {
            name: "Read",
            description: "Reads a file from the local filesystem.",
            input_schema: { ... }
          },
          ...
        ],
        stream: true
      }
    ↓
11. Claude API 响应 (流式):
    - message_start: { id: "msg_...", usage: { input_tokens: 12584 } }
    - content_block_start: { index: 0, content_block: { type: "thinking" } }
    - content_block_delta: { index: 0, delta: { type: "thinking_delta", thinking: "用户想读取..." } }
    - content_block_stop: { index: 0 }
    - content_block_start: { index: 1, content_block: { type: "tool_use", name: "Read" } }
    - content_block_delta: { index: 1, delta: { type: "input_json_delta", partial_json: '{"file' } }
    - content_block_delta: { index: 1, delta: { type: "input_json_delta", partial_json: '_path":' } }
    - content_block_delta: { index: 1, delta: { type: "input_json_delta", partial_json: '"/path/to/README.md"}' } }
    - content_block_stop: { index: 1 }
    - message_delta: { delta: { stop_reason: "tool_use" }, usage: { output_tokens: 156 } }
    - message_stop: {}
    ↓
12. src/services/api/claude.ts: processStream()
    - 解析流事件
    - 累积 content_blocks
    - 生成 AssistantMessage
    - yield AssistantMessage
    ↓
13. src/query.ts: queryLoop()
    - 检测 stop_reason === 'tool_use'
    - 提取 tool_use: { name: "Read", input: { file_path: "/path/to/README.md" } }
    - 调用 runTools()
    ↓
14. src/services/tools/toolOrchestration.ts: runTools()
    - 查找 FileReadTool
    - 检查权限
    - 调用 FileReadTool.call()
    ↓
15. src/tools/FileReadTool/FileReadTool.ts: call()
    - 验证输入
    - 读取文件
    - 返回 { type: 'text', file: { content: "...", numLines: 42 } }
    ↓
16. src/query.ts: queryLoop()
    - 创建 ToolResultMessage
    - 添加到消息历史: [...history, assistantMessage, toolResultMessage]
    - 再次调用 queryModelWithStreaming()
    ↓
17. Claude API 第二轮响应:
    - content_block_start: { index: 0, content_block: { type: "text" } }
    - content_block_delta: { index: 0, delta: { type: "text_delta", text: "我已经读取了 README.md..." } }
    - message_delta: { delta: { stop_reason: "end_turn" } }
    - message_stop: {}
    ↓
18. src/query.ts: queryLoop()
    - stop_reason === 'end_turn'
    - 退出循环
    ↓
19. src/screens/REPL.tsx
    - 渲染最终 AssistantMessage
    - 显示给用户
```

---

## 总结

### 核心设计原则

1. **流式优先**: 默认使用流式响应,提供实时反馈
2. **自动降级**: 流式失败自动切换非流式
3. **智能重试**: 429/529 错误自动重试,连续失败则切换模型
4. **多云兼容**: 统一接口支持 5 个平台
5. **性能监控**: TTFT、Stall、Throughput 全面监控
6. **容错设计**: 超时、中断、错误多层处理

### 关键数据

| 指标 | 数值 | 说明 |
|------|------|------|
| **核心文件行数** | 3,489 | claude.ts |
| **流事件类型** | 7 种 | message_start ~ message_stop |
| **重试次数** | 10 | 默认最大重试 |
| **流式超时** | 30 分钟 | idle timeout |
| **非流式超时** | 5 分钟 | total timeout |
| **支持平台** | 5 个 | Anthropic/Bedrock/Vertex/Foundry/Azure |
| **缓存命中率** | 70-90% | Prompt Caching |

### 扩展方向

1. **更多模型**: Gemini, GPT-4, 本地模型
2. **更多工具**: 数据库、Docker、K8s
3. **更智能重试**: 基于错误类型的动态策略
4. **更细粒度监控**: 每个工具的性能追踪
5. **更好的缓存**: 工具结果缓存、消息去重

这套架构展示了如何构建一个**生产级的 AI Agent 系统**，具有高可用性、高性能和良好的用户体验！ 🚀
