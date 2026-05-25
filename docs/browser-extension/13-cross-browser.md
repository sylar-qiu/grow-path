# 多浏览器兼容

## Edge

Chromium 内核，与 Chrome 几乎完全一致。

| 项目 | 兼容度 |
|---|---|
| Manifest V3 | 完全兼容 |
| API | 99% 一致 |
| Web Store | Edge Add-ons 市场 |

主要差异：

- 注册开发者不需要交费
- 审核速度通常比 Chrome 快
- `chrome.edge` / `browser` 别名（Chrome 用 `chrome`）
- 少数 API 差异（参考 [Edge Extension API 文档](https://learn.microsoft.com/en-us/microsoft-edge/extensions-chromium/)）

**建议**：Chrome 的扩展直接复制一份配置到 Edge 市场发布就行。

---

## Firefox

### 核心差异

Firefox 使用 `browser.*` API（基于 Promise）而非 `chrome.*`（混合回调/Promise）。

```javascript
// Firefox 原生 Promise
browser.storage.local.set({ key: value });

// Chrome（MV3 已支持一些）
chrome.storage.local.set({ key: value }).then(...)
```

### 兼容写法

```javascript
// 统一 API 命名
if (typeof browser === 'undefined') {
  var browser = chrome;
}

// 或者用 webextension-polyfill
import browser from 'webextension-polyfill';
browser.storage.local.set({ key: value });
```

### Firefox 对 MV2 的支持

Firefox 是目前**唯一**继续完整支持 MV2 的主流浏览器：

- `webRequest.blocking` 完全可用
- `background.page` 仍然支持
- Manifest V2 扩展持续可用到至少 2026 年底

### Firefox 差异清单

| 项目 | Chrome | Firefox |
|---|---|---|
| 开发工具 | chrome://extensions | about:debugging |
| DNR | 支持 | 支持（差异较大） |
| webRequest blocking | ❌ MV3 不可用 | ✅ 仍然可用 |
| Side Panel | ✅ (MV3) | ✅ (sidebar_action) |
| Service Worker | SW | Event Pages |
| 代码签名 | 自动 | 需要 AMO 签名 |

### Firefox 特定 API

```json
// manifest.json
"sidebar_action": {
  "default_title": "我的侧边栏",
  "default_panel": "sidebar.html"
}
```

---

## Safari

### 限制

Safari 的 Web Extension 是各大浏览器中**功能最少**的：

- API 支持度不到 Chrome 的一半
- 没有完整的 DNR 支持
- Storage API 有差异
- Service Worker 行为不同
- 需要 Apple Developer Program（$99/年）

### 转换工具

Safari 提供 [Xcode 转换工具](https://developer.apple.com/documentation/safariservices/safari_web_extensions/converting_a_web_extension_for_safari)，可以把 Chrome 扩展转成 Safari 版本。

但结果通常需要额外修改——很多 Chrome 特有 API 在 Safari 上缺失。

---

## 跨浏览器策略

### 方案一：Chrome 优先 + 逐步兼容

1. 先在 Chrome 上开发测试
2. 用 `webextension-polyfill` 统一 API
3. 在 Firefox Edge 上测试
4. 只有需要时才兼容 Safari

### 方案二：抽象出浏览器检测层

```javascript
const browserAPI = {
  storage: chrome.storage,
  runtime: chrome.runtime,
  tabs: chrome.tabs,
  ...(typeof browser !== 'undefined'
    ? { menus: browser.menus }  // Firefox 用 browser.menus，Chrome 用 chrome.contextMenus
    : { menus: chrome.contextMenus })
};
```

### 方案三：WXT 框架

WXT 支持配置多浏览器 target：

```typescript
// wxt.config.ts
export default defineConfig({
  browser: ['chrome', 'firefox', 'edge'],
  manifest: {
    permissions: ['storage'],
  }
});
```

WXT 会自动处理各个浏览器的 manifest 差异和 API 差异。

---

## 参考

- [Firefox Extension Workshop](https://extensionworkshop.com/)
- [Safari Web Extensions](https://developer.apple.com/documentation/safariservices/safari_web_extensions)
- [webextension-polyfill](https://github.com/mozilla/webextension-polyfill)
