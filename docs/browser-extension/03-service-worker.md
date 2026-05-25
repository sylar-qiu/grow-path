# Service Worker 层

这是 MV3 最核心也最容易踩坑的变化。

## 事件驱动模型

SW 不是常驻进程，而是事件驱动的：

```
空闲 → 事件触发 → 执行 → 空闲（5 分钟后休眠）
```

### 能唤醒 SW 的事件

```javascript
chrome.runtime.onInstalled        // 安装/更新/权限变更
chrome.runtime.onMessage          // 收到消息
chrome.runtime.onConnect          // 收到长连接
chrome.alarms.onAlarm             // 定时器触发
chrome.webNavigation.onCompleted  // 页面导航完成
chrome.tabs.onUpdated             // 标签更新
chrome.commands.onCommand         // 快捷键
chrome.storage.onChanged          // 存储变化
```

### 关键限制

- **30 秒事件超时**：一个事件处理器最多跑 30 秒，超时自动终止
- **5 分钟空闲休眠**：没有新事件 5 分钟后 SW 被休眠

这两个限制意味着你**不能**：

```javascript
// ❌ 依赖内存变量持久存活
let counter = 0;
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  counter++;  // 休眠后丢失
  sendResponse({ count: counter });
});

// ❌ 长时间循环
chrome.alarms.onAlarm.addListener(() => {
  while (true) { /* 30 秒后被强制终止 */ }
});
```

---

## 状态管理

### 正确做法：显式持久化

```javascript
// ✅ 写入 storage，每次启动从 storage 恢复
async function getState() {
  const { counter = 0 } = await chrome.storage.session.get('counter');
  return counter;
}

async function incrementState() {
  const counter = await getState();
  await chrome.storage.session.set({ counter: counter + 1 });
  return counter + 1;
}

// 事件入口
chrome.runtime.onMessage.addListener(async (msg, sender, sendResponse) => {
  if (msg.type === 'get-counter') {
    const count = await getState();
    sendResponse({ count });
  }
  return true; // 保持通道开放等待异步响应
});
```

### chrome.storage.session

MV3 新增，专门为 SW 设计——内存级存储，速度快，没有写入延迟，但只存活于当前会话。SW 关闭后数据丢失。

```
local / sync → 跨会话持久
session → 同一会话内持久（SW 休眠唤醒后还在？）
```

答案是：**在**。`chrome.storage.session` 在 SW 休眠期间保持数据，适合存运行时状态。

---

## 绕过限制

### 1. 定期唤醒（Alarms）

```javascript
// 注册一个定期 alarm
chrome.runtime.onInstalled.addListener(() => {
  chrome.alarms.create('heartbeat', { periodInMinutes: 4.5 });
});

// 每 4.5 分钟被唤醒一次
chrome.alarms.onAlarm.addListener((alarm) => {
  if (alarm.name === 'heartbeat') {
    // 检查队列、做维护
  }
});
```

4.5 分钟小于 SW 的 5 分钟空闲阈值，所以可以维持持续活跃。

### 2. Offscreen Document

当 SW 需要 DOM 能力时（解析 HTML、生成图片、播放音频），用 Offscreen Document：

```javascript
// 在 SW 中
chrome.offscreen.createDocument({
  url: 'offscreen.html',
  reasons: ['DOM_PARSER'],
  justification: '解析 HTML 内容需要 DOM API'
});
```

Offscreen Document 是有 DOM 的不可见页面，SW 通过消息与它通信。但注意：

- 一次只能有一个实例
- 需要特定 `reasons` 参数（理由声明）
- Chrome 会在合理时机自动关闭

### 3. 长时间任务拆分

```javascript
// ❌ 一次性处理大量数据
chrome.runtime.onMessage.addListener(async (msg) => {
  const result = await processLargeData();  // 可能超时
  return result;
});

// ✅ 拆成批处理，用 alarm 驱动
let batchQueue = [];

chrome.runtime.onInstalled.addListener(() => {
  chrome.alarms.create('batch-process', { periodInMinutes: 0.5 });
});

chrome.runtime.onMessage.addListener((msg) => {
  batchQueue.push(msg.data);
  return true; // 不需要立即处理
});

chrome.alarms.onAlarm.addListener((alarm) => {
  if (alarm.name === 'batch-process') {
    processBatch(batchQueue.splice(0, 10));
  }
});
```

---

## SW 能力完整清单

| 类别 | 可用 |
|---|---|
| Extension API | 全部 |
| fetch / XMLHttpRequest | ✅ |
| Cache API | ✅ |
| IndexedDB | ✅ |
| WebSocket | ✅（需心跳维持） |
| WebRTC | ❌ |
| DOM / window / document | ❌ |
| setTimeout > 5min | ❌（SW 会休眠） |
| Audio / Video | ❌（用 Offscreen） |
| Canvas | ❌（用 Offscreen） |

---

## 参考

- [Chrome Extension Service Worker](https://developer.chrome.com/docs/extensions/mv3/service_workers/)
- [Offscreen Documents](https://developer.chrome.com/docs/extensions/mv3/offscreen_documents/)
