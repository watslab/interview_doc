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

| 问题 | 触发场景 | 影响范围 | 核心特征 |
|------|----------|----------|----------|
| **缓存穿透** | 查询根本不存在的数据 | 中 | 缓存和数据库都没有 |
| **缓存击穿** | 热点 Key 过期瞬间 | 中高 | 单点热点，高并发 |
| **缓存雪崩** | 大量 Key 同时过期或 Redis 宕机 | 最高 | 大面积失效 |

---

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
|------|------|
| **恶意攻击** | 攻击者故意请求不存在的数据，绕过缓存 |
| **业务缺陷** | 业务逻辑错误，产生大量无效查询 |
| **数据删除** | 数据被删除，但缓存未同步清理 |

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

| 优点 | 缺点 |
|------|------|
| 实现简单 | 占用额外内存 |
| 有效减少数据库压力 | 需要设置合理的过期时间 |
| | 可能存在数据不一致 |

#### 方案二：布隆过滤器

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

**布隆过滤器原理**：

```mermaid
flowchart LR
    subgraph Input["输入元素"]
        E1["元素 A"]
        E2["元素 B"]
        E3["元素 C"]
    end
    
    subgraph Hash["哈希函数"]
        H1["Hash1"]
        H2["Hash2"]
        H3["Hash3"]
    end
    
    subgraph BitArray["位数组"]
        B0["0"]
        B1["1"]
        B2["1"]
        B3["0"]
        B4["1"]
        B5["0"]
        B6["1"]
        B7["0"]
    end
    
    E1 --> Hash
    E2 --> Hash
    E3 --> Hash
    Hash --> BitArray
    
    style Input fill:#e3f2fd,stroke:#1565c0
    style Hash fill:#fff3e0,stroke:#ef6c00
    style BitArray fill:#c8e6c9,stroke:#2e7d32
```

**布隆过滤器特点**：

| 特性 | 说明 |
|------|------|
| **空间效率高** | 使用位数组，占用内存小 |
| **查询速度快** | O(k) 时间复杂度，k 为哈希函数数量 |
| **存在误判** | 可能判断存在但实际不存在（假阳性） |
| **不存在误判** | 判断不存在则一定不存在 |

**Redisson 实现示例**：

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

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **缓存空值** | 实现简单 | 占用内存、数据不一致 | 数据量小、穿透频率低 |
| **布隆过滤器** | 内存占用小、效率高 | 存在误判、需要预热 | 数据量大、穿透频率高 |
| **接口校验** | 提前拦截 | 无法防止所有情况 | 参数校验、基础防护 |

---

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

| 原因 | 说明 |
|------|------|
| **热点数据过期** | 高频访问的 Key 恰好过期 |
| **缓存未命中** | 大量并发请求同时发现缓存失效 |
| **无保护机制** | 没有限制并发查询数据库 |

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
    
    try {
        boolean locked = redis.setnx(lockKey, "1", 10);
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
        redis.del(lockKey);
    }
}
```

**双重检测优化**：

```java
public User getUserWithDoubleCheck(Long id) {
    String key = "user:" + id;
    String lockKey = "lock:user:" + id;
    
    String value = redis.get(key);
    if (value != null) {
        return JSON.parseObject(value, User.class);
    }
    
    try {
        boolean locked = redis.setnx(lockKey, "1", 10);
        if (locked) {
            value = redis.get(key);
            if (value != null) {
                return JSON.parseObject(value, User.class);
            }
            
            User user = userMapper.selectById(id);
            if (user != null) {
                redis.set(key, JSON.toJSONString(user), 3600);
            }
            return user;
        } else {
            Thread.sleep(50);
            return getUserWithDoubleCheck(id);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return null;
    } finally {
        redis.del(lockKey);
    }
}
```

#### 方案二：逻辑过期

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

**数据结构设计**：

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

| 方案 | 一致性 | 性能 | 复杂度 | 适用场景 |
|------|--------|------|--------|----------|
| **互斥锁** | 强一致 | 有等待延迟 | 中 | 对一致性要求高 |
| **逻辑过期** | 最终一致 | 无阻塞 | 高 | 对性能要求高 |
| **永不过期** | 最终一致 | 最高 | 低 | 热点数据、定时更新 |

---

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

| 原因 | 说明 |
|------|------|
| **同时过期** | 大量 Key 设置了相同的过期时间 |
| **Redis 宕机** | Redis 服务故障，缓存不可用 |
| **缓存预热失败** | 系统启动时缓存未预热 |

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

```java
@Configuration
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        List<CaffeineCache> caches = new ArrayList<>();
        caches.add(new CaffeineCache("user", 
            Caffeine.newBuilder()
                .expireAfterWrite(300, TimeUnit.SECONDS)
                .maximumSize(1000)
                .build()));
        
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofSeconds(3600));
        
        return new CompositeCacheManager(
            new CaffeineCacheManager(),
            RedisCacheManager.builder(factory).cacheDefaults(config).build()
        );
    }
}
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

| 方案 | 作用 | 优点 | 缺点 |
|------|------|------|------|
| **过期时间随机化** | 防止同时过期 | 实现简单 | 无法防止 Redis 宕机 |
| **多级缓存** | 多层保护 | 高可用 | 数据一致性复杂 |
| **熔断降级** | 保护数据库 | 防止级联故障 | 用户体验下降 |
| **Redis 高可用** | 防止宕机 | 自动故障转移 | 架构复杂 |

---

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

| 场景 | 推荐方案 |
|------|----------|
| **防止穿透** | 布隆过滤器 + 接口参数校验 |
| **防止击穿** | 互斥锁（强一致）或 逻辑过期（高性能） |
| **防止雪崩** | 过期时间随机化 + 多级缓存 + 熔断降级 |
| **高可用** | Redis 集群/哨兵 + 本地缓存兜底 |

### 5.3 面试高频问题

| 问题 | 答案要点 |
|------|----------|
| **三大问题区别** | 穿透是数据不存在，击穿是热点过期，雪崩是大面积失效 |
| **布隆过滤器原理** | 位数组 + 多哈希，存在误判但不会漏判 |
| **互斥锁 vs 逻辑过期** | 互斥锁强一致有阻塞，逻辑过期高性能最终一致 |
| **如何设计缓存** | 多级缓存 + 随机过期 + 熔断降级 + 高可用 |

---

## 参考资料

- [Redis 缓存穿透、击穿、雪崩解决方案](https://blog.csdn.net/m0_73784704/article/details/152006873)
- [Redis 布隆过滤器原理与应用](https://blog.csdn.net/zhang_hao_chao/article/details/132219157)
- [Redis 缓存三大核心问题](https://blog.csdn.net/weixin_43290370/article/details/154689046)
