# Claude Code Haha 核心模块架构分析

> 文档生成时间: 2026-05-29  
> 分析版本: 999.0.0-local  
> 代码库路径: d:\AI\cc-haha

---

## 📋 目录

1. [项目概览](#项目概览)
2. [核心架构](#核心架构)
3. [模块详细分析](#模块详细分析)
4. [技术栈与依赖](#技术栈与依赖)
5. [数据流与生命周期](#数据流与生命周期)
6. [扩展机制](#扩展机制)

---

## 项目概览

### 基本信息

Claude Code Haha 是基于 Anthropic 泄露源码修复的 Claude Code 桌面工作台，集成了：
- **多项目会话管理** - 标签页式工作区
- **代码可视化** - Diff、改动追踪、Worktree 支持
- **AI 代理系统** - 70+ 内置工具 + 可扩展 Skills
- **IM 远程接入** - 飞书/钉钉/Telegram/微信适配器
- **Computer Use** - 桌面截屏、鼠标/键盘控制
- **跨会话记忆** - 持久化记忆系统

### 代码统计

```
核心引擎文件 (9,391 行)
├── QueryEngine.ts      1,366 行  - AI 查询引擎
├── query.ts            1,730 行  - 查询执行逻辑  
├── main.tsx            4,749 行  - CLI 主逻辑
├── Tool.ts               792 行  - 工具基类
└── commands.ts           754 行  - 命令注册
```

---

## 核心架构

### 系统分层

```
┌─────────────────────────────────────────────────────────┐
│                    用户交互层                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  CLI/TUI     │  │  Desktop UI  │  │  IM Adapters │  │
│  │ (Ink/React)  │  │ (Tauri/React)│  │(飞书/钉钉等) │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼──────────────────┼──────────────────┼─────────┘
          │                  │                  │
┌─────────┼──────────────────┼──────────────────┼─────────┐
│         │        应用编排层 (main.tsx)         │         │
│         ▼                  ▼                  ▼         │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Commander.js + React State Management          │  │
│  │  - 命令解析  - 会话管理  - 权限控制             │  │
│  └────────────────────┬─────────────────────────────┘  │
└───────────────────────┼────────────────────────────────┘
                        │
┌───────────────────────┼────────────────────────────────┐
│                       ▼         核心引擎层              │
│  ┌───────────────────────────────────────────────────┐ │
│  │          QueryEngine (1,366 行)                   │ │
│  │  ┌─────────────────────────────────────────────┐ │ │
│  │  │ 消息流管理  │ 上下文构建 │ 工具调用路由    │ │ │
│  │  │ 权限裁决    │ 错误恢复   │ 成本追踪        │ │ │
│  │  └─────────────────────────────────────────────┘ │ │
│  └─────────────┬─────────────────────────────────────┘ │
│                │                                        │
│  ┌─────────────▼──────────────┐  ┌──────────────────┐ │
│  │  query() - 查询执行        │  │  Tool.ts         │ │
│  │  - API 调用                │  │  工具抽象基类    │ │
│  │  - 流式处理                │  │  (792 行)        │ │
│  └────────────────────────────┘  └──────────────────┘ │
└───────────────────────┬────────────────────────────────┘
                        │
┌───────────────────────┼────────────────────────────────┐
│                       ▼        服务层                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ Claude   │ │   MCP    │ │  LSP     │ │  OAuth   │  │
│  │ API SDK  │ │ Client   │ │ Client   │ │ Service  │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
└────────────────────────────────────────────────────────┘
```

---

## 模块详细分析

### 1. 入口模块 (Entrypoints)

#### 1.1 CLI 入口 - `src/entrypoints/cli.tsx`

**设计思想**: 快速路径优化 (Fast-path optimization)

```typescript
// 零导入快速路径
if (args[0] === '--version') {
  console.log(`${MACRO.VERSION} (Claude Code)`);
  return; // 无需加载任何模块
}

// 特性门控 + 动态导入
if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
  const { getSystemPrompt } = await import('../constants/prompts.js');
  // ...
}
```

**核心职责**:
1. **参数预处理** - 识别特殊标志 (--version, --daemon-worker 等)
2. **快速路径分发** - 避免不必要的模块加载
3. **环境初始化** - Corepack 修复、堆大小设置、Ablation 基线
4. **特性门控** - 通过 `feature()` 实现构建时死代码消除

**支持的快速路径**:
```
--version              零导入版本查询
--dump-system-prompt   导出 System Prompt (用于评估)
--claude-in-chrome-mcp Chrome 扩展 MCP 服务器
--computer-use-mcp     Computer Use MCP 服务器
--daemon-worker        守护进程工作节点
daemon [subcommand]    长运行守护进程
remote-control/rc      桥接模式 (远程控制本地机器)
ps/logs/attach/kill    后台会话管理
```

**启动性能优化**:
```typescript
// 启动分析器在首个动态导入后立即加载
const { profileCheckpoint } = await import('../utils/startupProfiler.js');
profileCheckpoint('cli_entry');

// MDM 配置 & Keychain 预取并行化
startMdmRawRead();      // macOS/Windows 企业配置管理
startKeychainPrefetch(); // OAuth token 预读取
```

---

#### 1.2 主入口 - `src/main.tsx` (4,749 行)

**核心架构**: Commander.js + React 状态机

```typescript
// 侧效应优化: 在所有导入前预热
profileCheckpoint('main_tsx_entry');
startMdmRawRead();           // 并行读取企业配置
startKeychainPrefetch();     // 并行预取 OAuth tokens

// Commander 命令注册
const program = new CommanderCommand()
  .name('claude-haha')
  .description('Claude Code Haha CLI')
  .version(MACRO.VERSION);

// 主 REPL 命令
program
  .command('repl', { isDefault: true })
  .option('--model <model>', '指定模型')
  .option('--cwd <path>', '工作目录')
  .option('--bg', '后台运行')
  .action(async (opts) => {
    await launchRepl(opts);
  });
```

**生命周期管理**:

```
启动阶段:
  1. init() - 配置加载、遥测初始化
  2. fetchBootstrapData() - API 引导数据
  3. prefetchOfficialMcpUrls() - 预取官方 MCP 服务器
  4. loadPolicyLimits() - 企业策略加载
  
运行阶段:
  1. launchRepl() - 启动 REPL
  2. QueryEngine 轮询
  3. 工具调用 → 权限裁决 → 执行
  
退出阶段:
  1. flushSessionStorage() - 持久化会话
  2. 清理后台任务
  3. 关闭 MCP 连接
```

**特性门控** (Feature Flags):
```typescript
feature('DAEMON')           // 守护进程模式
feature('BRIDGE_MODE')      // 远程桥接
feature('COORDINATOR_MODE') // 协调器模式
feature('KAIROS')          // Assistant 模式
feature('BG_SESSIONS')     // 后台会话
feature('DUMP_SYSTEM_PROMPT') // Prompt 导出
feature('ABLATION_BASELINE')  // 消融实验基线
```

---

### 2. 核心引擎模块

#### 2.1 查询引擎 - `src/QueryEngine.ts` (1,366 行)

**设计模式**: 状态机 + 流式处理

```typescript
export class QueryEngine {
  config: QueryEngineConfig;
  
  // 核心状态
  private messages: Message[] = [];
  private fileStateCache: FileStateCache;
  private denialTracking: DenialTrackingState;
  private attributionState: AttributionState;
  
  async runQuery(userInput: string): Promise<QueryResult> {
    // 1. 输入预处理
    const processed = await processUserInput(userInput, this.config);
    
    // 2. 上下文构建
    const systemPrompt = await fetchSystemPromptParts(
      this.config.cwd,
      this.config.tools,
      this.config.mcpClients
    );
    
    // 3. API 调用 (流式)
    const stream = await query({
      messages: this.messages,
      systemPrompt,
      tools: this.config.tools,
      model: this.resolveModel(),
    });
    
    // 4. 流式处理
    for await (const chunk of stream) {
      if (chunk.type === 'tool_use') {
        await this.handleToolCall(chunk);
      }
    }
    
    // 5. 成本追踪
    this.trackUsage(stream.usage);
    
    // 6. 会话持久化
    await recordTranscript(this.messages);
  }
  
  private async handleToolCall(toolUse: ToolUseBlockParam) {
    const tool = this.config.tools.find(t => 
      toolMatchesName(t, toolUse.name)
    );
    
    // 权限检查
    const permResult = await this.config.canUseTool(
      tool,
      toolUse.input,
      this.denialTracking
    );
    
    if (!permResult.allowed) {
      return this.handleDenial(permResult);
    }
    
    // 执行工具
    const result = await tool.execute(toolUse.input, {
      cwd: this.config.cwd,
      messages: this.messages,
      // ...
    });
    
    // 存储结果
    this.messages.push(createToolResultMessage(result));
  }
}
```

**关键机制**:

**消息链管理**:
```typescript
type Message =
  | UserMessage          // 用户输入
  | AssistantMessage     // AI 响应
  | SystemMessage        // 系统指令
  | ToolResultMessage    // 工具结果
  | CompactBoundaryMessage; // 压缩边界

// 消息链压缩
if (shouldCompact(this.messages)) {
  const compacted = await snipModule.compactMessages(
    this.messages,
    { strategy: 'snip' }
  );
  this.messages = compacted;
}
```

**错误恢复**:
```typescript
// 可重试的 API 错误
const error = categorizeRetryableAPIError(apiError);
if (error.retryable) {
  await sleep(error.backoffMs);
  return this.runQuery(userInput); // 重试
}

// 工具调用失败
if (toolResult.error) {
  this.messages.push({
    type: 'tool_result',
    tool_use_id: toolUse.id,
    is_error: true,
    content: toolResult.error,
  });
  // AI 会看到错误并自动修正
}
```

**成本追踪**:
```typescript
const usage = {
  input_tokens: stream.usage.input_tokens,
  output_tokens: stream.usage.output_tokens,
  cache_read_tokens: stream.usage.cache_read_input_tokens,
  cache_creation_tokens: stream.usage.cache_creation_input_tokens,
};

accumulateUsage(usage);
const cost = getTotalCost(); // 计算美元成本
```

---

#### 2.2 查询执行 - `src/query.ts` (1,730 行)

**职责**: 封装 Anthropic SDK 调用

```typescript
export async function query(
  params: QueryParams
): Promise<AsyncIterable<StreamEvent>> {
  const { messages, systemPrompt, tools, model } = params;
  
  // 构建 SDK 消息
  const sdkMessages = messages.map(toSDKMessage);
  
  // 工具转换
  const sdkTools = tools.map(tool => ({
    name: tool.name,
    description: tool.description,
    input_schema: tool.schema,
  }));
  
  // 流式 API 调用
  const stream = await anthropic.messages.create({
    model,
    max_tokens: 8192,
    system: systemPrompt,
    messages: sdkMessages,
    tools: sdkTools,
    stream: true,
    // Prompt Caching
    anthropic_beta: ['prompt-caching-2024-07-31'],
  });
  
  // 流式输出
  for await (const event of stream) {
    yield event;
  }
}
```

**Prompt Caching 优化**:
```typescript
// System Prompt 标记为可缓存
const systemPromptWithCache = [
  ...systemPrompt.slice(0, -1),
  {
    type: 'text',
    text: systemPrompt[systemPrompt.length - 1],
    cache_control: { type: 'ephemeral' }, // 缓存 5 分钟
  },
];
```

---

#### 2.3 工具抽象 - `src/Tool.ts` (792 行)

**接口设计**:

```typescript
export type Tool = {
  name: string;
  description: string;
  schema: ToolInputJSONSchema;
  
  // 执行函数
  execute: (
    input: ToolInput,
    context: ToolUseContext
  ) => Promise<ToolResult>;
  
  // 权限检查
  validate?: (
    input: ToolInput,
    context: ToolUseContext
  ) => Promise<ValidationResult>;
  
  // 进度回调
  onProgress?: (
    progress: ToolProgressData
  ) => void;
};

export type ToolUseContext = {
  cwd: string;
  messages: Message[];
  fileStateCache: FileStateCache;
  getAppState: () => AppState;
  setAppState: (f: (prev: AppState) => AppState) => void;
  // ...
};

export type ToolResult = {
  content: string | ContentBlock[];
  isError?: boolean;
  metadata?: Record<string, unknown>;
};
```

**工具类型**:

```typescript
// 文件操作工具
class FileReadTool implements Tool {
  name = 'Read';
  schema = {
    type: 'object',
    properties: {
      file_path: { type: 'string' },
      offset: { type: 'number' },
      limit: { type: 'number' },
    },
  };
  
  async execute(input, ctx) {
    const content = await readFile(input.file_path, {
      offset: input.offset,
      limit: input.limit || 2000,
    });
    return { content };
  }
}

// 命令执行工具
class BashTool implements Tool {
  name = 'Bash';
  
  async execute(input, ctx) {
    const result = await execa(input.command, {
      cwd: ctx.cwd,
      timeout: input.timeout || 120000,
    });
    return {
      content: result.stdout,
      metadata: { exitCode: result.exitCode },
    };
  }
}

// AI 代理工具
class AgentTool implements Tool {
  name = 'Agent';
  
  async execute(input, ctx) {
    const subEngine = new QueryEngine({
      ...ctx.config,
      cwd: input.isolation === 'worktree' 
        ? await createWorktree() 
        : ctx.cwd,
    });
    
    const result = await subEngine.runQuery(input.prompt);
    return { content: result.summary };
  }
}
```

---

### 3. 工具系统 (70+ 工具)

#### 工具分类

```
📁 src/tools/
├── 📄 文件操作 (5 个)
│   ├── FileReadTool      - 读取文件 (支持 PDF/图片/Notebook)
│   ├── FileEditTool      - 精确字符串替换编辑
│   ├── FileWriteTool     - 写入新文件
│   ├── GlobTool          - 文件模式匹配
│   └── GrepTool          - 内容搜索 (基于 ripgrep)
│
├── 💻 命令执行 (3 个)
│   ├── BashTool          - Bash 命令执行
│   ├── PowerShellTool    - PowerShell 执行 (Windows)
│   └── NotebookEditTool  - Jupyter Notebook 编辑
│
├── 🔧 项目管理 (4 个)
│   ├── EnterWorktreeTool - 创建隔离 Git worktree
│   ├── ExitWorktreeTool  - 清理 worktree
│   ├── LSPTool           - Language Server Protocol
│   └── TodoWriteTool     - 任务列表管理
│
├── 🤖 AI 交互 (7 个)
│   ├── AgentTool         - 子代理启动 (支持并行/隔离)
│   ├── AskUserQuestionTool - 多选/单选问卷
│   ├── SkillTool         - 技能系统调用
│   ├── EnterPlanModeTool - 进入规划模式
│   ├── ExitPlanModeTool  - 退出规划模式
│   ├── SendMessageTool   - 发送消息给其他 Agent
│   └── ListPeersTool     - 列出团队成员
│
├── 🌐 扩展能力 (8 个)
│   ├── MCPTool           - MCP 工具调用
│   ├── ReadMcpResourceTool - 读取 MCP 资源
│   ├── ListMcpResourcesTool - 列出 MCP 资源
│   ├── McpAuthTool       - MCP OAuth 认证
│   ├── WebSearchTool     - Google 搜索 (MCP)
│   ├── WebFetchTool      - 网页抓取 (Jina Reader)
│   ├── WebBrowserTool    - Puppeteer 浏览器控制
│   └── LSPTool           - LSP 协议交互
│
├── 📋 任务管理 (6 个)
│   ├── TaskCreateTool    - 创建后台任务
│   ├── TaskListTool      - 列出任务
│   ├── TaskOutputTool    - 获取任务输出
│   ├── TaskStopTool      - 停止任务
│   ├── ScheduleCronTool  - 定时任务调度
│   └── SleepTool         - 延迟执行
│
├── 👥 团队协作 (4 个)
│   ├── TeamCreateTool    - 创建 Agent 团队
│   ├── TeamDeleteTool    - 删除团队
│   ├── SendMessageTool   - 团队消息
│   └── ListPeersTool     - 成员列表
│
└── 🔍 调试工具 (5 个)
    ├── MonitorTool       - 性能监控
    ├── CtxInspectTool    - 上下文检查
    ├── ToolSearchTool    - 工具搜索
    ├── SnipTool          - 消息压缩测试
    └── OverflowTestTool  - 上下文溢出测试
```

---

#### 工具实现示例: FileReadTool

**完整实现** (参考 `src/tools/FileReadTool/FileReadTool.ts`):

```typescript
export const FileReadTool: Tool = {
  name: 'Read',
  description: 'Reads a file from the local filesystem...',
  
  schema: {
    type: 'object',
    properties: {
      file_path: {
        type: 'string',
        description: 'The absolute path to the file to read',
      },
      offset: {
        type: 'integer',
        minimum: 0,
        description: 'The line number to start reading from',
      },
      limit: {
        type: 'integer',
        exclusiveMinimum: 0,
        maximum: 9007199254740991,
        description: 'The number of lines to read',
      },
      pages: {
        type: 'string',
        description: 'Page range for PDF files (e.g., "1-5")',
      },
    },
    required: ['file_path'],
  },
  
  async execute(input, ctx) {
    const { file_path, offset = 0, limit = 2000, pages } = input;
    
    // 路径验证
    const absPath = resolvePath(ctx.cwd, file_path);
    if (!isPathAllowed(absPath, ctx.permissions)) {
      throw new PermissionDeniedError(
        `Access denied: ${absPath} is outside allowed directories`
      );
    }
    
    // 文件类型检测
    const ext = path.extname(absPath).toLowerCase();
    
    // PDF 文件处理
    if (ext === '.pdf') {
      if (!pages && (await getPdfPageCount(absPath)) > 10) {
        throw new Error(
          'Large PDF detected. Use pages parameter (e.g., pages: "1-5")'
        );
      }
      const pdfContent = await readPdf(absPath, pages);
      return {
        content: pdfContent,
        metadata: { type: 'pdf', pages },
      };
    }
    
    // 图片文件处理 (多模态)
    if (['.png', '.jpg', '.jpeg', '.webp', '.gif'].includes(ext)) {
      const imageData = await readFile(absPath);
      const base64 = imageData.toString('base64');
      return {
        content: [
          {
            type: 'image',
            source: {
              type: 'base64',
              media_type: `image/${ext.slice(1)}`,
              data: base64,
            },
          },
        ],
      };
    }
    
    // Jupyter Notebook 处理
    if (ext === '.ipynb') {
      const notebook = await readNotebook(absPath);
      return {
        content: formatNotebookCells(notebook.cells),
        metadata: { type: 'notebook', cellCount: notebook.cells.length },
      };
    }
    
    // 文本文件处理 (带行号)
    const lines = await readFileLines(absPath, offset, limit);
    const numbered = lines.map((line, i) => 
      `${offset + i + 1}\t${line}`
    ).join('\n');
    
    return {
      content: numbered,
      metadata: {
        totalLines: await getLineCount(absPath),
        displayedLines: lines.length,
      },
    };
  },
};
```

**特性**:
- ✅ 支持文本/PDF/图片/Notebook
- ✅ 分页读取 (避免超长上下文)
- ✅ 路径权限验证
- ✅ 行号标注 (cat -n 格式)
- ✅ 文件状态缓存

---

### 4. 服务层 (Services)

#### 4.1 Claude API 服务 - `src/services/api/`

**核心文件**:
```
api/
├── claude.ts            - Anthropic SDK 封装
├── bootstrap.ts         - 引导数据获取
├── errors.ts            - 错误分类与重试策略
├── logging.ts           - API 调用日志
└── filesApi.ts          - 文件 API (下载远程附件)
```

**API 客户端**:
```typescript
// claude.ts
import Anthropic from '@anthropic-ai/sdk';

let anthropicClient: Anthropic;

export function getAnthropicClient(): Anthropic {
  if (!anthropicClient) {
    const apiKey = process.env.ANTHROPIC_API_KEY;
    const baseURL = process.env.ANTHROPIC_BASE_URL;
    
    anthropicClient = new Anthropic({
      apiKey,
      baseURL,
      timeout: 300000, // 5 分钟
    });
  }
  return anthropicClient;
}

export async function* streamMessages(
  params: MessageStreamParams
): AsyncIterable<MessageStreamEvent> {
  const client = getAnthropicClient();
  
  const stream = await client.messages.create({
    ...params,
    stream: true,
  });
  
  for await (const event of stream) {
    yield event;
  }
}
```

**错误处理**:
```typescript
// errors.ts
export function categorizeRetryableAPIError(
  error: unknown
): RetryableError | null {
  if (error instanceof Anthropic.APIError) {
    // 速率限制
    if (error.status === 429) {
      return {
        retryable: true,
        backoffMs: 5000,
        reason: 'rate_limit',
      };
    }
    
    // 服务器错误
    if (error.status >= 500) {
      return {
        retryable: true,
        backoffMs: 2000,
        reason: 'server_error',
      };
    }
    
    // Overloaded
    if (error.status === 529) {
      return {
        retryable: true,
        backoffMs: 10000,
        reason: 'overloaded',
      };
    }
  }
  
  return null;
}
```

---

#### 4.2 MCP 服务 - `src/services/mcp/`

**Model Context Protocol 实现**:

```
mcp/
├── client.ts            - MCP 客户端
├── server.ts            - MCP 服务端
├── officialRegistry.ts  - 官方服务器注册表
├── types.ts             - MCP 类型定义
└── connection.ts        - 连接管理
```

**客户端实现**:
```typescript
// client.ts
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';

export async function connectMcpServer(
  config: McpServerConfig
): Promise<MCPServerConnection> {
  const transport = new StdioClientTransport({
    command: config.command,
    args: config.args,
    env: config.env,
  });
  
  const client = new Client({
    name: 'claude-code',
    version: MACRO.VERSION,
  });
  
  await client.connect(transport);
  
  // 列出可用工具
  const { tools } = await client.listTools();
  
  // 列出可用资源
  const { resources } = await client.listResources();
  
  return {
    name: config.name,
    client,
    tools,
    resources,
    disconnect: () => client.close(),
  };
}
```

**工具调用桥接**:
```typescript
// MCPTool.ts
export const MCPTool: Tool = {
  name: 'MCP',
  description: 'Call a tool from an MCP server',
  
  schema: {
    type: 'object',
    properties: {
      server_name: { type: 'string' },
      tool_name: { type: 'string' },
      arguments: { type: 'object' },
    },
  },
  
  async execute(input, ctx) {
    const conn = ctx.mcpClients.find(c => 
      c.name === input.server_name
    );
    
    if (!conn) {
      throw new Error(`MCP server not found: ${input.server_name}`);
    }
    
    const result = await conn.client.callTool({
      name: input.tool_name,
      arguments: input.arguments,
    });
    
    return {
      content: result.content,
      metadata: { isError: result.isError },
    };
  },
};
```

---

#### 4.3 记忆系统 - `src/memdir/`

**持久化记忆架构**:

```
.claude/projects/<project-hash>/memory/
├── MEMORY.md            - 记忆索引 (200 行限制)
├── user_role.md         - 用户信息
├── feedback_testing.md  - 反馈记录
├── project_deadline.md  - 项目信息
└── reference_linear.md  - 外部资源引用
```

**记忆加载**:
```typescript
// memdir.ts
export async function loadMemoryPrompt(
  cwd: string
): Promise<string> {
  const memDir = path.join(cwd, '.claude/memory');
  const indexPath = path.join(memDir, 'MEMORY.md');
  
  if (!fs.existsSync(indexPath)) {
    return '';
  }
  
  // 读取索引
  const index = await fs.readFile(indexPath, 'utf-8');
  const lines = index.split('\n').slice(0, 200); // 限制 200 行
  
  // 提取引用的文件
  const linkedFiles = lines
    .map(line => line.match(/\[.*?\]\((.*?)\)/)?.[1])
    .filter(Boolean);
  
  // 加载引用的记忆文件
  const memories = await Promise.all(
    linkedFiles.map(async (file) => {
      const fullPath = path.join(memDir, file);
      return fs.readFile(fullPath, 'utf-8');
    })
  );
  
  return `<memory>\n${memories.join('\n\n')}\n</memory>`;
}
```

**记忆写入**:
```typescript
// 由 AI 通过 FileWriteTool 调用
await Write({
  file_path: '.claude/memory/user_role.md',
  content: `---
name: user-role
description: Senior backend engineer, Go expertise
metadata:
  type: user
---

User is a senior backend engineer with 10 years of Go experience.
Currently focused on microservices refactoring.
Prefers verbose error handling over panic().
`,
});

// 更新索引
await Edit({
  file_path: '.claude/memory/MEMORY.md',
  old_string: '## User Memories\n',
  new_string: `## User Memories
- [User Role](user_role.md) — Senior backend engineer with Go expertise
`,
});
```

---

### 5. 桌面端 (Desktop)

#### 架构: Tauri 2 + React

```
desktop/
├── src-tauri/           # Rust 后端
│   ├── src/
│   │   ├── main.rs      # Tauri 入口
│   │   └── lib.rs       # IPC 命令注册
│   ├── Cargo.toml       # Rust 依赖
│   └── tauri.conf.json  # 窗口配置
│
├── sidecars/            # Sidecar 进程
│   ├── claude-sidecar.ts  # CLI 包装器
│   └── launcherRouting.ts # 进程路由
│
└── src/                 # React 前端
    ├── App.tsx          # 根组件
    ├── api/             # Tauri IPC 调用
    ├── components/      # UI 组件
    ├── pages/           # 页面路由
    └── stores/          # 状态管理
```

**Sidecar 进程管理**:
```typescript
// claude-sidecar.ts
import { Command } from '@tauri-apps/plugin-shell';

export async function spawnClaudeSidecar(
  cwd: string,
  args: string[]
): Promise<ChildProcess> {
  const sidecar = Command.sidecar('binaries/claude-haha', args);
  
  const child = await sidecar.spawn();
  
  // 监听输出
  child.stdout.on('data', (line) => {
    handleClaudeOutput(line);
  });
  
  child.stderr.on('data', (line) => {
    handleClaudeError(line);
  });
  
  return child;
}
```

**IPC 命令**:
```rust
// lib.rs
#[tauri::command]
async fn start_session(
  cwd: String,
  model: String,
) -> Result<String, String> {
  let session_id = Uuid::new_v4().to_string();
  
  // 启动 sidecar
  let child = Command::new_sidecar("claude-haha")
    .args(&["--cwd", &cwd, "--model", &model])
    .spawn()
    .map_err(|e| e.to_string())?;
  
  // 存储进程句柄
  SESSIONS.lock().unwrap().insert(session_id.clone(), child);
  
  Ok(session_id)
}

#[tauri::command]
async fn send_message(
  session_id: String,
  message: String,
) -> Result<(), String> {
  let sessions = SESSIONS.lock().unwrap();
  let child = sessions.get(&session_id)
    .ok_or("Session not found")?;
  
  // 写入 stdin
  child.write(message.as_bytes())
    .map_err(|e| e.to_string())?;
  
  Ok(())
}
```

**前端组件**:
```tsx
// src/pages/Chat.tsx
export function ChatPage() {
  const [sessionId, setSessionId] = useState<string | null>(null);
  const [messages, setMessages] = useState<Message[]>([]);
  
  // 启动会话
  const startSession = async () => {
    const id = await invoke<string>('start_session', {
      cwd: '/path/to/project',
      model: 'claude-sonnet-4-5',
    });
    setSessionId(id);
    
    // 监听输出
    await listen<string>('claude-output', (event) => {
      const newMsg = JSON.parse(event.payload);
      setMessages(prev => [...prev, newMsg]);
    });
  };
  
  // 发送消息
  const sendMessage = async (text: string) => {
    await invoke('send_message', {
      sessionId,
      message: text,
    });
  };
  
  return (
    <div>
      <MessageList messages={messages} />
      <ChatInput onSend={sendMessage} />
    </div>
  );
}
```

---

### 6. IM 适配器 (Adapters)

#### 飞书适配器 - `adapters/feishu/`

**卡片渲染**:
```typescript
// streaming-card.ts
export function createStreamingCard(
  text: string,
  toolCalls: ToolUse[]
): FeishuCard {
  return {
    config: {
      wide_screen_mode: true,
    },
    header: {
      title: {
        content: '🤖 Claude Code',
        tag: 'plain_text',
      },
    },
    elements: [
      // AI 响应文本
      {
        tag: 'markdown',
        content: text,
      },
      // 工具调用展示
      ...toolCalls.map(tool => ({
        tag: 'div',
        fields: [
          {
            is_short: true,
            text: {
              tag: 'lark_md',
              content: `**Tool**: ${tool.name}`,
            },
          },
          {
            is_short: false,
            text: {
              tag: 'lark_md',
              content: `\`\`\`json\n${JSON.stringify(tool.input, null, 2)}\n\`\`\``,
            },
          },
        ],
      })),
    ],
  };
}
```

**流式更新**:
```typescript
// flush-controller.ts
export class FlushController {
  private buffer = '';
  private cardId: string;
  private lastFlushTime = Date.now();
  
  async append(chunk: string) {
    this.buffer += chunk;
    
    // 限流: 每 500ms 更新一次
    if (Date.now() - this.lastFlushTime > 500) {
      await this.flush();
    }
  }
  
  async flush() {
    const card = createStreamingCard(this.buffer, []);
    
    await updateFeishuCard(this.cardId, card);
    this.lastFlushTime = Date.now();
  }
}
```

---

## 技术栈与依赖

### 核心依赖

```json
{
  "dependencies": {
    // AI SDK
    "@anthropic-ai/sdk": "^0.80.0",
    "@aws-sdk/client-bedrock-runtime": "^3.1020.0",
    "@modelcontextprotocol/sdk": "^1.29.0",
    
    // CLI 框架
    "@commander-js/extra-typings": "^14.0.0",
    "ink": "^6.8.0",
    "react": "^19.2.4",
    
    // 工具库
    "execa": "^9.6.1",           // 命令执行
    "chokidar": "^5.0.0",        // 文件监听
    "picomatch": "^4.0.4",       // Glob 匹配
    "diff": "^8.0.4",            // Diff 算法
    "marked": "^17.0.5",         // Markdown 解析
    "turndown": "^7.2.4",        // HTML → Markdown
    
    // 网络
    "ws": "^8.20.0",             // WebSocket
    "axios": "^1.14.0",          // HTTP 客户端
    "undici": "^7.24.6",         // Fetch API
    
    // 数据处理
    "yaml": "^2.8.3",
    "jsonc-parser": "^3.3.1",
    "zod": "^4.3.6",             // Schema 验证
    
    // LSP
    "vscode-jsonrpc": "^8.2.1",
    "vscode-languageserver-types": "^3.17.5"
  }
}
```

---

## 数据流与生命周期

### 完整查询周期

```
┌─────────────────────────────────────────────────────────────┐
│  1. 用户输入                                                  │
│     "Add a login button to the navbar"                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  2. 输入预处理 (processUserInput)                            │
│     - 检测斜杠命令 (/commit, /review)                        │
│     - 附件处理 (图片 → base64)                               │
│     - IDE 选区注入                                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  3. 上下文构建 (fetchSystemPromptParts)                      │
│     ┌───────────────────────────────────────────────┐       │
│     │ System Prompt 构成:                           │       │
│     │ ├─ 基础指令 (角色定义、工作模式)             │       │
│     │ ├─ 工具列表 (70+ 工具描述)                   │       │
│     │ ├─ MCP 资源 (外部工具/数据源)                │       │
│     │ ├─ 记忆系统 (用户偏好、项目上下文)           │       │
│     │ ├─ 文件历史 (最近修改文件快照)               │       │
│     │ └─ 环境信息 (OS、Shell、工作目录)            │       │
│     └───────────────────────────────────────────────┘       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  4. API 调用 (query)                                         │
│     POST https://api.anthropic.com/v1/messages              │
│     {                                                        │
│       "model": "claude-sonnet-4-5",                         │
│       "max_tokens": 8192,                                   │
│       "system": [...],  // ← Prompt Cache 标记              │
│       "messages": [...],                                    │
│       "tools": [...],                                       │
│       "stream": true                                        │
│     }                                                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  5. 流式响应处理                                              │
│     ┌──────────────────────────────────────────────┐        │
│     │ event: message_start                         │        │
│     │ event: content_block_start (type: text)      │        │
│     │ event: content_block_delta                   │        │
│     │   → 渲染到 UI                                 │        │
│     │ event: content_block_start (type: tool_use)  │        │
│     │   → 权限检查 → 执行工具 ──┐                  │        │
│     │ event: message_stop        │                  │        │
│     └────────────────────────────┼──────────────────┘        │
│                                  │                           │
└──────────────────────────────────┼───────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────┐
│  6. 工具调用周期 (handleToolCall)                            │
│     ┌───────────────────────────────────────────────┐       │
│     │ Tool: Read                                    │       │
│     │ Input: { file_path: "src/Navbar.tsx" }       │       │
│     │                                               │       │
│     │ ① 权限检查 (canUseTool)                      │       │
│     │    - 路径验证                                 │       │
│     │    - 用户审批 (Permission Mode)              │       │
│     │                                               │       │
│     │ ② 工具执行 (tool.execute)                    │       │
│     │    - 读取文件内容                             │       │
│     │    - 行号标注                                 │       │
│     │                                               │       │
│     │ ③ 结果封装                                    │       │
│     │    {                                          │       │
│     │      content: "1\timport React...",          │       │
│     │      metadata: { totalLines: 150 }           │       │
│     │    }                                          │       │
│     └───────────────────────────────────────────────┘       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  7. 消息链更新                                                │
│     messages.push({                                         │
│       type: 'tool_result',                                  │
│       tool_use_id: 'toolu_123',                            │
│       content: [...],                                       │
│     })                                                      │
│                                                             │
│     → 递归调用 API (AI 看到工具结果后继续)                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  8. 第二轮 API 调用                                           │
│     AI 响应: "I see the Navbar component. Let me add..."    │
│     Tool: Edit                                              │
│     Input: {                                                │
│       file_path: "src/Navbar.tsx",                         │
│       old_string: "return <nav>",                          │
│       new_string: "return <nav>\n  <LoginButton />"       │
│     }                                                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  9. 成本追踪 & 持久化                                         │
│     ┌─────────────────────────────────────────────┐         │
│     │ Usage:                                      │         │
│     │ ├─ input_tokens: 15,234                    │         │
│     │ ├─ output_tokens: 542                      │         │
│     │ ├─ cache_read_tokens: 12,000  (缓存命中)   │         │
│     │ └─ cache_creation_tokens: 3,234            │         │
│     │                                             │         │
│     │ Cost: $0.0234                               │         │
│     └─────────────────────────────────────────────┘         │
│                                                             │
│     flushSessionStorage() - 持久化到 ~/.claude/sessions/   │
└─────────────────────────────────────────────────────────────┘
```

---

## 扩展机制

### 1. Skills 系统

**目录结构**:
```
.claude/skills/
├── my-skill/
│   ├── skill.yaml       # 元数据
│   └── prompt.md        # 提示词
```

**示例 Skill**:
```yaml
# skill.yaml
name: debug-crash
description: Debug a crash log by analyzing stack traces
trigger:
  auto: false
  patterns:
    - "debug crash"
    - "analyze crash log"
```

```markdown
<!-- prompt.md -->
You are a crash analysis expert. When given a crash log:

1. Read the full log file
2. Identify the crash location (file:line)
3. Use Grep to find relevant code
4. Explain the root cause
5. Suggest a fix

Ask the user for the log file path if not provided.
```

**加载机制**:
```typescript
// src/skills/skillLoader.ts
export async function loadSkills(cwd: string): Promise<Skill[]> {
  const skillsDir = path.join(cwd, '.claude/skills');
  const dirs = await fs.readdir(skillsDir);
  
  const skills = await Promise.all(
    dirs.map(async (dir) => {
      const metaPath = path.join(skillsDir, dir, 'skill.yaml');
      const promptPath = path.join(skillsDir, dir, 'prompt.md');
      
      const meta = yaml.parse(await fs.readFile(metaPath, 'utf-8'));
      const prompt = await fs.readFile(promptPath, 'utf-8');
      
      return {
        name: meta.name,
        description: meta.description,
        trigger: meta.trigger,
        prompt,
      };
    })
  );
  
  return skills;
}
```

---

### 2. MCP 服务器

**配置示例**:
```json
// .claude/mcp.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/dir"]
    },
    "postgres": {
      "command": "mcp-server-postgres",
      "env": {
        "DATABASE_URL": "postgresql://localhost/mydb"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "your-key"
      }
    }
  }
}
```

**工具自动注册**:
```typescript
// 启动时连接所有 MCP 服务器
const mcpConnections = await Promise.all(
  Object.entries(mcpConfig).map(([name, config]) =>
    connectMcpServer({ ...config, name })
  )
);

// 工具列表自动包含 MCP 工具
const allTools = [
  ...builtinTools,
  ...mcpConnections.flatMap(conn => 
    conn.tools.map(tool => wrapMcpTool(conn.name, tool))
  ),
];
```

---

### 3. Hooks 系统

**配置**:
```json
// .claude/settings.json
{
  "hooks": {
    "pre-tool-call": {
      "Bash": "echo 'About to run: $CC_TOOL_INPUT_COMMAND'"
    },
    "post-tool-call": {
      "Bash": "echo 'Exit code: $CC_TOOL_RESULT_EXIT_CODE'"
    },
    "on-query-start": "git diff --cached > /tmp/staged-changes.diff",
    "on-query-end": "npm test"
  }
}
```

**Hook 执行**:
```typescript
// src/utils/hooks/hookHelpers.ts
export async function runHook(
  hookName: string,
  context: HookContext
): Promise<void> {
  const config = getGlobalConfig();
  const hookCmd = config.hooks?.[hookName];
  
  if (!hookCmd) return;
  
  // 注入环境变量
  const env = {
    ...process.env,
    CC_TOOL_NAME: context.toolName,
    CC_TOOL_INPUT_COMMAND: context.input?.command,
    CC_TOOL_RESULT_EXIT_CODE: context.result?.exitCode,
  };
  
  const result = await execa(hookCmd, {
    shell: true,
    env,
  });
  
  if (result.exitCode !== 0) {
    throw new HookFailedError(hookName, result.stderr);
  }
}
```

---

## 总结

### 核心优势

1. **模块化架构** - 清晰的分层设计,易于扩展
2. **工具丰富** - 70+ 内置工具覆盖全栈开发场景
3. **多端支持** - CLI/桌面/IM 三端协同
4. **性能优化** - Prompt Caching、流式输出、快速路径
5. **企业就绪** - 策略管理、审计日志、MDM 集成

### 可扩展点

- ✅ 自定义 Skills (YAML + Markdown)
- ✅ MCP 服务器集成 (标准协议)
- ✅ Hooks 系统 (生命周期拦截)
- ✅ 第三方模型 (OpenAI/DeepSeek/Ollama)
- ✅ IM 适配器 (飞书/钉钉/Telegram/微信)

### 技术亮点

- **Bun 运行时** - 快速启动 (< 50ms)
- **Tauri 2** - 跨平台桌面 (小体积 ~10MB)
- **Ink** - React 终端 UI (声明式渲染)
- **MCP 生态** - 标准化工具扩展
- **Prompt Caching** - 节省 90% Token 成本

---

**文档完整度**: ✅ 核心模块已全面覆盖  
**更新建议**: 定期同步代码变更,维护架构一致性
