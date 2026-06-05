# 后端集成

qn-server 后端服务与外部平台（飞书、数据库等）的集成经验。

> 这部分内容来自实际业务中踩过的坑和趟平的路，聚焦于**双向数据流**和**HTTP + IP 环境下的变通方案**。

---

| 文章 | 内容 | 场景 |
|------|------|------|
| [飞书多维表 × qn-server 连调配置手册](feishu-bitable-integration.md) | PG ↔ 飞书多维表双向同步：架构、飞书配置、ECS 部署、Webhook 回写、故障排查 | 高转化关键词监控运营流程 |

---

## 相关背景

- **ECS 服务器**：`8.163.126.172:8080`，阿里云，无 HTTPS 域名
- **数据库**：PostgreSQL（插件数据架构 v2），通过内部 API 暴露
- **前端**：Refine 后台，飞书 OAuth 登录
- **代码仓库**：`/projects/server`（qn-server，Fastify + Worker 架构）
