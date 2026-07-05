# 米哈游游戏助手 Android 移植方案

> 基于对 MAA-Meow、BetterGI、March7thAssistant 源码的深入分析

## 一、MAA-Meow：已完成的参考案例

### 1.1 项目概述

[MAA-Meow](https://github.com/Aliothmoon/MAA-Meow) 将 MaaAssistantArknights（明日方舟 PC 助手）成功移植到 Android，以原生 APK 形式运行，无需 PC 或模拟器。

### 1.2 技术栈

| 层面 | 技术 |
|---|---|
| 语言 | Kotlin + C++ (NDK) |
| UI | Jetpack Compose |
| DI | Koin |
| 权限 | Shizuku (免 Root) / Root (launcher.c) |
| 核心集成 | JNA 加载 libMaaCore.so |
| 截屏 | VirtualDisplay + ImageReader |
| 输入注入 | AIDL Remote Service + MotionEvent |
| 帧预览 | NDK bridge (EGL/GLES2) |
| OCR | ncnn (替代 PC 端的 PaddleOCR/FastDeploy) |

### 1.3 架构

```
┌─────────────────────────────────────────┐
│  App 进程 (Kotlin/Compose)               │
│  - 主界面、配置面板、日志、定时任务       │
│  - 通过 AIDL 与 Remote 进程通信          │
└──────────────┬──────────────────────────┘
               │ AIDL (RemoteService.aidl)
               │
┌──────────────▼──────────────────────────┐
│  Remote 进程 (以 Shell UID 运行)          │
│                                          │
│  MaaCoreService (JNA → libMaaCore.so)    │
│  ├─ AsstCreate → AsstAppendTask → Start  │
│  ├─ 回调分发: TaskChain / SubTask        │
│                                          │
│  VirtualDisplayManager                    │
│  ├─ 创建隐藏虚拟显示器                    │
│  ├─ 游戏渲染到虚拟 Surface               │
│  └─ ImageReader 捕获帧数据               │
│                                          │
│  ScreenManager                            │
│  ├─ setForcedDisplaySize (分辨率调整)     │
│  └─ WindowManager 接口 (需 System 权限)   │
│                                          │
│  InputControlUtils                        │
│  ├─ touchDown / touchMove / touchUp      │
│  └─ 基于 Android InputManager            │
└─────────────────────────────────────────┘
```

### 1.4 关键移植策略

#### 权限获取

**Shizuku 模式（推荐，免 Root）**：通过 Shizuku 服务以 ADB 权限运行 Remote 进程，获取 Shell UID (2000)。App 进程通过 AIDL 绑定 Remote Service。

**Root 模式**：`launcher.c` 以 Root 身份 fork 子进程，`setresgid/setresuid` 降权到 Shell UID，然后 `execv(/system/bin/app_process)` 启动 Java 服务。

#### 后台运行

`VirtualDisplayManager` 创建 `VIRTUAL_DISPLAY_FLAG_PUBLIC` 虚拟显示器。游戏启动到该虚拟显示器上，通过 `ImageReader` 获取帧数据。App 可在前台显示预览浮窗，或纯后台运行。

#### MAA Core 集成

1. `setup_maa_core.py` 从 MAA GitHub Release 下载 Android 预编译产物（arm64-v8a / x86_64 .so + 资源文件）
2. 部署到 `jniLibs/` (原生库) 和 `assets/MaaSync/MaaResource` (任务资源)
3. JNA 接口 `MaaCoreLibrary.java` 定义了 `AsstCreate/AsstAppendTask/AsstStart` 等 C API
4. OCR 从 PC 端的 PaddleOCR (FastDeploy) 替换为 ncnn 后端

#### Native Bridge

NDK 编译 `libbridge.so`：
- `bridge_frame_buffer.cpp`：VirtualDisplay 帧缓冲捕获，RGBA → Bitmap
- `bridge_preview.cpp`：EGL/GLES2 预览 Surface 管理
- `bridge_input.cpp`：触控事件注入（MotionEvent 构造）
- `bridge_capture.cpp`：屏幕截图工具

### 1.5 可复用的基础设施

MAA-Meow 的以下模块具有通用性，可被其他项目复用：

| 模块 | 路径 | 复用价值 |
|---|---|---|
| Shizuku/Root 权限管理 | `manager/ShizukuManager.kt`, `RootManager.kt` | 任何需要系统权限的 Android 自动化项目 |
| VirtualDisplay 管理 | `remote/internal/VirtualDisplayManager.kt` | 后台游戏运行/截屏 |
| 帧捕获 (NDK) | `app/src/main/native/bridge_frame_buffer.cpp` | 高性能帧数据获取 |
| 触控注入 (NDK) | `app/src/main/native/bridge_input.cpp` | 游戏触控操作模拟 |
| AIDL Remote Service | `app/src/main/aidl/RemoteService.aidl` | 跨进程自动化服务框架 |
| 定时任务 | `schedule/` | 挂机自动化 |

---

## 二、BetterGI 移植方案

### 2.1 当前架构分析

[BetterGI](https://github.com/babalae/better-genshin-impact) 是原神的 PC 辅助工具，**深度绑定 Windows**：

```
BetterGenshinImpact.csproj
  TargetFramework: net8.0-windows10.0.22621.0
  UseWPF: true
  UseWindowsForms: true
```

| 模块 | 实现 | Windows 依赖 |
|---|---|---|
| UI 框架 | WPF + WPF-UI | 100% |
| 截屏 | Fischless.GameCapture (BitBlt / DXGI / Graphics.Capture) | 100% Win32/DirectX |
| 输入模拟 | Fischless.WindowsInput (SendInput / PostMessage) | 100% Win32 |
| 键鼠监听 | MouseKeyHook + SharpDX.DirectInput | 100% Win32 |
| AI 推理 | ONNX Runtime DirectML / TorchSharp / YoloSharp | DirectML = Windows GPU |
| 硬件监控 | LibreHardwareMonitorLib | Windows |
| 脚本引擎 | Microsoft.ClearScript.V8 | 跨平台可行 |
| 图像识别 | OpenCvSharp4 | 跨平台可行 |

### 2.2 移植难度评估：极高

BetterGI 的 Windows 耦合度远超 MAA。MAA 核心是纯 C++，仅在 Controller 层做平台适配；而 BetterGI 从 UI 到截屏到输入，**每一层都是 Win32 API**。

### 2.3 推荐方案：Android 原生重写 + 复用算法

#### 整体架构

```
┌──────────────────────────────────────────────┐
│  Android App (Kotlin/Compose)                 │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ UI 层 (Jetpack Compose)                 │  │
│  │ - 任务配置、识别结果预览、日志           │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ 任务调度层                              │  │
│  │ - 一条龙、自动秘境、自动采集等任务编排   │  │
│  │ - 复用 BetterGI 的 Task/Script 逻辑     │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ 识别层 (OpenCV + ONNX/ncnn)             │  │
│  │ - 模板匹配 (拾取/剧情/按钮识别)         │  │
│  │ - 小地图识别 (采集路线导航)              │  │
│  │ - OCR (文字识别)                        │  │
│  │ - YOLO 目标检测 (敌人/宝箱)             │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ 平台层 (复用 MAA-Meow 基础设施)          │  │
│  │ - Shizuku/Root 权限管理                  │  │
│  │ - VirtualDisplay 截屏                    │  │
│  │ - MotionEvent 输入注入                   │  │
│  └─────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

#### 模块替换清单

| BetterGI 模块 | Android 替代方案 | 工作量 |
|---|---|---|
| WPF UI | Jetpack Compose | 大（全部重写） |
| BitBlt/DXGI 截屏 | VirtualDisplay + ImageReader | 中（复用 MAA-Meow） |
| SendInput 输入 | MotionEvent 注入 (Shizuku/Root) | 中（复用 MAA-Meow） |
| PostMessage 输入 | 不适用，Android 无窗口消息机制 | N/A |
| MouseKeyHook 键鼠监听 | Accessibility Service / 手势录制 | 小 |
| ONNX DirectML | ONNX Runtime Android (NNAPI) 或 ncnn | 小（API 兼容） |
| YoloSharp | YoloV8 ncnn Android | 中 |
| TorchSharp | 不适用，用 ncnn/ONNX 替代 | N/A |
| OpenCvSharp | OpenCV Android SDK | 小（API 几乎一致） |
| ClearScript.V8 | V8 Android / QuickJS | 中 |
| PresentMon FPS | 不需要（Android 无需 FPS 监控） | N/A |

#### 分阶段实施路径

**Phase 1（2-3 个月）：基础设施**
1. 搭建 Kotlin/Compose Android 项目骨架
2. 集成 Shizuku 权限管理（参考 MAA-Meow）
3. 集成 VirtualDisplay 截屏（移植 MAA-Meow bridge）
4. 集成 MotionEvent 输入注入（移植 MAA-Meow bridge）
5. 集成 OpenCV Android + ncnn

**Phase 2（3-4 个月）：核心识别**
1. 移植模板匹配逻辑（自动拾取/剧情跳过/按钮识别）
2. 移植小地图识别（采集路线导航）
3. 移植 OCR 文字识别
4. 训练/移植 YOLO 模型用于原神目标检测

**Phase 3（2-3 个月）：任务功能**
1. 自动秘境（战斗 + 领奖循环）
2. 一条龙（日常委托 + 领奖）
3. 自动采集/锄地（小地图导航）
4. 自动拾取 / 自动剧情

**Phase 4（1-2 个月）：打磨**
1. 定时任务
2. 消息推送
3. 自动更新
4. 性能优化与测试

### 2.4 核心挑战

1. **原神 3D 场景的复杂度**：与明日方舟的 2D 界面不同，原神需要实时处理 3D 画面，识别精度要求更高
2. **战斗 AI 移植**：BetterGI 的自动战斗逻辑依赖精确的键鼠操作序列，触屏环境需要完全重新设计交互
3. **小地图导航精度**：VirtualDisplay 截图下的小地图识别需要验证分辨率和颜色一致性
4. **设备性能要求**：原神 + 图像识别对手机 CPU/GPU 压力大，需要优化推理效率

---

## 三、March7thAssistant 移植方案

### 3.1 当前架构分析

[March7thAssistant](https://github.com/moesnow/March7thAssistant) 是星穹铁道的 PC 辅助工具，技术栈相比 BetterGI 更灵活：

```python
# 核心依赖
pyautogui       # 输入模拟
mss             # 截屏
opencv-python   # 图像识别
RapidOCR        # OCR (ONNX Runtime / OpenVINO)
PySide6         # GUI
selenium        # 云游戏模式
```

| 特征 | 说明 |
|---|---|
| 平台耦合度 | 中等（pyautogui/mss 仅 Windows，但可替换） |
| 已有云游戏模式 | `CloudGameController` 通过 Selenium + CDP 控制浏览器 |
| 输入接口抽象 | `InputBase` 抽象类，已有 `LocalInput` / `CdpInput` 两种实现 |
| 游戏控制器抽象 | `GameControllerBase` 抽象类，已有 `LocalGameController` / `CloudGameController` |

### 3.2 移植难度评估：中等

March7thAssistant 的**接口抽象设计**是关键优势——项目已经将截屏、输入、游戏管理抽象为接口层，支持本地和云游戏两种模式。这意味着移植工作量远小于 BetterGI。

### 3.3 推荐方案：云游戏优先 + 本地可选

#### 方案 A：云游戏模式（推荐，工作量最小）

利用 March7thAssistant 已有的 `CloudGameController` + `CdpInput` 架构：

```
┌──────────────────────────────────────────────┐
│  Android App (Kotlin/Compose 或 WebView)      │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ UI 层                                   │  │
│  │ - 任务选择、进度显示、设置               │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ 任务调度层 (移植 Python 逻辑)           │  │
│  │ - 清体力 / 每日实训 / 周常 / 委托       │  │
│  │ - 复用 Python 的 task 定义              │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ 识别层                                  │  │
│  │ - OpenCV 模板匹配 (Android SDK)         │  │
│  │ - RapidOCR → ONNX Runtime Android       │  │
│  │ - 截图通过 CDP Page.captureScreenshot   │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ 云游戏层                                │  │
│  │ - WebView 加载云星穹铁道页面             │  │
│  │ - 或 Chrome Remote Debugging + CDP      │  │
│  │ - CDP Input: dispatchMouseEvent/KeyEvent│  │
│  └─────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

**优势**：无需本地安装星穹铁道，绕过截屏和输入注入难题，设备性能要求低。

**挑战**：网络延迟影响识别精度；云游戏画质有限；需要处理登录态。

#### 方案 B：本地原生模式（工作量较大，体验更好）

参考 MAA-Meow 架构：

```
┌──────────────────────────────────────────────┐
│  Android App (Kotlin/Compose)                 │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ 平台层 (复用 MAA-Meow 基础设施)          │  │
│  │ - Shizuku/Root 权限                     │  │
│  │ - VirtualDisplay 截屏                   │  │
│  │ - MotionEvent 输入注入                  │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ 识别层                                  │  │
│  │ - OpenCV Android (模板匹配)             │  │
│  │ - RapidOCR → ncnn 或 ONNX Android      │  │
│  │ - Screen (界面状态机) 移植              │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │ 任务层                                  │  │
│  │ - tasks/ 目录的 Python 逻辑逐模块移植   │  │
│  │ - 或用 Chaquopy 在 Android 运行 Python  │  │
│  └─────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

#### 方案 C：混合模式（推荐的终极方案）

同时支持云游戏和本地游戏：

```
                    ┌──────────────┐
                    │  App UI      │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ 任务调度器   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼                         ▼
    ┌──────────────┐          ┌──────────────┐
    │ 云游戏模式    │          │ 本地游戏模式  │
    │ (WebView+CDP) │          │ (VirtualDisplay│
    │              │          │  +InputInject) │
    └──────────────┘          └──────────────┘
              │                         │
              └────────────┬────────────┘
                           ▼
                    ┌──────────────┐
                    │ 共享识别层   │
                    │ OpenCV+OCR   │
                    └──────────────┘
```

### 3.4 Python 代码移植策略

March7thAssistant 的核心逻辑是 Python。有三种处理方式：

| 策略 | 说明 | 适用场景 |
|---|---|---|
| **Kotlin 重写** | 将 Python 逻辑翻译为 Kotlin | 追求性能和原生体验 |
| **Chaquopy 嵌入** | 在 Android App 中嵌入 Python 运行时 | 快速移植，复用现有代码 |
| **远程 Python 服务** | Python 运行在 Termux/服务器，App 通过 API 通信 | 灵活，但依赖额外环境 |

**推荐 Chaquopy 方案**：Chaquopy 是 Android Studio 的 Python 集成插件，可在 Kotlin/Java 中直接调用 Python 模块。这样 `tasks/`、`module/` 下的大量 Python 代码可以**直接复用**，只需要重写 UI 层和平台层。

### 3.5 分阶段实施路径

**Phase 1（1-2 个月）：云游戏 MVP**
1. Kotlin/Compose 搭建 Android App
2. 内嵌 WebView 加载云星穹铁道
3. 实现 CDP 连接 + 输入模拟
4. 截图通过 CDP `Page.captureScreenshot`
5. 基本 OCR + 模板匹配识别
6. 实现"清体力"基本任务

**Phase 2（2-3 个月）：功能完善**
1. 移植全部任务逻辑（daily / weekly / challenge）
2. 集成 RapidOCR Android
3. 通知推送
4. 定时启动

**Phase 3（2-3 个月）：本地模式**
1. 集成 MAA-Meow 风格的 VirtualDisplay + Input Injection
2. 适配本地星穹铁道
3. Chaquopy 嵌入 Python 核心模块
4. 模拟宇宙 / 锄大地子项目评估

### 3.6 子项目移植

March7thAssistant 依赖两个独立子项目：

| 子项目 | 功能 | 移植难度 |
|---|---|---|
| [Auto_Simulated_Universe](https://github.com/CHNZYX/Auto_Simulated_Universe) | 模拟宇宙自动化 | 中（纯 Python + OpenCV） |
| [Fhoe-Rail](https://github.com/linruowuyin/Fhoe-Rail) | 锄大地自动化 | 低（路线数据 + 模板匹配） |

两者都是 Python + OpenCV，可通过 Chaquopy 嵌入或 Kotlin 重写移植。

---

## 四、总结对比

| 维度 | MAA (已移植) | BetterGI | March7thAssistant |
|---|---|---|---|
| 源平台 | C++ 跨平台 | Windows C#/.NET WPF | Windows Python |
| Windows 耦合度 | 低 | **极高** | 中 |
| 移植工作量 | 已完成 | **极大（接近重写）** | **中等** |
| 推荐 Android 方案 | MAA-Meow (已实现) | 原生重写 + 算法复用 | 云游戏优先 + Chaquopy |
| 预估工期 | - | 8-12 个月 | 5-8 个月 |
| 最大瓶颈 | 已解决 | 全栈 Win32 替换 | 云游戏延迟 + Python 嵌入 |
| 可复用代码量 | - | 仅算法层 (~20%) | 任务+识别层 (~60%) |

### 核心建议

1. **提取 MAA-Meow 的平台层为独立库**：Shizuku 权限管理、VirtualDisplay 管理、NDK bridge 是 Android 游戏自动化的通用基础设施
2. **March7thAssistant 优先走云游戏路线**：投入产出比最高，已有 CDP 模式可直接复用
3. **BetterGI 需要做好长期投入的准备**：建议先做一个最小可用版本（自动拾取 + 自动秘境），验证技术可行性后再扩展
4. **考虑统一的 Android 自动化框架**：三个项目的核心需求一致（截屏、识别、输入），可以构建一个通用的 Android Game Automation SDK
