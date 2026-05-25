# 案例 01：淘宝品类分析 — 架构总览与 Manifest

## 项目背景

一个淘宝搜索结果品类分析工具。功能链：

```
用户在淘宝搜索 → 搜索页 CS 采集商品数据 → 存入 storage
          → Side Panel 展示品类/SPU 列表
          → 用户勾选品类 → AI 分析（DeepSeek）→ 导出 CSV/XLS
```

## Manifest 架构

```json
{
  "manifest_version": 3,
  "name": "淘宝数据采集助手",
  "version": "0.7.0",
  "permissions": [
    "storage", "activeTab", "scripting", "tabs",
    "windows", "downloads", "sidePanel"
  ],
  "host_permissions": [
    "https://*.taobao.com/*",
    "https://*.tmall.com/*",
    "https://sycm.taobao.com/*",
    "https://api.deepseek.com/*"
  ]
}
```

关键设计决策：

### 为什么用 `activeTab` 而非 `<all_urls>`

只用 `activeTab` + 具体域名白名单。用户点击扩展时才有权限，日常不消耗。这是 MV3 推荐的最佳实践——权限最小化。

### 权限矩阵与实际用途

| 权限 | 实际用途 |
|---|---|
| `storage` | 品类/SPU/详情数据持久化 |
| `activeTab` | 注入 CS 到用户正在看的页面 |
| `scripting` | `chrome.scripting.executeScript` 动态注入 |
| `tabs` | 广播消息到指定标签 |
| `sidePanel` | MV3 侧栏作为核心 UI |
| `downloads` | 导出 CSV/XLS |
| `host_permissions` | 跨域请求淘宝页面 + DeepSeek API |

## 五大组件映射

| 组件 | 文件中 |
|---|---|
| Service Worker | `src/background/background.js` — 239 行 |
| Content Scripts | `src/content/search-page.js` — 1117 行 |
|  | `src/content/detail-page.js` — 202 行 |
|  | `src/content/shop-search-page.js` — 195 行 |
|  | `src/content/sycm-page.js` — 70 行 |
| Side Panel | `src/sidepanel/sidepanel.js` — 3048 行 |
|  | `src/sidepanel/shop-panel.js` — 1043 行 |
|  | `src/sidepanel/category-analytics.js` — 1092 行 |
| Popup | `src/popup/popup.js` — 130 行（辅助入口） |

架构模式：**Content Script 做数据采集，Side Panel 做交互展示，Background 做中转+导出**。这是一个典型的 MV3 架构——Service Worker 轻量化，复杂的 UI 工作交给 Side Panel。

## 消息流路径

```
搜索页 CS 采集数据 → sendMessage → SW 中转页 → forwardToTab → Side Panel
                                                        ↓
                                              Side Panel 展示 → 用户操作
                                                        ↓
                                              发回 CS 操作品类/删除
```

背景 SW 在这里扮演了**路由器**角色——Content Script 和 Side Panel 不能直接通过 `tabs.sendMessage` 互发消息（Side Panel 没有 `tabs` API），所以全部通过 SW 的 `forwardToTab` 中转。

## 关键架构模式

### 1. 没有 Popup，用 Side Panel 做主 UI

用户在淘宝搜索页面点击扩展图标 → `chrome.action.onClicked` → 打开 Side Panel。Side Panel 不随点击消失，适合长任务（数据分析、AI 分析、商品采集）。

### 2. Content Script 不直接调 Side Panel

CS → SW(forwardToTab) → Side Panel。这在代码中体现为：

```javascript
// background.js
if (msg.action === 'forwardToTab') {
  chrome.tabs.sendMessage(msg.tabId, msg.msg, function (response) {
    sendResponse(response);
  });
  return true;
}
```

### 3. 按页面类型拆 CS

四个独立的 Content Script 文件，每个对应一类淘宝页面：搜索、详情、店铺、生意参谋。manifest 中用 `matches` 精确匹配，而不是一个巨大的 CS 做全部事情。

---

## 参考

- [源码路径](https://github.com/sylar-qiu/grow-path/tree/main) 下的 `Projects/taobao-plugin/extension/manifest.json`
