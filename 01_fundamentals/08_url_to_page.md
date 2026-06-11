# 输入 URL 到页面显示：发生了什么？

## TL;DR

经典全栈考题，考察你对整个 Web 技术栈的理解深度。答案分 7 个阶段：DNS 解析 → TCP 握手 → TLS 握手 → HTTP 请求 → 服务端处理 → 浏览器渲染 → 页面呈现。

---

## 完整流程图

```mermaid
flowchart TD
    Input["用户输入 https://www.google.com/search?q=hello"] --> Cache["① 浏览器缓存\n检查本地 DNS 缓存"]
    Cache -- 命中 --> IP["获得 IP 地址"]
    Cache -- 未命中 --> DNS["② DNS 解析\n递归查询"]
    DNS --> IP

    IP --> TCP["③ TCP 三次握手\nSYN → SYN-ACK → ACK"]
    TCP --> TLS["④ TLS 握手\n证书验证 + 密钥交换"]
    TLS --> HTTP["⑤ HTTP 请求\nGET /search?q=hello HTTP/2"]
    HTTP --> Server["⑥ 服务端处理\n负载均衡 → 应用服务器 → DB/Cache"]
    Server --> Response["HTTP 响应\n200 OK + HTML"]
    Response --> Parse["⑦ 浏览器渲染\n解析 HTML/CSS/JS"]
    Parse --> Display["页面呈现"]
```

---

## ① DNS 解析

```mermaid
sequenceDiagram
    participant B as 浏览器
    participant LC as 本地缓存
    participant OS as 操作系统 /etc/hosts
    participant R as 本地 DNS 服务器（ISP）
    participant Root as 根 DNS 服务器
    participant TLD as .com TLD 服务器
    participant Auth as google.com 权威 DNS

    B->>LC: 查 www.google.com
    LC-->>B: 未命中
    B->>OS: 查 /etc/hosts
    OS-->>B: 未命中
    B->>R: 递归查询 www.google.com
    R->>Root: 查 .com NS 在哪
    Root-->>R: .com NS = a.gtld-servers.net
    R->>TLD: 查 google.com NS
    TLD-->>R: google.com NS = ns1.google.com
    R->>Auth: 查 www.google.com A记录
    Auth-->>R: 142.250.x.x, TTL=300
    R-->>B: 142.250.x.x（同时缓存 TTL 秒）
```

**关键点：**
- 浏览器缓存 → OS 缓存 → 本地 DNS → 根 DNS → TLD DNS → 权威 DNS（层层查找）
- TTL 控制缓存时间；CDN 靠低 TTL（几十秒）实现快速切换
- `hosts` 文件优先级高于 DNS（本地开发 `127.0.0.1 myapp.local`）

---

## ② TCP 三次握手

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    C->>S: SYN（seq=100）"我要连接"
    S-->>C: SYN-ACK（seq=200, ack=101）"好的，我准备好了"
    C->>S: ACK（ack=201）"收到，开始通信"
    Note over C,S: 连接建立，耗时 1 RTT
```

**为什么是三次而不是两次？**

两次握手服务器无法确认客户端能收到自己的包。三次握手双方各验证一次"对方能收到我发的包"。

**HTTP/2 的优化：** 同一个 TCP 连接复用多个请求（多路复用），不需要为每个资源重新握手。

---

## ③ TLS 握手（HTTPS）

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    C->>S: ClientHello（支持的 TLS 版本、加密套件列表、随机数 C）
    S-->>C: ServerHello（选定加密套件、随机数 S）
    S-->>C: Certificate（服务器证书，含公钥）
    S-->>C: ServerHelloDone
    C->>C: 验证证书（CA 签名链、域名、有效期）
    C->>S: ClientKeyExchange（用服务器公钥加密预主密钥）
    C->>S: ChangeCipherSpec + Finished
    S->>C: ChangeCipherSpec + Finished
    Note over C,S: 对称密钥 = f(随机数C, 随机数S, 预主密钥)\n后续通信用对称加密（快）
```

**TLS 1.3 的优化：** 1-RTT 完成握手（比 TLS 1.2 的 2-RTT 快），支持 0-RTT 恢复（再次连接时 0 延迟）。

---

## ④ HTTP 请求

```
GET /search?q=hello HTTP/2
Host: www.google.com
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, br       ← 支持压缩
Accept-Language: zh-CN,zh;q=0.9
Cookie: SIDCC=xxx; NID=xxx      ← 会话信息
Cache-Control: no-cache
User-Agent: Mozilla/5.0 ...
```

**HTTP/2 vs HTTP/1.1 关键区别：**

| | HTTP/1.1 | HTTP/2 |
|---|---|---|
| 传输格式 | 文本 | 二进制帧（Binary Frame）|
| 多路复用 | 不支持，队头阻塞 | 支持，同一连接并发多请求 |
| 头部压缩 | 无 | HPACK 压缩（重复头部只发差量）|
| 服务器推送 | 不支持 | 支持（Server Push）|

---

## ⑤ 服务端处理

```mermaid
flowchart TD
    Req["HTTP 请求"] --> LB["负载均衡\nL7 (Nginx/Envoy)"]
    LB --> Cache1{"CDN / Edge Cache\n静态资源命中?"}
    Cache1 -- 命中 --> Return1["直接返回（< 10ms）"]
    Cache1 -- 未命中 --> AppServer["应用服务器集群"]
    AppServer --> Cache2{"Redis 缓存\n查询结果命中?"}
    Cache2 -- 命中 --> Return2["返回缓存结果"]
    Cache2 -- 未命中 --> DB["数据库查询"]
    DB --> Cache2Write["写入 Redis 缓存"]
    Cache2Write --> Return3["返回结果"]

    AppServer --> Auth["认证服务\nJWT 验证 / Session"]
    AppServer --> Micro["下游微服务\ngRPC 调用"]
```

**Google 搜索的特殊之处：**
- 查询不打 DB，而是查**倒排索引**（预构建的搜索索引集群）
- 结果排序用 PageRank + 机器学习模型打分
- 整个查询路径 < 200ms（含网络传输）

---

## ⑥ 浏览器渲染流水线

```mermaid
flowchart LR
    HTML["HTML 字节流"] --> DOM["解析 HTML\n构建 DOM Tree"]
    CSS["CSS 字节流"] --> CSSOM["解析 CSS\n构建 CSSOM"]
    DOM --> RenderTree["合并\nRender Tree\n只含可见节点"]
    CSSOM --> RenderTree
    RenderTree --> Layout["Layout（回流）\n计算每个节点\n的位置和大小"]
    Layout --> Paint["Paint（重绘）\n绘制像素到图层"]
    Paint --> Composite["Composite\n合并图层\n送显卡渲染"]
    Composite --> Screen["显示器"]
```

### 关键渲染路径（Critical Rendering Path）

```
HTML → DOM → Render Tree → Layout → Paint → Display

阻塞因素：
  ① CSS 阻塞渲染：浏览器拿到全部 CSS 才能构建 CSSOM，再才能渲染
     → 优化：<link> 放 <head>，内联关键 CSS
  ② JS 阻塞解析：遇到 <script> 立即停止解析 HTML，执行 JS
     → 优化：<script defer>（下载不阻塞，DOM 解析完后执行）
            <script async>（下载不阻塞，下载完立即执行）
```

### 性能指标

```
FCP  (First Contentful Paint)：首次有内容出现 < 1.8s
LCP  (Largest Contentful Paint)：最大内容绘制 < 2.5s（Core Web Vital）
TTI  (Time to Interactive)：可交互时间 < 3.8s
CLS  (Cumulative Layout Shift)：布局稳定性 < 0.1（Core Web Vital）
```

---

## 全程时间线

```
0ms       用户按回车
0-2ms     浏览器检查缓存、hosts 文件
2-50ms    DNS 解析（有缓存则 < 1ms）
50-100ms  TCP 三次握手（1 RTT，假设 RTT=50ms）
100-250ms TLS 1.3 握手（1 RTT）
250ms     发出 HTTP 请求
250-500ms 服务端处理（含网络传输）
500ms     收到 HTML 第一字节（TTFB）
500-800ms 解析 HTML、CSS，加载关键资源
800ms     FCP：页面有内容出现
1-3s      加载 JS、图片，页面完全可交互
```

---

## 面试追问

**Q: HTTP 和 HTTPS 的区别？**

HTTPS = HTTP + TLS。TLS 提供：①加密（防窃听）②完整性（防篡改，HMAC）③身份验证（证书确认服务器身份）。HTTP 是明文，中间人可看到全部内容。

**Q: 浏览器怎么知道证书是真的？**

证书由 CA（证书颁发机构）签名，浏览器内置信任的根 CA 列表。验证链：网站证书 → 中间 CA → 根 CA。自签名证书不在信任链里，所以浏览器报警告。

**Q: 同一个 IP 怎么部署多个 HTTPS 网站（虚拟主机）？**

TLS 的 **SNI（Server Name Indication）**：在 ClientHello 里带上域名，服务器根据域名选择对应的证书。这样一个 IP 可以服务多个 HTTPS 域名。

**Q: 为什么有时候刷新页面还是用旧的 CSS？**

浏览器缓存了旧 CSS 文件（`Cache-Control: max-age=31536000`）。解决方法：**Cache Busting** — 文件名加内容哈希（`style.a1b2c3.css`），内容变了文件名变了，强制加载新文件。

**Q: HTTP/3 解决了什么问题？**

HTTP/2 的队头阻塞在 TCP 层仍存在（一个包丢失，所有流都等待）。HTTP/3 基于 **QUIC**（UDP 实现），每个流独立，一个流的丢包不影响其他流。握手也更快（0-RTT 恢复）。
