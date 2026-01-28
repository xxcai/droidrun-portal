# takeScreenshot API

本文档描述 Android AccessibilityService 的 `takeScreenshot()` API。

## API 概述

`takeScreenshot()` 是 Android 无障碍服务提供的截屏 API，允许无障碍服务在不需要 MediaProjection 权限的情况下截取屏幕。

## 版本要求

| 版本 | API Level | 支持情况 |
|------|-----------|----------|
| Android 11 | 30 | ✅ 支持 |
| Android 12 | 31 | ✅ 支持 |
| Android 13 | 33 | ✅ 支持 |
| Android 14+ | 34+ | ✅ 支持 |

## 方法签名

```kotlin
// Android 11+ (API 30+)
public fun takeScreenshot(
    displayId: Int,
    executor: Executor,
    callback: TakeScreenshotCallback
)
```

## 参数说明

| 参数 | 类型 | 描述 |
|------|------|------|
| `displayId` | `Int` | 屏幕 ID，通常使用 `Display.DEFAULT_DISPLAY` |
| `executor` | `Executor` | 回调执行的线程池 |
| `callback` | `TakeScreenshotCallback` | 截图结果回调 |

## 回调接口

```kotlin
interface TakeScreenshotCallback {
    fun onSuccess(result: ScreenshotResult)
    fun onFailure(errorCode: Int)
}
```

## ScreenshotResult

```kotlin
data class ScreenshotResult(
    val hardwareBuffer: HardwareBuffer,
    val colorSpace: ColorSpace
)
```

## 使用示例

### Kotlin

```kotlin
// 在 DroidrunAccessibilityService 中
fun takeScreenshotBase64(hideOverlay: Boolean = true): CompletableFuture<String> {
    val future = CompletableFuture<String>()

    try {
        takeScreenshot(
            Display.DEFAULT_DISPLAY,
            Executors.newSingleThreadExecutor(),
            object : TakeScreenshotCallback {
                override fun onSuccess(result: ScreenshotResult) {
                    try {
                        val bitmap = Bitmap.wrapHardwareBuffer(
                            result.hardwareBuffer,
                            result.colorSpace
                        )

                        if (bitmap == null) {
                            result.hardwareBuffer.close()
                            future.complete("error: Failed to create bitmap")
                            return
                        }

                        // 压缩为 PNG
                        val outputStream = ByteArrayOutputStream()
                        bitmap.compress(Bitmap.CompressFormat.PNG, 100, outputStream)

                        // Base64 编码
                        val base64 = Base64.encodeToString(outputStream.toByteArray(), Base64.NO_WRAP)

                        bitmap.recycle()
                        result.hardwareBuffer.close()
                        future.complete(base64)
                    } catch (e: Exception) {
                        future.complete("error: ${e.message}")
                    }
                }

                override fun onFailure(errorCode: Int) {
                    val errorMessage = when (errorCode) {
                        ERROR_TAKE_SCREENSHOT_INTERNAL_ERROR -> "Internal error"
                        ERROR_TAKE_SCREENSHOT_INTERVAL_TIME_SHORT -> "Interval too short"
                        ERROR_TAKE_SCREENSHOT_INVALID_DISPLAY -> "Invalid display"
                        ERROR_TAKE_SCREENSHOT_NO_ACCESSIBILITY_ACCESS -> "No accessibility access"
                        ERROR_TAKE_SCREENSHOT_SECURE_WINDOW -> "Secure window cannot be captured"
                        else -> "Unknown error (code: $errorCode)"
                    }
                    future.complete("error: $errorMessage")
                }
            }
        )
    } catch (e: Exception) {
        future.complete("error: ${e.message}")
    }

    return future
}
```

### HTTP API

```bash
# 获取截图（Base64 编码）
curl -H "Authorization: Bearer TOKEN" "http://localhost:8080/screenshot"

# 获取截图（隐藏覆盖层）
curl -H "Authorization: Bearer TOKEN" "http://localhost:8080/screenshot?hideOverlay=true"

# 获取截图（不隐藏覆盖层）
curl -H "Authorization: Bearer TOKEN" "http://localhost:8080/screenshot?hideOverlay=false"
```

## 错误码

| 错误码 | 常量 | 描述 |
|--------|------|------|
| 1 | `ERROR_TAKE_SCREENSHOT_INTERNAL_ERROR` | 内部错误 |
| 2 | `ERROR_TAKE_SCREENSHOT_INTERVAL_TIME_SHORT` | 截屏间隔太短 |
| 3 | `ERROR_TAKE_SCREENSHOT_INVALID_DISPLAY` | 无效的屏幕 |
| 4 | `ERROR_TAKE_SCREENSHOT_NO_ACCESSIBILITY_ACCESS` | 无障碍服务未获得截屏权限 |
| 5 | `ERROR_TAKE_SCREENSHOT_SECURE_WINDOW` | 无法截取安全窗口 |

## 权限要求

### 服务配置

在无障碍服务的 XML 配置中需要设置：

```xml
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityFlags="flagReportViewIds|flagRetriveInteractiveWindows"
    ... />
```

### AndroidManifest

无需额外权限，依赖无障碍服务权限。

## 限制

1. **FLAG_SECURE**：设置了 `FLAG_SECURE` 的窗口无法截取
2. **API 级别**：需要 Android 11+ (API 30+)
3. **返回格式**：返回 HardwareBuffer，需要转换为 Bitmap
4. **资源管理**：使用完毕后必须关闭 HardwareBuffer

## 相关文档

- [Android AccessibilityService 官方文档](https://developer.android.com/reference/android/accessibilityservice/AccessibilityService#takeScreenshot(int,java.util.concurrent.Executor,android.accessibilityservice.AccessibilityService.TakeScreenshotCallback))
- [截图流程分析](./01-screenshot-flow.md)
- [FLAG_SECURE 限制](./04-secure-window.md)
