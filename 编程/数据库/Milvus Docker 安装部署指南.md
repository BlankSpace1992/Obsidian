---
title: Milvus Docker 安装部署指南
author: Milvus 官方文档
source: https://milvus.io/docs/zh/install_standalone-docker.md
date: 2025-05-14
tags:
  - Milvus
  - 向量数据库
  - Docker
  - 部署
  - RAG
---

# Milvus Docker 安装部署指南

> 来源：[Milvus 官方文档](https://milvus.io/docs/zh/install_standalone-docker.md)

---

## 前置条件

- 安装 [Docker](https://docs.docker.com/get-docker/)
- 检查硬件和软件要求：[Prerequisites](https://milvus.io/docs/prerequisite-docker.md)

---

## 安装 Milvus

Milvus 提供安装脚本，一键以 Docker 容器方式安装：

```bash
# 下载安装脚本
curl -sfL https://raw.githubusercontent.com/milvus-io/milvus/master/scripts/standalone_embed.sh -o standalone_embed.sh

# 启动 Docker 容器
bash standalone_embed.sh start
```

### 安装完成后

| 项目 | 说明 |
|------|------|
| 容器名称 | `milvus` |
| 服务端口 | **19530** |
| 内嵌 etcd | 端口 **2379**，配置文件映射为 `embedEtcd.yaml` |
| 数据目录 | 映射为 `volumes/milvus` |
| WebUI 地址 | `http://127.0.0.1:9091/webui/` |

### v3.0 新特性

- **Streaming Node**：增强数据处理能力
- **Woodpecker MQ**：改进的消息队列，降低维护开销
- **优化架构**：整合组件，提升性能

> ⚠️ 始终下载最新脚本，以确保获取最新的配置和架构改进。

---

## （可选）修改配置

修改当前目录下的 `user.yaml` 文件，例如更改代理健康检查超时：

```yaml
# 覆盖默认 milvus.yaml 的额外配置
proxy:
  healthCheckTimeout: 1000  # ms，组件健康检查间隔
```

重启服务使配置生效：

```bash
bash standalone_embed.sh restart
```

---

## 升级 Milvus

使用内置升级命令自动下载最新配置和镜像：

```bash
bash standalone_embed.sh upgrade
```

升级命令会自动：
- 下载最新安装脚本和配置
- 拉取最新 Milvus Docker 镜像
- 用新版本重启容器
- **保留现有数据和配置**

---

## 停止和删除

```bash
# 停止 Milvus
bash standalone_embed.sh stop

# 删除 Milvus 数据
bash standalone_embed.sh delete
```

---

## 后续学习

安装完成后可以继续学习：

- **基础操作**：管理数据库、集合、分区
- **数据操作**：Insert / Upsert / Delete
- **搜索**：单向量搜索、混合搜索
- **可视化管理**：[Attu](https://github.com/zilliztech/attu) — 开源 GUI 工具
- **数据备份**：[Milvus Backup](https://github.com/zilliztech/milvus-backup)
- **监控**：集成 Prometheus
- **调试**：[Birdwatcher](https://github.com/milvus-io/birdwatcher) — 动态配置和调试工具
