# Redis 深入

> 你的三个项目都重度使用了 Redis，这是面试高频考点。

---

## 一、数据结构与底层实现

### 1.1 五种基本类型

| 类型 | 底层实现 | 常用场景 |
|------|---------|---------|
| String | SDS（简单动态字符串） | 缓存、计数器、分布式锁 |
| Hash | listpack / hashtable | 对象存储、用户信息 |
| List | quicklist（当前版本通常以 listpack 作为节点） | 消息队列、时间线 |
| Set | intset / hashtable | 去重、交集/并集/差集 |
| ZSet | listpack / skiplist + hashtable | 排行榜、延迟队列 |

> 备注：旧资料常写 `ziplist`，但在较新的 Redis 版本里，小型 Hash / ZSet 主要已经由 `listpack` 替代。

### 1.2 SDS vs C 字符串

- 保存长度（O(1) 获取长度）
- 二进制安全（可以存任意二进制数据）
- 空间预分配 + 惰性释放（减少内存重分配）
- 自动扩容

### 1.3 跳表（SkipList）

ZSet 底层实现，支持 O(log n) 的查找、插入、删除：
- 多层链表，每层是下层的"快速通道"
- 查找时从最高层开始，逐层向下
- 比平衡树实现更简单，范围查询更友好

---

## 二、持久化

### 2.1 RDB（快照）

- 按时间间隔生成内存快照到磁盘
- `save 900 1`：900 秒内至少 1 次修改就触发
- 优点：恢复速度快，文件紧凑
- 缺点：可能丢失最后一次快照后的数据

### 2.2 AOF（追加日志）

- 每次写操作追加到日志文件
- 策略：always（每次）/ everysec（每秒，推荐）/ no
- 优点：数据更完整
- 缺点：文件大，恢复慢

### 2.3 混合持久化（Redis 4.0+）

- AOF 重写时，前半段 RDB 格式 + 后半段 AOF 格式
- 兼顾恢复速度和数据完整性

---

## 三、缓存问题

### 3.1 缓存穿透

**问题**：查询一个数据库中也不存在的数据，每次都穿透到数据库

**解决方案：**
- 缓存空值（设短 TTL，如 5 分钟）
- 布隆过滤器（Bloom Filter）：请求先过布隆过滤器，不存在直接返回

### 3.2 缓存击穿

**问题**：热点 key 过期瞬间，大量并发请求同时穿透到数据库

**解决方案：**
- 互斥锁：只有一个线程去查数据库并更新缓存
- 逻辑过期：不设真实 TTL，在 value 中存过期时间，过期后异步更新
- 永不过期：热点数据手动管理更新

### 3.3 缓存雪崩

**问题**：大量 key 同时过期，大量请求同时穿透

**解决方案：**
- 过期时间加随机值，打散过期时间
- 多级缓存（本地缓存 + Redis）
- 熔断降级

### 3.4 你项目中的缓存策略

```java
// 6 小时 TTL 缓存（变电 3.0）
redisTemplate.opsForValue().set(key, result, 6, TimeUnit.HOURS);

// 外部 API Token 缓存（110 分钟，小于实际有效期）
redisTemplate.opsForValue().set("zt-token", token, 110, TimeUnit.MINUTES);

// 缓存注册表，用于批量清理
redisTemplate.opsForList().rightPush("cache:registry:" + module, key);
```

---

## 四、分布式锁

### 4.1 Redis 分布式锁原理

```bash
# 加锁（原子操作）
SET lock_key unique_token NX PX 20000
# NX：不存在才设置（互斥）
# PX：设置过期时间（防死锁）
# unique_token：UUID（防误释放）

# 释放锁（Lua 脚本保证原子性）
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### 4.2 你项目中的实现

```java
private static final DefaultRedisScript<Long> UNLOCK_SCRIPT =
    new DefaultRedisScript<>(
        "if redis.call('get', KEYS[1]) == ARGV[1] then " +
        " return redis.call('del', KEYS[1]) " +
        "else return 0 end",
        Long.class
    );

// RedisLockUtil - 非阻塞锁
public String tryLock(String lockKey, long expireTime) {
    String token = UUID.randomUUID().toString();
    Boolean success = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, token, expireTime, TimeUnit.MILLISECONDS);
    return Boolean.TRUE.equals(success) ? token : null;
}

// 阻塞锁（带超时重试）
public String lock(String lockKey, long expireTime, long timeout) {
    long start = System.currentTimeMillis();
    while (System.currentTimeMillis() - start < timeout) {
        String token = tryLock(lockKey, expireTime);
        if (token != null) return token;
        Thread.sleep(100);  // 每 100ms 重试
    }
    return null;
}

// 释放锁（Lua 脚本原子校验并删除，不要先 get 再 delete）
public void unlock(String lockKey, String token) {
    redisTemplate.execute(
        UNLOCK_SCRIPT,
        Collections.singletonList(lockKey),
        token
    );
}
```

### 4.3 分布式锁的问题和进阶

**问题 1：锁过期但业务没执行完？**
- Redisson 看门狗机制：默认 30 秒过期，后台线程每 10 秒续期
- 你的项目中通过设置足够长的过期时间（20 秒）来规避

**问题 2：Redis 主从切换导致锁丢失？**
- RedLock 算法：向多个独立 Redis 实例加锁，多数成功才算加锁成功
- 实际项目中如果不是强一致性要求，单实例锁 + 幂等设计就够了

---

## 五、Redis Lua 限流

### 5.1 令牌桶算法（你项目中的实现）

```java
// RateLimiterUtil - 基于 Lua 的令牌桶
rateLimiterUtil.tryAcquire(key, capacity, rate, requested);
// key: 限流标识
// capacity: 桶容量（突发流量上限）
// rate: 每秒补充令牌数
// requested: 本次请求消耗的令牌数
```

**Lua 脚本核心逻辑：**
```lua
-- 1. 读取当前令牌数和上次补充时间
local tokens = tonumber(redis.call('get', token_key) or capacity)
local last_time = tonumber(redis.call('get', time_key) or now)

-- 2. 计算应补充的令牌数
local elapsed = now - last_time
local new_tokens = math.min(capacity, tokens + elapsed * rate)

-- 3. 判断是否有足够令牌
if new_tokens >= requested then
    new_tokens = new_tokens - requested
    redis.call('set', token_key, new_tokens, 'PX', ttl)
    redis.call('set', time_key, now, 'PX', ttl)
    return 1  -- 通过
else
    return 0  -- 拒绝
end
```

### 5.2 常见限流算法对比

| 算法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| 固定窗口 | 时间窗内计数 | 简单 | 窗口临界突发 |
| 滑动窗口 | 细粒度时间窗 | 平滑 | 实现稍复杂 |
| 漏桶 | 固定速率处理 | 流量平滑 | 无法应对突发 |
| 令牌桶 | 固定速率补充，允许突发 | 允许一定突发 | 实现略复杂 |

---

## 六、Redis 集群

### 6.1 主从复制

- 从节点发送 PSYNC 命令给主节点
- 全量同步：主节点 RDB 快照 → 发送给从节点
- 增量同步：主节点将写命令通过 replication buffer 发送给从节点
- 作用：读写分离、数据备份

### 6.2 哨兵（Sentinel）

- 监控主从节点状态
- 主节点故障时自动故障转移（选举新主）
- 通知客户端新主节点地址
- 选举算法：过滤掉不健康节点 → 优先级 → 复制偏移量 → runid 最小

### 6.3 Cluster 集群

- 数据分片：16384 个 slot，每个节点负责部分 slot
- 客户端路由：key 通过 CRC16(key) % 16384 计算 slot
- 如果 key 不在当前节点，返回 MOVED 重定向
- 每个主节点可以有多个从节点（高可用）

---

## 七、Redis 其他高频问题

### 7.1 Redis 为什么这么快？

1. 纯内存操作
2. 单线程避免上下文切换和锁竞争（6.0 后 IO 多线程，命令执行仍单线程）
3. IO 多路复用（epoll）
4. 高效数据结构（SDS、listpack、skiplist）

### 7.2 Redis 过期策略

- **惰性删除**：访问时检查是否过期
- **定期删除**：每 100ms 随机抽取部分 key 检查

### 7.3 Redis 内存淘汰策略

| 策略 | 描述 |
|------|------|
| noeviction | 不淘汰，内存满时写入报错 |
| allkeys-lru | 所有 key 中淘汰最近最少使用的 |
| volatile-lru | 有过期时间的 key 中淘汰 LRU |
| allkeys-random | 随机淘汰 |
| volatile-ttl | 淘汰 TTL 最短的 |
| allkeys-lfu | 所有 key 中淘汰最不经常使用的（Redis 4.0+） |
