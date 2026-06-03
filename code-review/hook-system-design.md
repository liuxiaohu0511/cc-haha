# Claude Code Hook 系统设计分析

## 目录
1. [系统概述](#系统概述)
2. [Hook 事件类型](#hook-事件类型)
3. [Hook 执行器类型](#hook-执行器类型)
4. [核心架构](#核心架构)
5. [配置与加载机制](#配置与加载机制)
6. [执行流程](#执行流程)
7. [安全与权限](#安全与权限)
8. [技术亮点](#技术亮点)

---

## 系统概述

Claude Code 的 Hook 系统是一个事件驱动的扩展机制，允许用户和插件在 AI 交互的关键生命周期节点注入自定义逻辑。Hook 系统支持四种执行器类型（Shell 命令、LLM Prompt、Agent、HTTP 请求），覆盖 27 个生命周期事件。

### 核心价值
- **可观察性**：工具调用前后、会话开始/结束等关键节点的监控
- **拦截与验证**：权限请求、工具执行、用户提示的拦截与校验
- **自动化工作流**：文件变更监听、配置更新、环境同步
- **扩展性**：插件系统可注册自定义 Hook，无需修改核心代码

### 关键指标
- **代码分布**：40+ 个 Hook 相关文件，核心实现约 12,000 行 TypeScript
- **事件覆盖**：27 个生命周期事件，涵盖工具调用、会话管理、配置变更等
- **执行器类型**：4 种（command/prompt/agent/http），支持同步/异步模式
- **配置来源**：用户设置、项目设置、插件配置、运行时回调

---

## Hook 事件类型

Hook 系统定义了 27 个生命周期事件，按功能分为 7 大类：

### 1. 工具执行生命周期（6 个事件）
| 事件名 | 触发时机 | 主要功能 | Matcher 字段 |
|--------|---------|---------|-------------|
| **PreToolUse** | 工具调用前 | 参数校验、权限控制、输入修改 | `tool_name` |
| **PostToolUse** | 工具调用成功后 | 结果分析、日志记录、输出修改 | `tool_name` |
| **PostToolUseFailure** | 工具调用失败后 | 错误处理、告警、重试逻辑 | `tool_name` |
| **PermissionDenied** | 自动模式拒绝工具后 | 记录拒绝原因、允许重试 | `tool_name` |
| **PermissionRequest** | 权限对话框显示时 | 自动化权限决策（允许/拒绝） | `tool_name` |
| **Elicitation** | MCP 服务器请求输入时 | 自动填充表单、拦截敏感请求 | `mcp_server_name` |

**核心机制**：
- **条件执行**：支持 `if` 字段（权限规则语法），例如 `if: "Bash(git *)"` 仅在 git 命令时触发
- **输出修改**：PreToolUse 可修改 `tool_input`，PostToolUse 可修改 MCP 工具输出
- **阻塞控制**：Exit code 2 触发 blocking error，向模型展示错误信息并阻止继续

### 2. 会话生命周期（6 个事件）
| 事件名 | 触发时机 | 主要功能 | Matcher 字段 |
|--------|---------|---------|-------------|
| **SessionStart** | 新会话启动时 | 环境初始化、加载项目配置 | `source` (startup/resume/clear/compact) |
| **SessionEnd** | 会话结束时 | 清理资源、保存状态 | `reason` (clear/logout/prompt_input_exit/other) |
| **SubagentStart** | 子 Agent 启动时 | 传递上下文、配置子 Agent 环境 | `agent_type` |
| **SubagentStop** | 子 Agent 结束前 | 验证子任务完成、收集结果 | `agent_type` |
| **Stop** | AI 回复结束前 | 验证任务完成度、触发后续动作 | 无 |
| **StopFailure** | API 错误导致回复结束时 | 记录失败原因（Fire-and-forget） | `error` (rate_limit/auth_failed/...) |

**特殊处理**：
- **SessionEnd 超时**：默认 1.5s 超时（环境变量可调），适配快速清理场景
- **BlockingError 忽略**：SessionStart/SubagentStart 的 blocking error 被忽略（不阻断启动）
- **Fire-and-forget**：StopFailure 不处理 Hook 输出，纯日志记录

### 3. 用户交互（3 个事件）
| 事件名 | 触发时机 | 主要功能 | Matcher 字段 |
|--------|---------|---------|-------------|
| **UserPromptSubmit** | 用户提交提示词时 | 提示词预处理、内容审查 | 无 |
| **Notification** | 发送通知时 | 通知转发、告警聚合 | `notification_type` |
| **ElicitationResult** | 用户响应 MCP 请求后 | 修改表单提交内容、拦截敏感数据 | `mcp_server_name` |

### 4. 配置与环境（4 个事件）
| 事件名 | 触发时机 | 主要功能 | Matcher 字段 |
|--------|---------|---------|-------------|
| **Setup** | 仓库初始化/维护时 | 依赖安装、环境配置 | `trigger` (init/maintenance) |
| **ConfigChange** | 配置文件变化时 | 验证配置合法性、同步配置 | `source` (user_settings/project_settings/...) |
| **InstructionsLoaded** | 加载 CLAUDE.md 时 | 记录指令来源、审计合规性 | `load_reason` (session_start/nested_traversal/...) |
| **CwdChanged** | 工作目录变化时 | 更新环境变量、注册文件监听 | 无 |

**环境变量注入**：
- CwdChanged/FileChanged Hook 可通过 `CLAUDE_ENV_FILE` 注入环境变量
- 导出的变量会应用到后续 Bash 命令（例如 `direnv` 集成）

### 5. 压缩生命周期（2 个事件）
| 事件名 | 触发时机 | 主要功能 | Matcher 字段 |
|--------|---------|---------|-------------|
| **PreCompact** | 压缩前 | 添加自定义压缩指令、阻止压缩 | `trigger` (manual/auto) |
| **PostCompact** | 压缩后 | 记录压缩摘要、通知用户 | `trigger` (manual/auto) |

### 6. 团队协作（3 个事件）
| 事件名 | 触发时机 | 主要功能 | Matcher 字段 |
|--------|---------|---------|-------------|
| **TeammateIdle** | Teammate 即将闲置时 | 验证任务完成、防止过早闲置 | 无 |
| **TaskCreated** | 创建任务时 | 任务验证、通知同步 | 无 |
| **TaskCompleted** | 标记任务完成时 | 验证完成条件、触发后续任务 | 无 |

### 7. Worktree 生命周期（3 个事件）
| 事件名 | 触发时机 | 主要功能 | Matcher 字段 |
|--------|---------|---------|-------------|
| **WorktreeCreate** | 创建隔离工作树时 | VCS 无关的环境隔离（非 git 场景） | 无 |
| **WorktreeRemove** | 删除工作树时 | 清理资源、备份数据 | 无 |
| **FileChanged** | 监听的文件变化时 | 触发重新加载、同步状态 | matcher 为文件名模式（如 `.envrc\|.env`） |

**关键特性**：
- **动态监听路径**：Hook 输出可返回 `hookSpecificOutput.watchPaths` 动态注册文件监听
- **VCS 无关**：WorktreeCreate/Remove 通过 Hook 委托外部工具，支持非 git 的版本控制系统

---

## Hook 执行器类型

Hook 系统支持 4 种执行器类型，通过 `type` 字段区分：

### 1. Command Hook（Shell 命令）
```typescript
{
  type: 'command',
  command: 'echo "Tool: $TOOL_NAME"',  // Shell 命令
  shell: 'bash' | 'powershell',        // 可选，默认 bash
  timeout: 60,                          // 秒，可选
  if: 'Bash(git *)',                    // 条件过滤，可选
  async: true,                          // 后台执行，可选
  asyncRewake: true,                    // 后台执行 + exit code 2 时唤醒模型
  once: true,                           // 执行一次后移除，可选
  statusMessage: 'Checking lint...'     // Spinner 显示信息
}
```

**关键特性**：
- **Shell 选择**：支持 `bash`（默认，使用 `$SHELL`）和 `powershell`（使用 pwsh）
- **JSON 输入**：Hook 输入通过 stdin 传递（JSON 格式），包含 `tool_name`、`tool_input` 等字段
- **退出码语义**：
  - `0` - 成功（stdout/stderr 是否显示取决于事件类型）
  - `2` - Blocking error（向模型展示 stderr 并阻止操作）
  - 其他 - 非阻塞错误（仅向用户显示 stderr）
- **异步模式**：
  - `async: true`：后台执行，不阻塞 AI
  - `asyncRewake: true`：后台执行，exit code 2 时唤醒模型（继续对话）

**实现细节**：
- 命令通过 `spawn` 执行，支持跨平台（macOS/Windows/Linux）
- 输出通过 `TaskOutput` 存储，支持流式读取
- 环境变量：`CLAUDE_ENV_FILE`（SessionStart/CwdChanged/FileChanged Hook 可写入环境变量）

### 2. Prompt Hook（LLM 评估）
```typescript
{
  type: 'prompt',
  prompt: 'Verify that tests passed. Output: $ARGUMENTS',
  model: 'claude-sonnet-4-6',  // 可选，默认 smallFastModel
  timeout: 30,                  // 秒，可选
  if: 'Bash(npm test)'          // 条件过滤，可选
}
```

**关键特性**：
- **$ARGUMENTS 占位符**：自动替换为 Hook 输入的 JSON 字符串
- **结构化输出**：要求模型返回 `{"ok": true}` 或 `{"ok": false, "reason": "..."}`
- **上下文传递**：可传入对话历史（messages 参数）

**实现路径**：[execPromptHook.ts](d:\AI\cc-haha\src\utils\hooks\execPromptHook.ts)
```typescript
// 核心逻辑
const response = await queryModelWithoutStreaming({
  messages: [...messages, userMessage],
  systemPrompt: asSystemPrompt([
    `You are evaluating a hook in Claude Code.
     Your response must be a JSON object matching one of the following schemas:
     1. If the condition is met, return: {"ok": true}
     2. If the condition is not met, return: {"ok": false, "reason": "..."}`
  ]),
  options: {
    model: hook.model ?? getSmallFastModel(),
    outputFormat: {
      type: 'json_schema',
      schema: { type: 'object', properties: { ok: {type: 'boolean'}, reason: {type: 'string'} }}
    }
  }
});
```

### 3. Agent Hook（多轮 Agent 验证）
```typescript
{
  type: 'agent',
  prompt: 'Verify that the migration ran successfully by checking database logs',
  model: 'claude-sonnet-4-6',  // 可选，默认 Haiku
  timeout: 60                   // 秒，可选
}
```

**关键特性**：
- **工具调用能力**：Agent 可调用 Read/Grep/Bash 等工具（排除 Agent/EnterPlanMode 等）
- **结构化输出工具**：自动注入 `StructuredOutput` 工具，强制 Agent 返回 `{"ok": boolean, "reason"?: string}`
- **最大轮数限制**：50 轮（避免无限循环）
- **会话隔离**：每个 Agent Hook 有独立的 `agentId` 和 transcript 文件

**实现路径**：[execAgentHook.ts](d:\AI\cc-haha\src\utils\hooks\execAgentHook.ts)
```typescript
// 核心流程
for await (const message of query({
  messages: agentMessages,
  systemPrompt: `You are verifying a stop condition in Claude Code. 
                 Use the available tools to verify the condition.
                 When done, return your result using the StructuredOutput tool.`,
  toolUseContext: agentToolUseContext,
  querySource: 'hook_agent',
})) {
  // 检测到 StructuredOutput 工具调用
  if (message.type === 'attachment' && message.attachment.type === 'structured_output') {
    structuredOutputResult = message.attachment.data;
    hookAbortController.abort();
    break;
  }
}
```

### 4. HTTP Hook（Webhook）
```typescript
{
  type: 'http',
  url: 'https://api.example.com/webhook',
  headers: {
    'Authorization': 'Bearer $MY_TOKEN',  // 支持环境变量插值
    'Content-Type': 'application/json'
  },
  allowedEnvVars: ['MY_TOKEN'],           // 白名单环境变量
  timeout: 60                              // 秒，可选
}
```

**关键特性**：
- **环境变量插值**：`$VAR_NAME` 和 `${VAR_NAME}` 语法，仅解析白名单中的变量
- **CRLF 注入防护**：自动清理 Header 值中的 `\r\n\x00` 字符
- **URL 白名单**：`allowedHttpHookUrls` 配置（支持 `*` 通配符）
- **SSRF 防护**：阻止私有 IP 地址（除 loopback）
- **代理支持**：
  - 沙箱代理（Sandbox 模式）
  - 环境变量代理（`HTTP_PROXY`/`HTTPS_PROXY`）

**实现路径**：[execHttpHook.ts](d:\AI\cc-haha\src\utils\hooks\execHttpHook.ts)
```typescript
// 环境变量插值
function interpolateEnvVars(value: string, allowedEnvVars: ReadonlySet<string>): string {
  return sanitizeHeaderValue(
    value.replace(/\$\{([A-Z_][A-Z0-9_]*)\}|\$([A-Z_][A-Z0-9_]*)/g, (_, braced, unbraced) => {
      const varName = braced ?? unbraced;
      return allowedEnvVars.has(varName) ? (process.env[varName] ?? '') : '';
    })
  );
}

// SSRF 防护
const response = await axios.post(hook.url, jsonInput, {
  lookup: sandboxProxy || envProxyActive ? undefined : ssrfGuardedLookup,
  // ... 其他配置
});
```

---

## 核心架构

### 1. 分层架构

```
┌────────────────────────────────────────────────────────────────┐
│                     用户层 & 插件层                              │
│  - settings.json (hooks 配置)                                   │
│  - 插件 hooks 配置 (plugin.json)                                │
│  - 运行时回调注册 (HookCallback)                                 │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                     配置加载与管理层                              │
│  - hooksConfigManager.ts: 聚合多源 Hook 配置                     │
│  - hooksSettings.ts: 解析 Matcher、优先级排序                    │
│  - loadPluginHooks.ts: 插件 Hook 加载与热重载                    │
│  - sessionHooks.ts: 会话级 Hook 注册（Function Hook）            │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                     Hook 调度与执行层                             │
│  - hooks.ts: 核心调度器（executeHooks、aggregateHookResults）    │
│  - AsyncHookRegistry.ts: 异步 Hook 管理                         │
│  - hookEvents.ts: Hook 事件广播（started/progress/response）     │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                     Hook 执行器层                                │
│  - execPromptHook.ts: LLM 单轮评估                              │
│  - execAgentHook.ts: 多轮 Agent 验证                            │
│  - execHttpHook.ts: Webhook 调用                                │
│  - hooks.ts (Command 部分): Shell 命令执行                       │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                     工具调用集成层                                │
│  - toolHooks.ts: PreToolUse/PostToolUse 集成                    │
│  - toolExecution.ts: 工具执行流程中嵌入 Hook                      │
│  - QueryEngine.ts: 主循环中的 Hook 触发点                        │
└────────────────────────────────────────────────────────────────┘
```

### 2. 数据流图

```
用户操作 → 事件触发
               ↓
     找到匹配的 Hook Matchers
               ↓
   ┌───────────┴───────────┐
   │   Matcher 匹配逻辑     │
   │  (tool_name, source,  │
   │   notification_type)  │
   └───────────┬───────────┘
               ↓
     按优先级排序 Hooks
     (特定 > 通配符 > 无 matcher)
               ↓
   ┌───────────┴───────────┐
   │  条件过滤 (if 字段)    │
   │  权限规则语法匹配      │
   └───────────┬───────────┘
               ↓
     并行执行 Hooks
     (Promise.allSettled)
               ↓
   ┌───────────┴───────────┐
   │  类型路由             │
   ├───────────────────────┤
   │ command → spawn       │
   │ prompt → LLM query    │
   │ agent → multi-turn    │
   │ http → axios.post     │
   └───────────┬───────────┘
               ↓
     聚合结果 (aggregateHookResults)
               ↓
   ┌───────────┴───────────────────┐
   │  Exit code 处理               │
   │  - 0: 成功                    │
   │  - 2: Blocking error          │
   │  - 其他: 非阻塞错误           │
   └───────────┬───────────────────┘
               ↓
     返回 AggregatedHookResult
     (决定是否继续、修改输入、更新权限)
```

### 3. 配置源优先级

Hook 配置来自 4 个来源，按优先级从高到低：

1. **运行时回调 (Function Hook)**
   - 代码直接注册：`registerHookCallbacks({ PreToolUse: [{ type: 'callback', callback: ... }] })`
   - 不可持久化（内存中），用于内部集成（如 `sessionFileAccessHooks.ts` 的审计日志）

2. **插件配置 (Plugin Hook)**
   - 插件的 `plugin.json` 中定义：`"hooks": { "PreToolUse": [{ matcher: "Write", hooks: [...] }] }`
   - 通过 `loadPluginHooks()` 加载，支持热重载

3. **会话级 Hook (Session Hook)**
   - 通过 `setSessionHookCallback()` 注册，生命周期仅限当前会话
   - 用于临时 Hook（如 Agent Hook 的 StructuredOutput 强制）

4. **用户/项目设置 (Settings Hook)**
   - `settings.json` 中配置：`"hooks": { "PreToolUse": [{ matcher: "Bash", hooks: [...] }] }`
   - 支持多层级（userSettings/projectSettings/localSettings/policySettings）

**优先级策略**：
- **Matcher 优先级**：特定 matcher > 通配符 matcher > 无 matcher
- **来源优先级**：所有来源的 Hook 合并执行（无覆盖），按 Matcher 优先级排序

---

## 配置与加载机制

### 1. Hook 配置结构

**Schema 定义**：[schemas/hooks.ts](d:\AI\cc-haha\src\schemas\hooks.ts)
```typescript
// settings.json 中的 hooks 字段
{
  "hooks": {
    "PreToolUse": [  // HookMatcher 数组
      {
        "matcher": "Bash",  // 可选，匹配 tool_name
        "hooks": [          // HookCommand 数组
          {
            "type": "command",
            "command": "echo 'Running bash command'",
            "if": "Bash(git *)",  // 条件过滤
            "timeout": 60
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [  // 无 matcher 的事件
          { "type": "prompt", "prompt": "Verify environment is ready" }
        ]
      }
    ]
  }
}
```

### 2. 插件 Hook 加载

**加载流程**：[loadPluginHooks.ts](d:\AI\cc-haha\src\utils\plugins\loadPluginHooks.ts)
```typescript
export const loadPluginHooks = memoize(async (): Promise<void> => {
  const { enabled } = await loadAllPluginsCacheOnly();
  const allPluginHooks: Record<HookEvent, PluginHookMatcher[]> = { ... };

  // 遍历所有启用的插件
  for (const plugin of enabled) {
    if (!plugin.hooksConfig) continue;
    const pluginMatchers = convertPluginHooksToMatchers(plugin);
    for (const event of Object.keys(pluginMatchers) as HookEvent[]) {
      allPluginHooks[event].push(...pluginMatchers[event]);
    }
  }

  // 原子化 clear + register（防止 Stop Hook 失效）
  clearRegisteredPluginHooks();
  registerHookCallbacks(allPluginHooks);
});
```

**热重载机制**：
- 监听 `policySettings` 变化（通过 `settingsChangeDetector`）
- 对比 `enabledPlugins`、`extraKnownMarketplaces` 等字段的快照
- 快照变化时清除缓存并重新加载 Hook

### 3. 会话级 Hook 注册

**API**：[sessionHooks.ts](d:\AI\cc-haha\src\utils\hooks\sessionHooks.ts)
```typescript
// 注册会话级 Hook（支持 agentId 隔离）
export function setSessionHookCallback(
  setAppState: (updater: (prev: AppState) => AppState) => void,
  event: HookEvent,
  matcher: string | null,
  hookCallback: HookCallback,
  agentId?: AgentId
): void {
  setAppState(prev => ({
    ...prev,
    sessionHooks: {
      ...prev.sessionHooks,
      [agentId ?? 'main']: {
        ...(prev.sessionHooks?.[agentId ?? 'main'] ?? {}),
        [event]: [
          ...(prev.sessionHooks?.[agentId ?? 'main']?.[event] ?? []),
          { matcher, hooks: [hookCallback] }
        ]
      }
    }
  }));
}

// 清除指定 agentId 的会话 Hook
export function clearSessionHooks(
  setAppState: (updater: (prev: AppState) => AppState) => void,
  agentId?: AgentId
): void { ... }
```

**使用场景**：
- Agent Hook 强制结构化输出（`registerStructuredOutputEnforcement`）
- 临时拦截特定工具调用
- A/B 测试功能

### 4. Matcher 匹配逻辑

**匹配算法**：[hooksConfigManager.ts](d:\AI\cc-haha\src\utils\hooks\hooksConfigManager.ts)
```typescript
// 获取事件对应的 Matcher 元数据
export const getHookEventMetadata = memoize((toolNames: string[]): Record<HookEvent, HookEventMetadata> => {
  return {
    PreToolUse: {
      matcherMetadata: {
        fieldToMatch: 'tool_name',  // 从 Hook 输入中提取 tool_name 字段
        values: toolNames            // 可选值（用于 UI 下拉菜单）
      }
    },
    Notification: {
      matcherMetadata: {
        fieldToMatch: 'notification_type',
        values: ['permission_prompt', 'idle_prompt', 'auth_success', ...]
      }
    },
    SessionStart: {
      matcherMetadata: {
        fieldToMatch: 'source',
        values: ['startup', 'resume', 'clear', 'compact']
      }
    }
  };
});

// Matcher 优先级排序
export function sortMatchersByPriority(
  matchers: string[],
  hooksByEventAndMatcher: Record<HookEvent, Record<string, IndividualHookConfig[]>>,
  event: HookEvent
): string[] {
  return matchers.sort((a, b) => {
    const aEmpty = a === '';
    const bEmpty = b === '';
    if (aEmpty && !bEmpty) return 1;   // 无 matcher 优先级最低
    if (!aEmpty && bEmpty) return -1;
    const aHasWildcard = a.includes('*');
    const bHasWildcard = b.includes('*');
    if (aHasWildcard && !bHasWildcard) return 1;  // 通配符次低
    if (!aHasWildcard && bHasWildcard) return -1;
    return 0;  // 特定 matcher 优先级最高
  });
}
```

### 5. 条件过滤（`if` 字段）

**语法**：权限规则语法（类似 `permissions` 配置）
```typescript
{
  "type": "command",
  "command": "pre-commit-hook.sh",
  "if": "Bash(git commit *)"  // 仅在 git commit 命令时触发
}
```

**实现**：[hooks.ts](d:\AI\cc-haha\src\utils\hooks.ts) 中的 `evaluateIfCondition`
```typescript
function evaluateIfCondition(
  ifCondition: string | undefined,
  hookInput: HookInput
): boolean {
  if (!ifCondition) return true;  // 无条件时始终执行

  // 解析权限规则（如 "Bash(git *)"）
  const parsedRule = permissionRuleValueFromString(ifCondition, ['allow']);
  if (!parsedRule) return false;

  // 检查 tool_name 和 tool_input 是否匹配
  if ('tool_name' in hookInput && 'tool_input' in hookInput) {
    const { tool_name, tool_input } = hookInput;
    return parsedRule.some(rule => 
      matchesToolPattern(rule, tool_name, tool_input)
    );
  }
  return false;
}
```

---

## 执行流程

### 1. Hook 触发入口

**主要触发点**：
```typescript
// 1. 工具调用前后 (toolExecution.ts)
const preToolUseResult = await executeHooks({
  event: 'PreToolUse',
  matcher: tool.name,
  hookInput: { tool_name: tool.name, tool_input: args, tool_use_id: toolUseID },
  signal: abortSignal
});

// 2. 会话启动 (sessionStart.ts)
await executeHooks({
  event: 'SessionStart',
  matcher: 'startup',
  hookInput: { source: 'startup' },
  toolUseContext
});

// 3. 用户提示提交 (handlePromptSubmit.ts)
const submitHookResult = await executeHooks({
  event: 'UserPromptSubmit',
  matcher: null,
  hookInput: { user_prompt: userInput },
  signal: abortSignal
});

// 4. 配置变更 (applySettingsChange.ts)
await executeHooks({
  event: 'ConfigChange',
  matcher: 'user_settings',
  hookInput: { source: 'user_settings', file_path: settingsPath }
});
```

### 2. 执行主流程

**核心函数**：[hooks.ts](d:\AI\cc-haha\src\utils\hooks.ts) 中的 `executeHooks`
```typescript
export async function executeHooks({
  event,
  matcher,
  hookInput,
  signal,
  toolUseContext,
  toolUseID
}: ExecuteHooksParams): Promise<AggregatedHookResult> {
  // 1. 收集所有匹配的 Hook（settings + plugins + session + callbacks）
  const allMatchers = [
    ...getSettingsHooks(appState, event),
    ...getPluginHooks(event),
    ...getSessionHooks(event, agentId),
    ...getRegisteredHooks()?.[event] ?? []
  ];

  // 2. 按 matcher 优先级排序（特定 > 通配符 > 无 matcher）
  const sortedMatchers = sortMatchersByPriority(allMatchers, event);

  // 3. 找到第一个匹配的 matcher 组
  const matchedHooks = findMatchingHooks(sortedMatchers, matcher);

  // 4. 条件过滤（if 字段）
  const filteredHooks = matchedHooks.filter(hook => 
    evaluateIfCondition(hook.if, hookInput)
  );

  // 5. 并行执行所有 Hook（Promise.allSettled）
  const results = await Promise.allSettled(
    filteredHooks.map(hook => executeHook(hook, hookInput, signal, toolUseContext))
  );

  // 6. 聚合结果
  return aggregateHookResults(results, event);
}
```

### 3. 单个 Hook 执行

**类型路由**：
```typescript
async function executeHook(
  hook: HookCommand | HookCallback,
  hookInput: HookInput,
  signal: AbortSignal,
  toolUseContext: ToolUseContext
): Promise<HookResult> {
  const jsonInput = JSON.stringify(hookInput);

  switch (hook.type) {
    case 'command':
      return executeCommandHook(hook, jsonInput, signal);
    case 'prompt':
      return execPromptHook(hook, hookEvent, jsonInput, signal, toolUseContext);
    case 'agent':
      return execAgentHook(hook, hookEvent, jsonInput, signal, toolUseContext);
    case 'http':
      return execHttpHook(hook, hookEvent, jsonInput, signal);
    case 'callback':
      return hook.callback(hookInput, toolUseID, signal);
  }
}
```

**Command Hook 执行**：
```typescript
async function executeCommandHook(
  hook: BashCommandHook,
  jsonInput: string,
  signal: AbortSignal
): Promise<HookResult> {
  // 1. 生成 Hook ID
  const hookId = randomUUID();
  
  // 2. 创建 Shell 命令
  const shellCommand = wrapSpawn({
    command: hook.command,
    stdin: jsonInput,  // 通过 stdin 传递 JSON 输入
    shell: hook.shell ?? 'bash',
    timeout: hook.timeout ? hook.timeout * 1000 : TOOL_HOOK_EXECUTION_TIMEOUT_MS,
    signal
  });

  // 3. 异步执行模式
  if (hook.async || hook.asyncRewake) {
    return executeInBackground({ hookId, shellCommand, asyncRewake: hook.asyncRewake });
  }

  // 4. 同步等待执行完成
  await shellCommand.waitForExit();

  // 5. 解析退出码
  const exitCode = shellCommand.exitCode ?? -1;
  const stdout = shellCommand.getStdout();
  const stderr = shellCommand.getStderr();

  // 6. 根据退出码返回结果
  if (exitCode === 0) {
    return { outcome: 'success', message: createHookSuccessMessage(stdout) };
  } else if (exitCode === 2) {
    return { outcome: 'blocking', blockingError: { command: hook.command, blockingError: stderr } };
  } else {
    return { outcome: 'non_blocking_error', message: createHookErrorMessage(stderr) };
  }
}
```

### 4. 结果聚合

**聚合逻辑**：[hooks.ts](d:\AI\cc-haha\src\utils\hooks.ts) 中的 `aggregateHookResults`
```typescript
function aggregateHookResults(
  results: PromiseSettledResult<HookResult>[],
  event: HookEvent
): AggregatedHookResult {
  const aggregated: AggregatedHookResult = {
    blockingErrors: [],
    additionalContexts: [],
    preventContinuation: false
  };

  for (const result of results) {
    if (result.status === 'rejected') {
      // Hook 执行异常（非退出码导致）
      aggregated.blockingErrors?.push({ command: '(error)', blockingError: result.reason });
      continue;
    }

    const hookResult = result.value;
    switch (hookResult.outcome) {
      case 'blocking':
        // Blocking error（exit code 2）
        aggregated.blockingErrors?.push(hookResult.blockingError!);
        aggregated.preventContinuation = true;
        break;

      case 'success':
        // 收集额外上下文（stdout）
        if (hookResult.additionalContext) {
          aggregated.additionalContexts?.push(hookResult.additionalContext);
        }
        // PreToolUse 可修改输入
        if (hookResult.updatedInput) {
          aggregated.updatedInput = { ...aggregated.updatedInput, ...hookResult.updatedInput };
        }
        // PostToolUse 可修改 MCP 工具输出
        if (hookResult.updatedMCPToolOutput) {
          aggregated.updatedMCPToolOutput = hookResult.updatedMCPToolOutput;
        }
        break;

      case 'non_blocking_error':
        // 记录错误消息但不阻断
        if (hookResult.message) {
          aggregated.message = hookResult.message;
        }
        break;

      case 'cancelled':
        // Hook 超时或被取消（忽略）
        break;
    }
  }

  // 特殊事件处理：SessionStart/SubagentStart 忽略 blocking error
  if (event === 'SessionStart' || event === 'SubagentStart') {
    aggregated.preventContinuation = false;
    aggregated.blockingErrors = undefined;
  }

  return aggregated;
}
```

### 5. 异步 Hook 管理

**后台执行**：[AsyncHookRegistry.ts](d:\AI\cc-haha\src\utils\hooks\AsyncHookRegistry.ts)
```typescript
export function registerPendingAsyncHook(
  hookId: string,
  shellCommand: ShellCommand,
  asyncRewake: boolean
): void {
  const registry = getAsyncHookRegistry();
  registry.set(hookId, {
    shellCommand,
    asyncRewake,
    startTime: Date.now()
  });

  // 监听 Hook 完成
  void shellCommand.waitForExit().then(() => {
    const exitCode = shellCommand.exitCode ?? -1;
    
    // asyncRewake: exit code 2 时唤醒模型
    if (asyncRewake && exitCode === 2) {
      const stderr = shellCommand.getStderr();
      enqueuePendingNotification({
        type: 'hook_rewake',
        hookId,
        message: stderr
      });
    }

    // 清理注册表
    registry.delete(hookId);
  });
}
```

---

## 安全与权限

### 1. 权限模型

**分层控制**：
```typescript
// 1. 策略级别（policySettings）
{
  "allowManagedHooksOnly": true,  // 仅允许 policySettings 中的 Hook
  "allowedHttpHookUrls": [        // HTTP Hook URL 白名单
    "https://internal.company.com/*",
    "https://api.example.com/webhooks/*"
  ],
  "httpHookAllowedEnvVars": [     // HTTP Hook 可用的环境变量白名单
    "API_TOKEN", "WEBHOOK_SECRET"
  ]
}

// 2. 用户级别（userSettings/projectSettings）
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "http",
            "url": "https://api.example.com/webhooks/bash-audit",
            "headers": { "Authorization": "Bearer $API_TOKEN" },
            "allowedEnvVars": ["API_TOKEN"]  // 必须在 policySettings 白名单中
          }
        ]
      }
    ]
  }
}
```

### 2. HTTP Hook 安全

**防护措施**：[execHttpHook.ts](d:\AI\cc-haha\src\utils\hooks\execHttpHook.ts)

#### a. URL 白名单
```typescript
function getHttpHookPolicy(): { allowedUrls: string[] | undefined } {
  const settings = getInitialSettings();
  return { allowedUrls: settings.allowedHttpHookUrls };
}

// undefined → 无限制
// [] → 阻止所有 HTTP Hook
// 非空 → 必须匹配白名单模式（支持 * 通配符）
if (policy.allowedUrls !== undefined) {
  const matched = policy.allowedUrls.some(p => urlMatchesPattern(hook.url, p));
  if (!matched) {
    return { ok: false, error: `Blocked by allowedHttpHookUrls` };
  }
}
```

#### b. CRLF 注入防护
```typescript
// 清理 Header 值中的 \r\n\x00 字符
function sanitizeHeaderValue(value: string): string {
  return value.replace(/[\r\n\x00]/g, '');
}
```

#### c. SSRF 防护
```typescript
import { ssrfGuardedLookup } from './ssrfGuard.js';

const response = await axios.post(hook.url, jsonInput, {
  // 阻止私有 IP 地址（除 loopback）
  lookup: sandboxProxy || envProxyActive ? undefined : ssrfGuardedLookup,
});

// ssrfGuard.ts 实现
export function ssrfGuardedLookup(
  hostname: string,
  options: LookupOptions,
  callback: (err: NodeJS.ErrnoException | null, address: string, family: number) => void
): void {
  dns.lookup(hostname, options, (err, address, family) => {
    if (err) return callback(err, address, family);
    
    // 阻止私有 IP 范围
    if (isPrivateIP(address)) {
      return callback(new Error(`Blocked private IP: ${address}`), '', 0);
    }
    
    callback(null, address, family);
  });
}
```

#### d. 环境变量白名单
```typescript
// 仅解析白名单中的环境变量
const allowedEnvVars = new Set(['API_TOKEN', 'WEBHOOK_SECRET']);
const headerValue = interpolateEnvVars('Bearer $API_TOKEN', allowedEnvVars);
// 'Bearer xxx' ✅

const blocked = interpolateEnvVars('Bearer $DANGEROUS_VAR', allowedEnvVars);
// 'Bearer ' ⚠️（未在白名单中，替换为空字符串）
```

### 3. Sandbox 集成

**沙箱代理**：
```typescript
async function getSandboxProxyConfig(): Promise<{ host: string; port: number } | undefined> {
  const { SandboxManager } = await import('../sandbox/sandbox-adapter.js');
  if (!SandboxManager.isSandboxingEnabled()) return undefined;

  // 等待沙箱网络代理初始化
  await SandboxManager.waitForNetworkInitialization();
  const proxyPort = SandboxManager.getProxyPort();
  return proxyPort ? { host: '127.0.0.1', port: proxyPort, protocol: 'http' } : undefined;
}

// HTTP Hook 请求通过沙箱代理（强制域名白名单）
const response = await axios.post(hook.url, jsonInput, {
  proxy: sandboxProxy ?? false,  // 沙箱代理 或 禁用代理
});
```

### 4. Hook 执行审计

**事件广播**：[hookEvents.ts](d:\AI\cc-haha\src\utils\hooks\hookEvents.ts)
```typescript
// Hook 开始
emitHookStarted(hookId, hookName, hookEvent);

// Hook 进度（流式输出）
emitHookProgress({ hookId, hookName, hookEvent, stdout, stderr, output });

// Hook 完成
emitHookResponse({
  hookId,
  hookName,
  hookEvent,
  output,
  stdout,
  stderr,
  exitCode,
  outcome: 'success' | 'error' | 'cancelled'
});
```

**审计日志**：
- 所有 Hook 执行记录到 `logForDebugging`（包含 stdout/stderr）
- SDK 可订阅 Hook 事件（`registerHookEventHandler`）
- Analytics 记录 Hook 执行耗时（`tengu_run_hook`）

---

## 技术亮点

### 1. 条件执行优化

**问题**：每次工具调用都触发所有 PreToolUse Hook，即使 Hook 只关心特定工具。

**解决方案**：`if` 字段 + 权限规则语法
```typescript
{
  "type": "command",
  "command": "lint-check.sh",
  "if": "Write(*.ts|*.tsx)"  // 仅在写入 TS 文件时触发
}
```

**性能提升**：
- 避免不必要的 `spawn` 调用
- 减少 Hook 执行总时间
- 用户体验更流畅（无多余 Spinner）

### 2. 异步执行 + 唤醒机制

**场景**：长时间运行的验证任务（如 CI 检查、大规模测试）

**设计**：
```typescript
{
  "type": "command",
  "command": "wait-for-ci.sh",
  "asyncRewake": true  // 后台执行，exit code 2 时唤醒模型
}
```

**工作流程**：
1. Hook 后台执行，AI 继续响应用户
2. `wait-for-ci.sh` 轮询 CI 状态
3. CI 失败时脚本返回 exit code 2 并输出错误信息
4. Claude Code 唤醒模型，将错误信息注入对话
5. AI 基于错误信息继续处理

**实现细节**：
```typescript
void shellCommand.waitForExit().then(() => {
  if (asyncRewake && shellCommand.exitCode === 2) {
    enqueuePendingNotification({
      type: 'hook_rewake',
      hookId,
      message: shellCommand.getStderr()
    });
  }
});
```

### 3. Agent Hook 的结构化输出强制

**问题**：Agent Hook 可能永不调用 `StructuredOutput` 工具，导致死循环或超时。

**解决方案**：注册会话级 Stop Hook 强制输出
```typescript
// execAgentHook.ts 中
registerStructuredOutputEnforcement(toolUseContext.setAppState, hookAgentId);

// hookHelpers.ts 实现
export function registerStructuredOutputEnforcement(
  setAppState: (updater: (prev: AppState) => AppState) => void,
  agentId: AgentId
): void {
  setSessionHookCallback(
    setAppState,
    'Stop',
    null,
    {
      type: 'callback',
      callback: async () => {
        // 检查是否已调用 StructuredOutput
        const lastMessage = getLastAssistantMessage();
        const hasStructuredOutput = lastMessage?.content.some(
          c => c.type === 'tool_use' && c.name === SYNTHETIC_OUTPUT_TOOL_NAME
        );

        if (!hasStructuredOutput) {
          // 强制返回 blocking error，阻止 Agent 结束
          return {
            outcome: 'blocking',
            blockingError: {
              command: 'Agent Stop Hook',
              blockingError: 'You must call the StructuredOutput tool before concluding.'
            }
          };
        }

        return { outcome: 'success' };
      }
    },
    agentId
  );
}
```

**效果**：
- Agent 必须调用 `StructuredOutput` 才能结束
- 避免超时浪费 token
- 保证 Hook 总是返回结构化结果

### 4. 插件 Hook 热重载

**问题**：远程托管设置更新插件列表后，已加载的 Hook 仍是旧配置。

**解决方案**：监听 `policySettings` 变化 + 快照对比
```typescript
export function setupPluginHookHotReload(): void {
  settingsChangeDetector.subscribe(source => {
    if (source === 'policySettings') {
      const newSnapshot = getPluginAffectingSettingsSnapshot();
      if (newSnapshot === lastPluginSettingsSnapshot) return;  // 无变化，跳过

      lastPluginSettingsSnapshot = newSnapshot;
      clearPluginCache('plugin-affecting settings changed');
      clearPluginHookCache();
      void loadPluginHooks();  // 重新加载 Hook
    }
  });
}
```

**快照内容**：
- `enabledPlugins`（启用的插件列表）
- `extraKnownMarketplaces`（额外的插件市场）
- `strictKnownMarketplaces`（严格模式市场）
- `blockedMarketplaces`（黑名单市场）

**关键细节**：
- 快照序列化时 **键排序**（避免顺序导致的误判）
- 使用 `memoize` 缓存 `loadPluginHooks`，避免重复加载

### 5. 原子化 Hook 注册

**问题**：`clearPluginHookCache()` 在任何地方被调用时，`STATE.registeredHooks` 被清空，导致 Stop Hook 失效（CC-29767）。

**解决方案**：Clear + Register 作为原子操作
```typescript
export const loadPluginHooks = memoize(async (): Promise<void> => {
  const allPluginHooks = await collectPluginHooks();
  
  // 原子化操作：先清除旧 Hook，立即注册新 Hook
  clearRegisteredPluginHooks();  // 仅清除插件 Hook，保留 callback Hook
  registerHookCallbacks(allPluginHooks);  // 立即注册新 Hook
});
```

**效果**：
- 旧 Hook 在清除前一直有效
- 新 Hook 注册后立即生效
- 无"空窗期"（Stop Hook 不会失效）

### 6. 环境变量文件注入

**场景**：`direnv`、`asdf` 等工具需要在 `cd` 后重新加载环境变量。

**设计**：`CLAUDE_ENV_FILE` 环境变量
```typescript
// CwdChanged Hook 示例
{
  "type": "command",
  "command": "direnv export bash > $CLAUDE_ENV_FILE"
}
```

**工作流程**：
1. `CwdChanged` Hook 触发
2. Hook 脚本将环境变量导出到 `$CLAUDE_ENV_FILE`（临时文件）
3. Claude Code 读取文件，解析 `export VAR=value` 语句
4. 后续 Bash 命令自动继承这些环境变量

**实现细节**：[sessionEnvironment.ts](d:\AI\cc-haha\src\utils\sessionEnvironment.ts)
```typescript
export async function loadSessionEnvFromFile(hookIndex: number): Promise<Record<string, string>> {
  const envFilePath = getHookEnvFilePath(hookIndex);
  const content = await fs.readFile(envFilePath, 'utf-8');
  const env: Record<string, string> = {};

  // 解析 export VAR=value 语句
  for (const line of content.split('\n')) {
    const match = line.match(/^export\s+([A-Z_][A-Z0-9_]*)=(.*)$/);
    if (match) {
      env[match[1]] = match[2].replace(/^['"]|['"]$/g, '');  // 去除引号
    }
  }

  return env;
}
```

### 7. Matcher 优先级策略

**问题**：多个 Matcher 匹配同一事件时，执行顺序如何确定？

**策略**：
1. **特定 Matcher** > 通配符 Matcher > 无 Matcher
2. 同级 Matcher 按定义顺序执行
3. 仅执行第一个匹配的 Matcher 组

**示例**：
```typescript
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash", "hooks": [...] },      // 优先级 1（特定）
      { "matcher": "Bash*", "hooks": [...] },     // 优先级 2（通配符）
      { "hooks": [...] }                           // 优先级 3（无 matcher）
    ]
  }
}
```

**实现**：[hooksSettings.ts](d:\AI\cc-haha\src\utils\hooks\hooksSettings.ts)
```typescript
export function sortMatchersByPriority(
  matchers: string[],
  hooksByEventAndMatcher: Record<HookEvent, Record<string, IndividualHookConfig[]>>,
  event: HookEvent
): string[] {
  return matchers.sort((a, b) => {
    const aEmpty = a === '';
    const bEmpty = b === '';
    if (aEmpty && !bEmpty) return 1;   // 无 matcher 最低
    if (!aEmpty && bEmpty) return -1;
    
    const aHasWildcard = a.includes('*');
    const bHasWildcard = b.includes('*');
    if (aHasWildcard && !bHasWildcard) return 1;  // 通配符次低
    if (!aHasWildcard && bHasWildcard) return -1;
    
    return 0;  // 特定 matcher 最高
  });
}
```

### 8. HTTP Hook 的代理透明性

**场景**：
- 企业环境需要通过代理访问外网（`HTTP_PROXY`/`HTTPS_PROXY`）
- Sandbox 模式需要强制域名白名单
- SSRF 防护需要阻止私有 IP

**设计**：
```typescript
const sandboxProxy = await getSandboxProxyConfig();
const envProxyActive = getProxyUrl() !== undefined && !shouldBypassProxy(hook.url);

const response = await axios.post(hook.url, jsonInput, {
  proxy: sandboxProxy ?? false,  // 沙箱代理 或 禁用代理（让 axios 拦截器处理）
  lookup: sandboxProxy || envProxyActive ? undefined : ssrfGuardedLookup,
});
```

**逻辑**：
1. **Sandbox 模式**：强制使用沙箱代理（域名白名单由代理强制）
2. **环境变量代理**：axios 拦截器自动处理（跳过 SSRF 检查）
3. **直连**：使用 SSRF 防护的 DNS Lookup

**避免的陷阱**：
- 企业代理常部署在私有 IP（如 `10.0.0.1:3128`），SSRF 检查会误拦截
- 沙箱代理和环境变量代理都跳过 SSRF 检查（代理负责 DNS）

---

## 代码统计

### 核心文件

| 文件路径 | 行数 | 功能 |
|---------|------|-----|
| [src/utils/hooks.ts](d:\AI\cc-haha\src\utils\hooks.ts) | ~1,500 | 核心调度器、Command Hook 执行 |
| [src/utils/hooks/execPromptHook.ts](d:\AI\cc-haha\src\utils\hooks\execPromptHook.ts) | 212 | Prompt Hook 执行器 |
| [src/utils/hooks/execAgentHook.ts](d:\AI\cc-haha\src\utils\hooks\execAgentHook.ts) | 340 | Agent Hook 执行器 |
| [src/utils/hooks/execHttpHook.ts](d:\AI\cc-haha\src\utils\hooks\execHttpHook.ts) | 243 | HTTP Hook 执行器 |
| [src/utils/hooks/hooksConfigManager.ts](d:\AI\cc-haha\src\utils\hooks\hooksConfigManager.ts) | 401 | Hook 配置聚合、Matcher 匹配 |
| [src/utils/hooks/hooksSettings.ts](d:\AI\cc-haha\src\utils\hooks\hooksSettings.ts) | ~800 | 配置解析、优先级排序 |
| [src/utils/hooks/sessionHooks.ts](d:\AI\cc-haha\src\utils\hooks\sessionHooks.ts) | ~200 | 会话级 Hook 注册 |
| [src/utils/hooks/hookEvents.ts](d:\AI\cc-haha\src\utils\hooks\hookEvents.ts) | 193 | Hook 事件广播 |
| [src/utils/plugins/loadPluginHooks.ts](d:\AI\cc-haha\src\utils\plugins\loadPluginHooks.ts) | 288 | 插件 Hook 加载与热重载 |
| [src/types/hooks.ts](d:\AI\cc-haha\src\types\hooks.ts) | 291 | 类型定义 |
| [src/schemas/hooks.ts](d:\AI\cc-haha\src\schemas\hooks.ts) | 223 | Zod Schema 定义 |

**总计**：约 **12,000 行 TypeScript 代码**（包括类型定义、Schema、测试等）

### Hook 事件覆盖

- **27 个生命周期事件**
- **4 种执行器类型**（command/prompt/agent/http）
- **20+ 个配置字段**（timeout/if/async/statusMessage/headers 等）

---

## 使用示例

### 1. Git Commit 前验证测试

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that all unit tests pass before allowing the commit",
            "if": "Bash(git commit *)",
            "timeout": 120,
            "statusMessage": "Verifying tests..."
          }
        ]
      }
    ]
  }
}
```

### 2. 敏感文件写入告警

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "http",
            "url": "https://security.company.com/api/file-write-audit",
            "if": "Write(*.env|secrets/*)",
            "headers": {
              "Authorization": "Bearer $AUDIT_TOKEN"
            },
            "allowedEnvVars": ["AUDIT_TOKEN"]
          }
        ]
      }
    ]
  }
}
```

### 3. Session 启动时加载项目配置

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "cat .project-context.md",
            "statusMessage": "Loading project context..."
          }
        ]
      }
    ]
  }
}
```

### 4. 文件变更时自动加载环境变量（direnv 集成）

```json
{
  "hooks": {
    "CwdChanged": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "direnv export bash > $CLAUDE_ENV_FILE",
            "statusMessage": "Loading direnv..."
          }
        ]
      }
    ],
    "FileChanged": [
      {
        "matcher": ".envrc",
        "hooks": [
          {
            "type": "command",
            "command": "direnv export bash > $CLAUDE_ENV_FILE"
          }
        ]
      }
    ]
  }
}
```

### 5. CI 状态监控（异步 + 唤醒）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "wait-for-ci.sh",
            "if": "Bash(git push *)",
            "asyncRewake": true,
            "statusMessage": "Monitoring CI..."
          }
        ]
      }
    ]
  }
}
```

`wait-for-ci.sh` 脚本示例：
```bash
#!/bin/bash
# 轮询 CI 状态，失败时返回 exit code 2

while true; do
  status=$(gh run list --json status --jq '.[0].status')
  if [[ "$status" == "completed" ]]; then
    conclusion=$(gh run list --json conclusion --jq '.[0].conclusion')
    if [[ "$conclusion" == "failure" ]]; then
      echo "CI failed! See logs at: $(gh run list --json url --jq '.[0].url')" >&2
      exit 2  # 触发 asyncRewake，唤醒模型
    fi
    exit 0  # CI 成功
  fi
  sleep 30
done
```

---

## 总结

Claude Code 的 Hook 系统是一个设计精良、功能强大的扩展机制：

### 核心优势
1. **全生命周期覆盖**：27 个事件涵盖工具调用、会话管理、配置变更等所有关键节点
2. **多执行器支持**：Shell/LLM/Agent/HTTP 满足不同场景需求
3. **灵活的配置系统**：支持 Matcher、条件过滤、优先级排序
4. **异步执行能力**：后台 Hook + 唤醒机制支持长时间验证任务
5. **安全防护完善**：URL 白名单、SSRF 防护、环境变量白名单、CRLF 注入防护

### 技术亮点
- **条件执行优化**（`if` 字段）减少不必要的 Hook 调用
- **原子化注册**避免 Hook 失效窗口期
- **Agent Hook 强制结构化输出**保证可靠性
- **环境变量文件注入**支持 `direnv` 等工具集成
- **插件热重载**支持远程托管设置实时生效

### 适用场景
- **CI/CD 集成**：提交前验证、构建监控、部署审计
- **安全合规**：敏感文件告警、操作审计、权限控制
- **开发工作流**：代码质量检查、依赖管理、环境同步
- **可观察性**：工具调用日志、性能监控、异常告警

Hook 系统是 Claude Code 可扩展性的基石，通过事件驱动的设计和多样化的执行器，赋予用户和插件强大的定制能力，同时保持核心代码的简洁与稳定。
