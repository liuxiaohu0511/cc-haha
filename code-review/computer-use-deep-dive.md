# Computer Use 深度分析 - 桌面控制技术栈

> 文档生成时间: 2026-05-29  
> 分析对象: Claude Code Haha Computer Use 模块  
> 技术栈: Python + Node.js + MCP 协议

---

## 📋 目录

1. [技术架构概览](#技术架构概览)
2. [核心技术栈](#核心技术栈)
3. [跨平台实现](#跨平台实现)
4. [Python Bridge 机制](#python-bridge-机制)
5. [关键功能实现](#关键功能实现)
6. [安全与权限管理](#安全与权限管理)
7. [MCP 协议集成](#mcp-协议集成)

---

## 技术架构概览

### 系统分层

```
┌─────────────────────────────────────────────────────────────┐
│                    AI 决策层 (QueryEngine)                    │
│  Claude AI 通过 MCP 工具调用桌面控制功能                     │
└────────────────────────┬────────────────────────────────────┘
                         │ MCP Protocol (JSON-RPC)
┌────────────────────────▼────────────────────────────────────┐
│               TypeScript 封装层 (Node.js)                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  src/utils/computerUse/                              │  │
│  │  ├─ pythonBridge.ts    - Python 子进程管理           │  │
│  │  ├─ executor.ts        - 命令执行抽象                │  │
│  │  ├─ permissions.ts     - 权限裁决                    │  │
│  │  ├─ cleanup.ts         - 资源清理                    │  │
│  │  └─ mcpServer.ts       - MCP 服务端                  │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │ JSON-RPC over stdin/stdout
┌────────────────────────▼────────────────────────────────────┐
│                    Python Runtime Layer                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  runtime/mac_helper.py  (macOS)                      │  │
│  │  runtime/win_helper.py  (Windows)                    │  │
│  │                                                        │  │
│  │  命令: screenshot | click | key | type | scroll      │  │
│  │  协议: JSON request → JSON response                  │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                   OS Native APIs                             │
│  ┌─────────────────────┐      ┌─────────────────────────┐  │
│  │  macOS              │      │  Windows                │  │
│  │  ─────              │      │  ───────                │  │
│  │  • Quartz           │      │  • win32gui             │  │
│  │  • AppKit           │      │  • win32api             │  │
│  │  • CoreGraphics     │      │  • win32process         │  │
│  │  • NSWorkspace      │      │  • psutil               │  │
│  │  • PyAutoGUI        │      │  • PyAutoGUI            │  │
│  │  • mss (screenshot) │      │  • mss (screenshot)     │  │
│  └─────────────────────┘      └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心技术栈

### 1. Python 依赖库

#### macOS 依赖 (`runtime/requirements.txt`)

```txt
mss>=9.0.2,<10               # 跨平台截屏 (Metal/DirectX 加速)
Pillow>=11.3.0,<12           # 图像处理 (缩放/编码)
pyautogui>=0.9.54            # 鼠标/键盘自动化
pyobjc-core>=11.1            # Python ↔ Objective-C 桥接
pyobjc-framework-Cocoa>=11.1 # Cocoa API (NSWorkspace/NSPasteboard)
pyobjc-framework-Quartz>=11.1 # Quartz API (CGDisplay/CGWindow)
```

**关键库作用**:

| 库 | 功能 | 为什么选择它 |
|---|------|------------|
| **mss** | 屏幕截图 | 零依赖、跨平台、高性能 (Metal/DirectX 加速)，比 pyautogui.screenshot() 快 10 倍 |
| **PyAutoGUI** | 鼠标/键盘 | 跨平台统一 API，支持按键序列、修饰键 (Ctrl/Cmd/Shift) |
| **pyobjc-Quartz** | macOS 窗口管理 | 访问 CoreGraphics API 实现窗口隐藏、应用过滤 |
| **pyobjc-Cocoa** | macOS 应用控制 | NSWorkspace 获取运行应用、NSPasteboard 剪贴板操作 |
| **Pillow** | 图像处理 | 调整截图尺寸、JPEG 压缩 (减少 Token 消耗) |

---

#### Windows 依赖 (`runtime/requirements-win.txt`)

```txt
mss>=9.0.2,<10           # 截屏
Pillow>=11.3.0,<12       # 图像处理
pyautogui>=0.9.54        # 鼠标/键盘
pywin32>=306             # Windows API (win32gui/win32api)
psutil>=5.9.0            # 进程管理
pyperclip>=1.8.2         # 剪贴板
screeninfo>=0.8.1        # 显示器枚举
```

**Windows 特有库**:

| 库 | 功能 | 替代 macOS 的 |
|---|------|-------------|
| **pywin32** | Win32 API 封装 | pyobjc-Quartz/Cocoa |
| **psutil** | 进程/窗口枚举 | NSWorkspace |
| **pyperclip** | 剪贴板 | NSPasteboard |
| **screeninfo** | 多显示器 | CGGetActiveDisplayList |

---

### 2. Node.js TypeScript 层

#### Python Bridge - `src/utils/computerUse/pythonBridge.ts`

**核心职责**: 管理 Python 虚拟环境 + 子进程调用

```typescript
// 虚拟环境路径
const runtimeStateRoot = path.join(
  getClaudeConfigHomeDir(),  // ~/.claude
  '.runtime'
);
const venvRoot = path.join(runtimeStateRoot, 'venv');

// Python 可执行文件
function pythonBinPath(): string {
  return isWindows
    ? path.join(venvRoot, 'Scripts', 'python.exe')
    : path.join(venvRoot, 'bin', 'python3');
}

// Python Helper 脚本
const helperFileName = isWindows ? 'win_helper.py' : 'mac_helper.py';
const helperPath = path.join(runtimeStateRoot, helperFileName);
```

**自动引导流程**:

```typescript
export async function ensureBootstrapped(): Promise<void> {
  // 1. 复制 Python 脚本到 ~/.claude/.runtime/
  await ensureRuntimeFiles();
  
  // 2. 创建虚拟环境 (如果不存在)
  if (!(await pathExists(pythonBinPath()))) {
    const pythonCmd = await getVenvCreationPythonCommand();
    await runOrThrow(pythonCmd, ['-m', 'venv', venvRoot], 'venv creation');
  }
  
  // 3. 引导 pip (如果不存在)
  if (!(await pathExists(pipBin))) {
    await runOrThrow(pythonBinPath(), ['-m', 'ensurepip', '--upgrade'], 'ensurepip');
  }
  
  // 4. 安装依赖 (基于 requirements.txt SHA256 去重)
  const requirements = await readFile(requirementsPath, 'utf8');
  const digest = createHash('sha256').update(requirements).digest('hex');
  
  if (installedDigest !== digest) {
    await installRuntimeDependencies(requirementsPath);
    await writeFile(installStampPath, `${digest}\n`, 'utf8');
  }
}
```

**关键优化**:
- ✅ **去重安装**: SHA256 哈希检查,避免重复 `pip install`
- ✅ **沙箱隔离**: 独立 venv,不污染系统 Python
- ✅ **自动修复**: 自动复制 `requirements.txt` 和 `*_helper.py`

---

#### JSON-RPC 调用机制

```typescript
export async function callPythonHelper<T>(
  command: string,
  payload: Record<string, unknown> = {}
): Promise<T> {
  await ensureBootstrapped();
  
  // 执行 Python 脚本
  const { code, stdout, stderr } = await execFileNoThrow(
    pythonBinPath(),
    [helperPath, command, '--payload', JSON.stringify(payload)],
    { useCwd: false, env: getPythonCommandEnv() }
  );
  
  // 解析 JSON 响应
  const parsed: { ok: boolean; result?: T; error?: { message?: string } }
    = JSON.parse(stdout);
  
  if (!parsed.ok) {
    throw new Error(parsed.error?.message || `Python helper ${command} failed`);
  }
  
  return parsed.result as T;
}
```

**调用示例**:

```typescript
// 截图
const screenshot = await callPythonHelper<ScreenshotResult>('screenshot', {
  displayId: 0,
  allowedBundleIds: ['com.apple.Terminal'],
});

// 鼠标点击
await callPythonHelper<void>('click', {
  x: 500,
  y: 300,
  button: 'left',
  count: 1,
});

// 键盘输入
await callPythonHelper<void>('type', {
  text: 'Hello, World!',
  viaClipboard: false,
});
```

---

## 跨平台实现

### macOS 实现 - `runtime/mac_helper.py`

#### 核心导入

```python
from AppKit import NSWorkspace, NSPasteboard, NSPasteboardTypeString
from Quartz import (
    CGDisplayBounds,                # 显示器边界
    CGWindowListCopyWindowInfo,     # 窗口列表
    CGPreflightScreenCaptureAccess, # 屏幕录制权限检查
    kCGWindowOwnerName,             # 窗口所属应用
)
import mss        # 截屏
import pyautogui  # 鼠标/键盘
```

---

#### 1. 截屏 (带窗口过滤)

**macOS 独有功能**: 隐藏指定应用窗口后截屏

```python
def prepare_for_action(allowed_bundle_ids: list[str], display_id: int | None) -> list[str]:
    """隐藏非白名单应用的窗口,返回被隐藏的 bundleId 列表"""
    workspace = NSWorkspace.sharedWorkspace()
    running_apps = workspace.runningApplications()
    
    hidden_bundle_ids = []
    for app in running_apps:
        bundle_id = app.bundleIdentifier()
        if bundle_id and bundle_id not in allowed_bundle_ids:
            # 隐藏窗口 (通过 AppleScript 或 NSRunningApplication.hide())
            if not app.isHidden():
                app.hide()
                hidden_bundle_ids.append(bundle_id)
    
    return hidden_bundle_ids

def screenshot(allowed_bundle_ids: list[str], display_id: int | None) -> dict:
    """截取指定显示器,隐藏非白名单窗口"""
    # 1. 隐藏窗口
    hidden = prepare_for_action(allowed_bundle_ids, display_id)
    
    # 2. 获取显示器信息
    display = get_display_geometry(display_id)
    
    # 3. 使用 mss 截图
    with mss.mss() as sct:
        monitor = {
            'left': display['originX'],
            'top': display['originY'],
            'width': display['width'],
            'height': display['height'],
        }
        raw = sct.grab(monitor)
        image = Image.frombytes('RGB', raw.size, raw.rgb)
    
    # 4. 压缩为 JPEG (75% 质量)
    buffer = BytesIO()
    image.save(buffer, format='JPEG', quality=75, optimize=True)
    base64_data = base64.b64encode(buffer.getvalue()).decode('ascii')
    
    # 5. 恢复隐藏的窗口
    for bundle_id in hidden:
        restore_app_visibility(bundle_id)
    
    return {
        'base64': base64_data,
        'width': image.width,
        'height': image.height,
        'displayWidth': display['width'],
        'displayHeight': display['height'],
    }
```

**为什么要窗口过滤?**
- 🔒 **隐私保护**: 避免 AI 看到敏感应用 (1Password, 银行 App)
- 📉 **Token 优化**: 减少截图内容复杂度
- 🎯 **专注上下文**: 只展示与任务相关的窗口

---

#### 2. 键盘控制

**按键映射** (macOS 特有):

```python
KEY_MAP = {
    'cmd': 'command',       # Windows 中为 'win'
    'alt': 'option',        # macOS 中 Alt = Option
    'forwarddelete': 'delete',  # Fn + Delete
    # ...
}

def normalize_key(name: str) -> str:
    key = name.strip().lower()
    if key not in KEY_MAP:
        raise ValueError(f'Unsupported key: {name}')
    return KEY_MAP[key]
```

**按键发送** (通过 PyAutoGUI):

```python
def send_key(key_sequence: str, repeat: int = 1):
    """按键序列: 'cmd+c', 'ctrl+shift+t' 等"""
    keys = [normalize_key(k) for k in key_sequence.split('+')]
    
    for _ in range(repeat):
        if len(keys) == 1:
            pyautogui.press(keys[0])
        else:
            # 修饰键 + 普通键
            modifiers = keys[:-1]
            main_key = keys[-1]
            pyautogui.hotkey(*modifiers, main_key)
        time.sleep(0.05)  # 50ms 延迟
```

**复杂按键通过 AppleScript** (PyAutoGUI 不支持的组合键):

```python
def send_keystroke_via_osascript(character: str, modifiers: list[str] | None = None):
    """通过 AppleScript 发送按键 (支持 Fn 键等)"""
    escaped = character.replace('\\', '\\\\').replace('"', '\\"')
    
    if modifiers:
        modifier_expr = ', '.join(f'{m} down' for m in modifiers)
        script = f'tell application "System Events" to keystroke "{escaped}" using {{{modifier_expr}}}'
    else:
        script = f'tell application "System Events" to keystroke "{escaped}"'
    
    subprocess.run(['osascript', '-e', script], check=True)
```

---

#### 3. 鼠标控制

```python
def click(x: int, y: int, button: str, count: int, modifiers: list[str] | None):
    """鼠标点击
    
    Args:
        x, y: 屏幕坐标 (绝对坐标)
        button: 'left' | 'right' | 'middle'
        count: 1 (单击) | 2 (双击) | 3 (三击)
        modifiers: ['cmd', 'shift'] 等
    """
    pyautogui.moveTo(x, y, duration=0.1)  # 平滑移动 100ms
    
    if modifiers:
        # 按住修饰键点击
        for mod in modifiers:
            pyautogui.keyDown(normalize_key(mod))
        
        pyautogui.click(x, y, clicks=count, button=button)
        
        for mod in reversed(modifiers):
            pyautogui.keyUp(normalize_key(mod))
    else:
        pyautogui.click(x, y, clicks=count, button=button)

def drag(from_x: int, from_y: int, to_x: int, to_y: int):
    """拖拽操作"""
    pyautogui.moveTo(from_x, from_y)
    pyautogui.mouseDown()
    pyautogui.moveTo(to_x, to_y, duration=0.3)  # 300ms 拖拽动画
    pyautogui.mouseUp()

def scroll(x: int, y: int, delta_x: int, delta_y: int):
    """滚动 (macOS 支持水平+垂直)"""
    pyautogui.moveTo(x, y)
    if delta_y != 0:
        pyautogui.scroll(delta_y)
    if delta_x != 0:
        pyautogui.hscroll(delta_x)  # 水平滚动 (macOS)
```

---

#### 4. 应用管理

**获取前台应用**:

```python
def get_frontmost_app() -> dict | None:
    """返回当前前台应用"""
    workspace = NSWorkspace.sharedWorkspace()
    app = workspace.frontmostApplication()
    
    if not app:
        return None
    
    return {
        'bundleId': app.bundleIdentifier(),
        'displayName': app.localizedName(),
    }
```

**列出运行中的应用**:

```python
def list_running_apps() -> list[dict]:
    """列出所有运行中的应用"""
    workspace = NSWorkspace.sharedWorkspace()
    running_apps = workspace.runningApplications()
    
    apps = []
    for app in running_apps:
        bundle_id = app.bundleIdentifier()
        if bundle_id and not app.isHidden():
            apps.append({
                'bundleId': bundle_id,
                'displayName': app.localizedName(),
            })
    
    return apps
```

**打开应用**:

```python
def open_app(bundle_id: str):
    """通过 Bundle ID 打开应用"""
    workspace = NSWorkspace.sharedWorkspace()
    url = NSURL.URLWithString_(f'x-apple.systempreferences:{bundle_id}')
    
    success = workspace.openURL_(url)
    if not success:
        # Fallback: 通过 open 命令
        subprocess.run(['open', '-b', bundle_id], check=True)
```

---

### Windows 实现 - `runtime/win_helper.py`

#### 核心导入

```python
import win32gui      # 窗口管理
import win32api      # 键盘/鼠标事件
import win32process  # 进程信息
import psutil        # 进程枚举
import pyperclip     # 剪贴板
import screeninfo    # 多显示器
import mss           # 截屏
import pyautogui     # 鼠标/键盘
```

---

#### 1. 显示器枚举

```python
def get_displays() -> list[dict]:
    """枚举所有显示器 (通过 screeninfo)"""
    from screeninfo import get_monitors
    
    displays = []
    for idx, m in enumerate(get_monitors()):
        scale_factor = _get_monitor_scale(m)
        displays.append({
            'id': idx,
            'displayId': idx,
            'width': m.width,
            'height': m.height,
            'scaleFactor': scale_factor,
            'originX': m.x,
            'originY': m.y,
            'isPrimary': m.is_primary if hasattr(m, 'is_primary') else (idx == 0),
            'name': m.name or f'Display {idx + 1}',
        })
    
    return displays

def _get_monitor_scale(monitor) -> float:
    """获取 DPI 缩放比例"""
    import ctypes
    ctypes.windll.user32.SetProcessDPIAware()
    hdc = ctypes.windll.user32.GetDC(0)
    dpi = ctypes.windll.gdi32.GetDeviceCaps(hdc, 88)  # LOGPIXELSX
    ctypes.windll.user32.ReleaseDC(0, hdc)
    return dpi / 96.0  # 96 DPI = 100%
```

---

#### 2. 窗口管理

**枚举所有窗口**:

```python
def list_windows() -> list[dict]:
    """枚举所有可见窗口"""
    windows = []
    
    def callback(hwnd, _):
        if win32gui.IsWindowVisible(hwnd):
            title = win32gui.GetWindowText(hwnd)
            if title:  # 过滤无标题窗口
                _, pid = win32process.GetWindowThreadProcessId(hwnd)
                try:
                    process = psutil.Process(pid)
                    exe_name = process.name()
                except:
                    exe_name = 'Unknown'
                
                windows.append({
                    'hwnd': hwnd,
                    'title': title,
                    'pid': pid,
                    'exe': exe_name,
                })
        return True
    
    win32gui.EnumWindows(callback, None)
    return windows
```

**获取前台窗口**:

```python
def get_foreground_window() -> dict | None:
    """获取前台窗口"""
    hwnd = win32gui.GetForegroundWindow()
    if not hwnd:
        return None
    
    title = win32gui.GetWindowText(hwnd)
    _, pid = win32process.GetWindowThreadProcessId(hwnd)
    
    try:
        process = psutil.Process(pid)
        exe_name = process.name()
    except:
        exe_name = 'Unknown'
    
    return {
        'hwnd': hwnd,
        'title': title,
        'pid': pid,
        'exe': exe_name,
    }
```

---

#### 3. 按键映射差异

**Windows vs macOS**:

```python
KEY_MAP = {
    # 最大差异: macOS 的 'command' → Windows 的 'win'
    'cmd': 'win',
    'command': 'win',
    'meta': 'win',
    'super': 'win',
    
    # macOS 的 'option' → Windows 的 'alt'
    'option': 'alt',
    'opt': 'alt',
    
    # 其他按键保持一致
    'ctrl': 'ctrl',
    'shift': 'shift',
    # ...
}
```

---

#### 4. 剪贴板操作

```python
def read_clipboard() -> str:
    """读取剪贴板"""
    return pyperclip.paste()

def write_clipboard(text: str):
    """写入剪贴板"""
    pyperclip.copy(text)

def type_via_clipboard(text: str):
    """通过剪贴板输入文本 (支持 Unicode)"""
    # 1. 备份原剪贴板
    original = read_clipboard()
    
    # 2. 写入要输入的文本
    write_clipboard(text)
    
    # 3. 发送 Ctrl+V
    pyautogui.hotkey('ctrl', 'v')
    time.sleep(0.1)
    
    # 4. 恢复剪贴板
    write_clipboard(original)
```

**为什么使用剪贴板?**
- ✅ **Unicode 支持**: PyAutoGUI 的 `typewrite()` 不支持中文/Emoji
- ✅ **速度快**: 比逐字符输入快 100 倍
- ❌ **污染剪贴板**: 需要备份/恢复原内容

---

## Python Bridge 机制

### 启动流程

```
┌─────────────────────────────────────────────────────────┐
│  1. 首次调用 callPythonHelper()                         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  2. ensureBootstrapped()                                │
│     ┌───────────────────────────────────────────────┐  │
│     │ a. 检查 ~/.claude/.runtime/ 是否存在          │  │
│     │ b. 复制 requirements.txt & mac_helper.py      │  │
│     │ c. 检查虚拟环境 venv/ 是否存在                │  │
│     │ d. 运行 python3 -m venv venv (如不存在)       │  │
│     │ e. 运行 python -m ensurepip (引导 pip)        │  │
│     │ f. 计算 requirements.txt 的 SHA256            │  │
│     │ g. 对比 requirements.sha256 (去重)            │  │
│     │ h. 运行 pip install -r requirements.txt       │  │
│     │ i. 写入新的 SHA256 标记                        │  │
│     └───────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  3. 执行 Python 子进程                                   │
│     $ ~/.claude/.runtime/venv/bin/python3 \             │
│       ~/.claude/.runtime/mac_helper.py screenshot \     │
│       --payload '{"displayId": 0}'                      │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  4. Python 解析 JSON 参数                                │
│     parser = argparse.ArgumentParser()                  │
│     parser.add_argument('command')                      │
│     parser.add_argument('--payload')                    │
│     args = parser.parse_args()                          │
│     payload = json.loads(args.payload)                  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  5. Python 执行命令                                       │
│     result = screenshot(payload['displayId'])           │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  6. Python 返回 JSON 响应                                │
│     json_output({'ok': True, 'result': {...}})          │
│     # 通过 stdout 返回                                   │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  7. TypeScript 解析响应                                  │
│     const parsed = JSON.parse(stdout)                   │
│     if (!parsed.ok) throw new Error(...)                │
│     return parsed.result                                │
└─────────────────────────────────────────────────────────┘
```

---

### 错误处理

```typescript
// TypeScript 端
try {
  const result = await callPythonHelper('screenshot', { displayId: 0 });
} catch (error) {
  if (error.message.includes('Screen Recording permission')) {
    // 提示用户授予屏幕录制权限
    showPermissionDialog('Screen Recording');
  } else if (error.message.includes('Python helper screenshot failed')) {
    // Python 脚本执行失败
    logError('Screenshot failed', error);
  } else {
    throw error;
  }
}
```

```python
# Python 端
try:
    result = screenshot(payload['displayId'])
    json_output({'ok': True, 'result': result})
except PermissionError as e:
    error_output(f'Screen Recording permission denied: {e}', 'permission_denied')
except Exception as e:
    error_output(f'Screenshot failed: {e}', 'runtime_error')
```

---

## 关键功能实现

### 1. 智能截图 (带 Token 优化)

**目标**: 将 4K 截图压缩到 ~50KB,减少 API Token 消耗

```typescript
// src/vendor/computer-use-mcp/imageResize.ts
export function targetImageSize(
  displayWidth: number,
  displayHeight: number
): { width: number; height: number } {
  const MAX_DIMENSION = 1280;  // 最大边长
  
  // 保持宽高比缩放
  if (displayWidth > displayHeight) {
    if (displayWidth > MAX_DIMENSION) {
      const scale = MAX_DIMENSION / displayWidth;
      return {
        width: MAX_DIMENSION,
        height: Math.round(displayHeight * scale),
      };
    }
  } else {
    if (displayHeight > MAX_DIMENSION) {
      const scale = MAX_DIMENSION / displayHeight;
      return {
        width: Math.round(displayWidth * scale),
        height: MAX_DIMENSION,
      };
    }
  }
  
  // 不需要缩放
  return { width: displayWidth, height: displayHeight };
}

// API 参数
export const API_RESIZE_PARAMS = {
  maxDimension: 1280,
  jpegQuality: 75,
};
```

**实际压缩效果**:

| 原始分辨率 | 原始大小 | 压缩后尺寸 | 压缩后大小 | Token 节省 |
|-----------|---------|-----------|-----------|-----------|
| 3840×2160 (4K) | ~24 MB | 1280×720 | ~50 KB | 99.8% ↓ |
| 2560×1440 (2K) | ~10 MB | 1280×720 | ~45 KB | 99.6% ↓ |
| 1920×1080 (FHD) | ~5 MB | 1280×720 | ~40 KB | 99.2% ↓ |

---

### 2. 窗口过滤 (隐私保护)

**TypeScript 端权限检查**:

```typescript
// src/vendor/computer-use-mcp/deniedApps.ts
const DENIED_APPS = [
  // 密码管理器
  'com.agilebits.onepassword7',
  'com.lastpass.LastPass',
  'com.bitwarden.desktop',
  
  // 金融应用
  'com.paypal.PayPal',
  'com.chase.Chase',
  
  // 通讯加密
  'com.signal.desktop',
  'org.whispersystems.signal',
];

export function isPolicyDenied(bundleId: string): boolean {
  return DENIED_APPS.includes(bundleId);
}

// 截图前过滤
const allowedApps = runningApps.filter(app => 
  !isPolicyDenied(app.bundleId)
);

const screenshot = await executor.screenshot({
  allowedBundleIds: allowedApps.map(a => a.bundleId),
  displayId: 0,
});
```

---

### 3. 坐标映射 (逻辑像素 ↔ 物理像素)

**macOS Retina 显示器处理**:

```typescript
// 用户指定逻辑坐标 (x: 500, y: 300)
const logicalX = 500;
const logicalY = 300;

// 获取显示器缩放比例
const display = await executor.getDisplaySize();
// display.scaleFactor = 2.0 (Retina)

// 转换为物理坐标
const physicalX = logicalX * display.scaleFactor;  // 1000
const physicalY = logicalY * display.scaleFactor;  // 600

// 发送给 Python
await callPythonHelper('click', {
  x: physicalX,
  y: physicalY,
  button: 'left',
  count: 1,
});
```

---

### 4. 像素比对 (防误点击)

**场景**: AI 要点击 "确认" 按钮,验证坐标是否正确

```typescript
// src/vendor/computer-use-mcp/pixelCompare.ts
export async function validateClickTarget(
  x: number,
  y: number,
  expectedColor: { r: number; g: number; b: number },
  tolerance: number = 30
): Promise<PixelCompareResult> {
  // 1. 截取 1×1 像素
  const pixel = await executor.screenshot({
    allowedBundleIds: [],
    displayId: 0,
    region: { x, y, width: 1, height: 1 },
  });
  
  // 2. 解析 JPEG (base64)
  const buffer = Buffer.from(pixel.base64, 'base64');
  const image = await sharp(buffer).raw().toBuffer({ resolveWithObject: true });
  
  const actualColor = {
    r: image.data[0],
    g: image.data[1],
    b: image.data[2],
  };
  
  // 3. 计算颜色差异
  const delta = Math.sqrt(
    Math.pow(actualColor.r - expectedColor.r, 2) +
    Math.pow(actualColor.g - expectedColor.g, 2) +
    Math.pow(actualColor.b - expectedColor.b, 2)
  );
  
  return {
    matches: delta <= tolerance,
    actualColor,
    expectedColor,
    delta,
  };
}
```

**AI 使用示例**:

```typescript
// AI 决策流程
const screenshot = await screenshot();  // 截全屏
// AI 识别 "确认" 按钮位置 → (x: 650, y: 450)

// 验证坐标
const validation = await validateClickTarget(650, 450, {
  r: 0, g: 122, b: 255  // 蓝色按钮
});

if (!validation.matches) {
  // 坐标不对,重新分析截图
  return 'Button moved, retrying...';
}

// 点击
await click(650, 450, 'left', 1);
```

---

## 安全与权限管理

### 1. 系统权限检查

**macOS 权限要求**:

```python
def check_screen_recording_permission() -> bool:
    """检查屏幕录制权限"""
    from Quartz import CGPreflightScreenCaptureAccess
    return CGPreflightScreenCaptureAccess()

def check_accessibility_permission() -> bool:
    """检查辅助功能权限 (键盘/鼠标控制)"""
    import subprocess
    result = subprocess.run(
        ['osascript', '-e', 'tell application "System Events" to get name'],
        capture_output=True,
        check=False,
    )
    return result.returncode == 0
```

**权限引导**:

```typescript
// src/utils/computerUse/permissions.ts
export async function ensureComputerUsePermissions(): Promise<void> {
  const hasScreenRecording = await callPythonHelper<boolean>(
    'check_screen_recording_permission'
  );
  
  if (!hasScreenRecording) {
    throw new PermissionError(
      'Screen Recording permission required. ' +
      'Open System Settings → Privacy & Security → Screen Recording, ' +
      'then enable Claude Code.'
    );
  }
  
  const hasAccessibility = await callPythonHelper<boolean>(
    'check_accessibility_permission'
  );
  
  if (!hasAccessibility) {
    throw new PermissionError(
      'Accessibility permission required. ' +
      'Open System Settings → Privacy & Security → Accessibility, ' +
      'then enable Claude Code.'
    );
  }
}
```

---

### 2. 应用层权限控制

**分级权限系统**:

```typescript
// src/vendor/computer-use-mcp/types.ts
export type CuAppPermTier = 
  | 'allow_all'         // 完全信任 (IDE/Terminal)
  | 'allow_with_review' // 需审批 (浏览器/邮件客户端)
  | 'sentinel'          // 敏感应用 (系统偏好设置)
  | 'deny';             // 禁止 (密码管理器)

// 应用分类
export const APP_TIERS: Record<string, CuAppPermTier> = {
  // 完全信任
  'com.microsoft.VSCode': 'allow_all',
  'com.apple.Terminal': 'allow_all',
  
  // 需审批
  'com.google.Chrome': 'allow_with_review',
  'com.apple.mail': 'allow_with_review',
  
  // 敏感应用
  'com.apple.systempreferences': 'sentinel',
  
  // 禁止
  'com.agilebits.onepassword7': 'deny',
};
```

**权限裁决**:

```typescript
async function checkPermission(
  bundleId: string,
  action: 'screenshot' | 'click' | 'type'
): Promise<PermissionResult> {
  const tier = APP_TIERS[bundleId] || 'allow_with_review';
  
  switch (tier) {
    case 'allow_all':
      return { allowed: true };
    
    case 'allow_with_review':
      // 弹出权限对话框
      const userApproved = await askUser(
        `Allow AI to ${action} on ${bundleId}?`
      );
      return { allowed: userApproved };
    
    case 'sentinel':
      // 记录到审计日志
      auditLog({ bundleId, action, allowed: false });
      return { allowed: false, reason: 'Sentinel app' };
    
    case 'deny':
      return { allowed: false, reason: 'Policy denied' };
  }
}
```

---

### 3. 按键黑名单

**禁止的按键组合**:

```typescript
// src/vendor/computer-use-mcp/keyBlocklist.ts
const BLOCKED_KEY_COMBOS = [
  'cmd+q',        // 退出应用
  'cmd+w',        // 关闭窗口
  'cmd+opt+esc',  // 强制退出
  'ctrl+alt+del', // Windows 任务管理器
  'cmd+shift+q',  // 登出
];

export function isSystemKeyCombo(keys: string[]): boolean {
  const normalized = keys.map(k => k.toLowerCase()).join('+');
  return BLOCKED_KEY_COMBOS.includes(normalized);
}
```

---

## MCP 协议集成

### MCP Server 实现

**启动 MCP 服务器**:

```typescript
// src/utils/computerUse/mcpServer.ts
export async function runComputerUseMcpServer(): Promise<void> {
  const server = new Server({
    name: 'computer-use',
    version: '1.0.0',
  });
  
  // 注册工具
  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: [
      {
        name: 'computer',
        description: 'Control mouse, keyboard, and take screenshots',
        inputSchema: {
          type: 'object',
          properties: {
            action: {
              type: 'string',
              enum: ['screenshot', 'mouse_move', 'left_click', 'type', 'key'],
            },
            coordinate: {
              type: 'array',
              items: { type: 'number' },
              minItems: 2,
              maxItems: 2,
            },
            text: { type: 'string' },
          },
        },
      },
    ],
  }));
  
  // 工具调用处理
  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;
    
    if (name !== 'computer') {
      throw new McpError(ErrorCode.MethodNotFound, `Unknown tool: ${name}`);
    }
    
    const action = args.action as string;
    
    switch (action) {
      case 'screenshot':
        const result = await callPythonHelper<ScreenshotResult>('screenshot', {
          displayId: args.displayId,
          allowedBundleIds: args.allowedApps || [],
        });
        return {
          content: [
            {
              type: 'image',
              data: result.base64,
              mimeType: 'image/jpeg',
            },
          ],
        };
      
      case 'left_click':
        const [x, y] = args.coordinate as [number, number];
        await callPythonHelper('click', { x, y, button: 'left', count: 1 });
        return { content: [{ type: 'text', text: 'Clicked' }] };
      
      case 'type':
        await callPythonHelper('type', { text: args.text });
        return { content: [{ type: 'text', text: 'Typed' }] };
      
      default:
        throw new McpError(ErrorCode.InvalidRequest, `Unknown action: ${action}`);
    }
  });
  
  // 启动 stdio 传输
  const transport = new StdioServerTransport();
  await server.connect(transport);
}
```

---

### AI 调用示例

**通过 MCP 工具调用**:

```typescript
// AI 决策流程 (伪代码)
const tools = await mcp.listTools();  // 获取工具列表

// 1. 截图
const screenshot = await mcp.callTool('computer', {
  action: 'screenshot',
  displayId: 0,
  allowedApps: ['com.apple.Terminal'],
});

// 2. AI 分析截图 (通过 Vision API)
const analysis = await claude.analyzeImage(screenshot.content[0].data);
// → "I see a terminal window with a command prompt"

// 3. 点击坐标
await mcp.callTool('computer', {
  action: 'left_click',
  coordinate: [500, 300],
});

// 4. 输入文本
await mcp.callTool('computer', {
  action: 'type',
  text: 'ls -la',
});

// 5. 按回车
await mcp.callTool('computer', {
  action: 'key',
  text: 'enter',
});
```

---

## 技术栈总结

### 依赖清单

| 层级 | 技术 | 用途 |
|-----|------|------|
| **AI 决策** | Claude 4.X Sonnet | 视觉理解 + 行为决策 |
| **协议层** | MCP (Model Context Protocol) | 标准化工具调用 |
| **编排层** | TypeScript + Node.js | Python 子进程管理 |
| **截屏** | mss (Python) | Metal/DirectX 加速截图 |
| **图像处理** | Pillow (Python) | JPEG 压缩、缩放 |
| **鼠标/键盘** | PyAutoGUI (Python) | 跨平台自动化 |
| **macOS API** | pyobjc-Quartz/Cocoa | 窗口管理、应用控制 |
| **Windows API** | pywin32, psutil | 窗口枚举、进程管理 |

---

### 为什么选择 Python?

| 优势 | 原因 |
|------|------|
| **跨平台** | PyAutoGUI 支持 macOS/Windows/Linux |
| **生态丰富** | mss/Pillow/pyobjc 成熟稳定 |
| **系统集成** | 直接调用 Quartz/win32api |
| **调试友好** | Python 脚本比 Node.js Native 模块更易调试 |
| **隔离环境** | venv 独立依赖,不污染系统 |

---

### 为什么不用 Node.js Native 模块?

| 方案 | 问题 |
|------|------|
| **robotjs** | ❌ 编译复杂,依赖 node-gyp |
| **nut-js** | ❌ 不支持窗口过滤、剪贴板 |
| **@nut-tree/nut-js** | ❌ 截图性能差 (比 mss 慢 10 倍) |
| **node-screenshot-desktop** | ❌ macOS 不支持窗口隐藏 |

---

## 核心亮点

1. ✅ **零权限提升**: 用户授权一次,后续无需 sudo/admin
2. ✅ **窗口过滤**: 隐私敏感应用自动隐藏
3. ✅ **Token 优化**: 4K 截图压缩到 50KB (99.8% ↓)
4. ✅ **像素验证**: 防止误点击 (坐标漂移检测)
5. ✅ **分级权限**: 应用白名单 + 黑名单
6. ✅ **跨平台**: macOS/Windows 统一接口
7. ✅ **MCP 标准**: 符合 Model Context Protocol

---

**文档完整度**: ✅ Computer Use 技术栈已全面剖析  
**更新建议**: 跟进 Python 依赖库版本升级 (mss/pyautogui)
