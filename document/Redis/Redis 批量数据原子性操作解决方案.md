# Redis 批量数据原子性操作解决方案

## 问题场景

**需求描述**：有一批票据数据A和B，服务程序要么将这些全部放进Redis，要么都不放进Redis。

**核心问题**：如何保证多个Redis操作的原子性？

---

## 方案概览

| 方案 | 原子性 | 性能 | 复杂度 | 适用场景 |
|------|--------|------|--------|----------|
| **Lua脚本（推荐）** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 中等 | 复杂业务逻辑，需要真正原子性 |
| MULTI/EXEC事务 | ⭐⭐⭐ | ⭐⭐⭐⭐ | 简单 | 简单批量操作，无运行时错误风险 |
| WATCH+事务 | ⭐⭐⭐⭐ | ⭐⭐⭐ | 较高 | 需要乐观锁控制的并发场景 |
| MSET命令 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 最低 | 仅简单SET操作 |

---

## 方案一：Lua脚本（最佳实践）

### 原理说明

Redis执行Lua脚本时具有以下特性：

- **原子性执行**：整个脚本作为一个整体执行，执行期间不会插入其他客户端命令
- **隔离性**：脚本执行期间Redis处于阻塞状态，不会被其他命令打断
- **失败回滚**：如果脚本中任何一步失败，整个脚本执行失败

### 代码示例

#### Java + Jedis 实现

```java
import redis.clients.jedis.Jedis;

public class RedisAtomicOperation {
    
    private static final String LUA_SCRIPT = 
        "-- 批量存储票据数据\n" +
        "local ticketA = ARGV[1]\n" +
        "local ticketB = ARGV[2]\n" +
        "\n" +
        "-- 参数校验\n" +
        "if ticketA == nil or ticketB == nil then\n" +
        "    return redis.error_reply('ERR: ticket data cannot be nil')\n" +
        "end\n" +
        "\n" +
        "-- 原子性存储\n" +
        "redis.call('SET', KEYS[1], ticketA)\n" +
        "redis.call('SET', KEYS[2], ticketB)\n" +
        "\n" +
        "-- 设置过期时间（可选）\n" +
        "redis.call('EXPIRE', KEYS[1], 3600)\n" +
        "redis.call('EXPIRE', KEYS[2], 3600)\n" +
        "\n" +
        "return 'OK'";

    public void saveTicketsAtomic(String ticketA, String ticketB) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {
            Object result = jedis.eval(
                LUA_SCRIPT, 
                2,                              // KEYS数量
                "ticket:A", "ticket:B",         // KEYS
                ticketA, ticketB                // ARGV
            );
            System.out.println("执行结果: " + result);
        }
    }
}
```

#### Python + redis-py 实现

```python
import redis

class RedisAtomicOperation:
    
    LUA_SCRIPT = """
    -- 批量存储票据数据
    local ticketA = ARGV[1]
    local ticketB = ARGV[2]
    
    -- 参数校验
    if ticketA == nil or ticketB == nil then
        return redis.error_reply('ERR: ticket data cannot be nil')
    end
    
    -- 原子性存储
    redis.call('SET', KEYS[1], ticketA)
    redis.call('SET', KEYS[2], ticketB)
    
    -- 设置过期时间（可选）
    redis.call('EXPIRE', KEYS[1], 3600)
    redis.call('EXPIRE', KEYS[2], 3600)
    
    return 'OK'
    """
    
    def __init__(self, host='localhost', port=6379):
        self.redis = redis.Redis(host=host, port=port)
    
    def save_tickets_atomic(self, ticket_a: str, ticket_b: str):
        result = self.redis.eval(
            self.LUA_SCRIPT, 
            2, 
            'ticket:A', 'ticket:B', 
            ticket_a, ticket_b
        )
        return result
```

#### Spring Boot + RedisTemplate 实现

```java
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;
import java.util.Arrays;
import java.util.List;

@Service
public class TicketService {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    private static final String LUA_SCRIPT = """
        redis.call('SET', KEYS[1], ARGV[1])
        redis.call('SET', KEYS[2], ARGV[2])
        return 'OK'
        """;
    
    public TicketService(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    
    public void saveTicketsAtomic(String ticketA, String ticketB) {
        DefaultRedisScript<String> script = new DefaultRedisScript<>();
        script.setScriptText(LUA_SCRIPT);
        script.setResultType(String.class);
        
        List<String> keys = Arrays.asList("ticket:A", "ticket:B");
        redisTemplate.execute(script, keys, ticketA, ticketB);
    }
}
```

### 优缺点分析

| 优点 | 缺点 |
|------|------|
| ✅ 真正的原子性保证 | ⚠️ Lua脚本不能太长，会阻塞Redis |
| ✅ 一次网络往返，性能最优 | ⚠️ 调试相对困难 |
| ✅ 支持复杂业务逻辑（条件判断、循环） | ⚠️ 需要学习Lua语法 |
| ✅ 执行期间不会被其他命令打断 | |

---

## 方案二：MULTI/EXEC 事务

### 原理说明

Redis事务通过以下命令实现：

- `MULTI`：开启事务，后续命令入队
- `EXEC`：执行事务队列中的所有命令
- `DISCARD`：取消事务

### 代码示例

#### Java + Jedis 实现

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Transaction;
import java.util.List;

public class RedisTransaction {
    
    public void saveTicketsWithTransaction(String ticketA, String ticketB) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {
            Transaction tx = jedis.multi();
            
            try {
                tx.set("ticket:A", ticketA);
                tx.set("ticket:B", ticketB);
                
                List<Object> results = tx.exec();
                System.out.println("事务执行成功，结果数量: " + results.size());
            } catch (Exception e) {
                tx.discard();
                System.out.println("事务已取消: " + e.getMessage());
            }
        }
    }
}
```

#### Python + redis-py 实现

```python
import redis

def save_tickets_with_transaction(ticket_a: str, ticket_b: str):
    r = redis.Redis(host='localhost', port=6379)
    
    pipe = r.pipeline(transaction=True)
    try:
        pipe.set('ticket:A', ticket_a)
        pipe.set('ticket:B', ticket_b)
        results = pipe.execute()
        print(f"事务执行成功，结果数量: {len(results)}")
    except Exception as e:
        print(f"事务执行失败: {e}")
```

### ⚠️ 重要注意事项

**Redis事务的原子性限制：**

| 错误类型 | 行为 | 是否满足原子性 |
|----------|------|----------------|
| 语法错误（命令不存在、参数错误） | 整个事务被拒绝执行 | ✅ 是 |
| 运行时错误（类型错误，如对String执行LPUSH） | 只有错误命令失败，其他命令继续执行 | ❌ 否 |
| EXEC前连接断开 | 事务自动放弃 | ✅ 是 |

**结论**：MULTI/EXEC 不能保证"运行时错误时的回滚"，如果对原子性要求严格，建议使用Lua脚本。

---

## 方案三：WATCH + MULTI/EXEC（乐观锁）

### 原理说明

`WATCH` 命令用于监视一个或多个key，如果在事务执行前这些key被其他客户端修改，则事务会被放弃。

### 适用场景

- 需要确保数据在操作期间没有被其他客户端修改
- 实现乐观锁机制

### 代码示例

#### Java + Jedis 实现

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Transaction;
import java.util.List;

public class RedisOptimisticLock {
    
    public boolean saveTicketsWithOptimisticLock(String ticketA, String ticketB) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {
            int maxRetries = 3;
            
            for (int i = 0; i < maxRetries; i++) {
                // 监视相关key
                jedis.watch("ticket:A", "ticket:B");
                
                // 开启事务
                Transaction tx = jedis.multi();
                tx.set("ticket:A", ticketA);
                tx.set("ticket:B", ticketB);
                
                // 执行事务
                List<Object> result = tx.exec();
                
                if (result != null) {
                    System.out.println("事务执行成功");
                    return true;
                }
                
                System.out.println("数据被修改，重试第 " + (i + 1) + " 次");
            }
            
            return false;
        }
    }
}
```

#### Python + redis-py 实现

```python
import redis

def save_tickets_with_optimistic_lock(ticket_a: str, ticket_b: str, max_retries: int = 3):
    r = redis.Redis(host='localhost', port=6379)
    
    for attempt in range(max_retries):
        try:
            # 监视key
            r.watch('ticket:A', 'ticket:B')
            
            # 开启事务
            pipe = r.pipeline(transaction=True)
            pipe.set('ticket:A', ticket_a)
            pipe.set('ticket:B', ticket_b)
            
            # 执行事务
            result = pipe.execute()
            
            if result:
                print("事务执行成功")
                return True
                
        except redis.WatchError:
            print(f"数据被修改，重试第 {attempt + 1} 次")
        finally:
            r.unwatch()
    
    return False
```

---

## 方案四：MSET 命令（简单场景）

### 原理说明

`MSET` 是Redis提供的原子性批量设置命令，一次设置多个key-value对。

### 代码示例

```java
// Java + Jedis
jedis.mset("ticket:A", "dataA", "ticket:B", "dataB");
```

```python
# Python + redis-py
r.mset({'ticket:A': 'dataA', 'ticket:B': 'dataB'})
```

### 特点

| 优点 | 缺点 |
|------|------|
| ✅ 原子性操作 | ❌ 只适用于简单的SET操作 |
| ✅ 性能最高，一次网络往返 | ❌ 不支持设置过期时间 |
| ✅ 代码最简单 | ❌ 不支持条件判断等复杂逻辑 |

---

## 方案选择决策树

```
是否需要原子性保证？
├── 否 → 使用Pipeline（性能最优）
└── 是 → 操作是否复杂？
    ├── 简单SET操作 → 使用MSET
    ├── 需要乐观锁 → 使用WATCH + MULTI/EXEC
    └── 复杂业务逻辑 → 使用Lua脚本（推荐）
```

---

## 最佳实践建议

### 1. 优先选择Lua脚本

对于票据数据场景，**强烈推荐使用Lua脚本**：

```lua
-- 完整的票据存储Lua脚本
local ticketA = ARGV[1]
local ticketB = ARGV[2]
local expireTime = tonumber(ARGV[3]) or 3600

-- 参数校验
if not ticketA or not ticketB then
    return redis.error_reply('ERR: ticket data cannot be empty')
end

-- 业务校验（示例：检查是否已存在）
if redis.call('EXISTS', KEYS[1]) == 1 then
    return redis.error_reply('ERR: ticket A already exists')
end

-- 原子性存储
redis.call('SET', KEYS[1], ticketA, 'EX', expireTime)
redis.call('SET', KEYS[2], ticketB, 'EX', expireTime)

return 'OK'
```

### 2. 注意事项

1. **Lua脚本长度控制**：脚本不宜过长，避免阻塞Redis过久
2. **错误处理**：在Lua脚本中添加适当的错误检查
3. **脚本缓存**：对于频繁使用的脚本，使用 `SCRIPT LOAD` 和 `EVALSHA` 提高性能
4. **超时设置**：设置合理的 `lua-time-limit` 配置

### 3. 性能优化

```java
// 使用EVALSHA提高性能
String sha1 = jedis.scriptLoad(LUA_SCRIPT);
jedis.evalsha(sha1, 2, "ticket:A", "ticket:B", ticketA, ticketB);
```

---

## 参考资料

- [Redis官方文档 - Transactions](https://redis.io/docs/interact/transactions/)
- [Redis官方文档 - Lua Scripting](https://redis.io/docs/interact/programmability/eval-intro/)
- [Redis事务原子性分析](http://m.toutiao.com/group/7553264817981620799/)
- [Redis Lua脚本原子性操作](https://blog.csdn.net/u010316188/article/details/105432027)
