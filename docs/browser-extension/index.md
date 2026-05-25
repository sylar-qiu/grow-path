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

### 一、核心架构层

#### 1.1 Manifest V3 规范

- `manifest.json` 所有字段详解
- MV3 vs MV2 结构性差异
- 权限模型（`host_permissions` / `permissions` 分离）

#### 1.2 五大组件

| 组件 | 说明 |
|------|------|
| Service Worker | 替代 Background Page，事件驱动，非持久化 |
| Popup | 点击图标弹出，瞬态 HTML 页面 |
| Content Script | 注入目标页面，运行在 Isolated World |
| Options Page | 扩展设置页 |
| Side Panel | MV3 新增，浏览器侧边栏 |

#### 1.3 消息传递

- **单向**：`sendMessage` / `onMessage`
- **长连接**：`connect` / `onConnect` / `postMessage` / `onDisconnect`
- **跨组件通信路径**：SW ↔ Popup ↔ Content Script ↔ 页面
- 消息安全与校验

#### 1.4 生命周期

- 安装事件（`onInstalled: install / update / update_permissions`）
- Service Worker 休眠与唤醒机制
- Popup 瞬态生命周期
- Content Script 注入时机（`document_idle / start / end`）

#### 1.5 隔离模型

- Content Script 的 Isolated World
- Main World 注入（通过 inject script 进入页面 JS 上下文）
- Service Worker 的非 DOM 环境
- Iframe / Shadow DOM 隔离

---

### 二、内容脚本层

#### 2.1 注入方式

- 声明式：`manifest content_scripts` + `matches`
- 程序化：`chrome.scripting.executeScript` / `registerContentScripts`
- 动态注入（MV3 `user_scripts` 限制）

#### 2.2 与页面交互

- 读取 / 操作页面 DOM
- 监听页面事件
- 注入自定义 UI（覆盖层、按钮、表单）
- 样式隔离：Shadow DOM 场景

#### 2.3 页面内脚本协同

- `postMessage` / `CustomEvent` 双向通信
- 注入 script 标签到 Main World
- `window` 属性共享 vs 隔离

---

### 三、Service Worker 层

#### 3.1 事件驱动模型

- 能唤醒 SW 的事件：`onInstalled` / `onMessage` / `alarms` / `webNavigation`
- 30 秒事件超时 + 5 分钟空闲休眠
- `persistent: false` 不可逆

#### 3.2 绕过限制

- `chrome.alarms` 定期唤醒
- Offscreen Document（需要 DOM 时的临时方案）
- 长时间运行任务拆分
- 存储状态恢复模式

#### 3.3 SW 能力边界

| | |
|---|---|
| ✅ 能 | 所有 extension API、fetch、Cache、IndexedDB |
| ❌ 不能 | DOM、window/document、setTimeout 过夜 |
| ⚠️ 有条件 | WebSocket（需 keepalive 策略） |

---

### 四、存储层

#### 4.1 chrome.storage

| 类型 | 说明 | 配额 |
|------|------|------|
| `local` | 本地持久，无上限 | 无上限（需申请） |
| `sync` | 跨设备同步 | 102KB / item |
| `session` | 内存级，MV3 新增 | 仅当前会话 |
| `managed` | 企业策略控制 | 只读 |

#### 4.2 浏览器内置存储

- IndexedDB（SW 中可用）
- Cache API（配合请求缓存）
- Cookie API

#### 4.3 存储选型决策

```
配置项 → chrome.storage.sync
大量本地数据 → IndexedDB
缓存响应 → Cache API
会话间暂存 → chrome.storage.session
```

---

### 五、UI 层

- **Action**：图标 / Badge / Tooltip / 弹窗 / 禁用状态
- **Popup**：标准 Web 页面，与 SW / Content Script 通信
- **Side Panel**：多标签隔离 vs 共享，与 Popup 的定位差异
- **Options Page** / Override Pages（新标签页 / 历史 / 书签）
- **其他**：Context Menus / Omnibox / Commands / Notifications

---

### 六、网络层

#### 6.1 Declarative Net Request (DNR)

- 静态规则 vs 动态规则
- 规则语法：URL filter、条件、动作
- 配额限制与优化策略
- MV2 `webRequest` → DNR 等价转换

#### 6.2 webRequest（MV3 受限）

- 保留观察模式（observer）
- `blocking` 模式不再支持
- DNR 覆盖不了的场景

#### 6.3 跨域与安全

- 扩展特有的跨域权限
- CSP 配置
- 沙箱页面

---

### 七、浏览器 API 全景

- 标签页与窗口：`tabs` / `windows`
- 下载 / 剪贴板 / 书签 / 历史 / 密码
- 屏幕捕获：`desktopCapture` / `tabCapture`
- 打印 / 企业策略 / 无障碍 / 系统

---

### 八、DevTools 扩展

- `devtools_page` 入口
- 创建面板 / 侧边栏
- 审查元素 + 获取页面信息
- 与主扩展通信

---

### 九、原生通信

#### 9.1 Native Messaging

- 注册 Native Host
- 通信协议（stdin/stdout JSON 消息）
- 安全模型

#### 9.2 本机功能集成

- 执行本地命令
- 调用本机 API
- 文件系统访问

---

### 十、构建与工程化

- 打包工具选型（Vite / Webpack / Rollup / WXT）
- 多入口配置（SW / Popup / CS / Side Panel / Options）
- TypeScript 最佳实践
- 环境变量与构建模式
- 热重载方案
- 推荐项目目录结构

---

### 十一、调试与测试

- Service Worker 调试（`chrome://serviceworker-internals` / DevTools）
- 日志追踪
- 错误报告与崩溃恢复
- 自动化测试（Puppeteer / Playwright）

---

### 十二、发布与运维

- Chrome Web Store 上架流程
- 隐私政策与数据声明
- 版本管理与强制更新
- 国际化 / 本地化

---

### 十三、多浏览器兼容

| 浏览器 | 要点 |
|--------|------|
| Edge | ≈ Chrome，微小差异 |
| Firefox | MV2 持续支持，`webRequest` 保留 |
| Safari | Web Extension 阉割版，大量 API 不支持 |

---

*内容持续建设中。每完成一个主题会添加链接到具体文章。*
