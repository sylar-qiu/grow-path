# DevTools 扩展

让你自己的面板出现在浏览器的 F12 中。

## 基本结构

```json
// manifest.json
"devtools_page": "devtools/devtools.html"
```

`devtools.html` 本身是个空页面（实际不显示），它的作用是初始化各类面板：

```html
<!-- devtools/devtools.html -->
<script src="devtools.js"></script>
```

```javascript
// devtools.js
chrome.devtools.panels.create(
  '我的扩展',        // 面板名称
  'icon.png',       // 图标
  'panel.html',     // 面板内容
  (panel) => {
    console.log('面板已创建');
  }
);
```

## DevTools 页面的特殊性

- **独立作用域**：DevTools 页面有自己独立的 window，与主扩展的 SW 不共享
- **不能直接调用**大部分 extension API——只能通过消息与 SW 通信
- **只能调用的 API**：
  - `chrome.devtools.*`
  - `chrome.runtime.sendMessage`
  - `chrome.runtime.connect`

## 可用 API

```javascript
// 检查当前选中的元素
chrome.devtools.inspectedWindow.eval(
  'document.querySelector("div").innerHTML',
  (result, exceptionInfo) => {}
);

// 获取选中元素的 DOM 信息
chrome.devtools.panels.elements.onSelectionChanged.addListener(() => {
  chrome.devtools.inspectedWindow.eval(
    '[$0.tagName, $0.id, $0.className]',
    (result) => console.log(result)
  );
});

// 监听网络请求
chrome.devtools.network.onRequestFinished.addListener((request) => {
  console.log(request.request.url, request.response.status);
});

// 监听控制台日志
chrome.devtools.panels.sources.onSelectionChanged.addListener(...);
```

## 与主扩展通信

```javascript
// DevTools 页面
const port = chrome.runtime.connect({ name: 'devtools' });
port.postMessage({ type: 'FROM_DEVTOOLS', data: { ... } });
port.onMessage.addListener((msg) => {
  // 收到 SW 的回复
});

// SW 中
chrome.runtime.onConnect.addListener((port) => {
  if (port.name === 'devtools') {
    port.onMessage.addListener((msg) => {
      // 处理 DevTools 发出的消息
    });
  }
});
```

DevTools 页面**也有** content script 一样的跨域限制，不能直接发跨域请求。

## 应用场景

| 场景 | 说明 |
|---|---|
| 页面分析 | 读取页面 DOM、JS 变量、网络请求 |
| 抓包工具 | 监听并展示所有请求 |
| CSS 调试辅助 | 批量修改样式，导出修改记录 |
| 性能监控 | 展示页面性能指标 |
| 爬虫辅助 | 分析页面结构、数据结构 |

---

## 参考

- [DevTools Extension API](https://developer.chrome.com/docs/extensions/mv3/devtools/)
