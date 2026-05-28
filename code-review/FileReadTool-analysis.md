# FileReadTool 工具详解

> 分析日期: 2026-05-27
> 文件位置: `src/tools/FileReadTool/`
> 核心功能: 从本地文件系统读取各种类型的文件

---

## 📋 目录

- [概述](#概述)
- [文件结构](#文件结构)
- [核心功能](#核心功能)
- [输入输出定义](#输入输出定义)
- [执行流程](#执行流程)
- [文件类型支持](#文件类型支持)
- [安全机制](#安全机制)
- [性能优化](#性能优化)
- [关键技术细节](#关键技术细节)
- [使用示例](#使用示例)

---

## 概述

**FileReadTool** 是 Claude Code 中最核心的工具之一，负责读取文件系统中的各种文件。它不仅仅是简单的文件读取，而是一个功能强大的文件处理系统，支持：

- ✅ 文本文件 (支持行号、分页、部分读取)
- ✅ 图片文件 (PNG/JPG/GIF/WEBP + 自动压缩)
- ✅ PDF 文档 (完整读取 + 分页提取)
- ✅ Jupyter Notebook (.ipynb)
- ✅ 自动去重 (避免重复读取相同内容)
- ✅ 权限检查 (基于配置的访问控制)
- ✅ Token 限制 (防止超出 API 限额)

---

## 文件结构

```
src/tools/FileReadTool/
├── FileReadTool.ts        # 核心工具实现 (1184 行)
├── UI.tsx                 # 终端 UI 渲染组件
├── prompt.ts              # 工具描述 prompt 模板
├── limits.ts              # 读取限制配置
└── imageProcessor.ts      # 图片处理器封装
```

---

## 核心功能

### 1. 工具定义

```typescript
export const FileReadTool = buildTool({
  name: 'Read',
  searchHint: 'read files, images, PDFs, notebooks',
  strict: true,
  maxResultSizeChars: Infinity,  // 由 maxTokens 限制

  // 核心方法
  description(),    // 工具描述
  prompt(),         // Prompt 指令
  validateInput(),  // 输入验证
  checkPermissions(), // 权限检查
  call(),           // 执行读取

  // UI 渲染
  renderToolUseMessage(),
  renderToolResultMessage(),
  renderToolUseErrorMessage(),
});
```

### 2. 主要特性

| 特性 | 说明 |
|------|------|
| **并发安全** | `isConcurrencySafe() = true` - 可并行调用 |
| **只读操作** | `isReadOnly() = true` - 不修改文件系统 |
| **搜索工具** | `isSearchOrReadCommand() = { isRead: true }` |
| **自动推断** | 自动从路径推断文件类型 |
| **错误恢复** | 友好的错误提示 + 文件建议 |

---

## 输入输出定义

### 输入 Schema (Input)

```typescript
{
  file_path: string,        // 必需: 文件绝对路径
  offset?: number,          // 可选: 起始行号 (默认 1)
  limit?: number,           // 可选: 读取行数
  pages?: string            // 可选: PDF 页码范围 (如 "1-5")
}
```

**参数说明**:

| 参数 | 类型 | 必需 | 说明 | 示例 |
|------|------|------|------|------|
| `file_path` | string | ✅ | 文件绝对路径 | `/home/user/file.txt` |
| `offset` | number | ❌ | 起始行号 (1-based) | `10` |
| `limit` | number | ❌ | 读取行数 | `100` |
| `pages` | string | ❌ | PDF 页码 (仅 PDF) | `"1-5"`, `"3"`, `"10-20"` |

### 输出 Schema (Output)

输出是一个 **discriminated union**，根据文件类型返回不同结构：

#### 1. 文本文件 (type: 'text')

```typescript
{
  type: 'text',
  file: {
    filePath: string,      // 文件路径
    content: string,       // 文件内容 (带行号)
    numLines: number,      // 返回的行数
    startLine: number,     // 起始行号
    totalLines: number     // 文件总行数
  }
}
```

#### 2. 图片文件 (type: 'image')

```typescript
{
  type: 'image',
  file: {
    base64: string,                 // Base64 编码的图片数据
    type: 'image/jpeg' | 'image/png' | 'image/gif' | 'image/webp',
    originalSize: number,           // 原始文件大小 (bytes)
    dimensions?: {                  // 尺寸信息
      originalWidth?: number,       // 原始宽度
      originalHeight?: number,      // 原始高度
      displayWidth?: number,        // 显示宽度 (压缩后)
      displayHeight?: number        // 显示高度 (压缩后)
    }
  }
}
```

#### 3. PDF 文件 (type: 'pdf')

```typescript
{
  type: 'pdf',
  file: {
    filePath: string,      // 文件路径
    base64: string,        // Base64 编码的 PDF 数据
    originalSize: number   // 原始文件大小
  }
}
```

#### 4. PDF 分页提取 (type: 'parts')

```typescript
{
  type: 'parts',
  file: {
    filePath: string,      // 文件路径
    originalSize: number,  // 原始文件大小
    count: number,         // 提取的页数
    outputDir: string      // 提取的图片目录
  }
}
```

#### 5. Jupyter Notebook (type: 'notebook')

```typescript
{
  type: 'notebook',
  file: {
    filePath: string,      // 文件路径
    cells: Array<any>      // Notebook 单元格数组
  }
}
```

#### 6. 文件未改变 (type: 'file_unchanged')

```typescript
{
  type: 'file_unchanged',
  file: {
    filePath: string       // 文件路径
  }
}
```

---

## 执行流程

### 整体流程图

```
用户调用 FileReadTool
  ↓
1. validateInput() - 输入验证
  ├─ 验证 pages 参数格式
  ├─ 检查路径是否在拒绝列表
  ├─ 检查是否为 UNC 路径
  ├─ 检查是否为二进制文件
  └─ 检查是否为阻塞设备文件
  ↓
2. backfillObservableInput() - 路径展开
  └─ 展开 ~ 和相对路径
  ↓
3. checkPermissions() - 权限检查
  └─ 检查用户权限配置
  ↓
4. call() - 执行读取
  ├─ 检查去重缓存 (文件未修改?)
  ├─ 发现技能目录 (Skills)
  ├─ 根据文件类型调用 callInner()
  │   ├─ Notebook → readNotebook()
  │   ├─ Image → readImageWithTokenBudget()
  │   ├─ PDF → readPDF() / extractPDFPages()
  │   └─ Text → readFileInRange()
  └─ 返回结果
  ↓
5. mapToolResultToToolResultBlockParam() - 转换为 API 格式
  └─ 根据输出类型格式化结果
  ↓
返回给 Claude API
```

### 详细步骤

#### 步骤 1: 输入验证 (validateInput)

```typescript
async validateInput({ file_path, pages }, context) {
  // 1. 验证 PDF pages 参数
  if (pages !== undefined) {
    const parsed = parsePDFPageRange(pages);
    if (!parsed) {
      return { result: false, message: '无效的页码格式', errorCode: 7 };
    }
    if (rangeSize > PDF_MAX_PAGES_PER_READ) {
      return { result: false, message: '页码范围超过限制', errorCode: 8 };
    }
  }

  // 2. 展开路径
  const fullFilePath = expandPath(file_path);

  // 3. 检查拒绝规则
  const denyRule = matchingRuleForInput(fullFilePath, context, 'read', 'deny');
  if (denyRule !== null) {
    return { result: false, message: '文件在拒绝目录中', errorCode: 1 };
  }

  // 4. UNC 路径检查 (安全: 防止 NTLM 凭证泄露)
  if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
    return { result: true }; // 延迟到权限批准后
  }

  // 5. 二进制文件检查
  if (hasBinaryExtension(fullFilePath) && !isPDFExtension(ext) && !IMAGE_EXTENSIONS.has(ext)) {
    return { result: false, message: '无法读取二进制文件', errorCode: 4 };
  }

  // 6. 阻塞设备文件检查
  if (isBlockedDevicePath(fullFilePath)) {
    return { result: false, message: '无法读取设备文件', errorCode: 9 };
  }

  return { result: true };
}
```

**阻塞的设备文件**:

```typescript
const BLOCKED_DEVICE_PATHS = new Set([
  // 无限输出 - 永远不会到达 EOF
  '/dev/zero',
  '/dev/random',
  '/dev/urandom',
  '/dev/full',

  // 阻塞等待输入
  '/dev/stdin',
  '/dev/tty',
  '/dev/console',

  // 无意义的读取
  '/dev/stdout',
  '/dev/stderr',

  // fd 别名
  '/dev/fd/0',
  '/dev/fd/1',
  '/dev/fd/2',
]);
```

#### 步骤 2: 权限检查 (checkPermissions)

```typescript
async checkPermissions(input, context): Promise<PermissionDecision> {
  return checkReadPermissionForTool(
    FileReadTool,
    input,
    context.toolPermissionContext
  );
}
```

- 根据用户配置的权限规则检查
- 支持 allow/deny 规则
- 支持通配符匹配

#### 步骤 3: 执行读取 (call)

```typescript
async call({ file_path, offset = 1, limit, pages }, context) {
  const fullFilePath = expandPath(file_path);

  // 1. 检查去重缓存
  const existingState = readFileState.get(fullFilePath);
  if (existingState && !dedupKillswitch) {
    const rangeMatch = existingState.offset === offset && existingState.limit === limit;
    if (rangeMatch) {
      const mtimeMs = await getFileModificationTimeAsync(fullFilePath);
      if (mtimeMs === existingState.timestamp) {
        // 文件未修改，返回去重标记
        return { data: { type: 'file_unchanged', file: { filePath: file_path } } };
      }
    }
  }

  // 2. 发现技能目录 (异步, 不阻塞)
  const newSkillDirs = await discoverSkillDirsForPaths([fullFilePath], cwd);
  if (newSkillDirs.length > 0) {
    addSkillDirectories(newSkillDirs).catch(() => {});
  }

  // 3. 调用内部读取逻辑
  try {
    return await callInner(/* ... */);
  } catch (error) {
    // 处理文件不存在错误
    if (getErrnoCode(error) === 'ENOENT') {
      // macOS 截图路径特殊处理
      const altPath = getAlternateScreenshotPath(fullFilePath);
      if (altPath) {
        try {
          return await callInner(/* altPath */);
        } catch {}
      }

      // 友好的错误提示
      const similarFilename = findSimilarFile(fullFilePath);
      const cwdSuggestion = await suggestPathUnderCwd(fullFilePath);
      let message = `文件不存在。当前目录: ${getCwd()}`;
      if (cwdSuggestion) {
        message += ` 你是指 ${cwdSuggestion} 吗?`;
      } else if (similarFilename) {
        message += ` 你是指 ${similarFilename} 吗?`;
      }
      throw new Error(message);
    }
    throw error;
  }
}
```

#### 步骤 4: 内部读取逻辑 (callInner)

```typescript
async function callInner(
  file_path: string,
  fullFilePath: string,
  resolvedFilePath: string,
  ext: string,
  offset: number,
  limit: number | undefined,
  pages: string | undefined,
  maxSizeBytes: number,
  maxTokens: number,
  readFileState: Map,
  context: ToolUseContext,
  messageId: string | undefined
) {
  // --- Jupyter Notebook ---
  if (ext === 'ipynb') {
    const cells = await readNotebook(resolvedFilePath);
    const cellsJson = jsonStringify(cells);

    // 检查大小限制
    if (Buffer.byteLength(cellsJson) > maxSizeBytes) {
      throw new Error(`Notebook 内容超过最大大小限制...`);
    }

    // 检查 token 限制
    await validateContentTokens(cellsJson, ext, maxTokens);

    // 保存到缓存
    const stats = await fs.stat(resolvedFilePath);
    readFileState.set(fullFilePath, {
      content: cellsJson,
      timestamp: Math.floor(stats.mtimeMs),
      offset,
      limit,
    });

    return {
      data: {
        type: 'notebook',
        file: { filePath: file_path, cells }
      }
    };
  }

  // --- 图片 ---
  if (IMAGE_EXTENSIONS.has(ext)) {
    const data = await readImageWithTokenBudget(resolvedFilePath, maxTokens);

    const metadataText = data.file.dimensions
      ? createImageMetadataText(data.file.dimensions)
      : null;

    return {
      data,
      ...(metadataText && {
        newMessages: [createUserMessage({ content: metadataText, isMeta: true })]
      })
    };
  }

  // --- PDF ---
  if (isPDFExtension(ext)) {
    // 分页提取
    if (pages) {
      const parsedRange = parsePDFPageRange(pages);
      const extractResult = await extractPDFPages(resolvedFilePath, parsedRange);

      if (!extractResult.success) {
        throw new Error(extractResult.error.message);
      }

      // 读取提取的图片
      const entries = await readdir(extractResult.data.file.outputDir);
      const imageFiles = entries.filter(f => f.endsWith('.jpg')).sort();
      const imageBlocks = await Promise.all(
        imageFiles.map(async f => {
          const imgPath = path.join(extractResult.data.file.outputDir, f);
          const imgBuffer = await readFileAsync(imgPath);
          const resized = await maybeResizeAndDownsampleImageBuffer(imgBuffer, ...);
          return {
            type: 'image',
            source: {
              type: 'base64',
              media_type: `image/${resized.mediaType}`,
              data: resized.buffer.toString('base64')
            }
          };
        })
      );

      return {
        data: extractResult.data,
        newMessages: [createUserMessage({ content: imageBlocks, isMeta: true })]
      };
    }

    // 完整读取
    const pageCount = await getPDFPageCount(resolvedFilePath);
    if (pageCount > PDF_AT_MENTION_INLINE_THRESHOLD) {
      throw new Error(`PDF 有 ${pageCount} 页，太多无法一次读取。请使用 pages 参数...`);
    }

    const readResult = await readPDF(resolvedFilePath);
    if (!readResult.success) {
      throw new Error(readResult.error.message);
    }

    return {
      data: readResult.data,
      newMessages: [
        createUserMessage({
          content: [{
            type: 'document',
            source: {
              type: 'base64',
              media_type: 'application/pdf',
              data: readResult.data.file.base64
            }
          }],
          isMeta: true
        })
      ]
    };
  }

  // --- 文本文件 ---
  const lineOffset = offset === 0 ? 0 : offset - 1;
  const { content, lineCount, totalLines, totalBytes, readBytes, mtimeMs } =
    await readFileInRange(
      resolvedFilePath,
      lineOffset,
      limit,
      limit === undefined ? maxSizeBytes : undefined,
      context.abortController.signal
    );

  // 检查 token 限制
  await validateContentTokens(content, ext, maxTokens);

  // 保存到缓存
  readFileState.set(fullFilePath, {
    content,
    timestamp: Math.floor(mtimeMs),
    offset,
    limit,
  });

  // 通知监听器
  for (const listener of fileReadListeners.slice()) {
    listener(resolvedFilePath, content);
  }

  const data = {
    type: 'text',
    file: {
      filePath: file_path,
      content,
      numLines: lineCount,
      startLine: offset,
      totalLines,
    }
  };

  // 记忆文件的新鲜度标记
  if (isAutoMemFile(fullFilePath)) {
    memoryFileMtimes.set(data, mtimeMs);
  }

  // 记录分析日志
  logEvent('tengu_session_file_read', {
    totalLines,
    readLines: lineCount,
    totalBytes,
    readBytes,
    offset,
    ...(limit !== undefined && { limit }),
  });

  return { data };
}
```

---

## 文件类型支持

### 1. 文本文件

**支持格式**: 所有文本文件 (.txt, .js, .ts, .md, .json, ...)

**特性**:
- ✅ 带行号输出 (cat -n 格式)
- ✅ 部分读取 (offset + limit)
- ✅ 自动编码检测
- ✅ Token 限制检查

**输出示例**:

```
     1→import React from 'react';
     2→
     3→function App() {
     4→  return <div>Hello World</div>;
     5→}
     6→
     7→export default App;
```

**实现**:

```typescript
// 读取文件指定行范围
const { content, lineCount, totalLines, mtimeMs } = await readFileInRange(
  filePath,
  lineOffset,
  limit,
  maxSizeBytes,
  signal
);

// 添加行号
const formattedContent = addLineNumbers({
  content,
  startLine: offset
});
```

---

### 2. 图片文件

**支持格式**: PNG, JPG, JPEG, GIF, WEBP

**特性**:
- ✅ 自动压缩 (基于 token 预算)
- ✅ 尺寸调整 (保持宽高比)
- ✅ 格式转换 (JPEG 压缩)
- ✅ 元数据提取 (宽度/高度)

**压缩策略**:

```typescript
async function readImageWithTokenBudget(
  filePath: string,
  maxTokens: number = 25000,
  maxBytes?: number
): Promise<ImageResult> {
  // 1. 读取文件 (一次性, 限制最大字节数)
  const imageBuffer = await fs.readFileBytes(filePath, maxBytes);
  const originalSize = imageBuffer.length;

  // 2. 检测格式
  const detectedMediaType = detectImageFormatFromBuffer(imageBuffer);
  const detectedFormat = detectedMediaType.split('/')[1] || 'png';

  // 3. 标准压缩
  let result: ImageResult;
  try {
    const resized = await maybeResizeAndDownsampleImageBuffer(
      imageBuffer,
      originalSize,
      detectedFormat
    );
    result = createImageResponse(
      resized.buffer,
      resized.mediaType,
      originalSize,
      resized.dimensions
    );
  } catch (e) {
    if (e instanceof ImageResizeError) throw e;
    result = createImageResponse(imageBuffer, detectedFormat, originalSize);
  }

  // 4. 检查 token 预算
  const estimatedTokens = Math.ceil(result.file.base64.length * 0.125);
  if (estimatedTokens > maxTokens) {
    // 5. 激进压缩
    try {
      const compressed = await compressImageBufferWithTokenLimit(
        imageBuffer,
        maxTokens,
        detectedMediaType
      );
      return {
        type: 'image',
        file: {
          base64: compressed.base64,
          type: compressed.mediaType,
          originalSize
        }
      };
    } catch (e) {
      // 6. 降级: 极度压缩 (400x400, quality 20)
      const fallbackBuffer = await sharp(imageBuffer)
        .resize(400, 400, { fit: 'inside', withoutEnlargement: true })
        .jpeg({ quality: 20 })
        .toBuffer();

      return createImageResponse(fallbackBuffer, 'jpeg', originalSize);
    }
  }

  return result;
}
```

**图片处理器**:

使用 `sharp` 库进行图片处理 (或原生 `image-processor-napi` 在打包模式下):

```typescript
// imageProcessor.ts
export async function getImageProcessor(): Promise<SharpFunction> {
  if (isInBundledMode()) {
    // 优先使用原生模块
    try {
      const imageProcessor = await import('image-processor-napi');
      return imageProcessor.sharp || imageProcessor.default;
    } catch {
      console.warn('原生图片处理器不可用，回退到 sharp');
    }
  }

  // 使用 sharp
  const imported = await import('sharp');
  return unwrapDefault(imported);
}
```

---

### 3. PDF 文件

**支持格式**: .pdf

**特性**:
- ✅ 完整 PDF 读取 (Sonnet 3.5 v2+)
- ✅ 分页提取 (转为图片)
- ✅ 页码范围指定 (如 "1-5")
- ✅ 大 PDF 自动分页

**读取模式**:

| 模式 | 条件 | 说明 |
|------|------|------|
| **完整读取** | 模型支持 PDF + 文件小 | 返回完整 PDF 的 base64 |
| **分页提取** | 大 PDF / 旧模型 | 使用 poppler 提取为图片 |
| **指定页面** | 提供 pages 参数 | 只提取指定页面 |

**实现**:

```typescript
// PDF 完整读取
if (isPDFSupported() && fileSize <= PDF_EXTRACT_SIZE_THRESHOLD) {
  const readResult = await readPDF(resolvedFilePath);
  return {
    data: {
      type: 'pdf',
      file: {
        filePath: file_path,
        base64: readResult.data.file.base64,
        originalSize: fileSize
      }
    },
    newMessages: [
      createUserMessage({
        content: [{
          type: 'document',
          source: {
            type: 'base64',
            media_type: 'application/pdf',
            data: readResult.data.file.base64
          }
        }],
        isMeta: true
      })
    ]
  };
}

// PDF 分页提取
const extractResult = await extractPDFPages(resolvedFilePath, parsedRange);
// 提取结果为 outputDir 下的 .jpg 图片文件
const entries = await readdir(extractResult.data.file.outputDir);
const imageFiles = entries.filter(f => f.endsWith('.jpg')).sort();

// 压缩并转换为 image blocks
const imageBlocks = await Promise.all(
  imageFiles.map(async f => {
    const imgPath = path.join(extractResult.data.file.outputDir, f);
    const imgBuffer = await readFileAsync(imgPath);
    const resized = await maybeResizeAndDownsampleImageBuffer(imgBuffer, ...);
    return {
      type: 'image',
      source: {
        type: 'base64',
        media_type: `image/${resized.mediaType}`,
        data: resized.buffer.toString('base64')
      }
    };
  })
);
```

**页码范围解析**:

```typescript
// 支持格式: "3", "1-5", "10-20"
function parsePDFPageRange(pages: string): {
  firstPage: number;
  lastPage: number | Infinity;
} | null {
  const match = pages.match(/^(\d+)(?:-(\d+))?$/);
  if (!match) return null;

  const firstPage = parseInt(match[1], 10);
  const lastPage = match[2] ? parseInt(match[2], 10) : firstPage;

  if (firstPage < 1 || lastPage < firstPage) return null;

  return { firstPage, lastPage };
}
```

---

### 4. Jupyter Notebook

**支持格式**: .ipynb

**特性**:
- ✅ 完整 cells 数组
- ✅ 包含输出 (output)
- ✅ 代码 + Markdown 混合

**输出结构**:

```typescript
{
  type: 'notebook',
  file: {
    filePath: '/path/to/notebook.ipynb',
    cells: [
      {
        cell_type: 'markdown',
        source: ['# Title\n', 'Some text']
      },
      {
        cell_type: 'code',
        source: ['import numpy as np\n', 'print("hello")'],
        outputs: [
          {
            output_type: 'stream',
            text: ['hello\n']
          }
        ]
      }
    ]
  }
}
```

**实现**:

```typescript
// 读取 notebook
const cells = await readNotebook(resolvedFilePath);

// 序列化为 JSON
const cellsJson = jsonStringify(cells);

// 检查大小和 token 限制
const cellsJsonBytes = Buffer.byteLength(cellsJson);
if (cellsJsonBytes > maxSizeBytes) {
  throw new Error(
    `Notebook 内容 (${formatFileSize(cellsJsonBytes)}) 超过最大大小 (${formatFileSize(maxSizeBytes)})。` +
    `使用 Bash 工具和 jq 读取特定部分:\n` +
    `  cat "${file_path}" | jq '.cells[:20]' # 前 20 个 cells\n` +
    `  cat "${file_path}" | jq '.cells[100:120]' # cells 100-120`
  );
}

await validateContentTokens(cellsJson, 'ipynb', maxTokens);

return {
  data: {
    type: 'notebook',
    file: { filePath: file_path, cells }
  }
};
```

---

## 安全机制

### 1. 权限检查

基于用户配置的权限规则:

```typescript
// 权限配置示例 (settings.json)
{
  "toolPermissions": {
    "read": {
      "allow": [
        "/home/user/projects/**",
        "~/.claude/**"
      ],
      "deny": [
        "**/.env",
        "**/.aws/credentials",
        "**/secrets/**"
      ]
    }
  }
}
```

**检查流程**:

```typescript
async function checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState();
  return checkReadPermissionForTool(
    FileReadTool,
    input,
    appState.toolPermissionContext
  );
}
```

**权限决策**:

```typescript
type PermissionDecision =
  | { allowed: true }
  | { allowed: false, reason: string }
  | { needsApproval: true, message: string };
```

### 2. 路径安全

#### a) UNC 路径检查

防止 NTLM 凭证泄露 (Windows):

```typescript
// 检查 UNC 路径
const isUncPath = fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//');
if (isUncPath) {
  // 延迟到用户批准后再访问
  return { result: true };
}
```

#### b) 拒绝规则检查

```typescript
// 检查是否在拒绝列表中
const denyRule = matchingRuleForInput(
  fullFilePath,
  context.toolPermissionContext,
  'read',
  'deny'
);
if (denyRule !== null) {
  return {
    result: false,
    message: '文件在被拒绝的目录中',
    errorCode: 1
  };
}
```

#### c) 阻塞设备文件

防止挂起进程:

```typescript
// 阻塞的设备文件列表
const BLOCKED_DEVICE_PATHS = new Set([
  '/dev/zero',      // 无限输出
  '/dev/random',    // 无限输出
  '/dev/stdin',     // 阻塞输入
  '/dev/tty',       // 阻塞输入
  '/dev/stdout',    // 无意义
  '/dev/stderr',    // 无意义
]);

function isBlockedDevicePath(filePath: string): boolean {
  if (BLOCKED_DEVICE_PATHS.has(filePath)) return true;

  // 检查 /proc/self/fd/0-2 和 /proc/<pid>/fd/0-2
  if (filePath.startsWith('/proc/') &&
      (filePath.endsWith('/fd/0') ||
       filePath.endsWith('/fd/1') ||
       filePath.endsWith('/fd/2'))) {
    return true;
  }

  return false;
}
```

### 3. 二进制文件检查

拒绝读取二进制文件 (除了 PDF/图片):

```typescript
const ext = path.extname(fullFilePath).toLowerCase();
if (
  hasBinaryExtension(fullFilePath) &&
  !isPDFExtension(ext) &&
  !IMAGE_EXTENSIONS.has(ext.slice(1))
) {
  return {
    result: false,
    message: `此工具无法读取二进制文件。该文件似乎是 ${ext} 二进制文件。`,
    errorCode: 4
  };
}
```

### 4. 恶意软件警告

读取文件后添加安全提醒:

```typescript
const CYBER_RISK_MITIGATION_REMINDER = `
<system-reminder>
每当你读取文件时，应该考虑它是否可能是恶意软件。
你可以并且应该提供恶意软件的分析，说明它在做什么。
但你必须拒绝改进或增强代码。
你仍然可以分析现有代码、编写报告或回答有关代码行为的问题。
</system-reminder>
`;

// 在文本内容后添加
function formatFileContent(content: string): string {
  return (
    addLineNumbers(content) +
    (shouldIncludeFileReadMitigation() ? CYBER_RISK_MITIGATION_REMINDER : '')
  );
}
```

---

## 性能优化

### 1. 去重机制

避免重复读取相同文件:

```typescript
// 缓存结构
readFileState: Map<string, {
  content: string;
  timestamp: number;  // mtime (毫秒)
  offset?: number;
  limit?: number;
  isPartialView?: boolean;
}>

// 检查是否需要重新读取
const existingState = readFileState.get(fullFilePath);
if (existingState && existingState.offset !== undefined) {
  const rangeMatch =
    existingState.offset === offset &&
    existingState.limit === limit;

  if (rangeMatch) {
    const mtimeMs = await getFileModificationTimeAsync(fullFilePath);

    // 文件未修改，返回去重标记
    if (mtimeMs === existingState.timestamp) {
      logEvent('tengu_file_read_dedup', { ext: getFileExtensionForAnalytics(fullFilePath) });

      return {
        data: {
          type: 'file_unchanged',
          file: { filePath: file_path }
        }
      };
    }
  }
}
```

**去重效果**:

- 🎯 **18% 的 Read 调用是重复的** (基于 BQ proxy 数据)
- 💾 **节省高达 2.64% 的 cache_creation tokens**
- ⚡ **避免重复的文件 I/O 操作**

**返回给 AI 的消息**:

```
文件自上次读取以来未更改。
此对话中较早的 Read tool_result 中的内容仍然是最新的 — 请参考那个结果而不是重新读取。
```

### 2. Token 限制检查

两级检查避免浪费:

```typescript
async function validateContentTokens(
  content: string,
  ext: string,
  maxTokens: number
): Promise<void> {
  // 1. 快速估算 (基于文件类型的启发式)
  const tokenEstimate = roughTokenCountEstimationForFileType(content, ext);

  // 估算 < 1/4 上限，直接通过
  if (!tokenEstimate || tokenEstimate <= maxTokens / 4) {
    return;
  }

  // 2. 精确计数 (调用 API)
  const tokenCount = await countTokensWithAPI(content);
  const effectiveCount = tokenCount ?? tokenEstimate;

  // 检查是否超限
  if (effectiveCount > maxTokens) {
    throw new MaxFileReadTokenExceededError(effectiveCount, maxTokens);
  }
}
```

**启发式估算示例**:

```typescript
function roughTokenCountEstimationForFileType(content: string, ext: string): number {
  const charCount = content.length;

  switch (ext) {
    case 'js':
    case 'ts':
    case 'jsx':
    case 'tsx':
      // 代码通常压缩比高
      return Math.ceil(charCount / 3.5);

    case 'json':
      // JSON 压缩比低
      return Math.ceil(charCount / 3.0);

    case 'md':
    case 'txt':
      // 文本压缩比中等
      return Math.ceil(charCount / 4.0);

    default:
      // 默认估算
      return Math.ceil(charCount / 3.5);
  }
}
```

### 3. 文件大小限制

两层限制:

| 限制 | 默认值 | 检查时机 | 成本 | 超限行为 |
|------|--------|----------|------|----------|
| **maxSizeBytes** | 256 KB | 读取前 (stat) | 1 次 stat 调用 | 抛出错误 |
| **maxTokens** | 25000 | 读取后 (API) | 1 次 API 调用 | 抛出错误 |

```typescript
// limits.ts
export const DEFAULT_MAX_OUTPUT_TOKENS = 25000;

export type FileReadingLimits = {
  maxTokens: number;           // Token 上限
  maxSizeBytes: number;        // 字节上限 (256 KB)
  includeMaxSizeInPrompt?: boolean;    // 是否在 prompt 中提示大小限制
  targetedRangeNudge?: boolean;        // 是否建议使用 offset/limit
};

export const getDefaultFileReadingLimits = memoize((): FileReadingLimits => {
  // 从 GrowthBook 获取实验配置
  const override = getFeatureValue_CACHED_MAY_BE_STALE<Partial<FileReadingLimits>>('tengu_amber_wren', {});

  // 环境变量覆盖
  const envMaxTokens = process.env.CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS;
  const maxTokens = envMaxTokens
    ? parseInt(envMaxTokens, 10)
    : (override?.maxTokens ?? DEFAULT_MAX_OUTPUT_TOKENS);

  const maxSizeBytes = override?.maxSizeBytes ?? MAX_OUTPUT_SIZE;

  return {
    maxTokens,
    maxSizeBytes,
    includeMaxSizeInPrompt: override?.includeMaxSizeInPrompt,
    targetedRangeNudge: override?.targetedRangeNudge,
  };
});
```

**环境变量覆盖**:

```bash
# 设置自定义 token 上限
export CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS=50000
```

### 4. 部分读取

对于大文件，使用 `offset` 和 `limit` 参数:

```typescript
// 读取前 100 行
{
  file_path: '/path/to/large_file.txt',
  offset: 1,
  limit: 100
}

// 读取第 1000-1100 行
{
  file_path: '/path/to/large_file.txt',
  offset: 1000,
  limit: 100
}
```

**实现**:

```typescript
// readFileInRange() 使用流式读取，避免加载整个文件
async function readFileInRange(
  filePath: string,
  lineOffset: number,
  limit: number | undefined,
  maxBytes: number | undefined,
  signal: AbortSignal
): Promise<{
  content: string;
  lineCount: number;
  totalLines: number;
  totalBytes: number;
  readBytes: number;
  mtimeMs: number;
}> {
  // 使用 readline 逐行读取
  const fileStream = fs.createReadStream(filePath);
  const rl = readline.createInterface({
    input: fileStream,
    crlfDelay: Infinity
  });

  let currentLine = 0;
  let readLines = 0;
  const lines: string[] = [];

  for await (const line of rl) {
    currentLine++;

    // 跳过 offset 之前的行
    if (currentLine <= lineOffset) continue;

    lines.push(line);
    readLines++;

    // 达到 limit，停止读取
    if (limit !== undefined && readLines >= limit) {
      rl.close();
      fileStream.destroy();
      break;
    }
  }

  return {
    content: lines.join('\n'),
    lineCount: readLines,
    totalLines: currentLine,
    totalBytes: (await fs.stat(filePath)).size,
    readBytes: Buffer.byteLength(lines.join('\n')),
    mtimeMs: (await fs.stat(filePath)).mtimeMs
  };
}
```

### 5. 异步技能发现

不阻塞读取操作:

```typescript
// 发现技能目录 (fire-and-forget)
const newSkillDirs = await discoverSkillDirsForPaths([fullFilePath], cwd);
if (newSkillDirs.length > 0) {
  // 异步加载技能，不等待完成
  addSkillDirectories(newSkillDirs).catch(() => {});

  // 激活条件技能
  activateConditionalSkillsForPaths([fullFilePath], cwd);
}
```

---

## 关键技术细节

### 1. macOS 截图路径处理

macOS 截图文件名中的空格字符可能不同:

```typescript
// macOS 截图格式: "Screenshot 2026-05-27 at 10:30:45 AM.png"
// 不同版本 macOS 在 AM/PM 前使用不同的空格字符:
// - 普通空格 (U+0020)
// - 细空格 (U+202F, NARROW NO-BREAK SPACE)

const THIN_SPACE = String.fromCharCode(8239); // U+202F

function getAlternateScreenshotPath(filePath: string): string | undefined {
  const filename = path.basename(filePath);
  const amPmPattern = /^(.+)([ \u202F])(AM|PM)(\.png)$/;
  const match = filename.match(amPmPattern);

  if (!match) return undefined;

  const currentSpace = match[2];
  const alternateSpace = currentSpace === ' ' ? THIN_SPACE : ' ';

  return filePath.replace(
    `${currentSpace}${match[3]}${match[4]}`,
    `${alternateSpace}${match[3]}${match[4]}`
  );
}

// 使用
try {
  return await callInner(filePath, ...);
} catch (error) {
  if (isENOENT(error)) {
    const altPath = getAlternateScreenshotPath(filePath);
    if (altPath) {
      try {
        return await callInner(altPath, ...);
      } catch (altError) {
        // 两种路径都不存在，继续错误处理
      }
    }
  }
  throw error;
}
```

### 2. 文件监听器系统

允许其他服务在文件读取时获得通知:

```typescript
// 监听器类型
type FileReadListener = (filePath: string, content: string) => void;
const fileReadListeners: FileReadListener[] = [];

// 注册监听器
export function registerFileReadListener(
  listener: FileReadListener
): () => void {
  fileReadListeners.push(listener);

  // 返回取消注册函数
  return () => {
    const i = fileReadListeners.indexOf(listener);
    if (i >= 0) fileReadListeners.splice(i, 1);
  };
}

// 触发监听器
for (const listener of fileReadListeners.slice()) {
  listener(resolvedFilePath, content);
}
```

**使用场景**:

- 记忆系统监听文件读取，自动提取知识
- 分析系统收集使用统计
- 插件系统响应文件访问

### 3. 记忆文件新鲜度标记

自动记忆文件会添加时间戳:

```typescript
// 使用 WeakMap 避免污染输出 schema
const memoryFileMtimes = new WeakMap<object, number>();

// 检测是否为自动记忆文件
if (isAutoMemFile(fullFilePath)) {
  memoryFileMtimes.set(data, mtimeMs);
}

// 在序列化时添加新鲜度前缀
function memoryFileFreshnessPrefix(data: object): string {
  const mtimeMs = memoryFileMtimes.get(data);
  if (mtimeMs === undefined) return '';

  return memoryFreshnessNote(mtimeMs); // 如 "(2 天前更新)"
}
```

### 4. 会话文件类型检测

用于分析日志:

```typescript
function detectSessionFileType(
  filePath: string
): 'session_memory' | 'session_transcript' | null {
  const configDir = getClaudeConfigHomeDir(); // ~/.claude

  // 只匹配 Claude 配置目录内的文件
  if (!filePath.startsWith(configDir)) {
    return null;
  }

  const normalizedPath = filePath.split(win32.sep).join(posix.sep);

  // 会话记忆文件: ~/.claude/session-memory/*.md
  if (normalizedPath.includes('/session-memory/') && normalizedPath.endsWith('.md')) {
    return 'session_memory';
  }

  // 会话转录文件: ~/.claude/projects/*/*.jsonl
  if (normalizedPath.includes('/projects/') && normalizedPath.endsWith('.jsonl')) {
    return 'session_transcript';
  }

  return null;
}
```

### 5. 友好的错误提示

文件不存在时建议相似文件:

```typescript
// 查找相似文件名
function findSimilarFile(targetPath: string): string | null {
  const dir = path.dirname(targetPath);
  const targetName = path.basename(targetPath);

  try {
    const files = fs.readdirSync(dir);

    // 使用 Levenshtein 距离找最相似的文件
    let minDistance = Infinity;
    let bestMatch: string | null = null;

    for (const file of files) {
      const distance = levenshteinDistance(targetName, file);
      if (distance < minDistance && distance <= 3) {
        minDistance = distance;
        bestMatch = file;
      }
    }

    return bestMatch ? path.join(dir, bestMatch) : null;
  } catch {
    return null;
  }
}

// 建议在 cwd 下的路径
async function suggestPathUnderCwd(targetPath: string): Promise<string | null> {
  const cwd = getCwd();
  const relativePath = path.relative(cwd, targetPath);

  // 如果目标在 cwd 之外，建议相对路径
  if (!relativePath.startsWith('..')) {
    const relativeExists = await fs.exists(path.join(cwd, path.basename(targetPath)));
    if (relativeExists) {
      return path.join(cwd, path.basename(targetPath));
    }
  }

  return null;
}

// 错误消息
let message = `文件不存在。当前工作目录: ${getCwd()}`;
if (cwdSuggestion) {
  message += ` 你是指 ${cwdSuggestion} 吗?`;
} else if (similarFilename) {
  message += ` 你是指 ${similarFilename} 吗?`;
}
throw new Error(message);
```

---

## UI 渲染

### 终端显示 (UI.tsx)

#### 工具调用消息

```tsx
export function renderToolUseMessage(
  { file_path, offset, limit, pages }: Partial<Input>,
  { verbose }: { verbose: boolean }
): React.ReactNode {
  if (!file_path) return null;

  // Agent 输出文件特殊处理
  if (getAgentOutputTaskId(file_path)) {
    return ''; // 不显示括号
  }

  const displayPath = verbose ? file_path : getDisplayPath(file_path);

  // PDF 分页
  if (pages) {
    return (
      <>
        <FilePathLink filePath={file_path}>{displayPath}</FilePathLink>
        {` · pages ${pages}`}
      </>
    );
  }

  // 部分读取
  if (verbose && (offset || limit)) {
    const startLine = offset ?? 1;
    const lineRange = limit
      ? `lines ${startLine}-${startLine + limit - 1}`
      : `from line ${startLine}`;
    return (
      <>
        <FilePathLink filePath={file_path}>{displayPath}</FilePathLink>
        {` · ${lineRange}`}
      </>
    );
  }

  return <FilePathLink filePath={file_path}>{displayPath}</FilePathLink>;
}
```

**显示效果**:

```
Read src/index.ts
Read large_file.txt · lines 100-200
Read document.pdf · pages 1-5
```

#### 结果消息

```tsx
export function renderToolResultMessage(output: Output): React.ReactNode {
  switch (output.type) {
    case 'image': {
      const { originalSize } = output.file;
      const formattedSize = formatFileSize(originalSize);
      return (
        <MessageResponse height={1}>
          <Text>Read image ({formattedSize})</Text>
        </MessageResponse>
      );
    }

    case 'notebook': {
      const { cells } = output.file;
      return (
        <MessageResponse height={1}>
          <Text>
            Read <Text bold>{cells.length}</Text> cells
          </Text>
        </MessageResponse>
      );
    }

    case 'pdf': {
      const { originalSize } = output.file;
      const formattedSize = formatFileSize(originalSize);
      return (
        <MessageResponse height={1}>
          <Text>Read PDF ({formattedSize})</Text>
        </MessageResponse>
      );
    }

    case 'parts': {
      return (
        <MessageResponse height={1}>
          <Text>
            Read <Text bold>{output.file.count}</Text>{' '}
            {output.file.count === 1 ? 'page' : 'pages'} ({formatFileSize(output.file.originalSize)})
          </Text>
        </MessageResponse>
      );
    }

    case 'text': {
      const { numLines } = output.file;
      return (
        <MessageResponse height={1}>
          <Text>
            Read <Text bold>{numLines}</Text>{' '}
            {numLines === 1 ? 'line' : 'lines'}
          </Text>
        </MessageResponse>
      );
    }

    case 'file_unchanged': {
      return (
        <MessageResponse height={1}>
          <Text dimColor>Unchanged since last read</Text>
        </MessageResponse>
      );
    }
  }
}
```

**显示效果**:

```
Read 42 lines
Read image (2.3 MB)
Read PDF (1.5 MB)
Read 3 pages (1.2 MB)
Read 156 cells
Unchanged since last read
```

#### 错误消息

```tsx
export function renderToolUseErrorMessage(
  result: ToolResultBlockParam['content'],
  { verbose }: { verbose: boolean }
): React.ReactNode {
  if (!verbose && typeof result === 'string') {
    // 文件不存在
    if (result.includes(FILE_NOT_FOUND_CWD_NOTE)) {
      return (
        <MessageResponse>
          <Text color="error">File not found</Text>
        </MessageResponse>
      );
    }

    // 其他错误
    if (extractTag(result, 'tool_use_error')) {
      return (
        <MessageResponse>
          <Text color="error">Error reading file</Text>
        </MessageResponse>
      );
    }
  }

  return <FallbackToolUseErrorMessage result={result} verbose={verbose} />;
}
```

---

## 使用示例

### 1. 读取文本文件

```typescript
// 完整读取
{
  name: 'Read',
  input: {
    file_path: '/home/user/project/src/index.ts'
  }
}

// 部分读取 (前 100 行)
{
  name: 'Read',
  input: {
    file_path: '/home/user/project/large_file.txt',
    offset: 1,
    limit: 100
  }
}

// 读取特定范围 (第 500-600 行)
{
  name: 'Read',
  input: {
    file_path: '/home/user/project/large_file.txt',
    offset: 500,
    limit: 100
  }
}
```

### 2. 读取图片

```typescript
{
  name: 'Read',
  input: {
    file_path: '/home/user/screenshot.png'
  }
}

// 输出 (自动压缩到 token 预算内)
{
  type: 'image',
  file: {
    base64: 'iVBORw0KGgoAAAANSUhEUgAA...',
    type: 'image/png',
    originalSize: 2458624,
    dimensions: {
      originalWidth: 1920,
      originalHeight: 1080,
      displayWidth: 1600,
      displayHeight: 900
    }
  }
}
```

### 3. 读取 PDF

```typescript
// 完整读取 (小 PDF)
{
  name: 'Read',
  input: {
    file_path: '/home/user/document.pdf'
  }
}

// 读取特定页面
{
  name: 'Read',
  input: {
    file_path: '/home/user/large_document.pdf',
    pages: '1-5'  // 读取前 5 页
  }
}

// 读取单页
{
  name: 'Read',
  input: {
    file_path: '/home/user/large_document.pdf',
    pages: '10'  // 只读第 10 页
  }
}
```

### 4. 读取 Jupyter Notebook

```typescript
{
  name: 'Read',
  input: {
    file_path: '/home/user/analysis.ipynb'
  }
}

// 输出
{
  type: 'notebook',
  file: {
    filePath: '/home/user/analysis.ipynb',
    cells: [
      {
        cell_type: 'markdown',
        source: ['# Data Analysis\n']
      },
      {
        cell_type: 'code',
        source: ['import pandas as pd\n', 'df = pd.read_csv("data.csv")\n'],
        outputs: [...]
      }
    ]
  }
}
```

---

## 总结

### 核心优势

1. **多文件类型支持**: 文本、图片、PDF、Notebook 一站式读取
2. **智能优化**: 自动压缩、去重、部分读取
3. **安全可靠**: 权限检查、路径验证、恶意软件警告
4. **性能优化**: Token 限制、流式读取、异步处理
5. **用户友好**: 友好错误提示、文件建议、进度显示

### 关键数字

| 指标 | 值 | 说明 |
|------|------|------|
| **代码行数** | 1184 | 核心实现文件 |
| **默认 token 上限** | 25000 | 可通过环境变量调整 |
| **默认字节上限** | 256 KB | 文件大小限制 |
| **PDF 最大页数** | 20 | 单次请求限制 |
| **去重命中率** | ~18% | 基于 BQ proxy 数据 |
| **Token 节省** | ~2.64% | cache_creation tokens |

### 使用建议

1. **大文件**: 使用 `offset` 和 `limit` 参数分段读取
2. **图片**: 自动压缩，无需手动处理
3. **PDF**: 大于 10 页时使用 `pages` 参数
4. **权限**: 配置合理的 allow/deny 规则
5. **性能**: 依赖自动去重和 token 限制

---

**FileReadTool** 是 Claude Code 中设计最精良的工具之一，兼顾了功能性、安全性和性能。它的实现展示了如何构建一个生产级的文件读取系统。
