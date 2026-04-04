# ElasticSearch 集群状态详解

## 一、集群状态检查接口

### 1. 查看集群健康状态

```json
GET /_cluster/health
```

**响应示例**：

```json
{
  "cluster_name": "my-application",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 10,
  "active_shards": 20,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 0,
  "number_of_in_flight_fetch": 0,
  "task_max_waiting_in_queue_millis": 0,
  "active_shards_percent_as_number": 100.0
}
```

### 2. 关键指标说明

| 指标 | 说明 |
|------|------|
| `status` | 集群状态：green/yellow/red |
| `number_of_nodes` | 节点总数 |
| `number_of_data_nodes` | 数据节点数 |
| `active_primary_shards` | 活跃的主分片数 |
| `active_shards` | 活跃的分片总数（含副本） |
| `unassigned_shards` | 未分配的分片数 |
| `initializing_shards` | 正在初始化的分片数 |
| `relocating_shards` | 正在迁移的分片数 |

### 3. 查看索引级别的健康状态

```json
GET /_cluster/health?level=indices
```

### 4. 查看分片级别的健康状态

```json
GET /_cluster/health?level=shards
```

### 5. 等待集群恢复到指定状态

```json
# 等待集群状态变为 yellow 或 green
GET /_cluster/health?wait_for_status=yellow&timeout=30s

# 等待集群状态变为 green
GET /_cluster/health?wait_for_status=green&timeout=60s
```

### 6. 查看未分配的分片

```json
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason
```

### 7. 查看分片分配原因

```json
GET /_cluster/allocation/explain
```

### 8. 查看具体索引的分片状态

```json
GET /_cluster/allocation/explain?index=problem_index
```

---

## 二、集群健康状态说明

### 状态分类

ElasticSearch 集群有三种健康状态：

| 状态 | 颜色 | 含义 | 分片情况 |
|------|------|------|----------|
| **Green** | 🟢 绿色 | 健康 | 所有主分片和副本分片都正常 |
| **Yellow** | 🟡 黄色 | 警告 | 所有主分片正常，部分副本分片不可用 |
| **Red** | 🔴 红色 | 异常 | 部分主分片不可用，数据丢失风险 |

---

## 三、各状态详解

### Green（绿色）- 健康

- 所有主分片（Primary Shard）和副本分片（Replica Shard）都正常运行
- 集群中的所有数据都可以正常访问
- 具有高可用性和冗余性

### Yellow（黄色）- 警告

- 所有主分片都正常运行，数据可以正常访问
- 部分副本分片不可用

**常见原因**：
- 单节点集群（无法分配副本）
- 节点故障导致副本丢失
- 副本数设置过多

**影响**：数据可访问，但高可用性降低

### Red（红色）- 异常

- 部分主分片不可用
- 某些数据无法访问

**常见原因**：
- 多节点故障
- 磁盘损坏
- 分片分配失败

**影响**：部分数据丢失，需要紧急处理

---

## 四、常见问题处理

### Q1: 集群状态为 Yellow 如何处理？

**排查步骤**：

```json
# 1. 查看集群健康
GET /_cluster/health

# 2. 查看未分配的分片
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason

# 3. 查看分片分配原因
GET /_cluster/allocation/explain
```

**常见解决方案**：
- 单节点集群：将副本数设为 0
- 节点故障：恢复故障节点
- 磁盘空间不足：清理磁盘空间

### Q2: 集群状态为 Red 如何处理？

**排查步骤**：

```json
# 1. 查看哪些索引为 Red
GET /_cluster/health?level=indices

# 2. 查看具体分片状态
GET /_cat/shards?v&h=index,shard,prirep,state&s=state

# 3. 查看分片分配解释
GET /_cluster/allocation/explain?index=problem_index
```

**解决方案**：
- 修复故障节点
- 恢复磁盘空间
- 必要时重新索引数据

---

## 五、生产环境建议

### 1. 监控集群状态

```json
# 定期检查集群健康
GET /_cluster/health

# 检查节点状态
GET /_cat/nodes?v

# 检查磁盘使用情况
GET /_cat/allocation?v
```

### 2. 设置告警

建议对以下情况设置告警：
- 集群状态变为 Yellow 或 Red
- 未分配分片数 > 0
- 磁盘使用率 > 85%

### 3. 操作后等待恢复

```json
# 关闭/打开索引后等待恢复
POST /my_index/_close
# ... 更新操作 ...
POST /my_index/_open

# 等待集群恢复
GET /_cluster/health?wait_for_status=yellow&timeout=30s
```

---

## 参考资料

- [Elasticsearch集群健康状态(Green/Yellow/Red)的含义及处理方案](https://wenku.csdn.net/answer/85zse5e8e3)
- [ElasticSearch集群状态异常(Red、Yellow)原因分析](https://blog.csdn.net/jsugs/article/details/124297256)
