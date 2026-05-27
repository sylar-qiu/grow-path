# Apache Handler 全景

Handler 是 Apache 中决定"谁来处理这个请求"的核心机制。当请求经过所有 Hook（URL 映射、认证、授权）之后，最后一步是选择一个 Handler 来生成响应内容。

Handler 的分配方式：

```
SetHandler handler-name        # 目录/位置级别
AddHandler handler-name .ext   # 按文件扩展名
Action handler-name cgi-path   # 按 MIME 类型触发
```

---

## 一、静态文件服务

### default-handler（核心，内置）

最基础的 Handler。Apache 的核心行为：把文件系统上的文件内容作为 HTTP 响应直接返回。

```
SetHandler default-handler
```

内部实现：

```
1. Apache 检查文件是否存在（stat() 系统调用）
2. 检查文件权限（Apache 进程用户是否有读权限）
3. 根据扩展名查 MIME 类型映射表 → Content-Type header
4. 读取并输出文件内容

输出方式：
- 小文件（< 8KB）：mmap + write
- 大文件：sendfile() 系统调用（零拷贝）
  → 数据从磁盘 Page Cache → NIC 发送缓冲区
  → 完全绕过用户空间
  → 理论上限 ≈ 磁盘 IO 带宽
```

**特性：**
- Range Requests（断点续传）：`Accept-Ranges: bytes`，mod_negotiation 自动处理 If-Range
- ETag / Last-Modified + If-None-Match / If-Modified-Since → 304 Not Modified
- Precompressed files（预压缩文件）：如果 `.gz` 或 `.br` 版本存在且客户端支持，自动发压缩版（通过 MultiViews 或 mod_negotiation）

**适用场景：** HTML、CSS、JS、图片、字体、下载文件。几乎所有静态资源。

---

## 二、内容协商与类型映射

### type-map（mod_negotiation）

根据客户端请求头（Accept、Accept-Language、Accept-Encoding）选择最合适的文件版本返回。

```
SetHandler type-map

# 配合 Options +MultiViews 自动使用
Options +MultiViews
```

**多语言站点示例：**

假设你有：

```
index.html.en    → English version
index.html.zh    → 中文版本
index.html.ja    → 日文版本
index.html.var   → type-map 文件
```

`index.html.var` 内容：

```
URI: index.html

URI: index.html.en
Content-Type: text/html; charset=utf-8
Content-Language: en

URI: index.html.zh
Content-Type: text/html; charset=utf-8
Content-Language: zh

URI: index.html.ja
Content-Type: text/html; charset=utf-8
Content-Language: ja
```

客户端请求 `Accept-Language: zh` → Apache 返回 `index.html.zh`。

**原理：** 基于 RFC 2296 的透明内容协商。Apache 对每个变体计算一个"quality factor"，选最高的。

**性能影响：** 每次请求都要 stat() 所有变体文件。在变体很多（>10）时，会明显增加请求耗时。不推荐在高流量页面使用，除非真的需要。

---

## 三、目录索引

### autoindex（mod_autoindex）

当请求的路径是一个目录且没有 index 文件（index.html / index.php 等）时，Apache 生成一个目录文件列表页面。

```
<Directory /var/www/html/downloads>
    Options +Indexes
    SetHandler autoindex
</Directory>
```

**生成的内容：**

```
Index of /downloads/
[parent directory]
tools/           2026-05-28 14:30   -
docs/            2026-05-27 09:15   -
readme.txt       2026-05-26 18:00  2.4K
setup.exe        2026-05-25 11:22  15M
```

**可定制项：**

| 指令 | 作用 |
|---|---|
| `HeaderName HEADER.html` | 在列表顶部嵌入自定义 HTML |
| `ReadmeName README.html` | 在列表底部嵌入自定义 HTML |
| `IndexStyleSheet` | 自定义 CSS |
| `IndexOrderDefault` | 排序方式 |
| `AddIcon` / `AddAlt` | 不同文件类型显示不同图标 |
| `FancyIndexing` | 是否显示图标、大小、日期列 |

**安全注意事项：**

```
# 千万不要暴露包含敏感信息的目录
Options -Indexes           # 关闭目录列表
# 或
<Directory /var/www/html/admin>
    Deny from all
</Directory>
```

**性能：** 每次请求都做 `ls` 操作（readdir + stat 每个文件/子目录）。目录下文件超过几千个时，响应时间会显著增加。

---

## 四、服务端包含（SSI）

### server-parsed / server-parsed-html（mod_include）

在 HTML 文件中嵌入服务器端指令，在响应返回客户端之前处理。

```
AddHandler server-parsed .shtml
# 或让所有 .html 文件也支持 SSI
AddType text/html .shtml
AddHandler server-parsed .shtml
Options +Includes
```

**支持的指令：**

```
<!--#include virtual="/header.html" -->     # 包含其他文件
<!--#echo var="DATE_LOCAL" -->              # 输出当前时间
<!--#set var="user" value="admin" -->       # 设置变量
<!--#if expr="$REMOTE_ADDR = /192\\.168/ --> # 条件判断
  内网用户可见内容
<!--#else -->
  外网用户可见内容
<!--#endif -->

<!--#exec cgi="/counter.cgi" -->            # 执行 CGI 程序（需要 +ExecCGI 选项）
<!--#flastmod file="index.html" -->          # 文件最后修改时间
<!--#fsize file="download.zip" -->          # 文件大小
```

**性能影响：**

- SSI 文件即使没有变更也会定期重新解析
- 每个 SSI include 都是一个额外的文件系统操作
- `exec cmd` / `exec cgi` 会 fork 子进程（和 CGI 一样重）
- 现代站点几乎不用 SSI，被模板引擎（PHP / Jinja2 / JSX）取代

**现代替代方案：** Nginx SSI 模块、Edge Side Includes（ESI）、客户端模板。

---

## 五、原始 HTTP 输出

### send-as-is（mod_asis）

按原样发送文件内容，不做任何 HTTP 头处理。内容必须包含完整 HTTP 头。

```
AddHandler send-as-is .asis
```

**示例文件 `redirect.asis`：**

```
HTTP/1.1 301 Moved Permanently
Location: https://new-site.com/
Content-Type: text/html

<html><body>Moved</body></html>
```

**用途：**
- 自定义重定向
- 自定义 HTTP 状态码的响应
- 非标准 MIME 类型场景
- 单文件实现快速跳转，无需配置

**注意事项：** Apache 不会帮你做任何合法性检查。如果你写错了 HTTP 头格式，客户端收到的是语法错误的响应。

---

## 六、图像地图

### imap-file（mod_imagemap）

处理服务端图像地图（Server-Side Image Map）。浏览器在图片上点击某个坐标，服务器根据坐标范围决定跳转到不同 URL。

```
AddHandler imap-file .map
```

**文件格式 `nav.map`：**

```
default /default.html
rect /products.html 0,0 100,50
circle /about.html 150,25 20
poly /contact.html 100,50 200,80 150,120
point /faq.html 50,100
```

- `rect`：矩形区域（左上 x1,y1 右下 x2,y2）
- `circle`：圆形区域（圆心 x,y 半径 r）
- `poly`：多边形区域（三个以上顶点 x1,y1 x2,y2 ...）
- `point`：最近的点（兜底）

**工作原理：**

```
1. 浏览器加载包含 <img src="nav.map" ismap> 的页面
2. 用户点击图片 → 浏览器发送 GET /nav.map?x,y
3. Apache 解析坐标 → 匹配区域 → 302 重定向到对应 URL
```

**性能：** 每个请求都是简单内存计算，没有磁盘 IO。**已经几乎被废弃**，被客户端 JavaScript 图像地图（配合 CSS/JS）完全取代。

---

## 七、服务器状态 / 信息

### server-status（mod_status）

返回 Apache 服务器的运行时状态页面。

```
<Location "/server-status">
    SetHandler server-status
    Require ip 127.0.0.1
</Location>
```

**输出内容：**

```
Apache Server Status for example.com (via 127.0.0.1)

Server Version: Apache/2.4.62 (Unix) PHP/8.2 PHP-FPM
Server Built: 2026-01-15T14:22:00

Current Time: Thursday, 28-May-2026 14:30:00 CST
Restart Time: Thursday, 28-May-2026 08:00:00 CST
Parent Server Config. Generation: 3
Parent Server MPM Generation: 3
Server uptime: 6 hours 30 minutes
Server load: 0.42 0.35 0.28
Total accesses: 154203 - Total Traffic: 2.3 GB
CPU Usage: u5.2 s3.8 cu0 cu0 - .0386% CPU load

1.2 requests/sec - 5.3 kB/second - 4.2 kB/request
3 requests currently being processed, 12 idle workers

  __W__.........................................................
  ................................................................
  ..................................K................................
```

**关键信息：**
- `_` — 空闲 worker
- `W` — 正在发送响应
- `K` — 保持连接（keepalive）
- `R` — 正在读请求
- `S` — 正在启动中

**扩展模式：`?auto`** — 机器可读的 key=value 格式，适合被监控工具（Nagios、Prometheus）采集。

**安全：** 永远不要对外暴露。这会泄露你的精确流量、版本、子进程数量。

### server-info（mod_info）

显示 Apache 配置详情：加载了哪些模块、每个模块的配置指令、当前生效的配置值。

```
<Location "/server-info">
    SetHandler server-info
    Require ip 127.0.0.1
</Location>
```

**用途：** 调试时确认哪个配置在生效、追踪模块加载顺序、排查指令冲突。

**安全：** 绝对不要对外暴露。配置详情包含网站路径、模块版本、认证方式等敏感信息。

---

## 八、CGI（Common Gateway Interface）

### cgi-script（mod_cgi / mod_cgid）

每个请求 fork 一个外部进程来生成响应。历史最悠久的动态内容处理方式。

```
AddHandler cgi-script .cgi .pl .py .sh
```

**mod_cgi vs mod_cgid：**

| | mod_cgi | mod_cgid |
|---|---|---|
| 进程管理 | Apache 子进程直接 fork CGI | 独立的 cgid 守护进程统一 fork |
| 适用 MPM | prefork（主流） | worker / event（线程化 MPM） |
| 设计原因 | 线程安全限制 | 线程安全限制 → 用独立守护进程管理 |

**CGI 的完整流程：**

```
Apache 收到 .cgi 文件请求
  → 验证文件属性（可执行 + 所有者身份）
  → 构造环境变量（40+ 个 CGI 环境变量）
  → 重定向 stdin/stdout 到管道
  → fork() + exec() CGI 程序
  → 等待子进程退出（同步阻塞）
  → 从 stdout 管道读取响应
  → 发送给客户端
```

**环境变量列表（完整）：**

```
SERVER_SOFTWARE=Apache/2.4.62
SERVER_NAME=example.com
GATEWAY_INTERFACE=CGI/1.1
SERVER_PROTOCOL=HTTP/1.1
SERVER_PORT=443
REQUEST_METHOD=GET
PATH_INFO=/extra/path
PATH_TRANSLATED=/var/www/html/extra/path
SCRIPT_NAME=/cgi-bin/test.cgi
QUERY_STRING=key=value&page=2
REMOTE_ADDR=192.168.1.100
REMOTE_PORT=54321
REMOTE_HOST= (如果 HostnameLookups 开启)
CONTENT_TYPE= (POST 请求)
CONTENT_LENGTH= (POST 请求)
HTTP_HOST=example.com
HTTP_USER_AGENT=Mozilla/5.0 ...
HTTP_ACCEPT=text/html,...
HTTP_ACCEPT_LANGUAGE=zh-CN
HTTP_COOKIE=session=abc123
HTTPS=on
```

**性能瓶颈量化：**

```
CGI 脚本 "Hello World" 的请求耗时分解：
  Apache 请求解析：         ~20μs
  fork() + exec()：         ~500μs - 2ms
  Perl/Python 解释器启动：  ~10ms - 50ms
  脚本执行：                ~100μs
  输出 + 回写：             ~50μs
  ─────────────────────────────────
  总计：                   ~11ms - 52ms

对比 PHP-FPM 动态请求（无 OPcache，同机器）：
  总计：                   ~3ms - 8ms（因为解释器已常驻）

CGI 是解释器启动开销 + 进程创建开销双重叠加。
```

**每个进程内存：** Perl CGI ~10-20MB，Python CGI ~15-25MB。

**100 并发 CGI 请求 = 100 个 CGI 进程 = 1-2.5GB 内存。**

---

## 九、FastCGI（Proxy 模式）

### proxy:fcgi://（mod_proxy_fcgi）

Apache 通过 FastCGI 二进制协议与独立的语言进程池通信。**这是现代动态内容处理的标准方式。**

```
<FilesMatch \.php$>
    SetHandler "proxy:fcgi://127.0.0.1:9000"
</FilesMatch>

# 或者带文件路径
ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://127.0.0.1:9000/var/www/html/$1
```

**支持的 FastCGI 进程管理器：**

| 语言 | 进程管理器 | 监听地址 |
|---|---|---|
| PHP | PHP-FPM | 127.0.0.1:9000 或 /run/php/php8.2-fpm.sock |
| Python | uWSGI (FastCGI 模式) | 127.0.0.1:4000 |
| Python | Gunicorn (FastCGI 模式) | /run/gunicorn.sock |
| Ruby | Unicorn (FastCGI 模式) | /run/unicorn.sock |

**FastCGI 协议摘要：**

FastCGI 是二进制协议（不是 HTTP），使用记录（Record）传输：

```
Record header (8 bytes):
  Version (1B) | Type (1B) | RequestId (2B) | ContentLength (2B) | PaddingLength (1B) | Reserved (1B)

Type 定义：
  BEGIN_REQUEST   = 1    开始请求
  ABORT_REQUEST   = 2    中断请求
  END_REQUEST     = 3    请求结束
  PARAMS          = 4    环境变量（key=value 对，类似 CGI 环境变量）
  STDIN           = 5    POST body
  STDOUT          = 6    响应输出
  STDERR          = 7    错误输出
  DATA            = 8    额外数据（很少使用）
  GET_VALUES      = 9    查询管理器能力
  GET_VALUES_RESULT = 10 能力返回
  UNKNOWN_TYPE    = 11   未知类型
```

**性能对比（mod_php vs PHP-FPM FastCGI）：**

```
相同硬件，100 并发请求，PHP 执行 50ms：

mod_php (prefork 模式)：
  100 个 Apache 子进程 × 30MB = 3GB
  上下文切换：100 个进程竞争 CPU
  最大吞吐：~2000 req/s

PHP-FPM (dynamic, max_children=50)：
  50 个 FPM worker + Apache event 线程（轻量）
  50 × 30MB = 1.5GB（FPM）+ Apache ~50MB
  上下文切换：50 个进程竞争 CPU
  最大吞吐：~1000 req/s（受限于 50 个 worker）
  
  max_children 调大可以提升吞吐，但受 CPU 核心数和 PHP 执行时间限制
  50ms 执行时间 → 每个 worker 每秒处理 20 个请求
  50 workers → 1000 req/s
  100 workers → 2000 req/s（但内存 3GB）
```

**FastCGI 的关键优势：**

| 特性 | 说明 |
|---|---|
| **资源隔离** | PHP crash 只影响 FPM worker，不拉垮 Apache |
| **慢请求保护** | `request_terminate_timeout` 超时杀 worker |
| **进程池管理** | static / dynamic / ondemand 三种模式 |
| **慢日志** | 超过阈值自动 dump 调用栈 |
| **独立资源限制** | ulimit / rlimit 可以单独设置 |

---

## 十、通用反向代理

### proxy:http:// / proxy:https://（mod_proxy_http）

把请求转发给另一个 HTTP 服务器。最常用：反向代理到 Node.js / Java / .NET 应用。

```
ProxyPass /app http://localhost:3000/
ProxyPassReverse /app http://localhost:3000/

# 或者用 SetHandler 方式：
<Location "/api">
    SetHandler "proxy:http://127.0.0.1:8080"
</Location>
```

**连接复用：** 支持 Keep-Alive 连接到后端，避免每个请求都建立新 TCP 连接。

**HTTP 头转发：** 自动添加 `X-Forwarded-For`、`X-Forwarded-Proto`、`X-Forwarded-Host` 头。

**超时控制：**

```
ProxyTimeout 30           # 后端响应超时
ProxyPass /api http://backend/ timeout=60
```

**缓冲控制：**

```
ProxyPass /api http://backend/ disablereuse=on
```

### proxy:ajp://（mod_proxy_ajp）

Apache JServ Protocol——专门用于连接 Apache 和 Java 应用服务器（Tomcat / Jetty / JBoss）。

```
ProxyPass /webapp ajp://127.0.0.1:8009/
```

| 特性 | HTTP 代理 | AJP 代理 |
|---|---|---|
| 协议 | HTTP 1.1 | AJP 1.3（二进制） |
| 数据格式 | ASCII 文本 | 二进制（更紧凑） |
| 头传递 | 每个请求都传所有 header | **优化：** Tomcat 和 Apache 之间的长连接共享常见 header |
| 性能 | 略慢（解析开销大） | 略快（二进制协议） |
| 适用场景 | 通用后端 | Java 应用服务器 |

AJP 的优势在于它优化了请求头传递——常见的 header（Content-Type、Cache-Control 等）有预定义编码，不需要每次都传完整字符串。但在现代网络环境下（内网连接的延迟通常 <1ms），这个差异可以忽略。

### proxy:ws:// / proxy:wss://（mod_proxy_wstunnel）

WebSocket 反向代理——把 WebSocket 连接通过隧道传递给后端。

```
ProxyPass /ws/ ws://127.0.0.1:3000/ws/
ProxyPassReverse /ws/ ws://127.0.0.1:3000/ws/
```

**工作原理：**

```
1. 客户端发 HTTP Upgrade: websocket 请求
2. Apache 识别并建立到后端的 WebSocket 连接
3. Apache 开始双向隧道
   - 客户端 → Apache（加密 TCP）→ 后端（明文/加密）
   - 后端 → Apache → 客户端
4. Apache 在两个方向之间桥接数据，不解析内容
```

**注意：** Apache 的 WebSocket 代理是隧道模式，不是代理模式——数据流只是透传，不做任何解析或缓存。mod_proxy_wstunnel 在 Apache 2.4.5+ 可用。

### proxy:h2:// / proxy:h2c://（mod_proxy_http2）

HTTP/2 反向代理——把请求转发给支持 HTTP/2 的后端。

```
ProxyPass / h2c://127.0.0.1:8080/
```

- `h2://` — HTTP/2 over TLS
- `h2c://` — HTTP/2 cleartext（无需 TLS，但浏览器不支持，适合内部通信）

**说明：** Apache 作为 HTTP/2 前端非常成熟（mod_http2），但作为 HTTP/2 的代理客户端是较新的功能（Apache 2.4.50+ 才稳定）。

### proxy:balancer://（mod_proxy_balancer）

负载均衡——把请求分发到多个后端。

```
<Proxy "balancer://mycluster">
    BalancerMember http://192.168.1.10:8080
    BalancerMember http://192.168.1.11:8080
    BalancerMember http://192.168.1.12:8080
    ProxySet lbmethod=byrequests
</Proxy>

ProxyPass / balancer://mycluster/
```

**负载均衡算法：**

| 方法 | 说明 |
|---|---|
| `byrequests` | 轮询 (Round Robin)，默认 |
| `bytraffic` | 按流量比例分配 |
| `bybusyness` | 分配给最空闲的后端 |
| `heartbeat` | 基于后端心跳（需要 mod_slotmem_shm） |

**粘滞会话（Sticky Session）：**

```
ProxySet stickysession=ROUTEID
```

当 `ROUTEID` cookie 存在时，同一个 session 的所有请求路由到同一台后端服务器。这对于有本地状态的 Java/PHP 应用很重要。

**健康检查：** 可选方案：

```
# 被动检查（默认）— 后端返回 50x 错误才标记为 down
# 主动检查：
<Proxy "balancer://mycluster">
    BalancerMember http://192.168.1.10:8080
    ProxySet lbmethod=bytraffic
</Proxy>
# 需要配合 mod_proxy_hcheck 做主动健康检查
```

### proxy:ftp://（mod_proxy_ftp）

FTP 代理——通过 Apache 代理 FTP 协议访问。

```
ProxyPass /ftp/ ftp://ftp.example.com/
```

**限制：**
- 只支持读取（文件下载），不支持和 FTP 服务器的交互式控制
- 不支持 FTP over TLS（FTPS）
- 现代架构中很少使用

---

## 十一、嵌入式脚本语言模块

### php*_module（mod_php）

PHP 解释器编译为 Apache 模块，嵌入到 Apache 进程空间中运行。

```
# 方式一：通过 AddType + SetHandler
AddType application/x-httpd-php .php
SetHandler application/x-httpd-php

# 方式二：通过 FilesMatch（推荐）
<FilesMatch \.php$>
    SetHandler application/x-httpd-php
</FilesMatch>
```

**SAPI 名称：** `apache2handler`（不是 `fpm-fcgi`，也不是 `cli`）

**生命周期（与 Apache 进程绑定）：**

```
Apache 子进程启动 → PHP 解释器初始化一次
  → 加载 php.ini
  → 加载所有扩展（mysql, openssl, gd, curl...）
  → 初始化内存池

每个请求：
  → SAPI 激活
  → 编译/OPcache 命中
  → 执行
  → 清理（但不释放内存池，保持分配以备重用）

子进程退出（MaxRequestsPerChild 或 crash）→ PHP 释放所有资源
```

**内存泄漏问题：**

```
PHP 脚本中的内存泄漏模式：
  - 静态变量自增：function count() { static $c = 0; $c++; ... }
  - 全局变量：$GLOBALS['data'][] = $largeValue
  - 循环引用但没有垃圾回收

后果：Apache 子进程 RSS 持续增长
解决：MaxRequestsPerChild 定期重启子进程
  MaxRequestsPerChild 10000
  → 每处理 10000 个请求后，子进程自动退出，父进程 fork 新的
```

**局限：**
- 只能用 prefork MPM（PHP 非线程安全）
- 一个进程 crash 拉垮整个子进程，影响该进程上的所有连接
- 内存占用高（每个子进程都有一份 PHP 解释器）
- 已不是推荐配置方案

### perl-script（mod_perl）

Perl 解释器嵌入 Apache，支持完整的 Perl 环境下开发 Web 应用。

```
PerlModule Apache::Registry
<FilesMatch \.pl$>
    SetHandler perl-script
    PerlResponseHandler ModPerl::Registry
    Options +ExecCGI
</FilesMatch>
```

**能力：**

| 特性 | mod_cgi (perl) | mod_perl |
|---|---|---|
| 解释器 | 每个请求 fork+exec | 嵌入 Apache，常驻 |
| 性能 | 慢（进程创建 + 解释器启动） | 快（预编译 + 缓存） |
| Perl 模块 | 每次重新加载 | 启动时一次加载 |
| 数据库连接 | 每次重连 | 连接池复用 |
| Apache API 访问 | 不能 | 可以直接操作 Apache 请求对象 |

**Apache API 直接访问（mod_perl 独有）：**

```perl
# 可以直接读写 Apache 内部数据结构
my $r = Apache->request;
$r->content_type('text/html');
$r->headers_out->set('X-Custom', 'value');
$r->print("<html>Hello World</html>");
```

**现状：** 在 Perl 社区仍然活跃（特别是 Catalyst / Dancer 框架用户），但整体使用量持续下降。Perl 在 Web 领域的份额被 PHP / Python / Ruby / Go 替代。

### wsgi-script（mod_wsgi）

Python WSGI（Web Server Gateway Interface）应用嵌入 Apache。

```
# 守护进程模式（推荐）
WSGIDaemonProcess myapp python-path=/var/www/myapp
WSGIProcessGroup myapp

<VirtualHost *:443>
    WSGIScriptAlias / /var/www/myapp/myapp.wsgi
</VirtualHost>
```

**两种模式：**

| 模式 | 说明 | 内存 |
|---|---|---|
| **嵌入式（Embedded）** | Python 解释器嵌入 Apache 子进程 | 和 Apache 共享进程，内存占用高 |
| **守护进程（Daemon）** | 独立的 Python 进程池，通过 socket 通信 | 隔离性好，Apache crash 不影响 Python |

**WSGI 应用协议：**

```python
# 最小的 WSGI 应用
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return [b'Hello World']
```

`environ` 包含所有 CGI 环境变量，`start_response` 是回调函数。Apache 通过 mod_wsgi 把 HTTP 请求翻译成 WSGI 调用。

**现状：** Django / Flask / FastAPI 在部署时，现代做法是通过 Gunicorn / uWSGI 与 Nginx 配合，mod_wsgi 的使用量在下降，但仍然是大规模 Python 部署的可靠选择。

### python-script（mod_python，已废弃）

Python 的早期 Apache 模块（1997-2010），直接让 Python 代码作为 Apache handler 运行。

```
AddHandler python-script .py
PythonHandler my_handler
```

**缺点：**
- 与 Apache 内部版本号绑定（升级 Apache 就要重新编译）
- 解释器初始化代价高
- 安全设计不完善
- **Apache 2.4 起不维护，完全不可用**

**不要在新项目中使用。** 所有 python-script 的需求应该用 mod_wsgi 替代。

---

## 十二、动作触发

### action（mod_actions）

根据 MIME 类型或文件扩展名，自动触发一个 CGI 脚本处理请求。

```
Action image-processor /cgi-bin/process-image.cgi
AddHandler image-processor .jpg .png .gif
```

**工作原理：**

```
1. 请求 GET /photos/photo.jpg
2. Apache 检测到 .jpg → 匹配 image-processor handler
3. Action 指令映射到 /cgi-bin/process-image.cgi
4. Apache 执行 CGI 脚本，把请求信息传给它
5. CGI 脚本的响应作为最终响应返回
```

**用途：**
- 上传文件后自动处理
- 图片按需缩放
- URL 级别的重定向逻辑
- 自定义日志分析

**注意：** Action handler 和普通 CGI 的区别是——Action 不直接暴露到 URL 路径，而是通过文件类型间接触发。

---

## 十三、过滤链作为 Handler

Apache 的 Handler 和 Filter 是两套独立的系统，但有交叉。某些 Handler 本质上依赖 Filter 链。

### 通过 AddOutputFilter 设置

```
AddOutputFilter INCLUDES .shtml    # 输出前经过 SSI 过滤器
AddOutputFilter DEFLATE .html       # 输出前 gzip 压缩
AddOutputFilter SUBSTITUTE .html    # 输出前文本替换
```

**注意事项：**

```
# Filter 不改变 MIME 类型——文件仍然是 text/html
# Handler 可以改变响应——CGI handler 把 .cgi 变成动态内容

两者可以叠加：
  AddHandler cgi-script .cgi        # CGI Handler
  AddOutputFilter DEFLATE .cgi      # 输出再经过压缩

处理顺序：Handler → Output Filter Chain
```

### mod_filter（Filter 链配置）

```
FilterDeclare CUSTOM_FILTER
FilterProvider CUSTOM_FILTER SUBSTITUTE "%{req:Accept-Language} =~ /zh/"
FilterChain CUSTOM_FILTER
```

更高级的 Filter 用法，根据条件决定哪些 Filter 应用到响应上。

---

## 十四、HTTP/2 协议支持

### mod_http2（协议升级，非传统 Handler）

mod_http2 不是传统意义上的 Handler——它不处理某个文件类型，而是在连接层面把 HTTP/1.1 升级到 HTTP/2。

```
Protocols h2 http/1.1    # 优先使用 HTTP/2
```

**工作机制：**

```
客户端的第一个请求是 HTTP/1.1 → 带 Upgrade: h2c 头
Apache 回复 101 Switching Protocols → 切换到 HTTP/2

或者通过 TLS ALPN 协商：
  Client Hello → 支持 h2
  Server Hello → 选择 h2
  → 直接走 HTTP/2
```

**对 Handler 的影响：** 同一物理连接上，HTTP/2 可以同时发送多个请求。Apache 内部把每个 stream 映射成一个独立的请求对象，走和 HTTP/1.1 完全相同的 Handler 选择逻辑。

---

## 十五、TLS/SSL

### mod_ssl（连接级安全，非 Handler）

mod_ssl 处理 TLS 握手、证书验证、加密解密。它在 Handler 之前工作——先建立安全连接，然后才把解密后的请求交给 Handler。

```
SSLEngine on
SSLCertificateFile /etc/ssl/certs/example.com.crt
SSLCertificateKeyFile /etc/ssl/private/example.com.key
```

**TLS 握手步骤（简化）：**

```
Client Hello → Server Hello + Certificate → Key Exchange → Finished
                        ↓
             加密通道建立后
              → HTTP 请求在此通道上传输
              → Apache Handler 层完全不知道 TLS 的存在
              → mod_ssl 在底层透明完成加解密
```

---

## 十六、完整的 Handler 注册与匹配顺序

当一个请求到达 Apache 时，Handler 的选择遵循以下优先级：

```
优先级从高到低：

1. SetHandler 显式指定（配置文件或 <Location> <Files>）
2. AddHandler 扩展名匹配
3. Action MIME 类型匹配
4. default-handler（兜底）
```

**SetHandler 和 AddHandler 的区别：**

```
# SetHandler — 目录/位置下所有文件都走同一个 Handler
<Location "/api">
    SetHandler "proxy:http://127.0.0.1:3000"
</Location>

# AddHandler — 按扩展名指定 Handler（可以和 SetHandler 混合使用）
AddHandler cgi-script .cgi
AddHandler server-parsed .shtml
```

**同一个文件匹配多个 Handler 时的处理：**

如果一个文件同时匹配了多个 AddHandler 规则（很少见），Apache 使用**最后一个匹配的**。

**Handler 不存在的处理：**

如果一个请求匹配到的 Handler 名称没有对应的模块加载，Apache 返回 500 Internal Server Error，日志记录 `unknown handler name`。

---

## 十七、Handler 选择流程图

```
请求到达 Apache
  ↓
URL → 文件路径映射 (translate_name)
  ↓
路径映射完成
  |
  ├─ SetHandler 显式指定？ ──→ 用指定 Handler
  |
  ├─ AddHandler 扩展名匹配？ ──→ 用匹配的 Handler
  |
  ├─ Action MIME 类型匹配？ ──→ 执行对应的 CGI
  |
  ├─ ProxyPass 匹配？ ──→ 用对应 proxy handler
  |
  ├─ ServerAlias/Redirect 匹配？ ──→ 用 redirect handler
  |
  └─ 都没有？ → 检查是否是目录
       ├─ 是目录，有 Indexes → autoindex handler
       ├─ 是目录，有 index.html → default-handler 服务 index.html
       └─ 是文件 → default-handler 服务文件内容
  ↓
Handler 执行 → 经过 Output Filter 链 → 输出给客户端
```

---

## 十八、各 Handler 的现代适用场景汇总

| Handler | 适用场景 | 状态 |
|---|---|---|
| `default-handler` | 所有静态文件 | ✅ 核心 |
| `type-map` | 多语言/多格式内容协商 | ⚠️ 少用 |
| `autoindex` | 文件下载目录 | ✅ 常用 |
| `server-parsed` | 遗留系统 | ❌ 不推荐 |
| `send-as-is` | 自定义状态码页面 | ⚠️ 少数场景 |
| `imap-file` | — | ❌ 已废弃 |
| `server-status` | 运维监控 | ✅ 必开 |
| `server-info` | 配置调试 | ⚠️ 仅内网 |
| `cgi-script` | 极简轻量脚本 | ⚠️ 少数场景 |
| `proxy:fcgi://` | PHP 动态内容 | ✅ 推荐 |
| `proxy:http://` | Node/Python/Go/etc 反向代理 | ✅ 推荐 |
| `proxy:ajp://` | Java Tomcat 后端 | ✅ 遗留系统 |
| `proxy:ws://` | WebSocket 代理 | ✅ 推荐 |
| `proxy:h2://` | HTTP/2 内部通信 | ⚠️ 新兴 |
| `proxy:balancer://` | 多后端负载均衡 | ✅ 推荐 |
| `proxy:ftp://` | FTP 代理 | ❌ 几乎不用 |
| `php*_module` | 遗留 PHP 项目 | ❌ 不推荐 |
| `perl-script` | Perl 遗留应用 | ⚠️ 少数场景 |
| `wsgi-script` | Python 遗留部署 | ⚠️ 少数场景 |
| `action` | 按 MIME 触发 CGI | ⚠️ 少数场景 |
