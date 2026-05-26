# Manifest.json 字段详解

## 开篇：manifest.json 是什么

manifest.json 不是普通的配置文件，它在 Chrome 扩展体系中承担三个角色：

**身份证明** — 告诉浏览器你是谁（扩展名/版本/ID）。没有 manifest，一个文件夹只是文件夹。有了它，浏览器才识别出"这是一个扩展"。

**权限声明** — 你声明要访问哪些数据、哪些网站、哪些浏览器 API。Chrome 在安装时把这些声明展示给用户看，用户同意后才能使用。

**入口清单** — 告诉浏览器你的扩展有哪些组件（Service Worker / Content Script / Popup / Side Panel 等），各自对应什么文件、在什么条件下激活。

这三个角色决定了 manifest.json 的设计哲学：**声明式，而非命令式**。你只声明"我要什么"和"我有什么"，浏览器自己决定怎么加载、怎么运行、怎么销毁。

---

## 一、基础元数据

### manifest_version

```json
"manifest_version": 3
```

当前只有两个有效值：

| 值 | 状态 | 说明 |
|---|---|---|
| 2 | 已弃用 | 2023 年起 Chrome 停止接受新 MV2 扩展，2024 年起逐步禁用现有 MV2 |
| 3 | 当前版本 | 2019 年推出，2022 年强制新提交使用 MV3 |

MV2 和 MV3 本质上是两套不同的扩展运行时。MV3 的核心变化：Service Worker 替代后台页面、远程代码全面禁止、webRequest blocking 模式废除。这三个变化影响了几乎所有扩展的架构设计。

**实践中**：2026 年的今天，直接写 `3`。不需要纠结兼容性。

---

### name / short_name

```json
"name": "淘宝数据采集助手",
"short_name": "采集助手"
```

- `name`：在扩展管理页面（`chrome://extensions`）、商店列表、权限提示中展示的名称。最长 45 字符。
- `short_name`：可选字段。工具栏图标上空间不够时的备选名（最多 12 字符，推荐英文）。大部分情况下不需要设置。

两者都不支持本地化——如果要根据用户语言显示不同名称，需要用 `default_locale` + `_locales/` 目录下的 `messages.json`。

---

### version / version_name

```json
"version": "0.4.0",
"version_name": "0.4.0-beta"
```

- `version`：**必填**。由 1-4 个以点分隔的整数组成（如 `"1"`、`"1.2"`、`"1.2.3"`、`"1.2.3.4"`）。每个整数最大 65535。Chrome 自动更新时，比较 `version` 字段判断是否需要更新。
- `version_name`：可选。给用户看的版本名称，可以是任意字符串。不影响自动更新判断，仅用于展示。

**实践中**：建议使用语义化版本（semver）格式 `major.minor.patch`。Chrome Web Store 不允许降版本号，发布前确认版本号已递增。

---

### description

```json
"description": "淘宝搜索结果页品类分析工具——采集商品数据、按品类管理分析"
```

最长 132 字符。这行文字会出现在 Chrome Web Store 的搜索结果和扩展管理页面。用户看到 permissions 警告时也会看到这个描述。

**实践中**：不要在 description 里堆砌关键词做 SEO。Chrome Web Store 会拒绝夸大或不实的描述。

---

### icons

```json
"icons": {
  "16": "icons/icon16.png",
  "48": "icons/icon48.png",
  "128": "icons/icon128.png"
}
```

不同尺寸用于不同场景：

| 尺寸 | 用途 |
|---|---|
| 16x16 | 扩展管理页面、favicon 位置 |
| 32x32 | Windows 任务栏（可选，但推荐） |
| 48x48 | 扩展管理页面 |
| 128x128 | Chrome Web Store 列表、安装时展示 |

至少需要 48px 和 128px。PNG 格式，支持透明背景。

**一个常见陷阱**：许多开发者只设置 `icons` 但忘记给 `action.default_icon` 设置相同图标。结果是工具栏图标在某些场景下显示空白方块。

---

### default_locale

```json
"default_locale": "zh_CN"
```

只有当你需要国际化（i18n）时才需要这个字段。声明后，Chrome 要求你提供 `_locales/` 目录下的翻译文件。`default_locale` 对应的语言文件作为兜底——当用户的语言没有对应翻译时显示这个。

**即使你只做一个语言，只要用了 `chrome.i18n.getMessage()`，就必须声明 `default_locale`。**

---

### minimum_chrome_version

```json
"minimum_chrome_version": "116"
```

如果你的扩展依赖某个 Chrome 版本才有的 API（如 MV3 的 `sidePanel` 在 Chrome 114 才稳定），用这个字段阻止用户在旧版本上安装。

**不要滥用**。设置过高的版本号会剔除大量用户。只有当你确定用到了某个版本专属的 API 时才设置。

---

## 二、权限体系

Chrome 扩展的权限系统是安全模型的核心。每一行权限声明都在告诉用户："这个扩展有权做这些事"。

### permissions（普通权限）

```json
"permissions": [
  "storage",
  "activeTab",
  "scripting",
  "tabs",
  "windows",
  "downloads",
  "sidePanel"
]
```

普通权限是指那些不涉及具体网站域名的 API 访问能力。常见的权限及触发警告情况：

| 权限 | 用途 | 安装时警告 |
|---|---|---|
| `storage` | 读写 `chrome.storage.local/sync/session/managed` | 否 |
| `activeTab` | 临时获得当前活动标签页的权限 | 否 |
| `scripting` | 动态注入脚本（`chrome.scripting.executeScript`） | 否 |
| `tabs` | 操作标签页（查询/创建/更新/移动/分组） | 否 |
| `alarms` | 定时任务 | 否 |
| `downloads` | 下载文件 | 否 |
| `sidePanel` | 使用侧边栏 | 否 |
| `contextMenus` | 右键菜单 | 否 |
| `notifications` | 桌面通知 | 否 |
| `tts` | 文字转语音 | 否 |
| `bookmarks` | 读写书签 | **是**（"读取和修改书签"） |
| `cookies` | 读写 Cookie | **是**（"读取和修改你的浏览数据"） |
| `history` | 读写浏览历史 | **是**（"读取和修改浏览历史"） |
| `tabs` + `host_permissions` | 读取标签页 URL | **是**（取决于 host 范围） |

**关键原则：权限最小化**。只声明你真正需要的。`activeTab` 比 `<all_urls>` + `tabs` 好一百倍。

---

### host_permissions（主机权限）

```json
"host_permissions": [
  "https://*.taobao.com/*",
  "https://*.tmall.com/*",
  "https://sycm.taobao.com/*",
  "https://api.deepseek.com/*"
]
```

MV3 把主机权限从 `permissions` 中拆成了独立的 `host_permissions` 字段。这是 MV2 到 MV3 的**精妙改进**：普通权限和主机权限分开声明，权限请求的粒度更清晰。

格式规则（和 Content Script 的 `matches` 模式一样）：

| 模式 | 匹配范围 |
|---|---|
| `*://*/*` | 所有 URL（http/https/ws/wss） |
| `<all_urls>` | 同上，等效写法 |
| `https://*.taobao.com/*` | taobao.com 及其所有子域名 |
| `https://api.deepseek.com/*` | 指定域名下所有路径 |
| `https://taobao.com/` | 仅首页（不匹配子路径） |

**关键区别**：

| | permissions 中的 API | host_permissions |
|---|---|---|
| 安装时展示 | "访问 api.deepseek.com 上的数据" |
| 用户能撤销 | 否，只能禁用扩展 | 是，可以在扩展管理页面上逐站关闭 |
| 范围 | 特定 API 功能 | 特定网站的 DOM/Cookie/Network |

**实践中**：永远用最窄的匹配模式。`https://*.taobao.com/*` 比 `<all_urls>` 更安全，也更容易通过 Chrome Web Store 审核。

---

### optional_permissions / optional_host_permissions

```json
"optional_permissions": ["clipboardRead", "clipboardWrite"]
```

安装时不请求，运行中通过 `chrome.permissions.request()` 向用户动态申请。用户授予后生效，也可以随时在扩展管理页面撤销。

适合场景：
- 扩展的进阶功能需要更高权限（如 AI 分析功能需要 `clipboardRead`）
- 用户需要自行决定是否信任某些权限
- 大部分用户用不到的功能，不应该让全部用户承担权限警告

**使用流程**：
```
用户打开高级功能 → 扩展检查权限 → 没有权限 → 弹窗申请
→ 用户同意 → 授权成功 → 功能可用
→ 用户拒绝 → 降级到基础功能
```

---

## 三、内容脚本（content_scripts）

```json
"content_scripts": [
  {
    "matches": ["https://s.taobao.com/*", "https://*.taobao.com/search*"],
    "js": ["lib/parser-utils.js", "src/lib/storage.js", "src/content/search-page.js"],
    "run_at": "document_end"
  }
]
```

内容脚本是注入到目标页面的 JavaScript。每个声明块包含：

### matches（必填）

指定在哪些 URL 上注入脚本。格式与 host_permissions 相同：
- `https://s.taobao.com/*` — 精确到子域名
- `https://*.taobao.com/search*` — 子域名下以 /search 开头的路径
- `<all_urls>` — 所有页面（极少场景使用，滥用会触发权限警告）

**注意**：matches 模式中不能使用 IP 地址（localhost 除外）。

### exclude_matches

```json
"exclude_matches": ["https://*.taobao.com/admin/*"]
```

排除不需要注入的页面。优先级高于 `matches`。

### js / css

注入的脚本和样式文件列表。按数组顺序依次执行。

**一个常见陷阱**：MV3 中 content_scripts 的 `js` 和 `css` 文件路径是相对于扩展根目录的，**不是**相对于 manifest 文件所在目录的。也就是说，不要把 `src/content/search-page.js` 写成 `../content/search-page.js`。

### run_at

脚本注入的时机：

| 值 | 时机 | 适用场景 |
|---|---|---|
| `document_start` | DOM 构建之前（但仍可访问 DOM API） | 需要在页面加载最早阶段拦截/修改 |
| `document_end` | DOM 已就绪，但图片/资源可能还在加载 | **推荐**。绝大多数场景用这个 |
| `document_idle` | 页面加载完成后（浏览器空闲时） | 低优先级操作，不影响页面加载 |

**实践中**：除非你有明确理由，否则都用 `document_end`。`document_start` 在你需要修改页面最早的行为（如重写全局函数）时有用。`document_idle` 适合低优先级的 UI 增强。

### world

**MV3 新增**。控制脚本运行在哪个 JavaScript 执行环境：

- `"ISOLATED"`（默认）：Content Script 只能访问 DOM，不能访问页面自己的 JS 变量
- `"MAIN"`：脚本运行在页面的主世界，可以访问页面 JS 变量，但不能使用 `chrome.*` API

```json
{
  "matches": ["https://*.taobao.com/*"],
  "js": ["src/content/bridge.js"],
  "world": "MAIN"
}
```

**实践中**：大部分场景用默认的 ISOLATED 即可。只有当你需要劫持或调用页面已有的 JS 函数时，才需要 MAIN world。

### match_about_blank / match_origin_as_fallback

```json
"match_about_blank": true
```

控制是否在 `about:blank`、`about:srcdoc` 等特殊页面（如 iframe 内容）上注入脚本。默认 false。

---

## 四、后台入口（background）

```json
"background": {
  "service_worker": "src/background/background.js"
}
```

### service_worker

MV3 的 Service Worker 替代了 MV2 的后台页面。一个扩展只有一个 Service Worker。

**关键特性**：
- 事件驱动：没有事件时不运行，不消耗内存
- 约 30 秒不活动后自动休眠
- 休眠后被事件（`onMessage`、`onAlarm`、`onInstalled` 等）唤醒
- 全局变量在休眠后丢失——**必须显式持久化所有状态**

```json
// ❌ 错误写法——休眠后 count=undefined
let count = 0;
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  count++;
});

// ✅ 正确写法——从 storage 读取
chrome.runtime.onMessage.addListener(async (msg, sender, sendResponse) => {
  const { count } = await chrome.storage.local.get('count');
  await chrome.storage.local.set({ count: (count || 0) + 1 });
});
```

### type: "module"

```json
"background": {
  "service_worker": "src/background/background.js",
  "type": "module"
}
```

当你的 SW 文件使用 `import`/`export`（ES Module）时需要指定 `"type": "module"`。不加的话，Chrome 会把 SW 文件当作普通脚本解析，遇到 `import` 会报错。

**实践中**：如果你用 WXT / Vite 等构建工具，它们通常会自动处理 SW 的 module 声明。如果你手写 SW（如 taobao-plugin 的 239 行无构建 SW），用 CommonJS 风格的 message handler，不需要 `type: "module"`。

### MV2 兼容（了解即可）

```json
// MV2 写法，仅用于参考已弃用的代码
"background": {
  "scripts": ["background.js"],
  "persistent": false  // true=常驻后台页面，false=事件页面
}
```

MV3 不存在 `persistent` 或 `scripts`（数组）了。

---

## 五、用户界面

### action（工具栏按钮）

```json
"action": {
  "default_title": "淘宝品类分析",
  "default_icon": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

在 MV3 中，`browser_action` 和 `page_action` 合并为 `action`。控制扩展图标在工具栏上的行为。

| 子字段 | 作用 |
|---|---|
| `default_title` | 鼠标悬停时显示的文字 |
| `default_icon` | 图标（16/32/48/128） |
| `default_popup` | **点击图标时弹出的 HTML 页面** |

#### 有 Popup vs 无 Popup

```json
// 有 Popup——点击自动弹窗
"action": {
  "default_popup": "src/popup/popup.html"
}

// 无 Popup——点击触发 chrome.action.onClicked 事件
// taobao-plugin 就是这个模式
"action": {
  "default_title": "淘宝品类分析",
  "default_icon": { ... }
  // 没有 default_popup！
}
```

**这是一个重要的架构决策**：
- 有 Popup → 点击弹出临时页面，适合短暂交互（开关、查看状态）。关闭即销毁。
- 无 Popup → 在 SW 中监听 `chrome.action.onClicked`，适合打开 Side Panel 或新标签页。交互不随点击消失。

#### Badge

```json
// 通过 chrome.action.setBadgeText / setBadgeBackgroundColor 设置
chrome.action.setBadgeText({ text: "42" });
chrome.action.setBadgeBackgroundColor({ color: "#FF0000" });
```

工具栏图标右上角的角标文字。适合显示计数（未读消息数、待处理任务数等）。气泡颜色和文字均可通过代码控制。

---

### side_panel（侧边栏）

**MV3 新增**。

```json
"side_panel": {
  "default_path": "src/sidepanel/sidepanel.html"
}
```

Side Panel 是 MV3 最实用的新特性之一。与 Popup 的关键区别：

| | Popup | Side Panel |
|---|---|---|
| 生命周期 | 点击出现，失焦消失 | 手动打开/关闭，可常驻 |
| 内容复杂度 | 适合简单交互 | 适合复杂/耗时的操作 |
| 打开方式 | 点击图标 | 点击图标或快捷键 |
| 同时性 | 用户可同时做其他事 | 用户可同时做其他事（侧栏不抢占焦点） |

**典型的架构分工**：Popup 做快捷入口/开关控制，Side Panel 做主工作区。taobao-plugin 的 3000+ 行代码全在 Side Panel 里，Popup 仅 130 行作为辅助入口。

---

### options_ui（设置页面）

```json
"options_ui": {
  "page": "src/options/options.html",
  "open_in_tab": true
}
```

用户右键扩展图标 → "选项" 时打开的页面。`open_in_tab: true` 在新标签页打开（适合复杂的设置界面），`false` 则在弹窗中打开（适合简短设置）。

---

### commands（快捷键）

```json
"commands": {
  "_execute_action": {
    "suggested_key": {
      "default": "Ctrl+Shift+Y",
      "mac": "Command+Shift+Y"
    }
  },
  "open-side-panel": {
    "suggested_key": {
      "default": "Alt+Shift+T"
    },
    "description": "打开侧边栏"
  }
}
```

- `_execute_action`：触发 action.onClicked 的快捷键（特殊预定义命令）
- `_execute_side_panel_action`：打开/关闭 Side Panel（MV3）  
- 自定义命令：在 SW 中通过 `chrome.commands.onCommand` 监听

用户可以覆盖你建议的快捷键。

---

## 六、网络与安全

### web_accessible_resources

```json
"web_accessible_resources": [{
  "resources": ["icons/*", "src/sidepanel/ai-report-print.css"],
  "matches": ["<all_urls>"]
}]
```

指定哪些资源可以被网页访问。不声明的资源，网页通过 `chrome-extension://` URL 访问会返回 404。

**注意**：`matches` 指定哪些**网站**可以访问这些资源，不是为了匹配生效页面。`<all_urls>` 表示任何网站都可以访问。

```json
// 更精确的写法——只允许 taobao.com 页面访问
"web_accessible_resources": [{
  "resources": ["icons/*"],
  "matches": ["https://*.taobao.com/*"]
}]
```

**use_dynamic_url**（MV3）：

```json
"web_accessible_resources": [{
  "resources": ["assets/*"],
  "matches": ["<all_urls>"],
  "use_dynamic_url": true
}]
```

`use_dynamic_url: true` 时，每次安装/更新生成不同的随机 URL，防止其他网站硬编码资源 URL。这是安全增强机制。

---

### content_security_policy

MV3 中使用 `content_security_policy` 不再是一个字符串，而是结构化的 `extension_pages` 和 `sandbox` 字段：

```json
"content_security_policy": {
  "extension_pages": "script-src 'self'; object-src 'self'",
  "sandbox": "sandbox allow-scripts allow-forms; script-src 'self'"
}
```

MV3 默认 CSP：`script-src 'self'; object-src 'self'`（不允许 `'unsafe-eval'`）。

**这是 MV3 的硬限制**——你不能因为图方便就放开 CSP 允许 `eval()`。如果你的代码使用 `eval()` 或 `new Function()`，需要重构。这也是为什么 MV3 禁止远程代码的原因——远程代码 = 无法在构建时静态分析 = 安全风险。

---

### sandbox

```json
"sandbox": {
  "pages": ["src/sandbox/export.html"]
}
```

默认情况下，扩展页面运行在严格 CSP 下。如果你需要在扩展中使用 `eval()` 或 `new Function()`（比如用户自定义表达式计算），需要把相关页面声明为 sandbox 页面。沙盒页面失去了 `chrome.*` API 访问权限，需要通过 postMessage 与父页面通信。

---

## 七、扩展通信

### externally_connectable

```json
"externally_connectable": {
  "ids": ["*"],
  "matches": ["https://*.taobao.com/*"]
}
```

控制哪些**其他扩展**和**网页**可以给你的扩展发送消息。不声明这个字段时，只有你的扩展内部可以通信。

| 字段 | 作用 |
|---|---|
| `ids` | 允许哪些扩展 ID 连接。`"*"` 表示所有扩展 |
| `matches` | 允许哪些网站通过 `chrome.runtime.connect` 或 `sendMessage` 调用你的扩展 |

**关键安全场景**：如果你在 Content Script 中通过 `window.postMessage` 与 Main World 通信，那任何页面脚本都可以发消息给 Content Script。`externally_connectable` 无法防范这个——你需要自己在消息 handler 中校验来源。

---

### update_url

```json
"update_url": "https://your-server.com/updates.xml"
```

Chrome Web Store 上的扩展使用商店的自动更新，不需要这个字段。只有通过开发者模式或企业策略侧载（side-loading）的扩展才需要自建更新服务器。

实践中，绝大多数场景不需要这个字段。

---

### key

```json
"key": "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."
```

用于固定扩展的公钥。扩展 ID 是由公钥生成的。不指定 `key` 时，Chrome 为每个扩展生成随机 ID。`key` 字段确保**不同电脑/不同构建方式**下生成相同的扩展 ID。

**当你需要**：开发者模式加载的扩展需要固定 ID（比如 OAuth 重定向 URL 需要扩展 ID）。发布到 CWS 后，由商店签名，ID 自然固定。

---

## 八、高级功能 & 冷门字段

### devtools_page

```json
"devtools_page": "src/devtools/index.html"
```

注册一个 F12 开发者工具面板。独立的 HTML 页面，有独立的开发工具窗口生命周期。不在 DevTools 中不会加载，不消耗资源。

### omnibox

```json
"omnibox": {
  "keyword": "tb"
}
```

在地址栏输入 `tb` + 空格，地址栏切换为扩展搜索模式。用户在地址栏输入的内容会触发 `chrome.omnibox.onInputChanged` 事件，扩展可以提供建议列表。

### storage.managed_schema

```json
"storage": {
  "managed_schema": "schemas/policy-schema.json"
}
```

定义企业策略（Group Policy）的 JSON Schema。只有被管理的 Chrome 实例（通过 GPO/MDM 配置）才会从这个 schema 读取配置。普通用户用不到。

### offline_enabled

```json
"offline_enabled": true
```

ChromeOS/Android 上使用。告知系统你的扩展可以在离线状态下工作。

### tts_engine

```json
"tts_engine": {
  "voices": [{"voice_name": "Alice", "lang": "zh-CN"}]
}
```

注册一个 TTS（文字转语音）引擎到 Chrome 的语音系统中。很少用到。

### file_browser_handlers / file_system_provider_capabilities

```json
"file_browser_handlers": [...]
```

ChromeOS 文件管理器的集成。ChromeOS 专属，桌面 Chrome 不支持。

### automation

```json
"automation": {
  "desktop": true
}
```

允许扩展通过 `chrome.automation` API 访问无障碍树。用于自动化测试或屏幕阅读器类扩展。

---

## 附录：字段快速索引

| 分类 | 字段 | 必填 | MV3 |
|---|---|---|---|
| 元数据 | `manifest_version` | ✅ | 3 |
| | `name` | ✅ | — |
| | `version` | ✅ | — |
| | `description` | — | — |
| | `icons` | — | — |
| | `default_locale` | — | — |
| | `minimum_chrome_version` | — | — |
| 权限 | `permissions` | — | 拆分出 host_permissions |
| | `host_permissions` | — | ✅ 新增 |
| | `optional_permissions` | — | — |
| | `optional_host_permissions` | — | ✅ 新增 |
| 后台 | `background` | — | service_worker 替代 scripts |
| 内容脚本 | `content_scripts` | — | 新增 world 字段 |
| UI | `action` | — | browser_action+page_action 合并 |
| | `side_panel` | — | ✅ 新增 |
| | `options_ui` | — | — |
| | `commands` | — | — |
| 安全 | `web_accessible_resources` | — | 新格式（数组） |
| | `content_security_policy` | — | 新格式（结构化） |
| | `sandbox` | — | — |
| 通信 | `externally_connectable` | — | — |
| | `key` | — | — |
| 高级 | `devtools_page` | — | — |
| | `omnibox` | — | — |
| | `storage` | — | — |
| | `update_url` | — | — |
| | `offline_enabled` | — | — |
| | `tts_engine` | — | — |
| | `automation` | — | — |

---

## 参考

- [Chrome Extensions Manifest File Format](https://developer.chrome.com/docs/extensions/mv3/manifest/)
- [Permissions documentation](https://developer.chrome.com/docs/extensions/mv3/declare_permissions/)
- Chrome Web Store [Manifest documentation](https://developer.chrome.com/docs/webstore/manifest/)
- [Manifest V2 → V3 migration guide](https://developer.chrome.com/docs/extensions/mv3/mv2-migration/)
