---
title: OpenShift Agent-Based安装 - Systemd启动序列分析
description: OCP Agent-Based安装的systemd服务启动序列：网络配置、assisted service、集群注册
pubDate: '2025-02-02'
categories:
- OpenShift
tags:
- OpenShift
- 安装
- Systemd
- Agent
---
# OpenShift Agent-based Installer - Systemd 服务启动顺序分析

> Keywords: Agent-based installation, Assisted Service, systemd services, network configuration, node0, cluster registration

---

## 1. 概述

本文档详细分析 OpenShift Agent-based Installer 的 systemd 服务启动顺序，特别是网络配置和集群注册流程。

---

## 2. 完整的服务启动顺序图

### 2.1 Phase 0: 早期启动 (Pre-Network)

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 0: 早期启动 (Pre-Network)                                      │
├─────────────────────────────────────────────────────────────────────┤
│ 1. pre-network-manager-config.service                               │
│    ├─ Before: NetworkManager.service                                │
│    ├─ ExecStart: /usr/local/bin/pre-network-manager-config.sh      │
│                │
│    └─ 功能: ⭐ 准备网络配置文件（解析 agent-config.yaml）            │
│                                                                      │
│ 2. NetworkManager.service (系统服务)                                 │
│    └─ 应用网络配置，启动网络接口                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Phase 1: 网络就绪后 (Post-Network)

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 1: 网络就绪后 (Post-Network)                                   │
├─────────────────────────────────────────────────────────────────────┤
│ 3. set-hostname.service                                             │
│    ├─ After: local-fs.target                                        │
│    ├─ Wants: network-online.target                                  │
│    ├─ ExecStart: /usr/local/bin/set-hostname.sh                    │
│    └─ 功能: 设置主机名                                               │
│                                                                      │
│ 4. assisted-service-db.service                                      │
│    └─ 功能: 启动 Assisted Service 数据库容器                         │
│                                                                      │
│ 5. assisted-service-pod.service                                     │
│    └─ 功能: 创建 Assisted Service Pod                                │
│                                                                      │
│ 6. assisted-service.service                                         │
│    ├─ After: network-online.target, assisted-service-pod.service   │
│    ├─ Requires: assisted-service-db.service                         │
│    └─ 功能: 启动 Assisted Service 主服务                             │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Phase 2: 集群注册 (Cluster Registration) - Node0 专属

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 2: 集群注册 (Cluster Registration) - 仅 Node0 执行             │
├─────────────────────────────────────────────────────────────────────┤
│ 7a. agent-register-cluster.service (新集群安装)                      │
│     ├─ After: network-online.target, assisted-service.service      │
│     ├─ Conflicts: agent-import-cluster.service                     │
│     ├─ Condition: /etc/assisted/node0 存在                          │
│     ├─ Condition: /etc/assisted/add-nodes.env 不存在                │
│     ├─ Condition: /etc/assisted/interactive-ui 不存在                │
│     ├─ ExecStart: podman run ... agent-installer-client registerCluster│
│     └─ 功能: 注册新集群到 Assisted Service                           │
│                                                                      │
│ 7b. agent-import-cluster.service (添加节点场景)                      │
│     ├─ After: network-online.target, assisted-service.service      │
│     ├─ Conflicts: agent-register-cluster.service                   │
│     ├─ Condition: /etc/assisted/node0 存在                          │
│     ├─ Condition: /etc/assisted/add-nodes.env 存在                  │
│     ├─ ExecStart: podman run ... agent-installer-client importCluster│
│     └─ 功能: 导入已存在的集群                                        │
│                                                                      │
│ 8. agent-register-infraenv.service                                  │
│    ├─ After: agent-register-cluster.service OR                     │
│    │         agent-import-cluster.service                           │
│    ├─ Condition: /etc/assisted/node0 存在                          │
│    ├─ ExecStart: podman run ... agent-installer-client registerInfraEnv│
│    └─ 功能: 注册基础设施环境                                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.4 Phase 3: 主机配置 (Host Configuration) - Node0 专属

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 3: 主机配置 (Host Configuration) - 仅 Node0 执行               │
├─────────────────────────────────────────────────────────────────────┤
│ 9. apply-host-config.service                                        │
│    ├─ After: network-online.target, agent-register-infraenv.service│
│    ├─ Requires: agent-register-infraenv.service                    │
│    ├─ Condition: /etc/assisted/node0 存在                          │
│    ├─ Condition: /etc/assisted/interactive-ui 不存在                │
│    ├─ ExecStart: podman run ... agent-installer-client configure   │
│    └─ 功能: ⭐ 应用主机特定配置（包括网络、存储等）                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.5 Phase 4: 集群安装 (Cluster Installation)

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 4: 集群安装 (Cluster Installation)                             │
├─────────────────────────────────────────────────────────────────────┤
│ 10. start-cluster-installation.service                              │
│     ├─ After: apply-host-config.service                             │
│     ├─ Requires: apply-host-config.service                          │
│     ├─ Condition: /etc/assisted/node0 存在                          │
│     ├─ Condition: /etc/assisted/add-nodes.env 不存在                │
│     ├─ ExecStart: /usr/local/bin/start-cluster-installation.sh     │
│     └─ 功能: 启动集群安装流程         
向 Assisted Service 发送 POST /clusters/{id}/actions/install
Assisted Service 负责实际的 OpenShift 安装过程                               │
│                                                                      │
│ 11. agent.service (所有节点)                                         │
│     ├─ After: network-online.target, set-hostname.service          │
│     ├─ Wants: network-online.target, set-hostname.service           │
│     ├─ ExecStart: /usr/local/bin/start-agent.sh                    │
│     ├─ 实际执行: exec /usr/local/bin/agent --url "${SERVICE_BASE_URL}" --infra-env-id "${INFRA_ENV_ID}"│
│     └─ 功能: 运行 Assisted Installer Agent（持续运行）echo "Waiting for infra-env-id to be available"              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Node0 vs 其他节点的服务执行对比

| 服务 | Node0 (新集群) | Node0 (添加节点) | 其他节点 (Node1, Node2...) |
|------|---------------|------------------|----------------------------|
| **agent.service** | ✅ 执行 | ✅ 执行 | ✅ 执行 |
| **agent-register-cluster.service** | ✅ 执行 | ❌ 不执行 | ❌ 不执行 |
| **agent-import-cluster.service** | ❌ 不执行 | ✅ 执行 | ❌ 不执行 |
| **agent-register-infraenv.service** | ✅ 执行 | ✅ 执行 | ❌ 不执行 |
| **apply-host-config.service** | ✅ 执行 | ✅ 执行 | ❌ 不执行 |
| **start-cluster-installation.service** | ✅ 执行 | ❌ 不执行 | ❌ 不执行 |

---

## 5. 关键条件文件

### 5.1 Node0 标识文件
- **`/etc/assisted/node0`**: 标识这是主节点
- **创建时机**: 由 `set-node-zero.sh` 在节点初始化时创建
- **作用**: 控制只有 node0 执行集群级别操作

### 5.2 场景控制文件
- **`/etc/assisted/add-nodes.env`**: 标识添加节点场景
  - 存在 → 执行 `agent-import-cluster.service`
  - 不存在 → 执行 `agent-register-cluster.service`

- **`/etc/assisted/interactive-ui`**: 标识交互式 UI 模式
  - 存在 → 跳过自动化服务

---

## 6. 网络配置时间线

```
T0: RHCOS Boot
│
├─ T1: pre-network-manager-config.service 启动 ⭐
│   ├─ 读取 /etc/assisted/agent-config.yaml
│   ├─ 解析 hosts[].networkConfig
│   ├─ 生成 /etc/NetworkManager/system-connections/*.nmconnection
│   │   ├─ bond0.nmconnection (主管理网络)
│   │   ├─ eth4.2605.nmconnection (VLAN 2605)
│   │   └─ eth5.2606.nmconnection (VLAN 2606)
│   └─ 完成
│
├─ T2: NetworkManager.service 启动
│   ├─ 读取 system-connections/ 下的配置
│   ├─ 创建 bond0 接口
│   ├─ 配置 VLAN 接口
│   ├─ 分配静态 IP 地址
│   ├─ 设置 DNS 和路由
│   └─ 网络接口 UP
│
├─ T3: network-online.target 就绪
│   └─ 网络完全可用
│
├─ T4: set-hostname.service
│   └─ 设置主机名（基于 agent-config.yaml）
│
├─ T5: assisted-service.service 启动
│   └─ Assisted Service 可访问
│
├─ T6: 并行启动 ⭐
│   ├─ agent.service (所有节点)
│   │   └─ 注册当前主机到 Assisted Service
│   └─ agent-register-cluster.service (仅 node0)
│       └─ 注册集群信息到 Assisted Service
│
├─ T7: agent-register-infraenv.service (仅 node0)
│   └─ 注册基础设施环境
│
├─ T8: apply-host-config.service (仅 node0) ⭐
│   └─ 应用从 Assisted Service 获取的主机配置
│
└─ T9: start-cluster-installation.service (仅 node0)
    └─ 开始 OpenShift 集群安装
```

---

## 7. 两个不同的 "Agent" 概念

### 7.1 `agent.service` - Assisted Installer Agent (持续运行)
```bash
ExecStart=/usr/local/bin/start-agent.sh
# 实际执行：
exec /usr/local/bin/agent --url "${SERVICE_BASE_URL}" --infra-env-id "${INFRA_ENV_ID}"
```

**功能**：
- **持续运行的 Agent 进程**
- 向 Assisted Service **注册主机**（host registration）
- 监控主机状态
- 等待安装指令
- 执行安装任务

### 7.2 `agent-register-cluster.service` - 集群注册容器 (一次性)
```bash
ExecStart=podman run ... agent-installer-client registerCluster
```

**功能**：
- **一次性容器**，运行 `agent-installer-client registerCluster`
- **注册集群信息**到 Assisted Service
- 不是主机注册，而是**集群级别的注册**

---

## 8. 关键依赖关系总结

| 服务 | Before | After | Requires | 网络相关 | Node0专属 |
|------|--------|-------|----------|----------|-----------|
| **pre-network-manager-config** | NetworkManager | - | - | ⭐ 配置网络 | ❌ |
| **NetworkManager** | - | pre-network-manager-config | - | ⭐ 应用网络 | ❌ |
| **set-hostname** | - | local-fs.target | - | 使用网络 | ❌ |
| **agent-register-cluster** | - | network-online.target | - | 需要网络 | ✅ |
| **agent-import-cluster** | - | network-online.target | - | 需要网络 | ✅ |
| **agent-register-infraenv** | - | agent-register-cluster | - | 需要网络 | ✅ |
| **apply-host-config** | - | agent-register-infraenv | agent-register-infraenv | ⭐ 可能调整网络 | ✅ |
| **start-cluster-installation** | - | apply-host-config | apply-host-config | 需要网络 | ✅ |
| **agent** | - | network-online.target | - | 需要网络 | ❌ |

---

## 9. 总结

1. **网络配置主要由两个服务负责**：
   - `pre-network-manager-config.service` (配置网络)
   - `apply-host-config.service` (应用主机配置)

2. **Node0 负责集群级别操作**：
   - 集群注册、基础设施环境注册、主机配置应用、集群安装启动
   - 通过 `/etc/assisted/node0` 文件标识

3. **所有节点都运行 agent.service**：
   - 负责向 Assisted Service 注册自己
   - 持续运行，等待和执行安装任务

4. **Dell 的网络适配脚本**：
   - `pre-network-manager-config.sh` 是 Dell 专有脚本
   - 解决逻辑接口名与实际接口名的映射问题
   - 确保在不同硬件环境下的网络配置兼容性

- **Agent-based / Assisted Installer ISO**
  - Ephemeral ISO generated by `openshift-install agent create image`;
  - Contains discovery/installer agent + RHCOS live environment.
- **Assisted Service**
  - Installation coordination center, responsible for:
    - Receiving node hardware/network reports;
    - Generating cluster definition and Ignition from manifests;
    - Performing validations (CPU/memory/disk/network etc.);
    - Triggering OpenShift cluster deployment.
- **Assisted Agent / assisted-installer-agent**
  - Agent running on nodes;
  - Responsible for self-registration, hardware reporting, Ignition retrieval, and installation execution.
- **Rendezvous host (node 0)**
  - Selected "node 0" among all nodes;
  - Runs Assisted Service locally during early boot;
  - Installation coordination center, eventually joins cluster as control plane node.
  
- **Ignition**
  - **One-time initialization configuration (first boot config)** for RHCOS/Fedora CoreOS;
  - Determines what files to write, systemd units to enable, certificates/kubeconfig to inject on first boot.
- **Bootstrap ignition**
  - Special Ignition that temporarily makes a node a **bootstrap node**, brings up temporary control plane, exits after bootstrap completion.

---

## 3. Pre-Installation (Local ISO Generation)

1. **Prepare manifests / config files** (example):
   - `agent-config.yaml`: rendezvousIP, hosts, node discovery config;
   - Cluster network/domain config (can also use ZTP SiteConfig/AgentClusterInstall/InfraEnv);
   - Pull secret, SSH public key, etc.
2. **Generate ISO**:
   ```bash
   openshift-install agent create image \
     --dir=/path/to/install-dir \
     --log-level=info
   # Output: /path/to/install-dir/agent.x86_64.iso
   ```
3. **Distribute ISO**:
   - Mount ISO to target nodes (virtual platform ISO mount, physical machine via virtual media/IPMI etc.).

---

## 4. Node Boot & "Set node specific networking"

### 1. Node boot
- All nodes boot from Agent ISO, enter RHCOS live environment.

### 2. Set node specific networking

**Goal**: Configure each machine's own network first, so it can reach Assisted Service on rendezvous host.

- Configuration sources:
  - **nmstate config**: Provide MAC/IP/VLAN/Bond/routing for each node in InfraEnv/manifests;
  - Or **DHCP**: No nmstate, NetworkManager gets address via DHCP;
  - Some environments use kernel args (`ip=` etc.) for very early networking.
- Executor:
  - **NetworkManager (+ nmstate tooling)** applies the node-level config in the live ISO environment;
  - The config itself typically comes from the ISO inputs (manifests/InfraEnv) and is consumed by the early discovery/installer logic;
  - Result: interfaces are brought up (static IP/DHCP/VLAN/bond/routes) so the node can reach `assisted-service`.
  - Goal: resolve and access `assisted-service` on node0.

> Interview one-liner: `Set node specific networking` = Apply static IP/VLAN/routing via nmstate/NetworkManager in live environment after ISO boot, enabling connection to rendezvous host's Assisted Service.

---

## 5. Node Role Branch: node0 vs Other Nodes

### 1. node0 (Left Branch)

Flow:
1. **Am I node 0?**
   - Determine if self is rendezvous host based on `agent-config` etc.
2. If **is node0**:
   - **Run assisted-service**: Start Assisted Service locally;
   - **Translate cluster manifests into AI REST API calls**:
     - Read local cluster/agent/ZTP manifests;
     - Convert to Assisted Installer REST API calls, defining cluster, network, roles etc.;
   - **Successful validations?**
     - Assisted Service validates all hosts (CPU/memory/disk/network/connectivity etc.);
       - **No** → Installation failed;
       - **Yes** → **Trigger cluster deployment**.

> This line describes: node0 as rendezvous host starts Assisted Service, defines and validates cluster, then issues "start installation" command.

### 2. Non-node0 nodes (Right Branch)

All nodes that can reach Assisted Service (including node0 as installation target) follow this path:

1. **Can I reach assisted-service?**
   - Must access rendezvous host's Assisted Service to continue, otherwise retry loop.
2. **Fetch new cluster ID**
   - Get cluster ID from Assisted Service, know which installation task belongs to.
3. **Start assisted-installer-agent**
   - Start the Assisted Installer agent (post-discovery stage) to interact with Assisted Service.
   - Note: some "agent" components already exist earlier in the live ISO to bring up networking and perform discovery; this step refers to the installation-stage agent that proceeds with registration/installation.
4. **Receive ignition**
   - Get own Ignition from Assisted Service:
     - Could be **bootstrap ignition**;
     - Or regular **control plane / worker ignition**.

---

## 6. Ignition Type Branch: bootstrap vs regular

### 1. Determine Ignition Type

- **Is it bootstrap ignition?**
  - **Yes** → `Run bootstrap`;
  - **No** → Write disk and reboot to join cluster.

### 2. Non-bootstrap nodes: Write disk + Reboot

1. **Write image and ignition to disk**
   - Write RHCOS image to local disk (e.g., `/dev/sda`);
   - Write Ignition from Assisted Service for first boot use.
2. **Write boot order and reboot into clustering**
   - Set boot order to boot from newly installed RHCOS disk;
   - After reboot, RHCOS first boot executes Ignition:
     - Write config files, start kubelet/CRI-O etc.;
     - Join cluster as master/worker.

> This line represents: regular nodes complete system initialization via Ignition and automatically join cluster.

### 3. Bootstrap node: Run bootstrap

1. **Run bootstrap**
   - Apply **bootstrap ignition**, temporarily make node a bootstrap host:
     - Write bootstrap manifests, certificates, kubeconfig;
     - Start temporary control plane components (usually done by bootkube etc.).
2. **Is bootstrap complete?**
   - During bootstrap process:
     - Help real master nodes form etcd cluster;
     - Make kube-apiserver/controller-manager/scheduler run stably on masters;
     - Key Operators start and take over control plane;
   - When control plane fully takes over → `bootstrap complete = true`.
3. After bootstrap complete:
   - Rendezvous/Bootstrap node:
     - Stop temporary bootstrap components;
     - Write formal RHCOS+Ignition, reboot as normal control-plane node.

> Interview one-liner: `Run bootstrap` = Execute bootstrap ignition, temporarily make node a bootstrap host, start temporary control plane, until real master control plane is ready, then exit and join as regular master.

---

## 7. Agent-based Full Flow Sequence (English Quick Reference)

```text
1. openshift-install agent create image
   → generate ephemeral ISO (with agents + config).

2. Nodes boot from ISO
   → set node specific networking via nmstate/NetworkManager.

3. Rendezvous host (node0)
   → run assisted-service
   → translate manifests into Assisted Installer REST API calls
   → validate all hosts
   → trigger cluster deployment.

4. All nodes (including node0) contact assisted-service
   → start assisted-installer-agent
   → register & report hardware
   → receive ignition (bootstrap or regular).

5. For non-bootstrap nodes
   → write RHCOS image + ignition to disk
   → set boot order and reboot into installed RHCOS
   → ignition runs on first boot, node joins the cluster.

6. For bootstrap node (rendezvous host)
   → run bootstrap using bootstrap ignition
   → bring up temporary control plane
   → wait until bootstrap complete
   → reboot and join as a normal control-plane node.

7. Bootstrap finishes, all nodes are part of the OCP cluster
   → installation is complete.
```

---

## 8. ZTP (Zero Touch Provisioning) (Optional)

- For large-scale/edge scenarios, use **ZTP manifests** to fully automate above process:
  - Common resources: `SiteConfig`, `AgentClusterInstall`, `InfraEnv`;
  - With GitOps, can batch deploy OpenShift across many sites with zero touch.
- Relationship:
  - Agent-based is single-cluster installation foundation;
  - ZTP = Agent-based + GitOps + multi-cluster orchestration.
