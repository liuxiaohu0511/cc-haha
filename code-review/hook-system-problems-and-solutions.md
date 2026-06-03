# Hook 系统要解决的核心问题与解决方案

## 目录
1. [可扩展性困境](#1-可扩展性困境)
2. [跨团队协作问题](#2-跨团队协作问题)
3. [运行时验证问题](#3-运行时验证问题)
4. [环境集成问题](#4-环境集成问题)
5. [可观察性问题](#5-可观察性问题)
6. [插件系统的基础](#6-插件系统的基础)
7. [Hook 系统的本质](#hook-系统的本质)
8. [实际案例](#实际案例)

---

## 1. 可扩展性困境

### 问题描述

假设你要在 AI 执行 `git commit` 前做一些检查：

```typescript
// ❌ 不使用 Hook 的做法：直接修改 BashTool 代码
async call(input, context) {
  // 核心逻辑
  const result = await executeBashCommand(input.command);
  
  // 需求1：公司要求 commit 前运行 lint
  if (input.command.includes('git commit')) {
    const lintResult = await runLint();
    if (!lintResult.success) {
      throw new Error('Lint failed');
    }
  }
  
  // 需求2：安全团队要求记录所有 git 操作
  if (input.command.includes('git')) {
    await logToAuditSystem(input.command);
  }
  
  // 需求3：A 团队要求 commit 前检查 Jira ticket
  if (input.command.includes('git commit')) {
    await validateJiraTicket(input.command);
  }
  
  return result;
}
```

**问题总结**：
- ❌ 核心代码越来越臃肿（每个需求都要改 BashTool）
- ❌ 团队间冲突（A 团队加的代码影响 B 团队）
- ❌ 无法动态配置（代码写死，需要重新编译）
- ❌ 测试困难（所有逻辑耦合在一起）

### 解决方案

使用 Hook 系统，用户自己配置，无需修改核心代码：

```json
// settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint",
            "if": "Bash(git commit *)"
          },
          {
            "type": "http",
            "url": "https://audit.company.com/api/log",
            "if": "Bash(git *)"
          },
          {
            "type": "agent",
            "prompt": "Verify commit message contains Jira ticket ID",
            "if": "Bash(git commit *)"
          }
        ]
      }
    ]
  }
}
```

**优势**：
- ✅ **零侵入**：BashTool 代码不变
- ✅ **动态配置**：用户/项目/插件各自配置，互不干扰
- ✅ **可组合**：多个 Hook 并行执行
- ✅ **可测试**：每个 Hook 独立测试

---

## 2. 跨团队协作问题

### 问题描述

一个大型项目有多个团队：
- **安全团队**：要求审计所有文件写入操作
- **合规团队**：禁止访问某些敏感路径
- **DevOps 团队**：commit 前自动运行测试
- **产品团队**：需要统计工具使用频率

传统做法是将所有团队的需求都塞进核心代码：

```typescript
// ❌ 所有团队的需求都塞进核心代码
async call(input, context) {
  // 安全团队的需求
  await auditFileWrite(input.path, input.content);
  
  // 合规团队的需求
  if (SENSITIVE_PATHS.includes(input.path)) {
    throw new Error('Access denied');
  }
  
  // DevOps 团队的需求
  if (input.command.includes('commit')) {
    await runTests();
  }
  
  // 产品团队的需求
  await trackToolUsage('FileWrite', input);
  
  // 核心逻辑（被淹没了）
  return fs.writeFile(input.path, input.content);
}
```

**问题总结**：
- ❌ 代码职责不清（安全、合规、DevOps、产品逻辑混在一起）
- ❌ 团队间冲突（所有人都要改同一个文件）
- ❌ 无法独立部署（一个团队的需求变更影响所有人）

### 解决方案

不同层级的配置文件，各团队管理自己的配置：

```typescript
// 1. policySettings.json (安全团队管理，只读)
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.internal/api/file-write"
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "agent",
            "prompt": "Check if path ${tool_input.path} is in sensitive list"
          }
        ]
      }
    ]
  }
}

// 2. projectSettings.json (DevOps 团队管理)
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "npm test",
            "if": "Bash(git commit *)"
          }
        ]
      }
    ]
  }
}

// 3. Plugin: product-analytics (产品团队开发)
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://analytics.internal/api/track"
          }
        ]
      }
    ]
  }
}
```

**优势**：
- ✅ **职责分离**：每个团队管理自己的配置
- ✅ **无冲突**：所有 Hook 并行执行，互不影响
- ✅ **细粒度控制**：policySettings 可以覆盖 userSettings
- ✅ **审计友好**：所有 Hook 执行日志集中记录

---

## 3. 运行时验证问题

### 问题 A：Stop Hook（验证任务完成）

#### 问题描述

用户：帮我实现登录功能，完成后必须确保测试通过

```typescript
// ❌ 没有 Hook：AI 说"完成了"，但实际没测试
AI: I've implemented the login feature. The code is ready.
User: Did you run the tests?
AI: Oh, let me do that now... [发现测试失败]
```

**问题总结**：
- ❌ AI 可能"说谎"（认为完成了，实际没做完）
- ❌ 用户需要手动提醒（"你测试了吗？"）
- ❌ 缺乏强制机制（AI 可以跳过验证）

#### 解决方案

使用 Stop Hook 强制验证：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that all unit tests pass before concluding"
          }
        ]
      }
    ]
  }
}
```

**工作流程**：
1. AI 尝试结束回复
2. Stop Hook 触发
3. Agent Hook 检查测试状态
4. 测试失败 → 返回 blocking error
5. AI 被迫继续工作，直到测试通过

**优势**：
- ✅ **强制验证**：AI 无法说"完成"除非测试通过
- ✅ **自动化**：用户无需手动提醒
- ✅ **可配置**：不同项目配置不同的完成条件

### 问题 B：PreToolUse Hook（防止危险操作）

#### 问题描述

用户在生产环境工作，AI 要执行 `rm -rf /data`

```typescript
// ❌ 没有 Hook：操作执行 → 数据丢失 → 灾难
AI: I'll clean up the data directory.
[执行 rm -rf /data]
[生产数据全部丢失]
```

**问题总结**：
- ❌ 缺乏提前验证（危险操作已经执行）
- ❌ 环境无感知（不知道当前在生产环境）
- ❌ 无法撤销（数据已丢失）

#### 解决方案

使用 PreToolUse Hook 拦截危险命令：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "check-dangerous-command.sh",
            "if": "Bash(rm *)"
          }
        ]
      }
    ]
  }
}
```

```bash
# check-dangerous-command.sh
#!/bin/bash
if [[ "$CLAUDE_ENV" == "production" ]] && [[ "$1" =~ "rm -rf" ]]; then
  echo "Blocked: Dangerous rm command in production" >&2
  exit 2  # Exit code 2 = blocking error
fi
```

**工作流程**：
1. AI 调用 `Bash({ command: "rm -rf /data" })`
2. PreToolUse Hook 触发
3. `check-dangerous-command.sh` 检测到危险命令
4. 返回 exit code 2 → blocking error
5. 工具执行被阻止

**优势**：
- ✅ **提前拦截**：操作执行前阻止
- ✅ **环境感知**：根据环境配置不同策略
- ✅ **零损失**：危险操作从未执行

---

## 4. 环境集成问题

### 问题 A：direnv 集成

#### 问题描述

你的项目使用 `direnv` 管理环境变量，AI 执行命令时需要加载这些环境。

```typescript
// ❌ 没有 Hook：AI 不知道 direnv，命令在错误的环境执行
User: Run npm test
AI: [执行 npm test]
Error: Node version 16 required, but current is 14
```

**问题总结**：
- ❌ 环境变量未加载（AI 使用系统默认环境）
- ❌ 用户需要手动提醒（"先运行 direnv allow"）
- ❌ 每次切换目录都要重新加载

#### 解决方案

使用 CwdChanged 和 FileChanged Hook 自动加载 direnv：

```json
{
  "hooks": {
    "CwdChanged": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "direnv export bash > $CLAUDE_ENV_FILE"
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

**工作流程**：
1. AI 执行 `cd /path/to/project` → CwdChanged Hook 触发
2. `direnv export bash` 导出环境变量到 `$CLAUDE_ENV_FILE`
3. Claude Code 读取文件，解析 `export VAR=value` 语句
4. 后续 Bash 命令自动继承这些环境变量
5. AI 执行 `npm test` → 使用正确的 Node 版本 ✅

**优势**：
- ✅ **无缝集成**：AI 自动适配用户的开发环境
- ✅ **自动触发**：切换目录或修改 `.envrc` 时自动重新加载
- ✅ **零配置**：用户无需手动运行 `direnv allow`

### 问题 B：CI 状态监控

#### 问题描述

用户：推送代码后，自动监控 CI 状态，失败时通知我

```typescript
// ❌ 没有 Hook：用户需要手动检查 CI
User: git push
AI: Pushed successfully.
[用户等待 5 分钟]
User: Check CI status
AI: CI failed. Let me fix it.
```

**问题总结**：
- ❌ 用户需要主动询问（"CI 状态如何？"）
- ❌ AI 无法主动通知（推送后就结束了）
- ❌ 浪费时间（等待期间 AI 空闲）

#### 解决方案

使用 asyncRewake Hook 实现异步监控：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "monitor-ci.sh",
            "if": "Bash(git push *)",
            "asyncRewake": true
          }
        ]
      }
    ]
  }
}
```

```bash
# monitor-ci.sh
#!/bin/bash
while true; do
  status=$(gh run list --json status --jq '.[0].status')
  if [[ "$status" == "completed" ]]; then
    conclusion=$(gh run list --json conclusion --jq '.[0].conclusion')
    if [[ "$conclusion" == "failure" ]]; then
      echo "CI failed! Logs: $(gh run list --json url --jq '.[0].url')" >&2
      exit 2  # 唤醒 AI，继续对话
    fi
    exit 0  # CI 成功，静默结束
  fi
  sleep 30
done
```

**工作流程**：
1. AI 执行 `git push` → PostToolUse Hook 触发
2. `monitor-ci.sh` 在后台运行（`asyncRewake: true`）
3. AI 继续响应用户（不阻塞）
4. Hook 每 30 秒检查一次 CI 状态
5. CI 失败 → Hook 返回 exit code 2
6. Claude Code 唤醒 AI，将错误消息注入对话
7. AI 基于错误日志自动修复

**优势**：
- ✅ **异步监控**：不阻塞 AI
- ✅ **主动通知**：CI 失败时自动唤醒 AI
- ✅ **自动修复**：AI 直接看到错误日志，无需用户转述

---

## 5. 可观察性问题

### 问题 A：审计日志

#### 问题描述

合规要求：记录所有敏感操作（写入 secrets/、修改 .env 文件等）

```typescript
// ❌ 没有 Hook：需要修改每个工具的核心代码
class FileWriteTool {
  async call(input, context) {
    // 添加审计逻辑（侵入核心代码）
    if (input.path.includes('secrets/') || input.path.endsWith('.env')) {
      await logToAuditSystem({
        tool: 'Write',
        path: input.path,
        user: context.userId,
        timestamp: new Date()
      });
    }
    
    return fs.writeFile(input.path, input.content);
  }
}
```

**问题总结**：
- ❌ 侵入核心代码（每个工具都要加审计逻辑）
- ❌ 难以维护（审计策略变更需要改多处）
- ❌ 容易遗漏（新增工具可能忘记加审计）

#### 解决方案

使用 PreToolUse Hook 统一审计：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.company.com/api/log",
            "if": "Write(secrets/*|*.env)",
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

**审计系统收到的数据**：
```json
{
  "tool_name": "Write",
  "tool_input": {
    "file_path": "secrets/api-key.txt",
    "content": "[REDACTED]"
  },
  "tool_use_id": "toolu_xxx",
  "user_id": "alice",
  "timestamp": "2026-06-03T10:30:00Z"
}
```

**优势**：
- ✅ **零侵入**：核心代码不需要添加日志逻辑
- ✅ **集中管理**：所有审计规则在一个配置文件
- ✅ **灵活过滤**：`if` 字段精确控制审计范围

### 问题 B：性能监控

#### 问题描述

需要监控每个工具的执行耗时，优化性能瓶颈。

```typescript
// ❌ 没有 Hook：需要修改每个工具
class GrepTool {
  async call(input, context) {
    const startTime = Date.now();
    const result = await ripGrep(...);
    const duration = Date.now() - startTime;
    
    // 上报耗时（侵入核心代码）
    await reportMetrics({
      tool: 'Grep',
      duration,
      numFiles: result.numFiles
    });
    
    return result;
  }
}
```

**问题总结**：
- ❌ 重复代码（每个工具都要加计时逻辑）
- ❌ 容易出错（忘记在 catch 块中上报）

#### 解决方案

使用 PostToolUse Hook 统一监控：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://metrics.internal/api/tool-duration"
          }
        ]
      }
    ]
  }
}
```

**Hook 输入**（自动包含耗时）：
```json
{
  "tool_name": "Grep",
  "tool_input": { "pattern": "login", "path": "src/" },
  "tool_output": { "numFiles": 15, "numLines": 42 },
  "duration_ms": 850
}
```

**优势**：
- ✅ **自动化**：无需手动添加计时代码
- ✅ **统一上报**：所有工具使用相同的监控格式
- ✅ **容错**：Hook 失败不影响工具执行

---

## 6. 插件系统的基础

### 问题描述

你想让第三方开发者扩展 Claude Code，但又不想让他们修改核心代码。

```typescript
// ❌ 传统插件系统：需要暴露复杂的 API
class PluginAPI {
  registerToolInterceptor(toolName: string, callback: Function);
  registerEventListener(event: string, callback: Function);
  modifyToolResult(toolName: string, transformer: Function);
  // ... 需要设计大量 API
}

// 插件开发者需要学习这些 API
plugin.register(api => {
  api.registerToolInterceptor('Bash', async (input) => {
    if (input.command.includes('git commit')) {
      // 提取 Jira ticket
    }
  });
});
```

**问题总结**：
- ❌ 插件 API 设计复杂（需要预见所有扩展点）
- ❌ 学习成本高（开发者需要学习 API）
- ❌ 侵入性强（插件可能修改核心行为）

### 解决方案

使用 Hook 系统，插件只需配置 JSON：

**插件示例：Jira 集成**
```json
// plugin.json
{
  "name": "jira-integration",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Extract Jira ticket ID from commit message: $ARGUMENTS",
            "if": "Bash(git commit *)"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "http",
            "url": "https://jira.company.com/api/comment",
            "if": "Bash(git commit *)"
          }
        ]
      }
    ]
  }
}
```

**插件示例：Slack 通知**
```json
{
  "name": "slack-notifier",
  "hooks": {
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://hooks.slack.com/services/xxx",
            "headers": {
              "Content-Type": "application/json"
            }
          }
        ]
      }
    ]
  }
}
```

**优势**：
- ✅ **安全隔离**：插件只能通过 Hook 扩展，无法修改核心
- ✅ **零学习成本**：配置 JSON 即可，无需学习 API
- ✅ **动态加载**：用户启用/禁用插件，Hook 自动热重载
- ✅ **市场生态**：第三方开发者可以发布插件到 marketplace

---

## Hook 系统的本质

### 1. 控制反转（IoC）

```
传统插件系统：核心代码主动调用插件
┌─────────────┐          ┌─────────────┐
│ 核心代码     │  调用     │  扩展逻辑    │
│             │ ───────>  │             │
│ (主动)      │          │  (被动)      │
└─────────────┘          └─────────────┘

Hook 系统：核心代码在关键点"询问"外部逻辑
┌─────────────┐          ┌─────────────┐
│ 核心代码     │  通知     │  Hook 逻辑   │
│             │ <───────  │             │
│ (被动)      │  返回决策  │  (主动)      │
└─────────────┘          └─────────────┘
```

### 2. 策略模式（Strategy Pattern）

不同场景配置不同策略：

```typescript
// 开发环境：允许所有操作
{
  "hooks": {}
}

// 测试环境：运行测试后才能 commit
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "npm test", "if": "Bash(git commit *)" }
        ]
      }
    ]
  }
}

// 生产环境：禁止危险操作 + 审计所有写入
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "check-dangerous.sh", "if": "Bash(rm *)" }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          { "type": "http", "url": "https://audit.internal/api/log" }
        ]
      }
    ]
  }
}
```

### 3. 责任链模式（Chain of Responsibility）

多个 Hook 串行/并行执行：

```typescript
PreToolUse Hooks:
  1. 安全扫描 → 通过
  2. 权限检查 → 通过
  3. 业务验证 → 失败 → 阻止工具执行

// 伪代码
const results = await Promise.allSettled([
  securityScanHook(input),
  permissionCheckHook(input),
  businessValidationHook(input)
]);

const hasBlocking = results.some(r => r.value.outcome === 'blocking');
if (hasBlocking) {
  return { error: 'Tool execution blocked by hooks' };
}
```

### 4. 事件驱动架构（Event-Driven Architecture）

```typescript
// 生命周期事件
SessionStart → 加载环境变量、初始化配置
PreToolUse → 验证参数、安全检查
PostToolUse → 记录日志、上报指标
Stop → 验证任务完成
SessionEnd → 清理资源、保存状态
```

---

## 为什么 Hook 比传统插件系统更好？

| 特性 | 传统插件系统 | Hook 系统 |
|------|------------|----------|
| **扩展点** | 插件 API（需要设计） | 生命周期事件（自然存在） |
| **侵入性** | 插件可能修改核心行为 | 仅在事件点注入逻辑 |
| **冲突处理** | 插件间可能冲突 | Hook 并行执行，结果聚合 |
| **动态配置** | 需要重启 | 热重载 |
| **学习成本** | 需要学习插件 API | 配置文件即可 |
| **调试难度** | 插件错误影响核心 | Hook 错误隔离 |
| **安全性** | 插件有完整访问权限 | Hook 仅能读取输入/输出 |

---

## 实际案例

### 案例 1：防止数据泄露

**场景**：公司禁止提交包含 API key 的代码

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "git diff --cached | grep -E 'api[_-]?key|secret|password' && exit 2 || exit 0",
            "if": "Bash(git commit *)",
            "statusMessage": "Checking for secrets..."
          }
        ]
      }
    ]
  }
}
```

**工作流程**：
1. AI 执行 `git commit -m "Add login"`
2. PreToolUse Hook 触发
3. `git diff --cached | grep ...` 检测暂存区是否包含敏感词
4. 检测到 `api_key` → 返回 exit code 2 → blocking error
5. 工具执行被阻止，AI 看到错误消息：
   ```
   Hook blocked tool execution: Found sensitive data in commit
   ```
6. AI 自动移除敏感数据，重新提交

**没有 Hook 的后果**：
- 需要修改 BashTool 核心代码
- 或依赖 git pre-commit hook（AI 可能通过 `--no-verify` 绕过）

---

### 案例 2：自动化工作流

**场景**：提交代码 → 等待 CI → 失败时自动修复

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "wait-for-ci-and-notify.sh",
            "if": "Bash(git push *)",
            "asyncRewake": true,
            "statusMessage": "Monitoring CI status..."
          }
        ]
      }
    ]
  }
}
```

```bash
# wait-for-ci-and-notify.sh
#!/bin/bash
for i in {1..60}; do  # 最多等待 30 分钟
  status=$(gh run list --limit 1 --json status,conclusion --jq '.[0]')
  
  if [[ $(echo "$status" | jq -r '.status') == "completed" ]]; then
    conclusion=$(echo "$status" | jq -r '.conclusion')
    
    if [[ "$conclusion" == "failure" ]]; then
      # CI 失败，唤醒 AI
      logs=$(gh run view --log-failed)
      echo "CI failed! Failed tests:\n$logs" >&2
      exit 2  # asyncRewake: exit code 2 = 唤醒 AI
    fi
    
    # CI 成功，静默结束
    exit 0
  fi
  
  sleep 30
done

# 超时（30 分钟），静默结束
exit 0
```

**工作流程**：
1. AI 执行 `git push origin main`
2. PostToolUse Hook 触发
3. `wait-for-ci-and-notify.sh` 在后台运行
4. AI 继续响应用户（不阻塞）
5. Hook 每 30 秒检查一次 CI 状态
6. CI 失败 → Hook 返回 exit code 2
7. Claude Code 将 stderr 注入对话，唤醒 AI
8. AI 看到失败日志：
   ```
   CI failed! Failed tests:
   - test/auth/login.test.ts: LoginForm should validate email format
   - test/auth/login.test.ts: LoginForm should show error on invalid credentials
   ```
9. AI 自动修复测试，重新推送

**没有 Hook 的后果**：
- 用户需要手动检查 CI
- 用户需要手动复制错误日志
- 用户需要手动通知 AI
- 整个过程浪费 5-10 分钟

---

### 案例 3：合规审计

**场景**：所有文件操作必须记录到审计系统（SOC 2 合规要求）

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|NotebookEdit",
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.internal/api/log",
            "headers": {
              "Authorization": "Bearer $AUDIT_TOKEN",
              "Content-Type": "application/json"
            },
            "allowedEnvVars": ["AUDIT_TOKEN"]
          }
        ]
      }
    ]
  }
}
```

**审计系统收到的数据**：
```json
{
  "event_type": "file_write",
  "tool_name": "Write",
  "tool_use_id": "toolu_abc123",
  "user_id": "alice@company.com",
  "file_path": "/home/alice/project/src/auth/login.ts",
  "file_size": 1024,
  "timestamp": "2026-06-03T14:30:00Z",
  "session_id": "sess_xyz789"
}
```

**审计报告示例**：
```
合规报告 - 2026年6月
总文件操作数：1,234
用户分布：
  - alice@company.com: 456 次
  - bob@company.com: 321 次
  - charlie@company.com: 457 次

敏感文件访问：
  - secrets/api-keys.txt: 12 次（用户：alice, bob）
  - .env.production: 8 次（用户：charlie）

异常活动：
  - 2026-06-15 02:30 - alice 在凌晨修改生产配置（已标记）
```

**没有 Hook 的后果**：
- 需要修改 Write/Edit/NotebookEdit 三个工具的核心代码
- 审计逻辑与核心逻辑耦合
- 新增工具（如 `FileReplace`）可能忘记加审计
- 审计策略变更需要改多处代码

---

### 案例 4：多环境配置

**场景**：开发/测试/生产环境使用不同的 Hook 策略

```typescript
// .claude/settings.json (开发环境)
{
  "hooks": {
    // 开发环境：无限制，快速迭代
  }
}

// .claude/settings.json (测试环境)
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "npm test",
            "if": "Bash(git commit *)",
            "statusMessage": "Running tests before commit..."
          }
        ]
      }
    ]
  }
}

// .claude/settings.json (生产环境)
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that command is safe for production environment: $ARGUMENTS",
            "if": "Bash(*)",
            "timeout": 30
          }
        ]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.internal/api/production-change"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that rollback plan is documented before concluding"
          }
        ]
      }
    ]
  }
}
```

**优势**：
- ✅ 开发环境：快速迭代，无限制
- ✅ 测试环境：自动运行测试
- ✅ 生产环境：AI 验证 + 审计 + 回滚计划

---

## 总结

### Hook 系统的核心价值

1. **可扩展性**：无需修改核心代码即可扩展功能
2. **可配置性**：不同环境/团队/用户配置不同策略
3. **可组合性**：多个 Hook 并行执行，互不干扰
4. **可观察性**：统一的审计、监控、日志记录
5. **安全性**：提前验证、运行时拦截、持续监控

### 对比总结

**如果没有 Hook 系统**，每个需求都需要：
- ❌ 修改核心代码（风险高）
- ❌ 重新编译发布（周期长）
- ❌ 所有用户强制升级（体验差）
- ❌ 团队间冲突（维护难）

**有了 Hook 系统**，扩展变成了：
- ✅ 修改配置文件（零风险）
- ✅ 立即生效（零延迟）
- ✅ 用户自选（零强制）
- ✅ 各自独立（零冲突）

### 设计原则

Hook 系统体现的核心设计原则：

1. **开闭原则（Open-Closed Principle）**
   - 对扩展开放：通过配置 Hook 扩展功能
   - 对修改封闭：核心代码无需修改

2. **单一职责原则（Single Responsibility Principle）**
   - 核心代码：专注业务逻辑
   - Hook 逻辑：处理扩展需求

3. **依赖倒置原则（Dependency Inversion Principle）**
   - 核心不依赖具体 Hook 实现
   - Hook 依赖核心提供的事件接口

4. **最少知识原则（Law of Demeter）**
   - Hook 只能访问事件输入/输出
   - 无法修改核心状态

这就是 Hook 系统的价值所在——**用最小的侵入性，实现最大的可扩展性**。
