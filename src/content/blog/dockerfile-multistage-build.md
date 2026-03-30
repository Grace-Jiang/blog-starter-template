---
title: Dockerfile多阶段构建最佳实践
description: Python应用的Docker多阶段构建示例与详解
pubDate: '2025-02-08'
categories:
- Kubernetes
tags:
- Docker
- Dockerfile
- 多阶段构建
---
# Dockerfile多阶段构建最佳实践

以下是一个Python应用的Docker多阶段构建示例，包含构建阶段和运行阶段的分离，最小化最终镜像大小。

```dockerfile
# ===== Build Stage：安装依赖，生成可复用层 =====
# 1. 指定构建阶段基础镜像
FROM python:3.11-slim AS build

# 2. 设置构建阶段工作目录
WORKDIR /app

# 3. 仅复制依赖文件，充分利用缓存
COPY requirements.txt .

# 4. 安装依赖到临时路径，供后续复制
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ===== Runtime Stage：运行应用 =====
# 5. 指定运行阶段基础镜像
FROM python:3.11-slim

# 6. 定义运行时需要的环境变量
ENV APP_ENV=production \
    APP_PORT=8080

# 7. 设置运行阶段工作目录
WORKDIR /app

# 8. 复制构建阶段安装好的依赖
COPY --from=build /install /usr/local

# 9. 复制应用源码
COPY . .

# 10. 指定容器启动命令
CMD ["python", "main.py"]


# 镜像更小：runtime 阶段只继承 python:3.11-slim + 应用代码和必要依赖，省去了构建阶段临时文件和构建工具，最终镜像体积更小、更易分发。
# 缓存更高效：先独立 COPY requirements.txt 并在 build 阶段安装依赖，依赖不变时会复用缓存层，加速后续构建。
# 安全性更佳：构建时用到的编译工具、临时凭证不会出现在运行镜像中，减小攻击面。
# 职责清晰：build 阶段专注于准备依赖，runtime 阶段只负责运行应用，Dockerfile 结构更易维护和扩展。

```
