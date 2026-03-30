---
title: 云原生网络设计笔记
description: 云原生网络、CNI、Service Mesh、Ingress/Gateway、Stretch Cluster设计笔记
pubDate: '2025-02-07'
categories:
- Kubernetes
tags:
- 云原生
- CNI
- Service Mesh
- Ingress
---
设计云原生网络方案、构建高可用K8s集群网络
- 物理层，management traffic 和storage traffic隔离，cross nic ha。 mgmt bond， storage 根据storage type 决定最佳网络配置。
- cni
- service 东西向  （service mesh）
- ingress/gateway 南北向 
- controller + worker   3+2

- stretch cluster 3个site，其中一个是witness site