# 记忆系统索引算法深度分析

> 文档生成时间: 2026-05-29  
> 分析对象: Claude Code Haha 记忆系统  
> 代码量: 1,770 行 TypeScript

---

## 📋 目录

1. [架构概览](#架构概览)
2. [索引结构设计](#索引结构设计)
3. [核心算法](#核心算法)
4. [智能检索机制](#智能检索机制)
5. [性能优化](#性能优化)
6. [数据流分析](#数据流分析)
7. [技术亮点](#技术亮点)

---

## 架构概览

### 系统分层

```
┌─────────────────────────────────────────────────────────────┐
│                     AI 交互层                                │
│  Claude 通过 Write/Edit/Read 工具读写记忆文件               │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                  索引管理层 (memdir.ts)                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  buildMemoryPrompt()  - 构建系统提示词               │  │
│  │  loadMemoryPrompt()   - 加载记忆指令                 │  │
│  │  truncateEntrypointContent() - MEMORY.md 截断        │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                 智能检索层 (findRelevantMemories.ts)         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  scanMemoryFiles()     - 扫描所有 .md 文件           │  │
│  │  selectRelevantMemories() - AI 选择相关记忆          │  │
│  │  formatMemoryManifest() - 生成记忆清单               │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                 文件系统层 (memoryScan.ts)                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  readFileInRange()  - 读取 frontmatter (前 30 行)   │  │
│  │  parseFrontmatter() - 解析元数据                     │  │
│  │  按 mtime 降序排序 - 最新优先                        │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                 存储层 (文件系统)                            │
│  ~/.claude/projects/<project-hash>/memory/                  │
│  ├─ MEMORY.md              - 索引文件 (200行/25KB限制)     │
│  ├─ user_role.md           - 用户信息                       │
│  ├─ feedback_testing.md    - 反馈记录                       │
│  ├─ project_deadline.md    - 项目上下文                     │
│  └─ reference_linear.md    - 外部资源引用                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 索引结构设计

### 1. 二级索引架构

```
MEMORY.md (一级索引)             topic_*.md (二级内容)
┌────────────────────────┐       ┌────────────────────────┐
│ # Memory Index         │       │ ---                    │
│                        │       │ name: user-role        │
│ ## User Memories       │       │ description: ...       │
│ - [User Role](...) ──┼───→   │ type: user             │
│                        │       │ ---                    │
│ ## Feedback            │       │                        │
│ - [Testing](...) ─────┼──┐    │ User is a senior...    │
│                        │  │    └────────────────────────┘
│ ## Project Context     │  │
│ - [Deadline](...) ────┼──┼─┐
│                        │  │ │  ┌────────────────────────┐
│ ## References          │  │ └→│ feedback_testing.md    │
│ - [Linear](...) ──────┼──┼───→│ ---                    │
│                        │  │    │ name: testing-policy   │
└────────────────────────┘  │    │ type: feedback         │
                            │    │ ---                    │
                            │    │                        │
                            │    │ Integration tests...   │
                            │    └────────────────────────┘
                            │
                            └───→ ...
```

**设计理念**:
- **MEMORY.md** - 轻量级索引,仅存储元数据 + 一行摘要
- **topic_*.md** - 完整内容,包含 frontmatter 和正文
- **延迟加载** - 仅在需要时 Read 具体 topic 文件

---

### 2. Frontmatter 元数据格式

**标准格式** (`src/memdir/memoryTypes.ts`):

```yaml
---
name: user-role
description: Senior backend engineer with Go expertise
metadata:
  type: user
---

User is a senior backend engineer with 10 years of Go experience.
Currently focused on microservices refactoring.
Prefers verbose error handling over panic().

**Why:** Has deep understanding of production systems.
**How to apply:** Frame technical explanations in terms of distributed systems.
```

**关键字段**:

| 字段 | 类型 | 用途 |
|------|------|------|
| `name` | string | 文件标识符 (kebab-case) |
| `description` | string | 一句话摘要 (用于检索) |
| `metadata.type` | enum | `user`/`feedback`/`project`/`reference` |

---

### 3. 四种记忆类型

**类型定义** (`src/memdir/memoryTypes.ts:14-19`):

```typescript
export const MEMORY_TYPES = [
  'user',      // 用户角色、偏好、技能
  'feedback',  // 行为纠正与确认
  'project',   // 项目上下文 (截止日期、决策)
  'reference', // 外部资源指针 (Linear/Grafana)
] as const;

export type MemoryType = (typeof MEMORY_TYPES)[number];
```

**排除规则** (`WHAT_NOT_TO_SAVE_SECTION`):
- ❌ 代码模式、架构 (可通过 Grep/LSP 推导)
- ❌ Git 历史、文件结构 (可通过 Git 工具查询)
- ❌ 调试方案、修复步骤 (应该在代码或 Commit Message 中)
- ❌ 已文档化的内容 (CLAUDE.md 已包含)
- ❌ 临时任务状态 (应使用 Plan/Todo)

---

## 核心算法

### 1. 索引截断算法

**目标**: 防止 MEMORY.md 过大导致上下文溢出

**实现** (`src/memdir/memdir.ts:57-103`):

```typescript
export const MAX_ENTRYPOINT_LINES = 200;
export const MAX_ENTRYPOINT_BYTES = 25_000; // ~25KB

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const trimmed = raw.trim();
  const contentLines = trimmed.split('\n');
  const lineCount = contentLines.length;
  const byteCount = trimmed.length;

  // 检测是否超限
  const wasLineTruncated = lineCount > MAX_ENTRYPOINT_LINES;
  const wasByteTruncated = byteCount > MAX_ENTRYPOINT_BYTES;

  if (!wasLineTruncated && !wasByteTruncated) {
    return { content: trimmed, lineCount, byteCount, ... };
  }

  // 第一步: 行数截断 (保留自然边界)
  let truncated = wasLineTruncated
    ? contentLines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    : trimmed;

  // 第二步: 字节截断 (在最后一个换行符处截断,不切断行)
  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES);
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES);
  }

  // 附加警告消息
  const reason = 
    wasByteTruncated && !wasLineTruncated
      ? `${formatFileSize(byteCount)} (limit: 25KB) — index entries are too long`
      : wasLineTruncated && !wasByteTruncated
        ? `${lineCount} lines (limit: 200)`
        : `${lineCount} lines and ${formatFileSize(byteCount)}`;

  return {
    content: truncated + 
      `\n\n> WARNING: MEMORY.md is ${reason}. Only part of it was loaded. ` +
      `Keep index entries to one line under ~200 chars; move detail into topic files.`,
    lineCount,
    byteCount,
    wasLineTruncated,
    wasByteTruncated,
  };
}
```

**算法特点**:
1. **双重限制**: 行数 (200) + 字节数 (25KB)
2. **优雅降级**: 先截断行,再截断字节 (保持行完整性)
3. **智能边界**: 字节截断时在最后一个 `\n` 处切断,不会切断单词
4. **用户提示**: 附加 WARNING 消息,说明截断原因

---

### 2. 记忆扫描算法

**目标**: 快速扫描所有 .md 文件,读取 frontmatter

**实现** (`src/memdir/memoryScan.ts:35-77`):

```typescript
const MAX_MEMORY_FILES = 200;
const FRONTMATTER_MAX_LINES = 30;

export async function scanMemoryFiles(
  memoryDir: string,
  signal: AbortSignal
): Promise<MemoryHeader[]> {
  // 1. 递归列出所有 .md 文件
  const entries = await readdir(memoryDir, { recursive: true });
  const mdFiles = entries.filter(
    f => f.endsWith('.md') && basename(f) !== 'MEMORY.md'
  );

  // 2. 并行读取所有文件的前 30 行 (frontmatter 区域)
  const headerResults = await Promise.allSettled(
    mdFiles.map(async (relativePath): Promise<MemoryHeader> => {
      const filePath = join(memoryDir, relativePath);
      
      // readFileInRange 返回 { content, mtimeMs }
      const { content, mtimeMs } = await readFileInRange(
        filePath,
        0,
        FRONTMATTER_MAX_LINES,
        undefined,
        signal
      );
      
      // 解析 YAML frontmatter
      const { frontmatter } = parseFrontmatter(content, filePath);
      
      return {
        filename: relativePath,
        filePath,
        mtimeMs,
        description: frontmatter.description || null,
        type: parseMemoryType(frontmatter.type),
      };
    })
  );

  // 3. 过滤失败的读取 (权限错误、文件删除等)
  // 4. 按 mtime 降序排序 (最新优先)
  // 5. 截断到前 200 个文件
  return headerResults
    .filter((r): r is PromiseFulfilledResult<MemoryHeader> => 
      r.status === 'fulfilled'
    )
    .map(r => r.value)
    .sort((a, b) => b.mtimeMs - a.mtimeMs)
    .slice(0, MAX_MEMORY_FILES);
}
```

**性能优化**:
- ✅ **单次遍历**: `readFileInRange` 内部 `stat()`,避免两次系统调用
- ✅ **并行 I/O**: `Promise.allSettled` 并行读取所有文件
- ✅ **部分读取**: 只读前 30 行,不读取完整文件
- ✅ **容错处理**: `allSettled` 允许部分文件读取失败

---

### 3. 记忆清单格式化

**目标**: 生成供 AI 选择器使用的简洁清单

**实现** (`src/memdir/memoryScan.ts:84-94`):

```typescript
export function formatMemoryManifest(memories: MemoryHeader[]): string {
  return memories
    .map(m => {
      const tag = m.type ? `[${m.type}] ` : '';
      const ts = new Date(m.mtimeMs).toISOString();
      return m.description
        ? `- ${tag}${m.filename} (${ts}): ${m.description}`
        : `- ${tag}${m.filename} (${ts})`;
    })
    .join('\n');
}
```

**输出示例**:

```
- [user] user_role.md (2026-05-28T10:23:45.678Z): Senior backend engineer with Go expertise
- [feedback] feedback_testing.md (2026-05-27T15:42:18.123Z): Integration tests must hit real DB
- [project] project_deadline.md (2026-05-26T09:10:32.456Z): Feature freeze on 2026-06-01
- [reference] reference_linear.md (2026-05-25T14:56:09.789Z): Pipeline bugs tracked in Linear INGEST
```

---

## 智能检索机制

### 1. 检索流程

```
用户查询: "Fix the login bug"
    ↓
┌───────────────────────────────────────┐
│ 1. scanMemoryFiles()                  │
│    扫描所有记忆文件 → 200 个候选      │
└───────────────────┬───────────────────┘
                    ↓
┌───────────────────────────────────────┐
│ 2. formatMemoryManifest()             │
│    生成清单 (filename + description)  │
└───────────────────┬───────────────────┘
                    ↓
┌───────────────────────────────────────┐
│ 3. selectRelevantMemories()           │
│    调用 Sonnet 4 API                  │
│    System Prompt: "选择最相关的记忆"  │
│    Input: 查询 + 记忆清单              │
│    Output: JSON { selected: [...] }  │
└───────────────────┬───────────────────┘
                    ↓
┌───────────────────────────────────────┐
│ 4. 返回 ≤ 5 个最相关的文件路径        │
│    [feedback_auth.md, user_role.md]   │
└───────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────┐
│ 5. 主 AI 通过 Read 工具读取内容       │
│    结合上下文处理查询                  │
└───────────────────────────────────────┘
```

---

### 2. AI 选择器实现

**System Prompt** (`src/memdir/findRelevantMemories.ts:18-24`):

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query. You will be given the user's query and a list of available memory files with their filenames and descriptions.

Return a list of filenames for the memories that will clearly be useful to Claude Code as it processes the user's query (up to 5). Only include memories that you are certain will be helpful based on their name and description.
- If you are unsure if a memory will be useful in processing the user's query, then do not include it in your list. Be selective and discerning.
- If there are no memories in the list that would clearly be useful, feel free to return an empty list.
- If a list of recently-used tools is provided, do not select memories that are usage reference or API documentation for those tools (Claude Code is already exercising them). DO still select memories containing warnings, gotchas, or known issues about those tools — active use is exactly when those matter.
`;
```

**调用逻辑** (`src/memdir/findRelevantMemories.ts:39-75`):

```typescript
export async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]> {
  // 1. 扫描所有记忆文件 (排除已展示的)
  const memories = (await scanMemoryFiles(memoryDir, signal)).filter(
    m => !alreadySurfaced.has(m.filePath)
  );
  
  if (memories.length === 0) return [];

  // 2. AI 选择相关记忆
  const selectedFilenames = await selectRelevantMemories(
    query,
    memories,
    signal,
    recentTools
  );

  // 3. 映射回 MemoryHeader 对象
  const byFilename = new Map(memories.map(m => [m.filename, m]));
  const selected = selectedFilenames
    .map(filename => byFilename.get(filename))
    .filter((m): m is MemoryHeader => m !== undefined);

  // 4. 返回路径 + mtime (用于陈旧警告)
  return selected.map(m => ({ path: m.filePath, mtimeMs: m.mtimeMs }));
}
```

**API 调用** (`src/memdir/findRelevantMemories.ts:82-141`):

```typescript
async function selectRelevantMemories(
  query: string,
  memories: MemoryHeader[],
  signal: AbortSignal,
  recentTools: readonly string[]
): Promise<string[]> {
  const manifest = formatMemoryManifest(memories);
  
  const toolsSection = recentTools.length > 0
    ? `\n\nRecently used tools: ${recentTools.join(', ')}`
    : '';

  const result = await sideQuery({
    model: getDefaultSonnetModel(),        // Sonnet 4.5
    system: SELECT_MEMORIES_SYSTEM_PROMPT,
    skipSystemPromptPrefix: true,
    messages: [{
      role: 'user',
      content: `Query: ${query}\n\nAvailable memories:\n${manifest}${toolsSection}`,
    }],
    max_tokens: 256,
    output_format: {                       // Structured Output
      type: 'json_schema',
      schema: {
        type: 'object',
        properties: {
          selected_memories: { type: 'array', items: { type: 'string' } }
        },
        required: ['selected_memories'],
      },
    },
    signal,
    querySource: 'memdir_relevance',
  });

  // 解析 JSON 响应
  const textBlock = result.content.find(b => b.type === 'text');
  const parsed: { selected_memories: string[] } = jsonParse(textBlock.text);
  
  // 验证文件名有效性
  const validFilenames = new Set(memories.map(m => m.filename));
  return parsed.selected_memories.filter(f => validFilenames.has(f));
}
```

---

### 3. 工具噪声过滤

**问题**: 当 AI 正在使用某个工具时,工具的参考文档会被误匹配

**示例**:
```
用户查询: "Use the mcp__filesystem__spawn tool"
记忆清单: 
  - mcp_usage.md: "How to use mcp__filesystem__spawn"
  
误匹配: AI 会选择这个记忆,但实际上对话中已经在使用这个工具了
```

**解决方案** (`src/memdir/findRelevantMemories.ts:92-95`):

```typescript
// System Prompt 中的特殊指令:
"If a list of recently-used tools is provided, do not select memories that are 
usage reference or API documentation for those tools (Claude Code is already 
exercising them). DO still select memories containing warnings, gotchas, or 
known issues about those tools — active use is exactly when those matter."

// 传入最近使用的工具列表
const toolsSection = recentTools.length > 0
  ? `\n\nRecently used tools: ${recentTools.join(', ')}`
  : '';
```

**区分**:
- ❌ **不选**: 工具使用文档 (已在使用,不需要参考)
- ✅ **选择**: 工具警告/陷阱 (正在使用时最需要)

---

## 性能优化

### 1. 路径缓存

**实现** (`src/memdir/paths.ts:223-232`):

```typescript
export const getAutoMemPath = memoize(
  (): string => {
    const override = getAutoMemPathOverride() ?? getAutoMemPathSetting();
    if (override) return override;
    const projectsDir = join(getMemoryBaseDir(), 'projects');
    return join(projectsDir, sanitizePath(getAutoMemBase()), AUTO_MEM_DIRNAME) + sep;
  },
  () => getProjectRoot(),  // 缓存 key = 项目根目录
);
```

**优势**:
- 每个项目只计算一次路径
- 基于 `lodash-es/memoize` 实现
- 缓存 key 使用项目根目录 (跨 CWD 切换仍有效)

---

### 2. 文件扫描优化

**策略**:

| 优化点 | 实现 | 效果 |
|-------|------|------|
| **并行读取** | `Promise.allSettled` | N 个文件并行读取,不阻塞 |
| **部分读取** | `readFileInRange(0, 30)` | 只读 frontmatter,节省 90% I/O |
| **单次 stat** | `readFileInRange` 返回 `mtimeMs` | 避免两次系统调用 |
| **容错处理** | `allSettled` 允许部分失败 | 一个文件损坏不影响其他 |
| **数量限制** | 截断到 200 个文件 | 防止海量文件拖慢扫描 |

**时间复杂度**:
```
N 个文件
├─ readdir(recursive): O(N)
├─ filter: O(N)
├─ 并行读取 frontmatter: O(1) (并行)
├─ sort by mtime: O(N log N)
└─ slice(0, 200): O(1)

总计: O(N log N) + 并行 I/O
```

---

### 3. 索引大小限制

**限制策略**:

| 限制 | 值 | 原因 |
|-----|-----|------|
| **MEMORY.md 行数** | 200 行 | ~25KB (p97 实际大小) |
| **MEMORY.md 字节数** | 25,000 bytes | 防止超长行绕过行数限制 |
| **扫描文件数** | 200 个 | 平衡性能与覆盖率 |
| **Frontmatter 读取** | 30 行 | 足够包含完整 YAML 头 |
| **AI 选择结果** | ≤ 5 个 | 控制上下文膨胀 |

---

### 4. 增量更新机制

**问题**: 每次扫描都读取所有文件很浪费

**未来优化方向** (当前未实现):

```typescript
// 缓存上次扫描结果
type ScanCache = {
  lastScanTime: number;
  files: Map<string, { mtimeMs: number; header: MemoryHeader }>;
};

// 增量扫描伪代码
async function incrementalScan(cache: ScanCache): Promise<MemoryHeader[]> {
  const allFiles = await readdir(memoryDir);
  const changed: MemoryHeader[] = [];
  
  for (const file of allFiles) {
    const cached = cache.files.get(file);
    const currentMtime = (await stat(file)).mtimeMs;
    
    if (!cached || cached.mtimeMs < currentMtime) {
      // 文件新增或修改,重新读取
      const header = await readHeader(file);
      changed.push(header);
    } else {
      // 未变化,复用缓存
      changed.push(cached.header);
    }
  }
  
  return changed.sort((a, b) => b.mtimeMs - a.mtimeMs).slice(0, 200);
}
```

---

## 数据流分析

### 完整生命周期

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 启动阶段                                                  │
│    loadMemoryPrompt() → buildMemoryLines()                  │
│    注入系统提示词 (包含记忆使用指南)                        │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ 2. 用户交互                                                  │
│    用户: "I prefer bun over npm"                            │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ 3. AI 写入记忆                                               │
│    Write({                                                  │
│      file_path: ".claude/memory/feedback_package_mgr.md",  │
│      content: "---\nname: package-manager\n..."            │
│    })                                                       │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ 4. 更新索引                                                  │
│    Edit({                                                   │
│      file_path: ".claude/memory/MEMORY.md",                │
│      old_string: "## Feedback\n",                          │
│      new_string: "## Feedback\n- [Package Mgr](...)\n"    │
│    })                                                       │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ 5. 后续会话                                                  │
│    用户: "Install a new package"                            │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ 6. 记忆检索 (可选)                                           │
│    findRelevantMemories("Install a new package")           │
│    → AI 选择: [feedback_package_mgr.md]                    │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ 7. 读取记忆                                                  │
│    Read(".claude/memory/feedback_package_mgr.md")          │
│    → AI 获知: "User prefers bun"                           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ 8. 执行命令                                                  │
│    Bash("bun install lodash")  ← 遵循记忆中的偏好           │
└─────────────────────────────────────────────────────────────┘
```

---

### 索引更新流程

```
AI 决定写入新记忆
    ↓
┌─────────────────────────────────────┐
│ Step 1: 写入 topic 文件              │
│ Write(feedback_testing.md)          │
│ 包含 frontmatter + 正文             │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ Step 2: 读取 MEMORY.md              │
│ Read(MEMORY.md)                     │
│ 获取当前索引内容                     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ Step 3: 编辑索引                     │
│ Edit({                              │
│   old_string: "## Feedback\n",     │
│   new_string: "## Feedback\n" +    │
│     "- [Testing Policy](...)\n"    │
│ })                                  │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ Step 4: 验证写入                     │
│ truncateEntrypointContent()        │
│ 检查是否超过 200行/25KB             │
└─────────────────────────────────────┘
```

---

## 技术亮点

### 1. 双层索引设计

**传统方案** (单文件):
```
all_memories.json (10MB)
├─ user_memories: [...]
├─ feedback_memories: [...]
└─ project_memories: [...]

问题:
❌ 全量加载浪费上下文
❌ 难以增量更新
❌ JSON 解析慢
```

**Claude Code 方案** (二级索引):
```
MEMORY.md (25KB)         topic_*.md (按需加载)
├─ 元数据索引            ├─ 完整内容
├─ 200 行限制            ├─ 结构化 frontmatter
└─ 延迟加载指针          └─ Why/How to apply

优势:
✅ 轻量级索引常驻上下文
✅ 按需加载详细内容
✅ 易于 AI 编辑 (Markdown)
```

---

### 2. AI 驱动的相关性检索

**传统方案** (关键词匹配):
```typescript
function findRelevant(query: string, memories: Memory[]): Memory[] {
  return memories.filter(m => 
    m.description.includes(query) || 
    m.tags.some(t => query.includes(t))
  );
}

问题:
❌ 无法理解语义
❌ 同义词匹配失败
❌ 假阳性高 (关键词重叠)
```

**Claude Code 方案** (AI 选择器):
```typescript
async function findRelevant(query: string, memories: Memory[]): Promise<Memory[]> {
  const prompt = `Query: ${query}\n\nMemories:\n${formatManifest(memories)}`;
  const result = await sonnet.query(prompt, { schema: SelectionSchema });
  return result.selected_memories; // ≤ 5 个
}

优势:
✅ 理解语义相关性
✅ 处理同义词和隐含关系
✅ 低假阳性 (AI 判断精确)
✅ 支持负样本过滤 (Recently used tools)
```

---

### 3. 结构化记忆格式

**Why/How 结构**:

```markdown
Integration tests must hit a real database, not mocks.

**Why:** Prior incident where mock/prod divergence masked a broken migration.
**How to apply:** When writing tests for schema changes or data migrations.
```

**优势**:
- ✅ **Why** - 提供上下文,帮助 AI 理解规则背后的原因
- ✅ **How to apply** - 明确适用场景,避免过度泛化
- ✅ **边界判断** - AI 可以判断新场景是否适用规则

**对比无结构记忆**:
```
"Don't mock the database"

问题:
❌ 为什么不 mock? (没有上下文)
❌ 适用于所有测试吗? (边界不清)
❌ 有例外情况吗? (无法判断)
```

---

### 4. 容错与降级

**文件系统错误处理**:

```typescript
// scanMemoryFiles 使用 Promise.allSettled
const headerResults = await Promise.allSettled(
  mdFiles.map(async (file) => await readHeader(file))
);

// 过滤失败的读取,继续处理成功的
return headerResults
  .filter((r): r is PromiseFulfilledResult<MemoryHeader> => 
    r.status === 'fulfilled'
  )
  .map(r => r.value);
```

**优势**:
- ✅ 一个文件损坏不影响其他文件
- ✅ 权限错误静默跳过
- ✅ 部分结果优于零结果

**AI 选择器错误处理**:

```typescript
try {
  const result = await sideQuery({ ... });
  return parseSelection(result);
} catch (e) {
  if (signal.aborted) return [];  // 用户取消
  logForDebugging(`selectRelevantMemories failed: ${e}`, { level: 'warn' });
  return [];  // 降级: 不检索记忆,继续主流程
}
```

---

### 5. 陈旧性检测

**实现** (`src/memdir/memoryAge.ts`):

```typescript
export function computeMemoryAge(mtimeMs: number): {
  ageInDays: number;
  isStale: boolean;
} {
  const now = Date.now();
  const ageMs = now - mtimeMs;
  const ageInDays = Math.floor(ageMs / (24 * 60 * 60 * 1000));
  
  // 90 天以上视为陈旧
  const isStale = ageInDays > 90;
  
  return { ageInDays, isStale };
}

// AI 引用陈旧记忆时附加警告
if (isStale) {
  context += `\n\n> This memory is ${ageInDays} days old. Verify it is still accurate before acting on it.`;
}
```

---

## 核心常量速查

```typescript
// 索引限制
MAX_ENTRYPOINT_LINES = 200        // MEMORY.md 最大行数
MAX_ENTRYPOINT_BYTES = 25_000     // MEMORY.md 最大字节数 (~25KB)
MAX_MEMORY_FILES = 200            // 最多扫描 200 个文件
FRONTMATTER_MAX_LINES = 30        // 读取前 30 行 (frontmatter)

// 记忆类型
MEMORY_TYPES = ['user', 'feedback', 'project', 'reference']

// 文件名
ENTRYPOINT_NAME = 'MEMORY.md'     // 索引文件名
AUTO_MEM_DIRNAME = 'memory'       // 默认目录名

// 检索限制
MAX_SELECTED_MEMORIES = 5         // AI 选择器最多返回 5 个
STALE_THRESHOLD_DAYS = 90         // 陈旧阈值 90 天

// 路径
~/.claude/projects/<hash>/memory/  // 默认存储位置
```

---

## 总结

### 设计哲学

1. **分层索引** - MEMORY.md 轻量级索引 + topic_*.md 延迟加载
2. **AI 驱动** - 使用 Sonnet 4 进行语义检索,不依赖关键词
3. **容错优先** - 部分文件损坏不影响整体功能
4. **结构化记忆** - Why/How 格式提供上下文和边界
5. **性能优化** - 并行 I/O、部分读取、缓存路径

### 核心优势

| 优势 | 实现 |
|------|------|
| **可扩展** | 支持数百个记忆文件,截断机制防止溢出 |
| **精确检索** | AI 语义理解,低假阳性率 |
| **易于编辑** | Markdown + frontmatter,AI 可直接操作 |
| **容错性强** | 部分失败降级,不阻断主流程 |
| **跨会话** | 持久化存储,支持长期记忆积累 |

### 代码统计

```
总计: 1,770 行 TypeScript
├─ memdir.ts            507 行  - 核心索引管理
├─ paths.ts             278 行  - 路径解析
├─ teamMemPaths.ts      292 行  - 团队记忆路径
├─ memoryTypes.ts       271 行  - 类型定义
├─ findRelevantMemories 141 行  - 智能检索
├─ teamMemPrompts.ts    100 行  - 团队提示词
├─ memoryScan.ts         94 行  - 文件扫描
├─ memoryAge.ts          53 行  - 陈旧性检测
└─ memoryShapeTelemetry  34 行  - 遥测
```

---

**文档完整度**: ✅ 记忆系统索引算法已全面剖析  
**更新建议**: 监控 `MAX_ENTRYPOINT_BYTES` 是否需要调整 (基于 p99 统计)
