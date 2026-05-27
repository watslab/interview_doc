# 向量化执行技术详解

## 一、概述

### 1.1 什么是向量化执行？

向量化执行（Vectorized Execution）是一种将数据组织为向量（数组）形式，通过单指令多数据（SIMD）指令集实现并行计算的技术。其核心思想在于：将传统逐元素操作（循环迭代）转化为对向量数据的批量处理，从而最大化硬件并行计算能力。

```mermaid
flowchart LR
    subgraph 标量执行["标量执行（逐行处理）"]
        A1["指令1 → 数据A"]
        A2["指令2 → 数据B"]
        A3["指令3 → 数据C"]
        A4["指令4 → 数据D"]
    end

    subgraph 向量化执行["向量化执行（批量处理）"]
        B1["指令1 → 数据A, B, C, D"]
    end

    A1 --> A2 --> A3 --> A4
    B1

    style 标量执行 fill:#ffcdd2,stroke:#c62828
    style 向量化执行 fill:#c8e6c9,stroke:#2e7d32
```

### 1.2 向量化执行 vs 列式存储

向量化执行与列式存储是两个不同层面的概念，常被混淆：

| 概念 | 层面 | 含义 |
|------|------|------|
| **列式存储** | 存储层 | 数据按列而非按行存储在磁盘上 |
| **向量化执行** | 计算层 | 数据以批量（向量）方式在内存中处理 |

列式存储是向量化的数据基石——同列数据类型一致、连续存储，天然适合 SIMD 批量加载和计算。但列式存储 ≠ 必然向量化，向量化也不依赖列式存储（只是行式存储下很难实现）。

### 1.3 历史渊源

向量化思想并非新生事物，其历史可追溯至：

| 时间 | 里程碑 |
|------|--------|
| **1957 年** | APL 语言提出数组编程理念 |
| **1990 年** | J 语言继承 APL 思想 |
| **2000s** | VectorWise 系统将向量化引入关系型数据库 |
| **2010s** | ClickHouse、Doris 等 OLAP 系统深度集成向量化执行 |

---

## 二、SIMD：向量化的硬件基础

### 2.1 SIMD 原理

SIMD（Single Instruction, Multiple Data，单指令多数据）是向量化执行的底层硬件支撑。其核心思想是：**一条 CPU 指令同时操作多个数据元素**。

以 32 位整数加法为例：

```
标量执行（4 次加法，4 条指令）：
  ADD R1, R5    →  a[0] + b[0]
  ADD R2, R6    →  a[1] + b[1]
  ADD R3, R7    →  a[2] + b[2]
  ADD R4, R8    →  a[3] + b[3]

SIMD 执行（1 次加法，1 条指令）：
  VPADDD ZMM0, ZMM1, ZMM2    →  a[0..15] + b[0..15]（AVX-512，16 个 Int32 同时计算）
```

### 2.2 SIMD 指令集演进

| 指令集 | 发布年份 | 寄存器宽度 | 寄存器数量 | 典型吞吐 |
|--------|----------|------------|------------|----------|
| **MMX** | 1996 | 64 位 | 8 | 2 个 Int32 |
| **SSE** | 1999 | 128 位 | 8/16 | 4 个 Int32 / 4 个 Float32 |
| **SSE2** | 2001 | 128 位 | 16 | 4 个 Int32 / 2 个 Float64 |
| **AVX** | 2011 | 256 位 | 16 | 8 个 Int32 / 8 个 Float32 |
| **AVX2** | 2013 | 256 位 | 16 | 8 个 Int32 + 整数向量运算 |
| **AVX-512** | 2013+ | 512 位 | 32 | 16 个 Int32 / 16 个 Float32 |
| **ARM NEON** | — | 128 位 | 32 | 4 个 Int32 |
| **ARM SVE2** | — | 128~2048 位 | 32 | 可变长度 |

### 2.3 AVX-512 关键特性

AVX-512 在前代指令集基础上引入了多项重要能力：

| 特性 | 说明 | 作用 |
|------|------|------|
| **掩码（Masking）** | 几乎所有操作支持谓词变体，仅对掩码指定的通道执行 | 避免无效计算，处理非对齐数据 |
| **置换（Permute）** | 根据索引向量在输入向量中复制元素 | 无需写回内存即可重排数据 |
| **选择性加载（Selective Load）** | 使用掩码从内存加载指定元素 | 减少不必要的内存读取 |
| **选择性存储（Selective Store）** | 使用掩码将指定元素写回内存 | 减少不必要的内存写入 |
| **压缩（Compress）** | 根据掩码将选中元素紧凑排列 | 高效实现过滤操作 |

### 2.4 SIMD 编程方式

| 方式 | 原理 | 优势 | 劣势 |
|------|------|------|------|
| **自动向量化** | 编译器识别可向量化的循环并自动转换 | 零开发成本 | 仅适用于简单循环，编译器保守 |
| **编译器提示** | 通过 `restrict` 关键字、`#pragma omp simd` 等辅助编译器 | 低成本，效果较好 | 需开发者保证正确性 |
| **显式向量化** | 使用 CPU Intrinsics 手动编写 SIMD 代码 | 性能最优，完全控制 | 不可移植，代码复杂 |
| **混合方式** | 编译器提示 + 关键路径手动优化 | 兼顾开发效率与性能 | 需要经验判断优化点 |

---

## 三、数据库执行模型演进

### 3.1 火山模型（Volcano Iterator Model）

传统数据库采用火山模型（又称迭代模型），查询计划由多个算子组成，每个算子实现 `next()` 接口，每次调用返回一行数据。

```mermaid
flowchart TB
    Scan["Scan\nnext() → 1 行"]
    Filter["Filter\nnext() → 1 行"]
    Agg["Aggregate\nnext() → 1 行"]
    Result["Result"]

    Scan --> Filter --> Agg --> Result
```

**执行示例**：`SELECT sum(price) FROM orders WHERE category = 'Electronics'`

```
Scan:     next() → (1, 'Electronics', 999)
Filter:   检查 category，匹配则传递
Aggregate: 取出 price，累加到 sum
... 重复 N 次
```

**火山模型的性能瓶颈**：

| 问题 | 原因 | 影响 |
|------|------|------|
| **虚函数调用开销** | 每次 `next()` 是虚函数派发，需查虚函数表 | CPU 分支预测失败率高 |
| **指令流水线中断** | 每步数据量太小，控制逻辑远多于实际计算 | CPU 流水线无法充分利用 |
| **无法利用 SIMD** | 逐行处理，数据不连续 | 硬件并行能力浪费 |
| **缓存利用率低** | 行式访问不同列数据，随机访问 | Cache Miss 频繁 |

### 3.2 向量化执行模型

向量化执行将 `next()` 接口从返回一行改为返回一个**数据块（Block/Chunk）**，数据以列式组织。

```mermaid
flowchart TB
    Scan["Scan\nnext() → Block（8192 行）"]
    Filter["Filter\n对整列生成位图"]
    Agg["Aggregate\n批量聚合"]
    Result["Result"]

    Scan --> Filter --> Agg --> Result
```

**执行示例**：`SELECT sum(price) FROM orders WHERE category = 'Electronics'`

```
Scan:     next() → Block（含 category 列数组和 price 列数组）
Filter:   对 category 列数组一次性比较，生成位图
Aggregate: 根据位图从 price 列取出对应值，批量求和
```

### 3.3 两种模型对比

| 维度 | 火山模型 | 向量化执行 |
|------|----------|------------|
| **处理单位** | 单行 | 数据块（数千行） |
| **函数调用** | 每行一次虚函数调用 | 每列一次批量调用 |
| **CPU 分支** | 频繁 if 判断 | 紧凑循环，极少分支 |
| **缓存命中** | 低（随机访问） | 高（顺序访问连续内存） |
| **SIMD 利用** | 几乎不可用 | 可大量使用 |
| **中间结果** | 无额外开销 | 需写回临时向量 |

---

## 四、ClickHouse 向量化执行实现

### 4.1 核心抽象

ClickHouse 的向量化执行建立在以下核心抽象之上：

```mermaid
flowchart TB
    subgraph Block["Block（数据块）"]
        C1["IColumn\n列数据"]
        C2["IDataType\n列类型"]
        C3["列名"]
    end

    subgraph IColumn实现["IColumn 实现"]
        ColumnUInt8["ColumnUInt8\n连续字节数组"]
        ColumnUInt32["ColumnUInt32\n连续 Int32 数组"]
        ColumnString["ColumnString\n字符数据 + 偏移量"]
        ColumnConst["ColumnConst\n单值常量列"]
    end

    Block --> IColumn实现
```

#### IColumn

`IColumn` 是内存中列数据的表示接口（严格说是列的分块），提供了关系运算的辅助方法：

| 方法 | 用途 | 对应 SQL |
|------|------|----------|
| `IColumn::filter` | 接受字节掩码过滤列 | `WHERE` / `HAVING` |
| `IColumn::permute` | 按指定排列重排列 | `ORDER BY` |
| `IColumn::cut` | 截取列的前 N 个元素 | `LIMIT` |

**内存布局**：

| 列类型 | 内存布局 | 特点 |
|--------|----------|------|
| **数值列**（ColumnUInt8/32/64 等） | 连续数组，类似 `std::vector` | 内存连续，SIMD 友好 |
| **字符串列**（ColumnString） | 字符数据向量 + 偏移量向量 | 两个连续数组 |
| **数组列**（ColumnArray） | 元素向量 + 偏移量向量 | 嵌套数据连续存储 |
| **常量列**（ColumnConst） | 仅存储一个值 | 对外表现为完整列 |

#### Field

`Field` 用于表示单个值，是 `UInt64`、`Int64`、`Float64`、`String`、`Array` 的可判别联合类型。通过 `IColumn::operator[]` 获取第 n 个值，但效率不高——向量化执行应尽量避免使用 `Field`，转而使用 `insertFrom`、`insertRangeFrom` 等批量方法。

#### Block

`Block` 是查询执行流水线中流动的数据单元，由多个 `IColumn` 及其对应的 `IDataType` 和列名组成。ClickHouse 的最小执行单元不是行，而是 Block，默认包含 `max_block_size`（通常 8192）行数据。

```
Block 示例：
  user_id  → [u1, u2, u3, ..., u8192]     (ColumnUInt64)
  price    → [10, 20, 30, ..., 5000]       (ColumnFloat64)
  category → ['A', 'B', 'A', ..., 'C']     (ColumnString)
```

### 4.2 抽象泄漏（Leaky Abstractions）

ClickHouse 的一个关键设计哲学是**抽象泄漏**：虽然 `IColumn` 提供了通用接口，但性能关键路径鼓励绕过通用接口，直接将 `IColumn` 强制转换为具体实现（如 `ColumnUInt64`），通过 `getData()` 方法获取内部数组的引用，直接操作连续内存。

```cpp
// 通用方式（低效）
Field val = column[i];  // 每次构造临时 Field 对象

// 抽象泄漏方式（高效）
auto & data = static_cast<const ColumnUInt64 &>(column).getData();
// 直接操作连续数组，编译器可自动向量化
for (size_t i = 0; i < data.size(); ++i) {
    result[i] = data[i] * factor;
}
```

这种设计牺牲了封装性，换取了极致性能——使得专门化例程可以充分利用内存布局和 SIMD 指令。

### 4.3 向量化函数执行流程

以 `SELECT a, abs(b) FROM test` 为例，ClickHouse 的执行流程如下：

```mermaid
sequenceDiagram
    participant Storage as TinyLogBlockInputStream
    participant Expr as ExpressionBlockInputStream
    participant Func as IFunction (abs)

    Storage->>Expr: readImpl() → Block
    Expr->>Expr: 从 Block 中筛选参数列
    Expr->>Expr: 在 Block 中添加空结果列
    Expr->>Func: execute(block, arguments, result_col)
    Func->>Func: 对整列数据执行 abs
    Note over Func: 编译器自动向量化<br/>或手动 Intrinsics
    Func-->>Expr: 结果写入结果列
    Expr-->>Expr: 返回包含结果的 Block
```

**关键步骤**：

1. 从底层存储流读取一个 Block
2. 筛选出函数参数对应的列
3. 在 Block 中添加一个空的结果列
4. 调用函数对整列数据执行计算
5. 结果写入结果列，Block 向上传递

### 4.4 向量化算子实现

ClickHouse 几乎在所有核心算子中应用了向量化执行：

| 算子 | 向量化实现方式 | 说明 |
|------|----------------|------|
| **WHERE 过滤** | 对整列生成字节掩码（Byte Mask） | `IColumn::filter(mask)` |
| **表达式计算** | 对列数组做逐元素运算 | 编译器自动向量化 + 手动 Intrinsics |
| **GROUP BY** | 向量化哈希聚合 | 批量插入哈希表 |
| **聚合函数** | 批量累加 | `vec_sum`、`vec_avg` 等 |
| **ORDER BY** | 部分阶段向量化 | `IColumn::permute` |
| **JOIN** | Build/Probe 阶段向量化 | 批量构建/探测哈希表 |

### 4.5 向量化执行示例

以 `SELECT sum(price * quantity) FROM orders WHERE category = 'Electronics'` 为例：

```
Step 1: Scan → 读取 Block
  category → ['Electronics', 'Books', 'Electronics', ...]
  price    → [999,  49, 1299, ...]
  quantity → [2,    5,  1,    ...]

Step 2: Filter → 对 category 列批量比较
  mask = [1, 0, 1, ...]  // category == 'Electronics' 的位图

Step 3: Expression → price * quantity 批量乘法
  result = [999×2, 49×5, 1299×1, ...] = [1998, 245, 1299, ...]

Step 4: Aggregate → 根据 mask 批量求和
  sum = 1998 + 1299 + ...  // 仅累加 mask=1 的位置
```

---

## 五、向量化执行 vs 运行时代码生成

### 5.1 两种加速方式

数据库查询加速有两种核心方法：

| 方式 | 原理 | 优势 | 劣势 |
|------|------|------|------|
| **向量化执行** | 以 Block 为单位批量处理，利用 SIMD | 易利用 SIMD，实现简单 | 产生临时向量，需写回缓存再读取 |
| **运行时代码生成（JIT）** | 运行时将多个算子融合为一段机器码 | 消除虚函数调用，融合操作减少中间结果 | 难以利用 SIMD，编译开销 |

### 5.2 详细对比

| 维度 | 向量化执行 | 运行时代码生成 |
|------|------------|----------------|
| **中间结果** | 需要写回临时向量到缓存 | 算子融合，中间结果留在寄存器 |
| **SIMD 利用** | 天然友好 | 较难利用 |
| **缓存压力** | 临时数据超出 L2 缓存时性能下降 | 无中间数据，缓存友好 |
| **实现复杂度** | 相对简单 | 需要嵌入编译器（如 LLVM） |
| **编译开销** | 无 | 首次查询有 JIT 编译延迟 |
| **适用场景** | 通用 OLAP 查询 | 复杂表达式、长算子链 |

### 5.3 ClickHouse 的选择

ClickHouse **以向量化执行为主**，对运行时代码生成提供有限支持。原因如下：

1. 向量化执行更容易利用 SIMD，而 SIMD 带来的加速是确定性的
2. 实现复杂度更低，维护成本更小
3. 研究表明，两种方式结合效果最佳，但向量化是基础

---

## 六、向量化执行为何高效

### 6.1 CPU Cache 友好

列式存储中同一列数据连续排列，向量化执行顺序访问整列数组，具有极高的空间局部性：

```
行式访问（Cache 不友好）：
  访问行1: [id, name, age, price]  → 跨多个缓存行
  访问行2: [id, name, age, price]  → 再次跨多个缓存行

列式访问（Cache 友好）：
  访问 age 列: [25, 30, 28, 35, ...]  → 连续内存，一次加载填满缓存行
```

### 6.2 SIMD 并行加速

以 `price > 100` 过滤为例：

```
标量执行：逐个比较，N 次指令
  for (int i = 0; i < N; i++)
      if (price[i] > 100) mask[i] = 1;

AVX-512 向量化：一次比较 16 个 Int32，N/16 次指令
  for (int i = 0; i < N; i += 16)
      zmm = _mm512_loadu_ps(&price[i]);
      mask = _mm512_cmpgt_ps(zmm, threshold);
      _mm512_storeu_ps(&result[i], zmm);
```

| 指令集 | 单指令处理 Int32 数量 | 理论加速比 |
|--------|----------------------|------------|
| SSE | 4 | 4x |
| AVX2 | 8 | 8x |
| AVX-512 | 16 | 16x |

### 6.3 减少函数调用与分支

| 维度 | 行式执行 | 向量化执行 |
|------|----------|------------|
| **虚函数调用** | 每行一次 | 每列一次 |
| **分支预测** | 每行 if 判断，预测失败率高 | 紧凑循环，预测成功率高 |
| **流水线效率** | 频繁中断 | 持续充满 |

### 6.4 循环展开与指令调度

向量化执行的紧凑循环结构使得编译器可以：

- **自动循环展开**：减少循环控制开销
- **指令重排**：隐藏内存访问延迟
- **软件预取**：提前加载数据到缓存

---

## 七、向量化执行的局限性

### 7.1 性能退化场景

| 场景 | 原因 | 影响 |
|------|------|------|
| **复杂 UDF / Lambda** | 逐行逻辑无法批量处理 | 退化为标量执行 |
| **复杂条件分支** | 每行逻辑不同，掩码开销大 | SIMD 优势减弱 |
| **极小数据量** | 批量处理的优势无法体现 | 向量化开销 > 收益 |
| **大量 LIKE / 正则** | 字符串操作难以向量化 | CPU 成本上升 |
| **临时向量超出 L2 缓存** | 中间结果需写回内存再读取 | 缓存成为瓶颈 |

### 7.2 向量化失效的 SQL 模式

```sql
-- ❌ 复杂 UDF：无法向量化
SELECT custom_complex_udf(col1) FROM t;

-- ❌ 逐行条件分支：掩码开销大
SELECT CASE
    WHEN a > 100 AND b < 50 AND c LIKE '%test%' THEN ...
    WHEN a > 200 AND d IN (1,2,3) THEN ...
    ELSE ...
END FROM t;

-- ❌ 极小数据量：向量化开销大于收益
SELECT * FROM t LIMIT 10;
```

### 7.3 向量化友好的 SQL 模式

```sql
-- ✅ 等值 / IN / 范围过滤
SELECT count(*) FROM orders
WHERE category = 'Electronics'
  AND price BETWEEN 100 AND 500;

-- ✅ 聚合计算
SELECT category, sum(price), avg(quantity)
FROM orders
GROUP BY category;

-- ✅ 简单表达式
SELECT price * quantity AS total FROM orders;
```

---

## 八、ClickHouse 向量化最佳实践

### 8.1 充分利用向量化的查询原则

| 原则 | 说明 |
|------|------|
| **优先全列扫描** | ClickHouse 的性能不是"索引查得快"，而是"即使扫，也扫得极快" |
| **减少查询列数** | 只 SELECT 需要的列，减少 I/O 和计算量 |
| **使用简单过滤条件** | 等值、IN、范围过滤最利于向量化 |
| **避免复杂 UDF** | 用内置函数替代自定义逻辑 |
| **合理设置 Block 大小** | `max_block_size` 默认 8192，大数据量场景可适当增大 |

### 8.2 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_block_size` | 8192 | 单个 Block 的最大行数，影响向量化批处理粒度 |
| `min_insert_block_size_rows` | 1048576 | 插入时最小 Block 行数 |
| `use_compact_format_in_distributed_parts_names` | 1 | 分布式表紧凑格式 |

### 8.3 性能验证方法

```sql
-- 查看查询执行计划
EXPLAIN PIPELINE SELECT sum(price) FROM orders WHERE category = 'Electronics';

-- 查看查询性能指标
SELECT
    query,
    read_rows,
    result_rows,
    memory_usage,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY event_date DESC
LIMIT 10;
```

---

## 九、总结

### 9.1 核心要点

1. **向量化执行的本质**：从"一次一行"到"一次一批"，将列数据作为数组批量处理
2. **硬件基础**：SIMD 指令集（SSE/AVX/AVX-512）允许单条指令同时处理多个数据
3. **数据基础**：列式存储提供连续内存布局，天然适配 SIMD 加载
4. **ClickHouse 实现**：基于 IColumn/Block 抽象，通过抽象泄漏直接操作连续数组
5. **性能来源**：减少虚函数调用、提高缓存命中率、利用 SIMD 并行、优化分支预测

### 9.2 一句话总结

> ClickHouse 的向量化执行：以 Block 为单位，把列当数组，用 SIMD 和顺序内存访问对整批数据做计算，从而极大降低 CPU 指令数和 Cache Miss。

---

## 参考

- [ClickHouse Architecture Overview](https://clickhouse.com/docs/en/development/architecture)
- [CMU 15-721 Lecture #06: Vectorized Query Execution](https://15721.courses.cs.cmu.edu/spring2024/notes/06-vectorization.pdf)
- [ClickHouse 源码笔记：函数调用的向量化实现](https://cloud.tencent.com/developer/article/1790111)
- [OceanBase：什么是向量化](https://open.oceanbase.com/blog/22701646097)
