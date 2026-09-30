# MaxKB4j 轻量级企业 RAG 智能知识库
**内网环境私有化部署与排障复盘 (V1.0)**

**文档拟定：** 付博扬

## 1. 业务场景与部署架构
本项目旨在为企业提供轻量级的内部知识问答系统。技术栈主要基于 SpringBoot 3、Java 21 搭配 LangChain4j，底层依托本地向量数据库进行知识切分与检索。考虑到企业数据安全要求，本次交付的核心挑战在于**“完全无外网（内网隔离）环境下的私有化部署与联调”**。

## 2. 离线网络环境适配核心方案
在无外网环境下，常规的在线下载依赖与公有云 API 调用均无法进行，为此制定了以下落地方案：
- **离线依赖构建：** 在有网环境下，利用 Maven 离线打包插件（`mvn dependency:go-offline`）预先拉取所有 Java 21 和 SpringBoot 3 依赖，封装为 `fat-jar` 与依赖库压缩包，通过 U 盘物理导入企业内网服务器。
- **本地化大模型接入：** 弃用公有云 API，协助客户在内网算力节点上通过 Ollama / vLLM 部署开源大模型（如 Qwen 或 Llama3），并将 MaxKB4j 的 `base_url` 指向内网服务 IP，彻底打通本地问答链路。
- **向量模型本地化：** 同理，将 Embedding 模型（用于文本向量化）下载至本地加载，避免在文档上传解析阶段因网络请求超时导致系统卡死。

## 3. 核心环境排障记录 (Troubleshooting)
在现场实施阶段，共解决 15+ 项环境组件冲突与依赖异常，以下为关键问题复盘：

### 🔴 异常一：JDK 版本不兼容引发的启动崩溃
- **问题现象：** 执行 `java -jar maxkb4j.jar` 时，控制台抛出 `UnsupportedClassVersionError: ... class file version 65.0`。
- **排查过程：** 报错指出 class file version 65.0（对应 Java 21），通过执行 `java -version` 发现客户服务器默认环境为 JDK 1.8（版本号 52.0）。
- **解决方案：** 未卸载客户原有的 JDK（避免影响其他业务），而是通过免安装版的 JDK 21 压缩包解压，使用绝对路径显式指定启动：`/path/to/jdk-21/bin/java -jar maxkb4j.jar`。

### 🔴 异常二：默认端口占用导致服务启动失败
- **问题现象：** SpringBoot 启动时抛出 `Web server failed to start. Port 8080 was already in use.`
- **排查过程：** 通过 `netstat -tulnp | grep 8080` 命令排查，发现该端口被企业内网的另外一套老旧系统占用。
- **解决方案：** 在启动命令中通过参数重写端口：`java -jar maxkb4j.jar --server.port=8082`，并同步修改了 Nginx 反向代理配置，将流量正确分发至 8082 端口。

### 🔴 异常三：知识库切分粒度导致检索准确率低下
- **问题现象：** 业务方反馈，在上传了《员工手册》后，大模型回答经常截断或答非所问。
- **排查过程：** 检查 LangChain4j 的 Document Splitter 默认配置，发现 Chunk Size（分块大小）过小，导致段落上下文语义被强制腰斩。
- **解决方案：** 结合业务手册多为“长段落条款”的特性，将 Chunk Size 调大至 800 tokens，增加 Chunk Overlap（重叠区）至 100 tokens，并调优检索召回的 top-k 参数。重启服务后重建向量库，检索准确度大幅提升。
