# 存储层

## chrome.storage 四种方案

### local

```javascript
await chrome.storage.local.set({ key: value })
await chrome.storage.local.get(['key'])
await chrome.storage.local.get() // 全部
await chrome.storage.local.getBytesInUse(null) // 用量
```

- **作用域**：当前设备
- **容量**：无硬性上限（建议不超 100MB，超出会弹警告）
- **性能**：异步 + 序列化，大量写入有延迟
- **适合**：缓存、用户数据

### sync

```javascript
await chrome.storage.sync.set({ key: value })
```

- **作用域**：同一 Google 账号跨设备同步
- **容量**：最大 102KB/项，总容量 512KB（各浏览器不同）
- **频率**：每分钟最多 120 次写入
- **适合**：配置项、偏好设置

### session

```javascript
await chrome.storage.session.set({ temp: value })
```

- **作用域**：当前浏览器会话
- **容量**：默认 10MB（可申请到 1GB）
- **性能**：内存级——最快，无 I/O
- **适合**：运行时状态、临时缓存、SW 状态恢复
- **注意**：关闭浏览器后清除

### managed

```json
// manifest.json
"storage": {
  "managed_schema": "schema.json"
}
```

- **作用域**：企业管理员策略控制
- **只读**：扩展只能读不能写
- **适合**：企业部署配置

---

## 选型决策

```
你要存什么？
├── 用户配置（需要跨设备？）
│   ├── 是 → chrome.storage.sync
│   └── 否 → chrome.storage.local
├── SW 运行时状态（会话内持久？）
│   ├── 是 → chrome.storage.session
│   └── 否 → chrome.storage.local
├── 大量结构化数据（>MB 级）
│   ├── 常用查询 → IndexedDB
│   └── 纯缓存 → Cache API
└── 企业策略配置 → chrome.storage.managed（只读）
```

---

## IndexedDB

SW 中可以使用 IndexedDB，适合大量结构化数据：

```javascript
// 在 SW 或 CS 中
const db = await openDB('my-db', 1, {
  upgrade(db) {
    const store = db.createObjectStore('records', { keyPath: 'id' });
  }
});

await db.add('records', { id: 1, data: '...' });
const record = await db.get('records', 1);
```

注意：IndexedDB 操作不会唤醒休眠的 SW。如果你依赖 IndexedDB 存取数据，确保在事件响应中提前打开数据库。

---

## Cache API

主要用于缓存网络请求响应，MV3 中 SW 可以直接操作：

```javascript
const cache = await caches.open('api-cache-v1');
await cache.put(url, response);
const cached = await cache.match(url);
```

适合缓存 API 响应等场景。

---

## 注意事项

### 存储变更监听

```javascript
chrome.storage.onChanged.addListener((changes, areaName) => {
  for (const [key, { oldValue, newValue }] of Object.entries(changes)) {
    console.log(`Key ${key} changed from ${oldValue} to ${newValue}`);
  }
});
```

### 存储配额

```javascript
// 检查剩余空间
const usage = await chrome.storage.local.getBytesInUse(null);
const quota = 5242880; // 5MB 默认
console.log(`使用: ${usage}, 剩余: ${quota - usage}`);
```

### 数据安全

- `chrome.storage` 的数据存储在本机加密沙箱中，不可被其他扩展访问
- 但**不**能用来存密码/Token——这是扩展存储，不是安全凭证存储
- 敏感凭证建议用 `chrome.identity.getAuthToken` 或 Chrome 内置密码管理器

---

## 参考

- [chrome.storage API](https://developer.chrome.com/docs/extensions/reference/storage/)
- [IndexedDB in extensions](https://developer.chrome.com/docs/extensions/mv3/service_workers/#indexeddb)
