---
title: Kubernetes Operator开发 - ConfigMapReloader实战
description: ConfigMapReloader Operator本地部署指南，基于Go的Kubernetes Operator开发
pubDate: '2025-02-06'
categories:
- Kubernetes
tags:
- Operator
- CRD
- Go
- K8s
---
# ConfigMapReloader 本地部署指南

## 概述

ConfigMapReloader 是一个 Kubernetes 操作器，用于监控 ConfigMap 变化并自动重启相关的 Deployment。

## 快速部署

### 1. 环境准备

确保已安装：
- Go 1.24.6+
- kubectl
- 访问 Kubernetes 集群

### 2. 安装 CRD

```bash
cd /home/mystic/DSP_WS/dpcm-operator/src

# 生成 CRD 和 RBAC 配置
make manifests

# 安装 CRD 到集群
make install
```

### 3. 运行控制器

#### 本地运行（推荐开发）

```bash
cd /home/mystic/DSP_WS/dpcm-operator/src
make run
```

#### 集群部署

```bash
# 构建镜像
docker build -t controller:latest .

# 部署到集群
make deploy
```

### 4. 验证部署

```bash
# 检查控制器 Pod
kubectl get pods -n src-system

# 查看控制器日志
kubectl logs -n src-system deployment/src-controller-manager
```

### 5. 测试功能

#### 创建测试资源

```yaml
# test-resources.yaml
apiVersion: dpcm.dpc.org/v1alpha1
kind: ConfigMapReloader
metadata:
  name: test-reloader
  namespace: default
spec:
  configMapRef:
    name: test-config
  deploymentSelector:
    matchLabels:
      app: test-app
  debounceSeconds: 5
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
  namespace: default
data:
  key1: value1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  namespace: default
  labels:
    app: test-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f test-resources.yaml
```

#### 测试 ConfigMap 变化

```bash
# 更新 ConfigMap
kubectl patch configmap test-config --type merge -p '{"data":{"key1":"updated-value-'$(date +%s)'"}}'

# 等待 debounce 时间（5 秒）
sleep 6

# 检查 Deployment 是否被更新
kubectl get deployment test-deployment -o yaml | grep "dpcm.io/restartedAt"
```

## 配置说明

### ConfigMapReloader Spec

```yaml
spec:
  configMapRef:
    name: configmap-name          # 要监控的 ConfigMap 名称
  deploymentSelector:
    matchLabels:
      app: app-label            # 匹配 Deployment 的标签
  debounceSeconds: 10            # 防抖延迟时间，默认 10 秒
```

### Status 字段

```yaml
status:
  observedConfigMapResourceVersion: "12345"    # 最后观察到的 ConfigMap 版本
  pendingRestartUntil: "2026-01-27T08:30:48Z" # 等待重启的时间点
  lastRestartTime: "2026-01-27T08:30:48Z"     # 最后重启时间
```

## 故障排查

### 常见问题

**Deployment 不重启**
- 检查 RBAC 权限：`make manifests && kubectl apply -f config/rbac/`
- 检查 Deployment selector 是否匹配
- 查看控制器日志

**权限错误**
```bash
# 重新生成和应用 RBAC
make manifests
kubectl apply -f config/rbac/role.yaml
```

### 调试命令

```bash
# 查看所有 ConfigMapReloader
kubectl get configmapreloader -A

# 查看控制器日志
kubectl logs -n src-system deployment/src-controller-manager --tail=50

# 检查事件
kubectl get events --sort-by=.metadata.creationTimestamp
```

## 清理

```bash
# 删除测试资源
kubectl delete -f test-resources.yaml

# 卸载 CRD
make uninstall
```

## 工作原理

1. **监控 ConfigMapReloader 资源**
2. **监控 ConfigMap 变化**
3. **Debounce 机制**：避免频繁重启
4. **Deployment 重启**：通过更新 Pod Template annotation 触发滚动更新

## 开发提示

- 修改代码后需要重新运行 `make run`
- 添加 RBAC 权限后需要运行 `make manifests` 和重新应用
- 使用 `logf.Log.Info()` 添加调试日志
- 测试时建议设置较小的 `debounceSeconds` 值（如 5 秒）
