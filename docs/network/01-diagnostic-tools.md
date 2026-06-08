# 01 — macOS 网络诊断工具

按层次归类，方便分层排查。

---

## 基础连通性

| 命令 | 用途 | 示例 |
|---|---|---|
| `ping <host>` | 测可达性、延迟、丢包 | `ping 8.8.8.8` |
| `nc -vz <host> <port>` | 测特定端口是否开放 | `nc -vz 192.168.1.1 80` |
| `nc -vz -G 3 <host> <port>` | 带连接超时的端口测试 | `nc -vz -G 3 8.163.126.172 22` |

---

## 对外出口信息

| 命令 | 用途 | 备注 |
|---|---|---|
| `curl -s ifconfig.me` | 本机公网 IP | HTTP 请求外部服务获取 |
| `curl -s ipinfo.io` | 公网 IP + 地理位置 + ASN + 运营商 | 更详细 |
| `traceroute -I -q 1 -w 1 <host>` | 到目标的每一跳及延迟 | `-I` 用 ICMP，`-q 1` 减探针数，`-w 1` 减超时 |
| `sudo mtr -n <host>` | 持续显示每跳丢包率+延迟 | 需 `brew install mtr`，`-r` 报告模式快速跑完 |

---

## 内网构成

| 命令 | 用途 | 示例/说明 |
|---|---|---|
| `arp -a` | 局域网活跃设备（IP + MAC） | 快速看内网有哪些设备 |
| `ifconfig` | 本机所有网络接口及 IP/MAC/状态 | `ifconfig en0 \| grep inet` 只看 WiFi IP |
| `networksetup -getinfo Wi-Fi` | WiFi 接口的 IP、子网掩码、网关、DNS | macOS 特有 |
| `netstat -rn` | 路由表 | 看所有路由路径 |
| `route -n get default` | 当前默认网关 | 最常用的出口网关 |

---

## 端口监听和连接

| 命令 | 用途 | 说明 |
|---|---|---|
| `lsof -i -P \| grep LISTEN` | 本机监听哪些端口 | 含进程名和 PID |
| `netstat -an \| grep LISTEN` | 同上另一种方式 | 系统自带，无需 sudo |
| `lsof -i :<port>` | 特定端口被什么进程占用 | e.g. `lsof -i :5432` |

---

## DNS

| 命令 | 用途 | 示例 |
|---|---|---|
| `scutil --dns` | 系统 DNS 配置、所有 resolver | 看当前用了哪些 DNS 服务器 |
| `dig <domain>` | DNS 查询（详细） | `dig baidu.com` |
| `nslookup <domain>` | DNS 查询（简洁） | `nslookup baidu.com` |
| `nslookup -type=<type> <domain>` | 查指定记录类型 | `nslookup -type=MX baidu.com` |
| `nslookup <ip>` | 反向查询：IP → 域名 | `nslookup 8.8.8.8` |

---

## 综合扫描

| 命令 | 用途 | 需安装 |
|---|---|---|
| `nmap -sn 192.168.1.0/24` | 扫描局域网活设备 | `brew install nmap` |
| `nmap -O <ip>` | 操作系统识别 | 同上 |
