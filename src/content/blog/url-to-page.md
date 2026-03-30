---
title: 从输入URL到页面显示 - 全链路详解
description: URL解析、DNS解析、TCP握手、TLS握手、HTTP请求、服务端处理、浏览器渲染全流程
pubDate: '2025-02-23'
categories:
- Network
tags:
- 浏览器
- DNS
- TCP
- TLS
- 面试
---
# 从输入 URL 到页面显示 —— 全链路详解

## 全链路总览

```
用户输入 URL
  ↓
① URL 解析
  ↓
② DNS 解析（域名 → IP）
  ↓
③ TCP 三次握手（建立连接）
  ↓
④ TLS 握手（如果是 HTTPS）
  ↓
⑤ 发送 HTTP 请求
  ↓
⑥ 服务端处理请求
  ↓
⑦ 返回 HTTP 响应
  ↓
⑧ 浏览器解析与渲染
  ↓
⑨ 连接管理（复用 / 关闭）
```

---

## ① URL 解析

浏览器拿到用户输入后，第一步是判断这是搜索词还是 URL，如果是 URL 就按规则拆解：

```
https://www.example.com:443/path/page?q=hello#section1
  │         │            │     │        │        │
协议     域名          端口   路径     查询参数   锚点
```

- 协议缺省时，现代浏览器默认尝试 HTTPS
- 端口缺省时，HTTP 用 80，HTTPS 用 443
- 浏览器会检查 **HSTS 列表**（HTTP Strict Transport Security）：如果该域名在列表中，即使用户输入 `http://` 也会在**发请求之前**被强制替换为 `https://`（307 Internal Redirect，不经过网络）

---

## ② DNS 解析

拿到域名后需要解析为 IP 地址，查找顺序是逐级缓存：

```
浏览器 DNS 缓存（Chrome: chrome://net-internals/#dns）
  ↓ 未命中
操作系统 DNS 缓存（含 /etc/hosts）
  ↓ 未命中
本地 DNS 服务器（递归解析器，通常是运营商/公司的）
  ↓ 缓存未命中，开始迭代查询
根域名服务器（.）→ 返回 .com 的 NS 地址
  ↓
顶级域名服务器（.com）→ 返回 example.com 的 NS 地址
  ↓
权威域名服务器（example.com）→ 返回 www.example.com 的 A 记录（IP）
  ↓
本地 DNS 缓存结果（TTL 控制时长），返回给浏览器
```

**关键细节：**
- 如果用了 CDN，权威 DNS 会返回一个 CNAME 指向 CDN 域名，CDN 的智能 DNS 再根据用户地理位置/网络状况返回最近边缘节点的 IP
- DNS 查询默认用 UDP（端口 53），响应超过 512 字节时改用 TCP
- 每一级都有 TTL 缓存，所以大多数请求不会走到根服务器

> 详见 [dns_cdn.md](dns_cdn.md)

---

## ③ TCP 三次握手

拿到 IP 后，浏览器与服务器建立 TCP 连接。注意：浏览器**不是**直接连到应用程序，而是连到目标 IP 的某个**端口**上正在 `listen()` 的进程：

### 连接的对端是谁？

```
浏览器 ──→ 目标 IP:443 ──→ 监听该端口的进程

# HTTP  默认连 80 端口
# HTTPS 默认连 443 端口
```

端口背后 `listen()` 的通常**不是**应用进程本身，而是最外层的入口：

```
浏览器
  │
  │  TCP 连接到 目标IP:443
  ↓
┌─────────────────────────────────────────┐
│ 最外层：监听 443 端口的进程              │
│                                         │
│  场景1：Nginx / HAProxy（反向代理）      │
│  场景2：云 LB（AWS ALB, 阿里云 SLB）     │
│  场景3：CDN 边缘节点                     │
│  场景4：小项目直接就是应用进程本身        │
└────────────────┬────────────────────────┘
                 │ 内部转发（新建/复用 TCP 连接）
                 ↓
          应用服务器（Gunicorn, Tomcat, Node...）
```

**浏览器的 TCP 连接只到第一层**，后面每一层之间都是独立的 TCP 连接。

### 三次握手过程

```
客户端                          服务端
  │  SYN, seq=x                  │
  │ ────────────────────────→    │  客户端 → SYN_SENT
  │                              │
  │  SYN+ACK, seq=y, ack=x+1    │
  │ ←────────────────────────    │  服务端 → SYN_RCVD
  │                              │
  │  ACK, seq=x+1, ack=y+1      │
  │ ────────────────────────→    │  双方 → ESTABLISHED
```

**耗时：1 个 RTT（第三次握手可以携带数据）**

如果是 HTTP/1.1，浏览器通常会并行建立 **6 个 TCP 连接**（同一域名的并发上限）来加速资源加载。HTTP/2 只需 1 个连接即可多路复用。

> 详见 [network.md - TCP 三次握手](network.md)

---

## ④ TLS 握手（HTTPS）

TCP 连接建立后，如果是 HTTPS 还需要 TLS 握手。

### 为什么同时用两种加密？

| | 对称加密 | 非对称加密 |
|--|---------|----------|
| **速度** | 快（AES 可达 GB/s） | 慢（比 AES 慢 100-1000 倍） |
| **密钥分发** | 难（怎么安全传给对方？） | 易（公钥可以公开） |

- 全程非对称加密 → 太慢
- 全程对称加密 → 密钥怎么安全送达？

TLS 的方案：**用非对称加密解决密钥分发，用对称加密解决性能问题**。

```
握手阶段（非对称加密）──→ 安全交换密钥（只用一次）
                           │
                           ↓
传输阶段（对称加密）  ──→ 加密所有通信数据（全程使用）
```

### TLS 1.2 握手过程（2-RTT）
```
客户端                                  服务端
  │ ClientHello (支持的加密套件, 随机数A)  │
  │ ──────────────────────────────────→ │
  │                                     │
  │ ServerHello (选定套件, 随机数B)       │
  │ Certificate (证书)                   │
  │ ServerKeyExchange (DH 参数)          │
  │ ServerHelloDone                     │
  │ ←────────────────────────────────── │
  │                                     │
  │ 验证证书 (CA 信任链)                  │
  │ ClientKeyExchange (预主密钥)          │
  │ ChangeCipherSpec + Finished          │
  │ ──────────────────────────────────→ │
  │                                     │
  │ ChangeCipherSpec + Finished          │
  │ ←────────────────────────────────── │
  │                                     │
  │ ═══ 对称加密通信开始 ═══              │
```

**TLS 1.3 优化到 1-RTT，重连可以 0-RTT。**

### 密钥协商本质
```
客户端                              服务端
  │                                  │
  │  拿到服务端的公钥（来自证书）       │
  │                                  │
  │  用公钥加密"预主密钥"发过去        │
  │ ────────────────────────────→    │
  │                                  │  用私钥解密，拿到预主密钥
  │                                  │
  │  双方各自用相同的材料推导出会话密钥  │
  │  会话密钥 = f(预主密钥, 随机数A, 随机数B)
  │                                  │
  │ ═══ 之后全部用会话密钥对称加密 ═══  │
```

### 证书验证过程
1. 检查证书是否由受信 CA 签发（浏览器内置根证书）
2. 检查域名是否匹配
3. 检查证书有效期
4. 检查是否被吊销（CRL / OCSP）

> 详见 [security.md - TLS 详解](security.md)

---

## ⑤ 发送 HTTP 请求

连接建立完成，浏览器构造并发送 HTTP 请求：

```http
GET /path/page?q=hello HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 ...
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cookie: session_id=abc123
Connection: keep-alive
If-None-Match: "etag-value"          ← 协商缓存
If-Modified-Since: Wed, 01 Jan 2025  ← 协商缓存
```

### 浏览器发请求前会先检查本地缓存
```
检查 Cache-Control / Expires（强缓存）
  ├─ 未过期 → 直接用缓存，状态 200 (from disk/memory cache)，不发请求
  └─ 已过期 → 带 ETag/Last-Modified 发请求（协商缓存）
       ├─ 服务端返回 304 → 用本地缓存
       └─ 服务端返回 200 → 用新内容
```

> 详见 [http.md - HTTP 缓存](http.md)

---

## ⑥ 服务端处理

请求到达服务器后，从网卡到你的业务代码，经历了以下链路：

### 6.1 网卡 → 内核协议栈（拆包）

数据包到达网卡后，内核按协议栈**自底向上逐层剥离**：

```
 以太网帧头 │ IP 头 │ TCP 头 │ HTTP 数据 │
     ↓ 剥离    ↓ 剥离   ↓ 剥离    ↓ 最终交给应用
  数据链路层  网络层   传输层    应用层
```

```
网卡收到以太网帧
  ↓
数据链路层：剥离帧头（MAC 地址），提取 IP 包
  ↓
网络层：剥离 IP 头（检查目标 IP 是不是自己），提取 TCP 段
  ↓
传输层：根据目标端口号找到对应的 Socket → 写入内核接收缓冲区 (recv buffer)
```

### 6.2 端口 → 进程（内核怎么找到进程）

内核维护了一张 Socket 表，通过**五元组**匹配：

```
(协议, 源IP, 源端口, 目的IP, 目的端口) → Socket fd → 进程

# 例：
# (TCP, 1.2.3.4, 54321, 10.0.0.1, 443) → fd 7 → Nginx (PID 1234)

# 查看当前映射：
ss -tlnp
# LISTEN  0  128  0.0.0.0:443  *:*  users:(("nginx", pid=1234, fd=7))
```

Nginx 调用 `accept()` 从 listen socket 取走已完成三次握手的连接，得到一个 connected socket fd：

```
listen socket (0.0.0.0:443)
  │
  │  accept() —— 取出已完成三次握手的连接
  ↓
connected socket (10.0.0.1:443 ←→ 1.2.3.4:54321)
  │
  │  read() / write()
  ↓
处理请求
```

### 6.3 Nginx（反向代理）

Nginx worker 通过 epoll 监听大量连接：

```
Nginx worker (epoll 事件循环)
  │
  │  epoll_wait() 返回就绪的 fd
  │  read() 读取 HTTP 请求
  ↓
解析 HTTP 请求
  │
  ├─ 静态文件？→ 直接读取磁盘返回（sendfile 零拷贝）
  │
  └─ 动态请求？→ 根据 location 规则转发
      │
      │  proxy_pass http://127.0.0.1:8000;
      │  （Nginx 与后端之间建立新的 TCP 连接）
      ↓
  应用服务器
```

### 6.4 应用服务器（Gunicorn / uWSGI / Node / Tomcat）

应用服务器监听内部端口，负责**管理 worker 进程/线程**并调用应用代码：

```
Gunicorn (master, 监听 127.0.0.1:8000)
  │
  ├─ worker 1 (进程)
  ├─ worker 2 (进程)
  └─ worker 3 (进程)
       │
       │  收到 Nginx 转发来的请求
       │  解析为 WSGI 标准格式
       │  调用 Flask/Django 入口
       ↓
     app(environ, start_response)   ← WSGI 接口
```

| 语言 | 应用服务器 | 接口协议 |
|------|----------|---------|
| Python | Gunicorn, uWSGI | WSGI / ASGI |
| Java | Tomcat, Jetty | Servlet API |
| Node.js | 内置 http 模块 | 直接处理 |
| Go | 内置 net/http | 直接处理 |

### 6.5 应用框架 → 业务代码

框架做**路由匹配**，找到对应的处理函数：

```python
# Flask 示例
@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = db.query(User).get(user_id)   # 查数据库
    return jsonify(user.to_dict())         # 返回 JSON
```

完整调用链：

```
Gunicorn 调用 app(environ, start_response)
  ↓
Flask 应用入口
  ↓
中间件链（日志、认证、CORS、限流...）
  ↓
路由匹配 URL → 视图函数
  ↓
你的业务代码（查 DB / 读缓存 / 调微服务）
  ↓
构造 Response 对象
  ↓
中间件链（反向，后处理）
  ↓
返回 HTTP 响应 → Gunicorn → 内核 → Nginx → 内核 → 网卡 → 浏览器
```

### 6.6 一图总结：请求在服务端的完整数据流

```
[网卡] 收到以太网帧
  ↓
[内核-数据链路层] 剥离帧头 → IP 包
  ↓
[内核-网络层] 剥离 IP 头 → TCP 段
  ↓
[内核-传输层] 根据端口号找到 Socket → 写入 recv buffer
  ↓
[Nginx] epoll 检测到 fd 可读 → read() → 解析 HTTP → proxy_pass
  ↓
[内核] 新 TCP 连接 → 写入 Gunicorn 的 recv buffer
  ↓
[Gunicorn] worker read() → 解析为 WSGI environ
  ↓
[Flask] 中间件 → 路由匹配 → 视图函数
  ↓
[你的代码] 查 DB → 构造响应 → 原路返回
```

> 详见 [dns_cdn.md - 负载均衡](dns_cdn.md)，[socket.md - IO 模型与 epoll](socket.md)

---

## ⑦ 返回 HTTP 响应

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 12345
Cache-Control: max-age=3600
ETag: "abc123"
Set-Cookie: session_id=xyz; HttpOnly; Secure
Transfer-Encoding: chunked  ← 流式传输时使用

<!DOCTYPE html>
<html>...响应体...</html>
```

如果返回 **301/302**，浏览器会拿到 `Location` 头中的新 URL，从第 ① 步重新开始。

> 详见 [http.md - HTTP 状态码](http.md)

---

## ⑧ 浏览器解析与渲染


---

## ⑨ 连接管理

```
HTTP/1.0：每次请求后关闭连接（四次挥手）
HTTP/1.1：默认 Keep-Alive，复用连接，空闲超时后关闭
HTTP/2：  单连接多路复用，连接生命周期更长
HTTP/3：  基于 QUIC/UDP，切换网络也不断连（Connection Migration）
```

四次挥手关闭时，主动关闭方会进入 **TIME_WAIT**（等待 2MSL），确保最后的 ACK 能到达对方。

> 详见 [network.md - TCP 四次挥手](network.md)，[http.md - HTTP 版本演进](http.md)

---

## 全流程耗时估算

| 步骤 | 典型耗时 |
|------|---------|
| URL 解析 | < 1ms |
| DNS 解析（缓存命中） | < 1ms |
| DNS 解析（完整查询） | 20-120ms |
| TCP 握手 | 1 RTT ≈ 10-100ms |
| TLS 1.2 握手 | 2 RTT ≈ 20-200ms |
| TLS 1.3 握手 | 1 RTT ≈ 10-100ms |
| HTTP 请求+响应 | 取决于网络和服务端处理 |
| 浏览器渲染 | 取决于页面复杂度 |

---

## ⑩ K8s/OCP 场景：通过 Ingress VIP 访问服务的完整流量路径

上面 ①-⑨ 描述的是通用场景。如果服务部署在 OpenShift/K8s 集群上，服务端处理链路有所不同。

### 完整流量路径

```
用户浏览器
  │
  │ ① DNS 解析
  │    day1.apps.ocp-cluster → Ingress VIP (如 192.168.1.100)
  ↓
② 交换机查 ARP 表，转发到持有 VIP 的 Node
  │    VIP 由 Keepalived 绑定在某一个 Node 上（同一时刻只有一个）
  │    Node 挂了 → VIP 秒级漂移到其他 Node（Gratuitous ARP 通告交换机）
  ↓
③ 数据包到达 Node 网卡 (eth0)
  │    Router Pod 使用 hostNetwork: true
  │    HAProxy 进程直接 listen 在宿主机 443 端口
  │    内核收到包 → 直接交给 HAProxy（不经过 NodePort / iptables DNAT）
  ↓
④ HAProxy (Router Pod) —— L7 层处理
  │    - TLS 终止（edge termination，解密 HTTPS）
  │    - 解析 HTTP 请求（状态机逐字节扫描，和 Nginx 原理一致）
  │    - 匹配 Route 规则：Host + Path
  │    - 从 Endpoints 拿到后端 Pod IP 列表（watch API Server 自动维护）
  │    - 负载均衡选一个 Pod，转发请求
  ↓
⑤ 数据包发往 Pod IP（如 172.28.4.18:5000）
  │    目标地址是 Pod IP，不是 Service ClusterIP
  │    数据包仍经过内核 netfilter 链（必经之路）
  │    但 kube-proxy/OVN 的 Service DNAT 规则不命中（目标不是 ClusterIP）
  │    不做 DNAT，直接放行
  ↓
⑥ OVN-Kubernetes（CNI）路由到目标 Pod
  │    ├─ Pod 在本机 → 通过 OVS bridge 直接到 Pod 网络命名空间
  │    └─ Pod 在其他 Node → Geneve 隧道封装 → 转发到目标 Node → 拆封 → 到 Pod
  ↓
⑦ Pod 里的容器进程（如 Gunicorn listen 0.0.0.0:5000）
  │    内核协议栈拆包 → Socket 匹配 → 应用代码处理请求
  ↓
⑧ 响应原路返回
     Pod → OVN → Node → HAProxy → Node 网卡 → 交换机 → 用户
```

### VIP 是怎么绑在一个 Node 上的？

```
NodeA (Master)                NodeB (Backup)               NodeC (Backup)
┌──────────────┐             ┌──────────────┐             ┌──────────────┐
│ Keepalived   │             │ Keepalived   │             │ Keepalived   │
│ 优先级: 100  │  ← VRRP →  │ 优先级: 90   │  ← VRRP →  │ 优先级: 80   │
│              │   心跳组播   │              │   心跳组播   │              │
│ VIP: ✅      │             │ VIP: ❌      │             │ VIP: ❌      │
│ 192.168.1.100│             │              │             │              │
│ 绑在 eth0 上 │             │              │             │              │
│              │             │              │             │              │
│ Router Pod   │             │ Router Pod   │             │ Router Pod   │
│ (HAProxy)    │             │ (待命)        │             │ (待命)       │
└──────────────┘             └──────────────┘             └──────────────┘

故障切换：
  NodeA 挂了 → NodeB 检测到心跳消失（1-3s）
  → NodeB 绑定 VIP + 发送 Gratuitous ARP → 交换机更新 MAC 表
  → 后续流量转发到 NodeB
```

### Service 在这条链路中的角色

```
Service 做了什么：   提供 Pod IP 列表（Endpoints），通过 selector 关联健康 Pod
Service 没做什么：   不转发流量，ClusterIP 没有被访问

HAProxy 不会访问 ClusterIP (10.96.0.100)
而是直接用从 Endpoints 拿到的 Pod IP：

  # HAProxy 自动生成的配置：
  backend be_http:dell-acp:mcp-operator-installer-ocp
    server pod:...:172.28.4.18:5000 172.28.4.18:5000 weight 1
    ↑                                ↑
    Pod 名字                          直连 Pod IP，没经过 ClusterIP
```

### HAProxy vs Nginx

HAProxy 和 Nginx 是两个不同的软件，但在这条链路里干的活一样：

| | HAProxy | Nginx |
|--|---------|-------|
| **定位** | 专业负载均衡器/代理 | Web 服务器 + 反向代理 + 负载均衡 |
| **能力** | 只做代理和负载均衡 | 还能托管静态文件、做 Web 服务器 |
| **用在哪** | OpenShift Router Pod | K8s Nginx Ingress Controller |
| **本质** | 都是 L7 反向代理：TLS 终止 + Host/Path 匹配 + 转发到后端 Pod |

### 各组件角色总结

```
组件                      角色                           是否转发流量
──────────────────────────────────────────────────────────────────────
Keepalived               VIP 绑定 + 故障漂移              否
HAProxy (Router Pod)     L7 路由 + TLS 终止 + 负载均衡    是（核心转发者）
Service                  提供 Pod IP 列表 (Endpoints)     否（只是数据源）
OVN-Kubernetes (CNI)     Pod 间网络路由                   是（网络层转发）
iptables / OVN LB        Service DNAT 规则               经过但不命中
```

> 详见 [../k8s/k8s.md - 流量链路](../k8s/k8s.md)

---

## 面试回答模板（精简版）

```
1. URL 解析 → 拆分协议/域名/端口/路径，HSTS 检查
2. DNS 解析 → 浏览器缓存 → OS 缓存 → 本地 DNS → 迭代查询（根→顶级→权威）
3. TCP 三次握手 → SYN → SYN+ACK → ACK，1 RTT
4. TLS 握手 → 证书验证 + 密钥协商，TLS 1.2 需 2 RTT，1.3 需 1 RTT
5. 发送 HTTP 请求 → 先检查本地缓存（强缓存 → 协商缓存）
6. 服务端处理 → LB → 反向代理 → 应用 → DB/缓存
7. 返回响应 → 状态码 + 响应头 + 响应体，301/302 会重定向
8. 浏览器渲染 → DOM → CSSOM → 渲染树 → 布局 → 绘制 → 合成
9. 连接复用/关闭 → Keep-Alive / 多路复用 / 四次挥手
```
