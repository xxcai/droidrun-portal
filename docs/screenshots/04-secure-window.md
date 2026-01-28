# FLAG_SECURE 截屏限制

本文档描述 Android 中 `FLAG_SECURE` 标志对截屏功能的限制。

## 概述

`FLAG_SECURE` 是 Android 的窗口标志，用于标记包含敏感内容的窗口，防止被截屏或录屏。这是一种安全保护机制，旨在保护用户的隐私和敏感数据。

## 工作原理

当一个窗口设置了 `FLAG_SECURE` 标志时：

```
┌─────────────────────────────────────────────────────────────┐
│                       窗口设置                               │
│               window.addFlags(WindowManager.LayoutParams.FLAG_SECURE) │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    截屏尝试                                  │
├─────────────────────────────────────────────────────────────┤
│  MediaProjection      │  返回错误或黑屏                       │
│  takeScreenshot()     │  返回 ERROR_TAKE_SCREENSHOT_        │
│                       │  SECURE_WINDOW 错误                  │
│  ADB screencap        │  被系统阻止                          │
│  第三方截屏工具        │  被系统阻止                          │
└─────────────────────────────────────────────────────────────┘
```

## 错误处理

Droidrun Portal 中的错误处理：

```kotlin
// DroidrunAccessibilityService.kt:1047-1056
val errorMessage = when (errorCode) {
    ERROR_TAKE_SCREENSHOT_INTERNAL_ERROR -> "Internal error occurred"
    ERROR_TAKE_SCREENSHOT_INTERVAL_TIME_SHORT -> "Screenshot interval too short"
    ERROR_TAKE_SCREENSHOT_INVALID_DISPLAY -> "Invalid display"
    ERROR_TAKE_SCREENSHOT_NO_ACCESSIBILITY_ACCESS -> "No accessibility access"
    ERROR_TAKE_SCREENSHOT_SECURE_WINDOW -> "Secure window cannot be captured"  // ⬅️
    else -> "Unknown error (code: $errorCode)"
}
```

## 常见受限应用

| 应用类型 | 示例 | 原因 |
|----------|------|------|
| 银行类 | 招商银行、中国银行 | 保护交易信息安全 |
| 支付类 | 支付宝、微信支付 | 保护支付密码 |
| 视频类 | Netflix、爱奇艺、优酷 | 版权保护 |
| 通讯类 | 微信、QQ（聊天记录） | 隐私保护 |
| 相册类 | 私密相册 | 隐私保护 |
| 密码输入 | 任何包含密码输入的界面 | 防止密码泄露 |

## 设置 FLAG_SECURE 的方法

### 在 Activity 中

```kotlin
// Kotlin
window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)

// 或
window.setFlags(
    WindowManager.LayoutParams.FLAG_SECURE,
    WindowManager.LayoutParams.FLAG_SECURE
)
```

### 在 Dialog 中

```kotlin
dialog.window?.addFlags(WindowManager.LayoutParams.FLAG_SECURE)
```

### 在自定义 View 中

```kotlin
// 需要通过 Window 设置
activity.window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)
```

## 检查窗口是否受保护

```kotlin
fun isWindowSecure(window: Window): Boolean {
    val params = window.attributes
    return (params.flags and WindowManager.LayoutParams.FLAG_SECURE) != 0
}
```

## 注意事项

1. **无法绕过**：这是 Android 的安全机制，无法通过无障碍服务或其他方式绕过
2. **全局生效**：一旦设置，整个 Activity 窗口都无法截屏
3. **不影响辅助功能**：屏幕阅读器（TalkBack）仍然可以读取内容
4. **用户体验**：截屏失败时应该给用户明确的提示

## 替代方案

如果确实需要截取受保护窗口的内容，可以考虑：

| 方案 | 可行性 | 说明 |
|------|--------|------|
| MediaProjection + 用户授权 | 需要用户手动授权 | 用户可以在通知栏授权截屏 |
| ADB + root | 需要 root 权限 | 系统级截屏，不受窗口标志限制 |
| 镜像投屏 | 可行 | 将屏幕镜像到另一设备后截取 |

## 相关文档

- [Android Window 官方文档](https://developer.android.com/reference/android/view/Window#addFlags(int))
- [截图流程分析](./01-screenshot-flow.md)
- [takeScreenshot API](./02-takeScreenshot-api.md)
