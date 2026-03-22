# Redis 抢购票据解决方案

## 问题场景

**需求描述**：用户a、b、c抢购票据A，需要解决两个核心问题：
1. **防止超卖**：票据只能被一个人抢到
2. **防止重复抢购**：同一用户不能重复抢购同一票据

---

## 核心思路

```
┌─────────────────────────────────────────────────────────────┐
│                      抢购流程                                │
├─────────────────────────────────────────────────────────────┤
│  1. 检查用户是否已抢购过（防重复）                            │
│  2. 检查库存是否充足（防超卖）                                │
│  3. 扣减库存 + 记录用户购买信息（原子操作）                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 方案概览

| 方案 | 原子性 | 性能 | 复杂度 | 推荐场景 |
|------|--------|------|--------|----------|
| **Lua脚本（推荐）** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 中等 | 复杂业务逻辑，高并发场景 |
| SETNX + 乐观锁 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 简单 | 简单场景，并发量不高 |
| 分布式锁 | ⭐⭐⭐⭐ | ⭐⭐⭐ | 较高 | 需要强一致性的场景 |

---

## Redis 数据结构设计

### 是否需要存储用户信息？

**答案：需要！** 原因如下：

| 存储内容 | 作用 | 必要性 |
|----------|------|--------|
| 用户购买记录 | 防止同一用户重复抢购 | ⭐⭐⭐⭐⭐ 必须 |
| 票据持有者 | 记录票据归属，便于查询 | ⭐⭐⭐⭐ 推荐 |
| 库存数量 | 防止超卖 | ⭐⭐⭐⭐⭐ 必须 |

### Key设计规范

```
┌─────────────────────────────────────────────────────────────┐
│                    Redis Key 设计                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 库存存储                                                 │
│     Key:   ticket:{票据ID}:stock                            │
│     Value: 库存数量                                          │
│     示例: ticket:A:stock = 1                                │
│                                                              │
│  2. 票据持有者                                               │
│     Key:   ticket:{票据ID}:owner                            │
│     Value: 用户ID                                            │
│     示例: ticket:A:owner = user:a                           │
│                                                              │
│  3. 用户购买记录（防重复）                                    │
│     Key:   user:{用户ID}:tickets                            │
│     Type:  Set                                               │
│     Value: 已购票据ID集合                                     │
│     示例: user:a:tickets = {A, B, C}                        │
│                                                              │
│  4. 用户抢购锁（可选）                                        │
│     Key:   user:{用户ID}:ticket:{票据ID}                    │
│     Value: 1                                                  │
│     TTL:  活动有效期                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 方案一：Lua脚本实现（推荐）

### 原理说明

- **原子性执行**：整个脚本作为一个整体执行，执行期间不会插入其他客户端命令
- **隔离性**：脚本执行期间Redis处于阻塞状态，不会被其他命令打断
- **一次网络往返**：减少网络延迟，提高性能

### Lua脚本代码

```lua
-- 秒杀抢购Lua脚本
-- KEYS[1]: 票据库存key (ticket:A:stock)
-- KEYS[2]: 票据持有者key (ticket:A:owner)
-- KEYS[3]: 用户已购票据集合key (user:a:tickets)
-- ARGV[1]: 用户ID
-- ARGV[2]: 票据ID

local stockKey = KEYS[1]
local ownerKey = KEYS[2]
local userTicketsKey = KEYS[3]
local userId = ARGV[1]
local ticketId = ARGV[2]

-- 1. 检查用户是否已抢购过该票据
if redis.call('SISMEMBER', userTicketsKey, ticketId) == 1 then
    return {0, 'ALREADY_PURCHASED'}  -- 用户已抢购过
end

-- 2. 检查库存
local stock = tonumber(redis.call('GET', stockKey) or 0)
if stock <= 0 then
    return {0, 'SOLD_OUT'}  -- 已售罄
end

-- 3. 原子性操作：扣减库存 + 记录持有者 + 记录用户购买
redis.call('DECR', stockKey)
redis.call('SET', ownerKey, userId)
redis.call('SADD', userTicketsKey, ticketId)

return {1, 'SUCCESS'}  -- 抢购成功
```

### Java + Jedis 实现

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;

public class TicketGrabService {
    
    private final JedisPool jedisPool;
    
    private static final String GRAB_SCRIPT = """
        local stockKey = KEYS[1]
        local ownerKey = KEYS[2]
        local userTicketsKey = KEYS[3]
        local userId = ARGV[1]
        local ticketId = ARGV[2]
        
        -- 检查用户是否已抢购过
        if redis.call('SISMEMBER', userTicketsKey, ticketId) == 1 then
            return {0, 'ALREADY_PURCHASED'}
        end
        
        -- 检查库存
        local stock = tonumber(redis.call('GET', stockKey) or 0)
        if stock <= 0 then
            return {0, 'SOLD_OUT'}
        end
        
        -- 原子性扣减并记录
        redis.call('DECR', stockKey)
        redis.call('SET', ownerKey, userId)
        redis.call('SADD', userTicketsKey, ticketId)
        
        return {1, 'SUCCESS'}
        """;
    
    public TicketGrabService(JedisPool jedisPool) {
        this.jedisPool = jedisPool;
    }
    
    public GrabResult grabTicket(String ticketId, String userId) {
        try (Jedis jedis = jedisPool.getResource()) {
            String stockKey = "ticket:" + ticketId + ":stock";
            String ownerKey = "ticket:" + ticketId + ":owner";
            String userTicketsKey = "user:" + userId + ":tickets";
            
            Object result = jedis.eval(
                GRAB_SCRIPT,
                3,
                stockKey, ownerKey, userTicketsKey,
                userId, ticketId
            );
            
            return GrabResult.fromRedisResult(result);
        }
    }
}
```

### Python + redis-py 实现

```python
import redis

class TicketGrabService:
    
    GRAB_SCRIPT = """
    local stockKey = KEYS[1]
    local ownerKey = KEYS[2]
    local userTicketsKey = KEYS[3]
    local userId = ARGV[1]
    local ticketId = ARGV[2]
    
    if redis.call('SISMEMBER', userTicketsKey, ticketId) == 1 then
        return {0, 'ALREADY_PURCHASED'}
    end
    
    local stock = tonumber(redis.call('GET', stockKey) or 0)
    if stock <= 0 then
        return {0, 'SOLD_OUT'}
    end
    
    redis.call('DECR', stockKey)
    redis.call('SET', ownerKey, userId)
    redis.call('SADD', userTicketsKey, ticketId)
    
    return {1, 'SUCCESS'}
    """
    
    def __init__(self, host='localhost', port=6379):
        self.redis = redis.Redis(host=host, port=port)
    
    def grab_ticket(self, ticket_id: str, user_id: str) -> tuple:
        stock_key = f'ticket:{ticket_id}:stock'
        owner_key = f'ticket:{ticket_id}:owner'
        user_tickets_key = f'user:{user_id}:tickets'
        
        result = self.redis.eval(
            self.GRAB_SCRIPT,
            3,
            stock_key, owner_key, user_tickets_key,
            user_id, ticket_id
        )
        
        return result
```

### Spring Boot + RedisTemplate 实现

```java
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;
import java.util.Arrays;
import java.util.List;

@Service
public class TicketGrabService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    private static final String GRAB_SCRIPT = """
        if redis.call('SISMEMBER', KEYS[3], ARGV[2]) == 1 then
            return {0, 'ALREADY_PURCHASED'}
        end
        local stock = tonumber(redis.call('GET', KEYS[1]) or 0)
        if stock <= 0 then
            return {0, 'SOLD_OUT'}
        end
        redis.call('DECR', KEYS[1])
        redis.call('SET', KEYS[2], ARGV[1])
        redis.call('SADD', KEYS[3], ARGV[2])
        return {1, 'SUCCESS'}
        """;
    
    public TicketGrabService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    
    @SuppressWarnings("unchecked")
    public List<Object> grabTicket(String ticketId, String userId) {
        DefaultRedisScript<List<Object>> script = new DefaultRedisScript<>();
        script.setScriptText(GRAB_SCRIPT);
        script.setResultType(List.class);
        
        List<String> keys = Arrays.asList(
            "ticket:" + ticketId + ":stock",
            "ticket:" + ticketId + ":owner",
            "user:" + userId + ":tickets"
        );
        
        return redisTemplate.execute(script, keys, userId, ticketId);
    }
}
```

---

## 方案二：SETNX + 乐观锁实现

### 适用场景

- 简单场景，不需要复杂逻辑
- 并发量不高的场景

### 实现代码

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;

public class TicketGrabSimpleService {
    
    private final JedisPool jedisPool;
    
    public TicketGrabSimpleService(JedisPool jedisPool) {
        this.jedisPool = jedisPool;
    }
    
    public boolean grabTicket(String ticketId, String userId) {
        try (Jedis jedis = jedisPool.getResource()) {
            String lockKey = "ticket:" + ticketId + ":lock";
            String ownerKey = "ticket:" + ticketId + ":owner";
            String userKey = "user:" + userId + ":ticket:" + ticketId;
            String stockKey = "ticket:" + ticketId + ":stock";
            
            // 1. 检查用户是否已抢购（使用SETNX）
            if (jedis.setnx(userKey, "1") == 0) {
                return false;  // 用户已抢购过
            }
            jedis.expire(userKey, 3600);  // 设置过期时间
            
            // 2. 使用SETNX抢锁
            if (jedis.setnx(lockKey, userId) == 1) {
                jedis.expire(lockKey, 10);  // 短暂锁定
                
                try {
                    // 3. 检查库存
                    String stock = jedis.get(stockKey);
                    if (stock != null && Integer.parseInt(stock) > 0) {
                        // 4. 扣减库存
                        jedis.decr(stockKey);
                        jedis.set(ownerKey, userId);
                        return true;
                    }
                } finally {
                    // 5. 释放锁
                    jedis.del(lockKey);
                }
            }
            
            return false;
        }
    }
}
```

---

## 完整流程图

```
用户请求抢购票据A
       │
       ▼
┌──────────────────┐
│ 检查用户是否已抢购 │
│ (SISMEMBER)      │
└────────┬─────────┘
         │
    ┌────┴────┐
    │ 已抢购？ │
    └────┬────┘
         │
    ┌────┴────┐
   是│        │否
    ▼         ▼
 返回失败   ┌──────────────────┐
           │ 检查库存是否充足  │
           │ (GET stock)      │
           └────────┬─────────┘
                    │
               ┌────┴────┐
               │ 库存>0？ │
               └────┬────┘
                    │
               ┌────┴────┐
              是│        │否
               ▼         ▼
        ┌────────────┐  返回售罄
        │ 执行Lua脚本 │
        │ (原子操作)  │
        └─────┬──────┘
              │
              ▼
        ┌─────────────────────┐
        │ 1. DECR 扣减库存     │
        │ 2. SET 记录持有者    │
        │ 3. SADD 记录用户购买 │
        └─────────┬───────────┘
                  │
                  ▼
              返回成功
```

---

## 最佳实践

### 1. 初始化数据

活动开始前，将库存预热到Redis：

```bash
# 初始化票据库存
SET ticket:A:stock 1

# 设置票据信息（可选）
HSET ticket:A:info name "演唱会门票" price 100
```

```java
public void preloadStock(String ticketId, int stock) {
    try (Jedis jedis = jedisPool.getResource()) {
        jedis.set("ticket:" + ticketId + ":stock", String.valueOf(stock));
    }
}
```

### 2. 设置过期时间

```java
// 用户购买记录设置过期时间（活动结束后自动清理）
jedis.expire("user:" + userId + ":tickets", 86400);
```

### 3. 异步落库

抢购成功后，发送消息到MQ异步写入数据库：

```java
if (grabResult.isSuccess()) {
    OrderMessage message = new OrderMessage(userId, ticketId, System.currentTimeMillis());
    rabbitTemplate.convertAndSend("order.queue", message);
}
```

### 4. 使用EVALSHA优化性能

```java
// 首次加载脚本，获取SHA1
String scriptSha = jedis.scriptLoad(GRAB_SCRIPT);

// 后续使用SHA1执行
jedis.evalsha(scriptSha, 3, stockKey, ownerKey, userTicketsKey, userId, ticketId);
```

### 5. 库存回滚处理

```java
// 如果订单创建失败，需要回滚库存
public void rollbackStock(String ticketId, String userId) {
    String rollbackScript = """
        redis.call('INCR', KEYS[1])
        redis.call('DEL', KEYS[2])
        redis.call('SREM', KEYS[3], ARGV[1])
        return 'OK'
        """;
    
    jedis.eval(
        rollbackScript,
        3,
        "ticket:" + ticketId + ":stock",
        "ticket:" + ticketId + ":owner",
        "user:" + userId + ":tickets",
        ticketId
    );
}
```

---

## 常见问题

### Q1: 为什么不用数据库直接扣库存？

**答**：数据库在高并发下容易出现超卖问题，Redis内存操作性能更高，适合高并发场景。

### Q2: Lua脚本执行时间过长会怎样？

**答**：Lua脚本执行期间会阻塞Redis，建议脚本尽量简短，避免复杂计算。可以通过 `lua-time-limit` 配置设置超时时间。

### Q3: 如何处理Redis宕机？

**答**：建议使用Redis主从或集群模式，配合持久化配置（AOF/RDB）保证数据安全。

### Q4: 用户购买记录用什么数据结构？

**答**：推荐使用 **Set** 类型，原因：
- O(1) 时间复杂度判断是否已购买
- 自动去重
- 支持批量查询

---

## 参考资料

- [Redis+Lua秒杀实战:彻底解决库存超卖](https://blog.csdn.net/m0_50859212/article/details/151054827)
- [Redis解决秒杀微服务抢购代金券超卖](https://blog.csdn.net/2401_86963291/article/details/142155589)
- [秒杀重复下单详解](https://blog.csdn.net/T_Y_F_/article/details/144227164)
- [springBoot+Lua+redis实现限量秒杀抢购模块](https://blog.csdn.net/yb223731/article/details/120854728)
- [Redis官方文档 - Lua Scripting](https://redis.io/docs/interact/programmability/eval-intro/)
