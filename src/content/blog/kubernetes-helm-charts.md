---
title: Kubernetes与Helm Chart详解
description: Helm chart结构、Route/Service/Deployment YAML详解、Ingress、PVC、RBAC概念
pubDate: '2025-02-05'
categories:
- Kubernetes
tags:
- K8s
- Helm
- Deployment
- Service
- 面试
---
打包： helm chart

mystic@ubuntu-vdi:/tmp/repo$ tree mychart/
mychart/
├── charts                 # 依赖的子 Chart（vendor）
├── Chart.yaml             # Chart 元数据：名称、版本、依赖等
├── templates              # 可渲染成 K8s 清单的模板
│   ├── deployment.yaml        # 应用 Deployment
│   ├── _helpers.tpl           # 公共模板函数/片段
│   ├── hpa.yaml               # Pod 水平自动扩缩容
│   ├── httproute.yaml         # Gateway API 的 HTTPRoute
│   ├── ingress.yaml           # 传统 Ingress 资源
│   ├── NOTES.txt              # helm install 完成后的提示信息
│   ├── serviceaccount.yaml    # Pod 绑定的 ServiceAccount
│   ├── service.yaml           # 对外暴露 Pod 的 Service
│   └── tests
│       └── test-connection.yaml   # helm test 用的连通性校验
└── values.yaml             # 模板默认值，可被 --set/-f 覆盖

**Chart.yaml**
```yaml
apiVersion: v2
name: mcp-operator-installer-ocp
type: application
version: 0.0.0
appVersion: "0.0.0"
```

service 使用cluster IP

deployment 里面定义了service account，nodeSelector，tolerations，container，livenessProbe，ports，resources，volumeMounts
PVC里定义了storageClassName，accessModes，resources(多大的volume)


## Route (OpenShift)
```yaml
{{- if and (eq .Values.global.runningEnv "OpenShift") (eq .Release.IsInstall true)}}
apiVersion: route.openshift.io/v1
kind: Route
metadata:
    name: route-mcp-day1-bringup-ocp
    annotations:
        haproxy.router.openshift.io/timeout: 1200s
spec:
    host: day1.apps.{{ .Values.global.ocp }}  #外部访问域名
    path: "/day1"   #HTTP 路径 /day1，Route 仅匹配该路径。
    port:
        targetPort: 5000
    tls:
        termination: edge  # edge | passthrough | reencrypt
    to:
        kind: Service
        name: {{ .Values.serviceName }}
{{- end }}
```

### TLS Termination 三种模式
- **edge**: 在 Router 处终止 TLS，Pod 收到的是明文 HTTP
- **passthrough**: 直接转发原始 TLS 流量，Pod 自己处理 TLS
- **reencrypt**: 在 Router 处终止 TLS，然后重新加密转发到后端

### 流量链路详解（OpenShift 4.x + OVN-Kubernetes）

```
用户浏览器
  │
  │ DNS 解析：*.apps.ocp-cluster → Ingress VIP
  ↓
① Ingress VIP（Keepalived）
  │  VIP 同一时刻只绑在一个 Node 的网卡上
  │  交换机查 ARP 表 → 转发到持有 VIP 的 Node
  │  Node 故障 → VIP 秒级漂移（Gratuitous ARP 通告交换机）
  ↓
② Router Pod（HAProxy, hostNetwork: true）
  │  HAProxy 直接 listen 在宿主机 443 端口
  │  不经过 NodePort / iptables DNAT
  │
  │  - TLS 终止（edge termination）
  │  - 解析 HTTP，匹配 Route 规则（Host + Path）
  │  - 从 Endpoints 拿到后端 Pod IP 列表（watch API Server）
  │  - 负载均衡选一个 Pod
  ↓
③ 直连 Pod IP（不经过 Service ClusterIP）
  │  目标地址是 Pod IP，不是 ClusterIP
  │  数据包经过内核 netfilter 链（必经之路）
  │  但 Service DNAT 规则不命中（目标不是 ClusterIP）→ 直接放行
  ↓
④ OVN-Kubernetes（CNI）路由到目标 Pod
  │  ├─ 同 Node → OVS bridge 直达 Pod
  │  └─ 跨 Node → Geneve 隧道封装 → 目标 Node → 拆封 → Pod
  ↓
⑤ Pod 容器进程处理请求
```

### VIP 与 Keepalived

```
NodeA (Master)                NodeB (Backup)               NodeC (Backup)
┌──────────────┐             ┌──────────────┐             ┌──────────────┐
│ Keepalived   │             │ Keepalived   │             │ Keepalived   │
│ 优先级: 100  │  ← VRRP →  │ 优先级: 90   │  ← VRRP →  │ 优先级: 80   │
│ VIP: ✅      │   心跳组播   │ VIP: ❌      │   心跳组播   │ VIP: ❌      │
│ 192.168.1.100│             │              │             │              │
│ Router Pod   │             │ Router Pod   │             │ Router Pod   │
└──────────────┘             └──────────────┘             └──────────────┘

# 在 Master Node 上可以看到 VIP：
ip addr show eth0
# inet 192.168.1.10/24        ← Node 自己的 IP
# inet 192.168.1.100/32       ← VIP，漂在这里
```

### Router Pod 接收流量

router pod 使用 hostNetwork，任何发到 node 80/443 端口的流量直接进入 pod，pod 里运行 haproxy 进程，HAProxy frontend 监听 80/443，根据 Route ACL 匹配域名，将请求转发到对应 backend：

```sh
bash-5.1$ cat /var/lib/haproxy/conf/haproxy.config  | grep installer
backend be_http:dell-acp:mcp-operator-installer-ocp
  server pod:mcp-operator-installer-ocp-6657db6c9-csjxf:mcp-operator-installer-ocp::172.28.4.18:5000 172.28.4.18:5000 cookie 7e1391b909c98d9d45c4081d90e6b72e weight 1
```

backend 配置里直接包含 Pod IP:Port，HAProxy 负载均衡到可用 Pod，health check 确保 Pod 可用。

### Service 在 Ingress 链路中的角色

```
Service 做了什么：   通过 selector 维护 Pod IP 列表（Endpoints）
Service 没做什么：   不转发流量，ClusterIP 没有被访问

流量路径：HAProxy → 直连 Pod IP（不经过 ClusterIP / iptables DNAT）
Service 只是数据源：告诉 HAProxy "哪些 Pod 是健康的后端"
```

对比直接访问 Service（不经过 Ingress）：
```
无 Ingress：Client → ClusterIP → iptables/OVN DNAT → Pod IP（Service 参与转发）
有 Ingress：Client → HAProxy → Pod IP（Service 只提供 Endpoints 列表）
```

### HAProxy vs Nginx

| | HAProxy | Nginx |
|--|---------|-------|
| **定位** | 专业负载均衡器/代理 | Web 服务器 + 反向代理 + 负载均衡 |
| **能力** | 只做代理和负载均衡 | 还能托管静态文件、做 Web 服务器 |
| **用在哪** | OpenShift Router Pod | K8s Nginx Ingress Controller |
| **本质** | 都是 L7 反向代理：TLS 终止 + Host/Path 匹配 + 转发到后端 Pod |

OpenShift 选了 HAProxy，K8s 社区默认选了 Nginx，在 Ingress 这条链路里干的活完全一样。

### 组件说明
- **router pod**: Router Pod 本质上是 HAProxy + Controller 逻辑, Controller 会 watch Route CRD 和 Service/Endpoints 的变化, 自动生成 HAProxy 配置（动态 reload）
- **HAProxy**: OpenShift Router 内部组件，运行在 `openshift-ingress` namespace 的 Pod 中
- **OVN-Kubernetes**: OpenShift 4.x 默认 CNI，Service 层负载均衡由 OVN Load Balancer 实现（不依赖 kube-proxy 的 iptables/IPVS）

### L4 vs L7 负载均衡

| 特性 | L4（传输层） | L7（应用层） |
|------|-------------|-------------|
| **协议** | TCP/UDP | HTTP/HTTPS/gRPC |
| **路由依据** | IP:Port | URL/Host/Header |
| **能看到的信息** | 源/目标 IP、端口 | HTTP Method、URL、Cookie、Header |
| **TLS 处理** | 透传或终止 | 可终止+重加密 |
| **性能** | 高（简单转发） | 较低（需解析协议） |
| **功能** | 基础负载均衡 | 智能路由、重写、限流、A/B测试 |
| **典型组件** | LVS、IPVS、OVN LB | Nginx、HAProxy、Envoy |
| **本环境** | OVN Load Balancer | HAProxy (Router) |

**一句话总结**：
- L4 只看 IP 和端口，快但"笨"
- L7 看 HTTP 内容，慢但"聪明"（能做复杂路由）

**本环境架构**：HAProxy 做 L7 路由（匹配 host + path），OVN 做 L4 转发（Service ClusterIP 到 Pod IP）


## Kubernetes 核心组件

### Control Plane（控制平面）组件

#### 1. **kube-apiserver**
- **作用**：集群的统一入口，所有操作都通过 API Server
- **功能**：
  - 提供 RESTful API 接口
  - 认证、授权、准入控制
  - 数据校验和持久化到 etcd
  - 与其他组件通信的中枢
- **特点**：无状态，可水平扩展

#### 2. **etcd**
- **作用**：分布式 KV 存储，集群的"数据库"
- **存储内容**：
  - 所有资源对象的状态，such as Pod、Service、Deployment、ConfigMap、Secret、PV/PVC
  - 集群配置信息，such as RBAC 权限、ServiceAccount、ResourceQuota、Admission Webhooks
  - 网络信息，such as Service Endpoints、Ingress/Route 规则、NetworkPolicy、Pod IP 分配
- **特点**：强一致性（Raft 协议），支持 watch 机制

#### 3. **kube-scheduler**
- **作用**：负责 Pod 调度，决定 Pod 运行在哪个 Node 上
- **调度流程**：
  1. **预选（Predicate）**：过滤不满足条件的 Node（资源、亲和性、污点容忍等）
  2. **优选（Priority）**：给候选 Node 打分（资源利用率、数据本地性等）
  3. **绑定（Bind）**：将 Pod 绑定到得分最高的 Node
- **可扩展**：支持自定义调度器

#### 4. **kube-controller-manager**
- **作用**：运行各种控制器，维护集群期望状态
- **常见控制器**：
  - **Deployment Controller**：管理 ReplicaSet 和滚动更新
  - **ReplicaSet Controller**：确保 Pod 副本数符合预期
  - **Node Controller**：监控 Node 健康状态
  - **Service Controller**：为 LoadBalancer 类型 Service 创建云厂商 LB
  - **Endpoints Controller**：维护 Service 和 Pod 的映射关系
  - **Namespace Controller**：管理命名空间生命周期
- **工作模式**：Watch-List 机制，监听 API Server 事件并调谐

#### 5. **cloud-controller-manager**（可选）
- **作用**：与云平台交互（AWS、Azure、GCP 等）
- **功能**：管理云资源（LoadBalancer、Volume、Node 等）

---

### Node（工作节点）组件

#### 1. **kubelet**
- **作用**：Node 上的"代理"，负责 Pod 生命周期管理
- **功能**：
  - 接收 API Server 分配的 PodSpec
  - 通过 CRI（Container Runtime Interface）调用container runtime（containerd、CRI-O）
  - 监控 Pod 和容器健康状态（liveness/readiness probe）
  - 上报 Node 和 Pod 状态到 API Server
  - 管理 Volume 挂载
- **特点**：每个 Node 必须运行一个 kubelet

#### 2. **kube-proxy**
- **作用**：实现 Service 的网络代理和负载均衡
- **工作模式**：
  - **iptables 模式**（默认）：通过 iptables 规则转发流量
  - **IPVS 模式**：性能更好，支持更多负载均衡算法
  - **userspace 模式**（已废弃）：在用户空间代理流量
- **功能**：
  - 监听 Service 和 Endpoints 变化
  - 维护 Service ClusterIP 到 Pod IP 的映射
  - 实现 NodePort、LoadBalancer 等 Service 类型
- **注意**：OpenShift 4.x + OVN-Kubernetes 不使用 kube-proxy，由 OVN Load Balancer 替代

#### 3. **Container Runtime**
- **作用**：真正运行容器的组件, 镜像管理、容器创建/运行/删除
- **常见实现**：
  - **containerd**（推荐，CNCF 项目）
  - **CRI-O**（专为 K8s 设计，OpenShift 默认）
  - **Docker**（已废弃，K8s 1.24+ 移除 dockershim）
- **接口**：通过 CRI（Container Runtime Interface）与 kubelet 通信

---

### 网络组件（CNI）

#### **CNI（Container Network Interface）插件**
- **作用**：实现 Pod 网络和网络策略
- **常见实现**：
  - **Calico**：支持网络策略，BGP 路由
  - **Flannel**：简单易用，VXLAN/host-gw 模式
  - **Cilium**：基于 eBPF，高性能
  - **OVN-Kubernetes**：OpenShift 默认，基于 OVS/OVN
  - **Weave Net**：自动网络发现

---

### 附加组件（Add-ons）

#### 1. **CoreDNS**
- **作用**：集群内 DNS 服务
- **功能**：Service 名称解析（`<service>.<namespace>.svc.cluster.local`）

#### 2. **Ingress Controller**
- **作用**：L7 负载均衡和路由
- **常见实现**：Nginx Ingress、HAProxy（OpenShift Router）、Traefik、Istio Gateway

#### 3. **Metrics Server**
- **作用**：收集资源使用指标（CPU、内存）
- **用途**：支持 `kubectl top`、HPA（水平自动扩缩容）

#### 4. **Dashboard**
- **作用**：Web UI 管理界面

---

### 组件交互流程示例

**创建 Deployment 的完整流程**：
```
1. kubectl apply -> API Server
2. API Server 认证/授权/准入控制 -> 写入 etcd
3. Deployment Controller 监听到事件 -> 创建 ReplicaSet
4. ReplicaSet Controller 监听到事件 -> 创建 Pod
5. Scheduler 监听到未调度的 Pod -> 选择 Node -> 绑定
6. 目标 Node 的 kubelet 监听到 Pod 分配 -> 调用 CRI 创建容器
7. kube-proxy/CNI 配置网络规则
8. Pod 运行，kubelet 持续上报状态
```

#### **Complete Call Chain for Creating Deployment**

```
kubectl apply
    ↓ (HTTP POST /apis/apps/v1/namespaces/default/deployments)
API Server
    ↓ (Authentication/Authorization/Admission Control)
etcd (write Deployment)
    ↓ (push events)
Deployment Controller
    ↓ (create ReplicaSet)
API Server
    ↓ (write ReplicaSet)
etcd
    ↓ (push events)
ReplicaSet Controller
    ↓ (create Pod)
API Server
    ↓ (write Pod, unscheduled)
etcd
    ↓ (push events)
Scheduler
    ↓ (select Node, bind Pod)
API Server
    ↓ (update Pod.Spec.NodeName)
etcd
    ↓ (push events)
kubelet (target Node)
    ↓ (call CRI)
Container Runtime
    ↓ (create container)
```

#### **Event Flow Timeline**

```
T0: kubectl apply → API Server
T1: API Server → etcd (write Deployment)
T2: API Server → Deployment Controller (push event)
T3: Deployment Controller → API Server (create ReplicaSet)
T4: API Server → etcd (write ReplicaSet)
T5: API Server → ReplicaSet Controller (push event)
T6: ReplicaSet Controller → API Server (create Pods)
T7: API Server → etcd (write Pods)
T8: API Server → Scheduler (push event)
T9: Scheduler → API Server (bind Pod to Node)
T10: API Server → etcd (update Pod)
T11: API Server → kubelet (push event)
T12: kubelet → Container Runtime (create container)
```

#### **Component Communication Patterns**

**Watch-Based Communication**
```
etcd ←→ API Server ←→ Controllers
  ↓          ↓           ↓
Storage   Gateway    Consumers
```

**HTTP/gRPC Calls**
```
kubectl → API Server (HTTP REST)
kubelet → Container Runtime (gRPC CRI)
```

**Event Types**
```
ADDED: Resource created
MODIFIED: Resource updated
DELETED: Resource deleted
```

---

### 面试高频问题

1. **API Server 为什么是无状态的？**
   - 所有状态存在 etcd，API Server 只负责处理请求和转发

2. **Controller 的工作原理？**
   - Watch-List 机制 + 调谐循环（Reconciliation Loop）：不断对比期望状态和实际状态
   
   **详细机制**：
   - **订阅资源**：Controller 启动时主动向 API Server 订阅（Watch）关注的资源
   - **事件推送**：API Server 写入 etcd 后立即推送事件给订阅者（毫秒级延迟）
   - **处理流程**：
     ```
     1. Controller 启动 -> List 全量数据 -> 建立本地缓存
     2. 发起 Watch 请求（长连接）
     3. API Server 推送事件（ADDED/MODIFIED/DELETED）
     4. 事件放入 WorkQueue
     5. Worker 执行 Reconcile（调谐）
     ```
   - **订阅示例**：
     - Deployment Controller 订阅：Deployment, ReplicaSet, Pod
     - ReplicaSet Controller 订阅：ReplicaSet, Pod
     - Endpoints Controller 订阅：Service, Pod
   - **时序关系**：
     ```
     kubectl apply -> API Server
       ↓
     认证/授权/准入控制
       ↓
     写入 etcd（持久化）
       ↓
     API Server 推送事件给 Controller（异步，几乎同时）
       ↓
     Controller 收到事件，执行调谐
     ```
   - **可靠性保证**：
     - Watch 断开自动重连
     - 重连时带 ResourceVersion，补齐中间事件
     - Leader Election 保证只有一个实例工作

3. **Scheduler 如何选择 Node？**
   - predicate（过滤）+ priority（打分）+ bind

4. **kubelet 和 kube-proxy 的区别？**
   - kubelet 管理 Pod 生命周期，kube-proxy 管理 Service 网络

5. **etcd 挂了会怎样？**
   - 集群只读，无法创建/更新/删除资源，但已运行的 Pod 不受影响

面试常见追问
Q: ImagePullPolicy 是谁控制的？

kubelet 根据 PodSpec 中的 imagePullPolicy（Always/IfNotPresent/Never）决定是否调用 CRI 拉取
Q: 镜像拉取失败怎么办？

Container Runtime 返回错误给 kubelet
kubelet 重试（有退避策略）
Pod 状态变为 ImagePullBackOff 或 ErrImagePull
Q: 私有镜像仓库的认证信息谁处理？

kubelet 从 Secret（imagePullSecrets）读取认证信息
通过 CRI 传递给 Container Runtime
Container Runtime 使用这些凭证连接私有仓库
一句话总结
kubelet 是"指挥官"（决定拉什么镜像），Container Runtime 是"执行者"（真正去拉镜像）。

---

## Deployment vs ReplicaSet 总结

### **核心区别**
Two-layer abstraction with clear responsibilities:
- Deployment layer: Manages application versions and rollout strategies
- ReplicaSet layer: Ensures desired replica count and pod health
| 特性 | Deployment | ReplicaSet |
|------|------------|------------|
| **主要职责** | 应用版本管理 | 副本数量管理 |
| **滚动更新** | ✅ 支持 | ❌ 不支持 |
| **回滚** | ✅ 支持 | ❌ 不支持 |
| **版本历史** | ✅ 记录多个版本 | ❌ 单一版本 |

---

## Pod 网络通信详解

### 1. Pod 到 Node 的网络路径
```
Pod eth0 → veth pair → 网桥 → 宿主机路由 → 物理网卡
```
- **Pod 网络命名空间**：每个 Pod 有独立 netns，包含自己的 eth0 和路由表
- **veth pair**：Pod eth0 在宿主机对应一个 veth 设备，形成点对点管道
- **网桥**：宿主机上的网桥（如 cbr0/docker0）连接所有 veth，工作在二层
- **路由决策**：网桥收到包后，根据目标 IP 决定转发（同节点另一 Pod）或交给宿主机路由
- **物理网卡**：最终通过物理网卡离开节点，进入集群网络

### 2. 跨节点通信（CIDR 分配 + 路由/封装）
- **CIDR 分配**：每个 Node 分配独立 Pod CIDR（如 10.244.1.0/24、10.244.2.0/24）
- **路由表建立**：CNI 在每个节点写入路由，格式如 `10.244.2.0/24 via <NodeB-IP>`
- **Overlay 模式**（如 Flannel VXLAN）：
  - 封装原始包到外层 UDP，目标为对端 Node IP
  - 对端 Node 解封装后转发给目标 Pod
- **路由模式**（如 Calico BGP）：
  - 通过 BGP 同步 Pod CIDR 路由，直接路由到目标 Node

### 3. Service 访问流程
```
客户端 Pod → DNS 解析 → Service ClusterIP → kube-proxy/iptables → DNAT → 目标 Pod IP
```
- **CoreDNS**：通过 watch API Server，实时维护 Service 名称 → ClusterIP 映射， pod把包发给clusterIP
- **kube-proxy**：在每个节点上安装 iptables/ipvs 规则，实现 ClusterIP → Pod IP 的 DNAT。通过iptables/ipvs 选择一个pod。先确定的是pod ip。具体pod在哪个node上是CNI 网络层的事情。
- **负载均衡**：根据策略（默认轮询）选择后端 Pod

### 4. NAT 出现的位置
- **SNAT**：Pod 访问集群外部时，源 IP 被转换为节点物理网卡 IP
- **DNAT**：访问 Service ClusterIP 时，目的 IP 被转换为后端 Pod IP
- **NodePort**：外部访问 NodeIP:NodePort 时，流量 DNAT 到后端 Pod
- **纯 Pod-to-Pod**：同节点或跨节点 Pod 通信通常不涉及 NAT

### 5. 网桥 MAC 表
- **学习机制**：网桥监听流经的帧，记录源 MAC 与进入端口的对应关系
- **转发决策**：根据 MAC 表进行点对点转发，避免广播
- **ARP 处理**：转发 ARP 请求/响应，让通信双方学习彼此的 MAC 地址

### 6. 完整数据流示例
```
Pod A (10.244.1.5) → Service B (10.96.0.10) → Pod B (10.244.2.8)
```
1. Pod A 发起请求，DNS 解析 Service B 的 ClusterIP
2. 包到达 Node A 的网桥，宿主机路由匹配 ClusterIP
3. kube-proxy 规则将 ClusterIP DNAT 到 Pod B IP
4. 宿主机路由判断目标 Pod B 在其他节点
5. CNI 封装或路由将包发送到 Node B
6. Node B 解封装/路由，将包转发给 Pod B
7. Pod B 处理请求，回包沿原路径返回

## Network Policy 总结

### **核心概念**
Network Policy 是 Kubernetes 的网络策略，用于控制 Pod 之间的网络流量。

### **流量方向**
| 方向 | 说明 | 示例 |
|------|------|------|
| **Ingress** | 入站流量 | 其他 Pod 访问当前 Pod |
| **Egress** | 出站流量 | 当前 Pod 访问其他 Pod |
| **Both** | 双向控制 | 同时控制入站和出站 |

### **基本示例**

#### **1. 允许特定 Pod 访问**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

#### **2. 默认拒绝所有**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}  # 选择所有 Pod
  policyTypes:
  - Ingress
  - Egress
```

### **工作原理**
```
Network Policy Controller
    ↓
监听 NetworkPolicy 变化
    ↓
转换为网络规则
    ↓
推送到 CNI 插件
    ↓
CNI 插件实现流量控制
```

### **CNI 插件支持**
| CNI 插件 | Network Policy 支持 |
|----------|-------------------|
| **Calico** | ✅ 完整支持 |
| **Flannel** | ❌ 不支持 |
| **Weave Net** | ✅ 支持 |
| **Cilium** | ✅ 支持 |
| **OVN-Kubernetes** | ✅ 支持 |

### **面试要点**
- **默认行为**：默认允许所有流量
- **工作原理**：通过 CNI 插件实现流量控制
- **选择器**：基于 Pod 标签和命名空间
- **限制范围**：只限制 Pod 间流量，不影响 Service

### **最佳实践**
1. **默认拒绝**：创建默认拒绝策略
2. **最小权限**：只允许必要的流量
3. **命名空间隔离**：按命名空间控制访问
4. **定期审计**：检查策略配置

