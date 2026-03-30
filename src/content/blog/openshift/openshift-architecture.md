---
title: OpenShift架构详解 - OCP vs K8s对比
description: OpenShift架构、OCP与K8s对比、RHCOS、OVN-Kubernetes、Router/Route、OAuth、S2I、Operators
pubDate: '2025-02-01'
categories:
- OpenShift
tags:
- OpenShift
- OCP
- K8s
- 面试
---
# OpenShift 架构面试资料

## 一、OpenShift vs Kubernetes 核心区别

### **OpenShift = Kubernetes + 企业级增强**

| 层级 | Kubernetes | OpenShift 增强价值 |
|------|------------|-------------------|
| **基础设施** | 任意 OS | RHCOS (不可变 OS) |
| **网络入口** | Ingress | Router + Route |
| **身份认证** | 多种方案 | OAuth Server 内置 |
| **镜像构建** | 外部工具 | S2I + BuildConfig |
| **运维管理** | 手动配置 | Operator 自动化 |
| **安全加固** | 可选配置 | 默认安全策略 |

---

## 二、架构分层详解

### **🧱 1. 基础设施层 (Infra/OS)**

#### **RHCOS (Red Hat Enterprise CoreOS)**
- **不可变 OS**：rpm-ostree 原子更新
- **自动升级**：Machine Config Operator 管理
- **安全加固**：SELinux + FIPS + systemd

#### **核心组件**
| 组件 | 作用 |
|------|------|
| **kubelet** | Pod 生命周期管理 |
| **CRI-O** | 容器运行时 (替代 Docker) |
| **OVN-Kubernetes** | SDN 网络实现 |
| **update-ca-trust** | 证书信任管理 |

### **☸️ 2. Kubernetes 核心层**
- **Control Plane**: kube-apiserver, kube-controller-manager, kube-scheduler, etcd
- **Node**: kubelet, CRI-O
- **资源模型**: Pod, Deployment, Service, ConfigMap, Secret

### **🚀 3. OpenShift 增强平台层**

#### **🌐 网络入口：Router + Route**
```
DNS → Router (HAProxy) → Service → Pod
```

**特性**：
- **TLS 终止模式**：edge / passthrough / reencrypt
- **高级路由**：蓝绿部署、权重分配、SNI 支持
- **动态配置**：Router watch Route CRD，自动生成 HAProxy 配置

#### **🔐 身份与权限 (AuthN/AuthZ)**
- **AuthN (认证)**：OAuth Server 内置
  - 支持：LDAP、GitHub、OIDC、OAuth
- **AuthZ (授权)**：Kubernetes RBAC 扩展

#### **📦 镜像与构建**
| 组件 | 功能 |
|------|------|
| **Image Registry** | 内置镜像仓库 |
| **BuildConfig** | 构建流水线配置 |
| **S2I** | Source-to-Image 源码直接构建 |

#### **🧠 Operator Framework**
**核心组件**：
- **CRD**：自定义资源定义
- **Controller**：控制循环逻辑
- **OLM**：Operator 生命周期管理

**职责**：
- ✅ 自动安装
- ✅ 自动升级
- ✅ 自动回滚
- ✅ 依赖管理

**内置 Operator**：
- ingress-operator, network-operator, monitoring-operator
- storage-operator, authentication-operator

#### **📊 可观测性**
- **监控栈**：Prometheus + Alertmanager + Grafana
- **日志栈**：Fluent Bit + Loki/Elasticsearch

#### **🔐 安全增强**
| 能力 | 说明 |
|------|------|
| **SCC** | Security Context Constraints (比 PodSecurity 强) |
| **SELinux** | 强制访问控制 |
| **mTLS** | 内部组件双向认证 |
| **全局 CA Bundle** | 统一证书管理 |

### **🧩 4. 平台能力层 (Day2 运维)**

#### **🛠 Cluster Operators**
```bash
# 查看所有 Cluster Operators
oc get co
```

**关键 Operator**：
- **kube-apiserver-operator**：API Server 管理
- **ingress-operator**：Router 管理
- **network-operator**：SDN 管理
- **machine-config-operator**：节点配置管理
- **authentication-operator**：OAuth Server 管理

#### **⚙ Machine Config Operator (MCO)**
**管理范围**：
- OS 配置
- 内核参数
- 证书管理
- systemd 服务

**实际应用**：
```bash
# additionalTrustBundle 安装到节点
# 就是 MCO 负责的
```

---

## 三、Core Interview Questions

### **Q1: OpenShift vs Kubernetes Advantages?**
- **Security**: Default security policies, SCC, SELinux
- **Operations**: Operator automation, self-healing capabilities
- **Development**: S2I, built-in CI/CD
- **Monitoring**: Complete observability stack

### **Q2: Router vs Ingress Differences?**
| Feature | Ingress | Route |
|---------|---------|-------|
| **Standard** | Kubernetes native | OpenShift specific |
| **Controller** | Needs installation | Built-in HAProxy |
| **TLS** | Basic support | edge/passthrough/reencrypt |
| **Features** | Basic routing | Blue-green, weight, SNI |

### **Q3: OAuth Server Purpose?**
- **Authentication**: LDAP, GitHub, OIDC
- **Token Management**: OAuth Token generation and management
- **Permission Mapping**: Identity → RBAC permissions
- **Relationship**: OAuth Server authentication → API Server authorization

### **Q4: Operator Value?**
- **Automated Operations**: Installation, upgrade, backup, recovery
- **State Management**: Maintain application desired state
- **Lifecycle**: OLM unified management
- **Application Ecosystem**: OperatorHub pre-packaged applications

### **Q5: Machine Config Operator Responsibilities?**
- **Node Configuration**: Unified management of node configurations
- **Certificate Management**: additionalTrustBundle
- **System Updates**: Atomic OS updates
- **Self-Healing**: Automatic repair of configuration drift

---

## MCO 原理详解

### **核心组件Key Resources**
| 组件resource | 作用 purpose |
|------|------|
| **MachineConfig** | 描述节点期望配置 |
| **MachineConfigPool (MCP)** | 按角色分组节点 (master/worker) |
| **MachineConfigController** | 监听变化，分发配置 |
| **Machine Config Daemon (MCD)** | 节点端 agent，应用配置 |

### **工作流程**
1. **定义期望**：创建/修改 MachineConfig
2. **控制循环**：生成 RenderedMachineConfig
3. **滚动更新**：按 MCP 顺序逐节点升级
4. **节点应用**：MCD 拉取配置，修改文件/systemd
5. **原子更新**：rpm-ostree 更新，必要时重启
6. **状态回报**：MCD 回报状态，MCO 判断成功

### **Master vs Worker 重启策略**

#### **共同流程**
```
Cordon → Drain → 下载配置 → 应用配置 → rpm-ostree → Reboot → Uncordon
```

#### **关键差异**
| 特性 | Master | Worker |
|------|--------|--------|
| **升级方式** | 严格串行 | 批量处理 |
| **并发数** | 最多 1 个 | 默认 1 个 (可调) |
| **可用性** | 至少 1 个 master 可用 | 保持足够容量 |
| **maxUnavailable** | 1 | 可调整 |

#### **策略特点**
- **滚动升级**：不会同时多节点重启
- **自愈机制**：失败自动重试/回滚
- **可观察性**：`oc get mcp` 查看进度

---

## 四、实际应用场景

### **1. 多租户环境**
- **Project 隔离**：比 Namespace 更强的隔离
- **网络策略**：默认网络隔离
- **资源配额**：ResourceQuota 限制

### **2. 企业级部署**
- **安全合规**：FIPS、SELinux、SCC
- **身份集成**：LDAP、AD 集成
- **审计日志**：完整的审计追踪

### **3. DevOps 流水线**
- **S2I 构建**：源码直接构建镜像
- **Jenkins 集成**：完整的 CI/CD
- **镜像仓库**：内置 Image Registry

---


## 六、总结

**OpenShift 的核心价值**：
1. **企业级安全**：默认安全策略和合规性
2. **运维自动化**：Operator 驱动的自愈能力
3. **开发友好**：S2I、内置 CI/CD
4. **完整生态**：监控、日志、网络一体化

**面试重点**：
- 理解架构分层和各层职责
- 掌握核心组件的作用和关系
- 熟悉实际应用场景和故障排查
- 能够对比 OpenShift 和 Kubernetes 的差异