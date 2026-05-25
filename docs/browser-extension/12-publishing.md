# 发布与运维

## Chrome Web Store 上架

### 准备材料

| 项目 | 说明 |
|---|---|
| 开发者账户 | 一次性注册费 $5，需 Google 账号 |
| 扩展包 | 压缩的 `.zip` 文件 |
| 图标 | 至少 128x128 PNG |
| 截图 | 至少 1 张 1280x800 或 640x400 |
| 宣传图（可选） | 440x280 小宣传图、920x680 大幅宣传图 |
| 隐私政策 | 收集任何数据就需要 |
| 说明文字 | 名称、简短描述、详细描述、类别 |

### 隐私政策

MV3 要求所有收集用户数据的扩展必须有隐私政策。

数据收集的定义比你想的宽：
- 使用 `chrome.identity`
- 发送 `GA / 统计代码`
- 存储用户行为记录
- 甚至 `chrome.storage.sync`（数据存到 Google 服务器）

没有隐私政策 → **不过审**。

### 发布流程

1. 登录 [Chrome Web Store Developer Dashboard](https://chrome.google.com/webstore/devconsole)
2. 上传 zip
3. 填写详情页
4. 提交审核

**审核时间**：首次提交通常 1-3 个工作日。更新通常更快（几小时到 1 天）。

### 常见拒审原因

- 权限过大（请求了不必要的权限）
- 没有隐私政策
- 功能过于简单（被判定为低质量）
- 诱导点击 / 欺骗性 UI
- 最小权限原则违反（能用 `activeTab` 就别要 `<all_urls>`）

---

## 版本管理

### manifest.json 版本号

```json
{
  "version": "1.2.3"
}
```

遵循 SemVer，但只有 4 位数字格式：`1.0.0.0` 到 `4.0.0.0`。

### 版本更新处理

```javascript
chrome.runtime.onInstalled.addListener((details) => {
  if (details.reason === 'install') {
    // 首次安装
    initializeExtension();
  } else if (details.reason === 'update') {
    // 版本更新
    const prevVersion = details.previousVersion;
    handleMigration(prevVersion);
  } else if (details.reason === 'chrome_update') {
    // Chrome 浏览器更新后
  } else if (details.reason === 'update_permissions') {
    // 新增了权限
    showNewPermissionsNotice();
  }
});
```

### 不兼容变更迁移

```javascript
async function handleMigration(prevVersion) {
  const semver = prevVersion.split('.').map(Number);

  if (semver[0] < 2) {
    // 从 v1.x 迁移到 v2.x——数据结构变了
    await migrateFromV1();
  }

  if (semver[1] < 3 && semver[0] === 2) {
    // v2.x → v2.3+，部分 key 重命名
    await renameKeys();
  }
}
```

---

## 强制更新

Chrome 会自动检查扩展更新（通常 5 小时内）。等不急可以手动触发：

`chrome://extensions` → 开发者模式 → **更新**

也可以提示用户去更新：

```javascript
// 你的服务端返回最新版本号
fetch('https://api.myext.com/version').then(res => res.json()).then(({latest}) => {
  if (latest > chrome.runtime.getManifest().version) {
    chrome.notifications.create({
      type: 'basic',
      iconUrl: 'icon128.png',
      title: '有新版本可用',
      message: `请到 chrome://extensions 检查更新`,
    });
  }
});
```

---

## 国际化

### 支持多语言

```json
// manifest.json
"default_locale": "zh_CN"
```

### 语言文件

`_locales/en/messages.json`：

```json
{
  "appName": { "message": "My Extension" },
  "appDesc": { "message": "Description" },
  "hello": { "message": "Hello, $USER$!", "placeholders": {
    "user": { "content": "$1" }
  }}
}
```

### 使用

```javascript
chrome.i18n.getMessage('hello', ['Sylar']);
```

---

## 数据统计

### 内置统计

Chrome Web Store Dashboard 提供基础的安装量、卸载量统计。

### 自行统计

如果你要自己统计，注意两点：
1. 不能使用 GA 等第三方脚本（被 CSP 限制）
2. 可以从 SW 发 fetch 到自己的统计接口

```javascript
// SW 中埋点
chrome.runtime.onInstalled.addListener(() => {
  fetch('https://api.myext.com/analytics/install', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      extId: chrome.runtime.id,
      version: chrome.runtime.getManifest().version,
      platform: navigator.platform,
    })
  });
});
```

---

## 参考

- [Chrome Web Store 发布指南](https://developer.chrome.com/docs/webstore/publish/)
- [隐私政策要求](https://developer.chrome.com/docs/webstore/program-policies/)
