# UI 层

## Action

浏览器工具栏图标的管理入口。通过 `chrome.action` API 控制：

```javascript
// 设置图标（16/32/48/128）
chrome.action.setIcon({ path: { 16: 'icon16.png', 32: 'icon32.png' } });

// Badge——图标右下角的小徽章
chrome.action.setBadgeText({ text: '3' });
chrome.action.setBadgeBackgroundColor({ color: '#FF0000' });

// Tooltip——鼠标悬停提示
chrome.action.setTitle({ title: '点击查看详情' });

// 启用/禁用（灰色）
chrome.action.enable(tabId);
chrome.action.disable(tabId);
```

### 点击行为

点击图标默认打开 Popup。你也可以覆写点击行为：

```javascript
// manifest.json 中去掉 default_popup
chrome.action.onClicked.addListener((tab) => {
  // 此时不会弹 popup，而是执行此回调
  chrome.sidePanel.open({ tabId: tab.id });
});
```

---

## Popup

标准的 HTML 页面，但有几个特殊性：

- **瞬态**：用户点击空白区域就关闭，状态丢失
- **Debug**：需要固定（Pin）后按 F12 才能保持
- **大小**：不建议太大，推荐 ~600x400

Popup 无法直接访问 DOM，通过消息与 SW 通信：

```javascript
// Popup 中
chrome.runtime.sendMessage({ type: 'GET_DATA' }, (response) => {
  document.getElementById('result').textContent = response;
});
```

---

## Side Panel

MV3 的重大新增。关键差异：

| | Popup | Side Panel |
|---|---|---|
| 展示方式 | 浮动小窗口 | 侧边固定面板 |
| 生命周期 | 点击即关，瞬态 | 可常驻 |
| 多标签 | 每个标签独立 | 可配置共享/隔离 |
| 空间 | 小 | 大 |
| 适合 | 快捷操作 | 长任务、对话、监控 |

### 基本使用

```json
// manifest.json
"side_panel": {
  "default_path": "sidepanel.html"
}
```

```javascript
// SW 中打开
chrome.sidePanel.open({ tabId: currentTab.id });

// 配置每个标签使用独立面板
chrome.sidePanel.setOptions({
  tabId: currentTab.id,
  path: 'sidepanel.html',
  enabled: true
});

// 面板状态——每个标签独立
chrome.sidePanel.setPanelBehavior({ openPanelOnActionClick: true });
```

### 与 Popup 共存

一个常见模式：点击图标——第一次打开 Popup，Popup 里点"展开"就切换为 Side Panel：

```javascript
// Popup 中
document.getElementById('expand-btn').addEventListener('click', () => {
  chrome.runtime.sendMessage({ type: 'OPEN_SIDE_PANEL' });
  window.close(); // 关掉 popup
});

// SW 中
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  if (msg.type === 'OPEN_SIDE_PANEL') {
    chrome.sidePanel.open({ tabId: sender.tab.id });
  }
});
```

---

## Options Page

```json
// manifest.json
"options_ui": {
  "page": "options.html",
  "open_in_tab": true
}
```

两种打开方式：

```javascript
// 扩展内部
chrome.runtime.openOptionsPage();

// 浏览器右键菜单 → 扩展管理 → 详情 → 扩展选项
```

---

## 其他 UI 入口

### Context Menus（右键菜单）

```javascript
chrome.contextMenus.create({
  id: 'my-action',
  title: '我的动作',
  contexts: ['selection', 'link', 'page']
});

chrome.contextMenus.onClicked.addListener((info, tab) => {
  console.log(info.selectionText); // 选中的文本
});
```

### Omnibox（地址栏）

```json
// manifest.json
"omnibox": {
  "keyword": "myext"
}
```

用户在地址栏输入 `myext + Tab` 后触发：

```javascript
chrome.omnibox.onInputChanged.addListener((text, suggest) => {
  suggest([{ content: 'cmd', description: '执行命令' }]);
});

chrome.omnibox.onInputEntered.addListener((text) => {
  // 处理输入
});
```

### Commands（快捷键）

```json
// manifest.json
"commands": {
  "_execute_action": {
    "suggested_key": { "default": "Ctrl+Shift+Y" }
  },
  "my-command": {
    "suggested_key": { "default": "Ctrl+Shift+U" },
    "description": "我的自定义命令"
  }
}
```

```javascript
chrome.commands.onCommand.addListener((command) => {
  if (command === 'my-command') { /* ... */ }
});
```

### Notifications

```javascript
chrome.notifications.create({
  type: 'basic',
  iconUrl: 'icon128.png',
  title: '提示',
  message: '操作完成'
});
```

---

## 参考

- [chrome.action API](https://developer.chrome.com/docs/extensions/reference/action/)
- [Side Panel API](https://developer.chrome.com/docs/extensions/reference/sidePanel/)
- [Context Menus API](https://developer.chrome.com/docs/extensions/reference/contextMenus/)
