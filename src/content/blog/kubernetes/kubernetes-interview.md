---
title: Kubernetes面试问题指南 - 核心概念、调度、网络、Operator与故障排查
description: K8s面试高频问题全覆盖：Pod生命周期、调度机制、Service网络、存储、RBAC、滚动更新、Operator/CRD开发、故障排查
pubDate: '2025-03-30'
categories:
- Kubernetes
tags:
- K8s
- 面试
- Pod
- Service
- 调度
- Operator
- CRD
---
# Kubernetes 面试问题指南

## 第1部分：面试常见问题速览

### 面试高频问题
```
# Q1: Kubernetes 是什么？解决了什么问题？
# 答：
# Kubernetes 是容器编排平台，解决大规模容器的部署、伸缩和管理问题。
# 核心能力：
# - 自动化部署和回滚
# - 服务发现和负载均衡
# - 自动装箱（资源调度）
# - 自我修复（容器故障自动重启/替换）
# - 水平伸缩（HPA）
# - 密钥和配置管理（Secret/ConfigMap）

# Q2: Pod 是什么？为什么不直接运行容器？
# 答：
# Pod 是 K8s 最小调度单元，可包含一个或多个共享网络和存储的容器。
# 原因：
# 1. 多容器协作（sidecar 模式）需要共享 localhost 和 Volume
# 2. Pod 提供统一的生命周期管理
# 3. 解耦容器运行时（支持 containerd、CRI-O 等）

# Q3: Deployment、ReplicaSet、Pod 三者的关系？
# 答：
# Deployment → 管理 → ReplicaSet → 管理 → Pod
# Deployment：负责声明式更新和回滚
# ReplicaSet：负责维持 Pod 副本数
# Pod：实际运行容器的最小单元

# Q4: Service 的类型有哪些？
# 答：
# - ClusterIP（默认）：集群内部访问
# - NodePort：通过节点端口暴露，范围 30000-32767
# - LoadBalancer：云厂商外部负载均衡器
# - ExternalName：映射到外部 DNS 名称（CNAME）

# Q5: ConfigMap 和 Secret 的区别？
# 答：
# ConfigMap：存储非敏感配置，明文存储在 etcd
# Secret：存储敏感数据，base64 编码（非加密！），可启用 etcd 加密
# 两者都可通过环境变量或 Volume 挂载到 Pod

# Q6: StatefulSet 和 Deployment 的区别？
# 答：
# StatefulSet 提供：
# 1. 稳定的网络标识（pod-0, pod-1, pod-2）
# 2. 稳定的持久化存储（每个 Pod 绑定独立 PVC）
# 3. 有序部署和扩缩容（按顺序启动/停止）
# 适用场景：数据库、ZooKeeper、Kafka 等有状态应用

# Q7: DaemonSet 的用途？
# 答：
# 确保每个（或指定）Node 上运行一个 Pod 副本。
# 典型场景：日志收集（Fluentd）、监控代理（Node Exporter）、网络插件（CNI）
```

### 面试准备清单

✅ 能说清 Pod、Deployment、Service 的关系和区别  
✅ 能解释 Pod 的完整生命周期和状态  
✅ 能描述 K8s 调度流程（预选 → 优选 → 绑定）  
✅ 能解释 Service 四种类型及流量路径  
✅ 能说清滚动更新和回滚策略  
✅ 理解 PV/PVC/StorageClass 的关系  
✅ 了解 RBAC 权限模型  
✅ 能用 kubectl 排查常见故障  
✅ 理解 Liveness/Readiness/Startup Probe 的区别  
✅ 了解资源限制（requests/limits）和 QoS 等级  

---

## 第2部分：Pod 生命周期与状态

### Pod 阶段（Phase）

| Phase | 说明 |
|-------|------|
| **Pending** | Pod 已被接受，但容器未就绪（调度中、拉取镜像中） |
| **Running** | 至少一个容器在运行 |
| **Succeeded** | 所有容器成功终止，不会重启 |
| **Failed** | 所有容器已终止，至少一个失败 |
| **Unknown** | 无法获取 Pod 状态（通常是 kubelet 通信问题） |

### Pod 完整生命周期

```
创建 Pod
  ↓
Scheduler 调度到 Node
  ↓
kubelet 拉取镜像（ImagePullPolicy）
  ↓
运行 Init Containers（按顺序，逐个运行）
  ↓
运行主容器
  ├─ 执行 postStart Hook
  ├─ Startup Probe（通过后才开始其他探针）
  ├─ Liveness Probe（失败则重启容器）
  ├─ Readiness Probe（失败则从 Endpoints 移除）
  └─ 容器运行中...
  ↓
终止信号（SIGTERM）
  ├─ 执行 preStop Hook
  ├─ 等待 terminationGracePeriodSeconds（默认 30s）
  └─ 超时发送 SIGKILL 强制终止
```

### 三种探针对比

| 探针 | 作用 | 失败后果 | 典型配置 |
|------|------|---------|---------|
| **Startup Probe** | 检测应用是否启动完成 | 杀死容器并重启 | 慢启动应用（Java） |
| **Liveness Probe** | 检测应用是否存活 | 杀死容器并重启 | 检测死锁/无响应 |
| **Readiness Probe** | 检测应用是否可接收流量 | 从 Service Endpoints 移除 | 应用初始化/依赖未就绪 |

### 探针配置示例
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: app
    image: myapp:v1
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10   # 首次探测延迟
      periodSeconds: 5          # 探测间隔
      failureThreshold: 3       # 连续失败几次才算失败
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 3
```

### Init Container

```
# 特点：
# 1. 在主容器之前按顺序运行
# 2. 每个 Init Container 必须成功退出，下一个才启动
# 3. 如果失败，Pod 会按 restartPolicy 重启
#
# 典型用途：
# - 等待依赖服务就绪（如数据库）
# - 初始化配置文件
# - 下载依赖数据
# - 设置文件权限
```

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z mysql-svc 3306; do sleep 2; done']
  containers:
  - name: app
    image: myapp:v1
```

---

## 第3部分：调度机制

### 调度流程

```
未调度的 Pod（nodeName 为空）
  ↓
① 预选（Filtering）
  │  过滤不满足条件的 Node
  │  - 资源是否充足（CPU/Memory requests）
  │  - nodeSelector / nodeAffinity 是否匹配
  │  - 是否容忍 Node 的 Taint
  │  - PV 所在拓扑是否匹配
  ↓
② 优选（Scoring）
  │  给候选 Node 打分（0-100）
  │  - 资源利用率均衡（LeastRequestedPriority）
  │  - Pod 亲和性（InterPodAffinityPriority）
  │  - 数据本地性（ImageLocalityPriority）
  ↓
③ 绑定（Binding）
  │  选最高分 Node
  │  更新 Pod.Spec.NodeName
  ↓
kubelet 拉起 Pod
```

### nodeSelector vs nodeAffinity

| 特性 | nodeSelector | nodeAffinity |
|------|-------------|--------------|
| **语法** | 简单键值对 | 表达式（In, NotIn, Exists...） |
| **软/硬** | 只有硬约束 | 支持 required（硬）和 preferred（软） |
| **灵活性** | 低 | 高（支持多条件组合） |

```yaml
# nodeSelector（简单）
spec:
  nodeSelector:
    disktype: ssd

# nodeAffinity（灵活）
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # 硬约束
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd", "nvme"]
      preferredDuringSchedulingIgnoredDuringExecution:  # 软约束
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["us-east-1a"]
```

### Taint 和 Toleration

```
# Taint：标记在 Node 上，"排斥" Pod
# Toleration：标记在 Pod 上，"容忍" Node 的 Taint

# Taint 效果（effect）：
# - NoSchedule：不调度新 Pod（已有 Pod 不影响）
# - PreferNoSchedule：尽量不调度
# - NoExecute：不调度 + 驱逐已有 Pod
```

```bash
# 给 Node 打 Taint
kubectl taint nodes node1 key=value:NoSchedule

# 移除 Taint
kubectl taint nodes node1 key=value:NoSchedule-
```

```yaml
# Pod 配置 Toleration
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

### Pod 亲和性与反亲和性

```yaml
# Pod 反亲和性：同一应用的 Pod 分散到不同 Node
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["web"]
        topologyKey: kubernetes.io/hostname
```

**面试常见追问：**
```
# Q: Master 节点为什么默认不调度普通 Pod？
# A: Master 节点有 Taint：node-role.kubernetes.io/master:NoSchedule
#    只有带对应 Toleration 的系统组件才会被调度上去

# Q: 如何让 Pod 调度到 Master？
# A: 给 Pod 加上对应的 Toleration，或者移除 Master 的 Taint（不推荐）
```

---

## 第4部分：Service 与网络

### Service 四种类型详解

```
                   外部用户
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
     LoadBalancer  NodePort   Ingress/Route
          │           │           │
          └─────┬─────┘           │
                ↓                 ↓
           ClusterIP         ClusterIP
                │                 │
           ┌────┼────┐      ┌────┼────┐
           ↓    ↓    ↓      ↓    ↓    ↓
         Pod1 Pod2 Pod3    Pod1 Pod2 Pod3
```

| 类型 | 访问方式 | 使用场景 |
|------|---------|---------|
| **ClusterIP** | `svc-name.ns.svc.cluster.local` | 内部微服务通信 |
| **NodePort** | `<NodeIP>:<30000-32767>` | 开发测试、简单外部访问 |
| **LoadBalancer** | 云厂商分配的外部 IP | 生产环境外部访问 |
| **ExternalName** | CNAME 到外部 DNS | 对接外部服务 |

### Service 流量路径（ClusterIP）

```
Pod A 访问 Service B（my-svc.default.svc.cluster.local）
  ↓
CoreDNS 解析 → ClusterIP（如 10.96.0.10）
  ↓
Pod A 发包，目的 IP = 10.96.0.10
  ↓
kube-proxy 的 iptables/IPVS 规则匹配 ClusterIP
  ↓
DNAT：目的 IP 改为后端 Pod IP（如 10.244.2.8）
  ↓
CNI 路由到目标 Pod
```

### Endpoints 和 EndpointSlice

```
# Service 通过 selector 匹配 Pod，自动维护 Endpoints
# Endpoints 包含所有就绪 Pod 的 IP:Port

kubectl get endpoints my-svc
# NAME     ENDPOINTS                                AGE
# my-svc   10.244.1.5:8080,10.244.2.8:8080         5d

# EndpointSlice（K8s 1.21+ 默认）：
# 将 Endpoints 拆分为多个 Slice，适合大规模集群（>1000 Pod）
```

### Headless Service

```yaml
# ClusterIP: None → 不分配 ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: my-headless-svc
spec:
  clusterIP: None
  selector:
    app: my-app
  ports:
  - port: 80
```

```
# DNS 查询 Headless Service 直接返回 Pod IP 列表
# 而非返回 ClusterIP
#
# 用途：
# - StatefulSet 的稳定网络标识（pod-0.my-headless-svc）
# - 客户端自己做负载均衡
# - 服务发现（获取所有 Pod IP）
```

### DNS 解析规则

```
# Service DNS：
# <service>.<namespace>.svc.cluster.local
# 例：my-svc.default.svc.cluster.local

# StatefulSet Pod DNS：
# <pod-name>.<headless-svc>.<namespace>.svc.cluster.local
# 例：mysql-0.mysql-headless.default.svc.cluster.local

# 短名解析（同 Namespace）：
# 直接用 my-svc 即可访问
```

---

## 第5部分：存储

### PV / PVC / StorageClass 关系

```
管理员 / StorageClass（动态）
       │
       ↓
  PersistentVolume (PV)          ←  实际存储资源
       ↑
       │ 绑定（Bound）
       ↓
  PersistentVolumeClaim (PVC)    ←  用户申请的存储需求
       ↑
       │ 挂载
       ↓
     Pod（volumeMounts）
```

| 概念 | 角色 | 类比 |
|------|------|------|
| **PV** | 集群存储资源 | 物理硬盘 |
| **PVC** | 用户存储申请 | 申请单 |
| **StorageClass** | 动态供应策略 | 自动供应机制 |

### 访问模式

| 模式 | 说明 | 缩写 |
|------|------|------|
| **ReadWriteOnce** | 单 Node 读写 | RWO |
| **ReadOnlyMany** | 多 Node 只读 | ROX |
| **ReadWriteMany** | 多 Node 读写 | RWX |
| **ReadWriteOncePod** | 单 Pod 读写（K8s 1.22+） | RWOP |

### 回收策略

```
# PV 回收策略（persistentVolumeReclaimPolicy）：
# - Retain：保留数据，手动清理（生产推荐）
# - Delete：自动删除 PV 和底层存储
# - Recycle：已废弃（简单 rm -rf）
```

### 动态供应示例

```yaml
# StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer  # 延迟绑定，等 Pod 调度后再创建
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 10Gi
---
# Pod 挂载
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-data
```

---

## 第6部分：滚动更新与回滚

### 滚动更新策略

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 最多多出 1 个 Pod（总数可到 5）
      maxUnavailable: 1    # 最多 1 个 Pod 不可用（最少 3 个可用）
```

### 更新过程

```
初始状态：v1-pod-a  v1-pod-b  v1-pod-c  v1-pod-d

Step 1：创建 v2-pod-e（maxSurge=1，总数=5）
        终止 v1-pod-a（maxUnavailable=1）

Step 2：v1-pod-a 终止完成
        创建 v2-pod-f
        终止 v1-pod-b

Step 3：... 逐步替换 ...

最终：  v2-pod-e  v2-pod-f  v2-pod-g  v2-pod-h
```

### 回滚操作

```bash
# 查看更新历史
kubectl rollout history deployment/my-app

# 回滚到上一版本
kubectl rollout undo deployment/my-app

# 回滚到指定版本
kubectl rollout undo deployment/my-app --to-revision=2

# 查看更新状态
kubectl rollout status deployment/my-app

# 暂停/恢复更新（用于金丝雀发布）
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app
```

### Deployment 更新触发条件

```
# 只有 Pod Template（.spec.template）变更才会触发滚动更新
# 以下操作会触发：
# - 更新镜像版本
# - 修改环境变量
# - 修改资源 limits/requests
# - 修改 Volume 挂载

# 以下操作不会触发：
# - 修改 replicas（只是扩缩容）
# - 修改 strategy
# - 修改 metadata（labels/annotations）
```

---

## 第7部分：RBAC 权限模型

### 核心概念

```
                  ┌──────────────┐
                  │  User/Group  │
                  │  ServiceAccount│
                  └──────┬───────┘
                         │
                   RoleBinding /
               ClusterRoleBinding
                         │
                  ┌──────┴───────┐
                  │ Role /       │
                  │ ClusterRole  │
                  └──────┬───────┘
                         │
                  ┌──────┴───────┐
                  │ Resources    │
                  │ (pods, svc)  │
                  │ Verbs        │
                  │ (get, list)  │
                  └──────────────┘
```

| 概念 | 作用域 | 说明 |
|------|--------|------|
| **Role** | Namespace | 定义某个命名空间内的权限 |
| **ClusterRole** | 集群 | 定义集群范围的权限 |
| **RoleBinding** | Namespace | 将 Role/ClusterRole 绑定到用户 |
| **ClusterRoleBinding** | 集群 | 将 ClusterRole 绑定到用户（集群范围） |

### RBAC 配置示例

```yaml
# Role：允许在 default namespace 读取 Pod
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
# RoleBinding：将 Role 绑定到用户
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount

```yaml
# 创建 ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
---
# Pod 使用 ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: app
    image: myapp:v1
```

```
# ServiceAccount Token 自动挂载到 Pod：
# /var/run/secrets/kubernetes.io/serviceaccount/token
# /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
# /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

---

## 第8部分：资源管理与 QoS

### Requests 和 Limits

```yaml
spec:
  containers:
  - name: app
    resources:
      requests:          # 调度依据：Node 至少有这么多可用资源
        cpu: "250m"      # 0.25 核
        memory: "128Mi"
      limits:            # 运行上限：超过会被限制或 OOMKill
        cpu: "500m"
        memory: "256Mi"
```

### QoS 等级

| QoS | 条件 | 优先级 | OOMKill 顺序 |
|-----|------|--------|-------------|
| **Guaranteed** | 所有容器 requests = limits | 最高 | 最后被杀 |
| **Burstable** | 至少一个容器设了 requests | 中等 | 其次 |
| **BestEffort** | 没有设置 requests/limits | 最低 | 最先被杀 |

```
# 面试要点：
# - requests 用于调度决策，limits 用于运行时约束
# - CPU 超限会被 throttle（限速），不会被杀
# - Memory 超限会被 OOMKill
# - 生产环境建议至少设置 requests
# - Guaranteed QoS 适合关键服务
```

### LimitRange 和 ResourceQuota

```yaml
# LimitRange：限制单个 Pod/Container 的资源范围
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
  - default:          # 默认 limits
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # 默认 requests
      cpu: "100m"
      memory: "128Mi"
    type: Container
---
# ResourceQuota：限制整个 Namespace 的资源总量
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-quota
  namespace: default
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
```

---

## 第9部分：故障排查

### 排查流程

```
Pod 异常
  ↓
kubectl get pods（查看状态）
  ↓
├─ Pending → kubectl describe pod（调度问题：资源不足/亲和性/Taint）
├─ CrashLoopBackOff → kubectl logs（应用崩溃/配置错误）
├─ ImagePullBackOff → 镜像名称/仓库凭证/网络问题
├─ OOMKilled → 增加 memory limits
├─ Evicted → Node 资源不足，Pod 被驱逐
└─ Running 但不正常 → kubectl exec 进入调试
```

### 常用排查命令

```bash
# 查看 Pod 状态和事件
kubectl get pods -o wide
kubectl describe pod <pod-name>

# 查看日志
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>   # 多容器 Pod
kubectl logs <pod-name> --previous             # 上次崩溃的日志

# 进入容器调试
kubectl exec -it <pod-name> -- /bin/sh

# 查看 Node 状态
kubectl describe node <node-name>
kubectl top nodes                              # 资源使用率

# 查看事件
kubectl get events --sort-by='.lastTimestamp'

# 查看资源配置
kubectl get pod <pod-name> -o yaml

# 网络调试
kubectl run debug --image=busybox --rm -it -- sh
# 在 debug Pod 里：
#   nslookup my-svc
#   wget -qO- http://my-svc:8080/healthz
```

### 常见故障与解决

| 状态 | 可能原因 | 排查方法 |
|------|---------|---------|
| **Pending** | 资源不足/无匹配 Node/PVC 未绑定 | `describe pod` 看 Events |
| **CrashLoopBackOff** | 应用启动失败/配置错误/依赖未就绪 | `logs --previous` 看崩溃日志 |
| **ImagePullBackOff** | 镜像不存在/仓库认证失败/网络不通 | 检查 imagePullSecrets |
| **OOMKilled** | 内存超出 limits | 增大 limits 或优化应用内存 |
| **CreateContainerConfigError** | ConfigMap/Secret 不存在 | 检查引用的资源是否存在 |
| **Terminating 卡住** | finalizer 未清理/preStop 阻塞 | 检查 finalizers，必要时 force delete |

### Node NotReady 排查

```bash
# 查看 Node 状态
kubectl describe node <node-name>

# 常见原因：
# 1. kubelet 挂了
systemctl status kubelet
journalctl -u kubelet -f

# 2. 磁盘压力
df -h

# 3. 内存压力
free -h

# 4. PID 压力
# 5. 网络不通（kubelet 无法连接 API Server）
```

---

## 第10部分：高频面试深度问题

### Pod 被删除的完整流程

```
kubectl delete pod my-pod
  ↓
API Server 设置 deletionTimestamp
  ↓
Endpoints Controller 从 Service Endpoints 中移除该 Pod
  ↓（同时）
kubelet 收到删除事件
  ↓
执行 preStop Hook（如果有）
  ↓
发送 SIGTERM 给容器主进程
  ↓
等待 terminationGracePeriodSeconds（默认 30s）
  ↓
超时未退出 → 发送 SIGKILL 强制杀死
  ↓
kubelet 通知 API Server，Pod 对象被删除
```

```
# 面试要点：
# - SIGTERM 和从 Endpoints 移除是并行的！
# - 可能存在"正在处理请求但已从 Endpoints 移除"或"已收到 SIGTERM 但流量还在进来"
# - 解决方案：preStop Hook 里 sleep 几秒，让 Endpoints 更新传播完成
```

### HPA（水平自动扩缩容）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # CPU 使用率超过 70% 就扩容
```

```
# HPA 工作原理：
# 1. Metrics Server 采集 Pod 的 CPU/Memory 使用率
# 2. HPA Controller 每 15s 查询一次 Metrics
# 3. 计算期望副本数：期望 = ceil(当前副本数 × 当前指标值 / 目标值)
# 4. 调整 Deployment 的 replicas
#
# 注意：
# - 必须设置 requests，HPA 才能计算使用率
# - 扩容较快（几秒），缩容较慢（默认 5 分钟稳定期）
```

### NetworkPolicy vs Service vs Ingress

```
# NetworkPolicy：L3/L4 网络策略，控制 Pod 间流量（允许/拒绝）
# Service：L4 负载均衡，将流量分发到后端 Pod
# Ingress：L7 负载均衡，基于 Host/Path 路由 HTTP 流量
#
# 它们的关系：
# NetworkPolicy 决定"能不能通"
# Service 决定"发给谁"
# Ingress 决定"怎么路由"
```

### etcd 备份与恢复

```bash
# 备份
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 恢复
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored
```

```
# 面试要点：
# - etcd 是集群的"大脑"，存储所有状态
# - 定期备份是运维必做的事
# - 恢复后需要重启 etcd 并指向新的 data-dir
# - 多 Master 集群需要在所有节点恢复
```

---

## 第11部分：Operator 与 CRD

### 什么是 Operator？

```
# Operator = CRD（自定义资源） + Controller（自定义控制器）
#
# 核心思想：将运维经验编码到软件中
# - 人类运维员知道"数据库扩容要先加从节点、同步数据、切换流量"
# - Operator 把这套流程写成控制器逻辑，自动执行
#
# 本质：扩展 Kubernetes API，让集群能管理自定义资源
# 就像 Deployment Controller 管理 Pod 一样，
# 你的 Operator 管理你的自定义资源
```

### CRD（Custom Resource Definition）

```yaml
# CRD 定义了一个新的资源类型
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: configmapreloaders.dpcm.dpc.org
spec:
  group: dpcm.dpc.org                    # API Group
  versions:
  - name: v1alpha1                        # API 版本
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              configMapRef:
                type: object
                properties:
                  name:
                    type: string
              deploymentSelector:
                type: object
                properties:
                  matchLabels:
                    type: object
                    additionalProperties:
                      type: string
              debounceSeconds:
                type: integer
                default: 10
          status:
            type: object
            properties:
              lastRestartTime:
                type: string
              observedConfigMapResourceVersion:
                type: string
    subresources:
      status: {}                          # 启用 status 子资源
  scope: Namespaced                       # Namespaced 或 Cluster
  names:
    plural: configmapreloaders            # 复数（API 路径）
    singular: configmapreloader           # 单数
    kind: ConfigMapReloader               # 资源类型名
    shortNames:
    - cmr                                 # kubectl get cmr
```

```
# 安装 CRD 后，就可以创建自定义资源（CR）：
kubectl apply -f crd.yaml
kubectl get configmapreloaders    # 或 kubectl get cmr
```

### CR（Custom Resource）实例

```yaml
apiVersion: dpcm.dpc.org/v1alpha1
kind: ConfigMapReloader
metadata:
  name: my-reloader
  namespace: default
spec:
  configMapRef:
    name: app-config              # 监控的 ConfigMap
  deploymentSelector:
    matchLabels:
      app: my-app                 # 目标 Deployment 的标签
  debounceSeconds: 5              # 防抖：5 秒内多次变更只触发一次重启
```

### Operator 工作原理（Reconciliation Loop）

```
                    ┌─────────────────────┐
                    │     API Server      │
                    │                     │
                    │  CRD 存储在 etcd    │
                    └──────┬──────────────┘
                           │ Watch 事件
                           │ (ADDED/MODIFIED/DELETED)
                           ↓
                    ┌─────────────────────┐
                    │  Operator Controller │
                    │                     │
                    │  1. 从 WorkQueue    │
                    │     取出事件        │
                    │  2. 读取 CR 期望状态 │
                    │  3. 读取实际状态     │
                    │  4. 对比差异         │
                    │  5. 执行调谐         │
                    │  6. 更新 Status     │
                    └─────────────────────┘
                           │
              ┌────────────┼────────────┐
              ↓            ↓            ↓
        创建/更新      重启 Pod     更新配置
        子资源         (滚动更新)    (Secret等)
```

### Reconcile 函数核心逻辑

```go
func (r *ConfigMapReloaderReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := logf.FromContext(ctx)

    // 1. 获取 CR 实例
    var reloader dpcmv1alpha1.ConfigMapReloader
    if err := r.Get(ctx, req.NamespacedName, &reloader); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil   // CR 被删除，无需处理
        }
        return ctrl.Result{}, err
    }

    // 2. 获取关联的 ConfigMap
    var configMap corev1.ConfigMap
    if err := r.Get(ctx, types.NamespacedName{
        Name:      reloader.Spec.ConfigMapRef.Name,
        Namespace: req.Namespace,
    }, &configMap); err != nil {
        return ctrl.Result{}, err
    }

    // 3. 检测 ConfigMap 是否变化
    currentVersion := configMap.ResourceVersion
    if currentVersion == reloader.Status.ObservedConfigMapResourceVersion {
        return ctrl.Result{}, nil   // 未变化，跳过
    }

    // 4. Debounce 机制
    if reloader.Status.PendingRestartUntil != nil {
        if time.Now().Before(reloader.Status.PendingRestartUntil.Time) {
            delay := time.Until(reloader.Status.PendingRestartUntil.Time)
            return ctrl.Result{RequeueAfter: delay}, nil   // 还在防抖期，稍后重试
        }
    }

    // 5. 重启匹配的 Deployment（通过更新 annotation 触发滚动更新）
    var deployments appsv1.DeploymentList
    if err := r.List(ctx, &deployments,
        client.InNamespace(req.Namespace),
        client.MatchingLabels(reloader.Spec.DeploymentSelector.MatchLabels),
    ); err != nil {
        return ctrl.Result{}, err
    }

    for i := range deployments.Items {
        deploy := &deployments.Items[i]
        if deploy.Spec.Template.Annotations == nil {
            deploy.Spec.Template.Annotations = make(map[string]string)
        }
        deploy.Spec.Template.Annotations["dpcm.io/restartedAt"] = time.Now().Format(time.RFC3339)
        if err := r.Update(ctx, deploy); err != nil {
            return ctrl.Result{}, err
        }
        log.Info("Restarted deployment", "name", deploy.Name)
    }

    // 6. 更新 Status
    reloader.Status.ObservedConfigMapResourceVersion = currentVersion
    reloader.Status.LastRestartTime = &metav1.Time{Time: time.Now()}
    if err := r.Status().Update(ctx, &reloader); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```

### Operator 开发框架对比

| 框架 | 语言 | 特点 | 适用场景 |
|------|------|------|---------|
| **Kubebuilder** | Go | K8s 官方推荐，脚手架完善 | 生产级 Operator |
| **Operator SDK** | Go/Ansible/Helm | Red Hat 维护，集成 OLM | OpenShift 生态 |
| **controller-runtime** | Go | Kubebuilder 底层库 | 自定义程度高 |
| **Kopf** | Python | 简单轻量，上手快 | 快速原型/小型 Operator |
| **KUDO** | 声明式 | 无需编码 | 简单状态机 |

### Kubebuilder 项目结构

```
my-operator/
├── api/
│   └── v1alpha1/
│       ├── types.go              # CR 的 Go 结构体定义
│       ├── groupversion_info.go  # API Group/Version 注册
│       └── zz_generated.deepcopy.go  # 自动生成的 DeepCopy
├── cmd/
│   └── main.go                   # 入口，启动 Manager
├── config/
│   ├── crd/                      # 自动生成的 CRD YAML
│   ├── rbac/                     # 自动生成的 RBAC 规则
│   ├── manager/                  # Controller Manager 部署
│   └── samples/                  # CR 示例
├── internal/
│   └── controller/
│       ├── reconciler.go         # Reconcile 核心逻辑
│       └── reconciler_test.go    # 单元测试
├── Dockerfile
├── Makefile                      # make manifests / make install / make run
└── PROJECT                       # Kubebuilder 元数据
```

### Operator RBAC 权限

```go
// 通过注释声明需要的权限，make manifests 自动生成 RBAC YAML

//+kubebuilder:rbac:groups=dpcm.dpc.org,resources=configmapreloaders,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=dpcm.dpc.org,resources=configmapreloaders/status,verbs=get;update;patch
//+kubebuilder:rbac:groups="",resources=configmaps,verbs=get;list;watch
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;update;patch
```

```
# 生成的 RBAC 规则会创建：
# - ServiceAccount（Operator Pod 使用）
# - ClusterRole（定义权限）
# - ClusterRoleBinding（绑定）
#
# 权限最小原则：只申请 Operator 需要操作的资源和动作
```

### Operator 面试高频问题

```
# Q1: Operator 和普通 Controller 的区别？
# A:
# - 普通 Controller 管理 K8s 内置资源（Deployment、Service 等）
# - Operator 管理自定义资源（CRD），封装了特定应用的运维逻辑
# - Operator = CRD + Controller + 领域知识

# Q2: 为什么需要 Operator？Helm 不够吗？
# A:
# Helm 只能做"Day 1"（安装部署）
# Operator 能做"Day 2"（运维管理）：
# - 自动扩缩容
# - 自动备份/恢复
# - 自动故障转移
# - 自动升级/配置更新
# 例：数据库 Operator 可以自动处理主从切换、数据迁移

# Q3: Reconcile 函数的设计原则？
# A:
# 1. 幂等性：多次执行结果一致（不能有副作用累积）
# 2. 声明式：对比期望状态和实际状态，而不是执行命令序列
# 3. 级别触发（Level-Triggered）而非边缘触发（Edge-Triggered）
#    - 不关心"发生了什么事件"
#    - 只关心"当前状态是否和期望一致"
# 4. 错误处理：返回 error 或 RequeueAfter 让框架自动重试

# Q4: 如何保证 Operator 的高可用？
# A:
# - Leader Election：多副本部署，只有 Leader 执行 Reconcile
# - 其他副本 Standby，Leader 挂了自动选举新 Leader
# - Kubebuilder 默认支持，通过 --leader-elect 参数启用
#
#   Manager 启动时：
#   mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
#       LeaderElection:   true,
#       LeaderElectionID: "my-operator-lock",
#   })

# Q5: CRD 版本升级（v1alpha1 → v1beta1 → v1）怎么处理？
# A:
# - Conversion Webhook：在不同版本之间自动转换
# - Storage Version：etcd 中只存储一个版本
# - 流程：
#   1. 新增 v1beta1 版本的 schema
#   2. 实现 Conversion Webhook
#   3. 逐步迁移，废弃旧版本
#   4. 设置新版本为 storage version

# Q6: Operator 如何监听多种资源？
# A:
# 使用 Watches 配置，支持 For / Owns / Watches：
#
#   ctrl.NewControllerManagedBy(mgr).
#       For(&v1alpha1.ConfigMapReloader{}).    // 主资源（触发 Reconcile）
#       Owns(&appsv1.Deployment{}).            // 子资源（Owner Reference）
#       Watches(&corev1.ConfigMap{},           // 额外监听的资源
#           handler.EnqueueRequestForOwner(...)).
#       Complete(r)
#
# - For：主资源变化直接触发 Reconcile
# - Owns：子资源变化通过 OwnerReference 找到父资源触发
# - Watches：自定义映射逻辑

# Q7: Finalizer 的作用？
# A:
# Finalizer 用于在资源删除前执行清理逻辑。
# 流程：
# 1. 创建 CR 时添加 Finalizer 标记
# 2. 用户执行 kubectl delete → K8s 设置 deletionTimestamp（但不真正删除）
# 3. Controller 检测到 deletionTimestamp，执行清理（如删除外部资源）
# 4. 清理完成，移除 Finalizer
# 5. K8s 检测到无 Finalizer，真正删除资源
#
# 典型场景：
# - 删除 CR 时清理关联的外部数据库
# - 删除 CR 时回收云资源（EBS Volume、DNS 记录）
# - 删除 CR 时通知外部系统
```

### Finalizer 实现示例

```go
const finalizerName = "dpcm.dpc.org/finalizer"

func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var cr dpcmv1alpha1.ConfigMapReloader
    if err := r.Get(ctx, req.NamespacedName, &cr); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 资源正在被删除
    if !cr.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(&cr, finalizerName) {
            // 执行清理逻辑
            if err := r.cleanupExternalResources(ctx, &cr); err != nil {
                return ctrl.Result{}, err
            }
            // 移除 Finalizer
            controllerutil.RemoveFinalizer(&cr, finalizerName)
            if err := r.Update(ctx, &cr); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }

    // 确保 Finalizer 存在
    if !controllerutil.ContainsFinalizer(&cr, finalizerName) {
        controllerutil.AddFinalizer(&cr, finalizerName)
        if err := r.Update(ctx, &cr); err != nil {
            return ctrl.Result{}, err
        }
    }

    // 正常 Reconcile 逻辑...
    return ctrl.Result{}, nil
}
```

### Operator 成熟度模型

```
Level 1: Basic Install
  │  自动安装和配置
  ↓
Level 2: Seamless Upgrades
  │  支持版本升级和补丁
  ↓
Level 3: Full Lifecycle
  │  备份、恢复、故障检测
  ↓
Level 4: Deep Insights
  │  监控指标、告警、日志分析
  ↓
Level 5: Auto Pilot
     自动扩缩容、自动调优、自动修复

# 大多数 Operator 在 Level 1-2
# 成熟的数据库 Operator（如 Crunchy PostgreSQL）达到 Level 4-5
```
