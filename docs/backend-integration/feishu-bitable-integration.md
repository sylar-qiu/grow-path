# 飞书多维表 × qn-server 连调配置手册

> **适用场景**：高转化关键词监控（`qn_metric_sycm_high_conv`）
> **数据流**：PG → 飞书多维表（指标同步）+ 飞书 → PG（处理状态回写）
> **ECS 示例**：`http://8.163.126.172:8080`
> **代码版本**：需 `7ca9ec1` 及以上（支持无 `record_id` 的自然键回写）

---

## 一、整体架构

```
插件 ingest 排行明细
        ↓
aggregate.sycm.high_conv  →  qn_metric_sycm_high_conv (PostgreSQL)
        ↓
sync.bitable (Worker)     →  飞书多维表 (Upsert)
        ↓
运营在飞书改「处理状态」
        ↓
多维表自动化 · HTTP POST  →  /webhook/feishu/bitable
        ↓
更新 PG.status            →  Refine 后台刷新可见
```

| 方向 | 触发方式 | 依赖 |
|------|----------|------|
| PG → 飞书 | Worker `sync.bitable` | `.env` 多维表配置 + Base 授权 |
| 飞书 → PG | 多维表自动化 HTTP | `FEISHU_WEBHOOK_TOKEN` + 自动化 Body |

---

## 二、飞书侧配置

### 2.1 创建独立 Base 多维表

1. 飞书 → **多维表格** → **新建多维表格**
2. 打开后 URL 形如：

```
https://u1gkjqbgljc.feishu.cn/base/ADWJbOBPjaVPpEsB2ILcTNpvnbe?table=tblCqtaWNsvgeEs2&view=vewBtYe1pP
```

从 URL 解析：

| 配置项 | 示例值 | 说明 |
|--------|--------|------|
| **app_token** | `ADWJbOBPjaVPpEsB2ILcTNpvnbe` | `base/` 后面那段 |
| **table_id** | `tblCqtaWNsvgeEs2` | `table=` 后面那段 |

> 知识库 Wiki 嵌入的表 URL 是 `wiki/...`，API 需额外换 token；**推荐用独立 Base**。

### 2.2 建列（字段名必须与映射一致）

在数据表 `tblCqtaWNsvgeEs2` 中创建以下列：

| 列名 | 推荐类型 | PG 列 / 用途 |
|------|----------|--------------|
| 搜索词 | 文本 | `keyword` |
| 类目名称 | 文本 | `cate_label` |
| 统计时间段 | 文本 | `stat_period_text` |
| 转化指数 | 文本 | `conv_index` |
| 增速 | 文本 | `search_growth` |
| 人气中值 | 文本 | `pop_median` |
| 组内排名 | 文本 | `conv_rank` |
| **处理状态** | **单选** 或 **文本** | `status`（回写 PG） |

**处理状态** 单选建议选项：

- 待处理
- 跟进中
- 已处理
- 忽略

> 列名必须与 `field_map` 一致（默认中文见上表）。类型推荐全部 **文本**，避免数字/单选格式转换问题。

### 2.3 飞书开放平台 · 同一自建应用

**与 Refine 飞书登录共用同一个应用**，不要新建第二个。

#### ① 打开开放平台

https://open.feishu.cn/app → 选择 `FEISHU_APP_ID` 对应的应用

#### ② 权限管理

搜索并开通：

- **查看、编辑和管理多维表格**（权限码 `bitable:app`）

保存后，若企业要求审批，等管理员通过。

#### ③ 版本发布

**版本管理与发布** → **创建版本** → **申请发布**（或企业内可用）

未发布 / 权限未生效时，API 会 403。

#### ④ 网页应用（OAuth，若已配可跳过）

| 配置项 | 值 |
|--------|-----|
| 主页 | `http://8.163.126.172:8080/` |
| 重定向 URL | `http://8.163.126.172:8080/auth/feishu/callback` |
| H5 可信域名 | `8.163.126.172:8080` |
| IP 白名单 | `8.163.126.172` |

#### ⑤ 机器人能力（建议开启）

应用详情 → **机器人** → 启用（便于部分场景识别应用）

### 2.4 Base 授权给应用（关键，易踩坑）

**不是** 普通「分享 → 添加协作者」，而是：

1. 打开 Base 页面
2. 右上角 **`···`（三个点）** → **更多**
3. **添加文档应用**
4. 搜索 **开放平台应用名称**
5. 权限选 **可编辑**（高级权限场景选 **可管理**）

搜不到应用时检查：

- [ ] 已开 `bitable:app` 权限
- [ ] 已发布新版本
- [ ] 应用可用范围包含当前用户

### 2.5 多维表自动化 · 飞书 → PG 回写（方案 B）

当前 ECS 为 **HTTP + IP**，无法用事件订阅（需 HTTPS 域名），用 **多维表自动化**。

#### 触发器

| 项 | 值 |
|----|-----|
| 触发 | **修改记录时** |
| 数据表 | 高转化监控表 |
| 监听字段 | **仅勾选「处理状态」** |

#### 动作：发送 HTTP 请求

| 项 | 值 |
|----|-----|
| 方法 | `POST` |
| URL | `http://8.163.126.172:8080/webhook/feishu/bitable` |

**请求头（Header）**

| Key | Value |
|-----|--------|
| `Content-Type` | `application/json` |
| `X-Feishu-Webhook-Token` | 见 ECS `.env` 中 `FEISHU_WEBHOOK_TOKEN`（自行生成，非飞书 Dev 后台） |

**请求体（Body）· Raw 格式 JSON**

先写固定键名，冒号后面用 **「+」插入变量**（来自「第 1 步 · 修改的记录」）：

```json
{
  "pg_table": "qn_metric_sycm_high_conv",
  "fields": {
    "keyword": "← 插入变量：搜索词",
    "stat_period_text": "← 插入变量：统计时间段",
    "cate_label": "← 插入变量：类目名称",
    "status": "← 插入变量：处理状态"
  }
}
```

**变量对照**

| JSON 键 | 插入的飞书变量 |
|---------|----------------|
| `pg_table` | 固定手打 `qn_metric_sycm_high_conv`（不用变量） |
| `fields.keyword` | 第1步 → 搜索词 |
| `fields.stat_period_text` | 第1步 → 统计时间段 |
| `fields.cate_label` | 第1步 → 类目名称 |
| `fields.status` | 第1步 → 处理状态（即改完后的新值） |

> **不需要 `record_id`**。服务端用三列定位 PG 行（需代码 ≥ `7ca9ec1`）。

**响应配置（出参）**

留空 `{}` 即可，**无需处理**。本场景为单向通知，不解析 HTTP 响应。

#### 保存并启用

改一行「处理状态」→ 自动化 **运行历史** 应看到 `status_code: 200`，body 含 `"ok":true`。

---

## 三、ECS 服务器配置

### 3.1 目录与更新

```bash
# 仓库路径
/projects/server

# 日常更新（含 API + Worker 重启）
qn-update
```

首次安装 `qn-update`：

```bash
sudo install -m 755 /projects/server/deploy/qn-update /usr/local/bin/qn-update
```

### 3.2 编辑 `.env`

```bash
vi /projects/server/.env
```

在现有配置基础上，补充或确认以下项：

```bash
# --- 基础 ---
HOST=0.0.0.0
PORT=8080
API_BASE=http://8.163.126.172:8080
DATABASE_URL=postgres://qn:***@127.0.0.1:5432/qn
INGEST_API_KEY=你的随机密钥

# --- 飞书 OAuth（Refine 登录，与多维表 API 同一应用）---
FEISHU_APP_ID=cli_xxxxxxxx
FEISHU_APP_SECRET=xxxxxxxxxxxxxxxx
FEISHU_REDIRECT_URI=http://8.163.126.172:8080/auth/feishu/callback

SESSION_SECRET=随机长字符串
COOKIE_SECURE=false

# --- Worker ---
SYNC_AFTER_INGEST=true
WORKER_SCHEDULE_ENABLED=true
AGGREGATE_RANK_CRON=0 */6 * * *

# --- 飞书多维表 · 高转化监控 ---
FEISHU_BITABLE_APP_TOKEN=ADWJbOBPjaVPpEsB2ILcTNpvnbe
FEISHU_BITABLE_TABLE_METRIC_SYCM_HIGH_CONV=tblCqtaWNsvgeEs2

# PG 列 → 飞书列名（JSON 一行；列名与 Base 一致时可照抄）
FEISHU_BITABLE_FIELD_MAP_METRIC_SYCM_HIGH_CONV={"keyword":"搜索词","cate_label":"类目名称","stat_period_text":"统计时间段","conv_index":"转化指数","search_growth":"增速","pop_median":"人气中值","conv_rank":"组内排名","status":"处理状态"}

# Webhook 回写鉴权（自己生成，勿提交 Git）
# 生成示例：openssl rand -hex 32
FEISHU_WEBHOOK_TOKEN=在这里填你生成的随机字符串
```

> `FEISHU_WEBHOOK_TOKEN` **不是**飞书开放平台里的 Verification Token，是 qn-server 自定义的共享密钥。

### 3.3 数据库迁移

```bash
cd /projects/server
npm run db:migrate
```

需包含：

- `007_bitable_sync.sql` — sync_map、sync_log
- `009_sycm_high_conv_metric.sql` — 高转化指标表
- `010_sycm_high_conv_status.sql` — `status` 列

### 3.4 启动服务

`qn-update` 会自动：

1. `git pull`
2. `npm install` + `build:web`
3. `db:migrate`
4. 重启 API（`npm start`）和 Worker（`npm run dev:worker`）

手动确认进程：

```bash
pgrep -af "node apps/api/src/server.js"
pgrep -af "node apps/worker/src/worker.js"
```

### 3.5 健康检查

```bash
curl -s http://127.0.0.1:8080/health | python3 -m json.tool
```

期望：

```json
{
  "status": "ok",
  "bitable": {
    "sycm_high_conv": "configured"
  }
}
```

### 3.6 安全组

阿里云 ECS 安全组需放行 **8080**（飞书 HTTP 自动化从公网访问）。

---

## 四、PG → 飞书 同步

### 4.1 前置：PG 有数据

插件 ingest 后，需聚合高转化 Top10：

```bash
curl -X POST 'http://127.0.0.1:8080/api/sync/enqueue' \
  -H 'Content-Type: application/json' \
  -H 'X-API-Key: 你的INGEST_API_KEY' \
  -d '{"job":"aggregate.sycm.high_conv"}'
```

### 4.2 触发同步到飞书

```bash
curl -X POST 'http://127.0.0.1:8080/api/sync/enqueue' \
  -H 'Content-Type: application/json' \
  -H 'X-API-Key: 你的INGEST_API_KEY' \
  -d '{"job":"sync.bitable","data":{"metric":"sycm_high_conv"}}'
```

### 4.3 查看 Worker 日志

```bash
tail -f /projects/server/qn-worker.log
```

期望：

```
[sync.bitable] start { metric: 'sycm_high_conv', ... }
[sync.bitable] done { synced: 10, failed: 0, ... }
```

### 4.4 查看同步审计

```bash
curl -s 'http://127.0.0.1:8080/api/sync/log?limit=10' \
  -H 'X-API-Key: 你的INGEST_API_KEY' | python3 -m json.tool
```

或 Refine 后台查看同步日志。

### 4.5 验证 sync_map（可选）

```bash
psql "$DATABASE_URL" -c "
SELECT h.keyword, m.bitable_record_id, h.status
FROM qn_metric_sycm_high_conv h
JOIN qn_feishu_sync_map m ON m.pg_id = h.id
WHERE m.pg_table = 'qn_metric_sycm_high_conv'
LIMIT 5;
"
```

---

## 五、飞书 → PG 回写验证

### 5.1 手动 curl 测试

```bash
curl -X POST 'http://8.163.126.172:8080/webhook/feishu/bitable' \
  -H 'Content-Type: application/json' \
  -H 'X-Feishu-Webhook-Token: 你的FEISHU_WEBHOOK_TOKEN' \
  -d '{
    "pg_table": "qn_metric_sycm_high_conv",
    "fields": {
      "keyword": "飞书里某行的搜索词",
      "stat_period_text": "飞书里该行的统计时间段",
      "cate_label": "飞书里该行的类目名称",
      "status": "已处理"
    }
  }'
```

成功响应：

```json
{"ok":true,"updated":1,"pgId":"uuid...","pgTable":"qn_metric_sycm_high_conv"}
```

### 5.2 飞书 UI 测试

1. 在多维表改一行「处理状态」
2. 自动化 → **运行历史** → 查看 HTTP 状态码与响应
3. Refine 刷新高转化列表，对应词 `status` 应已更新

---

## 六、故障排查

| 现象 | 原因 | 处理 |
|------|------|------|
| `bitable_not_configured` | `.env` 缺 token/table_id | 补 `.env` 后 `qn-update` |
| `FieldNameNotFound` | 飞书列名与 field_map 不一致 | 对齐列名或改 field_map JSON |
| `NumberFieldConvFail` / `TextFieldConvFail` | 列类型与写入值不匹配 | 列改文本，或对齐格式 |
| sync 总是 `rank_daily` skip | Worker 未更新到 pg-boss v10 修复版 | `qn-update` 到最新 main |
| Webhook `invalid_payload`（无 reason） | ECS 代码过旧，仍要求 record_id | `qn-update` 到 ≥ `7ca9ec1` |
| `missing_ops_fields` | Body 缺 `fields.status` | 检查自动化 JSON |
| `record_not_found` | 三列与 PG 不完全一致 | 对照 PG 与飞书单元格文本 |
| `unauthorized` | Token 不一致 | Header 与 `.env` 对齐 |
| 自动化不触发 | 规则未启用 / 未监听处理状态 | 检查触发条件 |
| HTTP 连接失败 | 8080 未对公网开放 | 阿里云安全组放行 |
| timeout / 504 | 飞书自动化 HTTP 超时（≤5s） | 检查 PG 查询性能或 ECS 负载 |

### 字段调试 API

```bash
curl -s 'http://127.0.0.1:8080/api/sync/bitable-fields?metric=sycm_high_conv' \
  -H 'X-API-Key: 你的INGEST_API_KEY' | python3 -m json.tool
```

返回飞书实际字段名与期望 field_map，用于对齐列名。

---

## 七、配置检查清单

### 飞书

- [ ] 独立 Base 已创建，URL 可解析 app_token / table_id
- [ ] 8 列已建好，列名与 field_map 一致
- [ ] 开放平台同一应用已开 `bitable:app` 并已发布
- [ ] Base 已通过「添加文档应用」授权（可编辑）
- [ ] 自动化：修改记录时 · 仅处理状态 · HTTP POST 已启用
- [ ] Body 含 `pg_table` + `fields` 四变量
- [ ] Header 含 `X-Feishu-Webhook-Token`

### ECS

- [ ] `.env` 飞书 OAuth + Bitable + WEBHOOK_TOKEN 已填
- [ ] `npm run db:migrate` 已执行
- [ ] API + Worker 进程在跑
- [ ] `/health` 显示 `sycm_high_conv: configured`
- [ ] 安全组 8080 已放行
- [ ] 代码 ≥ `7ca9ec1`
- [ ] `sync.bitable` 已成功（飞书表有数据）
- [ ] Webhook curl 测试返回 `ok: true`

---

## 八、附录

### A. 默认 field_map（可不改）

```json
{
  "keyword": "搜索词",
  "cate_label": "类目名称",
  "stat_period_text": "统计时间段",
  "conv_index": "转化指数",
  "search_growth": "增速",
  "pop_median": "人气中值",
  "conv_rank": "组内排名",
  "status": "处理状态"
}
```

### B. 相关 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/webhook/feishu/bitable` | 飞书自动化回写 |
| POST | `/webhook/feishu/event` | 事件订阅（需 HTTPS） |
| POST | `/api/sync/enqueue` | 手动触发 aggregate / sync |
| GET | `/api/sync/log` | 同步日志 |
| GET | `/api/sync/bitable-fields` | 飞书字段调试 |
| GET | `/health` | 健康与 bitable 配置状态 |

### C. 方案 A（可选，需 HTTPS 域名）

若将来有 HTTPS 域名，可在开放平台配置事件订阅：

- URL：`https://你的域名/webhook/feishu/event`
- 事件：`drive.file.bitable_record_changed_v1`

则不必配多维表自动化，服务端会自动拉取「处理状态」回写 PG。

---

> **来源**：qn-server 项目文档 `docs/feishu-bitable-integration.md`
> **文档版本**：2026-06-05 · qn-server main ≥ 7ca9ec1
