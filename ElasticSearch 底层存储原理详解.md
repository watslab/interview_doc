# ElasticSearch 底层存储原理详解

## 概述

ElasticSearch 基于 Apache Lucene 构建，其底层存储是一个设计精妙的系统，融合了多种数据结构和存储技术。

```
┌─────────────────────────────────────────────────────────────┐
│                  ElasticSearch 存储架构                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    Index（索引）                      │   │
│   │  ┌───────────────────────────────────────────────┐  │   │
│   │  │              Shard（分片）                      │  │   │
│   │  │  ┌─────────────────────────────────────────┐  │  │   │
│   │  │  │           Lucene Index                   │  │  │   │
│   │  │  │  ┌─────────────────────────────────┐    │  │  │   │
│   │  │  │  │    Segment（段）                 │    │  │  │   │
│   │  │  │  │  ┌───────────────────────────┐  │    │  │  │   │
│   │  │  │  │  │ 倒排索引  Doc Values  BKD │  │    │  │  │   │
│   │  │  │  │  │   FST      列式存储    树  │  │    │  │  │   │
│   │  │  │  │  └───────────────────────────┘  │    │  │  │   │
│   │  │  │  └─────────────────────────────────┘    │  │  │   │
│   │  │  └─────────────────────────────────────────┘  │  │   │
│   │  └───────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 一、核心数据结构

### 1.1 倒排索引（Inverted Index）

倒排索引是 ElasticSearch 实现全文搜索的核心数据结构。

#### 结构组成

```
┌─────────────────────────────────────────────────────────────┐
│                      倒排索引结构                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Term Dictionary        Posting List                       │
│   （词典）                （倒排列表）                        │
│   ┌──────────┐           ┌─────────────────────────────┐   │
│   │  apple   │ ────────► │ Doc1, Doc3, Doc5            │   │
│   ├──────────┤           ├─────────────────────────────┤   │
│   │  banana  │ ────────► │ Doc2, Doc4                  │   │
│   ├──────────┤           ├─────────────────────────────┤   │
│   │  orange  │ ────────► │ Doc1, Doc2, Doc3, Doc4, Doc5│   │
│   └──────────┘           └─────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 倒排列表存储内容

| 内容 | 说明 |
|------|------|
| Document ID | 文档编号 |
| TF（Term Frequency） | 词频，该词在文档中出现次数 |
| Position | 词在文档中的位置 |
| Offset | 词在文档中的偏移量（起始和结束位置） |
| Payload | 自定义元数据 |

#### 示例

文档内容：
- Doc1: "Elasticsearch is powerful"
- Doc2: "Elasticsearch is fast"

倒排索引：

| Term | Posting List |
|------|--------------|
| elasticsearch | [(Doc1, TF=1, Pos=0), (Doc2, TF=1, Pos=0)] |
| is | [(Doc1, TF=1, Pos=1), (Doc2, TF=1, Pos=1)] |
| powerful | [(Doc1, TF=1, Pos=2)] |
| fast | [(Doc2, TF=1, Pos=2)] |

---

### 1.2 Doc Values（列式存储）

Doc Values 是 ElasticSearch 的列式存储结构，用于排序、聚合和脚本访问。

#### 特点

| 特性 | 说明 |
|------|------|
| 列式存储 | 同一字段的值连续存储 |
| 压缩存储 | 使用多种压缩算法 |
| 内存映射 | 通过 mmap 加速访问 |
| 适合聚合 | 排序、聚合性能优秀 |

#### 与倒排索引对比

| 对比项 | 倒排索引 | Doc Values |
|--------|----------|------------|
| 存储方式 | Term → DocID | DocID → Value |
| 查询方式 | 通过词找文档 | 通过文档找值 |
| 适用场景 | 全文搜索、精确匹配 | 排序、聚合、脚本 |
| 压缩方式 | FST、FOR | 列式压缩 |

#### 存储示例

```
文档数据：
Doc1: { "name": "张三", "age": 25 }
Doc2: { "name": "李四", "age": 30 }
Doc3: { "name": "王五", "age": 28 }

Doc Values 存储（列式）：
name 列: [张三, 李四, 王五]
age 列: [25, 30, 28]
```

---

### 1.3 BKD 树（Block K-D Tree）

BKD 树用于数值类型和地理坐标的索引存储。

#### 特点

- 专为范围查询优化
- 支持多维数据（地理坐标）
- 空间效率高
- 查询时间复杂度 O(log N)

#### 适用数据类型

| 数据类型 | 索引结构 |
|----------|----------|
| integer, long, float, double | BKD 树 |
| date | BKD 树 |
| geo_point, geo_shape | BKD 树 |
| text, keyword | 倒排索引 |

---

## 二、存储单元：Segment（段）

### 2.1 Segment 概念

Segment 是 Lucene 中最小的存储单元，每个 Segment 本身就是一个独立的倒排索引。

```
┌─────────────────────────────────────────────────────────────┐
│                    Index 存储结构                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Index（ES索引）                                            │
│       │                                                      │
│       ├── Shard 0（分片）                                    │
│       │       │                                              │
│       │       ├── Segment 1（段）                            │
│       │       │     ├── .si（段信息）                        │
│       │       │     ├── .fnm（字段信息）                     │
│       │       │     ├── .tim（词典）                         │
│       │       │     ├── .tip（词典索引）                     │
│       │       │     ├── .doc（倒排列表）                     │
│       │       │     ├── .dvd（Doc Values数据）               │
│       │       │     └── .dvm（Doc Values元数据）             │
│       │       │                                              │
│       │       ├── Segment 2                                  │
│       │       └── Segment 3                                  │
│       │                                                      │
│       └── Commit Point（提交点）                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Segment 文件类型

| 文件扩展名 | 说明 |
|------------|------|
| `.si` | Segment 信息文件 |
| `.fnm` | 字段名称和元数据 |
| `.tim` | Term Dictionary（词典） |
| `.tip` | Term Index（词典索引，FST结构） |
| `.doc` | Posting List（倒排列表） |
| `.pos` | Position信息 |
| `.dvd` | Doc Values 数据 |
| `.dvm` | Doc Values 元数据 |
| `.fdt` | Stored Fields 数据 |
| `.fdx` | Stored Fields 索引 |

### 2.3 Segment 特性

| 特性 | 说明 |
|------|------|
| 不可变性 | Segment 一旦创建就不可修改 |
| 写入优化 | 追加写入，性能高 |
| 合并机制 | 后台自动合并小 Segment |
| 缓存友好 | 可被文件系统缓存 |

---

## 三、写入流程

### 3.1 写入流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    ES 写入流程                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   写入请求                                                   │
│       │                                                      │
│       ▼                                                      │
│   ┌─────────────┐                                           │
│   │ 写入内存缓冲 │  ← Memory Buffer（索引缓冲区）            │
│   │ (In-memory  │                                           │
│   │  Buffer)    │                                           │
│   └──────┬──────┘                                           │
│          │                                                   │
│          │ 同时                                              │
│          ▼                                                   │
│   ┌─────────────┐                                           │
│   │ 写入Translog │  ← Translog（事务日志，持久化保证）        │
│   │ (预写日志)   │                                           │
│   └──────┬──────┘                                           │
│          │                                                   │
│          │ refresh（默认1秒）                                │
│          ▼                                                   │
│   ┌─────────────┐                                           │
│   │ 生成Segment │  ← 新 Segment 在内存中，可被搜索           │
│   │ (In-memory  │                                           │
│   │  Segment)   │                                           │
│   └──────┬──────┘                                           │
│          │                                                   │
│          │ flush（默认30分钟或Translog达到512MB）            │
│          ▼                                                   │
│   ┌─────────────┐                                           │
│   │ Segment持久 │  ← Segment 写入磁盘                       │
│   │ 化到磁盘     │    清空 Translog                          │
│   │             │    更新 Commit Point                      │
│   └─────────────┘                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 写入步骤详解

#### Step 1：写入内存缓冲区

```
客户端写入请求 → Memory Buffer（索引缓冲区）
```

- 数据首先写入内存缓冲区
- 同时写入 Translog（保证数据不丢失）

#### Step 2：Refresh（刷新）

```
Memory Buffer → In-memory Segment → 可被搜索
```

- **默认间隔**：1 秒
- **作用**：将内存缓冲区数据生成新的 Segment
- **结果**：数据可被搜索，但仍在内存中

```json
// 修改 refresh 间隔
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "5s"
  }
}
```

#### Step 3：Flush（刷盘）

```
In-memory Segment → Disk Segment + Clear Translog + Commit Point
```

- **触发条件**：
  - 默认每 30 分钟
  - Translog 达到 512MB
- **操作**：
  1. 将内存中的 Segment 写入磁盘
  2. 清空 Translog
  3. 更新 Commit Point

#### Step 4：Segment Merge（段合并）

```
多个小 Segment → 合并 → 大 Segment
```

- **作用**：减少 Segment 数量，优化查询性能
- **触发**：后台自动执行
- **策略**：TieredMergePolicy（分层合并策略）

---

## 四、Translog（事务日志）

### 4.1 Translog 作用

| 作用 | 说明 |
|------|------|
| 数据持久化 | 保证写入数据不丢失 |
| 崩溃恢复 | 节点重启后可从 Translog 恢复 |
| 实时性 | 写入即可返回成功 |

### 4.2 Translog 配置

```yaml
# elasticsearch.yml

# Translog 持久化策略
index.translog.durability: request  # 每次请求都刷盘（安全但慢）
# index.translog.durability: async  # 异步刷盘（快但有风险）

# Translog 刷盘间隔
index.translog.sync_interval: 5s

# Translog 大小阈值
index.translog.flush_threshold_size: 512mb
```

### 4.3 持久化策略对比

| 策略 | 说明 | 性能 | 安全性 |
|------|------|------|--------|
| `request` | 每次请求都 fsync | 低 | 高 |
| `async` | 每隔 sync_interval fsync | 高 | 较低 |

---

## 五、压缩技术

### 5.1 FST（Finite State Transducer）

FST 用于压缩词典（Term Dictionary）。

#### 特点

| 特性 | 说明 |
|------|------|
| 压缩率高 | 内存占用降低 70%+ |
| 前缀共享 | 相同前缀只存储一次 |
| 快速查找 | O(1) 时间复杂度 |
| 支持前缀查询 | 天然支持前缀匹配 |

#### 示例

```
词典词项：apple, apply, application, approach

FST 结构（共享前缀）：
        a
        │
        p
        │
        p
       /│\
      l o l
     /│  │
    y i  e
      │  │
      c  │
      │  │
      a  │
      │  │
      t  │
      │  │
      i  │
      │  │
      o  │
      │  │
      n  │
```

### 5.2 FOR（Frame Of Reference）

FOR 用于压缩倒排列表（Posting List）。

#### 原理

- 将文档 ID 分组（每 128 个一组）
- 每组使用最小位数存储
- 差值编码（存储与前一个 ID 的差值）

#### 示例

```
原始 DocID: [100, 101, 105, 109, 110, 120]

差值编码: [100, 1, 4, 4, 1, 10]

分组压缩:
Block 1: [100] → 需要 7 bits
Block 2: [1, 4, 4, 1, 10] → 最大值 10，需要 4 bits
```

### 5.3 跳表（Skip List）

跳表用于加速倒排列表的遍历和合并。

```
Level 3:     [Head]─────────────────────────────►[100]
Level 2:     [Head]─────►[25]─────►[50]────────►[100]
Level 1:     [Head]►[10]►[25]►[35]►[50]►[75]──►[100]
Level 0:     [Head]►[5]►[10]►[15]►[25]►[35]►[50]►[75]►[100]
```

### 5.4 Bitset（位图）

位图用于快速合并多个查询结果。

```
DocID 集合: {1, 3, 5, 7, 9}

Bitset: 0 1 0 1 0 1 0 1 0 1
        0 1 2 3 4 5 6 7 8 9
```

---

## 六、存储文件结构

### 6.1 磁盘文件目录

```
/var/lib/elasticsearch/nodes/0/indices/
└── my_index/
    └── 0/                          # 分片编号
        ├── index/
        │   ├── segments_1          # Commit Point 文件
        │   ├── segments_2
        │   ├── _0.cfe              # 复合文件入口
        │   ├── _0.cfs              # 复合文件（包含多个段文件）
        │   ├── _0.si               # 段信息
        │   ├── _1.cfe
        │   ├── _1.cfs
        │   └── _1.si
        ├── translog/
        │   ├── translog-1.tlog     # Translog 文件
        │   └── translog-1.ckp      # Checkpoint
        └── _state/
            └── state-1.st          # 分片状态
```

### 6.2 Commit Point

Commit Point 记录当前所有有效的 Segment。

```
┌─────────────────────────────────────────────────────────────┐
│                    Commit Point                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   segments_1                                                 │
│   ├── Segment: _0  (size: 10MB, docs: 1000)                 │
│   ├── Segment: _1  (size: 15MB, docs: 1500)                 │
│   └── Segment: _2  (size: 8MB,  docs: 800)                  │
│                                                              │
│   每次 Flush 后更新 Commit Point                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、查询流程

### 7.1 查询执行过程

```
┌─────────────────────────────────────────────────────────────┐
│                    查询执行流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   1. 协调节点接收请求                                        │
│          │                                                   │
│          ▼                                                   │
│   2. 解析查询，确定目标分片                                  │
│          │                                                   │
│          ▼                                                   │
│   3. 向所有相关分片发送查询请求                              │
│          │                                                   │
│          ▼                                                   │
│   4. 各分片执行本地查询                                      │
│          │                                                   │
│          ├── 4.1 查询 FST 定位 Term                          │
│          │                                                   │
│          ├── 4.2 读取 Posting List                           │
│          │                                                   │
│          ├── 4.3 合并多个 Segment 结果                       │
│          │                                                   │
│          └── 4.4 计算相关性得分                              │
│          │                                                   │
│          ▼                                                   │
│   5. 协调节点收集并合并结果                                  │
│          │                                                   │
│          ▼                                                   │
│   6. 返回最终结果                                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 查询类型

| 查询类型 | 说明 | 使用的数据结构 |
|----------|------|----------------|
| 全文搜索 | match, match_phrase | 倒排索引 |
| 精确匹配 | term, terms | 倒排索引 |
| 范围查询 | range | BKD 树 |
| 地理查询 | geo_distance | BKD 树 |
| 排序 | sort | Doc Values |
| 聚合 | aggregation | Doc Values |

---

## 八、存储优化建议

### 8.1 索引设计优化

| 优化项 | 建议 |
|--------|------|
| 分片数 | 每个节点不超过 20 个分片/GB 堆内存 |
| 副本数 | 生产环境至少 1 个副本 |
| refresh_interval | 批量写入时设置为 30s 或 -1 |
| 批量写入 | 使用 Bulk API，每批 5-15MB |

### 8.2 存储配置优化

```yaml
# 禁用不需要的功能
index:
  number_of_replicas: 1
  refresh_interval: 30s
  
# 关闭不需要的字段索引
mappings:
  properties:
    content:
      type: text
      index: true           # 需要搜索
      norms: false          # 不需要评分
      doc_values: false     # 不需要排序/聚合
    
    timestamp:
      type: date
      doc_values: true      # 需要排序
```

### 8.3 Segment 合并优化

```yaml
# 合并策略配置
index.merge.policy.type: tiered
index.merge.policy.segments_per_tier: 10
index.merge.policy.max_merged_segment: 5gb

# 合并调度器
index.merge.scheduler.max_thread_count: 1
```

---

## 九、总结

### 核心存储组件

| 组件 | 作用 | 数据结构 |
|------|------|----------|
| 倒排索引 | 全文搜索 | Term → Posting List |
| Doc Values | 排序、聚合 | 列式存储 |
| BKD 树 | 范围查询 | 多维空间划分 |
| Segment | 存储单元 | 不可变文件 |
| Translog | 数据持久化 | 预写日志 |
| FST | 词典压缩 | 有限状态转换器 |

### 写入流程总结

```
写入 → Memory Buffer + Translog → Refresh(1s) → In-memory Segment
    → Flush(30min) → Disk Segment → Segment Merge
```

---

## 参考资料

- [Elasticsearch底层存储原理](https://blog.csdn.net/HTTP404_CN/article/details/150638739)
- [Elasticsearch 写入全链路:从单机到集群](https://blog.csdn.net/Daoxifu/article/details/150558953)
- [Elasticsearch 倒排索引原理](https://blog.csdn.net/qq_41187124/article/details/154542166)
- [Elasticsearch数据类型及其底层Lucene数据结构](https://blog.csdn.net/qq_29328443/article/details/151290494)
- [Elasticsearch 完全指南:原理、优势与应用场景](https://blog.csdn.net/u011265143/article/details/155396183)
