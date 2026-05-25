# 内容脚本层

## 注入机制

三种注入方式，各自适用的场景不同：

### 声明式注入（manifest）

```json
{
  "content_scripts": [{
    "matches": ["https://*.taobao.com/*"],
    "js": ["content.bundle.js"],
    "css": ["content.css"],
    "run_at": "document_idle"
  }]
}
```

- 浏览器自动管理注入时机
- 不能动态修改匹配规则
- 适合功能固定的场景

### 程序化注入（dynamic）

```javascript
// MV3
chrome.scripting.executeScript({
  target: { tabId: tab.id },
  files: ['content.bundle.js']
})

// 或注入代码（注意：不能是远程代码）
chrome.scripting.executeScript({
  target: { tabId: tab.id },
  func: () => { /* 你的行内代码 */ },
  args: ['arg1', 'arg2']
})
```

- 灵活，可按需注入
- 需要 `scripting` 权限
- func 方式在 MV3 中可行——函数被序列化后执行

### 动态注册（registerContentScripts）

```javascript
chrome.scripting.registerContentScripts([{
  id: 'dynamic-cs',
  matches: ['https://*.example.com/*'],
  js: ['content.bundle.js'],
  persistAcrossSessions: false
}])
```

- 运行时注册，可持久化
- 适合用户自定义注入规则的场景

---

## 三种 run_at 时机

| 值 | 触发时间 | 适合 |
|---|---|---|
| `document_start` | DOM 刚构建，外部资源未加载 | 需要最早拦截页面行为 |
| `document_end` | DOM 加载完成，图片等资源未加载 | 常见选择 |
| `document_idle` | 页面完成渲染（默认） | 不依赖页面加载的内容 |

注意：如果你的扩展需要在页面完全渲染前修改 DOM 结构，用 `document_start`。但此时 DOM 可能还不完整。

---

## 与页面交互

### Shadow DOM 隔离

一种有效但不完美的隔离方案，通常不需要。

### 注入自定义 DOM

直接在页面中插入元素，缺点是样式可能被页面 CSS 覆盖。如果你需要可靠样式隔离：

```javascript
const host = document.createElement('div');
document.body.appendChild(host);
const shadow = host.attachShadow({ mode: 'closed' });
shadow.innerHTML = `<style>/* 你的样式 */</style><div>你的 UI</div>`;
```

`mode: 'closed'` 防止页面通过 `host.shadowRoot` 访问你的内容。

---

## Main World 通信

当你的 inject script 需要和 Content Script 通信时：

```
Content Script              Main World (injected script)
     │                              │
     ├── window.postMessage ────────►
     │                              │
     ◄──── window 'message' event ──┤
```

Content Script 监听：

```javascript
window.addEventListener('message', (event) => {
  if (event.source !== window) return; // 只接受同源消息
  if (event.data?.source !== 'my-inject') return; // 你的标识
  // 处理页面数据
});
```

Main World 脚本发送：

```javascript
window.postMessage({ source: 'my-inject', type: 'DATA', payload: { ... } }, '*');
```

---

## 权限与限制

- Content Script 默认无权访问大部分 `chrome.*` API
- 能直接用的：`runtime`、`storage`、`i18n`
- 其他需要 API 需要发消息给 SW 代执行
- 跨域请求受目标页面 CSP 限制，但扩展自身发 fetch 不受限

---

## 参考

- [Content Scripts | Chrome Developers](https://developer.chrome.com/docs/extensions/mv3/content_scripts/)
- [Chrome scripting API](https://developer.chrome.com/docs/extensions/reference/scripting/)
