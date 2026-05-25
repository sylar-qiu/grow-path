# 浏览器 API 全景

所有 `chrome.*` API 按功能分类。这里只列常用和值得注意的，完整的去 Chrome Developers 参考文档。

## 标签页与窗口

```javascript
// Tabs
chrome.tabs.query({ active: true, currentWindow: true })
chrome.tabs.create({ url: 'https://example.com' })
chrome.tabs.update(tabId, { url: newUrl })
chrome.tabs.remove(tabId)
chrome.tabs.reload(tabId)
chrome.tabs.onUpdated.addListener((tabId, changeInfo, tab) => {})
chrome.tabs.onRemoved.addListener((tabId, removeInfo) => {})

// Tab Groups (MV3)
chrome.tabGroups.group({ tabIds: [1, 2, 3], groupId: groupId })
chrome.tabGroups.update(groupId, { color: 'blue', title: '工作' })

// Windows
chrome.windows.create({ url: 'popup.html', type: 'popup', width: 400, height: 300 })
```

Tab Groups 在 MV3 中正式可用。适合做标签分类管理类扩展。

## 下载

```javascript
chrome.downloads.download({
  url: 'https://example.com/file.pdf',
  filename: 'downloads/file.pdf',
  conflictAction: 'uniquify'
});

chrome.downloads.onCreated.addListener((downloadItem) => {});
chrome.downloads.onChanged.addListener((downloadDelta) => {});
```

可以监听所有下载，适合做下载管理类扩展。

## 剪贴板

需要在 manifest 中声明 `clipboardRead` / `clipboardWrite` 权限：

```javascript
// CS 中直接操作（通过 document.execCommand 或 Clipboard API）
// SW 中需要 Offscreen Document
await navigator.clipboard.writeText('text');
```

## 屏幕捕获

```javascript
// 捕获当前标签页
chrome.tabCapture.capture({
  audio: false,
  video: true,
  videoConstraints: { mandatory: { maxWidth: 1280, maxHeight: 720 } }
}, (stream) => {
  // stream 是 MediaStream 对象
});

// 捕获桌面（需要 desktopCapture 权限）
chrome.desktopCapture.chooseDesktopMedia(['screen', 'window'], (streamId) => {});
```

注意 `tabCapture` 只能捕获当前标签页，`desktopCapture` 可以让用户选择捕获目标。

## 空闲检测

```javascript
chrome.idle.onStateChanged.addListener((newState) => {
  // 'active' | 'idle' | 'locked'
  if (newState === 'idle') {
    // 用户不在电脑前
  }
});

chrome.idle.queryState(60, (state) => { /* 60秒无操作视为 idle */ });
```

适合做提醒类、计时类的扩展。

## 系统 API

```javascript
// 系统信息
chrome.system.cpu.getInfo((info) => {});
chrome.system.memory.getInfo((info) => {});
chrome.system.storage.getInfo((info) => {});
chrome.system.display.getInfo((info) => {});

// 语言
chrome.i18n.getMessage('messageName'); // 国际化
```

## 其他值得注意的 API

| API | 说明 | 权限 |
|---|---|---|
| `chrome.browsingData` | 清除浏览数据 | `browsingData` |
| `chrome.bookmarks` | 书签管理 | `bookmarks` |
| `chrome.history` | 历史记录 | `history` |
| `chrome.privacy` | 隐私设置 | `privacy` |
| `chrome.topsites` | 常用网站 | `topSites` |
| `chrome.fontSettings` | 字体设置 | `fontSettings` |
| `chrome.identity` | OAuth 认证 | `identity` |
| `chrome.certificateProvider` | 证书管理 | `certificateProvider` |

## 权限声明

```json
// 按需声明，不要申请不必要的权限
"permissions": ["storage", "tabs", "activeTab", "scripting", "downloads"],
"optional_permissions": ["clipboardRead", "bookmarks"]
```

用户审查权限时，少一个权限就多一分信任。能用 `activeTab` 就别用 `<all_urls>`。

---

## 参考

- [Chrome Extensions API Reference](https://developer.chrome.com/docs/extensions/reference/)
- [Permissions Guide](https://developer.chrome.com/docs/extensions/mv3/permission_warnings/)
