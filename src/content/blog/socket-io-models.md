---
title: Socket编程与IO模型 - select/poll/epoll详解
description: Socket基础、TCP Socket编程流程、5种IO模型、select/poll/epoll对比、Reactor模式
pubDate: '2025-02-22'
categories:
- Network
tags:
- Socket
- IO模型
- epoll
- 面试
---
# Socket 与 IO 模型面试题

## 第1部分：面试常见问题与答案

### 面试高频问题
```
# Q1: 什么是 Socket？
# 答：
# Socket 是应用层与传输层之间的抽象接口
# 本质是一个文件描述符（fd），代表一个网络连接的端点
# 一个连接由五元组唯一标识：(协议, 源IP, 源端口, 目的IP, 目的端口)

# Q2: TCP Socket 编程的基本流程？
# 答：见详解

# Q3: 五种 IO 模型是什么？
# 答：阻塞IO、非阻塞IO、IO多路复用、信号驱动IO、异步IO

# Q4: select/poll/epoll 的区别？
# 答：见详解对比表

# Q5: 什么是 Reactor 模式？
# 答：见详解

# Q6: 什么是 IO 多路复用？为什么需要它？
# 答：
# 一个线程同时监听多个 Socket 的 IO 事件
# 原因：如果每个连接一个线程，万级连接时线程开销太大
# 一个线程 + IO 多路复用可以处理大量并发连接
```

---

## 第2部分：TCP Socket 编程

### 服务端/客户端流程
```
服务端                            客户端
  │                                │
socket()  创建 Socket              socket()  创建 Socket
  │                                │
bind()    绑定 IP:Port             │
  │                                │
listen()  开始监听                  │
  │                                │
accept()  等待连接（阻塞）    ←──── connect()  发起连接
  │                                │
  │     三次握手完成                 │
  │                                │
read()    读取数据          ←──── write()   发送数据
  │                                │
write()   发送数据          ────→ read()    读取数据
  │                                │
close()   关闭连接                 close()   关闭连接
```

### Python Socket 示例
```python
# ===== 服务端 =====
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # TCP
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 8080))
server.listen(128)  # backlog = 128

while True:
    conn, addr = server.accept()  # 阻塞等待连接
    print(f"Connected by {addr}")
    data = conn.recv(1024)         # 阻塞读取
    conn.sendall(data.upper())     # 发送响应
    conn.close()

# ===== 客户端 =====
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('127.0.0.1', 8080))
client.sendall(b'hello')
data = client.recv(1024)
print(f"Received: {data}")
client.close()
```

### listen() 的 backlog 参数
```
# listen(backlog) 中的 backlog 含义：

# TCP 连接建立时有两个队列：
# 1. SYN 队列（半连接队列）：收到 SYN，等待三次握手完成
# 2. Accept 队列（全连接队列）：三次握手完成，等待 accept() 取走

# backlog 参数控制 Accept 队列的大小
# 队列满时，新连接会被拒绝（发送 RST）

# 查看系统限制：
# cat /proc/sys/net/core/somaxconn    # 系统最大 backlog
# cat /proc/sys/net/ipv4/tcp_max_syn_backlog  # SYN 队列大小
```

---

## 第3部分：五种 IO 模型

### 概述
```
# IO 操作分两个阶段：
# 1. 等待数据准备好（数据从网络到达内核缓冲区）
# 2. 将数据从内核拷贝到用户空间

# 五种模型在这两个阶段的行为不同：
```

### 1. 阻塞 IO (Blocking IO)
```
应用进程          内核
  │                │
  │── recvfrom() ─→│
  │   (阻塞等待)    │ 等待数据...
  │                │ 数据到达
  │                │ 拷贝到用户空间
  │←── 返回数据 ───│
  │                │

# 特点：两个阶段都阻塞
# 优点：编程简单
# 缺点：一个线程只能处理一个连接
```

### 2. 非阻塞 IO (Non-blocking IO)
```
应用进程          内核
  │                │
  │── recvfrom() ─→│
  │←── EAGAIN ────│   数据未就绪，立即返回
  │                │
  │── recvfrom() ─→│
  │←── EAGAIN ────│   轮询...
  │                │
  │── recvfrom() ─→│
  │                │   数据到达
  │   (阻塞)       │   拷贝到用户空间
  │←── 返回数据 ───│

# 特点：第一阶段非阻塞（轮询），第二阶段阻塞
# 优点：不会被阻塞住
# 缺点：忙轮询浪费 CPU
```

### 3. IO 多路复用 (IO Multiplexing)
```
应用进程          内核
  │                │
  │── select() ──→│
  │   (阻塞等待    │  监听多个 fd
  │    任一fd就绪)  │
  │←── 返回就绪fd ─│  某个 fd 数据到达
  │                │
  │── recvfrom() ─→│
  │   (阻塞)       │  拷贝到用户空间
  │←── 返回数据 ───│

# 特点：用一个系统调用监听多个 fd
# 优点：一个线程处理多个连接
# 缺点：两次系统调用（select + read）
# 实现：select, poll, epoll, kqueue
```

### 4. 信号驱动 IO (Signal-driven IO)
```
应用进程          内核
  │                │
  │── sigaction() ─→│  注册 SIGIO 信号处理函数
  │←── 返回 ───────│  立即返回，不阻塞
  │                │
  │  (做其他事情)    │  等待数据...
  │                │
  │←── SIGIO ─────│  数据到达，发信号通知
  │                │
  │── recvfrom() ─→│
  │   (阻塞)       │  拷贝到用户空间
  │←── 返回数据 ───│

# 特点：第一阶段通过信号通知，第二阶段阻塞
# 实际使用较少
```

### 5. 异步 IO (Asynchronous IO)
```
应用进程          内核
  │                │
  │── aio_read() ─→│
  │←── 返回 ───────│  立即返回，不阻塞
  │                │
  │  (做其他事情)    │  等待数据...
  │                │  数据到达
  │                │  拷贝到用户空间
  │                │
  │←── 信号/回调 ──│  全部完成后通知

# 特点：两个阶段都不阻塞
# 优点：真正的异步
# 缺点：实现复杂，Linux 原生支持不完善
# 实现：Linux aio, Windows IOCP, io_uring (Linux 5.1+)
```

### IO 模型对比

| 模型 | 等待数据 | 拷贝数据 | 复杂度 | 性能 |
|------|---------|---------|--------|------|
| **阻塞 IO** | 阻塞 | 阻塞 | 低 | 低 |
| **非阻塞 IO** | 非阻塞(轮询) | 阻塞 | 中 | 低(浪费CPU) |
| **IO 多路复用** | 阻塞(在select上) | 阻塞 | 中 | 高 |
| **信号驱动** | 非阻塞(信号通知) | 阻塞 | 高 | 高 |
| **异步 IO** | 非阻塞 | 非阻塞 | 高 | 最高 |

---

## 第4部分：select / poll / epoll 对比

### 核心区别

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| **数据结构** | bitmap (fd_set) | 数组 (pollfd) | 红黑树 + 就绪链表 |
| **最大连接数** | 1024 (FD_SETSIZE) | 无限制 | 无限制 |
| **fd 传递** | 每次调用都要拷贝全部fd | 每次调用都要拷贝全部fd | fd 只需注册一次 |
| **检测方式** | 线性遍历所有 fd | 线性遍历所有 fd | 回调机制，只返回就绪fd |
| **时间复杂度** | O(n) | O(n) | O(1) 就绪事件获取 |
| **触发模式** | 水平触发 (LT) | 水平触发 (LT) | LT + 边缘触发 (ET) |

### select
```c
// 伪代码
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(sockfd, &readfds);

// 阻塞等待，返回就绪 fd 数量
int n = select(maxfd + 1, &readfds, NULL, NULL, &timeout);

// 需要遍历所有 fd 检查哪个就绪
for (int i = 0; i <= maxfd; i++) {
    if (FD_ISSET(i, &readfds)) {
        // 处理 fd i
    }
}

// 缺点：
// 1. fd 数量限制 1024
// 2. 每次调用需要拷贝全部 fd_set 到内核
// 3. 返回后需要遍历所有 fd
```

### poll
```c
// 伪代码
struct pollfd fds[MAX];
fds[0].fd = sockfd;
fds[0].events = POLLIN;

int n = poll(fds, nfds, timeout);

for (int i = 0; i < nfds; i++) {
    if (fds[i].revents & POLLIN) {
        // 处理 fd
    }
}

// 相比 select：
// ✅ 没有 1024 限制
// ❌ 仍需每次拷贝全部 fd，仍需遍历
```

### epoll
```c
// 伪代码
// 1. 创建 epoll 实例
int epfd = epoll_create(1);

// 2. 注册 fd（只需一次）
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// 3. 等待事件
struct epoll_event events[MAX_EVENTS];
int n = epoll_wait(epfd, events, MAX_EVENTS, timeout);

// 4. 只遍历就绪的 fd
for (int i = 0; i < n; i++) {
    // events[i].data.fd 就是就绪的 fd
    handle(events[i].data.fd);
}

// 优势：
// ✅ fd 注册一次，不需要每次拷贝
// ✅ 只返回就绪的 fd，不需要遍历全部
// ✅ 支持边缘触发 (ET) 模式
```

### 水平触发 (LT) vs 边缘触发 (ET)
```
# 水平触发 (Level Triggered)：
# - 只要缓冲区有数据可读，就一直通知
# - select/poll 只支持 LT
# - 编程简单，不容易丢数据

# 边缘触发 (Edge Triggered)：
# - 只在状态变化时通知一次（从无数据变为有数据）
# - 必须一次性读完所有数据（配合非阻塞 IO）
# - 性能更高（减少 epoll_wait 调用次数）
# - 编程复杂，处理不当会丢数据

# ET 模式的典型写法：
# while True:
#     n = read(fd, buf, size)
#     if n == -1 and errno == EAGAIN:
#         break  # 读完了
```

### Python IO 多路复用示例
```python
import selectors
import socket

sel = selectors.DefaultSelector()  # Linux 上自动选择 epoll

def accept(sock, mask):
    conn, addr = sock.accept()
    conn.setblocking(False)
    sel.register(conn, selectors.EVENT_READ, read)

def read(conn, mask):
    data = conn.recv(1024)
    if data:
        conn.sendall(data)
    else:
        sel.unregister(conn)
        conn.close()

server = socket.socket()
server.bind(('0.0.0.0', 8080))
server.listen(128)
server.setblocking(False)
sel.register(server, selectors.EVENT_READ, accept)

while True:
    events = sel.select()  # 阻塞等待事件
    for key, mask in events:
        callback = key.data
        callback(key.fileobj, mask)
```

---

## 第5部分：Reactor 模式

### 单 Reactor 单线程
```
                    ┌─────────────┐
                    │   Reactor   │
                    │  (epoll)    │
                    │             │
                    │  事件循环    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ↓            ↓            ↓
         Acceptor      Handler 1    Handler 2
       (处理新连接)   (读/业务/写)  (读/业务/写)

# 特点：所有操作在一个线程
# 优点：简单，无线程安全问题
# 缺点：业务处理阻塞会影响其他连接
# 代表：Redis 6.0 之前
```

### 单 Reactor 多线程
```
                    ┌─────────────┐
                    │   Reactor   │
                    │  (epoll)    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ↓            ↓            ↓
         Acceptor      Handler 1    Handler 2
       (处理新连接)    (读/写)      (读/写)
                           │            │
                      ┌────┴────────────┴────┐
                      │     线程池            │
                      │  (处理业务逻辑)        │
                      └──────────────────────┘

# 特点：IO 在主线程，业务处理在线程池
# 优点：充分利用多核 CPU
# 缺点：主线程仍是瓶颈
```

### 多 Reactor 多线程（主从 Reactor）
```
         ┌──────────────┐
         │  Main Reactor │     (主 Reactor，只负责 accept)
         │   (epoll)     │
         └───────┬───────┘
                 │ 分发新连接
        ┌────────┼────────┐
        ↓        ↓        ↓
   Sub Reactor Sub Reactor Sub Reactor   (子 Reactor，负责 IO)
    (epoll)     (epoll)    (epoll)
      │           │          │
   Handler     Handler    Handler       (读/业务/写)

# 特点：
# - Main Reactor 只负责 accept 新连接
# - Sub Reactor 负责已建立连接的 IO 事件
# - 每个 Reactor 在自己的线程中运行

# 优点：充分利用多核，高并发
# 代表：Nginx, Netty, Memcached
```

---

## 第6部分：面试场景题

### 从输入 URL 到页面显示，经历了什么？

> 完整详解见 [url_to_page.md](url_to_page.md)

```
URL 解析 → DNS 解析 → TCP 握手 → TLS 握手 → HTTP 请求
→ 服务端处理 → HTTP 响应 → 浏览器渲染 → 连接管理
```

### 如何设计一个高并发服务器？
```
# 1. IO 模型选择
#    - epoll (Linux) / kqueue (macOS) / IOCP (Windows)
#    - 边缘触发 + 非阻塞 IO

# 2. 线程模型
#    - 多 Reactor 多线程（主从模式）
#    - 或 协程模型（Go goroutine, Python asyncio）

# 3. 连接管理
#    - 连接池复用
#    - 长连接 + 心跳检测
#    - 超时回收空闲连接

# 4. 协议设计
#    - 二进制协议（比文本协议高效）
#    - 长度前缀解决粘包

# 5. 其他优化
#    - 零拷贝 (sendfile, mmap)
#    - 内存池减少 malloc/free
#    - 无锁队列减少竞争
```
