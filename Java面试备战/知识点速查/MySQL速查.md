# MySQL

## 索引

### 数据结构

- **B+ 树**（InnoDB 默认）：非叶子节点只存 key，叶子节点存数据，叶子节点通过指针连成双向链表
- 为什么不用 B 树？B 树每个节点都存数据，扇出低、树更高、IO 次数多；B+ 树非叶子节点只存 key，单个节点能放更多索引项，树更矮；叶子节点有链表，范围查询直接遍历不用回溯
- 为什么不用 Hash？不支持范围查询和排序，只能等值查询

### 聚簇索引 vs 非聚簇索引

- **聚簇索引**：叶子节点存完整行数据（主键索引）
- **非聚簇索引**：叶子节点存主键值 → 需要**回表**查询完整数据

### 覆盖索引

查询的字段全部在索引中，无需回表。

```sql
-- idx_name_age(name, age)
SELECT name, age FROM user WHERE name = 'Tom';  -- 覆盖索引，不回表
SELECT *     FROM user WHERE name = 'Tom';       -- 需要回表
```

### 索引失效场景

1. `LIKE '%xxx'` 左模糊
2. 对索引列使用函数或计算
3. 隐式类型转换（varchar 列用数字查）
4. `OR` 条件中有非索引列
5. 违反最左前缀原则
6. `!=` / `NOT IN` / `IS NULL`（视优化器判断）

## 事务 ACID

| 特性 | 说明 | 实现机制 |
|------|------|----------|
| 原子性 | 全部成功或全部回滚 | undo log |
| 一致性 | 事务前后数据一致 | 其他三个特性保障 |
| 隔离性 | 事务间互不干扰 | MVCC + 锁 |
| 持久性 | 提交后数据不丢失 | redo log |

## 隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|------|------------|------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ |
| READ COMMITTED | ❌ | ✅ | ✅ |
| REPEATABLE READ（默认） | ❌ | ❌ | 快照读基本无幻读；当前读依赖 Next-Key Lock |
| SERIALIZABLE | ❌ | ❌ | ❌ |

## MVCC

- 每行数据有隐藏列：`trx_id`（创建事务ID）、`roll_pointer`（undo log 指针）
- **Read View**：快照读时生成 / 使用事务视图
  - RC：每次 SELECT 创建新 Read View
  - RR：第一次 SELECT 创建，后续复用

## 锁

### 按粒度

- **表锁**：开销小，并发低（MyISAM）
- **行锁**：开销大，并发高（InnoDB）
- **间隙锁（Gap Lock）**：锁定索引间隙，防止幻读

### 按类型

- **共享锁（S）**：`SELECT ... FOR SHARE`
- **排他锁（X）**：`SELECT ... FOR UPDATE`
- **意向锁**：表级，表明事务想要在行上加什么锁

### 死锁

产生条件：互斥、占有且等待、不可抢占、循环等待

解决：
- `innodb_deadlock_detect = ON`（自动检测，回滚小事务）
- 控制加锁顺序
- 缩短事务

## 三大日志

| 日志 | 层级 | 作用 | 写入时机 |
|------|------|------|----------|
| **redo log** | InnoDB 引擎层 | 保证持久性（崩溃恢复） | 事务执行过程中写入，先写 redo log buffer → 刷盘 |
| **undo log** | InnoDB 引擎层 | 保证原子性（回滚） + MVCC 多版本 | 修改数据前记录旧值 |
| **binlog** | Server 层 | 主从复制 + 数据恢复 | 事务提交时写入 |

**redo log vs binlog**

| 对比项 | redo log | binlog |
|--------|----------|--------|
| 层级 | InnoDB 引擎层 | MySQL Server 层 |
| 内容 | 物理日志（某页某偏移改了什么） | 逻辑日志（SQL 语句或行变更） |
| 大小 | 固定大小，循环写 | 追加写，不覆盖 |
| 用途 | 崩溃恢复 | 主从复制、数据恢复 |

**两阶段提交**：先写 redo log（prepare）→ 写 binlog → redo log（commit），保证两份日志一致。

**面试话术**：redo log 保证崩溃恢复不丢数据，undo log 保证回滚和 MVCC 多版本读，binlog 用于主从复制和数据恢复。事务提交用两阶段提交保证 redo log 和 binlog 的一致性。

## MySQL 怎么解决幻读

### 快照读（普通 SELECT）

- 通过 MVCC 解决，RR 级别下第一次 SELECT 创建 Read View，后续复用同一个快照
- 读不到其他事务新插入的行 → 不会幻读

### 当前读（SELECT ... FOR UPDATE / INSERT / UPDATE / DELETE）

- 通过 **Next-Key Lock**（行锁 + 间隙锁）解决

```
假设表中有 id = 5, 10, 15
SELECT * FROM t WHERE id > 5 AND id < 15 FOR UPDATE;

加锁范围：(5, 10]、(10, 15) → 其他事务无法在这个范围插入新行
```

- **Record Lock**：锁住具体行
- **Gap Lock**：锁住索引间隙（开区间）
- **Next-Key Lock** = Record Lock + Gap Lock（左开右闭）

**面试话术**：快照读靠 MVCC，当前读靠 Next-Key Lock。Next-Key Lock 是行锁加间隙锁，锁住索引记录和前面的间隙，阻止其他事务在这个范围内插入新行，从而防止幻读。

## SQL 优化

### EXPLAIN 详解

```sql
EXPLAIN SELECT * FROM user WHERE name = 'Tom';
```

**type 字段（访问类型，从好到差）**

| type | 含义 | 说明 |
|------|------|------|
| const | 主键或唯一索引等值 | 最快 |
| eq_ref | 关联查询用主键/唯一索引 | JOIN 时最优 |
| ref | 非唯一索引等值查询 | 常见的好情况 |
| range | 索引范围扫描 | BETWEEN / > / < / IN |
| index | 全索引扫描 | 比 ALL 好但仍需优化 |
| ALL | 全表扫描 | 必须优化 |

**Extra 字段（重点关注）**

| Extra | 含义 |
|-------|------|
| Using index | 覆盖索引，不回表（好） |
| Using where | 存储引擎返回后 Server 层再过滤 |
| Using temporary | 用了临时表（考虑优化） |
| Using filesort | 额外排序，没用上索引排序（考虑优化） |
| Using index condition | 索引下推 ICP（好，减少回表） |

### 慢查询排查完整流程

**第一步：开启慢查询日志**

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
SHOW VARIABLES LIKE 'slow_query_log_file';  -- 查看日志位置
```

**第二步：定位慢 SQL**
- `mysqldumpslow -s t -t 10 slow.log` — 按耗时排序取 top 10

**第三步：EXPLAIN 分析**
- 看 type 是否全表扫描
- 看 key 是否命中索引
- 看 Extra 是否有 filesort / temporary

**第四步：针对性优化**
1. 加索引 / 调整联合索引顺序
2. 避免 `SELECT *`，用覆盖索引
3. 分页优化：`WHERE id > last_id LIMIT 10` 替代大 `OFFSET`
4. 批量插入替代单条循环
5. 合理使用联合索引（最左前缀）
6. 避免索引失效（函数、隐式转换、左模糊）

**面试话术**：先开慢查询日志找到慢 SQL，用 EXPLAIN 看执行计划，重点关注 type 是不是全表扫描、key 有没有命中索引、Extra 有没有 filesort。然后针对性地加索引、改写 SQL、用覆盖索引减少回表。

## 高频面试题

1. **B+ 树和 B 树的区别？为什么 MySQL 用 B+ 树？**（B+ 树非叶子节点不存数据扇出高树更矮，叶子节点有链表支持范围查询）
2. **什么是回表？怎么避免？**（非聚簇索引查到主键后再查聚簇索引；用覆盖索引避免）
3. **MVCC 原理？RC 和 RR 的 Read View 有什么区别？**（RC 每次 SELECT 新建，RR 第一次创建后复用）
4. **MySQL 怎么解决幻读？**（快照读靠 MVCC，当前读靠 Next-Key Lock 锁住间隙）
5. **慢查询怎么优化？**（开 slow log → EXPLAIN 分析 → 加索引 / 改写 SQL / 覆盖索引）
6. **redo log / undo log / binlog 区别？**（redo 崩溃恢复、undo 回滚+MVCC、binlog 复制恢复，两阶段提交保证一致）
7. **EXPLAIN 重点看什么？**（type 不能是 ALL，Extra 避免 filesort/temporary，key 要命中索引）
