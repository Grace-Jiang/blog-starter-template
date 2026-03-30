---
title: OpenShift主机名配置深度分析
description: Agent-Based安装中LiveOS和PersistOS阶段的主机名配置技术分析
pubDate: '2025-02-03'
categories:
- OpenShift
tags:
- OpenShift
- 主机名
- LiveOS
- PersistOS
---
# OpenShift Agent-Based Installation - Hostname配置机制完整分析

## 📋 概述

本文档详细分析了OpenShift Agent-based安装中hostname的配置机制，涵盖LiveOS和Persist OS两个阶段的完整流程。

**分析环境**:
- 集群: c2-raven
- 节点: c2-dpc01.rackh16.fkln (172.24.22.101)
- 安装方式: Agent-based installation
- 分析时间: 2026-02-10

---

## 🔄 两个阶段概览

```
┌─────────────────────────────────────────────────────────────┐
│                    Hostname配置流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  LiveOS阶段                    Persist OS阶段                │
│  (临时系统)                    (持久化系统)                  │
│      ↓                              ↓                        │
│  Agent ISO启动                 OpenShift集群运行             │
│      ↓                              ↓                        │
│  Ignition配置                  MachineConfig配置             │
│      ↓                              ↓                        │
│  set-hostname.sh               acp-set-hostname.sh           │
│  (MAC地址匹配)                 (DNS反向解析)                 │
│      ↓                              ↓                        │
│  快速、静态                    灵活、动态                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🎯 阶段1: LiveOS阶段 (Agent ISO启动)

### 1.1 配置准备 (安装前)

#### **配置者**: 管理员/自动化工具

#### **配置文件**: `agent-config.yaml`
```yaml
# agent-config.yaml
hosts:
  - hostname: c2-dpc01.rackh16.fkln
    interfaces:
      - name: bond0
        macAddress: b8:e9:24:ee:bb:e4
  
  - hostname: c2-dpc02.rackh16.fkln
    interfaces:
      - name: eno1
        macAddress: d4:04:e6:2f:41:30
  
  - hostname: c2-dpc03.rackh16.fkln
    interfaces:
      - name: eno1
        macAddress: d4:04:e6:2f:a3:80
```

**关键设计**: MAC地址 → Hostname映射

---

### 1.2 ISO生成

#### **工具**: `openshift-install agent create image`

```bash
# 生成包含hostname配置的Agent ISO
openshift-install agent create image \
  --dir /path/to/install-config
```

**生成内容**:
- Agent ISO镜像
- 嵌入的Ignition配置
- hostname映射配置

---

### 1.3 Ignition配置 (08:53:28)

#### **配置者**: Ignition (CoreOS初始化系统)

#### **写入文件**:
```bash
# Ignition写入hostname配置文件
/etc/assisted/hostnames/b8:e9:24:ee:bb:e4  → c2-dpc01.rackh16.fkln
/etc/assisted/hostnames/d4:04:e6:2f:41:30  → c2-dpc02.rackh16.fkln
/etc/assisted/hostnames/d4:04:e6:2f:a3:80  → c2-dpc03.rackh16.fkln

# 创建systemd服务
/etc/systemd/system/set-hostname.service

# 创建hostname设置脚本
/var/usrlocal/bin/set-hostname.sh
https://github.com/openshift/installer/blob/main/data/data/agent/files/usr/local/bin/set-hostname.sh
```

#### **日志证据**:
```
Feb 10 08:53:28 localhost ignition[2809]: writing file "/sysroot/etc/assisted/hostnames/b8:e9:24:ee:bb:e4"
Feb 10 08:53:28 localhost ignition[2809]: writing file "/sysroot/etc/assisted/hostnames/d4:04:e6:2f:41:30"
Feb 10 08:53:28 localhost ignition[2809]: writing file "/sysroot/etc/assisted/hostnames/d4:04:e6:2f:a3:80"
Feb 10 08:53:30 localhost ignition[2809]: processing unit "set-hostname.service"
Feb 10 08:53:30 localhost ignition[2809]: writing unit "set-hostname.service"
Feb 10 08:53:58 localhost ignition[2809]: writing file "/sysroot/var/usrlocal/bin/set-hostname.sh"
```

---

### 1.4 systemd服务配置

#### **服务文件**: `/etc/systemd/system/set-hostname.service`
```ini
[Unit]
Description=Agent-based installer hostname update service
Wants=network-online.target
After=local-fs.target
Before=agent-interactive-console.service

[Service]
ExecStart=/usr/local/bin/set-hostname.sh
Type=oneshot
RemainAfterExit=true
KillMode=none

[Install]
WantedBy=multi-user.target
```

---

### 1.5 Hostname设置脚本

#### **脚本**: `/var/usrlocal/bin/set-hostname.sh`
```bash
#!/bin/bash

# hostname配置目录
HOSTNAMES_PATH=/etc/assisted/hostnames

# 遍历所有MAC地址配置文件
FILES=$(ls $HOSTNAMES_PATH)
for filename in ${FILES}
do
    # 检查当前主机是否有这个MAC地址
    MATCHED_MAC_ADDRESS_WITH_HOST=$(ip address | grep "${filename}")
    
    if [ "$MATCHED_MAC_ADDRESS_WITH_HOST" != "" ]; then
        # 读取对应的hostname
        HOSTNAME="$(cat "${HOSTNAMES_PATH}/${filename}")"
        
        echo "Host has matching MAC address: ${filename}" 1>&2
        echo "Setting hostname to ${HOSTNAME}" 1>&2
        
        # 设置hostname
        hostnamectl set-hostname "${HOSTNAME}"
    else
        echo "MAC address, ${filename}, does not exist on this host" 1>&2
    fi
done
```

**核心逻辑**: MAC地址匹配

---

### 1.6 执行过程 (08:54:27-08:54:30)

#### **执行流程**:
```bash
# 1. systemd启动服务
Feb 10 08:54:27 localhost systemd[1]: Starting Agent-based installer hostname update service...

# 2. 脚本执行，匹配MAC地址
Feb 10 08:54:29 localhost set-hostname.sh[4635]: Host has matching MAC address: b8:e9:24:ee:bb:e4
Feb 10 08:54:29 localhost set-hostname.sh[4635]: Setting hostname to c2-dpc01.rackh16.fkln

# 3. 检查其他MAC地址（不匹配）
Feb 10 08:54:30 c2-dpc01.rackh16.fkln set-hostname.sh[4635]: MAC address, d4:04:e6:2f:41:30, does not exist on this host
Feb 10 08:54:30 c2-dpc01.rackh16.fkln set-hostname.sh[4635]: MAC address, d4:04:e6:2f:a3:80, does not exist on this host

# 4. 服务完成
Feb 10 08:54:30 c2-dpc01.rackh16.fkln systemd[1]: Finished Agent-based installer hostname update service.
```

**结果**: hostname从 `localhost` 变成 `c2-dpc01.rackh16.fkln`

---

### 1.7 Agent注册 (08:55:52)

#### **Assisted Agent报告hostname**:
```bash
# Agent读取系统hostname
hostname=$(hostname)
# 输出: c2-dpc01.rackh16.fkln

# 报告给Assisted Service
POST /api/assisted-install/v2/infra-envs/.../hosts/.../instructions
{
  "inventory": {
    "hostname": "c2-dpc01.rackh16.fkln",
    ...
  }
}
```

#### **Assisted Service验证**:
```
Feb 10 08:55:52 c2-dpc01.rackh16.fkln service[7500]: "No request for hostname update for host ec46c268-0ecc-5fb0-2b00-2ef68eb0eb93"
Feb 10 08:55:52 c2-dpc01.rackh16.fkln service[7500]: "hostname-unique Status:success Message:Hostname c2-dpc01.rackh16.fkln is unique in cluster"
Feb 10 08:55:52 c2-dpc01.rackh16.fkln service[7500]: "hostname-valid Status:success Message:Hostname c2-dpc01.rackh16.fkln is allowed"
```

---

## 🎯 阶段2: Persist OS阶段 (OpenShift集群运行)

### 2.1 MachineConfig生成 (08:55:44)

#### **配置者**: Assisted Service

#### **生成的Manifest**:
```yaml
# openshift/99-worker-set-host-name-0.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-set-host-name-0
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,IyEvYmluL2Jhc2g...
        mode: 493
        overwrite: true
        path: /usr/local/bin/acp-set-hostname.sh
    systemd:
      units:
      - contents: |
          [Unit]
          Description=Try to set node with a non-localhost hostname
          Before=kubelet-dependencies.target

          [Service]
          Type=oneshot
          RemainAfterExit=yes
          User=root
          ExecStart=/usr/local/bin/acp-set-hostname.sh

          # Wait up to 5min for the node to get a non-localhost name
          TimeoutSec=600

          [Install]
          WantedBy=kubelet-dependencies.target
        enabled: true
        name: acp-set-hostname.service
```

#### **日志证据**:
```
Feb 10 08:55:44 c2-dpc01.rackh16.fkln service[7500]: Creating manifest openshift/99-worker-set-host-name-0.yaml
Feb 10 08:55:44 c2-dpc01.rackh16.fkln service[7500]: Successfully uploaded file 34b2becc-d17c-46c2-a15e-264096045c08/manifests/openshift/99-worker-set-host-name-0.yaml
Feb 10 08:55:44 c2-dpc01.rackh16.fkln service[7500]: Done creating manifest openshift/99-worker-set-host-name-0.yaml
```

---

### 2.2 acp-set-hostname.sh脚本

#### **配置方式**: DNS反向解析

```bash
#!/bin/bash

LOG_FILE=/var/log/acp-set-hostname.log

exec &>> >(tee -ia "$LOG_FILE")

while true; do
    hostname=$(cat /etc/hostname)
    
    # 检查hostname是否已设置
    if [ -z "$hostname" ]; then
        echo "hostname is null, try to set the hostname"
    elif ! echo $hostname | grep -q "localhost"; then
        echo "hostname has been set to $hostname successfully"
        break
    fi

    # 获取DNS服务器
    nameservers=$(cat /etc/resolv.conf | grep ^nameserver)
    if [ -z "$nameservers" ]; then
        echo "no nameserver is configured, retry..."
    else
        nameservers=$(echo "$nameservers" | awk '{print $2}')
        host_address=""
        ext_name_server=""
        
        # 区分本机IP和外部DNS服务器
        for nameserver in $nameservers; do
            matched_address=$(ip address | grep "inet $nameserver")
            ret_code=$?
            if [ $ret_code -ne 0 ] || [ -z "$matched_address" ]; then
                ext_name_server=$nameserver
            else
                host_address=$nameserver
            fi
        done
        
        if [ "$host_address" = "" ]; then
            echo "host address is not found in resolv.conf, retry..."
        else
            echo "host address: $host_address, nameserver: $ext_name_server"
            
            # DNS反向解析获取hostname
            records=$(nslookup "$host_address" "$ext_name_server")
            echo "nslookup results: $records"
            ret_code=$?
            
            if [ $ret_code -eq 0 ]; then
                record=$(echo "$records" | head -n 1 | awk '{print $4}')
                last_char=${record: -1}
                if [ "$last_char" = "." ]; then
                    record=${record%?}
                fi
                echo "set hostname to $record"
                hostnamectl hostname "$record"
            else
                echo "nslookup $hostaddress from $ext_name_server failed"
            fi
        fi
    fi
    sleep 1
done
```

**核心逻辑**: DNS PTR记录反向解析

---

### 2.3 执行流程

#### **部署到集群**:
```bash
# 1. Machine Config Operator应用配置
MCO检测到新的MachineConfig
  ↓
MCO将配置应用到对应的节点
  ↓
节点重启应用新配置

# 2. acp-set-hostname.service启动
Before=kubelet-dependencies.target  # 在kubelet启动前执行

# 3. acp-set-hostname.sh执行
while true; do
    # 获取本机IP地址
    host_address=$(从/etc/resolv.conf找到本机IP)
    
    # DNS反向解析
    hostname=$(nslookup $host_address $dns_server | awk '{print $4}')
    
    # 设置hostname
    hostnamectl hostname "$hostname"
    
    # 如果成功则退出
    if hostname != "localhost"; then
        break
    fi
    
    sleep 1
done
```

---

## 📊 两个阶段对比

### 配置机制对比

| 特性 | LiveOS阶段 | Persist OS阶段 |
|------|-----------|---------------|
| **配置来源** | agent-config.yaml | Assisted Service生成 |
| **配置格式** | Ignition (ISO内嵌) | MachineConfig (OpenShift资源) |
| **脚本名称** | set-hostname.sh | acp-set-hostname.sh |
| **匹配方式** | MAC地址匹配 | DNS反向解析 |
| **配置文件** | /etc/assisted/hostnames/<MAC> | 无（动态DNS查询） |
| **依赖** | agent-config.yaml预定义 | DNS PTR记录 |
| **执行时机** | LiveOS启动时 | kubelet启动前 |
| **systemd服务** | set-hostname.service | acp-set-hostname.service |
| **超时时间** | 无明确限制 | 600秒 (10分钟) |
| **日志位置** | journalctl | /var/log/acp-set-hostname.log |
| **存储位置** | 内存 (tmpfs) | 磁盘 (持久化) |
| **生命周期** | 临时 (重启丢失) | 永久 (重启保留) |

---



