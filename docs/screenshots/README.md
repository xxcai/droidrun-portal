# 截图功能

本文档描述 Droidrun Portal 的截图功能相关文档。

## 文档索引

| 文档 | 描述 |
|------|------|
| [01-screenshot-flow.md](./01-screenshot-flow.md) | 截图完整流程分析 |
| [02-takeScreenshot-api.md](./02-takeScreenshot-api.md) | takeScreenshot API 说明 |
| [03-2-finger-passthrough.md](./03-2-finger-passthrough.md) | 双指穿透标志说明 |
| [04-secure-window.md](./04-secure-window.md) | FLAG_SECURE 截屏限制 |

## 快速使用

```bash
# 获取截图（Base64 编码）
curl -H "Authorization: Bearer TOKEN" "http://localhost:8080/screenshot?hideOverlay=true"

# 获取截图（二进制 PNG）- WebSocket 端点
ws://localhost:8081?token=TOKEN
```

## API 列表

| 端点 | 方法 | 描述 |
|------|------|------|
| `/screenshot` | GET | 获取当前屏幕截图 |
| `/screenshot?hideOverlay=false` | GET | 获取截图（不隐藏覆盖层） |

## 相关文档

- [本地 API 文档](../local-api.md)
- [HTTP API 代码示例](../http-api-code-examples.md)
