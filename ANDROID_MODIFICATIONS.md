# BetterGI & March7thAssistant Android 移植修改清单

> 基于 MAA-Meow 移植模式，逐层分析需要替换的模块

## 核心思路：MAA-Meow 做了什么

MAA-Meow 的移植本质上是**替换了三个平台层**，而复用了 MAA Core 的 C++ 识别+任务引擎：

```
PC 端 MAA:
  截图: ADB screencap / Win32 GDI     → Android: VirtualDisplay + ImageReader (NDK)
  输入: ADB input / Win32 SendMessage  → Android: MotionEvent 注入 (NDK + Shizuku/Root)
  引擎: C++ libMaaCore.so (不变)      → Android: JNA 加载同一个 .so (不变)

GUI: C# WPF → 全部重写为 Kotlin/Compose (完全重写，但跟核心逻辑无关)
```

关键洞察：**识别算法和任务逻辑不需要重写**，只有"看到屏幕"和"操作屏幕"两件事需要替换。

---

## BetterGI 移植到 Android 需要改什么

### 第一层：截屏 — 必须替换

**现状**：`Fischless.GameCapture` 依赖 DirectX/Win32
```csharp
// BitBltCapture.cs — GDI32.BitBlt
// GraphicsCapture.cs — SharpDX.Direct3D11 + WinRT
// SharedSurfaceCapture.cs — DwmSharedSurface
```

**改为**：复用 MAA-Meow 的方案
```
Fischless.GameCapture/
  ├── BitBlt/              → 删掉
  ├── Graphics/            → 删掉
  ├── DwmSharedSurface/    → 删掉
  └── IGameCapture.cs      → 保留接口，新实现：
       └── VirtualDisplayCapture (Android)
            - VirtualDisplayManager 创建虚拟显示器
            - ImageReader 获取帧数据 (RGBA_8888)
            - 输出 cv::Mat / OpenCV Mat
```

具体改动：
- 移植 MAA-Meow 的 `VirtualDisplayManager.kt` + `bridge_frame_buffer.cpp` (NDK)
- `CaptureContent` 的 `Mat image` 输入源从 Win32 截屏改为 ImageReader 回调
- 分辨率归一化逻辑（`DeriveTo1080P()`）保留，但 crop 坐标需适配 Android 虚拟显示器

### 第二层：操控 — 必须替换

**现状**：`Fischless.WindowsInput` 依赖 Win32 `SendInput`
```csharp
// InputSimulator.cs → User32.SendInput
// PostMessageSimulator.cs → User32.PostMessage
```

**改为**：Shizuku/Root MotionEvent 注入
```
Fischless.WindowsInput/         → 替换为
├── InputSimulator.cs           →   AndroidInputInjector (Kotlin/JNI)
│   └── SendInput(key)              ├── touchDown(x, y)
│   └── SendInput(mouse)            ├── touchMove(x, y)
│                                    ├── touchUp()
PostMessageSimulator.cs → 删掉      ├── injectKeyEvent(keyCode)
                                     └── injectText(text)
```

具体改动：
- 移植 MAA-Meow 的 `bridge_input.cpp` + `InputControlUtils`
- `Simulation.SendInput.Keyboard.KeyPress(VK_F)` → `touchDown/touchUp` 在 F 键对应坐标
- `Simulation.SendInput.Mouse.VerticalScroll(2)` → `MotionEvent` 模拟滚轮（Android 手势）
- **难点**：BetterGI 大量依赖 `PostMessage` 做后台操作（不抢焦点），Android 上没有等价方案，需要走 VirtualDisplay 前台模式

### 第三层：识别引擎 — 大部分可保留

**可直接复用**（OpenCV 是跨平台的）：
- `RecognitionTypes.TemplateMatch` — OpenCV `matchTemplate`，Android 上 API 一致
- `RecognitionTypes.ColorMatch` — 纯 OpenCV 操作
- `RecognitionTypes.ColorRangeAndOcr` — 同上
- `BvImage` / `RecognitionObject` — 数据结构不变

**需要替换后端**：
- `PaddleOcrService` — PC 端用 PaddleOCR C++，Android 上要换成 ncnn 后端（参考 MAA-Meow 的 `convert_ocr_ncnn.py`）或 ONNX Runtime Android
- `BgiYoloPredictor` — ONNX Runtime DirectML（Windows GPU）→ ONNX Runtime Android (NNAPI delegate) 或 ncnn
- `PickTextInference` (SVTR) — ONNX 模型本身跨平台，推理后端替换即可

**需要适配**：
- 模板图片资源路径（PC 端用文件系统，Android 端用 assets）
- OCR 结果后处理逻辑（文本替换、正则过滤）不变

### 第四层：UI — 完全重写

```
WPF (BetterGenshinImpact/View/)        → Jetpack Compose
├── MainWindow.xaml                      → 删除
├── Pages/*.xaml                         → Compose Screens
├── Drawable/*.cs                        → Compose Canvas
└── ViewModel/                           → 保留逻辑，换 ViewModel 框架
     └── CommunityToolkit.Mvvm           → AndroidViewModel + StateFlow
```

### 第五层：任务逻辑 — 可大部分保留

**可保留**（纯业务逻辑，与平台无关）：
- `AutoPickTrigger` 的识别+判断逻辑（找 F 键 → 按 F）
- `AutoSkipTrigger` 的对话跳过逻辑
- `AutoDomainTask` 的秘境步骤编排
- `CombatScenes` 的角色识别+战斗脚本
- `AutoPathingTask` 的路线导航逻辑
- 所有 `Config` 类

**需要适配**：
- `ITaskTrigger.OnCapture()` 的调度从 WPF Dispatcher 改为 Android Handler/Coroutine
- `TaskTriggerDispatcher` 的帧循环从 Win32 Timer 改为 Android `Choreographer` 或固定频率协程
- 线程模型：WPF 的 Dispatcher → Android 的 Main/IO Dispatcher

---

## March7thAssistant 移植到 Android 需要改什么

### 第一层：截屏 — 必须替换

**现状**：
```python
# screenshot.py
import mss
sct.grab(region)  # mss 截屏

# 或云游戏模式
screenshot_bytes = driver.execute_cdp_cmd("Page.captureScreenshot", {})
```

**改为**：
```
module/automation/screenshot.py
├── mss → VirtualDisplay + ImageReader (通过 JNI 调用 MAA-Meow bridge)
├── win32gui → Android WindowManager API
└── 云游戏模式 → 保留（CDP 截图在 Android WebView 中也可用）
```

### 第二层：操控 — 必须替换

**现状**：
```python
# local_input.py
import pyautogui
pyautogui.click(x, y)
pyautogui.keyDown(key)
```

**改为**：
```
module/automation/
├── local_input.py (pyautogui) → android_input.py
│   ├── InputBase 接口不变
│   └── 实现改为 Shizuku/Root MotionEvent 注入
├── cdp_input.py → 保留（云游戏模式在 Android 上同样可用）
└── input_base.py → 不变（抽象接口）
```

关键改动：
- `LocalInput` 的每个方法从 `pyautogui.xxx` 改为 JNI 调用 Native bridge
- `CdpInput` **完全保留** — 这是 March7thAssistant 的最大优势
- `Automation.take_screenshot()` 的截图源替换

### 第三层：OCR — 后端替换

**现状**：
```python
# ocr.py
from rapidocr_onnxruntime import RapidOCR
ocr = RapidOCR()
result = ocr(img)
```

**改为**：
- `RapidOCR` → ONNX Runtime Android 版本（可用 Chaquopy 直接跑 Python RapidOCR，或用 Kotlin 重写为 ncnn 调用）
- 内存管理逻辑（`OCR_PERIODIC_FULL_GC_INTERVAL`、`OCR_OPENVINO_REINIT_INTERVAL`）保留

### 第四层：识别逻辑 — 可大部分保留

```python
# automation.py
def find_element(self, target, find_type, ...):
    if find_type == "image":
        # cv2.matchTemplate → OpenCV Android 直接可用
    elif find_type == "text":
        # OCR → 后端替换，逻辑不变
    elif find_type == "hsv":
        # cv2.cvtColor → OpenCV Android 直接可用
```

`find_element` / `click_element` 的核心逻辑全部保留，只替换底层截图和输入。

### 第五层：界面状态机 — 完全保留

```python
# screen.py
class Screen:
    def __init__(self, config_path):
        # screens.json → 保留
        self.screen_map = {}
    
    def get_current_screen(self):
        # 模板匹配判断当前界面 → 逻辑不变
```

`screens.json` 配置文件和 `Screen` 类完全保留。

### 第六层：任务逻辑 — 大部分可保留

```python
# daily.py, power.py, instance.py
# 纯 Python 流程编排，调用 auto.find_element / auto.click_element / screen.switch_to
```

**可保留**：
- `Daily.prepare_daily()` 全部逻辑
- `Power.run()` 体力管理
- `Instance.run()` 副本循环
- `Tasks.detect()` 每日实训检测
- 所有通知推送（`onepush` 跨平台）

**需要适配**：
- `pygetwindow` → Android 无窗口管理器概念，删除
- `winotify` → Android 通知 API
- `pyuac`（UAC 提权）→ 不需要，Android 走 Shizuku
- 子项目 `Auto_Simulated_Universe` / `Fhoe-Rail` 如果是纯 Python + OpenCV，可保留

### 第七层：GUI — 完全重写

```
app/ (PySide6) → Jetpack Compose 或 WebView
├── MainWindow → Compose Activity
├── Fluent Widgets → Material3 Components
└── 但 March7thAssistant 有命令行模式 (main.py) → 可先做 CLI 版本验证
```

---

## 移植难度对比

| 维度 | BetterGI | March7thAssistant |
|---|---|---|
| **截屏替换** | 中（需替换 3 种 Win32 截屏为 VirtualDisplay） | 低（mss 替换，云游戏模式保留） |
| **操控替换** | **高**（大量依赖 PostMessage 后台操作） | 低（pyautogui 替换，CDP 保留） |
| **识别层** | 中（YOLO + OCR 后端替换，OpenCV 保留） | 低（OCR 后端替换，OpenCV 保留） |
| **任务逻辑** | 低-中（纯业务逻辑保留） | **极低**（Python 流程几乎不动） |
| **GUI** | 高（WPF → Compose 完全重写） | 中（PySide6 → Compose，或先保留 CLI） |
| **平台依赖** | **极深**（Win32 API 遍布代码） | 中（集中在 `module/automation/` 和 `module/screen/`） |

## 结论

- **March7thAssistant** 移植难度远低于 BetterGI，因为它的接口抽象做得好（`InputBase` / `GameControllerBase`），且已有云游戏模式可以绕开最难的截屏/输入问题
- **BetterGI** 的 `PostMessage` 后台操作是最大的移植障碍 — Android 上没有窗口消息机制，必须走 VirtualDisplay 前台模式
- 两者都可以参考 MAA-Meow 的 **VirtualDisplay + Shizuku + NDK bridge** 作为 Android 基础设施层
