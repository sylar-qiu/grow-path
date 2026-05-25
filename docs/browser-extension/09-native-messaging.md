# 原生通信

扩展与本地进程（可执行文件、shell 脚本）交互的唯一通道。场景相对小众，但需要时它是无可替代的。

## Native Messaging 协议

### 工作流程

```
扩展                  Native Host (可执行文件)
  │                          │
  ├── chrome.runtime.connect →│  建立连接
  │                          │
  ├── port.postMessage ──────►│  发送 JSON 消息
  │                          │
  ◄────── port.onMessage ────┤  接收回复
  │                          │
  ├── port.disconnect ───────►│  断开连接
```

### Native Host 注册

本机需要安装一个 JSON 配置文件告诉浏览器"这个 Native Host 存在"。

**macOS** 路径：`~/Library/Application Support/Google/Chrome/NativeMessagingHosts/com.myapp.json`

```json
{
  "name": "com.myapp",
  "description": "与我扩展通信的本地进程",
  "path": "/usr/local/bin/my-native-app",
  "type": "stdio",
  "allowed_origins": ["chrome-extension://<你的扩展ID>/"]
}
```

**Windows** 路径：注册表 `HKEY_CURRENT_USER\Software\Google\Chrome\NativeMessagingHosts\com.myapp`

### 通信协议

消息格式：**4 字节长度（小端序）+ UTF-8 JSON**

```
[4字节长度][JSON数据]
```

每一条消息都是独立的 JSON 对象。Native Host 从 stdin 读入消息，输出到 stdout。

```javascript
// C 示例：读取消息
uint32_t len;
fread(&len, 4, 1, stdin);
char* buf = malloc(len + 1);
fread(buf, 1, len, stdin);
buf[len] = '\0';
// buf 就是 JSON 字符串

// 发送回复
char* response = "{\"status\":\"ok\"}";
uint32_t resp_len = strlen(response);
fwrite(&resp_len, 4, 1, stdout);
fwrite(response, 1, resp_len, stdout);
fflush(stdout);
```

### 安全性

- `allowed_origins` 限制了哪个扩展可以连接这个 Host
- Native Host 的运行身份是当前用户
- 扩展不能指定任意路径——必须通过已注册的 Host 名称连接

## 扩展侧代码

```javascript
// SW 中
const port = chrome.runtime.connectNative('com.myapp');

port.onMessage.addListener((msg) => {
  console.log('收到本地回复:', msg);
});

port.onDisconnect.addListener(() => {
  if (chrome.runtime.lastError) {
    console.error('连接断开:', chrome.runtime.lastError.message);
  }
});

// 发送消息
port.postMessage({ action: 'readFile', path: '/tmp/data.json' });
```

## 适合场景

| 场景 | 说明 |
|---|---|
| 文件系统操作 | 扩展无法直接读写的路径 |
| 执行本机命令 | 调用本机 CLI 工具 |
| 硬件交互 | USB、串口、蓝牙 |
| 本地数据库 | SQLite、RocksDB |
| 本地 HTTP 服务 | 启动 nginx 或 mock server |
| 与本机应用互操作 | 从你的电脑应用读取数据 |

## 不适合场景

- 简单网络请求（直接 fetch 就行）
- 定时任务（用 chrome.alarms）
- 少量文件操作（`File API` 配合 `activeTab` 足够）

---

## 参考

- [Native Messaging | Chrome Developers](https://developer.chrome.com/docs/extensions/mv3/nativeMessaging/)
