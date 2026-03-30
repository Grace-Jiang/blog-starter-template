---
title: HTTP协议详解 - 从HTTP/1.0到HTTP/3
description: HTTP与HTTPS、状态码、GET与POST、HTTP版本演进、Cookie/Session/Token、RESTful API
pubDate: '2025-02-21'
categories:
- Network
tags:
- HTTP
- HTTPS
- RESTful
- 面试
---
# HTTP 面试题

## 第1部分：面试常见问题与答案

### 面试高频问题
```
# Q1: HTTP 和 HTTPS 的区别？
# 答：
# - HTTP 明文传输，HTTPS = HTTP + TLS/SSL 加密
# - HTTP 默认端口 80，HTTPS 默认端口 443
# - HTTPS 需要 CA 证书
# - HTTPS 在 TCP 握手后还要进行 TLS 握手

# Q2: HTTP 常用状态码？
# 答：见详解表格

# Q3: GET 和 POST 的区别？
# 答：
# - GET 参数在 URL 中，POST 在请求体中
# - GET 有长度限制（浏览器限制），POST 无限制
# - GET 可缓存/可收藏，POST 不可
# - GET 是幂等的，POST 不是
# - GET 语义是获取资源，POST 语义是提交数据

# Q4: HTTP/1.0 vs HTTP/1.1 vs HTTP/2 vs HTTP/3？
# 答：见版本对比详解

# Q5: Cookie、Session、Token 的区别？
# 答：见详解

# Q6: 什么是 RESTful API？
# 答：
# - 使用 HTTP 方法表示操作：GET(查), POST(增), PUT(改), DELETE(删)
# - URL 表示资源：/api/users/123
# - 无状态：每个请求包含所有必要信息
# - 返回标准 HTTP 状态码
```

---

## 第2部分：HTTP 状态码

### 分类

| 分类 | 含义 | 常见状态码 |
|------|------|-----------|
| **1xx** | 信息性 | 100 Continue, 101 Switching Protocols |
| **2xx** | 成功 | 200, 201, 204 |
| **3xx** | 重定向 | 301, 302, 304 |
| **4xx** | 客户端错误 | 400, 401, 403, 404, 405 |
| **5xx** | 服务端错误 | 500, 502, 503, 504 |

### 高频状态码详解

```
# 2xx 成功
200 OK                    # 请求成功
201 Created               # 资源创建成功（POST 返回）
204 No Content            # 成功，但无返回内容（DELETE 返回）

# 3xx 重定向
301 Moved Permanently     # 永久重定向（搜索引擎更新 URL）
302 Found                 # 临时重定向（搜索引擎不更新 URL）
304 Not Modified          # 资源未修改，使用缓存

# 4xx 客户端错误
400 Bad Request           # 请求参数有误
401 Unauthorized          # 未认证（需要登录）
403 Forbidden             # 已认证但无权限
404 Not Found             # 资源不存在
405 Method Not Allowed    # HTTP 方法不被允许                           
429 Too Many Requests     # 请求过于频繁（限流）

# 5xx 服务端错误
500 Internal Server Error # 服务器内部错误
502 Bad Gateway           # 网关收到上游无效响应
503 Service Unavailable   # 服务暂时不可用（过载/维护）
504 Gateway Timeout       # 网关等待上游超时
```

### 面试常问：301 vs 302
```
# 301 永久重定向：
# - 浏览器会缓存重定向
# - 搜索引擎将新 URL 作为标准 URL
# - 场景：域名更换、HTTP 跳转 HTTPS
               
# 302 临时重定向：
# - 浏览器不缓存
# - 搜索引擎保留原 URL
# - 场景：临时跳转、登录后重定向

# 面试追问：如何实现 HTTP → HTTPS 跳转？
# Nginx 配置：
# server {
#     listen 80;
#     return 301 https://$host$request_uri;
# }
```

---

## 第3部分：HTTP 方法

### 常用方法对比

| 方法 | 语义 | 幂等 | 安全 | 请求体 | 典型场景 |
|------|------|------|------|--------|---------|
| **GET** | 获取资源 | 是 | 是 | 无 | 查询数据 |
| **POST** | 创建资源 | 否 | 否 | 有 | 提交表单、上传文件 |
| **PUT** | 全量更新 | 是 | 否 | 有 | 更新整个资源 |
| **PATCH** | 部分更新 | 否 | 否 | 有 | 更新部分字段 |
| **DELETE** | 删除资源 | 是 | 否 | 无/有 | 删除资源 |
| **HEAD** | 获取头部 | 是 | 是 | 无 | 检查资源是否存在 |
| **OPTIONS** | 预检请求 | 是 | 是 | 无 | CORS 跨域预检 |

### 幂等性解释
```
# 幂等：执行一次和执行多次效果相同

# GET /users/1    → 无论调几次，都是读取同一个用户 → 幂等
# DELETE /users/1 → 第一次删除成功，后续删除返回404，最终效果一致 → 幂等
# PUT /users/1    → 每次都用完整数据覆盖，结果一致 → 幂等
# POST /users     → 每次都会创建一个新用户 → 不幂等
```

---

## 第4部分：HTTP 版本演进

### HTTP/1.0 → HTTP/1.1

```
# HTTP/1.0 的问题：
# - 每个请求都要建立新的 TCP 连接（开销大）
# - 没有 Host 头（不支持虚拟主机）

# HTTP/1.1 改进：
# ✅ 持久连接 (Keep-Alive)：默认复用 TCP 连接
# ✅ 管道化 (Pipelining)：可以连续发送多个请求（但响应必须按序返回）
# ✅ Host 头：一个 IP 可以托管多个域名
# ✅ 分块传输 (Chunked Transfer Encoding)：流式传输
# ✅ 缓存控制增强：Cache-Control, ETag, If-None-Match
```

### HTTP/1.1 → HTTP/2

```
# HTTP/1.1 的瓶颈：
# - 队头阻塞 (Head-of-Line Blocking)：一个慢请求阻塞后续所有请求
# - 文本协议，解析效率低
# - 头部冗余：每次请求都发送大量重复头部

# HTTP/2 改进：
# ✅ 二进制分帧：将数据切为更小的帧，解析更高效
# ✅ 多路复用 (Multiplexing)：一个 TCP 连接上并行多个请求/响应
# ✅ 头部压缩 (HPACK)：减少冗余头部传输
# ✅ 服务器推送 (Server Push)：主动推送客户端需要的资源
# ✅ 流优先级：可以对请求设置优先级

# 多路复用 vs 管道化：
# 管道化：请求可以并发发送，但响应必须按序返回（仍有队头阻塞）
# 多路复用：请求和响应都可以乱序，通过 Stream ID 关联
```

### HTTP/2 → HTTP/3

```
# HTTP/2 的问题：
# - 基于 TCP，TCP 层的丢包会阻塞所有 Stream（TCP 队头阻塞）
# - TCP 握手 + TLS 握手 = 较长的连接建立时间

# HTTP/3 改进：
# ✅ 基于 QUIC 协议（运行在 UDP 之上）
# ✅ 解决 TCP 队头阻塞：每个 Stream 独立，一个丢包不影响其他
# ✅ 0-RTT 连接建立：首次 1-RTT，重连 0-RTT
# ✅ 连接迁移：切换网络（WiFi→4G）不断连，通过 Connection ID 识别
# ✅ 内建 TLS 1.3：安全性默认集成
```

### 版本对比总结

| 特性 | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|----------|--------|--------|
| **连接** | 短连接 | 持久连接 | 多路复用 | QUIC多路复用 |
| **协议格式** | 文本 | 文本 | 二进制帧 | 二进制帧 |
| **队头阻塞** | 有 | 有 | HTTP层解决 | 完全解决 |
| **头部压缩** | 无 | 无 | HPACK | QPACK |
| **传输层** | TCP | TCP | TCP | UDP (QUIC) |
| **TLS** | 可选 | 可选 | 事实上必须 | 内建 TLS 1.3 |

---

## 第5部分：Cookie / Session / Token

### Cookie
```
# 存储位置：客户端（浏览器）
# 大小限制：4KB
# 生命周期：由 Expires 或 Max-Age 控制

# 设置 Cookie（服务端响应头）：
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=3600

# 关键属性：
# - HttpOnly：JS 无法通过 document.cookie 读取（防 XSS）
# - Secure：只在 HTTPS 下发送
# - SameSite：防止 CSRF
#   - Strict：完全禁止跨站携带
#   - Lax：导航到目标网站的 GET 请求可携带（默认）
#   - None：允许跨站，但必须 Secure
```

### Session
```
# 存储位置：服务端（内存/Redis/数据库）
# 工作流程：
# 1. 用户登录 → 服务端创建 Session → 返回 Session ID（通过 Cookie）
# 2. 后续请求携带 Session ID → 服务端查找 Session → 认证成功

# 优点：数据存在服务端，安全
# 缺点：
# - 占用服务端资源
# - 分布式场景需要 Session 共享（Redis/Sticky Session）
# - 扩展性差
```

### Token (JWT)
```
# 存储位置：客户端（LocalStorage / Cookie）
# JWT 组成：Header.Payload.Signature

# Header:   {"alg": "HS256", "typ": "JWT"}
# Payload:  {"sub": "1234", "name": "user", "exp": 1700000000}
# Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)

# 工作流程：
# 1. 用户登录 → 服务端签发 JWT → 返回给客户端
# 2. 后续请求携带 JWT（Authorization: Bearer <token>）
# 3. 服务端验证签名 → 认证成功

# 优点：
# - 无状态，服务端无需存储
# - 天然支持分布式
# - 可携带用户信息

# 缺点：
# - 无法主动使 Token 失效（可用黑名单解决）
# - Payload 是 Base64 编码而非加密，不要存敏感信息
# - Token 体积较大
```

### 三者对比

| 特性 | Cookie | Session | JWT Token |
|------|--------|---------|-----------|
| **存储位置** | 客户端 | 服务端 | 客户端 |
| **安全性** | 较低 | 较高 | 中等 |
| **扩展性** | - | 差（需共享） | 好（无状态） |
| **跨域** | 受限 | 受限 | 灵活 |
| **服务端开销** | 无 | 有 | 无 |

---

## 第6部分：HTTP 缓存

### 缓存策略

```
# 强缓存（不发请求，直接用缓存）：
# Cache-Control: max-age=3600    # 缓存 1 小时
# Cache-Control: no-cache        # 每次需要验证（不是不缓存！）
# Cache-Control: no-store        # 完全不缓存
# Expires: Thu, 01 Jan 2025 00:00:00 GMT   # 过期时间（HTTP/1.0）

# 协商缓存（发请求，服务端判断是否用缓存）：
# 方案1：Last-Modified / If-Modified-Since
#   响应：Last-Modified: Wed, 01 Jan 2025 00:00:00 GMT
#   请求：If-Modified-Since: Wed, 01 Jan 2025 00:00:00 GMT
#   → 未修改返回 304，已修改返回 200 + 新内容

# 方案2：ETag / If-None-Match（优先级更高）
#   响应：ETag: "abc123"
#   请求：If-None-Match: "abc123"
#   → 匹配返回 304，不匹配返回 200 + 新内容
```

### 缓存流程
```
浏览器请求资源
  ↓
检查强缓存（Cache-Control / Expires）
  ├─ 命中 → 直接使用缓存（200 from cache）
  └─ 未命中 ↓
发送请求到服务端（携带 If-None-Match / If-Modified-Since）
  ↓
服务端判断资源是否变化
  ├─ 未变化 → 返回 304 Not Modified（使用缓存）
  └─ 已变化 → 返回 200 + 新资源 + 新缓存标识
```

---

## 第7部分：跨域与 CORS

### 同源策略
```
# 同源 = 协议 + 域名 + 端口 完全相同
# https://example.com:443/path

# 同源判断示例：
# https://a.com vs http://a.com        → 不同源（协议不同）
# https://a.com vs https://b.com       → 不同源（域名不同）
# https://a.com vs https://a.com:8080  → 不同源（端口不同）
# https://a.com/p1 vs https://a.com/p2 → 同源
```

### CORS（跨域资源共享）
```
# 简单请求（GET/HEAD/POST + 简单头部）：
# 浏览器直接发请求，服务端返回 CORS 头
# Access-Control-Allow-Origin: https://example.com
# Access-Control-Allow-Credentials: true

# 预检请求（PUT/DELETE/自定义头部等）：
# 1. 浏览器先发 OPTIONS 预检请求
#    Origin: https://example.com
#    Access-Control-Request-Method: PUT
#    Access-Control-Request-Headers: X-Custom-Header
#
# 2. 服务端回复允许的方法和头部
#    Access-Control-Allow-Origin: https://example.com
#    Access-Control-Allow-Methods: GET, POST, PUT, DELETE
#    Access-Control-Allow-Headers: X-Custom-Header
#    Access-Control-Max-Age: 86400   # 预检结果缓存时间
#
# 3. 预检通过后，浏览器发送实际请求
```

### 跨域解决方案
```
# 1. CORS（标准方案）：服务端设置 Access-Control-Allow-Origin
# 2. 代理（开发常用）：Nginx/Node 代理转发请求
# 3. JSONP（仅 GET，已淘汰）：利用 <script> 标签不受同源策略限制
```
