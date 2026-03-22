# ElasticSearch 常用分词器详解

## 概述

分词器（Analyzer）是ElasticSearch中用于文本分析的核心组件，它将文本分解为一个个独立的词项（Term），以便于建立倒排索引和搜索。

---

## 分词器分类

```
┌─────────────────────────────────────────────────────────────┐
│                    ElasticSearch 分词器                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  内置分词器  │  │  官方插件   │  │ 第三方分词器 │          │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤          │
│  │• Standard   │  │• SmartCN   │  │• IK (中文)  │          │
│  │• Simple     │  │  (中文)    │  │• jieba      │          │
│  │• Whitespace │  └─────────────┘  │• Pinyin     │          │
│  │• Keyword    │                   │• THULAC     │          │
│  │• Pattern    │                   │• HanLP      │          │
│  │• Stop       │                   └─────────────┘          │
│  │• Language   │                                             │
│  └─────────────┘                                             │
│                                                              │
│  说明：内置分词器开箱即用，官方插件需安装，第三方需自行集成    │
└─────────────────────────────────────────────────────────────┘
```

---

## 一、内置分词器

### 1. Standard Analyzer（标准分词器）

**默认分词器**，适用于大多数欧洲语言。

#### 特点

- 按词切分，去除标点符号
- 支持多语言
- 对中文按单字切分（不推荐用于中文）

#### 使用示例

```json
POST _analyze
{
  "analyzer": "standard",
  "text": "The Quick Brown Fox"
}
```

**输出结果**：`[the, quick, brown, fox]`

```json
POST _analyze
{
  "analyzer": "standard",
  "text": "中华人民共和国"
}
```

**输出结果**：`[中, 华, 人, 民, 共, 和, 国]`（中文按单字切分）

#### 索引映射配置

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "standard"
      }
    }
  }
}
```

---

### 2. Simple Analyzer（简单分词器）

#### 特点

- 按非字母字符切分
- 所有字符转为小写
- 去除数字和标点

#### 使用示例

```json
POST _analyze
{
  "analyzer": "simple",
  "text": "The2Quick Brown-Fox"
}
```

**输出结果**：`[the, quick, brown, fox]`

---

### 3. Whitespace Analyzer（空格分词器）

#### 特点

- 按空格切分
- 不转小写
- 不去除标点

#### 使用示例

```json
POST _analyze
{
  "analyzer": "whitespace",
  "text": "The Quick Brown Fox"
}
```

**输出结果**：`[The, Quick, Brown, Fox]`

---

### 4. Keyword Analyzer（关键字分词器）

#### 特点

- **不分词**，将整个文本作为一个词项
- 适用于精确匹配场景

#### 使用示例

```json
POST _analyze
{
  "analyzer": "keyword",
  "text": "The Quick Brown Fox"
}
```

**输出结果**：`[The Quick Brown Fox]`

#### 适用场景

- 精确匹配搜索
- ID、邮箱、手机号等字段

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "email": {
        "type": "keyword"
      }
    }
  }
}
```

---

### 5. Pattern Analyzer（模式分词器）

#### 特点

- 使用正则表达式切分
- 默认按非字母字符切分
- 支持自定义正则表达式

#### 使用示例

```json
POST _analyze
{
  "analyzer": "pattern",
  "text": "The2Quick-Brown_Fox"
}
```

**输出结果**：`[the, quick, brown, fox]`

#### 自定义正则表达式

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_pattern_analyzer": {
          "type": "pattern",
          "pattern": "[\\W_]+",
          "lowercase": true
        }
      }
    }
  }
}
```

---

### 6. Stop Analyzer（停用词分词器）

#### 特点

- 类似Simple分词器
- 额外去除停用词（the, a, an, is等）

#### 使用示例

```json
POST _analyze
{
  "analyzer": "stop",
  "text": "The Quick Brown Fox is running"
}
```

**输出结果**：`[quick, brown, fox, running]`

---

### 7. Language Analyzer（语言分词器）

#### 特点

- 针对特定语言优化
- 支持多种语言：english, french, german, dutch等
- **注意**：内置的Language Analyzer不直接支持中文

#### English分词器示例

```json
POST _analyze
{
  "analyzer": "english",
  "text": "The Quick Brown Foxes are running"
}
```

**输出结果**：`[quick, brown, fox, ar, run]`（进行词干提取）

#### 中文分词说明

**内置Language Analyzer不支持中文**，如尝试使用：

```json
POST _analyze
{
  "analyzer": "chinese",
  "text": "中华人民共和国"
}
```

会返回错误或按单字切分，效果与Standard分词器相同。

**解决方案**：使用官方提供的 **Smart Chinese Analysis 插件**（见下文"官方插件"部分）。

---

## 二、官方插件分词器

官方插件需要通过 `elasticsearch-plugin` 命令安装，由ElasticSearch官方维护。

---

### 1. Smart Chinese Analyzer（官方中文分词器）

ElasticSearch官方提供的中文分词插件，基于Lucene的SmartChineseAnalyzer实现。

#### 特点

- 官方维护，稳定可靠
- 基于隐马尔可夫模型（HMM）
- 零配置即可使用
- 功能与扩展性弱于IK分词器

#### 安装方式

```bash
# 进入ES安装目录
cd /usr/share/elasticsearch

# 安装smartcn插件
./bin/elasticsearch-plugin install analysis-smartcn

# 重启ElasticSearch
systemctl restart elasticsearch
```

#### 使用示例

```json
POST _analyze
{
  "analyzer": "smartcn",
  "text": "中华人民共和国国歌"
}
```

**输出结果**：`[中华人民共和国, 国歌]`

```json
POST _analyze
{
  "analyzer": "smartcn",
  "text": "我们都是中国人"
}
```

**输出结果**：`[我们, 都, 是, 中国, 人]`

#### 索引映射配置

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "smartcn"
      }
    }
  }
}
```

#### SmartCN vs IK 对比

| 特性 | SmartCN | IK分词器 |
|------|---------|----------|
| 维护方 | 官方 | 社区 |
| 分词模式 | 单一模式 | ik_smart / ik_max_word |
| 自定义词典 | ❌ 不支持 | ✅ 支持 |
| 扩展性 | 弱 | 强 |
| 配置复杂度 | 零配置 | 需配置词典 |
| 中文分词效果 | 一般 | 较好 |

**推荐**：如果需要更好的中文分词效果和自定义词典功能，建议使用IK分词器。

---

## 三、第三方中文分词器

第三方分词器由社区或个人开发维护，功能通常更丰富。

---

### 1. IK分词器（推荐）

**最流行的中文分词器**，支持智能分词和自定义词典。

#### 两种分词模式

| 模式 | 说明 | 示例输入 | 分词结果 |
|------|------|----------|----------|
| `ik_smart` | 粗粒度分词 | 华为手机 | [华为, 手机] |
| `ik_max_word` | 细粒度分词 | 华为手机 | [华为, 手, 手机] |

#### 安装方式

**方式一：在线安装**

```bash
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.0/elasticsearch-analysis-ik-7.17.0.zip
```

**方式二：离线安装**

1. 下载对应版本的IK分词器：https://github.com/medcl/elasticsearch-analysis-ik/releases
2. 解压到 `elasticsearch/plugins/ik` 目录
3. 重启ElasticSearch

```bash
# 解压到plugins目录
unzip elasticsearch-analysis-ik-7.17.0.zip -d elasticsearch/plugins/ik/

# 重启ES
./bin/elasticsearch
```

#### 使用示例

```json
POST _analyze
{
  "analyzer": "ik_smart",
  "text": "中华人民共和国国歌"
}
```

**输出结果**：`[中华人民共和国, 国歌]`

```json
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "中华人民共和国国歌"
}
```

**输出结果**：`[中华人民共和国, 中华人民, 中华, 华人, 人民共和国, 人民, 共和国, 共和, 国, 国歌]`

#### 索引映射配置

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      }
    }
  }
}
```

#### 自定义词典

**步骤1：创建词典文件**

```bash
cd elasticsearch/plugins/ik/config
vim my_dict.dic
```

添加自定义词汇：
```
华为手机
王者荣耀
绝地求生
```

**步骤2：配置词典**

编辑 `IKAnalyzer.cfg.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<properties>
    <comment>IK Analyzer 扩展配置</comment>
    <entry key="ext_dict">my_dict.dic</entry>
    <entry key="ext_stopwords">stopword.dic</entry>
</properties>
```

**步骤3：重启ElasticSearch**

---

### 2. Pinyin分词器

**拼音分词器**，支持中文转拼音搜索。

#### 安装方式

```bash
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v7.17.0/elasticsearch-analysis-pinyin-7.17.0.zip
```

#### 使用示例

```json
POST _analyze
{
  "analyzer": "pinyin",
  "text": "刘德华"
}
```

**输出结果**：`[liu, de, hua, ldh, 刘德华]`

#### 索引映射配置

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "ik_pinyin_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["pinyin_filter"]
        }
      },
      "filter": {
        "pinyin_filter": {
          "type": "pinyin",
          "keep_full_pinyin": true,
          "keep_first_letter": true,
          "keep_original": true
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "ik_pinyin_analyzer"
      }
    }
  }
}
```

#### 搜索示例

```json
GET /my_index/_search
{
  "query": {
    "match": {
      "name": "ldh"
    }
  }
}
```

---

### 3. jieba分词器

**结巴分词**，Python生态最流行的中文分词工具，也有ES插件版本。

#### 特点

- 支持两种分词模式（ES插件）
- 支持繁体分词
- 支持自定义词典
- 社区活跃，持续更新

#### 安装方式

**方式一：源码编译安装（推荐）**

```bash
# 1. 克隆项目
git clone https://github.com/sing1ee/elasticsearch-jieba-plugin.git

# 2. 进入项目目录
cd elasticsearch-jieba-plugin

# 3. 切换到对应ES版本的分支（如ES 7.10）
git checkout 7.10

# 4. 使用Gradle构建
./gradlew buildPluginZip

# 5. 安装插件到ES
cp build/distributions/elasticsearch-jieba-plugin-*.zip /path/to/elasticsearch/plugins/
cd /path/to/elasticsearch/plugins/
unzip elasticsearch-jieba-plugin-*.zip

# 6. 重启ElasticSearch
```

**方式二：下载预编译版本**

从GitHub Releases下载对应版本的插件包，解压到 `elasticsearch/plugins/` 目录。

#### 分词模式

ES插件提供两种分词器，**可直接使用，无需自定义配置**：

| 分词器 | 说明 | 适用场景 |
|--------|------|----------|
| `jieba_index` | 索引分词，分词粒度较细 | 索引时使用 |
| `jieba_search` | 搜索分词，分词粒度较粗 | 搜索时使用 |

#### 使用示例

```json
POST _analyze
{
  "analyzer": "jieba_index",
  "text": "中华人民共和国国歌"
}
```

**输出结果**：`[中华人民共和国, 中华人民, 中华, 华人, 人民共和国, 人民, 共和国, 共和, 国, 国歌]`

```json
POST _analyze
{
  "analyzer": "jieba_search",
  "text": "中华人民共和国国歌"
}
```

**输出结果**：`[中华人民共和国, 国歌]`

#### 索引映射配置

jieba_index和jieba_search是插件内置的分析器，**可以直接在字段中指定**：

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "jieba_index",
        "search_analyzer": "jieba_search"
      }
    }
  }
}
```

#### 自定义词典

jieba插件支持自定义词典，将词典文件放入指定目录即可自动加载。

**词典文件格式**

词典文件必须使用 `.dict` 后缀，格式为：`词语 词频`（词频可选）

```
小清新 3
百搭 3
显瘦 3
华为手机 100
王者荣耀 50
```

**配置步骤**

```bash
# 1. 进入插件的dic目录
cd ${ES_HOME}/plugins/jieba/dic

# 2. 创建自定义词典文件
cat > user.dict << 'EOF'
华为手机 100
王者荣耀 50
绝地求生 30
EOF

# 3. 重启ElasticSearch使词典生效
```

> **提示**：jieba插件支持动态添加字典，ES不需要重启即可加载新词典（将.dict文件放入dic目录后自动加载）。

**使用停用词**

```bash
# 1. 创建停用词目录
mkdir -p ${ES_HOME}/config/stopwords

# 2. 复制停用词文件
cp ${ES_HOME}/plugins/jieba/dic/stopwords.txt ${ES_HOME}/config/stopwords/
```

```json
# 3. 创建索引时配置分析器
PUT /jieba_index
{
  "settings": {
    "analysis": {
      "filter": {
        "jieba_stop": {
          "type": "stop",
          "stopwords_path": "stopwords/stopwords.txt"
        }
      },
      "analyzer": {
        "jieba_with_stop": {
          "tokenizer": "jieba_index",
          "filter": ["lowercase", "jieba_stop"]
        }
      }
    }
  }
}
```

#### Python独立使用

如果不需要ES集成，可以单独在Python中使用jieba。Python版jieba支持三种分词模式：

```python
import jieba

# 精确模式
seg_list = jieba.cut("我爱自然语言处理", cut_all=False)
print("精确模式: " + "/".join(seg_list))

# 全模式
seg_list = jieba.cut("我爱自然语言处理", cut_all=True)
print("全模式: " + "/".join(seg_list))

# 搜索引擎模式
seg_list = jieba.cut_for_search("我爱自然语言处理")
print("搜索引擎模式: " + "/".join(seg_list))
```

> **注意**：Python版jieba的三种模式（精确模式、全模式、搜索引擎模式）与ES插件的两种模式（jieba_index、jieba_search）不同，ES插件是专门为搜索场景优化的实现。

#### jieba vs IK 对比

| 特性 | jieba | IK分词器 |
|------|-------|----------|
| 维护方 | 社区 | 社区 |
| 安装方式 | 需编译 | 直接下载 |
| 分词模式 | jieba_index / jieba_search | ik_smart / ik_max_word |
| 自定义词典 | ✅ 支持 | ✅ 支持 |
| Python生态 | ✅ 原生支持 | ❌ |
| 中文分词效果 | 优秀 | 优秀 |

**推荐**：如果项目主要使用Python技术栈，jieba是更好的选择；如果需要开箱即用的ES插件，IK更方便。

---

### 4. THULAC分词器

**清华大学自然语言处理与社会人文计算实验室**开发的中文分词工具。

#### 特点

- 准确率高
- 支持词性标注
- 支持命名实体识别

---

### 5. HanLP分词器

**Han Language Processing**，功能全面的NLP工具包。

#### 特点

- 支持分词、词性标注、命名实体识别
- 支持依存句法分析
- 支持语义角色标注

---

## 四、分词器对比

| 分词器 | 类型 | 中文支持 | 性能 | 自定义词典 | 适用场景 |
|--------|------|----------|------|------------|----------|
| Standard | 内置 | ❌ 单字切分 | ⭐⭐⭐⭐⭐ | ❌ | 英文文本 |
| SmartCN | 官方插件 | ✅ 一般 | ⭐⭐⭐⭐ | ❌ | 简单中文场景 |
| IK | 第三方 | ✅ 优秀 | ⭐⭐⭐⭐ | ✅ | **中文搜索（推荐）** |
| Pinyin | 第三方 | ✅ 拼音 | ⭐⭐⭐⭐ | ✅ | 拼音搜索 |
| jieba | 第三方 | ✅ 优秀 | ⭐⭐⭐⭐ | ✅ | Python生态 |
| THULAC | 第三方 | ✅ 优秀 | ⭐⭐⭐ | ✅ | 学术研究 |
| HanLP | 第三方 | ✅ 优秀 | ⭐⭐⭐ | ✅ | 复杂NLP任务 |

---

## 五、最佳实践

### 1. 索引时与搜索时使用不同分词器

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      }
    }
  }
}
```

**说明**：
- `analyzer`：索引时使用细粒度分词，提高召回率
- `search_analyzer`：搜索时使用粗粒度分词，提高精确度

### 2. 组合使用多个分词器

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "ik_pinyin_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["pinyin_filter"]
        }
      },
      "filter": {
        "pinyin_filter": {
          "type": "pinyin",
          "keep_full_pinyin": true,
          "keep_first_letter": true
        }
      }
    }
  }
}
```

### 3. 使用停用词过滤

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "ik_max_word",
          "stopwords": ["的", "了", "是", "在"]
        }
      }
    }
  }
}
```

### 4. 测试分词效果

```json
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "测试分词效果"
}
```

---

## 六、常见问题

### Q1: 为什么中文搜索搜不到结果？

**答**：ElasticSearch默认的Standard分词器对中文按单字切分，建议使用IK分词器。

### Q2: 如何添加自定义词汇？

**答**：在IK分词器的config目录下创建自定义词典文件，并在`IKAnalyzer.cfg.xml`中配置。

### Q3: 索引时和搜索时应该用什么分词器？

**答**：建议索引时使用`ik_max_word`（细粒度），搜索时使用`ik_smart`（粗粒度）。

### Q4: 如何实现拼音搜索？

**答**：安装Pinyin分词器，结合IK分词器使用。

---

## 参考资料

- [Elasticsearch中文分词:内置与IK分词器的使用](https://blog.csdn.net/dl674756321/article/details/119979708)
- [Elasticsearch安装与使用IK中文分词器](https://blog.csdn.net/pan_junbiao/article/details/115249823)
- [Elasticsearch分词器详解](https://blog.51cto.com/u_16099320/14199760)
- [IK分词器GitHub](https://github.com/medcl/elasticsearch-analysis-ik)
- [Pinyin分词器GitHub](https://github.com/medcl/elasticsearch-analysis-pinyin)
