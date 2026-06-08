# 网络知识

日常网络诊断与原理认知的笔记汇总。不是教科书，是实践导向的速查和理解记录。

---

| 章节 | 内容 |
|---|---|
| [01 — macOS 网络诊断工具](01-diagnostic-tools.md) | 从连通性到路由追踪的分层命令速查 |
| [02 — Traceroute 原理](02-traceroute.md) | TTL、ICMP Time Exceeded、静默跳的机制 |
| [03 — DNS 记录类型](03-dns-records.md) | A/AAAA/CNAME/MX/TXT/NS/PTR 详解 |

---

## 核心认知

- **IP 是无连接、逐跳转发的协议** — 每个路由器只知道下一跳，不知道整条路径
- **TTL** 是数据包的生存跳数计数器，每经过一跳减 1，减到 0 触发 ICMP Time Exceeded
- **跳数反映网络层次复杂度**，不是物理距离；时耗大头在出口带宽的串行化延迟，不在每跳的路由处理
- **静默跳** — 路由器配置为不回复 ICMP Time Exceeded，traceroute 无法获知该跳 IP
- **DNS 与路由是分离的两层** — DNS 解决域名→IP 的映射，IP 路由解决数据包怎么走
