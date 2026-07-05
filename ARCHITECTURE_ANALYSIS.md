# 三大游戏助手架构深度分析

> 截图识别、操控方式、操作判断的源码级对比

## 一、截图识别

### 1.1 MAA (明日方舟)

**截图方式**：
- **PC 端**：通过 ADB 截屏（`adb exec-out screencap -p`）或 adb-lite 自实现的协议直接与设备通信，也支持 Win32 窗口句柄直接截屏（GDI / DXGI Desktop Duplication）
- **Android 端 (MAA-Meow)**：`VirtualDisplay` + `ImageReader`（后台模式，游戏渲染到隐藏的虚拟显示器），或 `MediaProjection`（前台模式，截取主屏幕）

**识别算法**（C++，`src/MaaCore/Vision/`）：

| 算法 | 类 | 用途 |
|---|---|---|
| **模板匹配** | `Matcher` | `cv::matchTemplate` + 颜色掩码 + 阈值过滤。用于识别按钮、图标、界面元素。支持多模板、颜色范围掩码（`mask_ranges`）、纯色模式 |
| **OCR 文字识别** | `OCRer` | PaddleOCR (PC) / ncnn (Android)。支持检测+识别两阶段，结果后处理包括二值化、trim、文本替换（`postproc_replace_`）、正则过滤 |
| **特征匹配** | `FeatureMatcher` | `cv::features2d`（SIFT/ORB 等），用于小地图定位、地图格子识别 |
| **分类器** | `BattlefieldClassifier` / `BattlefieldDetector` | 用于战斗场景中识别干员、击杀数、费用等 |
| **数色** | 内置于 `MatcherConfig` | `color_scales` 参数，统计 ROI 内符合颜色范围的像素比例 |

**任务配置驱动**：所有识别逻辑通过 `tasks.json` 声明式配置，每个任务节点定义 `recognition`（算法类型）、`template`（模板图片）、`roi`（感兴趣区域）、`threshold`（阈值）、`next`（下一步任务）。

### 1.2 BetterGI (原神)

**截图方式**：
- `Fischless.GameCapture` 库提供三种截屏模式：
  - **BitBlt**：`GDI32.BitBlt`，最基础，兼容性好
  - **DwmSharedSurface**：通过 DWM 共享表面截屏
  - **Windows.Graphics.Capture**：`SharpDX.Direct3D11` + WinRT API，支持 HDR→SDR 转换（`HdrToSdrShader`）
- 所有截图通过 `CaptureContent` 统一封装，自动归一化到 1080p 分辨率

**识别算法**（C#，`Core/Recognition/`）：

| 算法 | 类型枚举 | 用途 |
|---|---|---|
| **模板匹配** | `RecognitionTypes.TemplateMatch` | OpenCV `matchTemplate`，用于识别按钮、图标、F键提示等。阈值默认 0.8 |
| **OCR 文字识别** | `RecognitionTypes.Ocr` / `OcrMatch` | PaddleOCR（`PaddleOcrService`，含 Det + Rec 两阶段）或 SVTR ONNX 模型（`PickTextInference`） |
| **颜色匹配** | `RecognitionTypes.ColorMatch` | 检测指定颜色区域（如拾取提示的橙色边框） |
| **颜色+OCR** | `RecognitionTypes.ColorRangeAndOcr` | 先提取指定颜色区域，再做 OCR（提高特定场景精度） |
| **YOLO 目标检测** | `RecognitionTypes.Detect` | `BgiYoloPredictor`（ONNX Runtime + DirectML），用于战斗中识别敌人、角色、宝箱等。自训练 YOLO 模型 |

**实时任务驱动**：`ITaskTrigger` 接口，每帧回调 `OnCapture(CaptureContent)`。例如 `AutoPickTrigger` 在每帧中检测 F 键图标，找到后模拟按键。

### 1.3 March7thAssistant (星穹铁道)

**截图方式**：
- **本地模式**：`mss` 库截屏（跨平台），通过 `pyautogui.getWindowsWithTitle` 定位游戏窗口，计算客户区坐标后截取。支持 `win32gui.GetClientRect` 精确计算
- **云游戏模式**：Selenium CDP `Page.captureScreenshot` 获取浏览器内截图
- **后台截屏**（可选）：`win32gui` + `win32ui` + `PrintWindow` API，支持后台截取不被遮挡

**识别算法**（Python，`module/`）：

| 算法 | 实现 | 用途 |
|---|---|---|
| **模板匹配** | OpenCV `cv2.matchTemplate` | `Automation.find_element(type="image")`，用于识别按钮、图标、界面状态 |
| **OCR 文字识别** | RapidOCR（ONNX Runtime / OpenVINO） | `OCR.recognize_multi_lines()`，用于识别任务文本、进度数字、对话内容。支持周期性 GC 控制内存峰值 |
| **颜色/HSV 匹配** | `find_element(type="hsv")` | 检测特定颜色区域 |
| **像素匹配** | `find_element(type="pixel")` | 精确像素颜色判断 |
| **界面状态机** | `Screen` 类 | 基于 JSON 配置的界面状态图，通过模板匹配判断当前所在界面，支持自动导航 |

---

## 二、操控方式

### 2.1 MAA

**PC 端**：
- **ADB 方式**：`adb shell input tap x y`、`adb shell input swipe`。也有 `adb-lite` 自实现的轻量 ADB 协议客户端，避免依赖完整 adb 工具
- **Win32 方式**：`SendMessage/PostMessage` 发送鼠标键盘消息，支持多种输入方式枚举（`Win32InputMethod`）

**Android 端 (MAA-Meow)**：
- 触控事件通过 AIDL `RemoteService.touchDown/touchMove/touchUp` 注入
- Remote 进程以 Shell UID 运行，拥有足够权限调用 `InputManager.injectInputEvent()`
- NDK 层 `bridge_input.cpp` 构造 `MotionEvent` 并分发
- 支持文本输入（`InputArgs`）和按键事件（`KeyArgs`）

### 2.2 BetterGI

- `Fischless.WindowsInput` 库：封装 Win32 `SendInput` API
  - `Simulation.SendInput.Keyboard.KeyPress(key)` — 键盘按键
  - `Simulation.SendInput.Mouse.LeftButtonClick()` — 鼠标点击
  - `Simulation.SendInput.Mouse.VerticalScroll(delta)` — 鼠标滚轮
  - `Simulation.SendInput.Mouse.MoveMouseTo(x, y)` — 鼠标移动
- `PostMessageSimulator`：通过 `PostMessage` 发送到窗口句柄，支持后台操作
- `MouseEventSimulator`：底层鼠标事件模拟
- 所有操控通过 `Simulation` 静态类统一入口

### 2.3 March7thAssistant

- **本地模式** (`LocalInput`)：`pyautogui` 封装
  - `pyautogui.click(x, y)` — 点击
  - `pyautogui.mouseDown/mouseUp` — 按下/释放
  - `pyautogui.moveTo` — 移动
  - `pyautogui.scroll` — 滚轮
  - `pyautogui.keyDown/keyUp` — 键盘
- **云游戏模式** (`CdpInput`)：通过 CDP 协议发送 `Input.dispatchMouseEvent` / `Input.dispatchKeyEvent`
  - 将 pyautogui 的按键名映射到 CDP 的 key/code/vk 格式
  - 支持 modifier key（Shift/Ctrl/Alt）
- 接口抽象：`InputBase` 定义统一的 `mouse_click/mouse_down/mouse_up/press_key` 等方法，两种模式实现同一接口

---

## 三、操作判断（决策逻辑）

### 3.1 MAA — 声明式任务状态机

MAA 的核心是一套 **JSON 声明式的任务流程引擎** (`ProcessTask`)：

```json
{
  "StageTheme": {
    "recognition": "TemplateMatch",
    "template": "xxx.png",
    "roi": [100, 200, 300, 400],
    "threshold": 0.8,
    "action": "ClickSelf",
    "next": ["NextTask1", "NextTask2"]
  }
}
```

- **流程控制**：`ProcessTask` 按 `next` 列表顺序尝试识别，第一个匹配成功的任务执行其 `action`，然后跳转到该任务的 `next`
- **action 类型**：`ClickSelf`（点击识别区域中心）、`DoNothing`、`Swipe`、`ClickRect` 等
- **特殊值**：`#self`（重新识别自己）、`#next`（执行下一个）、`#back`（返回上一级）
- **业务逻辑**：更复杂的任务如战斗（`BattleHelper`）、基建换班（`InfrastAbstractTask`）通过 C++ 类实现，但底层仍调用 Vision 识别 + Controller 操控

### 3.2 BetterGI — 事件驱动 Trigger + 独立 Task

BetterGI 有两层决策体系：

**实时 Trigger 层**（每帧触发）：
- `ITaskTrigger` 接口，`OnCapture(CaptureContent)` 每帧回调
- `AutoPickTrigger`：每帧检测 F 键图标 → 找到后按 F → 检测滚轮图标则滚轮下滚
- `AutoSkipTrigger`：检测对话界面 → 自动点击推进 → 检测选项则选择 → 检测每日委托奖励则领取
- Trigger 有优先级（`Priority`）和互斥性（`IsExclusive`）

**独立 Task 层**（按需启动）：
- `ISoloTask` 接口，如 `AutoDomainTask`（自动秘境）
- 流程是**硬编码的步骤序列**：走到钥匙处 → 按 F 启动 → 战斗 → 寻找石化古树 → 走到古树 → 领取奖励 → 判断是否继续
- 战斗决策：`CombatScenes` 通过 YOLO 识别队伍角色，支持战斗脚本（`combatScops` JSON 配置），定义角色切换、技能释放顺序

**导航决策**：
- `AutoPathingTask`：基于路线数据的自动导航，通过小地图识别（`cvAutoTrack` 风格的坐标匹配）实现自动跑图
- 路线支持动作处理器：`IActionHandler`（`NormalAttackHandler`、`FishingHandler`、`ElementalCollectHandler` 等）

### 3.3 March7thAssistant — Python 流程编排 + 界面状态机

**界面状态机**：
- `Screen` 类：从 `screens.json` 加载界面定义，每个界面有 `image_path`（识别图片）和 `actions`（可切换的目标界面及操作序列）
- `screen.get_current_screen()`：通过模板匹配判断当前界面
- `screen.switch_to(target_screen)`：自动执行操作序列导航到目标界面

**任务编排**：
- `Daily.prepare_daily()` 为例：按优先级编排多个子任务
  1. 兑换码 → 2. 培养目标初始化 → 3. 活动 → 4. 历战余响 → 5. 清体力 → 6. 每日实训 → 7. 领取奖励
- 每个子任务内部是 Python 顺序逻辑：截图 → OCR/模板匹配 → 判断状态 → 执行操作 → 等待 → 验证结果
- `Instance.run()` 副本循环：准备副本 → 开始 → 等待战斗 → 检查失败/成功 → 再次开始或完成
- `Tasks.detect()` 每日实训：OCR 识别任务列表文本 → 正则匹配进度 `X/Y` → 判断已完成/待完成

**重试与异常**：
- `find_element` 内置 `max_retries` 参数，失败后自动重试
- 战斗失败自动复活重试（递归调用 `Instance.run(from_failure=True)`）

---

## 四、总结对比表

| 维度 | MAA | BetterGI | March7thAssistant |
|---|---|---|---|
| **截图** | ADB / Win32 / VirtualDisplay+ImageReader | BitBlt / DXGI / Graphics.Capture (Win32) | mss (跨平台) / CDP 截图 / PrintWindow |
| **识别算法** | 模板匹配 + OCR + 特征匹配 + 分类器 | 模板匹配 + OCR + YOLO + 颜色匹配 | 模板匹配 + OCR + HSV/像素匹配 |
| **AI 模型** | PaddleOCR / ncnn | PaddleOCR + YOLO (ONNX DirectML) | RapidOCR (ONNX / OpenVINO) |
| **操控** | ADB input / Win32 SendMessage | Win32 SendInput / PostMessage | pyautogui / CDP Input |
| **决策架构** | **JSON 声明式任务状态机** | **事件驱动 Trigger + 硬编码 Task** | **Python 流程编排 + 界面状态机** |
| **配置化程度** | 极高（tasks.json 声明式） | 中等（Trigger 可配置，Task 硬编码） | 中等（screens.json 状态机 + Python 逻辑） |
| **典型决策流程** | 截图→遍历 next 列表→匹配则执行 action→跳转 | 每帧 Trigger 回调→识别→模拟操作；Task 按步骤执行 | 截图→OCR/模板→Python if/else→操作→等待→验证 |
