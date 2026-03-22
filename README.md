# Interview Doc

技术面试相关文档整理，涵盖数据库、搜索引擎、缓存等核心技术领域。

## 文档列表

### MySQL

| 文档 | 概述 |
|------|------|
| [MySQL MVCC 多版本并发控制机制详解](./MySQL%20MVCC%20多版本并发控制机制详解.md) | 深入讲解 MySQL InnoDB 的 MVCC 机制，包括隐藏列、Undo Log、Read View、版本可见性判断规则，以及 MVCC 与事务隔离级别的关系、Next-Key Lock 解决幻读问题等 |
| [MySQL 多粒度锁机制详解](./MySQL%20多粒度锁机制详解.md) | 详解 MySQL InnoDB 的多粒度锁机制，包括表锁、行锁、意向锁（IS/IX）的工作原理与兼容性矩阵，以及意向锁与 MVCC 的协作关系 |

### PostgreSQL

| 文档 | 概述 |
|------|------|
| [PostgreSQL 分区表最佳实践与底层原理](./PostgreSQL%20分区表最佳实践与底层原理.md) | 全面介绍 PostgreSQL 分区表技术，包括声明式分区与继承分区的实现原理、分区策略选择、最佳实践与性能优化建议 |
| [PostgreSQL 分级锁机制详解](./PostgreSQL%20分级锁机制详解.md) | 深入解析 PostgreSQL 的三级锁体系：Spin Lock、Lightweight Lock、Regular Lock，以及 8 种表级锁的兼容性与应用场景 |
| [PostgreSQL 锁机制与分区表操作详解](./PostgreSQL%20锁机制与分区表操作详解.md) | 探讨 PostgreSQL 锁机制在分区表操作中的应用，包括分区表上的锁行为、并发操作注意事项与最佳实践 |

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

## 目录结构

```
interview_projects/
├── MySQL/
│   ├── MySQL MVCC 多版本并发控制机制详解.md
│   └── MySQL 多粒度锁机制详解.md
├── PostgreSQL/
│   ├── PostgreSQL 分区表最佳实践与底层原理.md
│   ├── PostgreSQL 分级锁机制详解.md
│   └── PostgreSQL 锁机制与分区表操作详解.md
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
│   └── Redis 批量数据原子性操作解决方案.md
├── .gitignore
└── README.md
```
