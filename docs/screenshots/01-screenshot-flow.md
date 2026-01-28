# 截图流程分析

本文档详细描述 Droidrun Portal 截图功能的数据流和时序。

## 数据流图

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     外部请求                                                  │
│                         GET /screenshot?hideOverlay=true                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     SocketServer                                             │
│                              (handleGetRequest → dispatch)                                   │
│                                                                                              │
│  文件：service/SocketServer.kt:177                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  ActionDispatcher                                            │
│                                    "screenshot"                                              │
│                                                                                              │
│  文件：service/ActionDispatcher.kt:98                                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     ApiHandler                                                │
│                                  getScreenshot(hideOverlay)                                  │
│                                                                                              │
│  1. 调用 stateRepo.takeScreenshot()                                                          │
│  2. 等待 future.get(5秒超时)                                                                 │
│  3. 返回 ApiResponse.Text(Base64)                                                            │
│                                                                                              │
│  文件：api/ApiHandler.kt:418                                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  StateRepository                                             │
│                               takeScreenshot(hideOverlay)                                    │
│                                                                                              │
│  文件：core/StateRepository.kt:31                                                            │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                          DroidrunAccessibilityService                                        │
│                                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐    │
│   │                           takeScreenshotBase64(hideOverlay)                         │    │
│   │   ┌─────────────────────────────────────────────────────────────────────────────┐  │    │
│   │   │  1. 如果 hideOverlay=true，临时禁用覆盖层绘制                                  │  │    │
│   │   │  2. 延迟 100ms 确保覆盖层隐藏                                                  │  │    │
│   │   │  3. 调用 performScreenshotCapture()                                           │  │    │
│   │   └─────────────────────────────────────────────────────────────────────────────┘  │    │
│   │                                                                                      │    │
│   │   ┌─────────────────────────────────────────────────────────────────────────────┐  │    │
│   │   │                    performScreenshotCapture()                               │  │    │
│   │   │                                                                              │  │    │
│   │   │  1. 调用 AccessibilityService.takeScreenshot()                               │  │    │
│   │   │     - 需要 FLAG_RETRIEVE_INTERACTIVE_WINDOWS 标志                            │  │    │
│   │   │                                                                              │  │    │
│   │   │  2. TakeScreenshotCallback.onSuccess()                                       │  │    │
│   │   │     - 接收 ScreenshotResult (HardwareBuffer + ColorSpace)                    │  │    │
│   │   │     - Bitmap.wrapHardwareBuffer() → Bitmap                                    │  │    │
│   │   │     - bitmap.compress(PNG) → ByteArray                                        │  │    │
│   │   │     - Base64.encodeToString() → Base64 字符串                                 │  │    │
│   │   │     - future.complete(base64String)                                           │  │    │
│   │   │                                                                              │  │    │
│   │   │  3. 清理资源                                                                  │  │    │
│   │   │     - bitmap.recycle()                                                        │  │    │
│   │   │     - hardwareBuffer.close()                                                  │  │    │
│   │   │                                                                              │  │    │
│   │   │  4. 恢复覆盖层绘制（如果之前禁用了）                                            │  │    │
│   │   └─────────────────────────────────────────────────────────────────────────────┘  │    │
│   └─────────────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                              │
│  文件：service/DroidrunAccessibilityService.kt:942                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     Android 系统                                              │
│                            AccessibilityService.takeScreenshot()                            │
│                                                                                              │
│  要求：Android 11+ (API 30+)                                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 时序图

```
客户端                    SocketServer          ActionDispatcher     ApiHandler      DroidrunAccessibilityService    Android系统
  │                           │                      │                  │                        │                        │
  ├─GET /screenshot─────────>│                      │                  │                        │                        │
  │                          ├─dispatch────────────>│                  │                        │                        │
  │                          │                      ├─getScreenshot───>│                        │                        │
  │                          │                      │                  ├─takeScreenshot───────>│                        │
  │                          │                      │                  │                        │                        │
  │                          │                      │                  │                        ├─takeScreenshot()─────>│
  │                          │                      │                  │                        │                        │
  │                          │                      │                  │                        │   onSuccess()          │
  │                          │                      │                  │                        │<───────────────────────│
  │                          │                      │                  │                        │                        │
  │                          │                      │                  │   future.complete()    │                        │
  │                          │                      │                  │<───────────────────────│                        │
  │                          │                      │   future.get()   │                        │                        │
  │                          │                      │<─────────────────│                        │                        │
  │                          │                      │                  │                        │                        │
  │                          │   Text(Base64)       │                  │                        │                        │
  │                          │<─────────────────────│                  │                        │                        │
  │                          │                      │                  │                        │                        │
  │<─HTTP 200 + Base64──────│                      │                  │                        │                        │
```

## 关键代码路径

| 层级 | 文件 | 关键方法 |
|------|------|----------|
| HTTP 层 | `service/SocketServer.kt:177` | `handleGetRequest` → `/screenshot` |
| 分发层 | `service/ActionDispatcher.kt:98` | `dispatch("screenshot", ...)` |
| API 层 | `api/ApiHandler.kt:418` | `getScreenshot(hideOverlay)` |
| 状态层 | `core/StateRepository.kt:31` | `takeScreenshot(hideOverlay)` |
| 核心层 | `service/DroidrunAccessibilityService.kt:942` | `takeScreenshotBase64(hideOverlay)` |
| 核心层 | `service/DroidrunAccessibilityService.kt:976` | `performScreenshotCapture()` |
| 回调层 | `service/DroidrunAccessibilityService.kt:988` | `TakeScreenshotCallback.onSuccess()` |

## 当前问题

代码注释中已指出：

```kotlin
// ApiHandler.kt:428-431
// Result is Base64 string from Service.
// decode it back to bytes to pass as Binary response.
// In future, Service should return bytes directly to avoid this encode/decode cycle.
```

| 当前实现 | 理想实现 |
|----------|----------|
| Service 返回 Base64 | Service 返回 ByteArray |
| ApiHandler 返回 Text(Base64) | ApiHandler 返回 Binary |
| SocketServer 返回 HTTP Text | SocketServer 返回 HTTP Binary (PNG) |

## 注意事项

1. **API 级别**：需要 Android 11+ (API 30+)
2. **权限**：不需要额外权限，但需要无障碍服务已启用
3. **超时**：API 调用有 5 秒超时限制
4. **覆盖层**：可通过 `hideOverlay` 参数控制是否隐藏覆盖层
5. **FLAG_SECURE**：设置了 FLAG_SECURE 的窗口无法截取
