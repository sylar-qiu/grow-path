# 案例 03：Service Worker — 淘宝插件后台

## 概述

`background.js`（239 行）是淘宝插件的 Service Worker。相比 Side Panel（3048 行），它非常精炼——这是 MV3 的正确设计模式。

## 事件驱动

```javascript
// 安装事件
chrome.runtime.onInstalled.addListener(() => {
  // 初始化存储
  // 设置 sidePanel 行为
  chrome.sidePanel.setPanelBehavior({ openPanelOnActionClick: true });
});

// 点击图标（无 Popup）
chrome.action.onClicked.addListener(function(tab) {
  chrome.sidePanel.open({ tabId: tab.id });
});

// 消息处理——最主要的工作
chrome.runtime.onMessage.addListener(function(msg, sender, sendResponse) {
  // 处理 7 个 action
});
```

三个事件覆盖了整个 SW 的工作。没有 `alarms`，没有周期性任务——它只在需要时被唤醒。

## 中转模式

SW 最核心的作用不是执行逻辑，而是**中转**。Content Script 和 Side Panel 不能直接通信（Side Panel 没有 `tabs` API）：

```javascript
// Content Script 需要发消息给 Side Panel
// → 必须先发给 SW，SW 再转发

// background.js
if (msg.action === 'forwardToTab') {
  chrome.tabs.sendMessage(msg.tabId, msg.msg, function (response) {
    sendResponse(response);
  });
  return true;  // 保持通道开放等异步响应
}
```

对应的发送端（Side Panel 中）：

```javascript
// sidepanel.js
chrome.runtime.sendMessage({
  action: 'forwardToTab',
  tabId: targetTabId,
  msg: { action: 'switchCategory', category: name }
}, function(response) { ... });
```

## 为什么 SW 不做数据采集逻辑？

SW 只有 239 行，而 Side Panel 有 3048 行，搜索页 CS 有 1117 行。这是设计选择：

| 功能 | 放在哪 | 原因 |
|---|---|---|
| 数据采集、DOM 操作 | Content Script | 需要页面 DOM |
| 品类展示、AI 分析、报表 | Side Panel | 需要持续 UI，空间大 |
| 中转、导出文件 | Service Worker | 不需要 UI，事件驱动即可 |

如果 SW 休眠了也不会影响——用户在 Side Panel 中操作时，会发送消息唤醒 SW，SW 处理完立即空闲。SW 不持有任何持久状态，所有数据都通过 `chrome.storage.local` 存取。

## 导出操作

SW 处理 CSV/XLS 导出：

```javascript
async function handleExport() {
  // 从 storage 读数据
  const { collected } = await chrome.storage.local.get('collected');
  // 生成 CSV
  const csv = itemsToCSV(collected);
  // 创建 Blob URL
  const blob = new Blob([csv], { type: 'text/csv' });
  const url = URL.createObjectURL(blob);
  // 触发下载
  await chrome.downloads.download({ url: url, filename: 'export.csv' });
}
```

这个操作利用了 SW 能调用的 API 限制内恰好能做的工作——不涉及 DOM，不依赖页面，纯数据转换 + 下载 API。

## 休眠与唤醒分析

在实际使用中，这个扩展的场景是：

1. 用户在淘宝搜索 → SW 不参与
2. CS 采集数据 → 存 `chrome.storage.local` → 发消息给 SW → SW 唤醒
3. SW 执行 `forwardToTab` → Side Panel 收到数据 → SW 空闲 → 5 分钟后休眠
4. 用户操作 Side Panel → 发消息给 SW → SW 再次唤醒

SW 每次活跃时间不超过几毫秒（只是消息转发），所以**不会触发 30 秒超时限制**。这是设计贴合系统的典型案例。

## 潜在的 SW 缺陷

1. **无持久状态**：SW 中没有缓存变量（除了 `console.log` 没有副作用损失），这其实是正确的
2. **无 Offscreen Document**：如果需要生成带样式的 HTML 导出文件，SW 做不了——需要 Offscreen Document。目前导出的是纯文本 CSV，不需要 DOM
3. **无错误恢复**：如果 `forwardToTab` 的目标标签已关闭，`chrome.tabs.sendMessage` 会静默失败

---

## 参考

- 源码：`Projects/taobao-plugin/extension/src/background/background.js`
- 知识体系：[Service Worker 层](04-service-worker.md)
