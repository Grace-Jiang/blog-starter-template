---
title: DNS与CDN - 域名解析、内容分发与负载均衡
description: DNS解析过程、DNS记录类型、CDN加速原理、负载均衡算法、正向与反向代理
pubDate: '2025-02-25'
categories:
- Network
tags:
- DNS
- CDN
- 负载均衡
- 面试
---
# DNS / CDN / 负载均衡 面试题

## 第1部分：DNS 面试常见问题

### 面试高频问题
```
# Q1: DNS 是什么？解析过程是怎样的？
# 答：DNS 将域名解析为 IP 地址。解析过程见详解。

# Q2: DNS 用的是 TCP 还是 UDP？
# 答：
# - 普通查询：UDP（端口 53），因为数据量小、追求速度
# - 区域传送（主从同步）：TCP（数据量大，需要可靠传输）
# - 响应超过 512 字节时：先 UDP 查询，发现截断后改用 TCP

# Q3: DNS 劫持和 DNS 污染的区别？
# 答：
# - DNS 劫持：篡改 DNS 服务器的解析结果（如运营商劫持广告）
# - DNS 污染：在传输过程中伪造 DNS 应答包
# - 解决：HTTPS + DoH (DNS over HTTPS) / DoT (DNS over TLS)

# Q4: 什么是 CDN？它是如何加速的？
# 答：见 CDN 详解

# Q5: 负载均衡的常见算法有哪些？
# 答：见负载均衡详解
```

---

## 第2部分：DNS 解析过程

### 完整解析流程
```
用户输入 www.example.com
  ↓
① 浏览器 DNS 缓存
  ├─ 命中 → 直接返回 IP
  └─ 未命中 ↓
② 操作系统 DNS 缓存 (/etc/hosts)
  ├─ 命中 → 直接返回 IP
  └─ 未命中 ↓
③ 本地 DNS 服务器（递归解析器，通常是运营商/公司 DNS）
  ├─ 缓存命中 → 直接返回 IP
  └─ 未命中 → 开始迭代查询 ↓
④ 根域名服务器 (.)
  └─ 返回 .com 顶级域名服务器地址
⑤ 顶级域名服务器 (.com)
  └─ 返回 example.com 权威域名服务器地址
⑥ 权威域名服务器 (example.com)
  └─ 返回 www.example.com 的 IP 地址
  ↓
本地 DNS 服务器缓存结果并返回给客户端
```

### 递归查询 vs 迭代查询
```
# 递归查询：客户端 → 本地 DNS
# - 客户端只问一次，本地 DNS 负责全部解析
# - "你帮我问到底"

# 迭代查询：本地 DNS → 各级域名服务器
# - 本地 DNS 每次收到一个指引，自己去问下一级
# - "我不知道，但你可以去问某某"
```

### DNS 记录类型
```
# A 记录：     域名 → IPv4 地址
#              example.com → 93.184.216.34

# AAAA 记录：  域名 → IPv6 地址
#              example.com → 2606:2800:220:1:248:1893:25c8:1946

# CNAME 记录： 域名 → 另一个域名（别名）
#              www.example.com → example.com
#              cdn.example.com → example.cdn-provider.com

# MX 记录：    邮件服务器
#              example.com → mail.example.com (priority 10)

# NS 记录：    域名的权威 DNS 服务器
#              example.com → ns1.example.com

# TXT 记录：   文本信息（常用于 SPF、域名验证）
#              example.com → "v=spf1 include:_spf.google.com ~all"

# SRV 记录：   服务定位（指定服务的主机和端口）
#              _http._tcp.example.com → 10 60 80 www.example.com
```

### DNS TTL（生存时间）
```
# TTL 决定 DNS 记录在缓存中的存活时间

# TTL 长（如 86400 = 1天）：
# ✅ 减少 DNS 查询次数，解析更快
# ❌ DNS 变更生效慢

# TTL 短（如 60 = 1分钟）：
# ✅ DNS 变更快速生效
# ❌ DNS 查询频繁，增加延迟

# 最佳实践：
# - 正常运维：TTL = 300~3600
# - 迁移前：提前将 TTL 调短，迁移后调回
```

---

## 第3部分：CDN

### CDN 是什么？
```
# CDN = Content Delivery Network（内容分发网络）
# 核心思想：将内容缓存到离用户最近的节点，加速访问

# 没有 CDN：
# 用户(北京) ──→ 源站(美国) ：延迟高

# 使用 CDN：
# 用户(北京) ──→ CDN 边缘节点(北京) ──(缓存未命中)──→ 源站(美国)
#                    ↑ 缓存命中直接返回
```

### CDN 工作流程
```
用户请求 static.example.com/image.png
  ↓
① DNS 解析（CNAME 指向 CDN 域名）
  static.example.com → cdn.provider.com
  ↓
② CDN 智能 DNS 调度
  - 根据用户 IP 判断地理位置
  - 根据节点负载、网络状况选择最优节点
  ↓
③ 返回最近 CDN 边缘节点 IP
  ↓
④ 用户请求到达边缘节点
  ├─ 缓存命中 → 直接返回内容（最快）
  └─ 缓存未命中 ↓
⑤ 边缘节点 → 回源到源站
  ↓
⑥ 源站返回内容，边缘节点缓存并返回给用户
```

### CDN 适合加速的内容
```
# 适合（静态资源）：
# - 图片、视频、音频
# - CSS、JS 文件
# - 静态 HTML 页面
# - 软件下载包

# 不太适合（动态内容）：
# - API 响应（个性化数据）
# - 实时数据（股票、聊天）
# - 需要认证的内容
# 但可以通过动态加速（DCDN）优化路由
```

---

## 第4部分：负载均衡

### 负载均衡层次

```
# L4 负载均衡（传输层）：
# - 基于 IP + 端口转发
# - 不解析 HTTP 内容，性能高
# - 工具：LVS, Nginx (stream), HAProxy (TCP mode)

# L7 负载均衡（应用层）：
# - 基于 HTTP 内容（URL/Header/Cookie）转发
# - 可做更精细的路由
# - 工具：Nginx, HAProxy (HTTP mode), Envoy, Traefik
```

### 常见负载均衡算法

```
# 1. 轮询 (Round Robin)
# 请求按顺序轮流分配到各服务器
# → A → B → C → A → B → C
# 优点：简单  缺点：不考虑服务器差异

# 2. 加权轮询 (Weighted Round Robin)
# 按权重比例分配（性能好的服务器权重高）
# A(weight=3) B(weight=1) → A A A B A A A B
# 适合：服务器配置不同

# 3. 最少连接 (Least Connections)
# 将请求分配给当前连接数最少的服务器
# 适合：请求处理时间差异大的场景

# 4. 加权最少连接 (Weighted Least Connections)
# 结合权重和连接数

# 5. IP 哈希 (IP Hash)
# 根据客户端 IP 计算哈希值，固定分配到同一台服务器
# 优点：实现会话保持  缺点：负载可能不均

# 6. 一致性哈希 (Consistent Hashing)
# 节点增减时只影响相邻节点，最小化重新映射
# 适合：缓存场景（Redis 集群）

# 7. 随机 (Random)
# 随机选择服务器
# 大量请求下趋近于轮询效果
```

### Nginx 负载均衡配置示例
```nginx
# 轮询（默认）
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

# 加权轮询
upstream backend {
    server 192.168.1.10:8080 weight=3;
    server 192.168.1.11:8080 weight=2;
    server 192.168.1.12:8080 weight=1;
}

# IP 哈希
upstream backend {
    ip_hash;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

# 最少连接
upstream backend {
    least_conn;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

# 健康检查 + 故障转移
upstream backend {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 backup;  # 备用服务器
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 第5部分：正向代理 vs 反向代理

```
# 正向代理（Forward Proxy）：代理客户端
#
# 客户端 → [正向代理] → 目标服务器
#
# - 客户端知道代理的存在
# - 服务端不知道真实客户端
# - 场景：科学上网、公司出口代理、缓存加速
# - 工具：Squid, V2Ray

# 反向代理（Reverse Proxy）：代理服务端
#
# 客户端 → [反向代理] → 后端服务器集群
#
# - 客户端不知道代理的存在（以为在和真实服务器通信）
# - 服务端知道代理的存在
# - 场景：负载均衡、SSL 终止、缓存、安全防护
# - 工具：Nginx, HAProxy, Envoy, Traefik
```

| 特性 | 正向代理 | 反向代理 |
|------|---------|---------|
| **代理对象** | 客户端 | 服务端 |
| **客户端感知** | 知道代理存在 | 不知道代理存在 |
| **典型场景** | 翻墙、企业出口 | 负载均衡、CDN |
| **配置方** | 客户端配置 | 服务端配置 |
