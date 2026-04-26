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
| [MySQL 集群架构详解](./document/MySQL/MySQL%20集群架构详解.md) | 系统介绍 MySQL 集群架构，包括主从复制架构、高可用方案（MHA、MGR、InnoDB Cluster）、分布式集群方案（NDB Cluster、PXC/Galera Cluster），详解认证复制原理、GCache 机制、状态同步方式 |

### PostgreSQL

| 文档 | 概述 |
|------|------|
| [PostgreSQL 分区表最佳实践与底层原理](./document/PostgreSQL/PostgreSQL%20分区表最佳实践与底层原理.md) | 全面介绍 PostgreSQL 分区表技术，包括声明式分区与继承分区的实现原理、分区策略选择、最佳实践与性能优化建议 |
| [PostgreSQL 分级锁机制详解](./document/PostgreSQL/PostgreSQL%20分级锁机制详解.md) | 深入解析 PostgreSQL 的三级锁体系：Spin Lock、Lightweight Lock、Regular Lock，以及 8 种表级锁的兼容性与应用场景 |
| [PostgreSQL 锁机制与分区表操作详解](./document/PostgreSQL/PostgreSQL%20锁机制与分区表操作详解.md) | PostgreSQL 锁机制与分区表操作详解 |

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
| [Redis 抢购票据解决方案](./document/Redis/Redis%20抢购票据解决方案.md) | 基于 Redis 的高并发抢购/抢票场景解决方案，包括原子操作、分布式锁、限流策略等 |
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

### AI

| 文档 | 概述 |
|------|------|
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
│   ├── MySQL 三大日志详解.md
│   ├── MySQL 多粒度锁机制详解.md
│   └── MySQL 集群架构详解.md
├── PostgreSQL/
│   ├── PostgreSQL 分区表最佳实践与底层原理.md
│   ├── PostgreSQL 分级锁机制详解.md
│   └── PostgreSQL 锁机制与分区表操作详解.md
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
│   ├── Redis 抢购票据解决方案.md
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
├── AI/
│   ├── AI 开发术语详解.md
│   ├── Spring AI 框架详解.md
│   └── 向量数据库选型与原理详解.md
├── SEO/
│   └── SEO 搜索引擎优化详解.md
└── 计算机网络/
    └── TCP 三次握手与四次挥手详解.md
```
