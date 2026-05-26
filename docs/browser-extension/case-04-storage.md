# 案例 04：存储层 — 淘宝品类数据模型

## 概述

`storage.js`（679 行）是淘宝插件的统一数据存储层。它是全插件最重要的代码——定义数据结构的文件。

## 存储选型

选型：**`chrome.storage.local` 做持久化，`chrome.storage.session` 做运行时缓存**

为什么不是 IndexedDB？

- IndexedDB 在 SW 中可用，但 API 更复杂
- 数据量预估：一个用户可能有 20-50 个品类，每个品类 500 个 SPU，总数据量 1-5MB——`chrome.storage.local` 完全够用
- `chrome.storage.local` 是键值对，序列化/反序列化简单

为什么不是 `chrome.storage.sync`？

- 品类/SPU 数据不需要跨设备同步
- sync 有 102KB/项的配额限制——一个品类数据就可能超限

## 数据结构

```javascript
{
  "storage_version": 1,
  "keywords": [
    { id: 1, name: "蓝牙耳机", created_at: "...", updated_at: "..." }
  ],
  "spu_1": [  // spu_{keywordId}
    { num_iid: "12345", title: "...", shop: "...",
      price: 99, sales: 5000, sales_rank: 3,
      detail_collected_at: "...", comments_collected_at: "..." }
  ],
  "_kw_counter": 2,
  "detail_body_12345": { /* 完整的详情快照 JSON */ },
  "item_detail_archive": { /* SPU 删除时保留的元数据归档 */ }
}
```

### 设计亮点

1. **SPU 列表与详情体解耦**

   `spu_1` 存的是 SPU 的**元数据**（标题、价格、销量），`detail_body_12345` 存的是**完整详情**（SKU 列表、参数、评论概要）。这样在品类列表中切换时只用加载轻量的元数据列表，详情是按需加载。

2. **前缀 + ID 的 key 设计**

   `spu_`、`detail_body_`、`_kw_` 前缀避免 key 冲突，同时可以用 `chrome.storage.local.get(null)` 一次读出所有数据再筛选。

3. **归档机制**

   ```javascript
   // 删除品类时，不直接删 detail 数据
   // SPU 元数据从 spu_* 移除，但详情体保留
   // 以后相同商品再次加入时，从 archive 恢复时间戳
   function removePreservingDetailArchive(name) { ... }
   ```

   这解决了实际使用场景中的问题：用户在清洗数据时会删掉旧品类，但后续可能又把同一商品加回来。归档让"被删 SPU 的详情时间戳"能复用。

## 数据写入策略：即时入库

项目在 `AGENTS.md` 中定义了一条核心规则——**即时入库**：

```
凡改变业务数据的操作（品类、SPU 列表与字段、详情/评论元数据等），
须在当次操作流程内写入 chrome.storage.local。
不得以仅靠内存或延迟同步作为常态。
```

对应代码：

```javascript
// storage.js 的 saveBatch 方法
function saveBatch(keywordName, rows) {
  // 1. 确保品类存在
  // 2. 写入 spu_{id}
  // 3. 更新品类更新时间和统计数据
}
```

"不延迟同步"意味着不依赖内存缓存等显式 flush——这是扩展存储和普通数据库使用习惯的一个重要差异。

## 存储 API 设计

```javascript
// 对外暴露的 API
storage.list()                         // 所有品类
storage.getByName(name)               // 按名称查品类
storage.getById(id)                    // 按 ID 查品类
storage.create(name)                   // 创建品类
storage.remove(name)                   // 删除品类
storage.removePreservingDetailArchive  // 删除但保留详情归档
storage.saveBatch(name, rows)          // 批量写入 SPU
storage.mergeDetailMeta(numIid, meta)  // 合并详情元数据
storage.get(numIid)                    // 按 SPU ID 读取
storage.patchSpuSales(numIid, sales)   // 更新销量
```

## 运行时缓存

Side Panel 在 JS 内存中维护了一套热数据缓存：

```javascript
var cachedCategories = [];    // 品类列表
var cachedSpuItems = [];      // 当前品类的 SPU
var detailCache = {};         // 详情快照内存缓存
```

但注意——**这些缓存不是数据源**。每次用户切换品类时，Side Panel 会重新从 storage 加载。缓存的目的是在同一个品类内快速切换视图（比如本地速览 → AI 分析），避免反复读盘。

## 与搜索页 CS 的协同

搜索页的 Content Script 有自己的内存状态：

```javascript
var storageKeywordMap = {};     // name → keyword row
var storageCategoryItems = {};  // category → [num_iid]
var storageItemDataMap = {};    // num_iid → item data
```

搜索页 CS 启动时通过 `hydrateStorageCache` 从 `chrome.storage` 加载到内存。**这里有一个典型的问题**：Side Panel 修改了数据后，搜索页 CS 不会自动知道。解决方案是通过消息同步：

```
Side Panel 添加 SPU → 发给 SW → forwardToTab → CS 收到 switchCategory
→ CS 从 storage 重新加载数据 → 重新显示浮层状态
```

---

## 参考

- 源码：`Projects/taobao-plugin/extension/src/lib/storage.js`
- 知识体系：[存储层](05-storage.md)
