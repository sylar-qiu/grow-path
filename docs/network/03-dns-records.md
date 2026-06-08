# 03 — DNS 记录类型

## 核心映射类

| 类型 | 作用 | 举例 |
|---|---|---|
| **A** | 域名 → IPv4 地址 | `baidu.com A 39.156.66.10` |
| **AAAA** | 域名 → IPv6 地址 | 双栈网站同时有 A 和 AAAA 记录 |

## 服务定位类

| 类型 | 作用 | 说明 |
|---|---|---|
| **CNAME** | 别名指向另一个域名 | `cdn.baidu.com CNAME cdn.aliyun.com`，浏览器得到别名后重新查 A 记录。好处是改 IP 时只改一处。注意：CNAME 不能和同名的其他记录共存 |
| **MX** | 指定域名的邮件服务器 | `baidu.com MX 10 mail.baidu.com`——优先级数字越小越优先。主服务器配高优先级，备服务器配低优先级 |
| **NS** | 指定该域名的权威 DNS 服务器 | `baidu.com NS ns1.baidu.com`。谁管这个域名的解析，由 NS 记录决定。换 DNS 服务商（如从 Namesilo 转到 Cloudflare）本质上就是改 NS 记录 |
| **SRV** | 指定特定服务的位置（含端口号） | 用于 SIP、LDAP 等特定协议的服务发现，日常用得少 |

## 验证类

| 类型 | 作用 | 典型用途 |
|---|---|---|
| **TXT** | 存储任意文本 | **域名所有权验证**：第三方平台要求你加一条指定内容的 TXT 记录来证明域名是你的；**SPF**：声明哪些服务器允许发 @你的域名的邮件；**DKIM**：邮件签名公钥验证 |

## 反向记录

| 类型 | 作用 | 说明 |
|---|---|---|
| **PTR** | IP → 域名（A 的反向） | 由机房/云服务商配。发邮件时，接收方 SMTP 服务器会反向查发件 IP 的 PTR——如果 PTR 指向的域名和发件域名不一致，直接标为垃圾邮件。有自管邮箱服务器必须配好 |

---

## 使用频率排序

1. **A** — 最基础
2. **CNAME** — 别名，改 IP 时只需改一处
3. **MX** — 涉及自管邮件就必须配
4. **TXT** — 域名验证 + 邮件安全（SPF/DKIM）
5. **AAAA** — 有 IPv6 用户时需要
6. **NS** — 只在换 DNS 服务商时才动，但错了就断网
7. **PTR** — 自管邮件服务器需要，日常用不到

---

## DNS 与路由的关系

**DNS 和路由是分离的两层：**

- **DNS**（应用层）— 把域名解析成 IP 地址。解析流程：本地 DNS → 根 DNS → 顶级域 NS → 权威 DNS
- **路由**（网络层）— 拿到 IP 后，数据包怎么走。查路由表 → 默认网关 → BGP 逐跳转发

例子：`nslookup ifconfig.me` 返回 IP（如 217.160.0.173），然后 `traceroute 217.160.0.173` 看到路由路径。两者各司其职，互不依赖。

---

## DNS 查询工具速查：nslookup

| 命令 | 作用 |
|---|---|
| `nslookup <domain>` | 查 A 记录（默认） |
| `nslookup -type=A <domain>` | 查 A 记录（显式指定） |
| `nslookup -type=AAAA <domain>` | 查 AAAA 记录 |
| `nslookup -type=MX <domain>` | 查 MX 记录 |
| `nslookup -type=NS <domain>` | 查 NS 记录 |
| `nslookup -type=TXT <domain>` | 查 TXT 记录 |
| `nslookup -type=CNAME <domain>` | 查 CNAME 记录 |
| `nslookup <ip>` | 反向查询（IP → 域名，即 PTR 记录） |

返回中的 "Non-authoritative answer" 表示该 DNS 服务器不是这个域名的官方（权威）服务器，只是缓存了其他源的查询结果。
