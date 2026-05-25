# 核心架构层

## Manifest V3 的核心变化

MV3 有三条根本性改变，理解它们就理解了整个架构设计：

### 1. Service Worker 替代后台页面

之前 `"persistent": true/false` 控制的后台页面，现在变成一个纯粹的事件驱动 Service Worker。详情在[Service Worker 层](03-service-worker.md)展开，这里只说架构层面的影响：

- 所有状态必须显式持久化，不能依赖内存变量
- 长时间操作需要拆分或使用 `alarms` 维持
- 需要 DOM 的场景必须用 Offscreen Document

### 2. 远程代码全面禁止

MV3 不允许执行远程代码（URL 或字符串传输的代码）：

```
// MV2 可以
chrome.tabs.executeScript(tabId, { file: "https://cdn/remote.js" })

// MV3 必须
chrome.scripting.executeScript(tabId, { files: ["bundled-content.js"] })
```

你的构建工具必须把**所有** JS 打包进扩展包。这个变化直接决定了工程化方案——你的 Vite/Webpack 必须正确处理多入口 bundle。

### 3. webRequest blocking 模式废除

`webRequest` 还能监听，但不能再拦截和修改请求。代替方案是 Declarative Net Request——声明式规则引擎，具体在[网络层](06-network.md)展开。

---

## 五大组件及其定位

```
                 ┌─────────────┐
                 │ Service     │  事件驱动，大脑
                 │ Worker      │  无 UI，无 DOM
                 └──────┬──────┘
                        │
         ┌──────────────┼──────────────┐
         │              │              │
    ┌────▼────┐  ┌─────▼─────┐  ┌─────▼──────┐
    │ Popup   │  │ Content   │  │ Side Panel │
    │ 点击弹出 │  │ Script    │  │侧边栏(MV3) │
    │ 瞬态页面 │  │页面内注入  │  │可常驻      │
    └─────────┘  └───────────┘  └────────────┘
```

五个组件各自不同的生命周期决定了你怎么设计数据流。

### Service Worker
- 唯一长时间存活（但有休眠）的后端组件
- 不能直接访问 DOM，不能调用 window/document
- 事件驱动——只在需要时运行

### Popup
- 用户点击图标时创建，点击其他地方就销毁
- 每次打开都是全新的页面实例
- 适合简短交互（开关、状态查看），不适合复杂操作
- 与 SW 通信必须通过消息传递

### Content Script
- 注入到目标页面，运行在 Isolated World
- 能访问页面 DOM，但不能访问页面 JS 变量（除非 inject Main World）
- 生命周期受目标页面影响——页面刷新/关闭时销毁

### Side Panel
- MV3 新增，比 Popup 更持久
- 可以随浏览器一直存在，不随点击消失
- 适合长任务、AI 对话面板、需要保持状态的工具

### Options Page
- 扩展设置页面
- 支持 `open_options_page` API 和 `chrome.runtime.openOptionsPage()`
- 在单独标签页或新窗口中打开

---

## 消息传递

### 路径图

```
Service Worker ←→ Popup
     ↕                    ↕
Content Script ←→ 目标页面 (Main World)
```

每个箭头都是 `sendMessage/onMessage`。两边都在时也可以用长连接 `connect/onConnect`。

### 单向 vs 长连接

| | sendMessage | connect |
|---|---|---|
| 连接 | 一次性，发完结束 | 持久通道 |
| 适合 | 查询状态、触发动作 | 持续数据流、订阅 |
| 资源 | 无开销 | 需要管理 disconnect |
| 休眠 | 能唤醒 SW | 能唤醒 SW |

### 消息安全

- 所有收到的消息都要校验 `sender`——不要假设谁发的
- `sender.tab` 可以判断是不是来自你的 CS
- `sender.id` 是你的扩展 ID

---

## 隔离模型

关键概念：Content Script 默认**不能**访问页面自己的 JS 变量。

```
Isolated World (你的 CS)          Main World (页面 JS)
┌─────────────────────┐      ┌──────────────────────┐
│ DOM ✅              │      │ DOM ✅               │
│ window.xxx ❌       │◄────►│ window.xxx ✅        │
│ chrome.* ✅         │      │ chrome.* ❌          │
│ 你的全局变量 ✅     │      │ 页面的全局变量 ✅     │
└─────────────────────┘      └──────────────────────┘
```

你需要操作页面 JS 时，用 Script Injection：

```javascript
// CS 中注入 Main World 脚本
const script = document.createElement('script');
script.src = chrome.runtime.getURL('inject.js');
document.documentElement.appendChild(script);
script.remove();
```

这个脚本就能访问页面所有 JS 变量了，但不再有 `chrome.*` API。

---

## 参考

- [Chrome Extension Manifest V3](https://developer.chrome.com/docs/extensions/mv3/intro)
- [Migration guide: MV2 to MV3](https://developer.chrome.com/docs/extensions/mv3/mv2-migration/)
