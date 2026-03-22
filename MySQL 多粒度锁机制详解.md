# MySQL 多粒度锁机制详解

## 概述

InnoDB 存储引擎支持**多粒度锁（Multiple Granularity Locking）**，允许行级锁与表级锁共存。这种机制使得数据库能够在不同粒度级别上进行并发控制，既保证了数据的一致性，又最大化了并发性能。

```mermaid
flowchart TB
    subgraph LockHierarchy["锁粒度层次"]
        direction TB
        
        subgraph TableLocks["表级锁"]
            direction LR
            S["共享锁（S）"]
            X["排他锁（X）"]
            IS["意向共享锁（IS）"]
            IX["意向排他锁（IX）"]
        end
        
        TableLocks --> RowLocks
        
        subgraph RowLocks["行级锁"]
            direction LR
            RS["行级共享锁"]
            RX["行级排他锁"]
        end
    end
    
    style S fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style X fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style IS fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style IX fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style RS fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style RX fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

---

## 一、表锁与行锁

### 1.1 表锁

MySQL 自身提供的表级锁能力：

| 锁类型 | SQL 语法 | 作用 |
|--------|----------|------|
| **表级共享锁（S锁）** | `LOCK TABLE table_name READ` | 阻塞其他事务的写操作 |
| **表级排他锁（X锁）** | `LOCK TABLE table_name WRITE` | 阻塞其他事务的读和写操作 |

### 1.2 行锁

InnoDB 存储引擎提供的行级锁能力（MySQL 本身不提供行级锁）：

| 锁类型 | SQL 语法 | 作用 |
|--------|----------|------|
| **行级共享锁** | `SELECT ... LOCK IN SHARE MODE` | 阻塞其他事务对该行的写操作 |
| **行级排他锁** | `SELECT ... FOR UPDATE` | 阻塞其他事务对该行的读和写操作 |

### 1.3 表锁与行锁共存的问题

```mermaid
sequenceDiagram
    participant TA as 事务A
    participant DB as 数据库
    participant TB as 事务B
    
    Note over TA,DB: 问题场景
    
    TA->>DB: SELECT * FROM users WHERE id=6 FOR UPDATE
    Note over TA: 获取 id=6 的行级排他锁
    
    TB->>DB: LOCK TABLE users WRITE
    Note over TB: 申请整表的排他锁
    
    Note over DB: 问题：如何判断事务B能否获取表锁？
    Note over DB: 方案1：遍历表中每一行检查是否有锁
    Note over DB: 方案2：通过意向锁快速判断ㅤㅤㅤㅤ
```

**问题分析**：

假设事务 A 获取了某一行的排他锁，事务 B 想要获取整张表的排他锁。数据库需要判断：

1. 当前没有其他事务持有该表的排他锁
2. 当前没有其他事务持有该表中**任意一行**的排他锁

**传统方案的缺陷**：为了检测条件 2，需要遍历表中的每一行，效率极其低下。

---

## 二、意向锁（Intention Lock）

### 2.1 意向锁的定义

> **意向锁是一个表级锁，其作用就是指明接下来的事务将会用到哪种锁。**

意向锁是 InnoDB 自动维护的，用户无法手动操作。在为数据行加共享/排他锁之前，InnoDB 会先获取该数据行所在表的对应意向锁。

### 2.2 意向锁的类型

| 意向锁类型 | 说明 | 触发场景 |
|------------|------|----------|
| **意向共享锁（IS）** | 事务有意向对表中的某些行加共享锁 | `SELECT ... LOCK IN SHARE MODE` |
| **意向排他锁（IX）** | 事务有意向对表中的某些行加排他锁 | `SELECT ... FOR UPDATE` |

### 2.3 意向锁的工作流程

```mermaid
flowchart TB
    Start(["事务申请行锁"]) --> CheckISORIX{"申请行级<br/>共享锁/排他锁?"}
    
    CheckISORIX -->|行级共享锁| GetIS["自动获取表的 IS 锁"]
    CheckISORIX -->|行级排他锁| GetIX["自动获取表的 IX 锁"]
    
    GetIS --> GetRowS["获取行级共享锁"]
    GetIX --> GetRowX["获取行级排他锁"]
    
    style GetIS fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style GetIX fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style GetRowS fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style GetRowX fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

### 2.4 意向锁的兼容性矩阵

#### 2.4.1 意向锁之间的兼容性

|  | 意向共享锁（IS） | 意向排他锁（IX） |
|--|----------------|----------------|
| **意向共享锁（IS）** | ✅ 兼容 | ✅ 兼容 |
| **意向排他锁（IX）** | ✅ 兼容 | ✅ 兼容 |

**关键点**：意向锁之间是**互相兼容**的。

#### 2.4.2 意向锁与表级锁的兼容性

|  | 意向共享锁（IS） | 意向排他锁（IX） |
|--|----------------|----------------|
| **表级共享锁（S）** | ✅ 兼容 | ❌ 互斥 |
| **表级排他锁（X）** | ❌ 互斥 | ❌ 互斥 |

> **重要**：意向锁不会与行级的共享/排他锁互斥！

---

## 三、意向锁如何解决问题

### 3.1 问题回顾

事务 A 获取了某一行的排他锁，事务 B 想要获取整张表的排他锁。

### 3.2 使用意向锁后的解决方案

```mermaid
sequenceDiagram
    participant TA as 事务A
    participant DB as 数据库
    participant TB as 事务B
    participant TC as 事务C
    
    Note over TA,DB: 事务A获取行锁
    
    TA->>DB: SELECT * FROM users WHERE id=6 FOR UPDATE
    Note over DB: 1. 自动获取 users 表的意向排他锁（IX）
    Note over DB: 2. 获取 id = 6 的行级排他锁ㅤㅤㅤㅤㅤㅤ
    
    Note over TB,DB: 事务B申请表锁
    
    TB->>DB: LOCK TABLE users WRITE
    Note over DB: 检测到意向排他锁（IX）存在ㅤㅤㅤ
    Note over DB: 意向排他锁（IX）与表级排他锁互斥
    Note over TB: 加锁请求被阻塞
    
    Note over TC,DB: 事务C申请其他行的锁
    
    TC->>DB: SELECT * FROM users WHERE id=5 FOR UPDATE
    Note over DB: 1. 申请 users 表的意向排他锁（IX）
    Note over DB: 2. 意向排他锁之间兼容，获取成功ㅤ
    Note over DB: 3. 获取 id = 5 的行级排他锁ㅤㅤㅤㅤ
    Note over TC: 成功获取锁
```

### 3.3 完整示例

假设 users 表数据如下：

| id | name |
|----|------|
| 1 | ROADHOG |
| 2 | Reinhardt |
| 3 | Tracer |
| 4 | Genji |
| 5 | Hanzo |
| 6 | Mccree |

**执行流程**：

```sql
-- 事务 A：获取某一行的排他锁
SELECT * FROM users WHERE id = 6 FOR UPDATE;
-- 1. 自动获取 users 表的意向排他锁（IX）
-- 2. 获取 id=6 的行级排他锁（X）

-- 事务 B：尝试获取表锁
LOCK TABLE users READ;
-- 检测到 IX 锁存在，IX 与表级 S 锁互斥
-- 加锁请求被阻塞

-- 事务 C：获取另一行的排他锁
SELECT * FROM users WHERE id = 5 FOR UPDATE;
-- 1. 申请 users 表的 IX 锁
-- 2. IX 与 IX 兼容，获取成功
-- 3. 获取 id=5 的行级排他锁
-- 成功！
```

### 3.4 效率对比

| 方案 | 判断方式 | 效率 |
|------|----------|------|
| **无意向锁** | 遍历表中每一行检查是否有锁 | O(n)，效率极低 |
| **有意向锁** | 检查表上的意向锁 | O(1)，效率极高 |

---

## 四、多粒度锁与 MVCC 的关系

### 4.1 MVCC 概述

MVCC（Multi-Version Concurrency Control，多版本并发控制）是 InnoDB 实现高并发读写的核心机制。它通过保存数据的多个历史版本，让读操作不用阻塞写操作。

### 4.2 读操作的分类

| 读类型 | 说明 | 实现机制 | 是否涉及锁 |
|--------|------|----------|------------|
| **快照读** | 普通 SELECT | MVCC | 不加锁 |
| **当前读** | SELECT FOR UPDATE | 锁机制 | 加锁 |

### 4.3 MVCC 与锁机制的协作

```mermaid
flowchart TB
    subgraph ReadTypes["读操作分类"]
        direction TB
        
        subgraph SnapshotRead["快照读"]
            S1["普通 SELECT"] --> S2["MVCC"]
            S2 --> S3["读取历史版本"]
            S3 --> S4["无需加锁"]
        end
        
        subgraph CurrentRead["当前读"]
            C1["SELECT FOR UPDATE<br/>SELECT LOCK IN SHARE MODE"] --> C2["锁机制"]
            C2 --> C3["读取最新版本"]
            C3 --> C4["需要加锁"]
        end
    end
    
    subgraph LockMechanism["锁机制"]
        direction TB
        L1["意向锁（表级）"] --> L2["行锁（行级）"]
    end
    
    CurrentRead --> LockMechanism
    
    style S2 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style C2 fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style L1 fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style L2 fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
```

### 4.4 MVCC 与意向锁的关系

| 维度 | MVCC | 意向锁 |
|------|------|--------|
| **目的** | 解决读写并发问题 | 解决表锁与行锁的协调问题 |
| **作用范围** | 行级数据版本 | 表级锁协调 |
| **触发场景** | 快照读（普通 SELECT） | 当前读（SELECT FOR UPDATE 等） |
| **是否互斥** | 快照读不与任何锁互斥 | 意向锁之间兼容，与表级锁有互斥关系 |

### 4.5 完整的并发控制流程

```mermaid
flowchart TB
    Start(["事务开始"]) --> ReadType{"读操作类型?"}
    
    ReadType -->|"快照读<br/>普通 SELECT"| MVCC["MVCC 机制"]
    MVCC --> ReadHistory["读取历史版本"]
    ReadHistory --> NoLock["无需加锁"]
    NoLock --> Done1(["完成"])
    
    ReadType -->|"当前读<br/>FOR UPDATE / LOCK IN SHARE MODE"| IntentType{"意向锁类型?"}
    
    IntentType -->|"行级共享锁<br/>LOCK IN SHARE MODE"| RequestIS["申请意向共享锁（IS）"]
    IntentType -->|"行级排他锁<br/>FOR UPDATE"| RequestIX["申请意向排他锁（IX）"]
    
    RequestIS --> CheckIS{"IS 与表级锁<br/>冲突?"}
    RequestIX --> CheckIX{"IX 与表级锁<br/>冲突?"}
    
    CheckIS -->|是| WaitIS["等待"]
    CheckIS -->|否| GetIS["获取意向共享锁（IS）"]
    
    CheckIX -->|是| WaitIX["等待"]
    CheckIX -->|否| GetIX["获取意向排他锁（IX）"]
    
    WaitIS --> CheckIS
    WaitIX --> CheckIX
    
    GetIS --> GetRowS["获取行级共享锁"]
    GetIX --> GetRowX["获取行级排他锁"]
    
    GetRowS --> ReadLatestS["读取最新版本"]
    GetRowX --> ReadLatestX["读取最新版本"]
    
    ReadLatestS --> Done2(["完成"])
    ReadLatestX --> Done3(["完成"])
    
    style MVCC fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style GetIS fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style GetIX fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style GetRowS fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style GetRowX fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
```

### 4.6 为什么需要两套机制？

| 场景 | MVCC | 锁机制（含意向锁） |
|------|------|-------------------|
| **普通查询** | ✅ 适用，无锁并发 | ❌ 不适用，会降低并发 |
| **精确更新** | ❌ 不适用，需要最新数据 | ✅ 适用，保证数据一致性 |
| **范围查询加锁** | ❌ 不适用 | ✅ 适用，配合 Next-Key Lock |
| **DDL 操作** | ❌ 不适用 | ✅ 适用，表级锁保护结构 |

**总结**：

- **MVCC**：优化读多写少场景，快照读无需加锁
- **意向锁**：优化表锁与行锁的协调，避免全表扫描
- **两者协作**：共同实现高并发、高一致性的数据库操作

---

## 五、总结

### 5.1 意向锁的核心价值

1. **效率提升**：将 O(n) 的行遍历检查优化为 O(1) 的表意向锁检查
2. **并发保证**：意向锁之间兼容，不影响不同行的并发操作
3. **协调机制**：实现表锁与行锁的和谐共存

### 5.2 关键要点

| 要点 | 说明 |
|------|------|
| **意向锁是表级锁** | 不会与行级锁冲突 |
| **意向锁之间兼容** | IS 与 IX 可以共存 |
| **意向锁与表级锁有互斥关系** | IX 与 S 互斥，IX 与 X 互斥 |
| **自动获取** | 无需手动操作，InnoDB 自动维护 |
| **与 MVCC 互补** | 快照读用 MVCC，当前读用锁机制 |

### 5.3 锁机制全景图

```mermaid
flowchart TB
    subgraph MySQLLock["MySQL InnoDB 锁机制"]
        direction TB
        
        subgraph TableLevel["表级锁"]
            direction LR
            TS["共享锁（S）"]
            TX["排他锁（X）"]
            TIS["意向共享锁（IS）"]
            TIX["意向排他锁（IX）"]
        end
        
        subgraph RowLevel["行级锁"]
            direction LR
            RS["行级共享锁"]
            RX["行级排他锁"]
            Gap["间隙锁"]
            NextKey["临键锁"]
        end
        
        subgraph MVCCMechanism["MVCC 机制"]
            direction LR
            RV["Read View"]
            UL["Undo Log"]
            VC["版本链"]
        end
    end
    
    style TableLevel fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style RowLevel fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style MVCCMechanism fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
```

---

## 参考资料

- [详解 MySql InnoDB 中意向锁的作用 - 掘金](https://juejin.cn/post/6844903666332368909)
- [MySQL InnoDB 意向锁详解 - 51CTO](https://www.51cto.com/article/743293.html)
- [MySQL 官方文档：InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
