# Interview Doc

技术面试相关文档整理，涵盖数据库、搜索引擎、缓存、编程语言等核心技术领域。

## 文档列表

### Java

| 文档 | 概述 |
|------|------|
| [Java HashMap 扩容机制详解](./Java%20HashMap%20扩容机制详解.md) | 深入解析 HashMap 扩容机制，包括触发条件、元素迁移流程、JDK 1.7 与 1.8 核心差异对比、面试高频问题总结 |
| [Java 单例模式详解](./Java%20单例模式详解.md) | 详解单例模式的四种实现方式：饿汉模式、懒汉模式（单线程/多线程/双重检查锁定），分析线程安全与性能权衡 |

### Spring

| 文档 | 概述 |
|------|------|
| [Spring Bean 作用域详解](./Spring%20Bean%20作用域详解.md) | 详解 Spring Bean 的六种作用域，重点分析单例 Bean 依赖原型 Bean 的问题与四种解决方案 |
| [Spring 事务管理机制详解](./Spring事务管理机制详解.md) | 深入解析 Spring 事务管理机制，包括事务传播行为、隔离级别、AOP 动态代理实现原理、@Transactional 注解详解、事务失效场景分析、跨线程事务解决方案 |

### MySQL

| 文档 | 概述 |
|------|------|
| [MySQL MVCC 多版本并发控制机制详解](./MySQL%20MVCC%20多版本并发控制机制详解.md) | 深入讲解 MySQL InnoDB 的 MVCC 机制，包括隐藏列、Undo Log、Read View、版本可见性判断规则，以及 MVCC 与事务隔离级别的关系、Next-Key Lock 解决幻读问题等 |
| [MySQL 多粒度锁机制详解](./MySQL%20多粒度锁机制详解.md) | 详解 MySQL InnoDB 的多粒度锁机制，包括表锁、行锁、意向锁（IS/IX）的工作原理与兼容性矩阵，以及意向锁与 MVCC 的协作关系 |
| [MySQL 三大日志详解](./MySQL三大日志详解.md) | 深入解析 MySQL Binlog、Redo Log、Undo Log 三大日志系统，包括核心原理、存储结构、刷盘策略、两阶段提交机制、崩溃恢复流程与最佳实践 |

### PostgreSQL

| 文档 | 概述 |
|------|------|
| [PostgreSQL 分区表最佳实践与底层原理](./PostgreSQL%20分区表最佳实践与底层原理.md) | 全面介绍 PostgreSQL 分区表技术，包括声明式分区与继承分区的实现原理、分区策略选择、最佳实践与性能优化建议 |
| [PostgreSQL 分级锁机制详解](./PostgreSQL%20分级锁机制详解.md) | 深入解析 PostgreSQL 的三级锁体系：Spin Lock、Lightweight Lock、Regular Lock，以及 8 种表级锁的兼容性与应用场景 |

### ElasticSearch

| 文档 | 概述 |
|------|------|
| [ElasticSearch 集群部署最佳实践](./ElasticSearch%20集群部署最佳实践.md) | ElasticSearch 集群部署的架构设计、硬件配置、JVM 调优等最佳实践指南 |
| [ElasticSearch 集群管理完全指南](./ElasticSearch%20集群管理完全指南.md) | ElasticSearch 集群的日常管理操作，包括节点管理、索引管理、集群监控与故障处理 |
| [ElasticSearch 集群状态详解](./ElasticSearch%20集群状态详解.md) | 深入解析 ElasticSearch 集群状态机制，包括状态同步流程、常见状态问题排查 |
| [ElasticSearch 索引分片设置最佳实践](./ElasticSearch%20索引分片设置最佳实践.md) | ElasticSearch 索引分片数量与大小的最佳设置策略，平衡性能与资源利用率 |
| [ElasticSearch 分片数量设置最佳实践详解](./ElasticSearch%20分片数量设置最佳实践详解.md) | 详细讲解 ElasticSearch 分片数量的设置原则、影响因素与优化建议 |
| [ElasticSearch 索引一致性参数详解](./ElasticSearch%20索引一致性参数详解.md) | ElasticSearch 索引一致性相关参数的配置与调优，包括 refresh、flush、sync 等机制 |
| [ElasticSearch 修改现有索引 Settings 详解](./ElasticSearch%20修改现有索引%20Settings%20详解.md) | 如何安全地修改 ElasticSearch 现有索引的配置参数，包括动态设置与静态设置的区别 |
| [ElasticSearch 底层存储原理详解](./ElasticSearch%20底层存储原理详解.md) | ElasticSearch 底层存储架构解析，包括 Lucene 段文件、倒排索引、存储结构等 |
| [ElasticSearch 数值类型索引结构详解](./ElasticSearch%20数值类型索引结构详解.md) | ElasticSearch 数值类型的索引结构原理，包括 BKD 树、数值排序与范围查询优化 |
| [ElasticSearch 常用分词器详解](./ElasticSearch%20常用分词器详解.md) | ElasticSearch 常用分词器的原理与使用场景，包括标准分词器、IK 分词器等 |
| [ElasticSearch教程](./ElasticSearch教程.md) | ElasticSearch 入门教程，涵盖基础概念、CRUD 操作、查询语法等内容 |

### Redis

| 文档 | 概述 |
|------|------|
| [Redis 抢购票据解决方案](./Redis%20抢购票据解决方案.md) | 基于 Redis 的高并发抢购/抢票场景解决方案，包括原子操作、分布式锁、限流策略等 |
| [Redis 批量数据原子性操作解决方案](./Redis%20批量数据原子性操作解决方案.md) | Redis 批量数据操作的原子性保证方案，包括 Pipeline、Lua 脚本、事务等技术的应用 |
| [Redis 缓存穿透击穿雪崩详解](./Redis缓存穿透击穿雪崩详解.md) | 深入解析 Redis 缓存三大问题，包括穿透、击穿、雪崩的场景分析、解决方案对比与最佳实践 |
| [Redis Pipeline 管道技术详解](./Redis%20Pipeline管道技术详解.md) | 深入解析 Redis Pipeline 管道技术，包括工作原理、与事务和 Lua 脚本的对比、适用场景、实战示例与性能优化建议 |

### 消息中间件MQ

| 文档 | 概述 |
|------|------|
| [消息中间件 MQ 对比详情](./消息中间件MQ对比详情.md) | 全面对比 ActiveMQ、RabbitMQ、Kafka、RocketMQ、Pulsar 五大消息中间件，包括架构图、工作流程、性能指标、功能对比与选型建议 |
| [RocketMQ 消息中间件详解](./RocketMQ消息中间件详解.md) | 深入解析 RocketMQ 核心架构、消息模型、消息发送与消费流程、顺序消息、事务消息、延迟消息等高级特性与最佳实践 |

### 微服务架构

| 文档 | 概述 |
|------|------|
| [Sentinel 流量控制与限流算法详解](./Sentinel流量控制与限流算法详解.md) | 深入解析 Sentinel 流量控制组件，包括核心概念、责任链工作原理、流量控制策略、熔断降级机制，以及固定窗口、滑动窗口、漏桶、令牌桶四种限流算法详解 |

### Docker

| 文档 | 概述 |
|------|------|
| [Docker 容器技术详解](./Docker容器技术详解.md) | 全面介绍 Docker 容器技术，包括核心概念、底层原理、常用命令、Dockerfile 编写、Docker Compose 使用与实战案例 |

## 目录结构

```
interview_projects/
├── Java/
│   ├── Java HashMap 扩容机制详解.md
│   └── Java 单例模式详解.md
├── Spring/
│   ├── Spring Bean 作用域详解.md
│   └── Spring事务管理机制详解.md
├── MySQL/
│   ├── MySQL MVCC 多版本并发控制机制详解.md
│   ├── MySQL 多粒度锁机制详解.md
│   └── MySQL三大日志详解.md
├── PostgreSQL/
│   ├── PostgreSQL 分区表最佳实践与底层原理.md
│   └── PostgreSQL 分级锁机制详解.md
├── ElasticSearch/
│   ├── ElasticSearch 集群部署最佳实践.md
│   ├── ElasticSearch 集群管理完全指南.md
│   ├── ElasticSearch 集群状态详解.md
│   ├── ElasticSearch 索引分片设置最佳实践.md
│   ├── ElasticSearch 分片数量设置最佳实践详解.md
│   ├── ElasticSearch 索引一致性参数详解.md
│   ├── ElasticSearch 修改现有索引 Settings 详解.md
│   ├── ElasticSearch 底层存储原理详解.md
│   ├── ElasticSearch 数值类型索引结构详解.md
│   ├── ElasticSearch 常用分词器详解.md
│   └── ElasticSearch教程.md
├── Redis/
│   ├── Redis 抢购票据解决方案.md
│   ├── Redis 批量数据原子性操作解决方案.md
│   ├── Redis缓存穿透击穿雪崩详解.md
│   └── Redis Pipeline管道技术详解.md
├── 消息中间件MQ/
│   ├── 消息中间件MQ对比详情.md
│   └── RocketMQ消息中间件详解.md
├── 微服务架构/
│   └── Sentinel流量控制与限流算法详解.md
├── Docker/
│   └── Docker容器技术详解.md
├── .gitignore
└── README.md
```
