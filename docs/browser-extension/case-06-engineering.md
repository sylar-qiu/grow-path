# 案例 06：工程化、数据流与架构总览

## 构建与工程化

项目使用 **no-build** 模式——所有 JS 文件是纯手写 ES5 兼容代码，通过 `IIFE`（立即执行函数）组织作用域。

```javascript
// 所有文件都是这种模式
(function () {
  'use strict';
  // ...

  // 不 export，通过 window 对象暴露给其他文件
  window.storage = storage;
})();
```

### 为什么不打包？

这个项目的特点是：
- Content Script 通过 manifest 的 `js` 数组加载多个文件
- 浏览器下载每个独立 JS 文件，IIFE 之间通过 `window.*` 共享
- 没有 Webpack、Vite 的构建过程

```json
// manifest.json
"js": ["lib/parser-utils.js", "lib/shared-utils.js",
       "src/lib/storage.js", "src/content/search-page.js"]
```

加载顺序就是依赖顺序：`parser-utils` → `shared-utils` → `storage` → `search-page`。

### 开发工具链

```json
// package.json
"scripts": {
  "lint": "eslint src/ lib/",
  "lint:fix": "eslint src/ lib/ --fix",
  "format": "prettier --write \"src/**/*.js\"",
  "test": "vitest run",
  "check": "node scripts/check-syntax.js"
}
```

| 工具 | 用途 |
|---|---|
| ESLint + @eslint/js | JS 代码规范 |
| Prettier | 格式化 |
| Stylelint | CSS 规范 |
| Vitest | 单元测试 |
| @types/chrome | Chrome API 类型提示（在 VS Code 中） |

### 函数调用图生成

项目有一个自定义脚本 `scripts/generate-fn-map.js`，扫描 `src/` 下所有 .js 文件，自动生成 `FUNCTIONS.md`，包含：

- 每个文件的函数列表（含行号）
- 消息 action 的发送/接收关系
- 文件内函数调用

这对理解项目数据结构特别有用。**但它的局限**：只分析文件内调用，不支持跨文件追踪和 import/export。

### 测试

```javascript
// __tests__/detail-page.test.js — 使用 vitest + jsdom
import { describe, it, expect } from 'vitest';
import { JSDOM } from 'jsdom';

describe('detail-page', () => {
  it('should extract basic info from DOM', () => {
    const dom = new JSDOM(`<div id="J_ItemInfo">...</div>`);
    // 测试数据提取函数
  });
});
```

测试覆盖了数据提取的核心逻辑，但 UI 交互（Side Panel 的 DOM 操作）没有测试。这是扩展测试的通病——UI 部分难以自动化。

---

## 完整数据流

```
┌─────────────────────────────────────────────────────────────────────┐
│ 淘宝服务器                                                          │
│  SSR HTML → 搜索结果页、详情页、店铺页                              │
└──────────┬──────────────────────────────────────────────┬──────────┘
           │ 加载页面                                       │ AJAX 数据
           ▼                                               ▼
┌────────────────────────┐                ┌──────────────────────────┐
│ Content Script         │                │ Content Script           │
│ search-page.js         │◄───────────────│ shop-search-page.js      │
│ detail-page.js         │  消息中转       │ sycm-page.js             │
│                        │                │                          │
│ 功能：                 │                │ 功能：                   │
│ 解析搜索列表 DOM        │                │ 解析店铺/分析页面 DOM     │
│ 管理浮动横条           │                │ 数据采集                 │
│ 维护品类内存状态       │                │                          │
└──────────┬────────────┘                └──────────┬───────────────┘
           │ sendMessage                              │ sendMessage
           ▼                                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Service Worker (background.js)                                     │
│                                                                    │
│ 功能：                                                             │
│ 消息路由 forwardToTab: CS ↔ Side Panel                            │
│ 触发下载 exportData / downloadXlsFile                              │
│ 初始化存储 onInstalled                                             │
└─────────────────────────────────────────────────────────────────────┘
           │ chrome.sidePanel.open / chrome.runtime.sendMessage
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Side Panel (sidebar 主界面)                                        │
│                                                                    │
│ 文件：                                                             │
│ sidepanel.js (3048行) — 主交互面板                                 │
│ shop-panel.js (1043行) — 店铺采集面板                              │
│ category-analytics.js (1092行) — 品类 AI 分析                      │
│                                                                    │
│ 功能：                                                             │
│ 品类管理（创建/删除/切换）                                         │
│ SPU 列表展示与操作                                                 │
│ 本地数据分析（SKU 词频、价格分布）                                 │
│ DeepSeek AI 分析                                                   │
│ CSV/XLS 导出                                                       │
└─────────────────────────────────────────────────────────────────────┘
           │ chrome.storage.local
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Storage Layer (storage.js / shop-storage.js)                       │
│                                                                    │
│ 两种存储：                                                         │
│ 1. 品类存储 (storage.js) — 按品类维度                              │
│    关键词(品类) → SPU 列表 → 详情元数据 → 详情体                   │
│ 2. 店铺存储 (shop-storage.js) — 按店铺维度                          │
│    shop → item → page → changes                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 分层架构

```
┌─────────────────────────────────────────┐
│ UI 层（Side Panel / Popup）              │
│ sidepanel.js + shop-panel.js            │
├─────────────────────────────────────────┤
│ 消息层                                   │
│ runtime.sendMessage / forwardToTab      │
├─────────────────────────────────────────┤
│ 业务逻辑层                                │
│ search-page.js / detail-page.js          │
│ category-analytics.js                    │
├─────────────────────────────────────────┤
│ 数据层                                   │
│ storage.js / shop-storage.js             │
├─────────────────────────────────────────┤
│ 基础设施                                  │
│ chrome.storage.local / chrome.storage    │
└─────────────────────────────────────────┘
```

这是典型的 MV3 扩展分层——层之间通过**消息传递**通信，不跨层直接调用函数。

## 与知识体系的对应

| 知识模块 | 项目中 | 状态 |
|---|---|---|
| Manifest V3 架构 | ✅ 完整实现 | 实践中 |
| Content Script | ✅ 4 个 CS 文件 | 实践中 |
| Service Worker | ✅ background.js | 实践中 |
| 存储层 | ✅ storage.js, shop-storage.js | 实践中 |
| UI 层 | ✅ Side Panel 为主 | 实践中 |
| 网络层 | ⚠️ 只用 host_permissions，未用 DNR | 待补充 |
| 构建与工程化 | ⚠️ 未用打包工具 | 待优化 |
| 调试与测试 | ✅ Vitest + generate-fn-map | 实践中 |
| DevTools 扩展 | ❌ 未使用 | 可扩展 |
| Native Messaging | ❌ 未使用 | 无需求 |
| 多浏览器兼容 | ❌ 仅 Chrome | 可扩展 |
| 发布 | ❌ 未上架 | 待决策 |

---

## 参考

- 源码：`Projects/taobao-plugin/extension/package.json`
- 源码：`Projects/taobao-plugin/extension/scripts/generate-fn-map.js`
- 知识体系：[构建与工程化](11-build-tooling.md)
