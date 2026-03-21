# 数据库与 SQL 优化

> 你简历提到了 SQL 性能优化、慢查询处理，面试必问。

---

## 一、MySQL 索引

### 1.1 索引数据结构：B+ 树

```
为什么用 B+ 树而不是 B 树 / 红黑树 / Hash？

B+ 树优势：
├── 非叶子节点只存索引 → 每个节点能存更多 key → 树更矮 → IO 更少
├── 叶子节点双向链表 → 范围查询高效（WHERE age > 18）
├── 所有数据都在叶子节点 → 查询性能稳定（每次都走到叶子）
└── 一般 3-4 层即可存储千万级数据

Hash 索引：O(1) 查找，但不支持范围查询、排序、最左前缀
红黑树：层数太深，IO 次数多
B 树：非叶子也存数据，每个节点能放的 key 少，树更高
```

### 1.2 索引类型

| 类型 | 说明 |
|------|------|
| 主键索引 | 聚簇索引，叶子节点存完整行数据 |
| 唯一索引 | 值唯一，允许 NULL |
| 普通索引 | 最基本的索引 |
| 复合索引 | 多列组合，遵循最左前缀原则 |
| 覆盖索引 | 查询的列都在索引中，无需回表 |

### 1.3 聚簇索引 vs 非聚簇索引

- **聚簇索引（主键索引）**：叶子节点存完整行数据，一张表只有一个
- **非聚簇索引（二级索引）**：叶子节点存主键值
- **回表**：通过二级索引找到主键值 → 再通过主键索引找到完整行
- **覆盖索引**：查询的列都在二级索引中，无需回表 → 性能好

### 1.4 最左前缀原则

复合索引 `(a, b, c)`：

| 查询条件 | 是否走索引 |
|---------|-----------|
| WHERE a = 1 | 走（a） |
| WHERE a = 1 AND b = 2 | 走（a, b） |
| WHERE a = 1 AND b = 2 AND c = 3 | 走（a, b, c） |
| WHERE b = 2 | 不走 |
| WHERE a = 1 AND c = 3 | 走（a），c 不走 |
| WHERE a > 1 AND b = 2 | a 走（范围），b 不走 |
| WHERE a = 1 ORDER BY b | 走（a, b） |

### 1.5 索引失效场景

1. **对索引列使用函数**：`WHERE YEAR(create_time) = 2024` ✗
2. **隐式类型转换**：`WHERE varchar_col = 123`（字符串列用数字查询）
3. **LIKE 左模糊**：`WHERE name LIKE '%张'` ✗ （`'张%'` ✓）
4. **OR 条件只有一侧有索引** ✗
5. **NOT IN / NOT EXISTS**（部分情况）
6. **IS NULL / IS NOT NULL**（部分情况）
7. **数据量太小**，优化器认为全表扫描更快

---

## 二、EXPLAIN 执行计划

### 2.1 关键字段

```sql
EXPLAIN SELECT * FROM user WHERE age > 18;
```

| 字段 | 重点关注 |
|------|---------|
| **type** | 访问类型（最重要） |
| **key** | 实际使用的索引 |
| **rows** | 预估扫描行数 |
| **Extra** | 额外信息 |

### 2.2 type 从好到差

```
system > const > eq_ref > ref > range > index > ALL

const：主键/唯一索引等值查询（最多一行）
eq_ref：JOIN 时被驱动表用主键/唯一索引
ref：非唯一索引等值查询
range：索引范围查询（BETWEEN、>、<、IN）
index：全索引扫描（遍历索引树）
ALL：全表扫描（最差，必须优化）
```

### 2.3 Extra 常见值

| Extra | 含义 |
|-------|------|
| Using index | 覆盖索引，无需回表（好） |
| Using where | 需要在 Server 层过滤 |
| Using temporary | 用了临时表（需优化） |
| Using filesort | 额外排序（需优化） |
| Using index condition | 索引下推（ICP，5.6+） |

---

## 三、SQL 优化实战

### 3.1 慢查询定位

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录

-- 查看慢查询日志
SHOW VARIABLES LIKE 'slow_query_log_file';
```

### 3.2 优化套路

**1. 避免 SELECT ***
```sql
-- 差
SELECT * FROM device WHERE type = 'breaker';
-- 好（只查需要的列，可能走覆盖索引）
SELECT id, name, status FROM device WHERE type = 'breaker';
```

**2. 小表驱动大表**
```sql
-- IN 适合子查询结果集小
SELECT * FROM order WHERE user_id IN (SELECT id FROM user WHERE status = 1);

-- EXISTS 适合子查询结果集大
SELECT * FROM user u WHERE EXISTS (SELECT 1 FROM order o WHERE o.user_id = u.id);
```

**3. 分批处理（你的跑批优化经验）**
```sql
-- 差：一次查全量
SELECT * FROM transaction WHERE status = 'pending';

-- 好：分批查询
SELECT * FROM transaction WHERE status = 'pending' AND id > #{lastId} LIMIT 1000;
```

**4. 避免 IN 超大列表（你项目规范）**
```sql
-- 差：IN 超过 1000 个值
WHERE device_id IN (1, 2, 3, ..., 5000)

-- 好：拆分批次，每批 <= 1000
-- 或者用临时表 JOIN
```

**5. 合理使用索引**
```sql
-- 差：函数导致索引失效
WHERE DATE(create_time) = '2024-01-01'
-- 好：改写为范围查询
WHERE create_time >= '2024-01-01' AND create_time < '2024-01-02'
```

---

## 四、MySQL 事务

### 4.1 ACID

| 特性 | 说明 | 实现机制 |
|------|------|---------|
| A 原子性 | 事务要么全成功，要么全回滚 | undo log |
| C 一致性 | 事务前后数据状态一致 | 由其他三个特性和业务约束共同保证 |
| I 隔离性 | 并发事务互不干扰 | MVCC + 锁 |
| D 持久性 | 事务提交后数据永久保存 | redo log |

### 4.2 隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ UNCOMMITTED | ✓ | ✓ | ✓ |
| READ COMMITTED | ✗ | ✓ | ✓ |
| **REPEATABLE READ（MySQL默认）** | ✗ | ✗ | 快照读基本无幻读；当前读依赖 Next-Key Lock |
| SERIALIZABLE | ✗ | ✗ | ✗ |

### 4.3 MVCC（多版本并发控制）

- 每行数据有隐藏列：`DB_TRX_ID`（最近修改事务ID）、`DB_ROLL_PTR`（回滚指针，指向 undo log）
- **Read View**：事务快照，记录当前活跃事务 ID 列表
- **可见性判断**：
  - 数据版本的 TRX_ID < 最小活跃事务 ID → 可见
  - 数据版本的 TRX_ID > 最大事务 ID → 不可见
  - 在活跃列表中 → 不可见
  - 不在活跃列表中 → 可见

- **RC**：每次 SELECT 创建新的 Read View
- **RR**：事务第一次 SELECT 时创建 Read View，之后复用
- 讨论幻读时要区分：
  - **快照读**：RR 下通常不会看到幻读
  - **当前读**（`FOR UPDATE` / `FOR SHARE`）：依赖 Next-Key Lock 防止幻读

### 4.4 锁机制

**按粒度：**
- 表锁：锁整张表，开销小，并发低
- 行锁：锁单行，开销大，并发高（InnoDB 支持）

**按类型：**
- 共享锁（S 锁）：`SELECT ... FOR SHARE`，可以并发读（`LOCK IN SHARE MODE` 属于旧写法）
- 排他锁（X 锁）：`SELECT ... FOR UPDATE`，独占
- 意向锁：表级锁，表示事务想要加行锁（快速判断表上是否有行锁）

**InnoDB 行锁类型：**
- Record Lock：锁单行记录
- Gap Lock：锁间隙，防止幻读（RR 级别）
- Next-Key Lock：Record + Gap，左开右闭区间

**你项目中遇到的锁争用：**
> 多微服务节点同时执行跑批，对同一批数据加锁导致死锁或超时。
> 解决：用 Redis 分布式锁控制跑批入口，确保同一时间只有一个节点执行。

---

## 五、分库分表（了解即可）

### 5.1 什么时候需要分库分表

- 单表数据量超过 2000 万行（经验值）
- 查询性能已经无法通过索引优化
- 单库写入 QPS 超过瓶颈

### 5.2 分片策略

- **水平分表**：按某个字段（如 user_id）取模分到不同表
- **水平分库**：按某个字段分到不同数据库
- **垂直分表**：将大表拆分为多个小表（如将 TEXT 字段分离）

### 5.3 分库分表带来的问题

- 分布式事务
- 跨库 JOIN
- 全局唯一 ID（雪花算法）
- 分页查询复杂

---

## 六、PostgreSQL 相关（你的项目用了 PG）

### 6.1 PostgreSQL vs MySQL

| 特性 | PostgreSQL | MySQL |
|------|-----------|-------|
| 标准兼容性 | 更好 | 较好 |
| JSON 支持 | 原生 JSONB（可建索引） | 原生 JSON（不能直接索引，但可借助生成列 / 函数索引） |
| 扩展性 | 支持自定义类型和扩展（如 pgvector） | 有限 |
| 全文检索 | 内置 | 需要 Elasticsearch |
| MVCC 实现 | 多版本直接在表中 | undo log |
| 适用场景 | 复杂查询、地理信息、向量检索 | Web 应用、高并发读 |

### 6.2 你项目中选 PG 的原因

- 变电 3.0：部分模块用 PG（和电网既有系统对接）
- RAG 项目：需要 pgvector 扩展做向量检索，PG 是唯一选择

### 6.3 pgvector 基本用法

```sql
-- 创建向量列
CREATE TABLE kb4_corpus (
    id VARCHAR PRIMARY KEY,
    embedding_vec vector(1024)  -- 1024 维向量
);

-- 创建索引（IVFFlat，近似最近邻）
CREATE INDEX idx_embedding ON kb4_corpus
USING ivfflat (embedding_vec vector_cosine_ops);

-- 向量相似度搜索（这里的 <=> 是 cosine distance）
SELECT *, embedding_vec <=> CAST(:query AS vector) AS distance
FROM kb4_corpus
ORDER BY embedding_vec <=> CAST(:query AS vector)
LIMIT 10;
```
