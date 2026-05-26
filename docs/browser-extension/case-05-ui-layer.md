# 案例 05：UI 层 — Side Panel 交互设计

## 概述

淘宝插件的主要 UI 不是 Popup，而是 **Side Panel**——`sidepanel.js`（3048 行）是项目最大的文件。同时还有 `category-analytics.js`（1092 行）和 `shop-panel.js`（1043 行）。

## Side Panel vs Popup 的选择

这个插件用了 Side Panel 而非 Popup，基于以下差异：

| | Popup | Side Panel |
|---|---|---|
| 空间 | 小窗口 ~600x400 | 侧边全高度 |
| 生命周期 | 点击即消失 | 可常驻 |
| 多标签 | 每个标签独立 | 可配置 |
| 适合 | 一次性操作 | **这个插件的场景**：数据采集、品类分析 |

数据采集和 AI 分析是典型的**长任务**——用户可能在一个品类上花 10 分钟查看 SPU、切换搜索条件。Popup 不够用。

## Side Panel 状态管理

Side Panel 维护了复杂的状态机：

```javascript
var currentView = 'search';       // 'search' | 'detail' | 'nomatch'
var currentCategory = '';         // 当前选中的品类名
var cachedCategories = [];        // 品类列表
var cachedSpuItems = [];          // 当前品类 SPU
var currentSortMode = 'comprehensive'; // 排序模式
var selectedNumId = null;         // 选中的 SPU
var analysisFullscreen = false;   // 分析区是否占满
var analysisDetailTab = 'local';  // 'local' | 'ai'
var categoryReportCache = {};     // 品类分析报告缓存
```

这些都是**内存状态**，页面刷新（Side Panel 关闭/重开）就丢失。但业务数据（品类、SPU、详情）在 `chrome.storage.local` 中，所以 Side Panel 每次打开时会重新加载：

```javascript
// 启动时初始化
function init() {
  loadCategories();    // 从 storage 加载品类
  setupListeners();    // 注册消息监听
  syncWithContent();   // 与搜索页同步
}
```

## 双栏布局

Side Panel 实现了双栏布局：

```
┌──────────────────────────────────┐
│ 品类下拉选择  [分析] [导出]       │  ← 工具栏
├────────────┬─────────────────────┤
│ SPU 列表   │   详情 / 分析面板    │  ← 双栏
│ ┌──┐       │                     │
│ │✓│ SPU-1  │   SKU 分析          │
│ │ │ SPU-2  │   AI 报告           │
│ │✓│ SPU-3  │   ……               │
│ └──┘       │                     │
├────────────┴─────────────────────┤
│ 状态栏：已采集 50/50             │  ← 底部
└──────────────────────────────────┘
```

左栏是 SPU 列表，右栏是详情/分析面板。这是电商数据分析工具的典型布局。

## 消息通信

Side Panel 是消息的**接收端**和**发送端**：

### 接收

```javascript
chrome.runtime.onMessage.addListener(function(msg, sender, sendResponse) {
  if (msg.action === 'syncSearchState') {
    updateSpuList(msg.data);       // 搜索页采集了新数据
  }
  if (msg.action === 'selectSpu') {
    showDetail(msg.data);          // 搜索页选中某个 SPU
  }
});
```

### 发送

```javascript
// 向搜索页 CS 发送品类切换
chrome.runtime.sendMessage({
  action: 'forwardToTab',
  tabId: currentTabId,
  msg: { action: 'switchCategory', category: name }
});
```

## Popup 的角色

Popup 在这个项目中只占了 130 行，功能是辅助入口：

```javascript
// popup.js
function updateCount() {
  // 显示当前采集的商品数量
  chrome.runtime.sendMessage({ action: 'getCount' }, function(resp) {
    document.getElementById('count').textContent = resp.count;
  });
}
```

它不做主要的交互——这在 manifest 中也体现为没有 `default_popup`，用的是无 Popup 模式。

## 其他 UI 方式

| UI 方式 | 是否使用 |
|---|---|
| Action badge | 否（可以加，显示当前采集数） |
| Context Menus | 否 |
| Omnibox | 否 |
| Commands | 否 |
| Notifications | 否 |
| Action onClicked | ✅ 打开 Side Panel |

项目中只用了 Side Panel 和少量的 Popup，其他 UI 方式都没用。这是合理的——淘宝插件不需要快捷键唤醒、不需要右键菜单、不需要搜索栏指令。**功能集中的工具，UI 入口也应该集中。**

---

## 参考

- 源码：`Projects/taobao-plugin/extension/src/sidepanel/sidepanel.js`
- 源码：`Projects/taobao-plugin/extension/src/popup/popup.js`
- 知识体系：[UI 层](06-ui-layer.md)
