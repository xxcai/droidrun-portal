# HTTP API 代码示例

本文档提供 Python 和 JavaScript 代码示例，帮助您快速集成 Droidrun Portal 的 HTTP API 到您的自动化脚本中。

## 快速开始

### 环境要求

- Android 11+ 设备
- 已安装 Droidrun Portal 并启用无障碍服务
- ADB 已配置

### ADB 连接步骤

```bash
# 1. 连接设备（USB 或 WiFi）
adb devices

# 2. 设置端口转发
adb forward tcp:8080 tcp:8080

# 3. 测试连接
curl http://localhost:8080/ping
# 响应: {"status":"success","result":"pong"}
```

---

## Python 示例

### 基础 HTTP 客户端类

```python
import requests
import base64
import json
from typing import Optional, Dict, Any


class DroidrunClient:
    """Droidrun Portal HTTP API 客户端"""

    def __init__(self, token: str, host: str = "localhost", port: int = 8080):
        self.base_url = f"http://{host}:{port}"
        self.headers = {
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/x-www-form-urlencoded"
        }

    def _request(self, method: str, endpoint: str, data: Optional[Dict] = None) -> Dict:
        """发送 HTTP 请求"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        if method.upper() == "GET":
            response = requests.get(url, headers=self.headers)
        else:
            response = requests.post(url, headers=self.headers, data=data)
        return response.json()

    # ========== 查询操作 ==========

    def ping(self) -> Dict:
        """测试连接（无需认证）"""
        return requests.get(f"{self.base_url}/ping").json()

    def get_version(self) -> str:
        """获取版本"""
        return self._request("GET", "/version")["result"]

    def get_state(self, filter: bool = True) -> Dict:
        """获取无障碍树和手机状态"""
        endpoint = "/state" if filter else "/state_full?filter=false"
        return self._request("GET", endpoint)

    def get_a11y_tree(self) -> Dict:
        """获取可见元素列表"""
        return self._request("GET", "/a11y_tree")

    def get_phone_state(self) -> Dict:
        """获取手机状态（当前应用、焦点元素、键盘状态等）"""
        return self._request("GET", "/phone_state")

    def get_packages(self) -> list:
        """获取已安装的可启动应用"""
        return self._request("GET", "/packages")["result"]

    def screenshot(self, hide_overlay: bool = True) -> bytes:
        """获取截图（返回 PNG 字节）"""
        url = f"{self.base_url}/screenshot?hideOverlay={str(hide_overlay).lower()}"
        response = requests.get(url, headers=self.headers)
        return response.content

    def save_screenshot(self, filepath: str, hide_overlay: bool = True) -> bool:
        """保存截图到文件"""
        try:
            data = self.screenshot(hide_overlay)
            with open(filepath, "wb") as f:
                f.write(data)
            print(f"Screenshot saved to {filepath}")
            return True
        except Exception as e:
            print(f"Failed to save screenshot: {e}")
            return False

    # ========== 操作命令 ==========

    def tap(self, x: int, y: int) -> Dict:
        """点击坐标"""
        return self._request("POST", "/tap", {"x": x, "y": y})

    def swipe(self, start_x: int, start_y: int, end_x: int, end_y: int, duration: int = 300) -> Dict:
        """滑动手势"""
        return self._request("POST", "/swipe", {
            "startX": start_x,
            "startY": start_y,
            "endX": end_x,
            "endY": end_y,
            "duration": duration
        })

    def global_action(self, action: int) -> Dict:
        """执行全局操作
        常用操作：
        - 3: HOME
        - 4: BACK
        - 187: RECENTS (最近任务)
        """
        return self._request("POST", "/global", {"action": action})

    def home(self) -> Dict:
        """返回主屏幕"""
        return self.global_action(3)

    def back(self) -> Dict:
        """返回"""
        return self.global_action(4)

    def recents(self) -> Dict:
        """打开最近任务"""
        return self.global_action(187)

    def open_app(self, package: str, activity: Optional[str] = None) -> Dict:
        """打开应用"""
        data = {"package": package}
        if activity:
            data["activity"] = activity
        return self._request("POST", "/app", data)

    def keyboard_input(self, text: str, clear: bool = True) -> Dict:
        """输入文本"""
        encoded = base64.b64encode(text.encode()).decode()
        return self._request("POST", "/keyboard/input", {
            "base64_text": encoded,
            "clear": str(clear).lower()
        })

    def keyboard_clear(self) -> Dict:
        """清空输入框"""
        return self._request("POST", "/keyboard/clear")

    def keyboard_key(self, key_code: int) -> Dict:
        """发送按键（Android key code）"""
        return self._request("POST", "/keyboard/key", {"key_code": key_code})

    def set_overlay_offset(self, offset: int) -> Dict:
        """设置覆盖层偏移"""
        return self._request("POST", "/overlay_offset", {"offset": offset})

    def set_overlay_visible(self, visible: bool) -> Dict:
        """切换覆盖层可见性"""
        return self._request("POST", "/overlay_visible", {"visible": str(visible).lower()})
```

### 使用示例

```python
#!/usr/bin/env python3
"""
Droidrun Portal 使用示例
"""

from droidrun_client import DroidrunClient


def main():
    # 初始化客户端
    # 请将 YOUR_TOKEN 替换为实际的认证令牌
    TOKEN = "your-token-here"
    client = DroidrunClient(token=TOKEN)

    # 1. 测试连接
    print("=== 测试连接 ===")
    result = client.ping()
    print(f"Ping: {result}")

    # 2. 获取版本
    print("\n=== 获取版本 ===")
    version = client.get_version()
    print(f"Version: {version}")

    # 3. 点击坐标
    print("\n=== 点击坐标 (300, 500) ===")
    result = client.tap(300, 500)
    print(f"Tap result: {result}")

    # 4. 滑动手势
    print("\n=== 滑动手势 ===")
    result = client.swipe(100, 500, 400, 500, duration=300)
    print(f"Swipe result: {result}")

    # 5. 返回主屏幕
    print("\n=== 返回主屏幕 ===")
    result = client.home()
    print(f"Home result: {result}")

    # 6. 打开应用
    print("\n=== 打开设置应用 ===")
    result = client.open_app("com.android.settings")
    print(f"Open app result: {result}")

    # 7. 输入文本
    print("\n=== 输入文本 ===")
    result = client.keyboard_input("Hello World", clear=True)
    print(f"Input result: {result}")

    # 8. 保存截图
    print("\n=== 保存截图 ===")
    client.save_screenshot("screenshot.png")

    # 9. 获取无障碍树
    print("\n=== 获取无障碍树 ===")
    state = client.get_state()
    print(f"Tree has {len(state.get('a11y_tree', []))} elements")

    # 10. 获取手机状态
    print("\n=== 获取手机状态 ===")
    phone_state = client.get_phone_state()
    print(f"Current package: {phone_state.get('result', {}).get('current_package', 'unknown')}")


if __name__ == "__main__":
    main()
```

### 自动化脚本示例

```python
#!/usr/bin/env python3
"""
自动化示例：打开应用并执行操作
"""

from droidrun_client import DroidrunClient
import time


def automate_open_app():
    """自动化：打开应用并点击特定按钮"""

    client = DroidrunClient(token="your-token-here")

    # 1. 确保在主屏幕
    print("返回主屏幕...")
    client.home()
    time.sleep(0.5)

    # 2. 打开设置
    print("打开设置应用...")
    client.open_app("com.android.settings")
    time.sleep(1)

    # 3. 获取当前状态
    print("获取无障碍树...")
    state = client.get_state()
    tree = state.get("a11y_tree", [])

    # 4. 查找并点击第一个可点击元素
    for element in tree:
        if element.get("type") == "Clickable":
            print(f"找到可点击元素: {element.get('text', 'unknown')}")
            # 点击元素
            # 注意：需要从 element 中提取坐标
            # 实际使用时需要根据返回的 rect 信息计算
            break

    # 5. 截图保存
    client.save_screenshot("after_action.png")
    print("截图已保存")


def automate_walk_through():
    """自动化：遍历屏幕并截图"""

    client = DroidrunClient(token="your-token-here")

    for i in range(5):
        print(f"步骤 {i + 1}: 截图...")
        client.save_screenshot(f"screenshot_{i + 1}.png")

        # 向下滑动
        client.swipe(200, 600, 200, 200, duration=500)
        time.sleep(0.5)


if __name__ == "__main__":
    automate_open_app()
    # automate_walk_through()
```

---

## JavaScript / Node.js 示例

### 基础 HTTP 客户端

```javascript
/**
 * Droidrun Portal HTTP API 客户端 (Node.js)
 */

const http = require('http');

class DroidrunClient {
    constructor(token, host = 'localhost', port = 8080) {
        this.baseUrl = `http://${host}:${port}`;
        this.headers = {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/x-www-form-urlencoded'
        };
    }

    _request(method, endpoint, data = null) {
        return new Promise((resolve, reject) => {
            const url = new URL(endpoint, this.baseUrl);
            const options = {
                hostname: url.hostname,
                port: url.port,
                path: url.pathname,
                method: method,
                headers: this.headers
            };

            const req = http.request(options, (res) => {
                let body = '';
                res.on('data', chunk => body += chunk);
                res.on('end', () => {
                    try {
                        resolve(JSON.parse(body));
                    } catch (e) {
                        resolve(body);
                    }
                });
            });

            req.on('error', reject);

            if (data) {
                const params = new URLSearchParams(data);
                req.write(params.toString());
            }

            req.end();
        });
    }

    // ========== 查询操作 ==========

    async ping() {
        // 无需认证
        const response = await fetch(`${this.baseUrl}/ping`);
        return response.json();
    }

    async getVersion() {
        const result = await this._request('GET', '/version');
        return result.result;
    }

    async getState(filter = true) {
        const endpoint = filter ? '/state' : '/state_full?filter=false';
        return this._request('GET', endpoint);
    }

    async getA11yTree() {
        return this._request('GET', '/a11y_tree');
    }

    async getPhoneState() {
        return this._request('GET', '/phone_state');
    }

    async getPackages() {
        const result = await this._request('GET', '/packages');
        return result.result;
    }

    async screenshot(hideOverlay = true) {
        const url = `${this.baseUrl}/screenshot?hideOverlay=${hideOverlay}`;
        const response = await fetch(url, {
            headers: { 'Authorization': this.headers['Authorization'] }
        });
        return response.buffer();
    }

    async saveScreenshot(filepath, hideOverlay = true) {
        try {
            const data = await this.screenshot(hideOverlay);
            const fs = require('fs');
            fs.writeFileSync(filepath, data);
            console.log(`Screenshot saved to ${filepath}`);
            return true;
        } catch (e) {
            console.error(`Failed to save screenshot: ${e.message}`);
            return false;
        }
    }

    // ========== 操作命令 ==========

    async tap(x, y) {
        return this._request('POST', '/tap', { x, y });
    }

    async swipe(startX, startY, endX, endY, duration = 300) {
        return this._request('POST', '/swipe', {
            startX, startY, endX, endY, duration
        });
    }

    async globalAction(action) {
        // action: 3=Home, 4=Back, 187=Recents
        return this._request('POST', '/global', { action });
    }

    async home() {
        return this.globalAction(3);
    }

    async back() {
        return this.globalAction(4);
    }

    async recents() {
        return this.globalAction(187);
    }

    async openApp(packageName, activity = null) {
        const data = { package: packageName };
        if (activity) data.activity = activity;
        return this._request('POST', '/app', data);
    }

    async keyboardInput(text, clear = true) {
        const encoded = Buffer.from(text).toString('base64');
        return this._request('POST', '/keyboard/input', {
            base64_text: encoded,
            clear: clear.toString()
        });
    }

    async keyboardClear() {
        return this._request('POST', '/keyboard/clear');
    }

    async keyboardKey(keyCode) {
        return this._request('POST', '/keyboard/key', { key_code: keyCode });
    }

    async setOverlayOffset(offset) {
        return this._request('POST', '/overlay_offset', { offset });
    }

    async setOverlayVisible(visible) {
        return this._request('POST', '/overlay_visible', { visible: visible.toString() });
    }
}

module.exports = DroidrunClient;
```

### 使用示例 (Node.js)

```javascript
/**
 * Droidrun Portal 使用示例 (Node.js)
 */

const DroidrunClient = require('./droidrun-client');

async function main() {
    // 初始化客户端
    const client = new DroidrunClient('your-token-here');

    // 1. 测试连接
    console.log('=== 测试连接 ===');
    const ping = await client.ping();
    console.log('Ping:', ping);

    // 2. 获取版本
    console.log('\n=== 获取版本 ===');
    const version = await client.getVersion();
    console.log('Version:', version);

    // 3. 点击坐标
    console.log('\n=== 点击坐标 (300, 500) ===');
    const tapResult = await client.tap(300, 500);
    console.log('Tap result:', tapResult);

    // 4. 滑动手势
    console.log('\n=== 滑动手势 ===');
    const swipeResult = await client.swipe(100, 500, 400, 500, 300);
    console.log('Swipe result:', swipeResult);

    // 5. 返回主屏幕
    console.log('\n=== 返回主屏幕 ===');
    await client.home();

    // 6. 打开应用
    console.log('\n=== 打开设置应用 ===');
    await client.openApp('com.android.settings');

    // 7. 输入文本
    console.log('\n=== 输入文本 ===');
    await client.keyboardInput('Hello from Node.js!', true);

    // 8. 保存截图
    console.log('\n=== 保存截图 ===');
    await client.saveScreenshot('screenshot.png');

    // 9. 获取无障碍树
    console.log('\n=== 获取无障碍树 ===');
    const state = await client.getState();
    console.log(`Tree has ${state.a11y_tree?.length || 0} elements`);

    // 10. 获取手机状态
    console.log('\n=== 获取手机状态 ===');
    const phoneState = await client.getPhoneState();
    console.log('Current package:', phoneState.result?.current_package);
}

main().catch(console.error);
```

### 使用 axios 的示例

```javascript
/**
 * 使用 axios 的简洁示例
 */

const axios = require('axios');
const fs = require('fs');

const TOKEN = 'your-token-here';
const BASE_URL = 'http://localhost:8080';

const client = axios.create({
    headers: {
        'Authorization': `Bearer ${TOKEN}`,
        'Content-Type': 'application/x-www-form-urlencoded'
    }
});

// 便捷方法
const api = {
    ping: () => axios.get(`${BASE_URL}/ping`),

    tap: (x, y) => client.post('/tap', { x, y }),

    swipe: (startX, startY, endX, endY, duration = 300) =>
        client.post('/swipe', { startX, startY, endX, endY, duration }),

    home: () => client.post('/global', { action: 3 }),

    back: () => client.post('/global', { action: 4 }),

    openApp: (pkg) => client.post('/app', { package: pkg }),

    inputText: (text) => {
        const encoded = Buffer.from(text).toString('base64');
        return client.post('/keyboard/input', { base64_text: encoded, clear: 'true' });
    },

    screenshot: async () => {
        const response = await client.get('/screenshot', { responseType: 'arraybuffer' });
        return response.data;
    },

    state: () => client.get('/state'),

    packages: () => client.get('/packages')
};

// 使用示例
async function example() {
    // 测试
    const ping = await api.ping();
    console.log('Ping:', ping.data);

    // 点击
    await api.tap(200, 400);

    // 截图
    const img = await api.screenshot();
    fs.writeFileSync('screenshot.png', img.data);
    console.log('Screenshot saved');

    // 打开应用
    await api.openApp('com.android.settings');
}

example().catch(console.error);
```

---

## Bash 快速命令

### 常用命令速查

```bash
# 配置
TOKEN="your-token-here"
export BASE_URL="http://localhost:8080"
export AUTH_HEADER="Authorization: Bearer $TOKEN"

# 测试连接
curl "$BASE_URL/ping"

# 获取版本
curl -H "$AUTH_HEADER" "$BASE_URL/version"

# 获取无障碍树
curl -H "$AUTH_HEADER" "$BASE_URL/state"

# 点击坐标
curl -X POST -H "$AUTH_HEADER" -d "x=300&y=500" "$BASE_URL/tap"

# 滑动手势
curl -X POST -H "$AUTH_HEADER" -d "startX=100&startY=500&endX=400&endY=500&duration=300" "$BASE_URL/swipe"

# 返回主屏幕
curl -X POST -H "$AUTH_HEADER" -d "action=3" "$BASE_URL/global"

# 返回
curl -X POST -H "$AUTH_HEADER" -d "action=4" "$BASE_URL/global"

# 打开应用
curl -X POST -H "$AUTH_HEADER" -d "package=com.android.settings" "$BASE_URL/app"

# 输入文本
curl -X POST -H "$AUTH_HEADER" -d "base64_text=aGVsbG8=&clear=true" "$BASE_URL/keyboard/input"

# 截图
curl -H "$AUTH_HEADER" "$BASE_URL/screenshot" > screenshot.png

# 获取已安装应用列表
curl -H "$AUTH_HEADER" "$BASE_URL/packages"
```

### 一键测试脚本

```bash
#!/bin/bash
# save as test_droidrun.sh

TOKEN="${1:-your-token-here}"
BASE_URL="http://localhost:8080"
AUTH_HEADER="Authorization: Bearer $TOKEN"

echo "=== Droidrun Portal 测试 ==="
echo "Token: ${TOKEN:0:8}..."
echo ""

echo "1. 测试连接..."
curl -s "$BASE_URL/ping"
echo ""

echo "2. 获取版本..."
curl -s -H "$AUTH_HEADER" "$BASE_URL/version"
echo ""

echo "3. 获取状态..."
curl -s -H "$AUTH_HEADER" "$BASE_URL/state" | head -c 200
echo "..."
echo ""

echo "4. 点击坐标 (300, 500)..."
curl -s -X POST -H "$AUTH_HEADER" -d "x=300&y=500" "$BASE_URL/tap"
echo ""

echo "5. 返回主屏幕..."
curl -s -X POST -H "$AUTH_HEADER" -d "action=3" "$BASE_URL/global"
echo ""

echo "测试完成！"
```

---

## API 速查表

### GET 请求

| 端点 | 描述 | 示例 |
|------|------|------|
| `/ping` | 测试连接（无需认证） | `GET /ping` |
| `/version` | 获取版本 | `GET /version` |
| `/a11y_tree` | 获取可见元素 | `GET /a11y_tree` |
| `/a11y_tree_full` | 获取完整树 | `GET /a11y_tree_full?filter=false` |
| `/state` | 获取组合状态 | `GET /state` |
| `/state_full` | 完整状态（无过滤） | `GET /state_full?filter=false` |
| `/phone_state` | 获取手机状态 | `GET /phone_state` |
| `/packages` | 获取已安装应用 | `GET /packages` |
| `/screenshot` | 获取截图 | `GET /screenshot?hideOverlay=true` |

### POST 请求

| 端点 | 参数 | 描述 |
|------|------|------|
| `/tap` | `x`, `y` | 点击坐标 |
| `/swipe` | `startX`, `startY`, `endX`, `endY`, `duration` | 滑动手势 |
| `/global` | `action` | 全局操作（3=Home, 4=Back, 187=Recents） |
| `/app` | `package`, `activity` | 打开应用 |
| `/keyboard/input` | `base64_text`, `clear` | 输入文本 |
| `/keyboard/clear` | - | 清空输入框 |
| `/keyboard/key` | `key_code` | 发送按键 |
| `/overlay_offset` | `offset` | 设置偏移 |
| `/overlay_visible` | `visible` | 切换覆盖层 |

---

## 故障排除

### ADB 连接问题

```bash
# 检查设备
adb devices

# 重启 ADB 服务器
adb kill-server
adb start-server
```

### 认证失败

```bash
# 检查令牌
adb shell content query --uri content://com.droidrun.portal/auth_token

# 确保请求头格式正确
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:8080/state
```

### 服务未运行

```bash
# 确保应用已安装且无障碍服务已启用
adb shell content query --uri content://com.droidrun.portal/version
```

### 截图失败

```bash
# 确保覆盖层已隐藏（某些设备需要）
curl -X POST -H "Authorization: Bearer TOKEN" -d "visible=false" http://localhost:8080/overlay_visible

# 重新尝试截图
curl -H "Authorization: Bearer TOKEN" "http://localhost:8080/screenshot?hideOverlay=true" > screenshot.png
```
