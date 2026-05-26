# 浏览器扩展

## 完整知识体系

```

├── 一、核心架构层
├── 二、内容脚本层
├── 三、Service Worker 层
├── 四、存储层
├── 五、UI 层
├── 六、网络层
├── 七、浏览器 API 全景
├── 八、DevTools 扩展
├── 九、原生通信
├── 十、构建与工程化
├── 十一、调试与测试
├── 十二、发布与运维
└── 十三、多浏览器兼容
```

---

| # | 模块 | 文章 |
|---|------|------|
| 01 | Manifest 字段详解 | [manifest.json 全部字段逐项解析、含义与实践](01-manifest-reference.md) |
| 02 | 核心架构层 | [Manifest V3、五大组件、消息传递、生命周期、隔离模型](02-core-architecture.md) |
| 03 | 内容脚本层 | [注入方式、页面交互、Script Injection、Main World 通信](03-content-scripts.md) |
| 04 | Service Worker 层 | [事件驱动、休眠与超时、状态管理、Offscreen Document](04-service-worker.md) |
| 05 | 存储层 | [chrome.storage 四种方案、IndexedDB、Cache API、选型决策](05-storage.md) |
| 06 | UI 层 | [Action/Popup/Side Panel/Options、Context Menu、Omnibox、Commands](06-ui-layer.md) |
| 07 | 网络层 | [DNR 静态/动态规则、webRequest 观察模式、CSP、Sandbox](07-network.md) |
| 08 | 浏览器 API 全景 | [Tabs/Downloads/Clipboard/ScreenCapture/Idle/System 等](08-api-panorama.md) |
| 09 | DevTools 扩展 | [F12 面板开发、InspectedWindow、与主扩展通信](09-devtools.md) |
| 10 | 原生通信 | [Native Messaging 协议、Host 注册、适用场景](10-native-messaging.md) |
| 11 | 构建与工程化 | [Vite / Webpack / WXT 配置、多入口、TypeScript](11-build-tooling.md) |
| 12 | 调试与测试 | [SW/Popup/CS 调试技巧、日志策略、Puppeteer 测试](12-debugging-testing.md) |
| 13 | 发布与运维 | [CWS 上架、审核要点、版本迁移、国际化](13-publishing.md) |
| 14 | 多浏览器兼容 | [Edge / Firefox / Safari 差异与兼容策略](14-cross-browser.md) |

---

*浏览器扩展内容已完成。下一方向：架构师之路。*

## 实战案例

结合 `~/Projects/taobao-plugin/` 实际项目的代码讲解：

| # | 案例 | 对应模块 |
|---|------|---------|
| 01 | [架构与 Manifest](case-01-architecture.md) | 核心架构层 |
| 02 | [内容脚本实战](case-02-content-scripts.md) | 内容脚本层 |
| 03 | [Service Worker 实战](case-03-service-worker.md) | Service Worker 层 |
| 04 | [存储层实战](case-04-storage.md) | 存储层 |
| 05 | [UI 层实战](case-05-ui-layer.md) | UI 层 |
| 06 | [工程化与数据流](case-06-engineering.md) | 构建与工程化 / 数据流 |
