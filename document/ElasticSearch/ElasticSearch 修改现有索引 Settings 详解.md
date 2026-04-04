# ElasticSearch 修改现有索引 Settings 详解

## 一、核心步骤

**必须先关闭索引，修改后再打开**，因为动态更新 settings 不支持添加新的 analyzer。

```
┌─────────────────────────────────────────────────────────────┐
│          修改现有索引 Settings 的步骤                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   1. POST /my_index/_close     ← 关闭索引                    │
│   2. PUT /my_index/_settings   ← 更新settings（添加分词器）  │
│   3. POST /my_index/_open      ← 打开索引                    │
│   4. PUT /my_index/_mapping    ← 更新mapping（使用新分词器） │
│                                                              │
│   【注意】关闭索引期间，索引不可读写                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、完整操作流程

### 步骤一：关闭索引

```json
POST /my_index/_close
```

### 步骤二：更新 settings（添加自定义分词器）

```json
PUT /my_index/_settings
{
  "settings": {
    "analysis": {
      "filter": {
        "my_stopwords": {
          "type": "stop",
          "stopwords": ["的", "了", "是", "在"]
        },
        "pinyin_filter": {
          "type": "pinyin",
          "keep_full_pinyin": true,
          "keep_first_letter": true
        }
      },
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "my_stopwords"]
        },
        "pinyin_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["pinyin_filter"]
        }
      }
    }
  }
}
```

### 步骤三：打开索引

```json
POST /my_index/_open
```

### 步骤四：更新 mapping（使用新分词器）

```json
PUT /my_index/_mapping
{
  "properties": {
    "title": {
      "type": "text",
      "analyzer": "my_analyzer",
      "search_analyzer": "ik_smart"
    }
  }
}
```

---

## 三、完整示例

```json
# 1. 关闭索引
POST /my_index/_close

# 2. 添加自定义分词器
PUT /my_index/_settings
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonym": {
          "type": "synonym",
          "synonyms": [
            "手机,移动电话,智能机",
            "电脑,计算机,PC"
          ]
        }
      },
      "analyzer": {
        "synonym_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "my_synonym"]
        }
      }
    }
  }
}

# 3. 打开索引
POST /my_index/_open

# 4. 等待索引恢复（可选）
GET /_cluster/health?wait_for_status=yellow&timeout=30s

# 5. 测试新分词器
POST /my_index/_analyze
{
  "analyzer": "synonym_analyzer",
  "text": "我买了一部手机"
}
```

---

## 四、注意事项

### 1. 关闭索引的影响

| 影响 | 说明 |
|------|------|
| 索引不可搜索 | 关闭期间无法查询该索引 |
| 索引不可写入 | 关闭期间无法写入数据 |
| 数据不会丢失 | 数据仍然保存在磁盘上 |

### 2. 哪些 settings 可以动态修改

| 设置类型 | 是否支持动态修改 | 说明 |
|----------|------------------|------|
| `number_of_replicas` | ✅ 支持 | 副本数可动态调整 |
| `refresh_interval` | ✅ 支持 | 刷新间隔可动态调整 |
| `analysis`（分词器） | ❌ 需要关闭索引 | 需要关闭索引后修改 |
| `number_of_shards` | ❌ 不支持 | 主分片数创建后不可修改 |

### 3. 生产环境建议

```json
# 在低峰期操作
# 1. 先检查集群状态
GET /_cluster/health

# 2. 关闭索引
POST /my_index/_close

# 3. 更新settings
PUT /my_index/_settings
{ ... }

# 4. 打开索引
POST /my_index/_open

# 5. 等待恢复
GET /_cluster/health?wait_for_status=yellow&timeout=30s

# 6. 验证分词器
POST /my_index/_analyze
{ ... }
```

---

## 五、替代方案：重建索引

如果数据量不大，可以考虑重建索引：

```json
# 1. 创建新索引（带新分词器）
PUT /my_index_new
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "my_analyzer"
      }
    }
  }
}

# 2. 迁移数据
POST /_reindex
{
  "source": { "index": "my_index" },
  "dest": { "index": "my_index_new" }
}

# 3. 删除旧索引
DELETE /my_index

# 4. 创建别名（可选）
POST /_aliases
{
  "actions": [
    { "add": { "index": "my_index_new", "alias": "my_index" } }
  ]
}
```

---

## 参考资料

- [Elasticsearch 向已存在的索引中加入自定义filter/analyzer](https://blog.csdn.net/weixin_34062329/article/details/85925886)
- [elasticsearch 添加或修改分词器](https://blog.csdn.net/nddjava/article/details/116201681)
