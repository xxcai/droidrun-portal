# FLAG_REQUEST_2_FINGER_PASSTHROUGH

本文档描述 `FLAG_REQUEST_2_FINGER_PASSTHROUGH` 标志的作用和使用方法。

## 概述

`FLAG_REQUEST_2_FINGER_PASSTHROUGH` 是 Android 14 (API 34) 引入的无障碍服务标志，允许无障碍服务在用户进行双指触摸操作时仍然可以正常截屏。

## 作用

当用户进行双指操作时（如双指缩放、滚动），系统通常会阻止截屏，以避免截到"不完整的交互状态"。启用此标志后，即使用户正在进行双指操作，截屏功能也能正常工作。

## 版本要求

| 版本 | API Level | 支持情况 |
|------|-----------|----------|
| Android 14 | 34 | ✅ 支持 |
| Android 13 | 33 | ❌ 不支持（会被忽略） |
| Android 12 | 31 | ❌ 不支持（会被忽略） |

## 代码示例

### Kotlin

```kotlin
class DroidrunAccessibilityService : AccessibilityService() {

    override fun onServiceConnected() {
        val info = AccessibilityServiceInfo().apply {
            feedbackType = AccessibilityServiceInfo.FEEDBACK_GENERIC

            flags = AccessibilityServiceInfo.FLAG_REPORT_VIEW_IDS or
                    AccessibilityServiceInfo.FLAG_RETRIEVE_INTERACTIVE_WINDOWS or
                    AccessibilityServiceInfo.FLAG_REQUEST_TOUCH_EXPLORATION_MODE

            // API 34+：启用双指穿透截屏
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
                flags = flags or AccessibilityServiceInfo.FLAG_REQUEST_2_FINGER_PASSTHROUGH
            }
        }
        accessibilityServiceInfo = info
    }
}
```

## 使用场景

### 1. 录制手势操作

当需要录制用户的双指缩放、滚动等操作时，启用此标志可以确保截屏不会中断。

```kotlin
// 录制缩放手势
fun recordZoomGesture() {
    // 启用双指穿透
    enable2FingerPassthrough()

    // 执行双指缩放操作
    performZoomGesture()

    // 截取操作过程
    captureScreenshot()
}
```

### 2. 自动化测试

在自动化测试中模拟双指操作时，需要确保能正常截屏记录测试过程。

### 3. 远程协助

在远程控制场景中，用户可能同时进行触摸操作，需要实时截屏。

## 与截图 API 的关系

```
┌─────────────────────────────────────────────────────────────┐
│                    takeScreenshot()                         │
│                    (API 30+ 支持)                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ FLAG_REQUEST_2_FINGER_PASSTHROUGH
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              用户进行双指触摸操作时的行为                      │
├─────────────────────────────────────────────────────────────┤
│  未设置标志 │  返回错误 ERROR_TAKE_SCREENSHOT_INTERVAL_     │
│            │  TIME_SHORT 或截取不完整的交互状态              │
├─────────────────────────────────────────────────────────────┤
│  已设置标志 │  正常截屏，忽略双指触摸的影响                    │
└─────────────────────────────────────────────────────────────┘
```

## 注意事项

1. **仅 API 34+**：此标志在 API 34 以下版本中不可用，设置后会被系统忽略
2. **非必需**：对于不需要截取双指操作场景的应用，可以不启用此标志
3. **隐私考虑**：启用此标志可能会截取到用户正在进行的敏感操作

## 相关文档

- [Android AccessibilityServiceInfo 官方文档](https://developer.android.com/reference/android/accessibilityservice/AccessibilityServiceInfo)
- [截图流程分析](./01-screenshot-flow.md)
- [takeScreenshot API](./02-takeScreenshot-api.md)
