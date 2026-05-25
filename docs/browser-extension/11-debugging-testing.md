# 调试与测试

## Service Worker 调试

### 方法一：DevTools

1. 扩展管理页（`chrome://extensions`）
2. 找到你的扩展，点击 **Service Worker** 链接
3. 弹出独立的 DevTools 窗口

在这个 DevTools 中：
- Console 可以看到 SW 所有日志
- Sources 可以打断点
- Application → Storage 可以查看 `chrome.storage` 内容

### 方法二：chrome://serviceworker-internals

列出浏览器所有 SW，可以强制 push、停止、查看状态。

### 常见调试痛点

**SW 已经休眠了怎么 Debug？**
在 DevTools 打开状态下保持日志面板，直到 SW 启动时的事件被捕捉。

**Popup 一点就消失怎么 Debug？**
保持 DevTools 在 Popup 上打开：

1. 关闭 Popup
2. 右键扩展图标 → 审查弹出内容
3. DevTools 打开后再操作 Popup

或者更好：在 Popup 中加 `window.onbeforeunload` 日志。

**Content Script 怎么 Debug？**
直接在目标页面的 DevTools Console 里找。
默认有多个 Content Script 时，需要从 Console 面板的下拉菜单选择对应的执行上下文。

---

## 日志策略

MV3 中 SW 没有持久控制台，建议建立日志系统：

```javascript
// shared/logger.ts
const LOG_PREFIX = '[MyExt]';

export function log(level: 'debug' | 'info' | 'warn' | 'error', ...args: any[]) {
  // 1. 始终输出到控制台
  console[level](LOG_PREFIX, ...args);

  // 2. 开发模式写 storage，生产环境可关掉
  if (__DEV__) {
    chrome.storage.session.get('logs').then(({ logs = [] }) => {
      logs.push({ level, time: Date.now(), msg: args.join(' ') });
      // 只保留最近的 200 条
      if (logs.length > 200) logs = logs.slice(-200);
      chrome.storage.session.set({ logs });
    });
  }
}
```

然后在 Popup 或 Side Panel 中展示历史日志。

---

## 错误追踪

### chrome.runtime.lastError

所有回调风格 API 都需要检查：

```javascript
chrome.storage.local.set({ key: value }, () => {
  if (chrome.runtime.lastError) {
    console.error('存储失败:', chrome.runtime.lastError.message);
  }
});
```

Promise 风格 API 会自动 throw，但也要 catch：

```javascript
try {
  await chrome.storage.local.set({ key: value });
} catch (e) {
  console.error('存储失败:', e);
}
```

### Chrome 扩展错误页面

`chrome://extensions` → 点击你的扩展 → **Errors** 按钮。
这里会列出扩展运行时所有未捕获的异常。

---

## 自动化测试

### Puppeteer / Playwright

两者都支持加载解包扩展：

```javascript
// Puppeteer
const browser = await puppeteer.launch({
  headless: false,
  args: [
    `--disable-extensions-except=${pathToExt}`,
    `--load-extension=${pathToExt}`,
  ]
});

// Playwright
const browser = await chromium.launch({
  headless: false,
  args: [
    `--disable-extensions-except=${pathToExt}`,
    `--load-extension=${pathToExt}`,
  ]
});
const context = await browser.newContext();
const page = await context.newPage();
```

### 测试重点

扩展测试比普通 Web 测试多了几层：

1. **SW 逻辑**：消息处理、状态管理、Alarms
2. **CS 逻辑**：页面注入、DOM 操作
3. **跨组件通信**：SW ↔ CS ↔ Popup
4. **UI**：Popup / Side Panel 渲染

建议分离逻辑测试和集成测试：

```javascript
// 单元测试（vitest / jest）——测试 SW 状态管理
import { processData } from '../background/processor';
test('processData handles empty input', () => {
  expect(processData([])).toEqual({ count: 0 });
});

// 集成测试（Puppeteer）——测试 CS 注入
test('content script adds button to page', async () => {
  await page.goto('https://example.com');
  await page.waitForSelector('my-ext-button');
  expect(await page.textContent('my-ext-button')).toBe('Click me');
});
```

---

## 参考

- [Debugging Extensions](https://developer.chrome.com/docs/extensions/mv3/debugging/)
- [Testing Chrome Extensions with Puppeteer](https://developer.chrome.com/docs/extensions/mv3/puppeteer/)
