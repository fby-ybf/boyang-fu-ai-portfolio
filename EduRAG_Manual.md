# EduRAG 学科在线答疑系统
**容器化部署与运维排障手册 (V1.0)**

**文档拟定：** 付博扬

## 1. 架构概览与环境准备
本项目采用微服务架构设计，核心组件涵盖大模型 API 接入网关、FastAPI 服务端、MySQL（结构化 FAQ 数据）、Milvus（向量数据库）以及 Redis（缓存服务）。为保障交付效率与环境一致性，所有服务均基于 Docker Compose 进行容器化编排部署。

**前置部署环境要求：**
- **操作系统：** Ubuntu 20.04+ 或 MacOS (M1/M2)
- **容器工具：** Docker v20.10+, Docker Compose v2.0+
- **端口规划清单：** MySQL (3306), Redis (6379), Milvus (19530), FastAPI (8000)
- **权限与网络：** 开放对应防火墙端口，确保容器间的内网桥接互通。

## 2. 双路混合检索方案 (FAQ + 知识库)
为在降低大模型算力成本的同时保障系统响应准确率（>90%），本项目设计并实施了“精确匹配 + 泛化检索”的双路召回策略：

- **第一路 (FAQ 库精确匹配)：** 用户提问优先进入 MySQL 中的 FAQ 库（包含 500+ 条高频问题）进行相似度计算。当置信度阈值 ≥ 0.85 时，判定为高频确切问题，直接拦截并返回标准答案。
- **第二路 (知识库混合召回)：** 若提问未触发第一路阈值，请求将进入 Milvus 向量数据库执行 Dense（向量）检索，并配合 BM25 算法执行 Sparse（关键词）检索。双路结果经过重排序（RRF）后，拼接上下文送入大模型生成最终答复。

## 3. 典型故障排查记录 (Troubleshooting)
在现场环境搭建与系统调试过程中，累计排查并解决了 20 余项环境配置、端口连通性及依赖缺失异常。以下摘录核心排障记录：

### 🔴 故障现象一：Milvus 容器启动反复退出，报错 Connection Refused
- **排查过程：** 通过 `docker logs` 提取 milvus-standalone 容器日志，发现日志尾部出现 OOM（Out Of Memory）异常，判定为宿主机可用内存不足，导致 Milvus 核心进程被系统强杀。
- **解决方案：** 增加宿主机 Swap 分区配置，同时在 `docker-compose.yml` 中为 milvus 节点增设 `deploy.resources.limits` 限制。重启后容器长效稳定运行。

### 🔴 故障现象二：FastAPI 无法连接到同机部署的 MySQL
- **排查过程：** 使用 Postman 调用接口报错数据库连接超时。通过 Xshell 登录宿主机直连 3306 端口正常。排查环境配置文件后发现，开发环境的 DB_HOST 填写的为 `127.0.0.1`，在容器隔离环境下指向了 FastAPI 容器自身，而非 MySQL 容器。
- **解决方案：** 将环境变量中的 DB_HOST 修改为 docker-compose 内部定义的容器名称（例如 `edurag-mysql`），并确保双方在 `networks` 块中配置了相同的桥接网络。

### 🔴 故障现象三：Docker 多服务依赖启动顺序引发的连接雪崩
- **排查过程：** 一键 `docker-compose up` 启动时，FastAPI 后端因为 Milvus 尚未完成初始化而频繁抛出重连失败异常。
- **解决方案：** 利用 docker-compose 的 `depends_on` 和 `condition: service_healthy` 特性。为数据库等底层组件编写 healthcheck 脚本，强制后端服务必须在数据库完全 Readiness 后再执行启动操作。

---
**【交付物清单】:** 本手册搭配《EduRAG系统用户操作指南》及项目完整 Docker-compose 编排源码一并交付。
