# Claude Code 项目调试指南

> 日期: 2026-05-29
> 作者: AI 代码分析
> 用途: 帮助开发者理解核心逻辑并高效调试

---

## 📋 目录

- [项目核心逻辑](#项目核心逻辑)
- [关键数据流](#关键数据流)
- [调试工具配置](#调试工具配置)
- [常见问题调试](#常见问题调试)
- [性能分析](#性能分析)

---

## 项目核心逻辑

### 1. 启动流程

```
bin/claude-haha (Bash 脚本)
  ↓
bun ./src/entrypoints/cli.tsx
  ↓
main() 函数 - 快速路径分发
  ├── --version → 直接输出版本号
  ├── --dump-system-prompt → 输出系统提示词
  ├── remote-control → 远程控制模式
  ├── daemon → 守护进程模式
  ├── ps/logs/attach/kill → 会话管理
  └── [默认] → 加载完整 CLI (print.tsx)
      ↓
  src/cli/print.tsx::print()
      ↓
  主事件循环 (Query Engine)
```

**关键文件**：
- `bin/claude-haha` - Bash 启动脚本
- `src/entrypoints/cli.tsx` - 入口点，快速路径分发
- `src/cli/print.ts` - CLI 主逻辑，会话管理
- `src/QueryEngine.ts` - 查询引擎，对话主循环
- `src/query.ts` - 单次查询执行

### 2. 对话循环 (Query Loop)

```
用户输入
  ↓
CommandQueue (messageQueueManager.ts)
  ↓
QueryEngine.ask() - 解析命令/提示
  ↓
query.ts::executeQuery()
  ├── 构建消息上下文 (messages)
  ├── 调用 Claude API (src/services/api/claude.ts)
  │   ↓
  │   anthropicRequest()
  │   ↓
  │   流式响应 (for await...of stream)
  │   ├── message_start
  │   ├── content_block_start
  │   ├── content_block_delta (文本/思考/工具调用)
  │   ├── content_block_stop
  │   └── message_stop
  │   ↓
  ├── 工具调用处理
  │   ↓
  │   src/services/tools/toolExecution.ts
  │   ↓
  │   executeTool()
  │   ├── validateInput()
  │   ├── checkPermissions()
  │   ├── call() - 工具主逻辑
  │   ├── mapToolResultToToolResultBlockParam()
  │   └── 返回 tool_result
  │   ↓
  └── 递归调用 (如有工具调用)
  ↓
渲染输出 (Ink 组件)
  ↓
等待下一轮输入
```

**关键文件**：
- `src/QueryEngine.ts:91` - `ask()` 方法，查询入口
- `src/query.ts:1730` - `executeQuery()` 核心逻辑
- `src/services/api/claude.ts:3489` - API 交互
- `src/services/tools/toolExecution.ts:1520` - 工具执行

### 3. 工具系统架构

```
Tool 定义 (src/Tool.ts)
  ↓
buildTool() - 工具构建器
  ├── name: 工具名称
  ├── inputSchema: 输入 Zod Schema
  ├── outputSchema: 输出 Zod Schema
  ├── call(): 工具主逻辑 (异步)
  ├── mapToolResultToToolResultBlockParam(): 映射到 API 格式
  ├── renderToolResultMessage(): UI 渲染
  ├── validateInput(): 输入验证
  ├── checkPermissions(): 权限检查
  └── 其他钩子方法
```

**工具执行流程**：

```typescript
// src/services/tools/toolExecution.ts

async function executeTool(
  tool: Tool,
  input: Record<string, unknown>,
  toolUseID: string,
  toolUseContext: ToolUseContext,
  assistantMessage: AssistantMessage,
) {
  // 1. 输入验证
  const validation = await tool.validateInput(input, toolUseContext);
  if (!validation.result) {
    throw new Error(validation.message);
  }

  // 2. 权限检查
  const permissionDecision = await tool.checkPermissions(input, toolUseContext);
  if (permissionDecision.status !== 'allow') {
    // 提示用户批准或拒绝
  }

  // 3. 执行工具
  const result = await tool.call(input, toolUseContext);

  // 4. 映射结果到 API 格式
  const toolResultBlock = tool.mapToolResultToToolResultBlockParam(
    result.data,
    toolUseID,
  );

  // 5. 返回结果
  return {
    toolOutput: result,
    toolResultBlock,
    newMessages: result.newMessages || [],
  };
}
```

**关键工具示例**：
- `src/tools/FileReadTool/FileReadTool.ts` - 文件读取
- `src/tools/BashTool/BashTool.tsx` - Shell 命令
- `src/tools/AgentTool/AgentTool.tsx` - 子代理

---

## 关键数据流

### 1. 消息流 (Message Flow)

```typescript
// 消息类型 (src/types/message.ts)

type Message =
  | UserMessage       // 用户消息 (文本/图片/工具结果)
  | AssistantMessage  // 助手消息 (文本/思考/工具调用)
  | SystemMessage     // 系统消息 (内部元数据)

// 消息结构
interface UserMessage {
  message: {
    role: 'user';
    content: ContentBlockParam[];  // 文本/图片/文档/工具结果
  };
  toolUseResult?: unknown;  // 工具执行结果 (UI 渲染用)
  uuid: string;
  isMeta?: boolean;  // 是否为元数据消息
}

interface AssistantMessage {
  message: {
    role: 'assistant';
    content: ContentBlock[];  // 文本/思考/工具调用
  };
  uuid: string;
  thinkingBlocks?: ThinkingBlock[];  // 思考块
}
```

### 2. API 请求/响应流

```typescript
// 请求 (Anthropic Messages API)
{
  model: 'claude-sonnet-4-6',
  max_tokens: 8192,
  messages: [
    { role: 'user', content: '你好' },
    { role: 'assistant', content: '你好！有什么我可以帮助的吗？' },
    { role: 'user', content: '读取 README.md' }
  ],
  tools: [
    {
      name: 'Read',
      description: '读取文件内容',
      input_schema: { type: 'object', properties: { file_path: { type: 'string' } } }
    }
  ],
  stream: true
}

// 响应 (流式)
event: message_start
data: { type: 'message', id: 'msg_123', role: 'assistant', ... }

event: content_block_start
data: { type: 'content_block_start', index: 0, content_block: { type: 'tool_use', id: 'toolu_456', name: 'Read' } }

event: content_block_delta
data: { type: 'content_block_delta', index: 0, delta: { type: 'input_json_delta', partial_json: '{"file' } }

event: content_block_delta
data: { type: 'content_block_delta', index: 0, delta: { type: 'input_json_delta', partial_json: '_path":"README.md"}' } }

event: content_block_stop
data: { type: 'content_block_stop', index: 0 }

event: message_stop
data: { type: 'message_stop' }
```

### 3. 工具调用流

```typescript
// 1. 模型返回工具调用
{
  type: 'tool_use',
  id: 'toolu_456',
  name: 'Read',
  input: { file_path: 'README.md' }
}

// 2. 执行工具
const result = await FileReadTool.call(
  { file_path: 'README.md' },
  toolUseContext
);
// 返回: { type: 'text', file: { content: '...', numLines: 100 } }

// 3. 映射到 API 格式
const toolResultBlock = {
  tool_use_id: 'toolu_456',
  type: 'tool_result',
  content: 'Read 100 lines'  // 或数组形式的复杂内容
};

// 4. 发送回模型
{
  role: 'user',
  content: [
    { type: 'tool_result', tool_use_id: 'toolu_456', content: '...' }
  ]
}

// 5. 模型继续响应
{
  role: 'assistant',
  content: [
    { type: 'text', text: '我已经读取了 README.md 文件，内容如下...' }
  ]
}
```

---

## 调试工具配置

### 1. VS Code 配置

**`.vscode/launch.json`**：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug CLI",
      "runtimeExecutable": "bun",
      "runtimeArgs": ["--inspect-brk", "./src/entrypoints/cli.tsx"],
      "console": "integratedTerminal",
      "cwd": "${workspaceFolder}",
      "env": {
        "NODE_ENV": "development",
        "DISABLE_TELEMETRY": "1"
      },
      "skipFiles": ["<node_internals>/**"],
      "outputCapture": "std"
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Tool Execution",
      "runtimeExecutable": "bun",
      "runtimeArgs": [
        "--inspect-brk",
        "./src/entrypoints/cli.tsx"
      ],
      "console": "integratedTerminal",
      "cwd": "${workspaceFolder}",
      "env": {
        "CLAUDE_CODE_DEBUG_TOOLS": "1",
        "LOG_LEVEL": "debug"
      }
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Specific Test",
      "runtimeExecutable": "bun",
      "runtimeArgs": ["test", "${file}"],
      "console": "integratedTerminal",
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

### 2. 日志系统

**启用调试日志**：

```bash
# 环境变量
export LOG_LEVEL=debug               # trace, debug, info, warn, error
export CLAUDE_CODE_DEBUG_TOOLS=1     # 工具执行详细日志
export CLAUDE_CODE_DEBUG_API=1       # API 请求/响应日志
export DISABLE_TELEMETRY=1           # 禁用遥测

# 启动
./bin/claude-haha
```

**日志位置**：
- 控制台输出：实时日志
- `~/.claude/logs/` - 持久化日志 (如果配置)
- `src/utils/log.ts` - 日志工具

**常用日志函数**：

```typescript
import { logError, logForDebugging } from 'src/utils/log.js';

// 错误日志
logError('Tool execution failed', error);

// 调试日志
logForDebugging('Processing tool result', { toolName, result });

// 诊断日志
import { logForDiagnosticsNoPII } from 'src/utils/diagLogs.js';
logForDiagnosticsNoPII('API request', { model, messageCount });
```

### 3. 断点调试关键位置

**入口点断点**：
- `src/entrypoints/cli.tsx:33` - `main()` 函数开始
- `src/cli/print.ts:XXX` - CLI 主循环入口
- `src/QueryEngine.ts:91` - `ask()` 方法

**对话循环断点**：
- `src/query.ts:XXX` - `executeQuery()` 开始
- `src/services/api/claude.ts:2005` - API 流式响应循环
- `src/services/api/claude.ts:2010` - `message_start` 事件
- `src/services/api/claude.ts:2020` - `content_block_delta` 事件

**工具执行断点**：
- `src/services/tools/toolExecution.ts:XXX` - `executeTool()` 开始
- `src/services/tools/toolExecution.ts:1292` - `mapToolResultToToolResultBlockParam()`
- `src/tools/FileReadTool/FileReadTool.ts:594` - `call()` 方法
- `src/tools/FileReadTool/FileReadTool.ts:869` - 图片读取逻辑

**UI 渲染断点**：
- `src/components/messages/UserToolResultMessage/UserToolSuccessMessage.tsx:65` - 工具结果渲染
- `src/tools/FileReadTool/UI.tsx:77` - `renderToolResultMessage()`

### 4. 网络抓包

**查看 API 请求/响应**：

```bash
# 方法 1: 环境变量
export CLAUDE_CODE_DEBUG_API=1
export DEBUG=anthropic:*

# 方法 2: 代码插桩
# 在 src/services/api/claude.ts 添加日志
console.log('Request:', JSON.stringify(request, null, 2));
console.log('Response chunk:', chunk);
```

**使用 mitmproxy 抓包**：

```bash
# 安装 mitmproxy
pip install mitmproxy

# 启动代理
mitmproxy -p 8080

# 设置环境变量
export HTTP_PROXY=http://localhost:8080
export HTTPS_PROXY=http://localhost:8080

# 启动 Claude Code
./bin/claude-haha
```

### 5. 内存分析

**检查内存泄漏**：

```bash
# 启用内存分析
bun --inspect ./src/entrypoints/cli.tsx

# 在 Chrome 中打开 chrome://inspect
# 连接到 Bun 进程
# 使用 Memory Profiler 拍摄快照
```

---

## 常见问题调试

### 问题 1: Read 工具返回 null / 图片不显示

**症状**：
- CLI 界面显示 `null`
- 图片文件存在但看不到内容

**调试步骤**：

1. **检查工具是否正确执行**：
   ```typescript
   // 在 src/tools/FileReadTool/FileReadTool.ts:869 添加断点
   const data = await readImageWithTokenBudget(resolvedFilePath, maxTokens);
   console.log('Image data:', data);  // 检查是否有 base64 数据
   ```

2. **检查 mapToolResultToToolResultBlockParam**：
   ```typescript
   // 在 src/tools/FileReadTool/FileReadTool.ts:654 添加断点
   case 'image': {
     console.log('Mapping image result:', data);
     return {
       tool_use_id: toolUseID,
       type: 'tool_result',
       content: [{
         type: 'image',
         source: { type: 'base64', data: data.file.base64, media_type: data.file.type }
       }]
     };
   }
   ```

3. **检查 UI 渲染**：
   ```typescript
   // 在 src/tools/FileReadTool/UI.tsx:80 添加断点
   case 'image': {
     const { originalSize } = output.file;
     console.log('Rendering image, size:', originalSize);
     const formattedSize = formatFileSize(originalSize);
     console.log('Formatted size:', formattedSize);
     return (
       <MessageResponse height={1}>
         <Text>Read image ({formattedSize})</Text>
       </MessageResponse>
     );
   }
   ```

4. **根本原因**：
   - CLI 模式不支持显示图片内容
   - `renderToolResultMessage()` 只渲染文本摘要
   - 图片数据已发送给模型，但用户在终端看不到

5. **解决方案**：
   - 使用桌面端查看图片
   - 或修改 `UI.tsx` 添加 iTerm2 图片协议支持

### 问题 2: 工具调用失败

**症状**：
- 模型返回工具调用，但执行失败
- 错误信息不清楚

**调试步骤**：

1. **启用工具调试日志**：
   ```bash
   export CLAUDE_CODE_DEBUG_TOOLS=1
   ```

2. **检查输入验证**：
   ```typescript
   // 在 tool.validateInput() 添加断点
   async validateInput({ file_path }, toolUseContext) {
     console.log('Validating input:', { file_path });
     // ... 验证逻辑
   }
   ```

3. **检查权限**：
   ```typescript
   // 在 tool.checkPermissions() 添加断点
   async checkPermissions(input, context) {
     console.log('Checking permissions:', input);
     // ... 权限检查
   }
   ```

4. **检查工具执行**：
   ```typescript
   // 在 tool.call() 添加 try-catch
   async call(input, context) {
     try {
       console.log('Executing tool:', this.name, input);
       const result = await this._innerCall(input, context);
       console.log('Tool result:', result);
       return result;
     } catch (error) {
       console.error('Tool execution error:', error);
       throw error;
     }
   }
   ```

### 问题 3: API 请求失败

**症状**：
- 模型无响应
- 连接超时
- 认证失败

**调试步骤**：

1. **检查环境变量**：
   ```bash
   echo $ANTHROPIC_API_KEY
   echo $ANTHROPIC_BASE_URL
   ```

2. **查看 API 日志**：
   ```typescript
   // 在 src/services/api/claude.ts:XXX 添加日志
   console.log('API Request:', {
     model,
     messages: messages.length,
     tools: tools?.length,
     stream,
   });
   ```

3. **测试 API 连通性**：
   ```bash
   curl -X POST https://api.anthropic.com/v1/messages \
     -H "x-api-key: $ANTHROPIC_API_KEY" \
     -H "anthropic-version: 2023-06-01" \
     -H "content-type: application/json" \
     -d '{"model":"claude-sonnet-4-6","max_tokens":1024,"messages":[{"role":"user","content":"Hello"}]}'
   ```

4. **检查代理设置**：
   - 如果使用 DeepSeek 等第三方，检查 `ANTHROPIC_BASE_URL`
   - 检查协议转换是否正常工作

### 问题 4: 性能问题

**症状**：
- 响应缓慢
- 内存占用高
- CPU 占用高

**调试步骤**：

1. **启用性能分析**：
   ```bash
   bun --inspect ./src/entrypoints/cli.tsx
   ```

2. **检查消息历史大小**：
   ```typescript
   // 在 src/query.ts 检查
   console.log('Message count:', messages.length);
   console.log('Total tokens estimate:', estimateTotalTokens(messages));
   ```

3. **检查工具结果大小**：
   ```typescript
   // 在 tool.call() 返回后检查
   const resultSize = JSON.stringify(result).length;
   console.log('Tool result size:', resultSize, 'bytes');
   ```

4. **使用性能分析工具**：
   ```bash
   # Bun 内置分析器
   bun --prof ./src/entrypoints/cli.tsx

   # 生成火焰图
   # ... 分析输出
   ```

---

## 性能分析

### 1. 启动性能

**测量启动时间**：

```typescript
// src/utils/startupProfiler.ts 已内置
// 查看输出：
// cli_entry -> 入口点
// cli_full_import -> 完整导入
// cli_ready -> 准备就绪
```

**优化建议**：
- 延迟加载大型模块
- 使用快速路径 (`--version`, `--help`)
- 减少启动时的 I/O 操作

### 2. 查询性能

**测量单次查询时间**：

```typescript
// 在 src/query.ts 添加计时
const startTime = Date.now();
const result = await executeQuery(...);
const duration = Date.now() - startTime;
console.log('Query duration:', duration, 'ms');
```

**性能瓶颈**：
- API 请求延迟 (TTFT: Time To First Token)
- 工具执行时间
- 消息序列化/反序列化
- UI 渲染

### 3. 内存优化

**检查内存使用**：

```bash
# 启动时监控
bun --expose-gc ./src/entrypoints/cli.tsx

# 在代码中手动 GC
global.gc();
console.log('Memory:', process.memoryUsage());
```

**常见内存泄漏**：
- 事件监听器未清理
- 大型对象未释放
- 循环引用
- 消息历史无限增长

**解决方案**：
- 定期清理旧消息
- 使用 WeakMap 存储临时数据
- 及时移除事件监听器

---

## 调试技巧总结

### 1. 快速定位问题

```
1. 重现问题
   ├── 记录复现步骤
   ├── 收集错误信息
   └── 确认环境配置

2. 缩小范围
   ├── 确定问题在哪一层 (入口/循环/工具/UI)
   ├── 添加日志定位具体位置
   └── 使用断点逐步调试

3. 分析原因
   ├── 检查数据流 (输入 → 处理 → 输出)
   ├── 验证假设 (添加断言/日志)
   └── 查看相关代码

4. 验证修复
   ├── 修改代码
   ├── 测试修复
   └── 回归测试
```

### 2. 有效的日志策略

```typescript
// ❌ 不好的日志
console.log('error');

// ✅ 好的日志
console.log('[FileReadTool] Failed to read image', {
  filePath,
  error: error.message,
  stack: error.stack,
});

// ✅ 结构化日志
import { logError } from 'src/utils/log.js';
logError('Tool execution failed', error, {
  tool: 'FileReadTool',
  input: { file_path },
  context: { messageId, toolUseId },
});
```

### 3. 断点调试技巧

- **条件断点**：只在特定条件下暂停
  ```typescript
  // 在 VS Code 断点上右键 → "Edit Breakpoint" → "Expression"
  // 输入条件：input.file_path.endsWith('.png')
  ```

- **日志断点**：不暂停，只输出日志
  ```typescript
  // 在 VS Code 断点上右键 → "Edit Breakpoint" → "Logpoint"
  // 输入：Tool: {this.name}, Input: {JSON.stringify(input)}
  ```

- **调用堆栈分析**：查看函数调用链
  ```
  executeTool()
    → tool.call()
    → readImageWithTokenBudget()
    → maybeResizeAndDownsampleImageBuffer()
    → [发现问题位置]
  ```

### 4. 最佳实践

1. **保持代码可调试性**
   - 避免过长的函数
   - 使用有意义的变量名
   - 添加类型注解

2. **编写可测试的代码**
   - 单一职责原则
   - 依赖注入
   - 纯函数优先

3. **善用工具**
   - TypeScript 类型检查
   - ESLint 静态分析
   - VS Code 调试器
   - Chrome DevTools (Bun Inspector)

4. **文档先行**
   - 记录设计决策
   - 注释复杂逻辑
   - 更新 README

---

## 附录

### A. 关键文件清单

| 文件 | 行数 | 作用 | 调试优先级 |
|------|------|------|-----------|
| `src/entrypoints/cli.tsx` | 500+ | CLI 入口点 | ⭐⭐⭐⭐⭐ |
| `src/cli/print.ts` | 3000+ | CLI 主循环 | ⭐⭐⭐⭐⭐ |
| `src/QueryEngine.ts` | 1000+ | 查询引擎 | ⭐⭐⭐⭐⭐ |
| `src/query.ts` | 1730 | 单次查询执行 | ⭐⭐⭐⭐⭐ |
| `src/services/api/claude.ts` | 3489 | API 交互 | ⭐⭐⭐⭐⭐ |
| `src/services/tools/toolExecution.ts` | 1520+ | 工具执行 | ⭐⭐⭐⭐ |
| `src/tools/FileReadTool/FileReadTool.ts` | 1184 | 文件读取工具 | ⭐⭐⭐⭐ |
| `src/tools/FileReadTool/UI.tsx` | 185 | 工具 UI 渲染 | ⭐⭐⭐ |
| `src/Tool.ts` | 500+ | 工具接口定义 | ⭐⭐⭐ |

### B. 常用命令

```bash
# 启动调试模式
bun --inspect-brk ./src/entrypoints/cli.tsx

# 查看版本
./bin/claude-haha --version

# 查看系统提示词
./bin/claude-haha --dump-system-prompt

# 启用详细日志
LOG_LEVEL=debug ./bin/claude-haha

# 禁用遥测
DISABLE_TELEMETRY=1 ./bin/claude-haha

# 简单模式 (禁用所有高级功能)
CLAUDE_CODE_SIMPLE=1 ./bin/claude-haha
```

### C. 环境变量参考

| 变量名 | 默认值 | 作用 |
|--------|--------|------|
| `ANTHROPIC_API_KEY` | - | Anthropic API 密钥 |
| `ANTHROPIC_BASE_URL` | `https://api.anthropic.com` | API 端点 |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-6` | 默认模型 |
| `LOG_LEVEL` | `info` | 日志级别 |
| `DISABLE_TELEMETRY` | `0` | 禁用遥测 |
| `CLAUDE_CODE_DEBUG_TOOLS` | `0` | 工具调试日志 |
| `CLAUDE_CODE_DEBUG_API` | `0` | API 调试日志 |
| `CLAUDE_CODE_SIMPLE` | `0` | 简单模式 |

### D. 参考资源

- **官方文档**: `docs/` 目录
- **代码示例**: `fixtures/` 目录
- **测试用例**: `tests/` 目录
- **已知问题**: GitHub Issues
- **更新日志**: `CHANGELOG.md`

---

## 结语

这份调试指南覆盖了 Claude Code 项目的核心逻辑和常见调试场景。调试的关键是：

1. **理解架构** - 知道数据如何流动
2. **快速定位** - 使用日志和断点缩小范围
3. **验证假设** - 用数据和测试验证你的猜测
4. **持续改进** - 记录问题和解决方案，优化代码

Happy Debugging! 🐛🔧
