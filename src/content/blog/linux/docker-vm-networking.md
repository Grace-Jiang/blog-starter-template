---
title: Docker与虚拟机网络模式全解析
description: Docker七种网络模式、虚拟机三种网络模式、veth pair、Linux Bridge、iptables NAT、跨主机通信
pubDate: '2025-02-28'
categories:
- Linux
tags:
- Docker
- 网络
- Bridge
- 虚拟机
- 面试
---
# Linux 网络模式 — Docker 容器 & 虚拟机

## 第1部分：面试常见问题与答案

### 面试高频问题
```
# Q1: Docker 有哪几种网络模式？各有什么特点？
# 答：
# 1. bridge（默认）：容器通过 veth pair 连接到 docker0 网桥，有独立 Network Namespace
# 2. host：容器直接使用宿主机网络栈，没有网络隔离
# 3. none：容器只有 lo 回环接口，无外部网络
# 4. container：与另一个容器共享 Network Namespace
# 5. overlay：跨主机容器通信（Swarm/K8s 场景）
# 6. macvlan：容器拥有独立 MAC 地址，直接接入物理网络
# 7. ipvlan：容器共享父接口 MAC，只分配不同 IP（适合云环境/MAC 受限场景）

# Q2: 虚拟机有哪几种网络模式？
# 答：
# 1. NAT：虚拟机通过宿主机 IP 上网，外部无法直接访问虚拟机
# 2. 桥接（Bridged）：虚拟机与宿主机在同一网段，有独立 IP
# 3. Host-Only：虚拟机只能与宿主机通信，无法访问外网

# Q3: Docker bridge 模式的网络通信原理？
# 答：
# - Docker 创建 docker0 虚拟网桥（默认 172.17.0.0/16）
# - 每个容器通过 veth pair 连接到 docker0
# - 容器间通过 docker0 网桥转发通信
# - 容器访问外网通过 NAT（iptables MASQUERADE）
# - 外部访问容器通过端口映射（-p 参数，iptables DNAT）

# Q4: Docker host 模式和 bridge 模式的区别？
# 答：
# - host：容器与宿主机共享网络栈，性能最好，但端口会冲突
# - bridge：容器有独立网络命名空间，有一定隔离性，需要端口映射

# Q5: 虚拟机 NAT 和桥接模式的区别？
# 答：
# - NAT：VM 通过宿主机做地址转换上网，外部看不到 VM 的 IP
# - 桥接：VM 拥有与宿主机同网段的独立 IP，相当于网络中的一台独立主机

# Q6: Bridge 模式下容器数据包怎么访问外网？
# 答：
# 1. 容器把 docker0 当网关，数据包的目标 MAC 填 docker0 的 MAC
# 2. docker0 收到后发现目标 MAC 是自己 → 上送内核协议栈（因为 docker0 有 IP，是内核的网络接口）
# 3. 内核查路由表，匹配默认路由，决定从 eth0 发出
# 4. 经过 iptables POSTROUTING 链的 MASQUERADE 规则，源 IP 从容器 IP 替换为宿主机 IP
# 5. conntrack 记录映射关系，回包时自动反向还原目标地址送回容器

# Q7: Host 模式下容器用的是宿主机 IP 吗？
# 答：
# 是的。Host 模式下容器不创建独立的 Network Namespace，
# 直接共享宿主机的网络栈（网卡、IP、路由表、iptables 全部共享）。
# 容器监听的端口 = 宿主机监听的端口，不需要 -p 端口映射。
# 缺点是端口会冲突，且没有网络隔离。

# Q8: Docker 容器跨主机通信怎么实现？
# 答：
# 1. overlay 网络（Docker Swarm / Flannel）：VXLAN 封装
# 2. macvlan：容器直接获得物理网络 IP
# 3. 第三方方案：Calico（BGP）、Weave、Cilium（eBPF）
```

### 面试准备清单

✅ 能说出 Docker 的 7 种网络模式及各自特点  
✅ 能画出 bridge 模式的网络拓扑图  
✅ 理解 veth pair、Network Namespace、网桥的概念  
✅ 能解释 bridge 模式下容器访问外网的完整链路（MAC → 内核路由 → SNAT → conntrack）  
✅ 能解释端口映射（-p）的底层原理（iptables DNAT）  
✅ 能说清 host 模式下容器直接使用宿主机 IP 和网络栈  
✅ 能区分 Macvlan 和 IPvlan（独立 MAC vs 共享 MAC）  
✅ 能区分虚拟机的 NAT、桥接、Host-Only 模式  
✅ 了解容器跨主机通信方案（overlay、macvlan、Calico）  
✅ 理解 Docker 网络与虚拟机网络的本质区别  

---

## 第2部分：Docker 容器网络模式

### 核心概念

```
# Network Namespace（网络命名空间）
# - Linux 内核特性，实现网络隔离
# - 每个 namespace 有独立的网卡、路由表、iptables 规则
# - Docker 容器的网络隔离就基于此

# veth pair（虚拟以太网对）
# - 一对虚拟网卡，数据从一端进，另一端出
# - 用于连接不同的 Network Namespace
# - 类比：一根网线的两个接头

# Linux Bridge（网桥）
# - 虚拟交换机，连接多个网络接口
# - docker0 就是 Docker 默认创建的网桥
```

### 2.1 Bridge 模式（默认）

```
# 网络拓扑：

┌─────────────────────────────────────────────────────┐
│                     宿主机                           │
│                                                     │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐      │
│  │ Container1 │  │ Container2 │  │ Container3 │      │
│  │ eth0       │  │ eth0       │  │ eth0       │      │
│  │ 172.17.0.2 │  │ 172.17.0.3 │  │ 172.17.0.4 │      │
│  └──┬────────┘  └──┬────────┘  └──┬────────┘      │
│     │ veth         │ veth         │ veth           │
│  ┌──┴──────────────┴──────────────┴──┐             │
│  │         docker0 (172.17.0.1)       │  ← 虚拟网桥 │
│  └──────────────┬─────────────────────┘             │
│                 │                                    │
│           iptables NAT   （SNAT, POSTROUTING）      │
│                 │                                    │
│  ┌──────────────┴──────────────┐                    │
│  │      eth0 (物理网卡)         │                    │
│  │      192.168.1.100           │                    │
│  └──────────────┬──────────────┘                    │
└─────────────────┼───────────────────────────────────┘
                  │
              外部网络
```

docker0 是一个二层设备（交换机），它收到帧后看 目标 MAC：

情况1：目标 MAC = 某个 veth 的 MAC
  → 二层转发，从对应的 veth 口发出（容器间通信）

情况2：目标 MAC = docker0 自己的 MAC   ← 访问外网走这条
  → 这个包是发给"我自己"的
  → 上送到内核协议栈做三层处理

内核匹配到默认路由，决定从 物理网卡 eth0 转发出去

SNAT（源地址转换）：将容器的私有 IP（172.17.0.x）转换为宿主机的公网 IP（192.168.1.100）



#### Bridge 模式容器访问外网的完整链路

```
容器 eth0 (172.17.0.2)
    │
    │ veth pair
    ▼
docker0 网桥 (172.17.0.1)
    │
    │ ① 网桥收到帧，看目标 MAC：
    │    目标 MAC = docker0 自己的 MAC（因为容器把 docker0 当网关）
    │    → 包是发给"我自己"的，上送内核协议栈做三层路由
    │
    │ 【为什么交给内核？】
    │    容器访问外网时不知道目标的 MAC，只知道默认网关 172.17.0.1
    │    所以容器通过 ARP 拿到 docker0 的 MAC，把包发给 docker0
    │    docker0 有 IP 地址（172.17.0.1），它是内核的一个网络接口
    │    任何发给该接口 MAC 的帧，内核都会接收并进入三层路由流程
    │    → 这不是 docker0 的特殊行为，而是 Linux 网络接口的通用行为
    │
    ▼
宿主机路由表
    │ ② 内核查路由表：
    │    default via 192.168.1.1 dev eth0  ← 匹配默认路由
    │    → 决定从 eth0 发出（前提：ip_forward=1，Docker 启动时自动开启）
    │
    ▼
iptables NAT (POSTROUTING 链)
    │ ③ MASQUERADE 规则：
    │    来自 172.17.0.0/16 且目标不是 172.17.0.0/16 的包 → 做源地址伪装
    │    改写前：src=172.17.0.2  dst=8.8.8.8
    │    改写后：src=192.168.1.100  dst=8.8.8.8
    │    同时 conntrack 记录映射关系
    │
    ▼
宿主机 eth0 (192.168.1.100)  → 物理网关 → 外网

回包路径（反向）：
    外网响应：src=8.8.8.8  dst=192.168.1.100
    → conntrack 查表，还原为：src=8.8.8.8  dst=172.17.0.2
    → 路由到 docker0 → veth → 容器
```

```bash
# 创建 bridge 网络（默认就是 bridge）
docker run -d --name web --network bridge -p 8080:80 nginx

# 查看 docker0 网桥
ip addr show docker0
brctl show docker0       # 查看桥接的 veth 接口

# 查看容器网络
docker inspect web --format '{{.NetworkSettings.IPAddress}}'

# 查看 iptables NAT 规则（端口映射原理）
iptables -t nat -L -n
# DNAT: 外部访问宿主机:8080 → 转发到容器:80
# MASQUERADE: 容器访问外网 → 源地址替换为宿主机 IP
```

**特点：**
- 容器有独立 IP（172.17.0.x 段）
- 容器间可通过 docker0 网桥互相通信
- 访问外网通过 NAT
- 外部访问容器需要端口映射（`-p`）
- 有一定的网络性能损耗（经过网桥和 NAT）

### 2.2 Host 模式

```
# 网络拓扑：

┌──────────────────────────────────────┐
│                宿主机                 │
│                                      │
│  ┌───────────┐  ┌───────────┐       │
│  │ Container1 │  │ Container2 │       │
│  │ (共享宿主机 │  │ (共享宿主机 │       │
│  │  网络栈)   │  │  网络栈)   │       │
│  └───────────┘  └───────────┘       │
│                                      │
│  ┌──────────────────────────────┐   │
│  │      eth0 (物理网卡)          │   │
│  │      192.168.1.100            │   │
│  └──────────────┬───────────────┘   │
└─────────────────┼────────────────────┘
                  │
              外部网络
```

**Host 模式 = 容器直接使用宿主机的 IP 和网络栈，没有独立的 Network Namespace。**

```bash
# Bridge 模式 vs Host 模式对比验证：

# bridge 模式 → 容器有独立 IP
docker run --rm alpine ip addr show
# eth0: 172.17.0.2    ← 容器自己的 IP

# host 模式 → 容器看到的就是宿主机网卡
docker run --rm --network host alpine ip addr show
# eth0: 192.168.1.100  ← 宿主机的 IP，和宿主机完全一样
```

```bash
# 使用 host 模式
docker run -d --name web --network host nginx

# 容器直接监听宿主机的 80 端口，不需要 -p 映射
curl localhost:80

# 容器内看到的网卡和宿主机一模一样
docker exec web ip addr show
```

**特点：**
- 容器与宿主机共享 Network Namespace
- 不需要端口映射，容器监听的端口 = 宿主机监听的端口
- **性能最好**（没有 NAT 和网桥的开销）
- **缺点**：端口冲突、没有网络隔离

```bash
# ❌ 端口冲突问题
docker run --network host nginx    # 绑定 :80 成功
docker run --network host nginx    # 绑定 :80 失败，端口已被占用

# ❌ 没有网络隔离
# 容器能看到宿主机所有网络接口、所有端口、所有连接
```

**Kubernetes 中的 hostNetwork：**

```yaml
# K8s 中等价的配置
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  hostNetwork: true    # ← 等价于 docker --network host
  containers:
  - name: app
    image: nginx
```

**适用场景：**
- 对网络性能要求极高的应用（如网络监控、高频交易）
- K8s 网络组件本身（kube-proxy、flannel、calico-node）需要操作宿主机网络
- Ingress Controller 需要直接绑定宿主机 80/443 端口

### 2.3 None 模式

```bash
# 使用 none 模式
docker run -d --name isolated --network none alpine sleep 3600

# 容器内只有 lo 回环接口
docker exec isolated ip addr show
# 1: lo: <LOOPBACK,UP,LOWER_UP>
#    inet 127.0.0.1/8 scope host lo
```

**特点：**
- 容器只有 lo 接口，完全网络隔离
- 需要手动配置网络
- **适用场景**：安全敏感的离线计算任务

### 2.4 Container 模式

```
# 网络拓扑：

┌───────────────────────────────┐
│  Pod / Container Group         │
│  ┌──────────┐ ┌──────────┐   │
│  │ Container1│ │ Container2│   │
│  │ (nginx)   │ │ (php-fpm) │   │  ← 共享同一个 Network Namespace
│  └──────────┘ └──────────┘   │
│        共享 eth0: 172.17.0.2  │
└──────────────┬────────────────┘
               │ veth
          ┌────┴────┐
          │ docker0  │
          └──────────┘
```

```bash
# 先启动一个容器
docker run -d --name app1 nginx

# 第二个容器共享 app1 的网络
docker run -d --name app2 --network container:app1 php:fpm

# app2 可以通过 localhost 访问 app1 的服务
docker exec app2 curl localhost:80
```

**特点：**
- 两个容器共享同一个 Network Namespace
- 通过 localhost 互相通信（性能好）
- **Kubernetes Pod 的网络实现原理**就是基于此（pause 容器）

### 2.5 Overlay 模式（跨主机通信）

```
# 网络拓扑：

┌───────────── Host 1 ──────────────┐   ┌───────────── Host 2 ──────────────┐
│  ┌─────────┐    ┌─────────┐       │   │       ┌─────────┐    ┌─────────┐ │
│  │ ContA   │    │ ContB   │       │   │       │ ContC   │    │ ContD   │ │
│  │10.0.0.2 │    │10.0.0.3 │       │   │       │10.0.0.4 │    │10.0.0.5 │ │
│  └──┬──────┘    └──┬──────┘       │   │       └──┬──────┘    └──┬──────┘ │
│     │ veth         │ veth         │   │          │ veth         │ veth   │
│  ┌──┴──────────────┴──────────┐   │   │   ┌──────┴──────────────┴──┐     │
│  │    br0 (overlay bridge)    │   │   │   │   br0 (overlay bridge) │     │
│  └────────────┬───────────────┘   │   │   └───────────┬────────────┘     │
│               │                   │   │               │                   │
│          VXLAN 封装               │   │          VXLAN 解封               │
│               │                   │   │               │                   │
│  ┌────────────┴───────────────┐   │   │   ┌───────────┴────────────┐     │
│  │   eth0: 192.168.1.100     │   │   │   │   eth0: 192.168.1.101  │     │
│  └────────────┬───────────────┘   │   │   └───────────┬────────────┘     │
└───────────────┼───────────────────┘   └───────────────┼───────────────────┘
                └───────────── 物理网络 ─────────────────┘
```

```bash
# 创建 overlay 网络（需要 Swarm 模式或外部 KV 存储）
docker network create -d overlay my-overlay

# VXLAN 原理：
# 将容器的二层帧封装在 UDP 包中（目标端口 4789）
# 外层用宿主机 IP 路由，内层保持容器网络通信
# 类似"隧道"技术
```

### 2.6 Macvlan 模式

```
# Macvlan = 独立 MAC + 独立 IP，容器在网络中像一台完全独立的物理机
#
# 宿主机 eth0:   MAC=aa:bb:cc:dd:ee:01   IP=192.168.1.100
# Container1:    MAC=aa:bb:cc:dd:ee:02   IP=192.168.1.201  ← 独立 MAC + 独立 IP
# Container2:    MAC=aa:bb:cc:dd:ee:03   IP=192.168.1.202  ← 独立 MAC + 独立 IP
#
# 注意：IP 不是宿主机 IP，而是和宿主机同网段的独立 IP
# 从路由器/交换机视角看，这三个就是三台不同的主机

# 网络拓扑：

                     ┌──────────────┐
                     │   路由器       │
                     │ 192.168.1.1   │
                     └──┬────┬────┬─┘
                        │    │    │
          ┌─────────────┘    │    └─────────────┐
          │                  │                  │
┌─────────┴───────┐ ┌───────┴───────┐ ┌────────┴──────┐
│ 宿主机           │ │ Container1     │ │ Container2    │
│ MAC=...ee:01    │ │ MAC=...ee:02   │ │ MAC=...ee:03  │
│ IP=192.168.1.100│ │ IP=192.168.1.201│ │ IP=192.168.1.202│
└─────────────────┘ └────────────────┘ └───────────────┘
  ↑ 路由器看来，三个都是独立主机，各有独立 MAC 和 IP
```

**和其他模式的区别（重点理解 IP 归属）：**

| 模式 | MAC | IP | 访问外网方式 |
|------|-----|-----|------------|
| **bridge** | 独立 MAC | 独立 IP（172.17.0.x 内部网段） | 需要 NAT |
| **host** | 用宿主机 MAC | 用宿主机 IP | 直接（共享网络栈） |
| **macvlan** | 独立 MAC | 独立 IP（和宿主机同网段） | 直接（路由器可达，不需要 NAT） |
| **ipvlan** | 共享宿主机 MAC | 独立 IP（和宿主机同网段） | 直接（路由器可达，不需要 NAT） |

```bash
# 创建 macvlan 网络
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-macvlan

# 容器直接获得物理网络中的独立 IP（不是宿主机 IP）
docker run -d --name web --network my-macvlan --ip 192.168.1.200 nginx

# 验证：容器有自己的 MAC 和 IP
docker exec web ip addr show
# eth0: <BROADCAST,MULTICAST,UP>
#     link/ether 02:42:c0:a8:01:c8    ← 独立 MAC
#     inet 192.168.1.200/24           ← 独立 IP，不是宿主机 IP
```

**特点：**
- 容器拥有独立的 MAC 地址 + 独立的 IP 地址（和宿主机同网段）
- 直接接入物理网络，像一台独立主机，路由器直接可达
- **性能接近物理网络**（不经过 NAT 和网桥）
- 缺点：需要网络设备支持混杂模式、MAC 地址数量有限

#### Macvlan 的限制

**① 必须开启混杂模式（Promiscuous Mode）**

```
# 为什么需要混杂模式？
#
# Macvlan 容器有独立 MAC，外部回包时 dst_MAC = 容器的 MAC
# 但物理上只有一张网卡，这张网卡的 MAC 是宿主机的 MAC
#
# 外部回包：dst_MAC = aa:bb:cc:dd:ee:02（容器 MAC）
# 网卡自己：MAC = aa:bb:cc:dd:ee:01（宿主机 MAC）
#
# 正常模式：dst_MAC ≠ 我的 MAC → 丢弃  ❌ 容器收不到包
# 混杂模式：不管 dst_MAC 是谁 → 全部接收 → 内核按 MAC 分发给对应的 macvlan 子接口 ✅
#
# 对比其他模式：
# bridge：回包 dst_MAC = 宿主机 MAC（因为 SNAT 了）→ 不需要混杂模式
# host：回包 dst_MAC = 宿主机 MAC（共享网络栈）→ 不需要混杂模式
# ipvlan：回包 dst_MAC = 宿主机 MAC（共享 MAC）→ 不需要混杂模式
# macvlan：回包 dst_MAC = 容器独立 MAC ≠ 网卡 MAC → 必须混杂模式

ip link set eth0 promisc on

# 云环境（AWS/阿里云等）通常禁止混杂模式 → Macvlan 无法使用
```

**各模式回包 dst_MAC 对比：**

| 模式 | 回包的 dst_MAC | 网卡能正常收吗 | 需要混杂模式 |
|------|---------------|-------------|------------|
| **bridge** | 宿主机 MAC（SNAT 后回包目标是宿主机） | ✅ | 不需要 |
| **host** | 宿主机 MAC（共享网络栈） | ✅ | 不需要 |
| **ipvlan** | 宿主机 MAC（共享 MAC） | ✅ | 不需要 |
| **macvlan** | 容器独立 MAC | ❌ 不是网卡的 MAC | **需要** |

**② 宿主机与 Macvlan 容器之间无法直接通信**

```
# 这是最容易踩的坑！
# 宿主机 ping 自己的 Macvlan 容器 → 不通

宿主机(192.168.1.100) → ping → Container(192.168.1.201)  ❌ 不通

# 原因：Linux 内核设计上，父接口（eth0）和它的 macvlan 子接口之间
#       的流量会被内核直接丢弃，不会走到物理网络再回来

# 解决方案：在宿主机上额外创建一个 macvlan 接口用于通信
ip link add macvlan-host link eth0 type macvlan mode bridge
ip addr add 192.168.1.250/24 dev macvlan-host
ip link set macvlan-host up
# 宿主机通过 macvlan-host 接口就能和容器通信了
```

**③ 交换机 MAC 地址表限制**

```
# 每个容器一个独立 MAC → 交换机 MAC 表需要记录所有容器的 MAC
# 交换机 MAC 表容量有限（通常几千到几万条）
# 容器数量多了会导致 MAC 表溢出 → 交换机退化为广播模式

# 100 个容器 = 交换机多 100 条 MAC 记录
# 对比 IPvlan：所有容器共享一个 MAC，交换机只需 1 条记录
```

**④ 云平台/虚拟化环境限制**

```
# AWS EC2：默认不允许非实例 MAC 的流量（需配置安全组允许）
# 阿里云 ECS：不支持混杂模式
# VMware/VirtualBox：需要手动开启混杂模式
# OpenStack：需要配置 port security 允许多 MAC
#
# → 云环境首选 IPvlan，不需要混杂模式，MAC 不变
```

### 2.8 IPvlan 模式

```
# 与 Macvlan 的核心区别：所有容器共享父接口的 MAC 地址，只有 IP 不同

# Macvlan：每个容器有独立的 MAC + 独立的 IP
# IPvlan：所有容器共享同一个 MAC，只有 IP 不同

┌─────────────────────────────────────────────────────┐
│                     宿主机                           │
│                                                     │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐      │
│  │ Container1 │  │ Container2 │  │ Container3 │      │
│  │ 192.168.1.201│ │ 192.168.1.202│ │ 192.168.1.203│  │
│  │ MAC=同父接口│  │ MAC=同父接口│  │ MAC=同父接口│      │
│  └──┬────────┘  └──┬────────┘  └──┬────────┘      │
│     │              │              │                 │
│  ┌──┴──────────────┴──────────────┴──┐             │
│  │     eth0 (父接口) 192.168.1.100    │             │
│  │     MAC: aa:bb:cc:dd:ee:ff         │  ← 共享 MAC │
│  └──────────────┬─────────────────────┘             │
└─────────────────┼───────────────────────────────────┘
                  │
              物理网络
```

**IPvlan 两种模式：**

```
# L2 模式（默认）：
# - 容器和父接口在同一子网
# - 类似 Macvlan，但共享 MAC
# - 交换机只看到一个 MAC 地址

# L3 模式：
# - 容器可以在不同子网
# - 父接口充当路由器角色
# - 不需要 ARP/广播，纯三层路由转发
# - 适合大规模容器部署
```

```bash
# 创建 IPvlan L2 网络
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  my-ipvlan-l2

# 创建 IPvlan L3 网络
docker network create -d ipvlan \
  --subnet=10.10.0.0/24 \
  -o parent=eth0 \
  -o ipvlan_mode=l3 \
  my-ipvlan-l3

docker run -d --name web --network my-ipvlan-l2 --ip 192.168.1.201 nginx
```

**Macvlan vs IPvlan 对比：**

| 特性 | Macvlan | IPvlan |
|------|---------|--------|
| **MAC 地址** | 每个容器独立 MAC | 所有容器共享父接口 MAC |
| **混杂模式** | 需要（物理交换机/网卡） | 不需要 |
| **云环境兼容** | ❌ 差（云平台通常限制 MAC） | ✅ 好（MAC 不变） |
| **交换机 MAC 表** | 每个容器占一条 | 只占一条 |
| **L3 路由能力** | 无 | IPvlan L3 模式支持 |
| **适用场景** | 物理机/少量容器 | 云环境/大规模容器 |

```
# 选择建议：
# 物理机 + 少量容器 + 需要独立 MAC → Macvlan
# 云环境 / MAC 受限 / 大规模部署   → IPvlan
# 需要跨子网三层路由              → IPvlan L3
```

---

## 第3部分：Docker 自定义网络

### 用户自定义 Bridge 网络

```bash
# 创建自定义网络
docker network create --driver bridge --subnet 10.10.0.0/16 my-net

# 在自定义网络中启动容器
docker run -d --name app1 --network my-net nginx
docker run -d --name app2 --network my-net alpine sleep 3600

# 自定义网络支持 DNS 解析（用容器名互相访问）
docker exec app2 ping app1   # ✅ 可以用容器名通信

# 默认 bridge 网络不支持 DNS
docker run -d --name app3 nginx
docker run -d --name app4 alpine sleep 3600
docker exec app4 ping app3   # ❌ 无法解析容器名
```

**自定义 Bridge vs 默认 Bridge：**

| 特性 | 默认 bridge | 自定义 bridge |
|------|-----------|-------------|
| **DNS 解析** | 不支持容器名 | 支持容器名互相解析 |
| **网络隔离** | 所有容器在同一网络 | 不同网络间默认隔离 |
| **灵活性** | 不可配置子网 | 可自定义子网、网关 |

---

## 第4部分：虚拟机网络模式（VMware/VirtualBox）

### 4.1 NAT 模式

```
# 网络拓扑：

                        外部网络（如公司局域网）
                            │
┌───────────────────────────┼──────────────────────────┐
│ 宿主机                     │                          │
│                ┌───────────┴──────────┐               │
│                │ 物理网卡 192.168.1.100│               │
│                └───────────┬──────────┘               │
│                            │                          │
│                      NAT 引擎                         │
│                 （地址转换 + DHCP）                     │
│                            │                          │
│                ┌───────────┴──────────┐               │
│                │ 虚拟交换机（VMnet8）   │               │
│                └─────┬──────────┬─────┘               │
│                      │          │                     │
│  ┌───────────────────┴┐  ┌─────┴──────────────────┐  │
│  │ VM1                 │  │ VM2                     │  │
│  │ eth0: 10.0.2.15     │  │ eth0: 10.0.2.16        │  │
│  └─────────────────────┘  └────────────────────────┘  │
└───────────────────────────────────────────────────────┘

# 数据流（VM 访问外网）：
# VM(10.0.2.15) → NAT引擎 → 替换源IP为宿主机IP(192.168.1.100) → 外部网络
# 响应：外部网络 → 宿主机(192.168.1.100) → NAT引擎 → VM(10.0.2.15)
```

**特点：**
- VM 在独立的虚拟子网中（如 10.0.2.0/24）
- VM 可以访问外网（通过宿主机 NAT）
- 外部网络**无法直接访问** VM
- 需要端口转发才能从外部访问 VM 中的服务
- **最常用的模式**（开箱即用，不依赖外部网络配置）

```bash
# VirtualBox 端口转发示例
VBoxManage modifyvm "myvm" --natpf1 "ssh,tcp,,2222,,22"
# 宿主机:2222 → VM:22
```

### 4.2 桥接模式（Bridged）

```
# 网络拓扑：

                        外部网络（路由器/交换机）
                            │
                     ┌──────┴──────┐
                     │   路由器      │
                     │ 192.168.1.1  │
                     └──┬───┬───┬──┘
                        │   │   │
          ┌─────────────┘   │   └─────────────┐
          │                 │                 │
┌─────────┴───────┐ ┌──────┴────────┐ ┌──────┴────────┐
│ 宿主机           │ │ VM1            │ │ VM2            │
│ 192.168.1.100   │ │ 192.168.1.101  │ │ 192.168.1.102  │
└─────────────────┘ └────────────────┘ └────────────────┘

# VM 和宿主机在同一网段，从路由器视角看是独立的主机
```

**特点：**
- VM 获得与宿主机同网段的独立 IP（从路由器 DHCP 获取）
- VM 相当于网络中的一台独立物理机
- VM 可以被局域网内其他主机直接访问
- **性能最好**，通信不需要 NAT
- **缺点**：依赖外部网络环境、占用额外 IP 地址

**适用场景：** VM 需要被局域网内其他设备访问（如搭建测试服务器）

### 4.3 Host-Only 模式（仅主机）

```
# 网络拓扑：

                        外部网络
                            │
┌───────────────────────────┼──────────────────────────┐
│ 宿主机                     │                          │
│                ┌───────────┴──────────┐               │
│                │ 物理网卡 192.168.1.100│               │
│                └─────────────────────┘               │
│                   （与虚拟网络不连通）                   │
│                                                      │
│                ┌─────────────────────┐               │
│                │ 虚拟网卡（VMnet1）    │               │
│                │ 192.168.56.1        │               │
│                └─────┬──────────┬────┘               │
│                      │          │                     │
│  ┌───────────────────┴┐  ┌─────┴──────────────────┐  │
│  │ VM1                 │  │ VM2                     │  │
│  │ 192.168.56.101      │  │ 192.168.56.102          │  │
│  └─────────────────────┘  └────────────────────────┘  │
└───────────────────────────────────────────────────────┘

# VM 只能与宿主机和同网络内的其他 VM 通信
# ❌ 不能访问外网
# ❌ 外部不能访问 VM
```

**特点：**
- VM 在隔离的虚拟子网中
- VM ↔ 宿主机 可以通信
- VM ↔ VM 可以通信（同一 Host-Only 网络内）
- VM **无法访问外网**
- 外部网络**无法访问** VM
- 安全隔离性最强

**适用场景：** 搭建隔离的测试/开发环境

### 4.4 内部网络模式（Internal / VirtualBox 特有）

```
# 与 Host-Only 类似，但更严格：
# VM ↔ VM 可以通信
# VM ↔ 宿主机 不可以通信（比 Host-Only 更隔离）
# 完全封闭的网络环境
```

---

## 第5部分：虚拟机三种模式对比

| 特性 | NAT | 桥接 (Bridged) | Host-Only |
|------|-----|----------------|-----------|
| **VM → 外网** | ✅ 可以 | ✅ 可以 | ❌ 不行 |
| **外网 → VM** | ❌ 不行（需端口转发） | ✅ 可以 | ❌ 不行 |
| **VM ↔ 宿主机** | ✅ 可以 | ✅ 可以 | ✅ 可以 |
| **VM ↔ VM** | ❌ 不行（默认） | ✅ 可以 | ✅ 可以 |
| **IP 分配** | 虚拟 DHCP | 外部 DHCP/手动 | 虚拟 DHCP |
| **是否占外部 IP** | 否 | 是 | 否 |
| **配置难度** | 低（默认） | 中 | 低 |
| **网络性能** | 一般 | 最好 | 一般 |
| **适用场景** | 日常开发 | 服务器/测试 | 隔离环境 |

---

## 第6部分：Docker vs 虚拟机网络对比

| 维度 | Docker 网络 | 虚拟机网络 |
|------|-----------|----------|
| **隔离层级** | Network Namespace（内核级） | 完整虚拟网卡（硬件级） |
| **性能开销** | 小（共享内核） | 大（虚拟化层） |
| **默认模式** | bridge | NAT |
| **网络配置** | docker network 命令 | 虚拟化软件 GUI/CLI |
| **跨主机通信** | overlay/macvlan/Calico | VPN/桥接 |
| **端口映射** | docker run -p | 端口转发规则 |
| **DNS** | 自定义网络内置 DNS | 依赖外部 DNS |

### 底层技术对比
```
# Docker Bridge 模式底层：
# Linux Namespace + veth pair + Linux Bridge + iptables NAT
# → 纯软件实现，轻量级

# 虚拟机 NAT 模式底层：
# Hypervisor 虚拟网卡 + 虚拟交换机 + NAT 引擎
# → 硬件虚拟化，重量级

# 本质区别：
# Docker 在操作系统层面隔离（共享内核）
# 虚拟机在硬件层面隔离（独立内核）
```

---

## 第7部分：常用命令速查

### Docker 网络命令

```bash
# 查看所有网络
docker network ls

# 创建自定义网络
docker network create --driver bridge --subnet 10.0.0.0/16 my-net

# 查看网络详情
docker network inspect bridge

# 将容器连接/断开网络
docker network connect my-net container1
docker network disconnect my-net container1

# 查看容器 IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container1

# 查看 docker0 网桥
brctl show
ip link show docker0

# 查看 veth pair
ip link show type veth

# 查看 iptables NAT 规则（端口映射）
iptables -t nat -L -n -v
```

### 虚拟机网络命令（VirtualBox）

```bash
# 查看虚拟机网络配置
VBoxManage showvminfo "myvm" | grep NIC

# 设置网络模式
VBoxManage modifyvm "myvm" --nic1 nat          # NAT
VBoxManage modifyvm "myvm" --nic1 bridged      # 桥接
VBoxManage modifyvm "myvm" --nic1 hostonly      # Host-Only

# 设置桥接的物理网卡
VBoxManage modifyvm "myvm" --bridgeadapter1 eth0

# 设置端口转发（NAT 模式）
VBoxManage modifyvm "myvm" --natpf1 "ssh,tcp,,2222,,22"
VBoxManage modifyvm "myvm" --natpf1 "http,tcp,,8080,,80"

# 查看端口转发规则
VBoxManage showvminfo "myvm" | grep "NIC.*Rule"
```

### Linux 网络排查

```bash
# 查看网络命名空间
ip netns list
ip netns exec <ns_name> ip addr show

# 查看网桥
brctl show
bridge link show

# 查看路由
ip route show
docker exec <container> ip route show

# 抓包分析
tcpdump -i docker0 -nn                    # 抓 docker0 上的包
tcpdump -i veth* -nn port 80              # 抓 veth 接口上的包
nsenter -t <container_pid> -n tcpdump     # 在容器网络命名空间中抓包
```

---

## 第8部分：Host 接口仅有 Link-Local IPv6 地址时的网络模式选择

### 场景定义

```
# 宿主机网络状态：
# eth0 上只有 link-local IPv6 地址，没有 IPv4，没有 global/ULA IPv6
#
# ip addr show eth0:
#   inet6 fe80::xxxx:xxxx:xxxx:xxxx/64 scope link
#   (NO inet, NO inet6 global)
#
# 这种情况常见于：
# - 刚接入网络、尚未通过 DHCP/DHCPv6/SLAAC 获取地址的主机
# - 纯 L2 互联场景（如裸金属部署的 provisioning 阶段）
# - 某些边缘计算/IoT 场景
```

### Link-Local IPv6 关键特性

```
# fe80::/10 地址特性：
#
# 1. 自动生成 —— 任何 IPv6 接口启用后内核自动分配，不需要 DHCP/手动配置
# 2. 不可路由 —— 路由器不会转发 src 或 dst 为 link-local 的包（RFC 4291）
# 3. 仅本链路有效 —— 只能和同一 L2 网段的邻居通信
# 4. 需要 zone ID —— 多接口时必须带 %eth0 后缀指定出口（fe80::1%eth0）
# 5. NDP 正常工作 —— Neighbor Solicitation/Advertisement 全部基于 link-local
#
# 能做什么：ping6 同网段邻居、NDP 发现、DHCPv6 请求、mDNS/LLMNR
# 不能做什么：跨路由器通信、访问互联网、被远端主机路由到
```

### 各模式可用性总览

```
┌──────────────┬─────────┬──────────┬─────────────┬───────────┬──────────┐
│ 模式          │ C ↔ C   │ C ↔ Host │ C → 同L2邻居 │ C → 外网   │ Docker   │
│              │ 容器互通  │ 容器↔宿主 │ link-local通信│ 路由可达    │ DNS      │
├──────────────┼─────────┼──────────┼─────────────┼───────────┼──────────┤
│ bridge       │ ✅ IPv4  │ ✅ 172.17 │ ❌           │ ❌         │ ✅ 自定义 │
│ host         │ ✅ lo    │ ✅ 同ns   │ ✅ fe80::    │ ❌         │ ❌ N/A   │
│ none         │ ❌       │ ❌        │ ❌           │ ❌         │ ❌       │
│ container    │ ✅ lo    │ 继承父容器 │ 继承父容器    │ 继承父容器  │ 继承     │
│ overlay      │ ✅ 单机  │ ✅ 单机    │ ❌           │ ❌         │ ✅ 单机   │
│ macvlan      │ ✅ L2    │ ❌ 天生限制│ ✅ 独立MAC   │ ❌         │ ✅       │
│ ipvlan L2    │ ✅       │ ✅        │ ⚠️ 共享MAC   │ ❌         │ ✅       │
│ ipvlan L3    │ ✅       │ ✅        │ ❌ 禁NDP     │ ❌         │ ✅       │
└──────────────┴─────────┴──────────┴─────────────┴───────────┴──────────┘
```

### 逐模式详细分析

#### 8.1 Bridge 模式 — 内部通信完全正常，外部不通

```
# Docker 创建 docker0 网桥时自行分配 172.17.0.0/16 IPv4 子网
# 这是 Docker 内部的虚拟网络，和宿主机物理接口的地址完全无关
#
# ✅ 容器间通信：通过 docker0 二层转发，走内部 IPv4，完全正常
# ✅ 容器↔宿主机：容器通过 172.17.0.1（docker0 网关）可达宿主机
# ✅ Docker 内置 DNS：用户自定义 bridge 网络的 127.0.0.11 DNS 正常工作
#
# ❌ 容器→外网（IPv4）：
#    MASQUERADE 规则需要将源 IP 替换为宿主机出口 IP
#    但 eth0 没有 IPv4 地址 → MASQUERADE 找不到可用地址 → 丢包
#
# ❌ 容器→外网（IPv6）：
#    Docker 默认不配置 ip6tables MASQUERADE
#    即使手动配置，MASQUERADE 到 fe80:: → link-local 不可路由 → 丢包
#
# ❌ 外部→容器（端口映射 -p）：
#    eth0 没有 IPv4 地址，外部主机无法寻址宿主机
#    docker-proxy 默认绑定 0.0.0.0，不监听 IPv6

# 结论：适合纯内部隔离的容器工作负载（微服务互调、测试环境）
```

#### 8.2 Host 模式 — 同网段 link-local 通信的最佳选择

```
# 容器直接共享宿主机的 Network Namespace
# 容器看到的网络 = 宿主机的网络 = eth0 只有 fe80::
#
# ✅ 容器间通信：共享 namespace，通过 127.0.0.1 不同端口通信
# ✅ 容器↔宿主机：同一个 namespace，天然互通
# ✅ 容器→同 L2 邻居（link-local IPv6）：
#    容器可以直接使用 eth0 发送/接收 link-local IPv6 流量
#    NDP 正常工作，能发现邻居
#
# 容器能做的事：
ping6 fe80::neighbor%eth0                    # ✅ ICMPv6
curl http://[fe80::neighbor%eth0]:8080       # ✅ TCP 连接
ssh fe80::neighbor%eth0                      # ✅ SSH
# mDNS 发现 / LLMNR                           # ✅ 组播
# DHCPv6 / SLAAC 请求获取全局地址               # ✅

# 容器不能做的事：
# 跨路由器通信                                   ❌
# 访问互联网                                     ❌
# 使用外部 DNS（8.8.8.8 不可达）                  ❌

# 缺点：端口冲突（多容器不能绑同一端口）、无网络隔离
# 结论：需要和同网段设备通信时的首选（如 provisioning、PXE、集群初始化）
```

#### 8.3 None 模式 — 与宿主机网络状态无关

```
# 容器只有 lo 回环接口，完全隔离
# 无论宿主机有什么地址，none 模式的行为都一样
# 如果需要网络可以手动通过 ip netns 配置
# 结论：总是可用，用于安全隔离场景
```

#### 8.4 Container 模式 — 继承父容器的能力

```
# 与另一个容器共享 Network Namespace
# 网络能力完全取决于父容器的网络模式：
#
# 父容器用 bridge → 效果同 bridge 分析
# 父容器用 host   → 效果同 host 分析
# 父容器用 none   → 只有 lo
#
# ✅ 容器间通信：两个容器共享 localhost，通过不同端口通信（零拷贝级性能）
# 典型用途：sidecar 模式（日志收集器 + 应用、Envoy + 服务）
# 结论：总是可用，取决于父容器选择
```

#### 8.5 Overlay 模式 — 单机可用，跨机不可用

```
# Overlay 依赖 VXLAN 隧道（UDP 4789）在主机间传输
# VXLAN 需要可路由的 IP 作为隧道端点（VTEP）
#
# 单机场景：
#   ✅ 同一台宿主机上的 overlay 容器间通信 = 本地交换，正常工作
#
# 跨机场景：
#   ❌ docker swarm init 需要 --advertise-addr，link-local 不被接受/不实用
#   ❌ Swarm 的 gossip 协议（Serf）交换成员 IP，link-local 无 zone ID 会歧义
#   ❌ VXLAN remote 地址需要可路由 IP
#   ❌ Raft 共识协议需要可路由地址
#
# 结论：仅限单机使用，跨主机 overlay 需要先获取可路由地址
```

#### 8.6 Macvlan 模式 — 容器在 L2 网段独立可见

```
# 每个容器获得独立 MAC 地址，直接出现在物理 L2 网段上
# 即使没有分配全局 IP，容器也会自动生成 link-local IPv6（基于独立 MAC 的 EUI-64）
#
# ✅ 容器间通信：同父接口的 macvlan 容器通过 L2 交换互通
# ✅ 容器→同 L2 邻居：
#    每个容器有独立 MAC → 独立的 fe80:: link-local 地址
#    NDP 正常工作，邻居能发现并区分每个容器（因为 MAC 不同）
#    这是最干净的 L2 段通信方案：每个容器 = 一台独立主机
#
# ❌ 容器↔宿主机：macvlan 的天生限制 — 父接口和子接口之间不能直接通信
#    解决方案：在宿主机额外创建一个 macvlan 子接口
#    ip link add macvlan-host link eth0 type macvlan mode bridge
#
# ❌ 容器→外网：没有可路由地址
#
# Docker 创建 macvlan 网络时需要 --subnet，但 link-local 不能作为 IPAM 管理的子网
# 解决方案：指定一个虚拟的 IPv4 或 ULA IPv6 子网用于 IPAM，
#          容器同时会自动获得 link-local IPv6 用于 L2 通信

docker network create -d macvlan \
  --subnet=192.168.100.0/24 \
  -o parent=eth0 \
  my_macvlan
# 容器获得 192.168.100.x（内部 IPAM 用）+ fe80::（L2 通信用）

# 结论：需要容器在 L2 网段独立可见时的最佳选择
#       每个容器有独立 MAC 和 link-local 地址，像独立主机一样被邻居发现
```

#### 8.7 IPvlan 模式 — 容器↔宿主机通信最佳

```
# 所有容器共享父接口 MAC，只有 IP 不同
#
# === IPvlan L2 模式 ===
#
# ✅ 容器间通信：本地 L2 交换
# ✅ 容器↔宿主机：共享 MAC，内核内部处理，不像 macvlan 有父子接口限制
#    → 这是 IPvlan 相对 Macvlan 的最大优势
# ⚠️ 容器→同 L2 邻居：
#    可以工作，但所有容器共享同一 MAC
#    邻居看到多个 link-local IPv6 来自同一个 MAC（一个 MAC 多个 IP 在 IPv6 中合法）
#    NDP 能工作，但无法像 macvlan 那样让邻居区分不同容器
# ❌ 容器→外网：没有可路由地址
#
# === IPvlan L3 模式 ===
#
# ✅ 容器间通信：宿主机内部三层路由
# ✅ 容器↔宿主机：内核路由
# ❌ 容器→同 L2 邻居：L3 模式显式禁用 ARP/NDP！容器无法做邻居发现
# ❌ 容器→外网：需要外部路由指回宿主机，但宿主机无可路由地址
#
# 结论：
# IPvlan L2 → 需要容器和宿主机互通时首选（macvlan 做不到）
# IPvlan L3 → 纯内部隔离路由域，不需要 L2 邻居通信时使用
```

### 场景化选型指南

```
# 场景1：只需要容器之间互相通信（隔离工作负载、微服务测试）
# → bridge（默认）或 user-defined bridge
#   Docker 内部 IPv4 网络完全自给自足，不依赖宿主机外部地址
#   user-defined bridge 还能用容器名 DNS 解析

# 场景2：容器需要和同 L2 网段的其他主机通信（link-local IPv6）
# → host 模式（最简单，容器直接用宿主机网络栈）
# → macvlan（每个容器独立 MAC + 独立 link-local，最干净的方案）
#   host：简单但端口冲突 + 无隔离
#   macvlan：隔离好但容器和宿主机之间不能直接通信

# 场景3：容器需要和宿主机通信 + 容器间通信
# → ipvlan L2（容器↔宿主机天然互通，比 macvlan 好）
# → bridge（通过 172.17.0.1 网关地址互通）

# 场景4：容器需要同时满足 容器间 + 容器↔宿主机 + 容器→同L2邻居
# → 这种场景下没有完美单一方案
# → 推荐 host 模式（牺牲隔离性换取全部互通能力）
# → 或 macvlan + 额外创建 macvlan-host 接口解决宿主机通信

# 场景5：需要最大隔离
# → none 模式（手动配置网络）

# 场景6：需要跨主机容器通信
# → 仅 link-local 无法实现！
# → 必须先获取可路由地址（ULA fd00::/8 或 global IPv6）
# → 然后才能用 overlay / VXLAN / Calico 等方案
```

### 面试回答模板

```
# Q: 如果宿主机接口只有 link-local IPv6 地址，能用哪些容器网络模式？
#
# 答：
# 首先明确 link-local IPv6（fe80::/10）的限制：不可路由，只能和同 L2 网段邻居通信。
# 所以所有需要 NAT 或路由到外网的能力都不可用。
#
# 但以下模式在特定场景下完全可用：
#
# 1. bridge — 容器间通信完全正常（Docker 内部 IPv4 自给自足），
#    只是容器无法访问外网，也无法被外部访问
#
# 2. host — 最适合同 L2 邻居通信，容器直接用宿主机的 fe80:: 地址，
#    能做 NDP 发现、ping6 邻居、建立 TCP 连接
#
# 3. macvlan — 每个容器有独立 MAC 和独立 link-local 地址，
#    在 L2 网段上表现为独立主机，邻居可以分别发现和寻址每个容器
#
# 4. ipvlan L2 — 容器共享宿主机 MAC，但能和宿主机直接通信
#    （macvlan 做不到），也能和 L2 邻居通信
#
# 5. container — 共享另一个容器的网络，能力取决于父容器
#
# 6. none — 完全隔离，与宿主机地址无关，总是可用
#
# 不可用/严重受限的：
# - overlay 跨主机：需要可路由地址建立 VXLAN 隧道
# - ipvlan L3：禁用 NDP，无法和 L2 邻居通信，也无法路由到外部
#
# 关键洞察：Docker bridge 的内部网络是自给自足的（172.17.0.0/16），
#           不依赖宿主机外部地址，所以容器间通信不受影响。
#           而对外通信全面受限于 link-local 的不可路由性。
```

---

## 第9部分：裸金属 Provisioning 实战 — Link-Local IPv6 容器网络

### 整体场景

```
# 裸金属基础设施管理系统：
# - 管理主机（Management Host）运行容器化服务
# - 新的裸金属节点接入同一 L2 网段
# - 所有通信使用 IPv6 link-local 地址（fe80::）
#
# 典型流程：
#   4.1 mDNS 发现新节点 → 4.2 调用 bootstrap-agent 配置节点 → 4.3 通过 iDRAC 管理硬件
#
# 网络拓扑：
#
#   Management Host                    New Bare-Metal Node
#   ┌────────────────────┐            ┌─────────────────────┐
#   │ ┌─────────────────┐│            │                     │
#   │ │ Discovery 容器   ││            │  bootstrap-agent    │
#   │ │ (mDNS 监听)     ││            │  (API on :8443)     │
#   │ └─────────────────┘│            │                     │
#   │ ┌─────────────────┐│            │  mDNS responder     │
#   │ │ Bootstrap 容器   ││            │  (port 5353)        │
#   │ │ (API 调用)       ││            └──────┬──────────────┘
#   │ └─────────────────┘│                    │ eth0: fe80::node
#   │ ┌─────────────────┐│                    │
#   │ │ Redfish 容器     ││     ┌──────────────┤
#   │ │ (iDRAC 管理)     ││     │              │
#   │ └─────────────────┘│     │         iDRAC port: fe80::idrac
#   │                    │     │
#   │ eth0: fe80::host   ├─────┤  ← 同一 L2 交换机（provisioning 网段）
#   │ eth1: fe80::mgmt   ├─────┘  ← 可能是单独的 management VLAN
#   └────────────────────┘
```

### 9.1 场景 4.1：mDNS 新节点发现

#### mDNS + IPv6 Link-Local 工作原理

```
# mDNS 协议（RFC 6762）：
# - IPv6 组播组：ff02::fb（link-local scope multicast）
# - 端口：UDP 5353
# - 组播范围 ff02:: = 仅本链路，包永远不会被路由器转发
#
# 发现流程：
#
# 1. 管理主机的容器加入 ff02::fb 组播组
# 2. 新节点上线，其 mDNS responder 向 ff02::fb 发布服务公告
#    例如：_bootstrap._tcp.local → fe80::node-addr
# 3. 管理容器收到 mDNS 响应，获得新节点的 link-local 地址
# 4. 内核自动通过 NDP（Neighbor Solicitation）解析 fe80:: → MAC 地址
# 5. 后续 TCP/UDP 通信可以开始
#
# 关键协议链：
#   mDNS（发现 fe80::）→ NDP（解析 MAC）→ TCP/HTTP（业务通信）
#
# mDNS 和 NDP 都是天生的 link-local 协议，不需要任何全局地址
```

#### 容器网络模式选择

```
# 核心需求：
# 1. 容器必须能加入物理网段的 ff02::fb 组播组
# 2. 容器必须能收发物理 L2 段上的 mDNS 包
# 3. 容器必须能看到真实的 fe80:: 地址（不能被 NAT）
#
# ❌ bridge 模式：完全不可用
#    docker0 是独立的 L2 域，和物理网段隔离
#    容器加入 ff02::fb 只在 docker0 段生效，收不到物理段的 mDNS
#
#    Physical:  [New Node] ←→ [eth0] ←→ (物理交换机)
#                                  ↕
#                            (不连通!)
#                                  ↕
#    Docker:   [Container eth0] ←→ [docker0 172.17.0.1]
#
# ✅ host 模式（推荐）：
#    容器直接使用宿主机的 eth0
#    ff02::fb 组播在物理接口上生效
#    NDP 使用宿主机的邻居表
#
# ✅ macvlan 模式：
#    容器有独立 MAC，直接在物理 L2 段
#    能独立加入 ff02::fb
#    多容器可以各自监听 5353 端口（不冲突）
#
# ⚠️ ipvlan L2 模式：
#    在物理 L2 段，但共享 MAC
#    组播行为依赖内核版本，可能导致重复接收
```

#### 推荐配置

```yaml
# docker-compose.yml — mDNS 发现服务
services:
  node-discovery:
    image: provisioning/node-discovery:latest
    network_mode: host          # ← 推荐：直接访问物理接口
    cap_add:
      - NET_RAW                 # ← 必须：mDNS 库需要 raw socket 做组播
    environment:
      - DISCOVERY_IFACE=eth0    # 指定监听的物理接口
      - MDNS_DOMAIN=_bootstrap._tcp.local
    restart: unless-stopped
```

```bash
# 等价 docker run 命令
docker run -d --name node-discovery \
  --network host \
  --cap-add NET_RAW \
  -e DISCOVERY_IFACE=eth0 \
  provisioning/node-discovery:latest
```

```
# 为什么 host 模式最好：
# 1. 零配置 — 容器直接看到 eth0，ff02::fb 在物理接口上
# 2. Zone ID 直接 — %eth0 就是物理网卡，无歧义
# 3. NDP 共享 — 用宿主机邻居表，发现的节点 MAC 自动缓存
#
# 需要注意：
# - 如果宿主机跑了 avahi-daemon，端口 5353 会冲突 → 关掉宿主机的 avahi
# - 需要 ip6tables 放行：ip6tables -A INPUT -p udp --dport 5353 -j ACCEPT
# - 多容器不能同时绑 5353 → 用一个统一的发现服务
```

### 9.2 场景 4.2：调用 Bootstrap-Agent API

#### Link-Local IPv6 HTTP 调用技术细节

```
# 发现了新节点 fe80::node-addr 后，需要调用它的 bootstrap-agent API
# 这是标准的 HTTP/HTTPS unicast 请求
#
# 核心技术挑战：Zone ID（%interface）的处理
#
# link-local 地址需要指定出口接口：
#   fe80::node-addr%eth0   ← 告诉内核从 eth0 发出
#
# 在 URL 中需要百分号编码（RFC 6874）：
#   http://[fe80::node-addr%25eth0]:8443/api/bootstrap
#                          ^^^^ %25 = URL 编码的 %
```

#### 各语言/工具的 Zone ID 支持

```
# 容器内调用 link-local HTTP API 时，各客户端的兼容性：
#
# ┌──────────────────┬────────┬─────────────────────────────────────────┐
# │ 工具/库           │ 支持   │ 语法示例                                 │
# ├──────────────────┼────────┼─────────────────────────────────────────┤
# │ curl             │ ✅ 完整 │ curl -g http://[fe80::1%eth0]:8443/api │
# │                  │        │ curl http://[fe80::1%25eth0]:8443/api   │
# ├──────────────────┼────────┼─────────────────────────────────────────┤
# │ Go net/http      │ ✅ 完整 │ http.Get("http://[fe80::1%25eth0]:8443")│
# │ Go net.Dial      │ ✅ 完整 │ net.Dial("tcp6","[fe80::1%eth0]:8443") │
# ├──────────────────┼────────┼─────────────────────────────────────────┤
# │ Python requests  │ ❌ 不支持│ 会解析错误，% 被当作格式化字符             │
# │ Python urllib3   │ ⚠️ 部分 │ v2.x 改善，仍有 quirk                    │
# │ Python socket    │ ✅ 底层 │ sock.connect(('fe80::1',port,0,scope_id))│
# │                  │        │ scope_id = socket.if_nametoindex('eth0') │
# │ Python pycurl    │ ✅ 完整 │ 底层用 libcurl，和 curl 一样              │
# ├──────────────────┼────────┼─────────────────────────────────────────┤
# │ Node.js http     │ ⚠️ 不稳 │ 某些版本会剥离 zone ID                   │
# └──────────────────┴────────┴─────────────────────────────────────────┘
#
# Python 解决方案（绕过 requests 的限制）：
import socket, ssl, http.client

scope_id = socket.if_nametoindex('eth0')
sock = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
sock.connect(('fe80::node-addr', 8443, 0, scope_id))

ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
conn = ctx.wrap_socket(sock, server_hostname='fe80::node-addr')
# 然后在 conn 上做 HTTP 请求
```

#### 容器内 Zone ID 与接口名映射

```
# Zone ID 中的接口名取决于容器内部看到的接口：
#
# ┌─────────────┬────────────────────────┬───────────────────────┬──────────────┐
# │ 网络模式     │ 容器内 eth0 实际是      │ 能到达物理 L2 段？     │ Zone ID 用法  │
# ├─────────────┼────────────────────────┼───────────────────────┼──────────────┤
# │ host        │ 宿主机的真实 eth0       │ ✅ 直接就是物理接口     │ %eth0 = 物理 │
# │ macvlan     │ macvlan 子接口          │ ✅ 在同一 L2 段        │ %eth0 = 可用 │
# │ ipvlan L2   │ ipvlan 子接口           │ ✅ 在同一 L2 段        │ %eth0 = 可用 │
# │ bridge      │ veth pair 连到 docker0  │ ❌ 隔离的 L2 域        │ %eth0 = 错误!│
# └─────────────┴────────────────────────┴───────────────────────┴──────────────┘
#
# host 模式最简单：容器内 %eth0 就是宿主机的物理 eth0，无歧义
# macvlan/ipvlan：容器内 %eth0 是虚拟子接口，但因为在同一 L2 段，也能工作
# bridge：%eth0 指向 veth，在 docker0 段上，物理段的节点完全不可达
```

#### 推荐配置

```yaml
# docker-compose.yml — Bootstrap Agent 调用服务
services:
  bootstrap-caller:
    image: provisioning/bootstrap-caller:latest
    network_mode: host
    depends_on:
      - node-discovery
    environment:
      - PROVISIONING_IFACE=eth0     # 用于构造 zone ID
      - BOOTSTRAP_PORT=8443
    restart: unless-stopped
```

```bash
# 容器内调用示例（host 模式下）：
curl -g -6 -k http://[fe80::node-addr%eth0]:8443/api/bootstrap \
  -X POST -H "Content-Type: application/json" \
  -d '{"hostname": "node01", "role": "worker"}'
```

### 9.3 场景 4.3：iDRAC Redfish API 访问

#### iDRAC IPv6 地址来源

```
# Dell iDRAC 的 IPv6 地址获取方式：
#
# 1. 自动 Link-Local：iDRAC 管理口启用后自动生成 fe80::（基于 MAC 的 EUI-64）
#    → 始终存在，无需配置
# 2. SLAAC：如果管理网段有 Router Advertisement，自动获取全局 IPv6
# 3. DHCPv6：通过 DHCPv6 服务器分配
# 4. 静态：在 iDRAC Web UI / RACADM / Lifecycle Controller 手动配置
#
# ✅ iDRAC 支持通过 link-local IPv6 访问
# Redfish API 示例：https://[fe80::idrac-addr%25eth1]/redfish/v1/
```

#### 网络拓扑：iDRAC 在哪个网段？

```
# 这是最关键的架构问题 — iDRAC 管理口可能和 provisioning 不在同一 L2 段
#
# === 情况 A：iDRAC 和 provisioning 在同一 L2 段 ===
#
#   Host eth0 ←→ [L2 交换机] ←→ iDRAC port
#   fe80::host              fe80::idrac
#
#   分析同场景 4.2，使用 %eth0 即可
#
# === 情况 B：iDRAC 在单独的管理 VLAN（更常见）===
#
#   Host eth0 ←→ [VLAN 100 - Provisioning]
#   Host eth1 ←→ [VLAN 200 - Management/iDRAC]
#                       ↕
#                 iDRAC port ←→ [VLAN 200]
#
#   link-local 不能跨 VLAN（不同 L2 段）!
#   必须用 eth1（管理接口）的 zone ID：%eth1
#
# === 情况 C：iDRAC 有全局/ULA 地址 ===
#
#   iDRAC: 2001:db8::idrac（全局地址）
#   不需要 zone ID，但宿主机也需要有到该子网的路由
#   如果宿主机只有 fe80::，仍然无法路由到全局地址
#
# → 在只有 link-local 的场景下，情况 A 和 B 是实际可行的
```

#### Host 模式的多接口优势

```
# host 模式下容器能看到宿主机的所有接口：
#   eth0 — provisioning 网段（mDNS 发现、bootstrap-agent）
#   eth1 — management 网段（iDRAC Redfish）
#   bond0/bond1 — 聚合接口
#   VLAN 接口等
#
# 容器可以通过不同的 zone ID 访问不同网段的 iDRAC：
curl -k -g https://[fe80::idrac-addr%eth1]/redfish/v1/Systems/System.Embedded.1
#                                    ^^^^ 管理接口

# 如果 iDRAC 和 provisioning 共用网段：
curl -k -g https://[fe80::idrac-addr%eth0]/redfish/v1/Systems/System.Embedded.1
#                                    ^^^^ provisioning 接口
#
# 对比 macvlan/ipvlan：
# 每个 Docker 网络只能绑定一个父接口（-o parent=eth0 或 eth1）
# 如果 iDRAC 在 eth1、provisioning 在 eth0，需要创建两个独立网络
# 容器要同时加入两个网络才能同时访问 → 配置复杂度大幅增加
```

#### Redfish API 特殊注意事项

```
# 1. TLS 证书问题：
#    iDRAC 的自签名证书 SAN 中通常不包含 fe80:: 地址
#    → 必须跳过证书验证：curl -k / verify=False / InsecureSkipVerify: true
#
# 2. Redfish 客户端库的 Zone ID 支持：
#    ┌─────────────────────┬────────┬───────────────────────────┐
#    │ 库                   │ 语言   │ link-local zone ID 支持    │
#    ├─────────────────────┼────────┼───────────────────────────┤
#    │ python-redfish (DMTF)│ Python │ ❌ 差（底层用 requests）    │
#    │ sushy (OpenStack)    │ Python │ ⚠️ 底层也用 requests       │
#    │ gofish              │ Go     │ ✅ 好（Go net/http 原生支持）│
#    │ curl                │ CLI    │ ✅ 完整支持                 │
#    │ racadm              │ CLI    │ ✅ Dell 原生工具             │
#    └─────────────────────┴────────┴───────────────────────────┘
#
# 3. 常用 Redfish 端点：
#    GET  /redfish/v1/                              # 服务根
#    GET  /redfish/v1/Systems/System.Embedded.1     # 系统信息
#    GET  /redfish/v1/Managers/iDRAC.Embedded.1     # iDRAC 信息
#    POST /redfish/v1/Systems/.../Actions/ComputerSystem.Reset  # 电源控制
#    GET  /redfish/v1/UpdateService/FirmwareInventory  # 固件版本
```

#### 推荐配置

```yaml
# docker-compose.yml — iDRAC Redfish 管理服务
services:
  redfish-manager:
    image: provisioning/redfish-manager:latest
    network_mode: host
    depends_on:
      - node-discovery
    environment:
      - IDRAC_IFACE=eth1            # iDRAC 管理网段接口
      - PROVISIONING_IFACE=eth0     # 备用（如果 iDRAC 共用 provisioning 网段）
      - REDFISH_VERIFY_SSL=false    # 跳过 iDRAC 自签名证书验证
    restart: unless-stopped
```

### 9.4 三个场景统一架构

#### 推荐方案：全部使用 Host 模式

```yaml
# docker-compose.yml — 完整的裸金属 Provisioning 服务栈
version: "3.8"

services:
  # 4.1 mDNS 新节点发现
  node-discovery:
    image: provisioning/node-discovery:latest
    network_mode: host
    cap_add:
      - NET_RAW                   # mDNS 组播需要 raw socket
    environment:
      - DISCOVERY_IFACE=eth0
      - MDNS_DOMAIN=_bootstrap._tcp.local
    restart: unless-stopped

  # 4.2 Bootstrap Agent 配置调用
  bootstrap-caller:
    image: provisioning/bootstrap-caller:latest
    network_mode: host
    depends_on:
      - node-discovery
    environment:
      - PROVISIONING_IFACE=eth0
      - BOOTSTRAP_PORT=8443
    restart: unless-stopped

  # 4.3 iDRAC Redfish 硬件管理
  redfish-manager:
    image: provisioning/redfish-manager:latest
    network_mode: host
    depends_on:
      - node-discovery
    environment:
      - IDRAC_IFACE=eth1          # iDRAC 管理网段
      - PROVISIONING_IFACE=eth0
      - REDFISH_VERIFY_SSL=false
    restart: unless-stopped
```

#### 为什么三个场景都用 Host 模式

```
# ┌───────────────┬────────┬──────────┬────────────┬────────┐
# │ 对比维度       │ host   │ macvlan  │ ipvlan L2  │ bridge │
# ├───────────────┼────────┼──────────┼────────────┼────────┤
# │ mDNS 组播发现  │ ✅     │ ✅       │ ⚠️ 组播quirk│ ❌     │
# │ link-local 单播│ ✅     │ ✅       │ ✅          │ ❌     │
# │ 多接口(eth0+1) │ ✅ 自动 │ ⚠️ 需多网络│ ⚠️ 需多网络  │ ❌     │
# │ Zone ID 简单性 │ ✅ 透明 │ ⚠️ 映射不同│ ⚠️ 映射不同  │ ❌     │
# │ 容器↔宿主机    │ ✅ 天然 │ ❌ 天生限制│ ✅          │ ✅ NAT │
# │ 配置复杂度     │ ✅ 最低 │ ⚠️ 中等   │ ⚠️ 中等     │ ✅ 默认 │
# │ 不需混杂模式   │ ✅     │ ❌ 需要   │ ✅          │ ✅     │
# │ 端口冲突风险   │ ⚠️ 有   │ ✅ 隔离   │ ✅ 隔离     │ ✅ 隔离 │
# └───────────────┴────────┴──────────┴────────────┴────────┘
#
# host 模式的优势：
# 1. 多接口支持 — 容器自动看到所有宿主机接口（eth0 provisioning + eth1 iDRAC）
# 2. Zone ID 透明 — %eth0/%eth1 直接对应物理接口，无虚拟化映射
# 3. 组播无障碍 — ff02::fb 直接在物理接口上，无内核版本兼容性问题
# 4. 配置最简 — 无需创建 Docker network，无需 IPAM dummy subnet
# 5. NDP 共享 — 宿主机和容器共用邻居缓存，mDNS 发现的 MAC 直接可用
#
# host 模式的代价（可接受）：
# - 端口冲突 → provisioning 场景通常是专用主机，端口冲突风险低
# - 无网络隔离 → 裸金属管理场景安全性依赖于网络分段，不依赖容器隔离
```

#### 常见坑和解决方案

```
# 1. avahi-daemon 端口冲突（port 5353）
#    宿主机可能自带 avahi → systemctl disable --now avahi-daemon
#
# 2. ip6tables 防火墙阻断
#    ip6tables -A INPUT -p udp --dport 5353 -j ACCEPT  # mDNS
#    ip6tables -A INPUT -p tcp --dport 8443 -j ACCEPT  # bootstrap-agent
#    ip6tables -A INPUT -p icmpv6 -j ACCEPT            # NDP 必须放行!
#
# 3. NDP 缓存过期/错误
#    节点重装后 MAC 可能变但 fe80:: 缓存了旧 MAC
#    → ip -6 neigh flush dev eth0
#
# 4. Python requests 不支持 Zone ID
#    → 改用 Go / curl / pycurl / 底层 socket
#    → 或写一个 thin wrapper 用 socket + http.client
#
# 5. iDRAC TLS 证书不含 fe80::
#    → 必须 verify_ssl=false（BMC 管理的标准做法）
#
# 6. iDRAC 在不同 VLAN
#    → 确保宿主机有接口在 iDRAC 的 VLAN 上
#    → 用正确的 zone ID（%eth1 而非 %eth0）
#    → host 模式下容器能同时访问两个接口
```

### 面试回答模板

```
# Q: 在只有 link-local IPv6 的裸金属 provisioning 环境中，
#    如何选择容器网络模式来实现 mDNS 发现、API 调用和 iDRAC 管理？
#
# 答：
# 三个场景都推荐 host 网络模式，原因：
#
# 1. mDNS 发现需要物理 L2 段的组播访问：
#    host 模式下容器直接使用物理 eth0，能加入 ff02::fb 组播组，
#    收到同网段新节点的 mDNS 公告，获得其 fe80:: 地址。
#    bridge 模式完全不行 — docker0 和物理段隔离。
#
# 2. bootstrap-agent API 调用需要 link-local 单播：
#    host 模式下 zone ID %eth0 直接对应物理网卡，
#    curl/Go HTTP 客户端能直接用 [fe80::node%eth0] 发送请求。
#    bridge 模式的 veth 在错误的 L2 域，无法到达物理段的节点。
#
# 3. iDRAC Redfish 可能在不同的管理接口：
#    host 模式的容器看到所有宿主机接口（eth0、eth1 等），
#    只需改变 zone ID（%eth1）就能访问不同网段的 iDRAC。
#    macvlan/ipvlan 每个网络只能绑一个父接口，多网段需要多个 Docker network。
#
# 额外注意：
# - mDNS 容器需要 NET_RAW capability
# - iDRAC 自签名证书需跳过 TLS 验证
# - Python requests 库不支持 link-local zone ID，推荐用 Go 或 curl
# - 需要放行 ip6tables 中的 ICMPv6（NDP 依赖它）
```
