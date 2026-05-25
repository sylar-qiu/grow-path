# 构建与工程化

## 多入口打包

扩展与其他 Web 项目的根本区别：**有多个独立入口**。每个组件都是一套独立的 HTML/CSS/JS。

```
SW 入口          → background.js
Popup 入口       → popup.html + popup.js
Content Script   → content.js（独立 bundle）
Side Panel       → sidepanel.html + sidepanel.js
Options          → options.html + options.js
DevTools         → devtools.html + devtools.js
```

每个入口之间不能 `import` 对方的模块——它们运行在不同的上下文中。

## 打包工具选型

### Vite

适合扩展开发，但原生不支持多入口 HTML：

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  build: {
    rollupOptions: {
      input: {
        background: resolve(__dirname, 'src/background/index.ts'),
        popup: resolve(__dirname, 'src/popup/index.html'),
        content: resolve(__dirname, 'src/content/index.ts'),
        sidepanel: resolve(__dirname, 'src/sidepanel/index.html'),
      },
      output: {
        entryFileNames: '[name].js',
      }
    },
    outDir: 'dist'
  }
});
```

注意：Vite 默认对 HTML 入口会处理资源路径。Content Script 和 Service Worker 是纯 JS，需要用 `build.rollupOptions.input` 直接指定。

### Webpack

成熟方案，社区插件丰富。

```javascript
// webpack.config.js
const CopyPlugin = require('copy-webpack-plugin');

module.exports = {
  entry: {
    background: './src/background/index.ts',
    popup: './src/popup/index.tsx',
    content: './src/content/index.ts',
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].js'
  },
  plugins: [
    new CopyPlugin({
      patterns: [
        { from: 'public/manifest.json', to: 'manifest.json' },
      ],
    }),
  ]
};
```

### WXT（推荐）

专门为浏览器扩展设计的框架，基于 Vite，开箱支持：

- 多入口管理
- manifest 自动生成
- HMR 热重载
- 多浏览器打包

```typescript
// wxt.config.ts
import { defineConfig } from 'wxt';

export default defineConfig({
  manifest: {
    name: '我的扩展',
    permissions: ['storage', 'tabs'],
  }
});
```

WXT 内部做得最多的事情正是"让不同入口各归其位"，如果你是新项目，推荐直接上 WXT。

---

## TypeScript

扩展开发中 TS 的价值：

- `chrome.*` API 类型提示（`@types/chrome`）
- 跨组件通信的消息类型定义
- 消息 Payload 的类型安全

### 消息类型定义

```typescript
// types/messages.ts
export type ExtensionMessage =
  | { type: 'FETCH_DATA'; url: string; method: 'GET' | 'POST' }
  | { type: 'UPDATE_BADGE'; count: number }
  | { type: 'SIDE_PANEL_TOGGLE' };

export type ContentMessage =
  | { type: 'PAGE_DATA'; title: string; url: string };
```

### 在消息传递中使用

```typescript
// SW
chrome.runtime.onMessage.addListener(
  (message: ExtensionMessage, sender, sendResponse) => {
    if (message.type === 'FETCH_DATA') {
      // message.url 和 message.method 有类型提示
    }
  }
);
```

---

## 目录结构推荐

参考你的项目结构：

```
chrome-extension/
├── src/
│   ├── background/
│   │   └── index.ts          # SW 入口
│   ├── content/
│   │   └── index.ts          # Content Script
│   ├── popup/
│   │   ├── index.html
│   │   └── main.ts
│   ├── sidepanel/
│   │   ├── index.html
│   │   └── main.ts
│   ├── options/
│   │   ├── index.html
│   │   └── main.ts
│   ├── assets/
│   │   └── icons/
│   └── shared/
│       ├── types.ts          # 共享类型
│       └── utils.ts          # 共享工具函数
├── dist/                     # 构建输出
├── manifest.config.ts        # manifest.json 模板
├── vite.config.ts
├── tsconfig.json
└── package.json
```

---

## 热重载

三种实现方式：

### 1. WXT 内置

开箱支持，无需配置。

### 2. web-ext (Mozilla 官方工具)

```bash
npx web-ext run --source-dir ./dist
```

支持文件变更自动重载。

### 3. 手动实现

```javascript
// dev-background.js (非 SW 方案，仅 MV2 参考)
// MV3 中需要在开发时保持 SW 活跃
chrome.runtime.onInstalled.addListener(() => {
  // 监听文件变更，通知扩展重载
});
```

---

## 参考

- [WXT Framework](https://wxt.dev/)
- [Vite Extension Plugin](https://vite.dev/)
- [Webpack Extension Template](https://github.com/awesome-web-extensions/webpack-extension-template)
