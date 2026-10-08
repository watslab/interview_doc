# MySQL MVCC 多版本并发控制机制详解

## 概述

MVCC（Multi-Version Concurrency Control，多版本并发控制）是 MySQL InnoDB 存储引擎实现高并发读写的核心机制。它通过"保存数据的多个历史版本"，让读操作不用阻塞写操作、写操作也不用阻塞读操作，从根本上解决了传统锁机制下"读写互斥"的性能瓶颈。

```mermaid
flowchart TB
    subgraph MVCCCore["MVCC 核心思想"]
        direction TB
        C1["保存数据多个历史版本"]
        C2["读操作看旧版本"]
        C3["写操作产生新版本"]
    end
    
    subgraph Benefits["解决的问题"]
        direction LR
        B1["脏读"]
        B2["不可重复读"]
        B3["读写互不阻塞"]
    end
    
    MVCCCore --> Benefits
    
    style C1 fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style C2 fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style C3 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style B1 fill:#ffcdd2,stroke:#c62828,stroke-width:2px
    style B2 fill:#ffcdd2,stroke:#c62828,stroke-width:2px
    style B3 fill:#a5d6a7,stroke:#2e7d32,stroke-width:2px
```

***

## 一、为什么需要 MVCC

### 1.1 传统锁机制的缺陷

| 问题       | 说明                  |
| -------- | ------------------- |
| **读阻塞写** | 读操作加共享锁后，写操作必须等待锁释放 |
| **写阻塞读** | 写操作加锁后，读操作必须等待锁释放   |
| **性能下降** | 锁竞争激烈时，并发性能急剧下降     |
| **死锁风险** | 可能引发死锁问题            |

### 1.2 MVCC 的优势

```mermaid
flowchart LR
    subgraph Traditional["传统锁机制"]
        direction TB
        T1["读操作"] --> T2["等待锁释放"]
        T2 --> T3["获取锁"]
        T3 --> T4["读取数据"]
    end
    
    subgraph MVCC["MVCC 机制"]
        direction TB
        M1["读操作"] --> M2["直接读取历史版本"]
        M2 --> M3["无需等待"]
    end
    
    Traditional -->|对比| MVCC
    
    style T2 fill:#ffcdd2,stroke:#c62828,stroke-width:2px
    style M2 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

| 优势        | 说明           |
| --------- | ------------ |
| **高并发读写** | 无锁快照读，读写互不阻塞 |
| **保证隔离性** | 解决脏读、不可重复读问题 |
| **减少锁竞争** | 避免大量锁等待和死锁   |

***

## 二、MVCC 的物理基础：隐藏列

### 2.1 MVCC 与聚簇索引的关系

MVCC 的隐藏列是存储在 **聚簇索引（Clustered Index）** 的叶子节点中的。在 InnoDB 中，聚簇索引就是表数据的实际存储位置，隐藏列作为行记录的一部分，与用户数据一起存储在聚簇索引的 B+ 树叶子节点中。

```mermaid
flowchart TB
    subgraph ClusteredIndex["聚簇索引 B+ 树结构"]
        direction TB
        
        subgraph Root["根节点"]
            R1["id=50"]
        end
        
        subgraph Branch["分支节点"]
            B1["id=25"]
            B2["id=75"]
        end
        
        subgraph Leaf["叶子节点（数据页）"]
            direction LR
            L1["id=1<br/>name=张三<br/>DB_TRX_ID=10<br/>DB_ROLL_PTR=NULL"]
            L2["id=10<br/>name=李四<br/>DB_TRX_ID=20<br/>DB_ROLL_PTR→undo"]
            L3["id=50<br/>name=王五<br/>DB_TRX_ID=30<br/>DB_ROLL_PTR→undo"]
            L4["id=100<br/>name=赵六<br/>DB_TRX_ID=40<br/>DB_ROLL_PTR→undo"]
        end
        
        R1 --> B1
        R1 --> B2
        B1 --> L1
        B1 --> L2
        B2 --> L3
        B2 --> L4
    end
    
    style L1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style L2 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style L3 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style L4 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

**关键点**：

| 特性 | 说明 |
|------|------|
| **存储位置** | 隐藏列与用户数据一起存储在聚簇索引叶子节点 |
| **索引组织表** | InnoDB 表即聚簇索引，数据按主键顺序存储 |
| **版本链起点** | 聚簇索引中的 DB_ROLL_PTR 指向 undo log 版本链 |
| **二级索引** | 二级索引不包含隐藏列，需回表才能获取 MVCC 信息 |

### 2.2 InnoDB 行记录隐藏列

InnoDB 为每行数据额外添加 3 个隐藏列，这是 MVCC 的物理基础：

| 隐藏列               | 长度   | 作用                  |
| ----------------- | ---- | ------------------- |
| **DB\_TRX\_ID**   | 6 字节 | 记录插入/更新该行的事务 ID     |
| **DB\_ROLL\_PTR** | 7 字节 | 回滚指针，指向该行的上一个历史版本   |
| **DB\_ROW\_ID**   | 6 字节 | 单调递增的行 ID（仅表无主键时使用） |

### 2.3 隐藏列示例

```sql
-- 初始插入一行数据
INSERT INTO user (id, name) VALUES (1, '张三');
-- 事务 ID 为 10

-- 行记录结构：
-- | id | name | DB_TRX_ID | DB_ROLL_PTR |
-- | 1  | 张三 | 10        | NULL        |
```

***

## 三、MVCC 的核心组件：Undo Log

### 3.1 Undo Log 概述

Undo Log（回滚日志）是 InnoDB 为事务回滚和 MVCC 版本管理专门开辟的日志区域。

| 类型                  | 说明                        | 生命周期                  |
| ------------------- | ------------------------- | --------------------- |
| **insert undo log** | 记录 INSERT 操作的 undo        | 事务提交后可立即删除            |
| **update undo log** | 记录 UPDATE/DELETE 操作的 undo | 需等 purge 线程确认无事务引用后清理 |

### 3.2 版本链的形成

```mermaid
flowchart TB
    subgraph VersionChain["版本链示例"]
        direction TB
        V3["主表（最新版）<br/>name=王五<br/>DB_TRX_ID=30"]
        V2["undo log（版本2）<br/>name=李四<br/>DB_TRX_ID=20"]
        V1["undo log（版本1）<br/>name=张三<br/>DB_TRX_ID=10"]
    end
    
    V3 -->|"DB_ROLL_PTR"| V2
    V2 -->|"DB_ROLL_PTR"| V1
    V1 -->|"NULL"| End["链尾"]
    
    style V3 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style V2 fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style V1 fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
```

**版本链形成过程**：

```sql
-- 事务 10：初始插入
INSERT INTO user (id, name) VALUES (1, '张三');
-- 主表：name=张三, DB_TRX_ID=10, DB_ROLL_PTR=NULL

-- 事务 20：更新数据
UPDATE user SET name = '李四' WHERE id = 1;
-- 主表：name=李四, DB_TRX_ID=20, DB_ROLL_PTR→undo log 版本1
-- undo log 版本1：name=张三, DB_TRX_ID=10

-- 事务 30：再次更新
UPDATE user SET name = '王五' WHERE id = 1;
-- 主表：name=王五, DB_TRX_ID=30, DB_ROLL_PTR→undo log 版本2
-- undo log 版本2：name=李四, DB_TRX_ID=20, DB_ROLL_PTR→undo log 版本1
```

***

## 四、MVCC 的核心逻辑：Read View

### 4.1 Read View 概述

Read View（读视图）是事务进行快照读时生成的"数据快照"，本质是一套"可见性判断规则"，用来筛选版本链中对当前事务可见的历史版本。

### 4.2 Read View 的核心字段

| 字段                   | 含义                                      |
| -------------------- | --------------------------------------- |
| **m\_ids**           | 生成 Read View 时，系统中所有活跃的（未提交的）读写事务 ID 列表 |
| **min\_trx\_id**     | m\_ids 中的最小值（当前活跃的最小事务 ID）              |
| **max\_trx\_id**     | 生成 Read View 时，系统下一个要分配的事务 ID           |
| **creator\_trx\_id** | 生成该 Read View 的当前事务 ID                  |

### 4.3 版本可见性判断规则

```mermaid
flowchart TB
    Start(["开始判断版本可见性"]) --> Check1{"DB_TRX_ID ==<br/>creator_trx_id?"}
    
    Check1 -->|是| Visible["✅ 可见<br/>自己修改的数据"]
    
    Check1 -->|否| Check2{"DB_TRX_ID <<br/>min_trx_id?"}
    
    Check2 -->|是| Visible2["✅ 可见<br/>事务已提交"]
    
    Check2 -->|否| Check3{"DB_TRX_ID >=<br/>max_trx_id?"}
    
    Check3 -->|是| Invisible["❌ 不可见<br/>将来事务"]
    
    Check3 -->|否| Check4{"DB_TRX_ID<br/>在 m_ids 中?"}
    
    Check4 -->|是| Invisible2["❌ 不可见<br/>事务未提交"]
    Check4 -->|否| Visible3["✅ 可见<br/>事务已提交"]
    
    style Start fill:#e1f5e1,stroke:#2e7d32,stroke-width:2px
    style Visible fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Visible2 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Visible3 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Invisible fill:#ffcdd2,stroke:#c62828,stroke-width:2px
    style Invisible2 fill:#ffcdd2,stroke:#c62828,stroke-width:2px
```

**判断规则详解**：

| 规则        | 条件                              | 结果            |
| --------- | ------------------------------- | ------------- |
| **规则 1**  | DB\_TRX\_ID == creator\_trx\_id | ✅ 可见（自己修改的数据） |
| **规则 2**  | DB\_TRX\_ID < min\_trx\_id      | ✅ 可见（事务已提交）   |
| **规则 3**  | DB\_TRX\_ID >= max\_trx\_id     | ❌ 不可见（将来事务）   |
| **规则 4a** | DB\_TRX\_ID 在 m\_ids 中          | ❌ 不可见（事务未提交）  |
| **规则 4b** | DB\_TRX\_ID 不在 m\_ids 中         | ✅ 可见（事务已提交）   |

***

## 五、MVCC 与事务隔离级别

### 5.1 事务并发问题

在了解 MVCC 如何支持不同隔离级别之前，需要先理解事务并发可能产生的问题：

#### 5.1.1 脏读（Dirty Read）

**概念**：一个事务读取到了另一个事务未提交的数据。

```mermaid
sequenceDiagram
    participant TA as 事务A
    participant DB as 数据库
    participant TB as 事务B
    
    TA->>DB: BEGIN
    TB->>DB: BEGIN
    
    TB->>DB: UPDATE user SET name='李四'<br/>WHERE id=1
    Note over TB: 未提交
    
    TA->>DB: SELECT name FROM user WHERE id=1
    DB->>TA: 返回 '李四'（脏读）
    
    TB->>DB: ROLLBACK
    Note over TA: TA 读到的数据是无效的
```

**示例**：

```sql
-- 事务 A
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 假设返回 1000

-- 事务 B（并发执行）
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;  -- 未提交

-- 事务 A（脏读）
SELECT balance FROM account WHERE id = 1;  -- 返回 500（读到未提交的数据）

-- 事务 B
ROLLBACK;  -- 回滚，余额实际还是 1000

-- 事务 A 基于错误数据做出了决策
```

#### 5.1.2 不可重复读（Non-Repeatable Read）

**概念**：同一事务内多次读取同一数据，结果不一致（针对**修改**操作）。

```mermaid
sequenceDiagram
    participant TA as 事务A
    participant DB as 数据库
    participant TB as 事务B
    
    TA->>DB: BEGIN
    TB->>DB: BEGIN
    
    TA->>DB: SELECT name FROM user WHERE id=1
    DB->>TA: 返回 '张三'
    
    TB->>DB: UPDATE user SET name='李四' WHERE id=1
    TB->>DB: COMMIT
    
    TA->>DB: SELECT name FROM user WHERE id=1
    DB->>TA: 返回 '李四'（不可重复读）
    
    Note over TA: 同一事务内两次读取结果不一致
```

**示例**：

```sql
-- 事务 A
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 返回 1000

-- 事务 B（并发执行）
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
COMMIT;

-- 事务 A
SELECT balance FROM account WHERE id = 1;  -- 返回 500（不可重复读）
-- 同一事务内两次读取结果不同
COMMIT;
```

#### 5.1.3 幻读（Phantom Read）

**概念**：同一事务内多次执行相同查询，结果集数量不一致（针对**插入/删除**操作）。

```mermaid
sequenceDiagram
    participant TA as 事务A
    participant DB as 数据库
    participant TB as 事务B
    
    TA->>DB: BEGIN
    TB->>DB: BEGIN
    
    TA->>DB: SELECT * FROM user WHERE id > 5
    DB->>TA: 返回 2 条记录
    
    TB->>DB: INSERT INTO user VALUES (6, '新用户')
    TB->>DB: COMMIT
    
    TA->>DB: SELECT * FROM user WHERE id > 5
    DB->>TA: 返回 3 条记录（幻读）
    
    Note over TA: 结果集多了一条"幻影"记录
```

**示例**：

```sql
-- 事务 A
BEGIN;
SELECT * FROM user WHERE age > 20;  -- 返回 5 条记录

-- 事务 B（并发执行）
BEGIN;
INSERT INTO user (id, name, age) VALUES (100, '新用户', 25);
COMMIT;

-- 事务 A
SELECT * FROM user WHERE age > 20;  -- 返回 6 条记录（幻读）
-- 结果集多了一条记录，像出现"幻影"一样
COMMIT;
```

#### 5.1.4 三种问题对比

| 问题 | 操作类型 | 表现 | 影响 |
|------|----------|------|------|
| **脏读** | 读取 | 读到未提交的数据 | 数据可能被回滚，导致决策错误 |
| **不可重复读** | 修改 | 同一数据两次读取结果不同 | 数据一致性被破坏 |
| **幻读** | 插入/删除 | 结果集数量变化 | 范围查询结果不一致 |

### 5.2 Read View 生成时机对比

```mermaid
flowchart TB
    subgraph RR["RR（可重复读）"]
        direction TB
        RR1["事务开始"] --> RR2["第一次 SELECT"]
        RR2 --> RR3["生成 Read View"]
        RR3 --> RR4["后续 SELECT"]
        RR4 --> RR5["复用 Read View"]
    end
    
    subgraph RC["RC（读已提交）"]
        direction TB
        RC1["事务开始"] --> RC2["第一次 SELECT"]
        RC2 --> RC3["生成 Read View"]
        RC3 --> RC4["后续 SELECT"]
        RC4 --> RC5["重新生成 Read View"]
    end
    
    style RR3 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style RR5 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style RC3 fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style RC5 fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
```

### 5.3 隔离级别对比

| 隔离级别                  | Read View 生成时机 | 脏读 | 不可重复读 | 幻读 | 效果                    |
| --------------------- | -------------- | ---- | ------ | ---- | --------------------- |
| **RU（读未提交）**          | 不生成            | ❌ 可能 | ❌ 可能   | ❌ 可能 | 直接读取最新版本              |
| **RC（读已提交）**          | 每次 SELECT 生成   | ✅ 解决 | ❌ 可能   | ❌ 可能 | 能看到已提交的最新数据 |
| **RR（可重复读）**          | 第一次 SELECT 生成  | ✅ 解决 | ✅ 解决   | ⚠️ 部分解决 | 事务内看到的数据一致 |
| **Serializable（串行化）** | 不使用 MVCC       | ✅ 解决 | ✅ 解决   | ✅ 解决 | 通过锁机制实现               |

> **注意**：RR 级别下，InnoDB 通过 MVCC + Next-Key Lock 可以完全解决幻读问题。

### 5.4 Next-Key Lock 与幻读解决

#### 5.4.1 为什么 MVCC 无法完全解决幻读

MVCC 只能解决**快照读**场景下的幻读问题，对于**当前读**场景，MVCC 无法防止幻读：

| 读类型 | MVCC 支持 | 幻读风险 |
|--------|-----------|----------|
| **快照读**（普通 SELECT） | ✅ 可以解决 | 无 |
| **当前读**（SELECT FOR UPDATE） | ❌ 无法解决 | 有 |

**当前读场景的幻读问题**：

```sql
-- 事务 A
BEGIN;
SELECT * FROM user WHERE id > 5 FOR UPDATE;  -- 返回 2 条记录

-- 事务 B（并发执行）
BEGIN;
INSERT INTO user VALUES (6, '新用户');  -- 被阻塞？如果没有锁，可以插入
COMMIT;

-- 事务 A
SELECT * FROM user WHERE id > 5 FOR UPDATE;  -- 返回 3 条记录（幻读）
COMMIT;
```

#### 5.4.2 Next-Key Lock 概念

**Next-Key Lock（临键锁）** 是 InnoDB 在 RR 隔离级别下默认的加锁算法，它是解决当前读幻读问题的关键。

```mermaid
flowchart TB
    subgraph NextKeyLock["Next-Key Lock 组成"]
        direction LR
        RL["行锁<br/>Record Lock"]
        GL["间隙锁<br/>Gap Lock"]
    end
    
    RL --> NKL["Next-Key Lock<br/>= 行锁 + 间隙锁"]
    GL --> NKL
    
    subgraph LockRange["锁定范围"]
        direction LR
        L1["(1, 5]"]
        L2["(5, 10]"]
        L3["(10, 15]"]
    end
    
    NKL --> LockRange
    
    style NKL fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style RL fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style GL fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
```

| 锁类型 | 说明 | 作用 |
|--------|------|------|
| **行锁（Record Lock）** | 锁住某一条具体记录 | 防止其他事务修改/删除该行 |
| **间隙锁（Gap Lock）** | 锁住记录之间的"空隙" | 防止其他事务在该区间插入新数据 |
| **Next-Key Lock** | 行锁 + 间隙锁 | 锁定左开右闭区间 (a, b] |

#### 5.4.3 Next-Key Lock 工作原理

**核心规则**：在 RR 隔离级别下，只要查询走索引扫描，加锁的**基本单位**就是 Next-Key Lock（左开右闭区间 `(a, b]`）；在此之上存在两种退化场景：等值查询命中唯一索引时退化为行锁，等值查询向右遍历至不满足等值条件的记录时退化为间隙锁。

![数据记录（id 唯一索引）与 Next-Key Lock 锁定区间](./images/临键锁区间划分.svg)

**示例**：假设数据表 user 的 id 列为主键（唯一索引），数据为：1、5、8、15

```sql
-- 场景 1：范围查询
SELECT * FROM user WHERE id BETWEEN 5 AND 8 FOR UPDATE;
-- 等价于 id >= 5 AND id <= 8，加锁分三步：
--   步骤1：id >= 5 等值定位，命中已存在的 id = 5 记录
--         Next-Key Lock (1, 5] 退化为行锁，只锁 id = 5 这条记录
--   步骤2：继续扫描到 id = 8，满足条件，加 Next-Key Lock (5, 8]
--   步骤3：继续扫描到 id = 15 以判断扫描结束（不满足 id <= 8）
--         MySQL 8.0.17 及之前：加 Next-Key Lock (8, 15]（见下文说明）
--         MySQL 8.0.18 及之后：不加锁
-- 加锁效果：
--   id = 5、id = 8 记录的修改和删除被阻塞；插入 id = 6、7 被阻塞（5 ~ 8 间隙）
--   插入 id = 2、3、4 等数据不受影响（1 ~ 5 间隙未加锁，且这些值不满足查询条件）
--   插入 id = 9 ~ 14：8.0.17 及之前被阻塞；8.0.18 及之后不受影响

-- 场景 2：等值查询（命中唯一索引）
SELECT * FROM user WHERE id = 5 FOR UPDATE;
-- 行锁：锁定 id = 5 记录
-- 说明：id = 5 记录存在，Next-Key Lock (1, 5] 退化为行锁，只锁 id = 5 这条记录
-- 其他事务可以插入 id = 3、4、6、7 等数据

-- 场景 3：等值查询（未命中记录）
SELECT * FROM user WHERE id = 6 FOR UPDATE;
-- 间隙锁：锁定 (5, 8) 区间（左开右开）
-- 说明：id = 6 记录不存在，扫描向右遍历到 id = 8 时不满足等值条件
--      Next-Key Lock (5, 8] 退化为间隙锁 (5, 8)，不含 id = 8 记录的行锁
-- 其他事务插入 id = 6、7 被阻塞
```

**为什么下界 id = 5 只加行锁？**

`id BETWEEN 5 AND 8` 等价于 `id >= 5 AND id <= 8`，扫描定位第一条记录时使用 `>=` 的等值定位。由于 id = 5 记录存在且 id 是唯一索引，依据"等值查询命中唯一索引时退化为行锁"，Next-Key Lock `(1, 5]` 退化为只锁 id = 5 这条记录。
> id = 5 记录被等值定位直接命中；唯一性约束保证同值记录不可能再插入，间隙内的其他值又不满足查询条件，相应的间隙锁没有防护意义（插入 id = 2、3、4 等数据均不满足 `BETWEEN 5 AND 8`，因此不可能成为幻影行），而对于 id = 5 记录，其在结果集中，必须防止其被其他事务修改或删除，因此需要对其加行锁。

**为什么 8.0.17 及之前会多锁 `(8, 15]`？**

范围扫描无法预知"id = 8 记录是最后一条匹配记录"，判定扫描结束的唯一方法是继续读取下一条记录（id = 15），发现不满足服务层设定的终止边界 `id <= 8` 后才停止（InnoDB 不分析 WHERE 条件，只接收扫描范围）。而加锁发生在"读取时"而非"匹配成功时"，因此 id = 15 记录被读取的那一刻，`(8, 15]` 已被加上。

从防幻读的角度看，这个锁没有防护意义：插入 id = 9~14 等数据不满足查询条件，不可能成为幻影行。所以其属于过度加锁（只损失并发度，不破坏隔离性），因此 MySQL 8.0.18 起将其修复：唯一索引范围查询中，不满足条件的终止记录和相应间隙不再加锁。

| 场景 | 终止记录 | 终止记录处的锁（8.0.18 及之后） | 原因 |
|------|---------|------------------------------|------|
| `id <= 8`（8 存在且命中） | id = 15 | 不加锁 | 插入 9~14 不满足条件，无幻影风险 |
| `id <= 11`（11 不存在） | id = 15 | 间隙锁 `(8, 15)` | 插入 9~11 满足 `id <= 11`，是真实的幻影风险，但是 id = 15 不满足查询条件无需对其加行锁，所以仅需附加间隙锁 |

##### Next-Key Lock 加锁规则总结

**规则 1：范围查询**（查询命中多条记录）

按扫描顺序加锁：

1. 下界等值定位命中且记录存在时，退化为行锁；其余扫描到的匹配记录加 Next-Key Lock `(a, b]`。
2. 终止记录处：8.0.17 及之前加 Next-Key Lock；8.0.18 及之后，上界命中则不附加锁，上界未命中则加间隙锁。

**规则 2：等值查询命中唯一索引**：退化为行锁，只锁该记录。

**规则 3：等值查询未命中**：退化为间隙锁，锁定目标值所在开区间。

#### 5.4.4 RR 级别解决幻读的完整机制

```mermaid
flowchart TB
    subgraph SnapshotRead["快照读"]
        direction TB
        S1["普通 SELECT"] --> S2["MVCC"]
        S2 --> S3["Read View 可见性判断"]
        S3 --> S4["看到事务开始时的数据"]
    end
    
    subgraph CurrentRead["当前读"]
        direction TB
        C1["SELECT FOR UPDATE<br/>UPDATE/DELETE"] --> C2["Next-Key Lock"]
        C2 --> C3["锁定范围区间"]
        C3 --> C4["阻止其他事务插入"]
    end
    
    subgraph Result["RR 级别幻读解决"]
        direction LR
        R1["快照读：MVCC 解决"]
        R2["当前读：Next-Key Lock 解决"]
    end
    
    SnapshotRead --> Result
    CurrentRead --> Result
    
    style S2 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style C2 fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style R1 fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style R2 fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
```

**完整示例**：

```sql
-- 事务 A（RR 级别）
BEGIN;
-- 快照读：MVCC 保证一致性
SELECT * FROM user WHERE id > 5;  -- 返回 2 条记录

-- 当前读：Next-Key Lock 锁定范围
SELECT * FROM user WHERE id > 5 FOR UPDATE;
-- 锁定区间：(5, 8]、(8, 15]、(15, +∞)

-- 事务 B（并发执行）
BEGIN;
INSERT INTO user VALUES (6, '新用户');  -- 被阻塞！无法插入
-- 等待事务 A 释放锁

-- 事务 A
SELECT * FROM user WHERE id > 5 FOR UPDATE;  -- 仍返回 2 条记录
COMMIT;  -- 释放锁

-- 事务 B
-- 此时可以插入成功
INSERT INTO user VALUES (6, '新用户');
COMMIT;
```

#### 5.4.5 Next-Key Lock 注意事项

| 注意点 | 说明 |
|--------|------|
| **生效前提** | InnoDB 引擎、RR 隔离级别、查询走索引扫描 |
| **唯一索引优化** | 唯一索引等值查询自动降级为行锁 |
| **无索引风险** | 无索引范围查询会升级为表锁，导致性能问题 |
| **死锁风险** | Next-Key Lock 可能增加死锁概率 |
| **版本差异** | 范围查询中终止记录处的加锁行为在 8.0.18 起有修复变化（详见 5.4.3） |

### 5.5 RC 与 RR 的核心区别

```sql
-- RC 级别示例
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

BEGIN;
-- 第一次查询：生成 Read View
SELECT name FROM user WHERE id = 1;  -- 假设返回 '张三'

-- 其他事务修改并提交
-- UPDATE user SET name = '李四' WHERE id = 1; COMMIT;

-- 第二次查询：重新生成 Read View
SELECT name FROM user WHERE id = 1;  -- 返回 '李四'（看到已提交的修改）
COMMIT;

-- RR 级别示例
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN;
-- 第一次查询：生成 Read View
SELECT name FROM user WHERE id = 1;  -- 假设返回 '张三'

-- 其他事务修改并提交
-- UPDATE user SET name = '李四' WHERE id = 1; COMMIT;

-- 第二次查询：复用 Read View
SELECT name FROM user WHERE id = 1;  -- 仍返回 '张三'（看不到新提交）
COMMIT;
```

***

## 六、快照读与当前读

### 6.1 两种读操作对比

| 类型      | 定义                       | 示例                                         | 特点        |
| ------- | ------------------------ | ------------------------------------------ | --------- |
| **快照读** | 普通 SELECT，基于 MVCC 读取历史版本 | `SELECT * FROM user WHERE id=1`            | 无锁，读写互不阻塞 |
| **当前读** | 加锁读，读取数据最新版本             | `SELECT * FROM user WHERE id=1 FOR UPDATE` | 加锁，保证数据最新 |

### 6.2 当前读的操作类型

```sql
-- 共享锁当前读
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;
SELECT * FROM user WHERE id = 1 FOR SHARE;  -- MySQL 8.0+

-- 排他锁当前读
SELECT * FROM user WHERE id = 1 FOR UPDATE;

-- 写操作（本质是当前读 + 修改）
INSERT INTO user (id, name) VALUES (1, '张三');
UPDATE user SET name = '李四' WHERE id = 1;
DELETE FROM user WHERE id = 1;
```

#### 为什么写操作本质是"当前读 + 修改"？

```mermaid
flowchart TB
    subgraph Update["UPDATE 执行过程"]
        U1["UPDATE user SET name='李四' WHERE id=1"]
        U2["步骤1：当前读<br/>找到 id=1 的记录<br/>加排他锁，读取最新版本"]
        U3["步骤2：修改<br/>更新数据为 '李四'"]
        U1 --> U2 --> U3
    end
    
    subgraph Delete["DELETE 执行过程"]
        D1["DELETE FROM user WHERE id=1"]
        D2["步骤1：当前读<br/>找到 id=1 的记录<br/>加排他锁，读取最新版本"]
        D3["步骤2：删除<br/>标记记录为已删除"]
        D1 --> D2 --> D3
    end
    
    subgraph Insert["INSERT 执行过程"]
        I1["INSERT INTO user VALUES (1, '张三')"]
        I2["步骤1：当前读<br/>检查唯一索引冲突<br/>（需读取最新数据判断）"]
        I3["步骤2：插入<br/>写入新记录"]
        I1 --> I2 --> I3
    end
    
    style U2 fill:#fff3e0,stroke:#ef6c00
    style D2 fill:#fff3e0,stroke:#ef6c00
    style I2 fill:#fff3e0,stroke:#ef6c00
```

| 操作 | 当前读阶段 | 修改阶段 |
|------|-----------|---------|
| **UPDATE** | 找到目标记录 + 加排他锁 + 读取最新版本 | 更新数据 |
| **DELETE** | 找到目标记录 + 加排他锁 + 读取最新版本 | 标记删除 |
| **INSERT** | 检查唯一索引冲突（需读取最新数据判断） | 写入新记录 |

**关键理解**：

| 问题 | 解释 |
|------|------|
| **为什么必须当前读？** | 写操作必须基于数据的**最新版本**，不能基于历史版本 |
| **为什么必须加锁？** | 防止其他事务同时修改同一条记录，保证数据一致性 |
| **INSERT 也需要当前读吗？** | 是的，INSERT 需要检查唯一索引冲突，这需要读取最新数据 |

### 6.3 快照读与当前读的选择

```mermaid
flowchart TB
    Start(["需要读取数据"]) --> Q1{"需要保证<br/>数据最新?"}
    
    Q1 -->|否| Q2{"读多写少<br/>场景?"}
    Q2 -->|是| Snapshot["快照读<br/>无锁并发"]
    Q2 -->|否| Q3{"是否需要<br/>修改数据?"}
    
    Q1 -->|是| Current["当前读<br/>加锁保证"]
    
    Q3 -->|是| Current
    Q3 -->|否| Snapshot
    
    style Snapshot fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Current fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
```

***

## 七、MVCC 工作流程示例

### 7.1 场景设置

```sql
-- 初始数据
-- id=1, name='张三', DB_TRX_ID=10（已提交）

-- 隔离级别：RR
-- 活跃事务：事务 20（未提交）、事务 30（未提交）
```

### 7.2 执行流程

```mermaid
sequenceDiagram
    participant T30 as 事务30
    participant T20 as 事务20
    participant DB as 数据库
    participant Undo as Undo Log
    
    Note over T30,Undo: 初始状态：name='张三', DB_TRX_ID=10
    
    T30->>DB: 第一次 SELECT
    DB->>T30: 生成 Read View<br/>m_ids=[20,30], min=20, max=31
    DB->>T30: DB_TRX_ID=10 < min=20<br/>✅ 可见，返回 '张三'
    
    T20->>DB: UPDATE SET name='李四'
    DB->>Undo: 写入 undo log<br/>name='张三', DB_TRX_ID=10
    DB->>DB: 更新主表<br/>name='李四', DB_TRX_ID=20
    
    T20->>DB: COMMIT
    
    T30->>DB: 第二次 SELECT
    DB->>T30: 复用 Read View<br/>m_ids=[20,30]
    DB->>T30: DB_TRX_ID=20 在 m_ids 中<br/>❌ 不可见
    DB->>Undo: 回溯版本链
    Undo->>T30: DB_TRX_ID=10 < min=20<br/>✅ 可见，返回 '张三'
```

### 7.3 结果分析

| 操作         | Read View          | 判断过程                                         | 结果   |
| ---------- | ------------------ | -------------------------------------------- | ---- |
| 第一次 SELECT | m\_ids=\[20,30]    | DB\_TRX\_ID=10 < min=20                      | '张三' |
| 第二次 SELECT | 复用 m\_ids=\[20,30] | DB\_TRX\_ID=20 在 m\_ids 中，回溯到 DB\_TRX\_ID=10 | '张三' |

**结论**：RR 级别下，事务 30 两次读结果一致，实现了"可重复读"。

***

## 八、MVCC 的优缺点

### 8.1 优点

| 优点        | 说明           |
| --------- | ------------ |
| **高并发读写** | 无锁快照读，读写互不阻塞 |
| **保证隔离性** | 解决脏读、不可重复读、幻读问题（RR 级别） |
| **减少锁竞争** | 避免大量锁等待和死锁   |

### 8.2 缺点

| 缺点          | 说明                        |
| ----------- | ------------------------- |
| **额外存储开销**  | 需存储 undo log 历史版本         |
| **版本链遍历开销** | 版本链过长时，遍历判断可见性损耗性能        |

***

## 九、MVCC 实现总结

```mermaid
flowchart TB
    subgraph MVCCImplementation["MVCC 实现架构"]
        direction TB
        
        subgraph Physical["物理基础"]
            P1["隐藏列<br/>DB_TRX_ID<br/>DB_ROLL_PTR"]
        end
        
        subgraph Storage["存储组件"]
            S1["Undo Log<br/>版本链"]
        end
        
        subgraph Logic["逻辑组件"]
            L1["Read View<br/>可见性判断"]
        end
        
        subgraph Isolation["隔离级别"]
            I1["RC：每次生成"]
            I2["RR：复用视图"]
        end
    end
    
    Physical --> Storage --> Logic --> Isolation
    
    style P1 fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style S1 fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style L1 fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    style I1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style I2 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

### 9.1 MVCC 三要素

| 要素            | 作用            |
| ------------- | ------------- |
| **隐藏列**       | 记录事务 ID 和回滚指针 |
| **Undo Log**  | 存储历史版本，形成版本链  |
| **Read View** | 判断版本可见性       |

### 9.2 核心流程

1. **写操作**：生成新版本，旧版本写入 undo log
2. **读操作**：根据 Read View 判断版本可见性
3. **版本遍历**：从最新版本开始，沿版本链找到可见版本

***

## 参考资料

- [MySQL 官方文档：InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)
- [MySQL 官方文档：Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)
- [小林 coding：MySQL 是怎么加锁的](https://xiaolincoding.com/mysql/lock/how_to_lock.html)
- [深入理解 MySQL MVCC：多版本并发控制完整版](http://m.toutiao.com/group/7582034789566284339/)
- [MySQL InnoDB 事务隔离与 MVCC、版本链与 ReadView 原理详解](http://m.toutiao.com/group/7578780736728089103/)
- [MySQL 总结--MVCC(read view 和 undo log)](https://blog.csdn.net/huangzhilin2015/article/details/115195777)

