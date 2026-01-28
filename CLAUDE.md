# CLAUDE.md

本文档为 Claude Code 提供处理此代码库的指导。

## 项目概述

**Droidrun Portal** 是一个使用 **Kotlin** 编写的 **Android 原生应用**。它是一个无障碍服务应用，提供以下功能：
- 实时视觉反馈和 Android 屏幕 UI 元素的数据采集
- 交互式覆盖层，高亮显示可点击、可检查、可编辑、可滚动和可聚焦的元素
- 通过多种接口实现远程控制和自动化

**关键技术细节：**
- **语言：** Kotlin
- **框架：** Android SDK (minSdk 30 / Android 11.0, targetSdk 34)
- **构建系统：** Gradle (Kotlin DSL)
- **版本：** 0.5.3 (versionCode 53)

## 项目结构

```
app/src/main/java/com/droidrun/portal/
├── api/                    - API 请求处理和响应处理
├── config/                 - 配置管理 (SharedPreferences 包装器)
├── core/                   - 核心逻辑：无障碍树构建、状态仓库
├── events/                 - 事件系统 (EventHub、WebSocket 服务器、事件模型)
├── input/                  - 自定义键盘 IME (DroidrunKeyboardIME)
├── model/                  - 数据模型：ElementNode、PhoneState
├── service/                - Android 服务
│   ├── DroidrunAccessibilityService.kt    - 核心无障碍服务
│   ├── SocketServer.kt                    - HTTP REST API (端口 8080)
│   ├── PortalWebSocketServer.kt           - WebSocket 事件 (端口 8081)
│   ├── ReverseConnectionService.kt        - 云端 WebSocket 连接
│   ├── ScreenCaptureService.kt            - WebRTC 屏幕投射
│   ├── DroidrunNotificationListener.kt    - 通知事件
│   └── DroidrunContentProvider.kt         - 基于 ADB 的数据访问
├── state/                  - 连接状态和应用可见性状态管理
├── streaming/              - WebRTC 屏幕投射 (WebRtcManager、ScrcpyControlChannel)
└── ui/                     - 界面活动：MainActivity、SettingsActivity 等
```

## 关键组件

### 主入口点
| 组件 | 文件 | 用途 |
|------|------|------|
| Main Activity | `ui/MainActivity.kt` | 应用启动器，显示认证令牌、连接状态、设置 |
| Accessibility Service | `service/DroidrunAccessibilityService.kt` | 核心服务，监控 UI 元素并绘制覆盖层 |
| Settings Activity | `ui/settings/SettingsActivity.kt` | 配置界面 |
| ContentProvider | `service/DroidrunContentProvider.kt` | ADB 可访问的数据提供程序 |

### 服务层
- **SocketServer** - 端口 8080 上的 HTTP REST API
- **PortalWebSocketServer** - 端口 8081 上的 WebSocket 事件
- **ReverseConnectionService** - 通过出站 WebSocket 连接实现云端控制
- **ScreenCaptureService** - 支持自动接受 MediaProjection 的 WebRTC 屏幕投射
- **DroidrunNotificationListener** - 支持逐事件切换的通知事件流
- **DroidrunContentProvider** - 基于 ADB 的数据访问

## 关键功能
1. 在可操作的 UI 元素上绘制交互式覆盖层
2. 本地 API：HTTP socket server、WebSocket server、ContentProvider
3. 用于云端控制的反向 WebSocket 连接
4. 支持自动接受的 WebRTC 屏幕投射
5. 通过 URL 安装 APK，支持自动接受
6. 通知事件流
7. 用于文本输入的自定义键盘 IME

## 构建配置
- **构建系统：** Gradle (Kotlin DSL)
- **文件：**
  - `/build.gradle.kts` - 根构建文件
  - `/app/build.gradle.kts` - 应用模块构建文件
  - `/gradle/libs.versions.toml` - 依赖版本目录

### 依赖
| 依赖 | 版本 | 用途 |
|------|------|------|
| AndroidX Core KTX | 1.10.1 | Android Kotlin 扩展 |
| Material Design | 1.10.0 | UI 组件 |
| Java-WebSocket | 1.6.0 | WebSocket 实现 |
| WebRTC | 137.7151.05 | 屏幕投射 |
| JUnit | 4.13.2 | 单元测试 |
| MockK | 1.13.12 | 测试模拟 |
| JSON | 20240303 | JSON 处理 |

## 测试
- **单元测试：** `app/src/test/`
- **仪表化测试：** `app/src/androidTest/`

## 命令
```bash
# 构建项目
./gradlew assembleDebug

# 运行单元测试
./gradlew test

# 运行 lint
./gradlew lint

# 构建发布版本
./gradlew assembleRelease
```

## 架构图

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                      外部系统                                                    │
│                          (ADB / HTTP 客户端 / WebSocket 客户端)                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          │ ADB / HTTP / WebSocket
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     外部接口层                                                    │
├─────────────────────────────┬─────────────────────────────┬─────────────────────────────────────┤
│    ContentProvider          │     SocketServer            │     PortalWebSocketServer           │
│    (ADB 命令)               │     (HTTP REST :8080)       │     (WebSocket :8081)               │
│    └─► Uri.parse()          │    └─► GET/POST             │    └─► JSON-RPC                     │
└──────────────┬──────────────┴──────────────┬──────────────┴────────────────┬────────────────────┘
               │                             │                               │
               ▼                             ▼                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   ActionDispatcher                                              │
│                     (统一命令分发：点击、滑动手势、输入、安装、流)                                │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
┌───────────────────────────┐ ┌───────────────────────┐ ┌───────────────────────────────────────┐
│      ApiHandler           │ │   ReverseConnection   │ │            EventHub                    │
│   (API 业务逻辑)          │ │   Service             │ │         (事件广播)                     │
│                           │ │   (云端连接)          │ │                                       │
│  ┌───────────────────┐   │ │                       │ │  ┌─────────────────────────────────┐   │
│  │ StateRepository   │   │ │  ┌───────────────┐    │ │  │ PortalWebSocketServer (订阅)   │   │
│  │ (状态访问)        │   │ │  │WebSocketClient│    │ │  │ ReverseConnectionService       │   │
│  └───────────────────┘   │ │  └───────┬───────┘    │ │  └─────────────────────────────────┘   │
└───────────────────────────┘ │          │            │ │           ▲                           │
                             │          ▼            │ │           │ emit()                     │
                             │   ┌──────────────┐    │ │  ┌────────┴────────┐                   │
                             │   │ 云端服务器    │    │ │  │ NotificationListener              │
                             │   │(Mobilerun)   │    │ │  │ (通知事件)                          │
                             │   └──────────────┘    │ │  └───────────────┘                   │
                             └───────────────────────┘ └───────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                          DroidrunAccessibilityService (核心)                                    │
│                     (Android 无障碍服务 - 系统级 UI 监控)                                        │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                  │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐ │
│  │ OverlayManager     │  │ StateRepository    │  │ EventHub           │  │ ScreenCapture      │ │
│  │ (UI 覆盖层)        │  │ (无障碍树/状态)    │  │ (事件发射)         │  │ (WebRTC 投射)      │ │
│  └────────────────────┘  └────────────────────┘  └────────────────────┘  └────────────────────┘ │
│                                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                    AccessibilityNodeInfo (系统 UI 树)                                   │    │
│  │  定期刷新 (~60FPS) → findAllVisibleElements() → ElementNode → 覆盖层绘制               │    │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                         系统交互层                                                       │    │
│  │  performAction(click), inputText(), takeScreenshot(), GestureController               │    │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       Android 系统                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Accessibility│  │  WindowManager│ │  PackageManager│ │ InputMethod │  │ MediaProjection│    │
│  │  Service     │  │  (覆盖层)    │  │  (APK 安装)   │ │  (键盘)     │  │  (屏幕录制)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 数据流

### 1. 命令执行流程
```
外部请求 → SocketServer / ContentProvider / WebSocket
        → ActionDispatcher.dispatch(method, params)
        → ApiHandler (业务逻辑)
        → DroidrunAccessibilityService.performAction()
        → Android 系统执行操作
```

### 2. UI 元素采集流程
```
AccessibilityService (监听 TYPE_WINDOW_STATE_CHANGED 事件)
        → 定期刷新 (250ms 间隔)
        → rootInActiveWindow
        → findAllVisibleElements() - 递归遍历
        → ElementNode (包含位置、文本、类型)
        → OverlayManager 绘制高亮框
        → StateRepository 缓存供 API 查询
```

### 3. 事件广播流程
```
系统事件 (通知、UI 变化)
        → DroidrunNotificationListener / AccessibilityService
        → EventHub.emit(PortalEvent)
        → PortalWebSocketServer.broadcast() → 本地客户端
        → ReverseConnectionService.send() → 云端服务器
```

### 4. 云端反向连接流程
```
用户点击连接 → ReverseConnectionService.start()
        → WebSocketClient.connect(wss://...)
        → onMessage() 接收 JSON-RPC 命令
        → ActionDispatcher.dispatch()
        → 执行操作并返回结果
        → 断线自动重连
```

### 5. 屏幕投射流程
```
stream/start → ApiHandler.startStream()
        → ScreenCaptureActivity (请求 MediaProjection 权限)
        → ScreenCaptureService → WebRtcManager
        → WebRTC 视频流 → 云端/客户端
```

## 重要注意事项
- 忽略 `build/`、`app/build/` 和其他编译器输出目录
- 应用需要 Android 11+ (API 30+) 才能运行
- 无障碍服务必须启用才能使用核心功能
- 屏幕投射需要 MediaProjection 权限
