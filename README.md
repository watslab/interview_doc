# Interview Doc

技术面试相关文档整理，涵盖数据库、搜索引擎、缓存、编程语言等核心技术领域。

## 文档列表

### Java

| 文档 | 概述 |
|------|------|
| [Java HashMap 扩容机制详解](./document/Java/Java%20HashMap%20扩容机制详解.md) | 深入解析 HashMap 扩容机制，包括触发条件、元素迁移流程、JDK 1.7 与 1.8 核心差异对比、面试高频问题总结 |
| [Java 单例模式详解](./document/Java/Java%20单例模式详解.md) | 详解单例模式的四种实现方式：饿汉模式、懒汉模式（单线程/多线程/双重检查锁定），分析线程安全与性能权衡 |
| [Java WebSocket 开发详解](./document/Java/Java%20WebSocket%20开发详解.md) | 全面解析 Java WebSocket 开发方案（JSR-356、Spring WebSocket、Netty），包括 STOMP 协议、心跳机制、断线重连、集群方案与最佳实践 |

### Spring

| 文档 | 概述 |
|------|------|
| [Spring 容器启动流程详解](./document/Spring/Spring%20容器启动流程详解.md) | 深入解析 Spring 容器启动流程，包括 refresh() 方法 12 个核心步骤、Spring Boot 启动流程、传统 Spring 与 Spring Boot 启动差异对比 |
| [Spring 循环依赖解决方案详解](./document/Spring/Spring%20循环依赖解决方案详解.md) | 深入解析 Spring 三级缓存解决循环依赖的原理，包括缓存架构、解决流程、AOP 代理兼容性、构造器注入与 Setter 注入差异对比 |
| [SpringMVC 核心原理详解](./document/Spring/SpringMVC%20核心原理详解.md) | 深入解析 SpringMVC 核心原理，包括请求处理流程、核心组件职责、拦截器机制、常用注解、前后端分离与传统 MVC 对比 |
| [Spring Bean 作用域详解](./document/Spring/Spring%20Bean%20作用域详解.md) | 详解 Spring Bean 的六种作用域，重点分析单例 Bean 依赖原型 Bean 的问题与四种解决方案 |
| [Spring 事务管理机制详解](./document/Spring/Spring%20事务管理机制详解.md) | 深入解析 Spring 事务管理机制，包括事务传播行为、隔离级别、AOP 动态代理实现原理、@Transactional 注解详解、事务失效场景分析、跨线程事务解决方案 |

### MySQL

| 文档 | 概述 |
|------|------|
| [MySQL MVCC 多版本并发控制机制详解](./document/MySQL/MySQL%20MVCC%20多版本并发控制机制详解.md) | 深入讲解 MySQL InnoDB 的 MVCC 机制，包括隐藏列、Undo Log、Read View、版本可见性判断规则，以及 MVCC 与事务隔离级别的关系、Next-Key Lock 解决幻读问题等 |
| [MySQL 多粒度锁机制详解](./document/MySQL/MySQL%20多粒度锁机制详解.md) | 详解 MySQL InnoDB 的多粒度锁机制，包括表锁、行锁、意向锁（IS/IX）的工作原理与兼容性矩阵，以及意向锁与 MVCC 的协作关系 |
| [MySQL 三大日志详解](./document/MySQL/MySQL%20三大日志详解.md) | 深入解析 MySQL Binlog、Redo Log、Undo Log 三大日志系统，包括核心原理、存储结构、刷盘策略、两阶段提交机制、崩溃恢复流程与最佳实践 |
| [MySQL Binlog 二进制日志详解](./document/MySQL/MySQL%20Binlog%20二进制日志详解.md) | 深入解析 MySQL Binlog 二进制日志，包括三种格式（STATEMENT/ROW/MIXED）原理与对比、主从复制机制、数据恢复实践、Canal 数据订阅应用 |
| [MySQL 集群架构详解](./document/MySQL/MySQL%20集群架构详解.md) | 系统介绍 MySQL 集群架构，包括主从复制架构、高可用方案（MHA、MGR、InnoDB Cluster）、分布式集群方案（NDB Cluster、PXC/Galera Cluster），详解认证复制原理、GCache 机制、状态同步方式 |
| [MySQL 索引原理与失效场景详解](./document/MySQL/MySQL%20索引原理与失效场景详解.md) | 深入解析 MySQL 索引原理，包括 B+ 树数据结构（逻辑结构、物理存储、与 B 树/哈希索引/二叉树对比）、InnoDB 聚簇索引与二级索引实现、索引优化机制（覆盖索引、ICP、AHI、最左前缀原则）、索引失效场景全面分析（12 种场景及解决方案）、索引设计原则 |

### PostgreSQL

| 文档 | 概述 |
|------|------|
| [PostgreSQL 分区表最佳实践与底层原理](./document/PostgreSQL/PostgreSQL%20分区表最佳实践与底层原理.md) | 全面介绍 PostgreSQL 分区表技术，包括声明式分区与继承分区的实现原理、分区策略选择、最佳实践与性能优化建议 |
| [PostgreSQL 分级锁机制详解](./document/PostgreSQL/PostgreSQL%20分级锁机制详解.md) | 深入解析 PostgreSQL 的三级锁体系：Spin Lock、Lightweight Lock、Regular Lock，以及 8 种表级锁的兼容性与应用场景 |
| [PostgreSQL 锁机制与分区表操作详解](./document/PostgreSQL/PostgreSQL%20锁机制与分区表操作详解.md) | PostgreSQL 锁机制与分区表操作详解 |

### ClickHouse

| 文档 | 概述 |
|------|------|
| [ClickHouse 列式数据库详解](./document/ClickHouse/ClickHouse%20列式数据库详解.md) | 深入解析 ClickHouse 列式数据库，包括核心架构（列式存储、向量化执行、稀疏索引）、MergeTree 存储引擎家族、分布式架构（分片与副本）、应用场景、与 MySQL/Elasticsearch 对比、最佳实践 |
| [ClickHouse 向量化执行技术详解](./document/ClickHouse/ClickHouse%20向量化执行技术详解.md) | 深入解析 ClickHouse 向量化执行技术，涵盖 SIMD 硬件基础（SSE/AVX 指令集）、数据库执行模型演进（火山模型→向量化模型→编译执行）、ClickHouse 向量化实现（Block 列式批处理、IColumn 类型派生）、与编译执行（Code Generation）对比、性能优势与局限性、最佳实践 |

### 数据库架构

| 文档 | 概述 |
|------|------|
| [OLTP与OLAP数据处理架构详解](./document/数据库/OLTP与OLAP数据处理架构详解.md) | 深入解析 OLTP（联机事务处理）与 OLAP（联机分析处理）两大数据处理架构，包括核心特征、技术架构、数据模型、全面对比、传统分离架构问题、HTAP 混合架构原理与代表产品（TiDB、OceanBase）、选型指南 |

### ElasticSearch

| 文档 | 概述 |
|------|------|
| [ElasticSearch 集群部署最佳实践](./document/ElasticSearch/ElasticSearch%20集群部署最佳实践.md) | ElasticSearch 集群部署的架构设计、硬件配置、JVM 调优等最佳实践指南 |
| [ElasticSearch 集群管理完全指南](./document/ElasticSearch/ElasticSearch%20集群管理完全指南.md) | ElasticSearch 集群的日常管理操作，包括节点管理、索引管理、集群监控与故障处理 |
| [ElasticSearch 集群状态详解](./document/ElasticSearch/ElasticSearch%20集群状态详解.md) | 深入解析 ElasticSearch 集群状态机制，包括状态同步流程、常见状态问题排查 |
| [ElasticSearch 索引分片设置最佳实践](./document/ElasticSearch/ElasticSearch%20索引分片设置最佳实践.md) | ElasticSearch 索引分片数量与大小的最佳设置策略，平衡性能与资源利用率 |
| [ElasticSearch 分片数量设置最佳实践详解](./document/ElasticSearch/ElasticSearch%20分片数量设置最佳实践详解.md) | 详细讲解 ElasticSearch 分片数量的设置原则、影响因素与优化建议 |
| [ElasticSearch 索引一致性参数详解](./document/ElasticSearch/ElasticSearch%20索引一致性参数详解.md) | ElasticSearch 索引一致性相关参数的配置与调优，包括 refresh、flush、sync 等机制 |
| [ElasticSearch 修改现有索引 Settings 详解](./document/ElasticSearch/ElasticSearch%20修改现有索引%20Settings%20详解.md) | 如何安全地修改 ElasticSearch 现有索引的配置参数，包括动态设置与静态设置的区别 |
| [ElasticSearch 底层存储原理详解](./document/ElasticSearch/ElasticSearch%20底层存储原理详解.md) | ElasticSearch 底层存储架构解析，包括 Lucene 段文件、倒排索引、存储结构等 |
| [ElasticSearch 数值类型索引结构详解](./document/ElasticSearch/ElasticSearch%20数值类型索引结构详解.md) | ElasticSearch 数值类型的索引结构原理，包括 BKD 树、数值排序与范围查询优化 |
| [ElasticSearch 常用分词器详解](./document/ElasticSearch/ElasticSearch%20常用分词器详解.md) | ElasticSearch 常用分词器的原理与使用场景，包括标准分词器、IK 分词器等 |
| [ElasticSearch 教程](./document/ElasticSearch/ElasticSearch%20教程.md) | ElasticSearch 入门教程，涵盖基础概念、CRUD 操作、查询语法等内容 |

### Redis

| 文档 | 概述 |
|------|------|
| [Redis HyperLogLog 详解](./document/Redis/Redis%20HyperLogLog%20详解.md) | 深入解析 Redis HyperLogLog 基数估计算法，包括原理、误差分析、命令详解与使用场景 |
| [Redis 批量数据原子性操作解决方案](./document/Redis/Redis%20批量数据原子性操作解决方案.md) | Redis 批量数据操作的原子性保证方案，包括 Pipeline、Lua 脚本、事务等技术的应用 |
| [Redis Pipeline 管道技术详解](./document/Redis/Redis%20Pipeline%20管道技术详解.md) | Redis Pipeline 管道技术详解 |
| [Redis 缓存穿透击穿雪崩详解](./document/Redis/Redis%20缓存穿透击穿雪崩详解.md) | 深入解析 Redis 缓存三大问题，包括穿透、击穿、雪崩的场景分析、解决方案对比与最佳实践 |

### 消息中间件

| 文档 | 概述 |
|------|------|
| [消息中间件 MQ 对比详情](./document/消息中间件/消息中间件%20MQ%20对比详情.md) | 全面对比 ActiveMQ、RabbitMQ、Kafka、RocketMQ、Pulsar 五大消息中间件，包括架构图、工作流程、性能指标、功能对比与选型建议 |
| [RocketMQ 消息中间件详解](./document/消息中间件/RocketMQ%20消息中间件详解.md) | 深入解析 RocketMQ 核心架构、消息模型、消息发送与消费流程、顺序消息、事务消息、延迟消息等高级特性与最佳实践 |

### 微服务架构

| 文档 | 概述 |
|------|------|
| [Seata 分布式事务详解](./document/微服务架构/Seata%20分布式事务详解.md) | 深入解析 Seata 分布式事务解决方案，包括分布式事务基础理论、三大核心角色（TC/TM/RM）、四种事务模式（AT/TCC/SAGA/XA）原理与对比、Spring Cloud Alibaba 集成配置与最佳实践 |
| [Nacos 注册配置中心详解](./document/微服务架构/Nacos%20注册配置中心详解.md) | 深入解析 Nacos 注册配置中心，包括三层隔离模型（Namespace/Group/Data ID）、服务注册发现与健康检查机制、配置管理与动态刷新、AP/CP 模式切换、Spring Cloud Alibaba 集成配置 |
| [SkyWalking 分布式链路追踪详解](./document/微服务架构/SkyWalking%20分布式链路追踪详解.md) | 深入解析 SkyWalking 分布式链路追踪系统，包括核心架构、分布式追踪原理、Java Agent 字节码增强机制、核心功能（服务拓扑、调用链追踪、性能监控、告警）、实战部署与最佳实践 |
| [Sentinel 流量控制与限流算法详解](./document/微服务架构/Sentinel%20流量控制与限流算法详解.md) | 深入解析 Sentinel 流量控制组件，包括核心概念、责任链工作原理、流量控制策略、熔断降级机制，以及固定窗口、滑动窗口、漏桶、令牌桶四种限流算法详解 |
| [Spring Cloud Alibaba 面试项目推荐方案](./document/微服务架构/Spring%20Cloud%20Alibaba%20面试项目推荐方案.md) | Spring Cloud Alibaba 面试项目推荐方案 |

### Docker

| 文档 | 概述 |
|------|------|
| [Docker 容器技术详解](./document/Docker/Docker%20容器技术详解.md) | 全面介绍 Docker 容器技术，包括核心概念、底层原理、常用命令、Dockerfile 编写、Docker Compose 使用与实战案例 |

### DevOps

| 文档 | 概述 |
|------|------|
| [CI/CD 持续集成与持续交付详解](./document/DevOps/CI-CD%20持续集成与持续交付详解.md) | 系统梳理现代 CI/CD 技术体系，涵盖核心概念、五阶段技术演进、主流工具对比（Jenkins/GitLab CI/GitHub Actions/ArgoCD/Flux）、GitOps 部署范式（Push vs Pull）、DevSecOps 安全左移实践、DORA 效能度量与未来趋势 |

### Kubernetes

| 文档 | 概述 |
|------|------|
| [Kubernetes 技术体系总论](./document/Kubernetes/Kubernetes%20技术体系总论.md) | 系统阐述容器编排的由来、Kubernetes 在 DevOps 体系中的定位、核心设计思想（声明式 API、调谐循环、不可变基础设施）、技术体系全景（六大领域）及学习路径，为后续分论奠定整体认知框架 |
| [Kubernetes 集群架构与核心组件详解](./document/Kubernetes/Kubernetes%20集群架构与核心组件详解.md) | 系统讲解 Kubernetes 集群构成与协作机制，覆盖控制平面与节点组件职责、API 请求全链路、声明式 API 与调谐循环设计、控制平面与 etcd 高可用方案 |
| [Kubernetes 工作负载与核心对象详解](./document/Kubernetes/Kubernetes%20工作负载与核心对象详解.md) | 系统阐述工作负载体系，涵盖 Pod 结构与生命周期、Init/Sidecar 容器差异、控制器调谐模式，以及 Deployment、StatefulSet、DaemonSet、Job、CronJob 五类工作负载对象职责与适用场景，梳理 Label、Selector 与 Namespace 组织机制 |
| [Kubernetes 服务路由与存储体系详解](./document/Kubernetes/Kubernetes%20服务路由与存储体系详解.md) | 系统梳理 Kubernetes 服务路由与存储体系，涵盖 Service 四种类型（ClusterIP/NodePort/LoadBalancer/ExternalName）、Endpoints/EndpointSlice、CoreDNS 服务发现、Headless Service、Ingress 与 IngressClass、kube-proxy 模式、Volume 体系、PV/PVC 静态供给与 accessModes/reclaimPolicy、StorageClass 动态供给与 CSI 标准接口 |
| [Kubernetes 调度与资源管理详解](./document/Kubernetes/Kubernetes%20调度与资源管理详解.md) | 深入解析 Kubernetes 调度器两阶段决策（Filter/Score/Bind）、资源 requests 与 limits、QoS 三级服务质量等级、节点亲和性与 Pod 亲和反亲和、污点与容忍、ResourceQuota 与 LimitRange 多租户资源治理 |
| [Kubernetes 安全与访问控制详解](./document/Kubernetes/Kubernetes%20安全与访问控制详解.md) | 系统阐述安全与访问控制体系，覆盖 API 请求安全链路（认证/鉴权/准入控制）、RBAC 权限管理、Secret 敏感数据保护、Pod Security Admission 安全基线、NetworkPolicy 网络隔离 |
| [Kubernetes 集群部署与生产运维实践详解](./document/Kubernetes/Kubernetes%20集群部署与生产运维实践详解.md) | 系统讲解 Kubernetes 生产部署与运维实践，涵盖 kubeadm 集群引导与高可用、CRI 容器运行时、Helm 包管理、GitOps 持续部署（ArgoCD/Flux）、可观测性三支柱（Prometheus 监控/EFK·PLG 日志/OpenTelemetry 链路追踪）、健康检查探针、集群升级与版本偏差、故障排查 |

### 大数据

| 文档 | 概述 |
|------|------|
| [大数据开发技术发展历程](./document/大数据/大数据开发技术发展历程.md) | 系统梳理大数据开发技术从萌芽到智能化的完整发展脉络，涵盖概述（5V特征/时代背景）、五大发展时代（萌芽期·磁盘计算·内存计算·实时计算·云原生与智能化）、三代架构演进（Lambda/Kappa/湖仓一体）、技术栈代际全景与未来趋势展望 |

### AI

| 文档 | 概述 |
|------|------|
| [AI 大模型技术发展历程](./document/AI/AI%20大模型技术发展历程.md) | 系统梳理 AI 大模型技术从深度学习基础到现代大语言模型的完整发展脉络，以"时间轴 + 技术突破 + 范式演进"为分析框架，涵盖 1986—2025 年间反向传播算法奠基、深度学习复兴、Transformer 架构革命、预训练范式确立（GPT/BERT/Scaling Law/Chinchilla）、对齐技术成熟（RLHF/InstructGPT）、大模型爆发（ChatGPT/GPT-4/百模大战）、推理模型与智能体时代（o1/DeepSeek-R1/Agentic AI）七个关键阶段 |
| [AI 大模型应用技术发展历程](./document/AI/AI%20大模型应用技术发展历程.md) | 系统梳理 AI 大模型应用技术从"能用"到"好用"再到"可靠"的完整发展脉络，以"时间轴 + 技术分层 + 范式演进"为分析框架，涵盖 2017—2026 年间提示工程时代（Zero-Shot/Few-Shot/CoT）、检索增强生成（RAG 四代演进/Advanced RAG）、工具调用与 Function Calling（ReAct/MCP）、Agent 框架爆发（四大设计范式/Workflow vs Agent）、应用工程范式跃迁（Context/Harness/Loop Engineering 四层嵌套）、评估技术成熟等关键阶段 |
| [Spring AI 框架详解](./document/AI/Spring%20AI%20框架详解.md) | 深入解析 Spring AI 框架，包括核心架构设计、ChatClient API、RAG 检索增强生成、函数调用、向量数据库集成与实际应用场景 |
| [向量数据库选型与原理详解](./document/AI/向量数据库选型与原理详解.md) | 全面解析向量数据库核心原理（HNSW、IVF、PQ 索引算法），主流产品对比（Milvus、Qdrant、Chroma、Pinecone），选型决策树与 RAG 架构实践 |
| [AI 开发术语详解](./document/AI/AI%20开发术语详解.md) | 详解 AI 开发核心术语，包括机器学习评估指标（准确率、精确率、召回率、F1、ROC/AUC）、大模型开发术语（Token、Temperature、Top-k、Top-p、RAG）等 |
| [AI 大模型幻觉与输出稳定性解决方案](./document/AI/AI%20大模型幻觉与输出稳定性解决方案.md) | 深入解析大模型幻觉问题成因与分类、幻觉解决方案（RAG、CoT、RLHF、幻觉检测）、输出不稳定原因与解决方案（参数控制、提示工程、后处理校验） |

### SEO

| 文档 | 概述 |
|------|------|
| [SEO 搜索引擎优化详解](./document/SEO/SEO%20搜索引擎优化详解.md) | 全面解析 SEO 搜索引擎优化，包括搜索引擎工作原理（爬取、索引、排名）、三大核心支柱（技术 SEO、内容 SEO、外部 SEO）、关键词策略、E-E-A-T 原则、白帽与黑帽 SEO 对比、常用工具推荐 |

### 计算机网络

| 文档 | 概述 |
|------|------|
| [TCP 三次握手与四次挥手详解](./document/计算机网络/TCP%20三次握手与四次挥手详解.md) | 深入解析 TCP 连接建立与断开过程，包括三次握手流程与目的、四次挥手流程与原因、TCP 状态转换、TIME_WAIT 状态作用、半连接队列与全连接队列、SYN Flood 攻击原理与防御 |
| [网络体系结构总论](./document/计算机网络/基础网络技术/网络体系结构总论.md) | 系统梳理网络体系结构基础，涵盖 OSI 七层参考模型（ISO/IEC 7498-1）、TCP/IP 四层模型（RFC 1122）、五层混合模型对比、数据封装与解封装过程（PDU 层级）、分层架构优缺点与 OSI 历史启示 |
| [1）物理层与数据链路层](./document/计算机网络/基础网络技术/1）物理层与数据链路层.md) | 系统讲解物理层与数据链路层基础，涵盖信号编码（Manchester/4B/5B/8B/10B）、传输介质（UTP/光纤）、IEEE 802.3 以太网帧格式、CSMA/CD 机制、MAC 地址（EUI-48）、ARP 协议（RFC 826）、交换机转发原理、VLAN（802.1Q）、STP/RSTP/MSTP 生成树协议 |
| [2）网络层及IP协议](./document/计算机网络/基础网络技术/2）网络层及IP协议.md) | 系统解析网络层核心技术，涵盖 IPv4 首部（RFC 791）、IPv4 地址分类与私有地址（RFC 1918）、CIDR 与 VLSM（RFC 1519）、IP 分片、IPv6 首部与地址类型（RFC 8200）、IPv4/IPv6 对比、ICMP（RFC 792）、路由协议（RIP/OSPF/BGP）、NAT 网络地址转换（RFC 3022） |
| [3）传输层及UDP与TCP协议](./document/计算机网络/基础网络技术/3）传输层及UDP与TCP协议.md) | 系统解析传输层核心协议，涵盖端口复用、UDP 协议（RFC 768）与伪首部、TCP 协议（RFC 9293）与首部字段、TCP 可靠传输（序列号/确认/重传/SACK）、RTO 计算（RFC 6298）、流量控制（滑动窗口/Nagle 算法）、拥塞控制（慢开始/拥塞避免/快重传/快恢复，RFC 5681）、CUBIC 与 BBR 算法对比 |
| [4）应用层及核心协议](./document/计算机网络/基础网络技术/4）应用层及核心协议.md) | 系统梳理应用层核心协议，涵盖 HTTP 请求方法与状态码、HTTP 版本演进（HTTP/1.1→HTTP/2→HTTP/3）与队头阻塞、DNS 层次结构与解析流程（RFC 1034/1035）、DHCP DORA 四步流程与租约管理（RFC 2131）、SMTP/POP3/IMAP 邮件协议对比、FTP 主动/被动模式 |
| [SDN 与 NFV 网络架构演进详解](./document/计算机网络/现代网络技术/SDN%20与%20NFV%20网络架构演进详解.md) | 系统解析 SDN 软件定义网络与 NFV 网络功能虚拟化架构，涵盖 ONF 三层架构、OpenFlow 协议演进、主流 SDN 控制器、ETSI NFV 参考架构与 MANO、VNF→CNF 演进、P4 可编程数据面、白盒交换机、SD-WAN、SRv6 与 EVPN-VXLAN |
| [云原生网络技术详解](./document/计算机网络/现代网络技术/云原生网络技术详解.md) | 深入解析云原生网络技术栈，包括 Kubernetes 网络模型与 CNI 规范、eBPF/XDP 数据路径、kube-proxy 三种模式（iptables/IPVS/nftables）、NetworkPolicy、Service Mesh（Istio Ambient 与 Cilium）、Gateway API、K8s 网络演进趋势 |
| [5G 网络架构与关键技术详解](./document/计算机网络/现代网络技术/5G%20网络架构与关键技术详解.md) | 系统解析 5G 网络架构，涵盖 ITU IMT-2020 三大场景（eMBB/URLLC/mMTC）、3GPP Rel-15~18 标准演进、SBA 服务化架构与 NF 清单、NSA Option 3x 与 SA Option 2 组网对比、CU-DU 切分、网络切片（S-NSSAI/5QI）、MEC 多接入边缘计算 |
| [零信任与 SASE 安全架构详解](./document/计算机网络/现代网络技术/零信任与%20SASE%20安全架构详解.md) | 深入解析零信任与 SASE 安全架构，涵盖 NIST SP 800-207 七大原则与 PE/PA/PEP 逻辑组件、信任算法、三种部署模式、Google BeyondCorp 系列实践、SASE（SD-WAN+SSE）融合架构、ZTNA 1.0 vs 2.0、mTLS、微分段、SPIFFE/SPIRE、CISA ZTMM 成熟度模型 |
| [高性能网络技术详解](./document/计算机网络/现代网络技术/高性能网络技术详解.md) | 系统解析高性能网络技术栈，涵盖 RDMA（InfiniBand/RoCEv2/iWARP）内核旁路与零拷贝、Verbs API 编程模型、DPDK PMD 轮询与 VFIO/HugePages、SmartNIC 与 DPU 硬件卸载架构、SR-IOV、XDP/AF_XDP、io_uring、VPP 向量化处理、技术性能对比与选型矩阵 |
| [HTTPS 原理与 TLS 握手详解](./document/计算机网络/HTTPS%20原理与%20TLS%20握手详解.md) | 系统梳理 HTTPS 核心技术体系，涵盖密码学基础（对称/非对称加密、哈希、数字签名）、PKI 证书体系（X.509、CA 层级、DV/OV/EV 证书、信任链验证）、TLS 1.2 与 1.3 握手流程对比、ECDHE 密钥交换与前向保密原理、HKDF 密钥派生、HSTS 安全实践、常见攻击与防御、后量子密码学趋势 |

### 性能优化

| 文档 | 概述 |
|------|------|
| [并发性能指标详解](./document/性能优化/并发性能指标详解.md) | 系统介绍并发性能指标体系，包括吞吐量指标（QPS、TPS）、响应时间指标（RT、P99）、并发指标、业务流量指标（PV、UV、DAU）、系统资源指标，详解利特尔法则与性能优化方向 |

### 工具

| 文档 | 概述 |
|------|------|
| [Arthas Java诊断工具详解](./document/工具/Arthas%20Java诊断工具详解.md) | 深入解析阿里巴巴开源的 Java 诊断工具 Arthas，包括核心原理（Java Agent、Instrumentation、字节码增强）、常用命令详解（dashboard、thread、trace、watch、jad 等）、实战案例（CPU 飙高、接口慢、死锁排查）与最佳实践 |

### Linux

| 文档 | 概述 |
|------|------|
| [Linux命令体系总论](./document/Linux/命令/Linux命令体系总论.md) | 系统梳理 Linux 命令体系基础，涵盖 Shell 类型与运行模式（交互/非交互、登录/非登录）、内建命令与外部命令、命令语法结构与查找机制、五种帮助方式（man/info/help/--help/tldr）、通配符与正则对比、标准 I/O 流与重定向管道、退出状态码、命令历史与别名、Bash 命令解析 12 阶段流水线 |
| [文件与目录管理命令](./document/Linux/命令/文件与目录管理命令.md) | 全面梳理 Linux 文件系统操作命令，涵盖文件系统层次标准（FHS）、目录导航与浏览（pwd、cd、ls、tree）、文件操作（touch、cp、mv、rm、mkdir、rmdir）、文件查找（find、locate、which、whereis、type）、文件权限与属性（chmod、chown、chgrp、umask、SUID/SGID/Sticky、chattr）、磁盘空间（df、du）、归档压缩（tar、gzip、bzip2、xz、zip）及 inode 与软硬链接原理 |
| [文本处理与查看命令](./document/Linux/命令/文本处理与查看命令.md) | 系统讲解 Linux 文本处理工具体系，涵盖内容查看（cat、tac、nl、more、less、head、tail）、文本处理三剑客（grep 模式匹配、sed 流编辑、awk 字段分析）、排序去重与截取（sort、uniq、cut、paste、tr、wc、split）及综合实战案例 |
| [系统与进程管理命令](./document/Linux/命令/系统与进程管理命令.md) | 系统讲解进程基础概念与状态机、进程查看（ps、top、htop）、进程控制与信号机制（kill、pkill、killall）、定时任务（crontab）、systemd 服务管理、系统资源监控、用户管理及进程诊断工具 |
| [网络与磁盘管理命令](./document/Linux/命令/网络与磁盘管理命令.md) | 系统讲解网络配置（ip/ifconfig/nmcli）、网络诊断（ping/traceroute/arping）、连接查看（ss/netstat）、数据传输（curl/wget/scp/rsync）、网络工具（nc/nmap/dig）、防火墙（iptables 四表五链/firewalld/ufw）、SSH 远程管理、包捕获（tcpdump/iperf3）、磁盘分区（fdisk/parted/gdisk）、文件系统管理（mkfs/fsck/tune2fs/mount/fstab）、磁盘空间（df/du/ncdu）、LVM 逻辑卷管理（PV/VG/LV/在线扩容/快照）、I/O 性能监控（iostat/iotop）及 Swap 管理 |

## 目录结构

```
document/
├── Java/
│   ├── Java HashMap 扩容机制详解.md
│   ├── Java WebSocket 开发详解.md
│   └── Java 单例模式详解.md
├── Spring/
│   ├── Spring Bean 作用域详解.md
│   ├── Spring 容器启动流程详解.md
│   ├── Spring 循环依赖解决方案详解.md
│   ├── Spring 事务管理机制详解.md
│   └── SpringMVC 核心原理详解.md
├── MySQL/
│   ├── MySQL MVCC 多版本并发控制机制详解.md
│   ├── MySQL Binlog 二进制日志详解.md
│   ├── MySQL 三大日志详解.md
│   ├── MySQL 多粒度锁机制详解.md
│   ├── MySQL 索引原理与失效场景详解.md
│   └── MySQL 集群架构详解.md
├── PostgreSQL/
│   ├── PostgreSQL 分区表最佳实践与底层原理.md
│   ├── PostgreSQL 分级锁机制详解.md
│   └── PostgreSQL 锁机制与分区表操作详解.md
├── ClickHouse/
│   ├── ClickHouse 列式数据库详解.md
│   └── ClickHouse 向量化执行技术详解.md
├── 数据库/
│   └── OLTP与OLAP数据处理架构详解.md
├── ElasticSearch/
│   ├── ElasticSearch 修改现有索引 Settings 详解.md
│   ├── ElasticSearch 分片数量设置最佳实践详解.md
│   ├── ElasticSearch 常用分词器详解.md
│   ├── ElasticSearch 底层存储原理详解.md
│   ├── ElasticSearch 教程.md
│   ├── ElasticSearch 数值类型索引结构详解.md
│   ├── ElasticSearch 索引一致性参数详解.md
│   ├── ElasticSearch 索引分片设置最佳实践.md
│   ├── ElasticSearch 集群状态详解.md
│   ├── ElasticSearch 集群管理完全指南.md
│   └── ElasticSearch 集群部署最佳实践.md
├── Redis/
│   ├── Redis HyperLogLog 详解.md
│   ├── Redis Pipeline 管道技术详解.md
│   ├── Redis 批量数据原子性操作解决方案.md
│   └── Redis 缓存穿透击穿雪崩详解.md
├── 消息中间件/
│   ├── 消息中间件 MQ 对比详情.md
│   └── RocketMQ 消息中间件详解.md
├── 微服务架构/
│   ├── Seata 分布式事务详解.md
│   ├── Nacos 注册配置中心详解.md
│   ├── SkyWalking 分布式链路追踪详解.md
│   ├── Sentinel 流量控制与限流算法详解.md
│   └── Spring Cloud Alibaba 面试项目推荐方案.md
├── Docker/
│   └── Docker 容器技术详解.md
├── DevOps/
│   └── CI-CD 持续集成与持续交付详解.md
├── Kubernetes/
│   ├── Kubernetes 技术体系总论.md
│   ├── Kubernetes 集群架构与核心组件详解.md
│   ├── Kubernetes 工作负载与核心对象详解.md
│   ├── Kubernetes 服务路由与存储体系详解.md
│   ├── Kubernetes 调度与资源管理详解.md
│   ├── Kubernetes 安全与访问控制详解.md
│   └── Kubernetes 集群部署与生产运维实践详解.md
├── 大数据/
│   └── 大数据开发技术发展历程.md
├── AI/
│   ├── AI 大模型技术发展历程.md
│   ├── AI大模型应用技术发展历程.md
│   ├── AI 大模型幻觉与输出稳定性解决方案.md
│   ├── AI 开发术语详解.md
│   ├── Spring AI 框架详解.md
│   └── 向量数据库选型与原理详解.md
├── SEO/
│   └── SEO 搜索引擎优化详解.md
├── 计算机网络/
│   ├── TCP 三次握手与四次挥手详解.md
│   ├── HTTPS 原理与 TLS 握手详解.md
│   ├── 基础网络技术/
│   │   ├── 网络体系结构总论.md
│   │   ├── 1）物理层与数据链路层.md
│   │   ├── 2）网络层及IP协议.md
│   │   ├── 3）传输层及UDP与TCP协议.md
│   │   └── 4）应用层及核心协议.md
│   └── 现代网络技术/
│       ├── SDN 与 NFV 网络架构演进详解.md
│       ├── 云原生网络技术详解.md
│       ├── 5G 网络架构与关键技术详解.md
│       ├── 零信任与 SASE 安全架构详解.md
│       └── 高性能网络技术详解.md
├── 性能优化/
│   └── 并发性能指标详解.md
├── 工具/
│   └── Arthas Java诊断工具详解.md
└── Linux/
    └── 命令/
        ├── Linux命令体系总论.md
        ├── 文件与目录管理命令.md
        ├── 文本处理与查看命令.md
        ├── 系统与进程管理命令.md
        └── 网络与磁盘管理命令.md
```
