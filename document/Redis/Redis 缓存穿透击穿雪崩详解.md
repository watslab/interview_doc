# Redis 缓存穿透、击穿、雪崩详解

## 一、问题概述

在使用 Redis 缓存的高并发系统中，缓存穿透、击穿、雪崩是三大经典问题，它们都会导致大量请求直接打到数据库，可能造成数据库崩溃、系统瘫痪。

```mermaid
flowchart TB
    subgraph Problems["缓存三大问题"]
        P1["缓存穿透<br/>Cache Penetration"]
        P2["缓存击穿<br/>Cache Breakdown"]
        P3["缓存雪崩<br/>Cache Avalanche"]
    end
    
    subgraph P1Desc["穿透描述"]
        P1D1["查询不存在的数据"]
        P1D2["绕过缓存直接打库"]
        P1D3["影响范围：中"]
    end
    
    subgraph P2Desc["击穿描述"]
        P2D1["热点 Key 过期"]
        P2D2["大量并发请求打库"]
        P2D3["影响范围：中高"]
    end
    
    subgraph P3Desc["雪崩描述"]
        P3D1["大量 Key 同时过期<br/>或 Redis 宕机"]
        P3D2["所有请求穿透到数据库"]
        P3D3["影响范围：最高"]
    end
    
    P1 --> P1Desc
    P2 --> P2Desc
    P3 --> P3Desc
    
    style Problems fill:#e3f2fd,stroke:#1565c0
    style P1Desc fill:#ffcdd2,stroke:#c62828
    style P2Desc fill:#fff3e0,stroke:#ef6c00
    style P3Desc fill:#f3e5f5,stroke:#7b1fa2
```

### 1.1 三大问题对比

| 问题       | 触发场景                  | 影响范围 | 核心特征      |
| -------- | --------------------- | ---- | --------- |
| **缓存穿透** | 查询根本不存在的数据            | 中    | 缓存和数据库都没有 |
| **缓存击穿** | 热点 Key 过期瞬间           | 中高   | 单点热点，高并发  |
| **缓存雪崩** | 大量 Key 同时过期或 Redis 宕机 | 最高   | 大面积失效     |

***

## 二、缓存穿透

### 2.1 问题场景

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Redis as Redis 缓存
    participant DB as 数据库
    
    Note over Client,DB: 恶意请求查询不存在的数据
    
    Client->>Redis: 查询 user:id=999999
    Redis-->>Client: 缓存未命中
    
    Client->>DB: 查询数据库
    DB-->>Client: 数据不存在
    
    Note over Client,DB: 由于数据不存在，无法缓存<br/>下次请求继续穿透
    
    Client->>Redis: 再次查询 user:id=999999
    Redis-->>Client: 缓存未命中
    
    Client->>DB: 再次查询数据库
    DB-->>Client: 数据不存在
```

**缓存穿透的原因**：

| 原因 | 说明 |
| ---- | ------------------ |
| **恶意攻击** | 攻击者故意请求不存在的数据，绕过缓存 |
| **业务缺陷** | 业务逻辑错误，产生大量无效查询 |
| **数据删除** | 数据被删除后，请求仍访问已删除的数据（如已下架商品、已注销用户） |

### 2.2 解决方案

#### 方案一：缓存空值

```mermaid
flowchart TB
    subgraph Process["查询流程"]
        A["请求查询数据"] --> B{"缓存是否存在?"}
        B -->|"是"| C["返回数据"]
        B -->|"否"| D["查询数据库"]
        D --> E{"数据是否存在?"}
        E -->|"是"| F["写入缓存<br/>返回数据"]
        E -->|"否"| G["缓存空值<br/>设置短过期时间"]
        G --> H["返回空"]
    end
    
    style Process fill:#e3f2fd,stroke:#1565c0
```

**代码示例**：

```java
public User getUserById(Long id) {
    String key = "user:" + id;
    String value = redis.get(key);
    
    if (value != null) {
        if ("NULL".equals(value)) {
            return null;
        }
        return JSON.parseObject(value, User.class);
    }
    
    User user = userMapper.selectById(id);
    
    if (user != null) {
        redis.set(key, JSON.toJSONString(user), 3600);
    } else {
        redis.set(key, "NULL", 300);
    }
    
    return user;
}
```

**优缺点**：

| 优点        | 缺点          |
| --------- | ----------- |
| 实现简单      | 占用额外内存      |
| 有效减少数据库压力 | 需要设置合理的过期时间 |
| <br />    | 可能存在数据不一致   |

#### 方案二：布隆过滤器

##### 什么是布隆过滤器

布隆过滤器（Bloom Filter）是一种空间效率很高的概率型数据结构，用于判断一个元素是否在一个集合中。它的特点是：

- **判断不存在**：100% 准确，元素一定不在集合中
- **判断存在**：可能误判，元素可能在集合中（假阳性）

##### 工作原理图解

```mermaid
flowchart TB
    subgraph BloomFilter["布隆过滤器架构"]
        subgraph Init["初始化阶段"]
            Load["加载数据库所有有效 ID"]
            BF["写入布隆过滤器"]
        end
        
        subgraph Query["查询阶段"]
            Request["请求到达"]
            Check{"布隆过滤器<br/>判断 ID 是否可能存在?"}
            Reject["直接拒绝<br/>返回空"]
            Cache["查询缓存/数据库"]
        end
    end
    
    Load --> BF
    Request --> Check
    Check -->|"不存在"| Reject
    Check -->|"可能存在"| Cache
    
    style BloomFilter fill:#e3f2fd,stroke:#1565c0
    style Init fill:#c8e6c9,stroke:#2e7d32
    style Query fill:#fff3e0,stroke:#ef6c00
```

##### 存储过程详解

```mermaid
flowchart TB
    subgraph Store["元素存储过程"]
        Input["输入元素: user:123"]
        HashFunc["使用 k 个哈希函数<br/>计算 k 个位置"]
        Pos1["位置 1: 5"]
        Pos2["位置 2: 17"]
        Pos3["位置 3: 29"]
        SetBits["将对应位置设为 1"]
    end
    
    subgraph BitArray["位数组（初始全 0）"]
        B0["0"]
        B1["0"]
        B2["0"]
        B3["0"]
        B4["0"]
        B5["1"]
        B6["0"]
        B7["0"]
        B8["..."]
        B9["0"]
        B10["1"]
        B11["0"]
        B12["..."]
        B13["0"]
        B14["1"]
        B15["0"]
    end
    
    Input --> HashFunc
    HashFunc --> Pos1
    HashFunc --> Pos2
    HashFunc --> Pos3
    Pos1 --> SetBits
    Pos2 --> SetBits
    Pos3 --> SetBits
    SetBits --> BitArray
    
    style Store fill:#e3f2fd,stroke:#1565c0
    style BitArray fill:#c8e6c9,stroke:#2e7d32
```

##### 查询过程详解

```mermaid
flowchart TB
    subgraph QueryExist["查询存在的元素"]
        Q1["查询: user:123"]
        QHash1["计算哈希位置: 5, 17, 29"]
        QCheck1["检查这些位置"]
        QResult1["全部为 1 → 可能存在"]
    end
    
    subgraph QueryNotExist["查询不存在的元素"]
        Q2["查询: user:999"]
        QHash2["计算哈希位置: 3, 8, 42"]
        QCheck2["检查这些位置"]
        QResult2["存在 0 → 一定不存在"]
    end
    
    subgraph QueryFalse["误判场景"]
        Q3["查询: user:888"]
        QHash3["计算哈希位置: 5, 17, 29"]
        QCheck3["检查这些位置"]
        QResult3["全部为 1 → 误判为存在<br/>实际不存在"]
    end
    
    style QueryExist fill:#c8e6c9,stroke:#2e7d32
    style QueryNotExist fill:#ffcdd2,stroke:#c62828
    style QueryFalse fill:#fff3e0,stroke:#ef6c00
```

##### 为什么会有误判

```mermaid
flowchart LR
    subgraph Scenario["误判原因"]
        A["元素 A 设置位置: 5, 17, 29"]
        B["元素 B 设置位置: 3, 8, 42"]
        C["元素 C 设置位置: 5, 8, 35"]
    end
    
    subgraph BitArray["位数组状态"]
        Bits["位置 5: 1 (A, C)<br/>位置 17: 1 (A)<br/>位置 29: 1 (A)<br/>位置 3: 1 (B)<br/>位置 8: 1 (B, C)<br/>位置 42: 1 (B)<br/>位置 35: 1 (C)"]
    end
    
    subgraph FalsePositive["误判示例"]
        D["查询元素 D<br/>哈希位置: 5, 17, 29"]
        Result["位置 5, 17, 29 都为 1<br/>判断: 可能存在<br/>实际: 不存在（被 A 的位覆盖）"]
    end
    
    Scenario --> BitArray
    BitArray --> FalsePositive
    
    style Scenario fill:#e3f2fd,stroke:#1565c0
    style BitArray fill:#c8e6c9,stroke:#2e7d32
    style FalsePositive fill:#ffcdd2,stroke:#c62828
```

**误判原因总结**：
- 不同元素经过哈希函数计算后，可能得到相同的位置
- 当查询一个不存在的元素时，其所有哈希位置可能恰好都被其他元素设置为 1
- 这种"巧合"导致误判（假阳性）

##### 关键参数选择

| 参数 | 说明 | 影响 |
|------|------|------|
| **n** | 预期元素数量 | 过小会导致误判率升高 |
| **m** | 位数组大小（bit） | 越大误判率越低，但占用内存越多 |
| **k** | 哈希函数数量 | 过多影响性能，过少误判率高 |
| **p** | 期望误判率 | 越低需要越大的 m 和越多的 k |

**参数计算公式**：

```
最优哈希函数数量: k = (m/n) × ln(2)
最小位数组大小: m = -n × ln(p) / (ln(2))²
```

**实际选择建议**：

| 预期元素数量 | 期望误判率 | 建议位数组大小 | 建议哈希函数数 |
|--------------|------------|----------------|----------------|
| 100 万 | 1% | 9.6 Mbit (1.2 MB) | 7 |
| 100 万 | 0.1% | 14.4 Mbit (1.8 MB) | 10 |
| 1000 万 | 1% | 96 Mbit (12 MB) | 7 |
| 1000 万 | 0.1% | 144 Mbit (18 MB) | 10 |

##### 布隆过滤器特点总结

| 特性 | 说明 |
|------|------|
| **空间效率高** | 使用位数组，每个元素只需几个 bit |
| **查询速度快** | O(k) 时间复杂度，k 为哈希函数数量 |
| **插入速度快** | O(k) 时间复杂度 |
| **存在误判** | 可能判断存在但实际不存在（假阳性） |
| **不存在误判** | 判断不存在则一定不存在 |
| **不能删除** | 不支持删除元素（会影响其他元素） |

##### Redisson 实现示例：

```java
@Configuration
public class BloomFilterConfig {
    
    @Bean
    public RBloomFilter<Long> userBloomFilter(RedissonClient redisson) {
        RBloomFilter<Long> bloomFilter = redisson.getBloomFilter("user:bloom");
        bloomFilter.tryInit(1000000L, 0.01);
        return bloomFilter;
    }
}

@Service
public class UserService {
    
    @Autowired
    private RBloomFilter<Long> userBloomFilter;
    
    public User getUserById(Long id) {
        if (!userBloomFilter.contains(id)) {
            return null;
        }
        
        String key = "user:" + id;
        String value = redis.get(key);
        
        if (value != null) {
            return JSON.parseObject(value, User.class);
        }
        
        User user = userMapper.selectById(id);
        if (user != null) {
            redis.set(key, JSON.toJSONString(user), 3600);
        }
        
        return user;
    }
}
```

#### 方案三：接口层校验

```java
@GetMapping("/user/{id}")
public Result getUser(@PathVariable Long id) {
    if (id == null || id <= 0) {
        return Result.error("参数错误");
    }
    
    if (!isValidId(id)) {
        return Result.error("无效ID");
    }
    
    return Result.success(userService.getUserById(id));
}
```

### 2.3 方案对比

| 方案        | 优点        | 缺点         | 适用场景       |
| --------- | --------- | ---------- | ---------- |
| **缓存空值**  | 实现简单      | 占用内存、数据不一致 | 数据量小、穿透频率低 |
| **布隆过滤器** | 内存占用小、效率高 | 存在误判、需要预热  | 数据量大、穿透频率高 |
| **接口校验**  | 提前拦截      | 无法防止所有情况   | 参数校验、基础防护  |

***

## 三、缓存击穿

### 3.1 问题场景

```mermaid
sequenceDiagram
    participant C1 as 客户端1
    participant C2 as 客户端2
    participant C3 as 客户端N
    participant Redis as Redis 缓存
    participant DB as 数据库
    
    Note over Redis: 热点 Key 过期瞬间
    
    par 并发请求
        C1->>Redis: 查询热点 Key
        C2->>Redis: 查询热点 Key
        C3->>Redis: 查询热点 Key
    end
    
    Redis-->>C1: 缓存过期
    Redis-->>C2: 缓存过期
    Redis-->>C3: 缓存过期
    
    par 大量请求打向数据库
        C1->>DB: 查询数据库
        C2->>DB: 查询数据库
        C3->>DB: 查询数据库
    end
    
    Note over DB: 数据库瞬间压力激增
```

**缓存击穿的原因**：

| 原因         | 说明             |
| ---------- | -------------- |
| **热点数据过期** | 高频访问的 Key 恰好过期 |
| **缓存未命中**  | 大量并发请求同时发现缓存失效 |
| **无保护机制**  | 没有限制并发查询数据库    |

### 3.2 解决方案

#### 方案一：互斥锁（分布式锁）

```mermaid
flowchart TB
    subgraph Process["互斥锁方案流程"]
        A["请求查询缓存"] --> B{"缓存是否存在?"}
        B -->|"是"| C["返回数据"]
        B -->|"否"| D["尝试获取分布式锁"]
        D --> E{"获取锁成功?"}
        E -->|"是"| F["查询数据库"]
        F --> G["写入缓存"]
        G --> H["释放锁"]
        H --> C
        E -->|"否"| I["等待并重试"]
        I --> A
    end
    
    style Process fill:#e3f2fd,stroke:#1565c0
```

**代码示例**：

```java
public User getUserWithLock(Long id) {
    String key = "user:" + id;
    String lockKey = "lock:user:" + id;
    
    String value = redis.get(key);
    if (value != null) {
        return JSON.parseObject(value, User.class);
    }
    
    boolean locked = false;
    try {
        locked = redis.setnx(lockKey, "1", 10);
        if (locked) {
            User user = userMapper.selectById(id);
            if (user != null) {
                redis.set(key, JSON.toJSONString(user), 3600);
            }
            return user;
        } else {
            Thread.sleep(50);
            return getUserWithLock(id);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return null;
    } finally {
        if (locked) {
            redis.del(lockKey);
        }
    }
}
```

**关键说明**：

| 问题 | 说明 |
|------|------|
| **为什么用 finally？** | 确保锁最终被释放，避免死锁 |
| **为什么判断 locked？** | 只有获取锁成功的线程才能释放锁 |
| **锁过期时间 10 秒** | 防止业务异常导致锁永不释放 |

**更安全的锁实现（Lua 脚本保证原子性）**：

```java
public User getUserWithSafeLock(Long id) {
    String key = "user:" + id;
    String lockKey = "lock:user:" + id;
    String lockValue = UUID.randomUUID().toString();
    
    String value = redis.get(key);
    if (value != null) {
        return JSON.parseObject(value, User.class);
    }
    
    boolean locked = false;
    try {
        locked = redis.setnx(lockKey, lockValue, 10);
        if (locked) {
            User user = userMapper.selectById(id);
            if (user != null) {
                redis.set(key, JSON.toJSONString(user), 3600);
            }
            return user;
        } else {
            Thread.sleep(50);
            return getUserWithSafeLock(id);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return null;
    } finally {
        if (locked) {
            String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                           "return redis.call('del', KEYS[1]) " +
                           "else return 0 end";
            redis.eval(script, Arrays.asList(lockKey), Arrays.asList(lockValue));
        }
    }
}
```

**安全锁的关键点**：

| 要点 | 说明 |
|------|------|
| **锁值唯一性** | 使用 UUID 作为锁值，标识锁的持有者 |
| **Lua 脚本释放** | 保证"判断锁归属"和"删除锁"的原子性 |
| **防止误删** | 只删除自己持有的锁，避免删除其他线程的锁 |

**为什么需要 UUID 标识锁持有者？**

当线程执行时间超过锁过期时间时，锁会自动释放，其他线程可能获取锁。此时原线程执行完毕后释放锁，若不检查锁的归属，会误删其他线程的锁。

```mermaid
sequenceDiagram
    participant A as 线程A
    participant Redis as Redis
    participant B as 线程B

    A->>Redis: SETNX lock:key value=UUID_A (过期10s)
    Redis-->>A: 获取锁成功
    Note over A: 执行业务逻辑...<br/>超过10s
    Redis->>Redis: 锁自动过期
    B->>Redis: SETNX lock:key value=UUID_B
    Redis-->>B: 获取锁成功
    Note over B: 执行业务逻辑...
    
    alt 不检查 value 直接删除
        A->>Redis: DEL lock:key
        Redis-->>A: 删除成功
        Note over B: 锁被误删！B 的业务失去保护
    else 检查 value 后删除
        A->>Redis: Lua: if get(key)==UUID_A then del(key)
        Redis-->>A: 返回 0 (未删除)
        Note over B: 锁安全，B 继续执行
    end
```

**误删场景示例**：

| 时间 | 线程A | 线程B | Redis 锁状态 |
|------|-------|-------|-------------|
| T1 | 获取锁，value=UUID_A | - | lock:key = UUID_A |
| T2 | 执行业务逻辑... | - | lock:key = UUID_A |
| T10 | (仍在执行) | - | 锁过期自动释放 |
| T11 | (仍在执行) | 获取锁，value=UUID_B | lock:key = UUID_B |
| T12 | 执行完毕，尝试释放锁 | 执行业务逻辑... | 若不检查value，锁被误删 |

**双重检测优化**：

```java
public User getUserWithDoubleCheck(Long id) {
    String key = "user:" + id;
    String lockKey = "lock:user:" + id;
    String lockValue = UUID.randomUUID().toString();
    
    String value = redis.get(key);
    if (value != null) {
        return JSON.parseObject(value, User.class);
    }
    
    int maxRetry = 10;
    for (int i = 0; i < maxRetry; i++) {
        boolean locked = redis.setnx(lockKey, lockValue, 10);
        if (locked) {
            try {
                value = redis.get(key);
                if (value != null) {
                    return JSON.parseObject(value, User.class);
                }
                
                User user = userMapper.selectById(id);
                if (user != null) {
                    redis.set(key, JSON.toJSONString(user), 3600);
                }
                return user;
            } finally {
                String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                               "return redis.call('del', KEYS[1]) " +
                               "else return 0 end";
                redis.eval(script, Arrays.asList(lockKey), Arrays.asList(lockValue));
            }
        }
        
        try {
            Thread.sleep(50);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        }
    }
    
    return null;
}
```

**循环模式的优势**：

| 优势 | 说明 |
|------|------|
| **避免栈溢出** | 无递归调用，不会 StackOverflowError |
| **可控重试次数** | 通过 maxRetry 限制最大重试次数 |
| **资源消耗低** | 无额外栈帧开销 |

#### 方案二：逻辑过期

##### 方案设计理念

逻辑过期方案的核心思想是：**热点数据永不过期，通过逻辑过期时间判断是否需要更新**。

```mermaid
flowchart TB
    subgraph Process["逻辑过期方案流程"]
        A["请求查询缓存"] --> B{"缓存是否存在?"}
        B -->|"否"| C["返回空或默认值"]
        B -->|"是"| D{"逻辑过期时间<br/>是否过期?"}
        D -->|"否"| E["返回数据"]
        D -->|"是"| F["尝试获取锁"]
        F --> G{"获取锁成功?"}
        G -->|"是"| H["开启异步线程<br/>更新缓存"]
        G -->|"否"| I["返回旧数据"]
        H --> E
        I --> E
    end
    
    style Process fill:#e3f2fd,stroke:#1565c0
```

##### 为什么缓存不存在直接返回 null？

```mermaid
flowchart TB
    subgraph Scenario["场景分析"]
        Q1["缓存不存在的原因"]
        A1["数据本身不存在"]
        A2["缓存未预热"]
        A3["缓存被清空"]
    end
    
    subgraph Reason["设计原因"]
        B1["逻辑过期方案适用于热点数据"]
        B2["热点数据应提前预热到缓存"]
        B3["缓存不存在说明非热点或数据无效"]
        B4["直接返回 null 避免缓存穿透"]
    end
    
    subgraph Solution["解决方案"]
        C1["数据不存在 → 返回 null 正确"]
        C2["缓存未预热 → 需要提前预热"]
        C3["缓存清空 → 需要重新预热"]
    end
    
    Scenario --> Reason
    Reason --> Solution
    
    style Scenario fill:#e3f2fd,stroke:#1565c0
    style Reason fill:#fff3e0,stroke:#ef6c00
    style Solution fill:#c8e6c9,stroke:#2e7d32
```

**关键说明**：

| 情况 | 原因 | 处理方式 |
|------|------|----------|
| **缓存不存在** | 数据本身不存在或未预热 | 直接返回 null（非热点数据） |
| **缓存存在但未过期** | 数据仍然有效 | 直接返回缓存数据 |
| **缓存存在但已过期** | 需要更新 | 返回旧数据 + 异步更新 |

##### 为什么获取不到锁返回旧数据？

```mermaid
sequenceDiagram
    participant T1 as 线程1（获取锁成功）
    participant T2 as 线程2（获取锁失败）
    participant Lock as 分布式锁
    participant Cache as 缓存
    participant DB as 数据库
    
    Note over T1,T2: 缓存数据逻辑过期，但旧数据仍可用
    
    T1->>Lock: 尝试获取锁
    Lock-->>T1: 获取成功
    T2->>Lock: 尝试获取锁
    Lock-->>T2: 获取失败
    
    par 线程1异步更新
        T1->>DB: 查询最新数据
        T1->>Cache: 更新缓存
        T1->>Lock: 释放锁
    and 线程2直接返回
        T2->>Cache: 读取旧数据
        T2-->>T2: 立即返回（无需等待）
    end
    
    Note over T1,T2: 线程2返回旧数据，保证性能<br/>线程1更新后，后续请求获得新数据
```

**返回旧数据的优势**：

| 优势 | 说明 |
|------|------|
| **无阻塞** | 用户不需要等待数据库查询和缓存更新 |
| **高性能** | 所有请求都能快速响应 |
| **最终一致** | 异步更新后，后续请求获得最新数据 |
| **避免击穿** | 不会大量请求打到数据库 |

**适用场景**：

| 场景 | 是否适用 | 原因 |
|------|----------|------|
| **热点数据** | 适用 | 数据已预热，缓存中始终存在 |
| **允许短暂过期** | 适用 | 可以接受返回稍旧的数据 |
| **高并发场景** | 适用 | 无阻塞，性能最优 |
| **强一致性要求** | 不适用 | 需要使用互斥锁方案 |

##### 数据结构设计：

```java
@Data
public class CacheData<T> {
    private T data;
    private Long expireTime;
}
```

**代码示例**：

```java
public User getUserWithLogicalExpire(Long id) {
    String key = "user:" + id;
    String lockKey = "lock:user:" + id;
    
    String value = redis.get(key);
    if (value == null) {
        return null;
    }
    
    CacheData<User> cacheData = JSON.parseObject(value, 
        new TypeReference<CacheData<User>>(){});
    
    if (cacheData.getExpireTime() > System.currentTimeMillis()) {
        return cacheData.getData();
    }
    
    if (redis.setnx(lockKey, "1", 10)) {
        executorService.submit(() -> {
            try {
                User user = userMapper.selectById(id);
                CacheData<User> newData = new CacheData<>();
                newData.setData(user);
                newData.setExpireTime(System.currentTimeMillis() + 3600000L);
                redis.set(key, JSON.toJSONString(newData));
            } finally {
                redis.del(lockKey);
            }
        });
    }
    
    return cacheData.getData();
}
```

#### 方案三：热点 Key 永不过期

```java
public void cacheHotData(Long id) {
    User user = userMapper.selectById(id);
    if (user != null) {
        redis.set("user:" + id, JSON.toJSONString(user));
    }
}

@Scheduled(fixedRate = 3600000)
public void refreshHotData() {
    List<Long> hotIds = getHotUserIds();
    for (Long id : hotIds) {
        cacheHotData(id);
    }
}
```

### 3.3 方案对比

| 方案       | 一致性  | 性能    | 复杂度 | 适用场景      |
| -------- | ---- | ----- | --- | --------- |
| **互斥锁**  | 强一致  | 有等待延迟 | 中   | 对一致性要求高   |
| **逻辑过期** | 最终一致 | 无阻塞   | 高   | 对性能要求高    |
| **永不过期** | 最终一致 | 最高    | 低   | 热点数据、定时更新 |

***

## 四、缓存雪崩

### 4.1 问题场景

```mermaid
sequenceDiagram
    participant Clients as 大量客户端
    participant Redis as Redis 缓存
    participant DB as 数据库
    
    Note over Redis: 大量 Key 同时过期<br/>或 Redis 宕机
    
    Clients->>Redis: 大量并发请求
    Redis-->>Clients: 缓存大面积失效
    
    Clients->>DB: 所有请求穿透到数据库
    
    Note over DB: 数据库瞬间崩溃
```

**缓存雪崩的原因**：

| 原因           | 说明                |
| ------------ | ----------------- |
| **同时过期**     | 大量 Key 设置了相同的过期时间 |
| **Redis 宕机** | Redis 服务故障，缓存不可用  |
| **缓存预热失败**   | 系统启动时缓存未预热        |

### 4.2 解决方案

#### 方案一：过期时间随机化

```mermaid
flowchart TB
    subgraph Problem["问题：相同过期时间"]
        K1["Key1 过期时间: 3600s"]
        K2["Key2 过期时间: 3600s"]
        K3["Key3 过期时间: 3600s"]
        K4["Key4 过期时间: 3600s"]
        Same["同一时刻过期"]
    end
    
    subgraph Solution["解决：随机过期时间"]
        S1["Key1 过期时间: 3600s"]
        S2["Key2 过期时间: 3700s"]
        S3["Key3 过期时间: 3500s"]
        S4["Key4 过期时间: 3800s"]
        Diff["分散过期时间点"]
    end
    
    Problem --> Solution
    
    style Problem fill:#ffcdd2,stroke:#c62828
    style Solution fill:#c8e6c9,stroke:#2e7d32
```

**代码示例**：

```java
public void setWithRandomExpire(String key, String value, long baseExpire) {
    Random random = new Random();
    long randomExpire = baseExpire + random.nextInt(600);
    redis.set(key, value, randomExpire);
}
```

#### 方案二：多级缓存

```mermaid
flowchart TB
    subgraph Architecture["多级缓存架构"]
        subgraph L1["一级缓存（本地）"]
            Caffeine["Caffeine<br/>进程内缓存"]
        end
        
        subgraph L2["二级缓存（分布式）"]
            Redis["Redis<br/>分布式缓存"]
        end
        
        subgraph L3["数据源"]
            DB["MySQL<br/>数据库"]
        end
    end
    
    Request["请求"] --> Caffeine
    Caffeine -->|"未命中"| Redis
    Redis -->|"未命中"| DB
    DB --> Redis
    Redis --> Caffeine
    
    style Architecture fill:#e3f2fd,stroke:#1565c0
    style L1 fill:#c8e6c9,stroke:#2e7d32
    style L2 fill:#fff3e0,stroke:#ef6c00
    style L3 fill:#ffcdd2,stroke:#c62828
```

**Spring Cache + Caffeine + Redis 实现**：

##### Maven 依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>com.github.ben-manes.caffeine</groupId>
        <artifactId>caffeine</artifactId>
    </dependency>
</dependencies>
```

##### 配置类实现

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        List<CacheManager> cacheManagers = new ArrayList<>();
        
        CaffeineCacheManager caffeineCacheManager = new CaffeineCacheManager();
        caffeineCacheManager.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(300, TimeUnit.SECONDS)
            .maximumSize(1000));
        cacheManagers.add(caffeineCacheManager);
        
        RedisCacheConfiguration redisConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofSeconds(3600))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        RedisCacheManager redisCacheManager = RedisCacheManager.builder(factory)
            .cacheDefaults(redisConfig)
            .build();
        cacheManagers.add(redisCacheManager);
        
        return new CompositeCacheManager(cacheManagers.toArray(new CacheManager[0]));
    }
}
```

##### 使用示例

```java
@Service
public class UserService {
    
    @Cacheable(value = "user", key = "#id")
    public User getUserById(Long id) {
        return userMapper.selectById(id);
    }
    
    @CacheEvict(value = "user", key = "#id")
    public void updateUser(User user) {
        userMapper.updateById(user);
    }
}
```

##### 关键说明

| 组件 | 作用 | 说明 |
|------|------|------|
| **CompositeCacheManager** | 组合多个缓存管理器 | 按顺序查找：先查 Caffeine，再查 Redis |
| **CaffeineCacheManager** | 本地缓存管理器 | L1 缓存，速度快但容量有限 |
| **RedisCacheManager** | 分布式缓存管理器 | L2 缓存，支持分布式共享 |
| **@EnableCaching** | 启用缓存注解 | 必须添加才能使用 @Cacheable 等注解 |

##### 多级缓存工作流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Service as 服务层<br/>@Cacheable
    participant Cache as Spring Cache<br/>CompositeCacheManager
    participant Caffeine as L1 本地缓存
    participant Redis as L2 Redis缓存
    participant DB as 数据库
    
    Client->>Service: 查询 user:1
    Service->>Cache: @Cacheable 触发
    Cache->>Caffeine: 查询 L1 缓存
    
    alt L1 命中
        Caffeine-->>Cache: 返回数据
        Cache-->>Service: 返回数据
        Service-->>Client: 返回数据
    else L1 未命中
        Cache->>Redis: 查询 L2 缓存
        alt L2 命中
            Redis-->>Cache: 返回数据
            Cache->>Caffeine: 自动写入 L1 缓存
            Cache-->>Service: 返回数据
            Service-->>Client: 返回数据
        else L2 未命中
            Service->>DB: 查询数据库
            DB-->>Service: 返回数据
            Service-->>Cache: 缓存数据
            Cache->>Redis: 自动写入 L2 缓存
            Cache->>Caffeine: 自动写入 L1 缓存
            Service-->>Client: 返回数据
        end
    end
```

#### 方案三：熔断降级

```mermaid
flowchart TB
    subgraph CircuitBreaker["熔断降级机制"]
        subgraph Normal["正常状态"]
            N1["请求查询缓存"]
            N2["缓存未命中查数据库"]
        end
        
        subgraph Degrade["降级状态"]
            D1["检测到数据库压力过大"]
            D2["开启熔断"]
            D3["返回默认值或错误"]
        end
        
        subgraph Recover["恢复状态"]
            R1["数据库压力降低"]
            R2["关闭熔断"]
            R3["恢复正常访问"]
        end
    end
    
    Normal --> Degrade
    Degrade --> Recover
    
    style CircuitBreaker fill:#e3f2fd,stroke:#1565c0
    style Normal fill:#c8e6c9,stroke:#2e7d32
    style Degrade fill:#ffcdd2,stroke:#c62828
    style Recover fill:#fff3e0,stroke:#ef6c00
```

**Sentinel 熔断配置**：

```java
@Service
public class UserService {
    
    @SentinelResource(value = "getUserById", 
        blockHandler = "handleBlock",
        fallback = "handleFallback")
    public User getUserById(Long id) {
        return userMapper.selectById(id);
    }
    
    public User handleBlock(Long id, BlockException ex) {
        return getDefaultUser();
    }
    
    public User handleFallback(Long id, Throwable ex) {
        return getDefaultUser();
    }
    
    private User getDefaultUser() {
        User user = new User();
        user.setId(-1L);
        user.setName("默认用户");
        return user;
    }
}
```

#### 方案四：Redis 高可用

```mermaid
flowchart TB
    subgraph HA["Redis 高可用架构"]
        subgraph Sentinel["哨兵模式"]
            S1["Sentinel 1"]
            S2["Sentinel 2"]
            S3["Sentinel 3"]
        end
        
        subgraph RedisNodes["Redis 节点"]
            Master["Master<br/>主节点"]
            Slave1["Slave 1<br/>从节点"]
            Slave2["Slave 2<br/>从节点"]
        end
    end
    
    Sentinel --> Master
    Master --> Slave1
    Master --> Slave2
    
    Sentinel -->|"监控 + 故障转移"| RedisNodes
    
    style HA fill:#e3f2fd,stroke:#1565c0
    style Sentinel fill:#fff3e0,stroke:#ef6c00
    style RedisNodes fill:#c8e6c9,stroke:#2e7d32
```

### 4.3 方案对比

| 方案            | 作用     | 优点     | 缺点            |
| ------------- | ------ | ------ | ------------- |
| **过期时间随机化**   | 防止同时过期 | 实现简单   | 无法防止 Redis 宕机 |
| **多级缓存**      | 多层保护   | 高可用    | 数据一致性复杂       |
| **熔断降级**      | 保护数据库  | 防止级联故障 | 用户体验下降        |
| **Redis 高可用** | 防止宕机   | 自动故障转移 | 架构复杂          |

***

## 五、综合对比与最佳实践

### 5.1 三大问题对比总结

```mermaid
flowchart TB
    subgraph Comparison["三大问题对比"]
        subgraph Penetration["缓存穿透"]
            P1["场景：查询不存在的数据"]
            P2["特征：缓存和数据库都没有"]
            P3["方案：布隆过滤器、缓存空值"]
        end
        
        subgraph Breakdown["缓存击穿"]
            B1["场景：热点 Key 过期"]
            B2["特征：单点热点，高并发"]
            B3["方案：互斥锁、逻辑过期"]
        end
        
        subgraph Avalanche["缓存雪崩"]
            A1["场景：大量 Key 同时过期"]
            A2["特征：大面积失效"]
            A3["方案：随机过期、多级缓存、熔断"]
        end
    end
    
    style Comparison fill:#e3f2fd,stroke:#1565c0
    style Penetration fill:#ffcdd2,stroke:#c62828
    style Breakdown fill:#fff3e0,stroke:#ef6c00
    style Avalanche fill:#f3e5f5,stroke:#7b1fa2
```

### 5.2 最佳实践

| 场景       | 推荐方案                  |
| -------- | --------------------- |
| **防止穿透** | 布隆过滤器 + 接口参数校验        |
| **防止击穿** | 互斥锁（强一致）或 逻辑过期（高性能）   |
| **防止雪崩** | 过期时间随机化 + 多级缓存 + 熔断降级 |
| **高可用**  | Redis 集群/哨兵 + 本地缓存兜底  |

### 5.3 面试高频问题

| 问题              | 答案要点                      |
| --------------- | ------------------------- |
| **三大问题区别**      | 穿透是数据不存在，击穿是热点过期，雪崩是大面积失效 |
| **布隆过滤器原理**     | 位数组 + 多哈希，存在误判但不会漏判       |
| **互斥锁 vs 逻辑过期** | 互斥锁强一致有阻塞，逻辑过期高性能最终一致     |
| **如何设计缓存**      | 多级缓存 + 随机过期 + 熔断降级 + 高可用  |

***

## 参考资料

- [Redis 缓存穿透、击穿、雪崩解决方案](https://blog.csdn.net/m0_73784704/article/details/152006873)
- [Redis 布隆过滤器原理与应用](https://blog.csdn.net/zhang_hao_chao/article/details/132219157)
- [Redis 缓存三大核心问题](https://blog.csdn.net/weixin_43290370/article/details/154689046)

