# DNS 协议：域名系统

## 协议定位

DNS 将人们易记的域名（baidu.com）映射为机器可用的 IP 地址（39.156.66.10）。没有 DNS，你要访问一个网站就只能记住并输入 IP 地址——这在只有几个站点时可行，在今天的互联网规模下不现实。

DNS 不在 OSI 七层模型的某个固定层级上，通常归类于**应用层**（使用 UDP 53 端口作为默认传输，必要时用 TCP 53 端口处理大数据量的响应如区域传输）。

## 域名的层级结构

一个完整的域名长这样：

```
www.baidu.com.
```

末尾的 `.` 是**根域**（root zone），通常省略不写。层级从右往左读：

```
.         根域（root）                   → 由 ICANN 管理
.com      顶级域（TLD）                   → 由 Verisign 等注册局运营
baidu     注册域名（second-level domain）→ 由域名注册人持有
www       主机名（subdomain）             → 属于 baidu.com 的管理者自行配置
```

每一级都有自己的权威 DNS，负责回答其下一级的查询。

### DNS 的树状委托结构

```
根服务器（13 组 Anycast 集群）
  └── .com 的权威服务器
        └── baidu.com 的权威服务器（你自己或 Cloudflare/NS1 等 DNS 服务商）
              └── www.baidu.com（一个 A 记录或 CNAME 记录）
```

每一级的安全机制是：上一级给你签发数字签名，保证下级数据没有被篡改（DNSSEC，还没广泛部署）。

## 解析流程：递归查询

当你访问 baidu.com 时，完整的解析流程如下：

```
你的浏览器
  │ 查询 "baidu.com"
  ▼
本地递归 DNS（通常是 192.168.1.1 或 8.8.8.8）
  │ 1. 找根服务器：".com 在哪？"
  ▼
根服务器（13 组集群）
  │ 回复：".com 的权威服务器是 a.gtld-servers.net"
  ▼
本地递归 DNS
  │ 2. 找 .com 服务器："baidu.com 在哪？"
  ▼
.com 顶级域服务器
  │ 回复："baidu.com 的权威服务器是 ns1.baidu.com"
  ▼
本地递归 DNS
  │ 3. 找 baidu.com 的权威 DNS："www.baidu.com 的 A 记录是什么？"
  ▼
baidu.com 权威 DNS（由百度自管或使用 DNS 服务商）
  │ 回复："39.156.66.10"（A 记录）
  ▼
本地递归 DNS
  │ 4. 将结果返回给浏览器，并缓存该结果（TTL 决定缓存时长）
  ▼
浏览器用 39.156.66.10 发起 TCP 连接
```

这里的每一步都是**真实的全路径**。第一次解析一个域名为啥会感觉慢，就是因为这一路问下来要跑好几个来回。

### Non-authoritative answer

当你用 nslookup 得到"Non-authoritative answer"时，说明你问的 DNS 服务器不是该域名的权威 DNS，而是它从其他服务器缓存过来的答案。这并不表示答案错误——缓存的答案和权威答案在 TTL 内是一致的。

权威回复则来自该域名注册的 Nameserver 自己，是"一手消息"。

## DNS 记录类型

### A 记录（Address Record）

域名 → IPv4 地址，DNS 最核心的记录类型。每个域名可以有多条 A 记录（多 IP）实现负载均衡或故障切换。

### AAAA 记录

域名 → IPv6 地址。功能与 A 记录相同，只是记录的是 128 位的 IPv6 地址而非 32 位的 IPv4。

### CNAME 记录（Canonical Name Record）

域名别名，将一个域名指向另一个域名而非直接指向 IP。

```
cdn.example.com  CNAME  cdn.cloudflare.com
```

浏览器访问 cdn.example.com 时，先查到 CNAME 记录，然后去查 cdn.cloudflare.com 的 A 记录，拿到真正的 IP。

**优势：** 如果更换 CDN 供应商，只需修改一条 CNAME 记录指向新服务商，不需要一个个更新所有使用该域名的资源。

**限制：** 域名的根（如 example.com 本身）不能是 CNAME，必须是 A/AAAA 记录（这是 DNS 规范的要求，部分 DNS 服务商通过 ANAME/ALIAS 模拟实现）。

### MX 记录（Mail Exchange Record）

指定处理该域名邮件的服务器地址。每条 MX 记录有优先级数字，数字越小越优先。

```
example.com  MX  10  mail.primary.example.com
example.com  MX  20  mail.backup.example.com
```

发件服务器会先尝试优先级 10 的服务器，不可用时才用 20（备选）。实际发送过程：发送方先通过 MX 记录找到目标域的邮件服务器 IP，再通过 SMTP 协议投递邮件。

### TXT 记录（Text Record）

存储任意文本信息，最初设计用于存放人类可读的注释，但现代 DNS 中广泛应用于：

- **SPF（Sender Policy Framework）**：声明哪些 IP 地址被授权为你的域名发送邮件
- **DKIM（DomainKeys Identified Mail）**：发布邮件签名公钥，接收方验证邮件是否被篡改
- **域名所有权验证**：第三方平台要求你在域名下添加一条特定内容的 TXT 记录，证明你拥有该域名
- **DMARC 策略**：告诉接收方如何处理未通过 SPF/DKIM 验证的邮件

### NS 记录（Nameserver Record）

指定谁是这个域名的权威 DNS 服务器。当你在注册商（如 Namesilo、GoDaddy）买了一个域名，注册商会将 NS 记录指向它们自己的 DNS 服务器。如果你要使用 Cloudflare 管理 DNS，你需要将 NS 记录改为指向 Cloudflare 的 Nameserver。

**关键点：** 如果不改 NS 记录，无论你在 Cloudflare 上配了什么 A 记录都不会生效——因为域名解析时根本不会去 Cloudflare 问。

### PTR 记录（Pointer Record）

A 记录的反向操作：IP → 域名。由网络管理员（通常是云服务商或自有机房）配置。

**邮件服务器的硬性要求：** 接收方 SMTP 服务器收到邮件后，会反向查询发件服务器 IP 的 PTR 记录。如果 PTR 指向的域名与声明发件域名不一致，该邮件会被判定为垃圾邮件并拒绝。配置邮件服务器时，必须同时向服务商申请 PTR 记录。

## TTL（记录的有效期）

每条 DNS 记录有一个 TTL（Time To Live），单位秒。它告诉递归 DNS 服务器和浏览器可以缓存这条记录多久，不要频繁回源查询。

| TTL 值 | 典型场景 |
|---------|----------|
| 30-60s | 高频变更的场景（CDN 切流、故障切换），但会增加 DNS 查询负担 |
| 300-600s（5-10min） | 日常推荐的折中值 |
| 3600-86400s（1h-1d） | 稳定的生产服务，降低 DNS 查询压力 |
| 无（0） | 开发测试时用的 no-cache 值，关闭缓存（大多数 DNS 服务器不严格遵循） |

**变更前降低 TTL：** 如果你要在 DNS 服务商后台改动记录，先提前一天把 TTL 降为 60s，等全网缓存刷新后再改配置，这样切换过程几秒内生效。改完后再把 TTL 调回去。

## 操作速查

```
nslookup baidu.com                      # 查询 A 记录
nslookup -type=A baidu.com              # 显式指定查 A 记录
nslookup -type=MX gmail.com             # 查邮件服务器
nslookup -type=NS baidu.com             # 查权威 DNS 服务器
nslookup -type=TXT _dmarc.baidu.com     # 查 DMARC 策略
nslookup 8.8.8.8                        # 反向查询（IP→域名）
nslookup -type=ANY baidu.com            # 查询所有可用记录（逐步弃用）

dig baidu.com                    # 更详细的 DNS 查询，显示完整回复报文
dig www.baidu.com +short         # 只返回 IP（简洁模式）
dig baidu.com MX                 # 查 MX 记录
dig baidu.com NS                 # 查 NS 记录
dig @8.8.8.8 baidu.com           # 指定用 8.8.8.8 查询
dig +trace baidu.com             # 显示完整递归过程（从根到域名）

scutil --dns                     # macOS 系统 DNS 配置
```

## 思考题

1. "Non-authoritative answer" 意味着 DNS 结果不可信吗？
2. 为什么域名根记录（如 example.com 本身）不能直接使用 CNAME 记录？
3. 如果一台服务器更换了公网 IP 但 DNS 的 TTL 还没有过期，客户端会有什么表现？
4. MX 记录中的优先级数字如何决定了邮件投递的容错能力？
