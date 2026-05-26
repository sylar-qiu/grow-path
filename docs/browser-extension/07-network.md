# 网络层

MV3 最大的网络变化：webRequest 的 blocking 模式被 Declarative Net Request (DNR) 替代。

## Declarative Net Request

核心思想：不写代码拦截请求，声明规则让浏览器替你执行。

### 规则结构

一条规则由三部分组成：

```json
{
  "id": 1,
  "priority": 1,
  "condition": {
    "urlFilter": "||example.com/ads/*",
    "resourceTypes": ["script", "image"]
  },
  "action": {
    "type": "block"
  }
}
```

### 规则存放方式

**静态规则**：manifest.json 中声明规则文件

```json
{
  "declarative_net_request": {
    "rule_resources": [{
      "id": "ruleset_1",
      "enabled": true,
      "path": "rules.json"
    }]
  }
}
```

**动态规则**：运行时通过 API 添加/删除

```javascript
await chrome.declarativeNetRequest.updateDynamicRules({
  addRules: [newRule],
  removeRuleIds: [oldRuleId]
});
```

### 规则类型

| type | 功能 |
|---|---|
| `block` | 拦截请求 |
| `redirect` | 重定向（去除参数、替换 URL） |
| `modifyHeaders` | 修改请求/响应头 |
| `allow` | 允许（优先级高于 block） |
| `allowAllRequests` | 允许该域名下所有请求 |

### 配额限制

| 项目 | 限额 |
|---|---|
| 静态规则集 | 最多 50 个 |
| 每个规则集规则数 | 最多 5000 条 |
| 动态规则总数 | 最多 5000 条 |
| 会话规则总数 | 最多 5000 条 |
| 正则表达式规则 | 最多 1000 条 |
| 修改请求头的规则 | 最多 50 条 |

### 实际案例：去掉 URL 中的追踪参数

```json
{
  "id": 1,
  "priority": 1,
  "condition": {
    "regexFilter": "^https?://[^/]+/[^?]*\\?(.*&)?(utm_source|utm_medium|utm_campaign|fbclid)=[^&]+"
  },
  "action": {
    "type": "redirect",
    "redirect": {
      "regexSubstitution": "\\0"
    }
  }
}
```

这个场景注意：正则比 urlFilter 慢，尽量先用 urlFilter 缩小范围。

---

## webRequest（观察模式）

MV3 中 webRequest 保留但只读：

```javascript
chrome.webRequest.onBeforeRequest.addListener(
  (details) => {
    console.log('请求:', details.url);
    // 可以读但不能修改
    return {}; // 不能返回 redirectUrl 或 cancel
  },
  { urls: ['<all_urls>'] },
  ['requestBody'] // 可以读 requestBody
);
```

### DNR 覆盖不了的场景

- 需要根据请求**内容**动态判断（读取 requestBody 的值决定是否拦截）
- 需要非常复杂的重定向逻辑（非正则可表达的）
- 需要与其他扩展协作

这些场景只能保留 webRequest 的观察模式，无法完全替代原来的 blocking 行为。

---

## 跨域与安全

### 扩展跨域

Content Script 受目标页面的 CSP 限制。但扩展自身（SW 或 Popup）发请求不受页面 CSP 限制：

```javascript
// SW 中，可以请求任何 URL（需 host_permissions）
const resp = await fetch('https://api.example.com/data');
```

manifest.json 中声明：

```json
"host_permissions": [
  "https://api.example.com/*"
]
```

### CSP 配置

```json
// manifest.json
"content_security_policy": {
  "extension_pages": "script-src 'self'; object-src 'self'",
  "sandbox": "sandbox allow-scripts allow-forms"
}
```

在 MV3 中，`extension_pages` 的 CSP 更严格——不能加 `'unsafe-eval'`，意味着你不能用 eval、new Function 等。

### Sandbox 页面

如果你需要执行用户提供的动态代码（比如用户脚本编辑器），必须使用 Sandbox：

```json
"sandbox": {
  "pages": ["sandbox/sandbox.html"]
}
```

Sandbox 页面不能访问任何 extension API。

---

## 参考

- [Declarative Net Request API](https://developer.chrome.com/docs/extensions/reference/declarativeNetRequest/)
- [MV3 Network Migration](https://developer.chrome.com/docs/extensions/mv3/mv3-migration/#network-request-modification)
- [Content Security Policy](https://developer.chrome.com/docs/extensions/mv3/manifest/content_security_policy/)
