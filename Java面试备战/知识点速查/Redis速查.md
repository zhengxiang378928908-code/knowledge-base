# Redis

## 数据结构

| 类型 | 底层实现 | 应用场景 |
|------|----------|----------|
| String | SDS | 缓存、计数器、分布式锁 |
| List | quicklist（ziplist + linkedlist） | 消息队列、时间线 |
| Hash | hashtable / ziplist | 对象存储 |
| Set | hashtable / intset | 标签、交集并集 |
| ZSet | skiplist + hashtable | 排行榜、延迟队列 |
| Bitmap | String | 签到、布隆过滤器 |
| HyperLogLog | 概率算法 | UV 统计 |
| Stream | Radix Tree | 消息队列（类 Kafka） |

## 持久化

### RDB（快照）

- `save`：阻塞主线程
- `bgsave`：fork 子进程，COW 机制
- 优点：恢复快、文件紧凑
- 缺点：可能丢失最后一次快照后的数据

### AOF（追加日志）

- 记录每条写命令
- 刷盘策略：`always` / `everysec`（推荐） / `no`
- AOF 重写：压缩日志文件

### 混合持久化（Redis 4.0+）

AOF 重写时前半段用 RDB 格式，后半段用 AOF 增量。

## 缓存三大问题

### 缓存穿透（查不存在的数据）

- 布隆过滤器拦截
- 缓存空值（设短过期时间）

### 缓存击穿（热点 key 过期）

- 互斥锁（setnx）重建缓存
- 逻辑过期（不设 TTL，异步更新）

### 缓存雪崩（大量 key 同时过期）

- 过期时间加随机偏移
- 多级缓存（本地缓存 + Redis）
- 熔断降级

## 分布式锁

```java
// 加锁
SET lock_key unique_value NX PX 30000

// 释放（Lua 脚本保证原子性）
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
end
```

### Redisson

- 看门狗自动续期
- 可重入锁
- 公平锁
- RedLock（多节点）

## 过期淘汰策略

### 过期策略

- **惰性删除**：访问时检查是否过期
- **定期删除**：定时随机检查

### 淘汰策略（内存满时）

- `noeviction`：不淘汰，写入报错
- `allkeys-lru`：全局 LRU（推荐）
- `volatile-lru`：设了过期时间的 key LRU
- `allkeys-lfu`：全局 LFU
- `allkeys-random`：随机淘汰

## 集群模式

| 模式 | 特点 |
|------|------|
| 主从复制 | 读写分离，异步复制 |
| 哨兵（Sentinel） | 自动故障转移 |
| Cluster | 分片（16384 个 slot），去中心化 |

## 高频面试题

1. **Redis 为什么快？**（纯内存、单线程无锁竞争、IO 多路复用、高效数据结构）
2. **Redis 6.0 为什么引入多线程？**（网络 IO 多线程，命令执行仍单线程）
3. **缓存和数据库一致性怎么保证？**（Cache Aside、先更新 DB 再删缓存 + 延迟双删）
4. **Redis 事务和 Lua 脚本区别？**
5. **大 key 和热 key 怎么处理？**
