# 案例 02：内容脚本 — 淘宝搜索页数据采集

## 概述

`search-page.js`（1117 行）是淘宝搜索页面（`s.taobao.com`）的 Content Script。除了它还运行 `detail-page.js`（详情页）、`shop-search-page.js`（店铺页）和 `sycm-page.js`（生意参谋）。

## 注入方式

```json
// manifest.json
{
  "matches": [
    "https://s.taobao.com/*",
    "https://*.taobao.com/search*",
    "https://list.tmall.com/*",
    ...
  ],
  "js": ["lib/parser-utils.js", "lib/shared-utils.js",
         "src/lib/storage.js", "src/content/search-page.js"],
  "run_at": "document_end"
}
```

注意 `js` 数组的顺序——依赖在前（`parser-utils.js` -> `shared-utils.js` -> `storage.js`），业务脚本在最后。这是在 manifest 中声明式注入。

### 为什么用 document_end？

淘宝页面用 SSR 渲染，`document_end` 时 DOM 已经就绪，但图片/广告等资源还在加载。商品卡片数据此时已在 HTML 中，可以直接采集。

## 状态管理

CS 在页面内维持一套内存缓存：

```javascript
var currentCategory = '';
var floatBar = null;
var pinnedBars = new Map();
var storageKeywordMap = {};
var storageCategoryItems = {};
var salesRankSnapshotByQuery = {};
```

所有状态都是页面级别的——页面刷新就丢失。**这导致了搜索页与侧栏之间的核心同步模式**：CS 和 Side Panel 不共享内存状态，每次数据变更都必须通过消息同步。

## 核心功能模块

### 1. 浮动横条

CS 直接在页面中注入 UI（`injectStyles` 函数）：

```javascript
(function injectStyles() {
  var style = document.createElement('style');
  style.textContent = [/* 全部 CSS */].join('\n');
  document.head.appendChild(style);
})();
```

这对应知识体系中"Content Script 注入自定义 UI"的模式——直接在页面 DOM 中插入元素。

### 2. 品类操作（数据层）

约有 40 个函数专门管理品类/SPU 的增删改查。典型链路：

```
toggleItemInCategory → saveItemData / removeItemFromCategory
                     → save → chrome.storage.local.set
                     → sendToSidepanel({ action: 'syncSearchState' })
```

### 3. 页面 DOM 监听

```javascript
var domObserver = new MutationObserver(function(mutations) {
  // 页面 AJAX 加载新商品时重新采集
  observeDomChanges();
});

function watchUrlChange() {
  // SPA 页面 URL 变化时重新初始化
  var lastUrl = location.href;
  setInterval(function() {
    if (location.href !== lastUrl) {
      lastUrl = location.href;
      reinitialize();
    }
  }, 500);
}
```

两种监听方式配合：MutationObserver 监听 DOM 变化，轮询监听 URL 变化（淘宝用 history.pushState，没有 popstate 事件）。

## Main World 通信

这个 CS 用了 `Isolated World` 模式——它不试图访问页面 JS 变量，而是通过 DOM 直接提取数据。商品名称、价格、销量都在 DOM 中，所以不需要 Main World 注入。

`detail-page.js` 稍微不同——它需要从页面内嵌的 JavaScript 变量（`window.__INIT_DATA__`）中提取参数和 SKU 数据：

```javascript
// detail-page.js 中
function extractParams() {
  // 读取淘宝 SSR 页面内嵌的初始化数据
  var script = document.querySelector('#__INIT_DATA__');
  if (script) {
    var data = JSON.parse(script.textContent);
    // 解析参数
  }
}
```

## 边界处理

### 扩展重新加载检测

```javascript
var extensionContextInvalidated = false;
function markExtensionContextInvalidatedIfSo(e) {
  // 当扩展在 chrome://extensions 重新加载后，旧 CS 仍在页面运行
  // 但 chrome.storage / runtime 已经断开了
  if (isExtensionContextInvalidatedError(e)) {
    extensionContextInvalidated = true;
    // 显示提示，引导用户刷新页面
  }
}
```

### 存储写入失败的处理

```javascript
function handleStorageRejection(err) {
  if (extensionContextInvalidated) return;
  // 重试或优雅降级
}
```

---

## 参考

- 源码：`Projects/taobao-plugin/extension/src/content/search-page.js`
- 知识体系：[内容脚本层](03-content-scripts.md)
