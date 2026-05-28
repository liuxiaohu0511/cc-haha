# Claude Code 如何支持 DeepSeek 模型

> 分析日期: 2026-05-27
> 核心机制: 协议转换代理 + 原生 Anthropic 协议支持
> 相关文件: `src/server/proxy/`, `src/server/config/providerPresets.json`

---

## 📋 目录

- [概述](#概述)
- [支持方式](#支持方式)
- [DeepSeek 配置详解](#deepseek-配置详解)
- [协议转换机制](#协议转换机制)
- [请求转换流程](#请求转换流程)
- [响应转换流程](#响应转换流程)
- [流式处理](#流式处理)
- [配置方式](#配置方式)
- [完整示例](#完整示例)

---

## 概述

Claude Code 支持 DeepSeek 的方式有 **两种**:

### 方式一: 原生 Anthropic 协议 (推荐 ⭐)

DeepSeek 官方提供了 **Anthropic 兼容接口**，可以直接使用，无需协议转换:

```
Claude Code → Anthropic Protocol → DeepSeek API (https://api.deepseek.com/anthropic)
```

**优势**:
- ✅ **零配置**: 直接使用，无需额外代理
- ✅ **原生支持**: DeepSeek 官方维护
- ✅ **功能完整**: 支持工具调用、流式响应、思考链
- ✅ **性能最优**: 无中间层转换

### 方式二: 协议转换代理

通过内置代理或 LiteLLM 将 Anthropic 协议转换为 OpenAI 协议:

```
Claude Code → Anthropic Protocol → Proxy → OpenAI Protocol → DeepSeek API
```

**适用场景**:
- 使用不支持 Anthropic 协议的旧版 DeepSeek API
- 需要统一管理多个不同协议的模型
- 使用第三方代理服务 (如 LiteLLM)

---

## 支持方式

### 1. DeepSeek 官方 Anthropic 接口 (推荐)

DeepSeek 在 `https://api.deepseek.com/anthropic` 提供了完整的 Anthropic Messages API 兼容接口。

**内置预设配置** (`src/server/config/providerPresets.json`):

```json
{
  "id": "deepseek",
  "name": "DeepSeek",
  "baseUrl": "https://api.deepseek.com/anthropic",
  "apiFormat": "anthropic",
  "defaultModels": {
    "main": "deepseek-v4-pro",
    "haiku": "deepseek-v4-flash",
    "sonnet": "deepseek-v4-pro",
    "opus": "deepseek-v4-pro"
  },
  "needsApiKey": true,
  "websiteUrl": "https://platform.deepseek.com",
  "apiKeyUrl": "https://platform.deepseek.com/api_keys",
  "authStrategy": "auth_token",
  "defaultEnv": {
    "ANTHROPIC_DEFAULT_HAIKU_MODEL_SUPPORTED_CAPABILITIES": "thinking,effort,adaptive_thinking,max_effort",
    "ANTHROPIC_DEFAULT_SONNET_MODEL_SUPPORTED_CAPABILITIES": "thinking,effort,adaptive_thinking,max_effort",
    "ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES": "thinking,effort,adaptive_thinking,max_effort"
  },
  "modelContextWindows": {
    "deepseek-v4-pro": 1000000,
    "deepseek-v4-flash": 1000000,
    "deepseek-chat": 1000000,
    "deepseek-reasoner": 1000000
  }
}
```

**关键字段解析**:

| 字段 | 值 | 说明 |
|------|-----|------|
| `apiFormat` | `"anthropic"` | 使用 Anthropic 协议，**无需转换** |
| `baseUrl` | `https://api.deepseek.com/anthropic` | DeepSeek 官方 Anthropic 端点 |
| `authStrategy` | `"auth_token"` | 使用 `Authorization: Bearer <token>` 认证 |
| `modelContextWindows` | `{ "deepseek-v4-pro": 1000000 }` | 上下文窗口 1M tokens |
| `defaultEnv.SUPPORTED_CAPABILITIES` | `"thinking,effort,..."` | 支持思考链、Effort 等特性 |

**支持的功能**:

- ✅ **工具调用** (Tool Use)
- ✅ **流式响应** (Streaming)
- ✅ **思考链** (Thinking / Reasoning)
- ✅ **Effort 控制** (Low/Medium/High)
- ✅ **百万级上下文** (1M tokens)

---

### 2. 内置协议转换代理

对于不支持 Anthropic 协议的 API，Claude Code 内置了协议转换代理。

**架构**:

```
src/server/proxy/
├── handler.ts                           # 代理主处理器
├── transform/
│   ├── anthropicToOpenaiChat.ts         # Anthropic → OpenAI Chat
│   ├── openaiChatToAnthropic.ts         # OpenAI Chat → Anthropic
│   ├── anthropicToOpenaiResponses.ts    # Anthropic → OpenAI Responses
│   └── openaiResponsesToAnthropic.ts    # OpenAI Responses → Anthropic
└── streaming/
    ├── openaiChatStreamToAnthropic.ts   # 流式转换 (Chat)
    └── openaiResponsesStreamToAnthropic.ts  # 流式转换 (Responses)
```

**支持的 API 格式**:

| 格式 | 端点 | 适用模型 |
|------|------|----------|
| `anthropic` | `/v1/messages` | Anthropic, DeepSeek, Kimi, MiniMax, GLM |
| `openai_chat` | `/v1/chat/completions` | OpenAI, DeepSeek (旧), Ollama, 本地模型 |
| `openai_responses` | `/v1/responses` | Azure OpenAI Responses API |

---

## DeepSeek 配置详解

### 支持的模型

| 模型 ID | 用途 | 上下文窗口 | 特性 |
|---------|------|------------|------|
| `deepseek-v4-pro` | 主力模型 | 1M tokens | 思考链、工具调用、高质量 |
| `deepseek-v4-flash` | 快速模型 | 1M tokens | 高速响应、低成本 |
| `deepseek-chat` | 通用对话 | 1M tokens | 标准对话模型 |
| `deepseek-reasoner` | 推理专用 | 1M tokens | 强化推理能力 |

### 模型映射策略

Claude Code 将 Claude 模型映射到 DeepSeek 模型:

```json
{
  "main": "deepseek-v4-pro",      // 默认模型
  "haiku": "deepseek-v4-flash",   // 快速、低成本 → Flash
  "sonnet": "deepseek-v4-pro",    // 平衡 → Pro
  "opus": "deepseek-v4-pro"       // 高质量 → Pro
}
```

**使用场景**:

- `/model` 命令选择 `haiku` → 实际使用 `deepseek-v4-flash`
- 系统内部调用 `getDefaultSonnetModel()` → 返回 `deepseek-v4-pro`

### 能力配置

```json
{
  "defaultEnv": {
    "ANTHROPIC_DEFAULT_HAIKU_MODEL_SUPPORTED_CAPABILITIES": "thinking,effort,adaptive_thinking,max_effort",
    "ANTHROPIC_DEFAULT_SONNET_MODEL_SUPPORTED_CAPABILITIES": "thinking,effort,adaptive_thinking,max_effort",
    "ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES": "thinking,effort,adaptive_thinking,max_effort"
  }
}
```

**能力标志位**:

| 能力 | 说明 |
|------|------|
| `thinking` | 支持思考链 (Extended Thinking) |
| `effort` | 支持 Effort 控制 (Low/Medium/High) |
| `adaptive_thinking` | 支持自适应思考预算 |
| `max_effort` | 支持最大 Effort |

---

## 协议转换机制

当使用不支持 Anthropic 协议的 DeepSeek API 时，内置代理会执行协议转换。

### 转换逻辑

#### 1. 请求转换: Anthropic → OpenAI

**文件**: `src/server/proxy/transform/anthropicToOpenaiChat.ts`

```typescript
export function anthropicToOpenaiChat(
  body: AnthropicRequest,
  options: { roundTripReasoningContent?: boolean; passThinkingToggle?: boolean } = {},
): OpenAIChatRequest {
  const messages: OpenAIChatMessage[] = [];

  // 1. 转换系统 Prompt
  if (body.system) {
    if (typeof body.system === 'string') {
      messages.push({ role: 'system', content: body.system });
    } else if (Array.isArray(body.system)) {
      const text = body.system.map((b) => b.text).join('\n');
      messages.push({ role: 'system', content: text });
    }
  }

  // 2. 转换消息
  for (const msg of body.messages) {
    convertMessage(msg, messages, options);
  }

  // 3. 构建请求
  const result: OpenAIChatRequest = {
    model: body.model,
    messages,
    stream: body.stream,
  };

  // 4. 转换参数
  if (body.temperature !== undefined) result.temperature = body.temperature;
  if (body.top_p !== undefined) result.top_p = body.top_p;
  if (body.stop_sequences && body.stop_sequences.length > 0) {
    result.stop = body.stop_sequences;
  }

  // 5. 转换工具
  if (body.tools && body.tools.length > 0) {
    result.tools = body.tools
      .filter((t) => t.name !== 'BatchTool')
      .map((t): OpenAITool => ({
        type: 'function',
        function: {
          name: t.name,
          description: t.description,
          parameters: t.input_schema,
        },
      }));
  }

  // 6. 转换 tool_choice
  if (body.tool_choice !== undefined) {
    result.tool_choice = convertToolChoice(body.tool_choice);
  }

  // 7. 转换思考配置: thinking → reasoning_effort
  if (body.thinking) {
    const budget = body.thinking.budget_tokens;
    if (budget !== undefined) {
      if (budget <= 1024) result.reasoning_effort = 'low';
      else if (budget <= 8192) result.reasoning_effort = 'medium';
      else result.reasoning_effort = 'high';
    } else if (body.thinking.type === 'enabled') {
      result.reasoning_effort = 'high';
    }
  }

  return result;
}
```

**关键转换点**:

| Anthropic 字段 | OpenAI 字段 | 转换逻辑 |
|----------------|-------------|----------|
| `system` (string/array) | `messages[0] = { role: 'system' }` | 系统消息作为首条 |
| `messages` | `messages` | 逐条转换 |
| `stop_sequences` | `stop` | 直接映射 |
| `tools[].input_schema` | `tools[].function.parameters` | 工具格式转换 |
| `tool_choice` | `tool_choice` | `auto` / `any` / `{name}` 转换 |
| `thinking.budget_tokens` | `reasoning_effort` | Token 预算 → 等级 (low/medium/high) |

#### 2. 响应转换: OpenAI → Anthropic

**文件**: `src/server/proxy/transform/openaiChatToAnthropic.ts`

```typescript
export function openaiChatToAnthropic(response: OpenAIChatResponse, model: string): AnthropicResponse {
  const choice = response.choices?.[0];
  if (!choice) {
    return createEmptyResponse(response, model);
  }

  const content: AnthropicContentBlock[] = [];

  // 1. 转换推理/思考内容 (多种格式兼容)
  const msg = choice.message as Record<string, unknown>;

  // Format 1: reasoning_content (DeepSeek, OpenRouter, XAI, Perplexity)
  if (typeof msg.reasoning_content === 'string' && msg.reasoning_content) {
    content.push({ type: 'thinking', thinking: msg.reasoning_content });
  }
  // Format 2: reasoning (GLM-5, Cerebras, Groq)
  else if (typeof msg.reasoning === 'string' && msg.reasoning) {
    content.push({ type: 'thinking', thinking: msg.reasoning });
  }
  // Format 3: thinking_blocks (OpenAI o-series)
  else if (Array.isArray(msg.thinking_blocks)) {
    for (const tb of msg.thinking_blocks as Array<Record<string, unknown>>) {
      if (tb.type === 'thinking' && typeof tb.thinking === 'string') {
        content.push({
          type: 'thinking',
          thinking: tb.thinking,
          signature: tb.signature as string | undefined
        });
      }
    }
  }

  // 2. 转换文本内容
  if (choice.message.content) {
    content.push({ type: 'text', text: choice.message.content });
  }

  // 3. 转换工具调用
  if (choice.message.tool_calls) {
    for (const tc of choice.message.tool_calls) {
      content.push({
        type: 'tool_use',
        id: tc.id,
        name: tc.function.name,
        input: parseOpenAIToolArguments(tc.function.arguments),
      });
    }
  }

  // 4. 返回 Anthropic 格式响应
  return {
    id: response.id || `msg_${Date.now()}`,
    type: 'message',
    role: 'assistant',
    content,
    model: response.model || model,
    stop_reason: mapFinishReason(choice.finish_reason),
    stop_sequence: null,
    usage: mapUsage(response.usage),
  };
}

// 映射停止原因
function mapFinishReason(reason: string | null): string {
  switch (reason) {
    case 'stop': return 'end_turn';
    case 'tool_calls': return 'tool_use';
    case 'length': return 'max_tokens';
    case 'content_filter': return 'end_turn';
    default: return 'end_turn';
  }
}

// 映射 token 使用量
function mapUsage(usage?: OpenAIChatResponse['usage']): AnthropicResponse['usage'] {
  if (!usage) {
    return { input_tokens: 0, output_tokens: 0 };
  }
  return {
    input_tokens: usage.prompt_tokens || 0,
    output_tokens: usage.completion_tokens || 0,
    cache_read_input_tokens: usage.prompt_tokens_details?.cached_tokens || 0,
  };
}
```

**关键转换点**:

| OpenAI 字段 | Anthropic 字段 | 转换逻辑 |
|-------------|----------------|----------|
| `choices[0].message.content` | `content[].type='text'` | 文本块 |
| `choices[0].message.reasoning_content` | `content[].type='thinking'` | 思考块 (DeepSeek) |
| `choices[0].message.tool_calls[]` | `content[].type='tool_use'` | 工具调用 |
| `finish_reason: 'stop'` | `stop_reason: 'end_turn'` | 停止原因映射 |
| `finish_reason: 'tool_calls'` | `stop_reason: 'tool_use'` | 工具调用结束 |
| `usage.prompt_tokens` | `usage.input_tokens` | Token 统计 |

---

## 请求转换流程

### 完整示例

**原始 Anthropic 请求**:

```json
{
  "model": "deepseek-v4-pro",
  "max_tokens": 8192,
  "messages": [
    {
      "role": "user",
      "content": "什么是量子计算?"
    }
  ],
  "system": [
    { "type": "text", "text": "你是一个有帮助的 AI 助手。" }
  ],
  "tools": [
    {
      "name": "Read",
      "description": "读取文件",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": { "type": "string" }
        }
      }
    }
  ],
  "tool_choice": { "type": "auto" },
  "thinking": {
    "type": "enabled",
    "budget_tokens": 10000
  },
  "stream": true
}
```

**转换后的 OpenAI 请求**:

```json
{
  "model": "deepseek-v4-pro",
  "messages": [
    {
      "role": "system",
      "content": "你是一个有帮助的 AI 助手。"
    },
    {
      "role": "user",
      "content": "什么是量子计算?"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "Read",
        "description": "读取文件",
        "parameters": {
          "type": "object",
          "properties": {
            "file_path": { "type": "string" }
          }
        }
      }
    }
  ],
  "tool_choice": "auto",
  "reasoning_effort": "high",
  "stream": true
}
```

**关键变化**:

1. ✅ `system` 数组 → `messages[0]` 系统消息
2. ✅ `tools[].input_schema` → `tools[].function.parameters`
3. ✅ `tool_choice: { type: 'auto' }` → `tool_choice: 'auto'`
4. ✅ `thinking.budget_tokens: 10000` → `reasoning_effort: 'high'`
5. ❌ `max_tokens` 被省略 (避免超出 DeepSeek 8192 限制)

---

## 响应转换流程

### 完整示例

**DeepSeek OpenAI 响应**:

```json
{
  "id": "chatcmpl-123",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "deepseek-v4-pro",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "量子计算是一种利用量子力学原理进行信息处理的新型计算方式...",
        "reasoning_content": "用户询问量子计算的定义。我需要用简洁易懂的语言解释这个复杂的概念...",
        "tool_calls": null
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 150,
    "completion_tokens": 200,
    "total_tokens": 350
  }
}
```

**转换后的 Anthropic 响应**:

```json
{
  "id": "chatcmpl-123",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "thinking",
      "thinking": "用户询问量子计算的定义。我需要用简洁易懂的语言解释这个复杂的概念..."
    },
    {
      "type": "text",
      "text": "量子计算是一种利用量子力学原理进行信息处理的新型计算方式..."
    }
  ],
  "model": "deepseek-v4-pro",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 150,
    "output_tokens": 200,
    "cache_read_input_tokens": 0
  }
}
```

**关键变化**:

1. ✅ `choices[0].message.reasoning_content` → `content[0] = { type: 'thinking' }`
2. ✅ `choices[0].message.content` → `content[1] = { type: 'text' }`
3. ✅ `finish_reason: 'stop'` → `stop_reason: 'end_turn'`
4. ✅ `usage.prompt_tokens` → `usage.input_tokens`
5. ✅ `usage.completion_tokens` → `usage.output_tokens`

---

## 流式处理

### 流式转换器

**文件**: `src/server/proxy/streaming/openaiChatStreamToAnthropic.ts`

流式响应需要实时转换 SSE 事件:

```typescript
export async function* openaiChatStreamToAnthropic(
  stream: ReadableStream<Uint8Array>,
  model: string,
): AsyncGenerator<Uint8Array> {
  const reader = stream.getReader();
  const decoder = new TextDecoder();

  let buffer = '';
  let messageId = `msg_${Date.now()}`;
  let contentBlocks: AnthropicContentBlock[] = [];

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (!line.trim() || line === 'data: [DONE]') continue;

        if (line.startsWith('data: ')) {
          const data = line.slice(6);
          try {
            const chunk = JSON.parse(data) as OpenAIChatStreamChunk;

            // 转换为 Anthropic 流事件
            const anthropicEvents = convertChunkToAnthropicEvents(chunk, messageId, contentBlocks);

            for (const event of anthropicEvents) {
              yield encoder.encode(`event: ${event.type}\ndata: ${JSON.stringify(event)}\n\n`);
            }
          } catch (e) {
            console.error('Failed to parse stream chunk', e);
          }
        }
      }
    }
  } finally {
    reader.releaseLock();
  }
}
```

**流事件映射**:

| OpenAI 事件 | Anthropic 事件 | 说明 |
|-------------|----------------|------|
| `chunk.choices[0].delta.content` | `content_block_delta { type: 'text_delta' }` | 文本增量 |
| `chunk.choices[0].delta.reasoning_content` | `content_block_delta { type: 'thinking_delta' }` | 思考增量 |
| `chunk.choices[0].delta.tool_calls` | `content_block_delta { type: 'input_json_delta' }` | 工具参数增量 |
| `chunk.choices[0].finish_reason` | `message_delta { stop_reason }` | 停止原因 |

---

## 配置方式

### 方式一: 桌面端配置 (推荐)

1. **打开桌面端设置**
   - 点击右上角齿轮图标 ⚙️
   - 进入 "Provider Management"

2. **选择 DeepSeek 预设**
   - 在 Provider 列表中找到 "DeepSeek"
   - 点击 "Activate"

3. **填入 API Key**
   - 从 https://platform.deepseek.com/api_keys 获取 API Key
   - 粘贴到 "API Key" 输入框
   - 点击 "Save"

4. **选择模型**
   - Main Model: `deepseek-v4-pro`
   - Haiku: `deepseek-v4-flash`
   - Sonnet: `deepseek-v4-pro`
   - Opus: `deepseek-v4-pro`

### 方式二: 环境变量配置

**.env 文件**:

```bash
# DeepSeek 官方 Anthropic 接口
ANTHROPIC_AUTH_TOKEN=sk-xxx  # 你的 DeepSeek API Key
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
ANTHROPIC_MODEL=deepseek-v4-pro
ANTHROPIC_DEFAULT_SONNET_MODEL=deepseek-v4-pro
ANTHROPIC_DEFAULT_HAIKU_MODEL=deepseek-v4-flash
ANTHROPIC_DEFAULT_OPUS_MODEL=deepseek-v4-pro

# 超时和遥测
API_TIMEOUT_MS=300000
DISABLE_TELEMETRY=1
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

### 方式三: settings.json 配置

**~/.claude/settings.json**:

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-xxx",
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro",
    "API_TIMEOUT_MS": "300000",
    "DISABLE_TELEMETRY": "1",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
```

---

## 完整示例

### 使用 DeepSeek Anthropic 接口 (推荐)

```bash
# 1. 设置环境变量
export ANTHROPIC_AUTH_TOKEN="sk-xxx"
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_MODEL="deepseek-v4-pro"
export DISABLE_TELEMETRY=1

# 2. 启动 Claude Code
./bin/claude-haha

# 3. 开始对话
> 你好！请帮我读取项目的 README.md 文件。

# DeepSeek 处理流程:
# - 收到 Anthropic Messages API 请求
# - 调用 FileReadTool
# - 返回文件内容
# - 生成回复
```

### 使用 LiteLLM 代理 (适用于不支持 Anthropic 协议的情况)

```bash
# 1. 安装 LiteLLM
pip install 'litellm[proxy]'

# 2. 创建配置文件 litellm_config.yaml
cat > litellm_config.yaml <<EOF
model_list:
  - model_name: deepseek-chat
    litellm_params:
      model: deepseek/deepseek-chat
      api_key: os.environ/DEEPSEEK_API_KEY
      api_base: https://api.deepseek.com

litellm_settings:
  drop_params: true
EOF

# 3. 启动 LiteLLM 代理
export DEEPSEEK_API_KEY="sk-xxx"
litellm --config litellm_config.yaml --port 4000

# 4. 配置 Claude Code
export ANTHROPIC_AUTH_TOKEN="sk-anything"
export ANTHROPIC_BASE_URL="http://localhost:4000"
export ANTHROPIC_MODEL="deepseek-chat"
export DISABLE_TELEMETRY=1

# 5. 启动 Claude Code
./bin/claude-haha
```

**数据流**:

```
Claude Code
  ↓ Anthropic Request
LiteLLM Proxy (localhost:4000)
  ↓ Protocol Conversion
  ↓ OpenAI Request
DeepSeek API (https://api.deepseek.com)
  ↓ OpenAI Response
LiteLLM Proxy
  ↓ Protocol Conversion
  ↓ Anthropic Response
Claude Code
```

---

## 注意事项

### 1. max_tokens 限制

DeepSeek 的 `max_tokens` 上限通常为 **8192**，而 Claude Code 默认请求可能高达 128K。

**解决方案**:

- 使用 Anthropic 接口时，DeepSeek 会自动处理
- 使用协议转换时，代理会省略 `max_tokens` 参数

```typescript
// anthropicToOpenaiChat.ts
// max_tokens — omit to let upstream provider use its own default/max.
// Claude Code sends very large values (e.g. 128K) that exceed many
// providers' limits (DeepSeek: 8192, etc.).
```

### 2. 思考链 (Reasoning) 支持

DeepSeek 通过 `reasoning_content` 字段返回思考过程:

```json
{
  "message": {
    "reasoning_content": "用户询问...",
    "content": "量子计算是..."
  }
}
```

转换器会自动识别并转换为 Anthropic 的 `thinking` 块。

### 3. 工具调用兼容性

DeepSeek 支持完整的 OpenAI Function Calling 格式，转换器会自动处理:

- Anthropic `tool_use` ↔ OpenAI `tool_calls`
- Anthropic `tool_result` ↔ OpenAI `tool` role message

### 4. 流式响应延迟

使用协议转换代理时，流式响应会有轻微延迟:

- **直连**: ~50-100ms TTFT
- **代理**: ~100-200ms TTFT (增加 50-100ms 转换开销)

### 5. Prompt Caching

DeepSeek 不支持 Anthropic 的 `cache_control`，但支持自己的缓存机制。使用 Anthropic 接口时会自动处理。

---

## 总结

### DeepSeek 支持方式对比

| 方式 | 延迟 | 配置复杂度 | 功能完整性 | 推荐度 |
|------|------|------------|-----------|--------|
| **官方 Anthropic 接口** | 低 | 简单 | 完整 | ⭐⭐⭐⭐⭐ |
| **内置代理转换** | 中 | 中等 | 较完整 | ⭐⭐⭐⭐ |
| **LiteLLM 代理** | 中 | 中等 | 较完整 | ⭐⭐⭐ |

### 推荐配置

```bash
# 最简配置 (推荐)
ANTHROPIC_AUTH_TOKEN=sk-your-deepseek-api-key
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
ANTHROPIC_MODEL=deepseek-v4-pro
```

### 关键优势

1. ✅ **零转换开销**: 直接使用 Anthropic 协议
2. ✅ **完整特性**: 支持工具调用、思考链、百万上下文
3. ✅ **简单配置**: 3 行环境变量即可
4. ✅ **性能优异**: 无中间层，TTFT 最低
5. ✅ **官方支持**: DeepSeek 官方维护的接口

DeepSeek 是除 Claude 官方外，对 Anthropic 协议支持最好的模型提供商之一！ 🚀
