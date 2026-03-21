# MySQL

## 索引

### 数据结构

- **B+ 树**（InnoDB 默认）：非叶子节点只存 key，叶子节点存数据，叶子节点通过指针连成链表
- 为什么不用 B 树？B+ 树叶子节点有链表，范围查询更高效
- 为什么不用 Hash？不支持范围查询和排序

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

## SQL 优化

```sql
-- 使用 EXPLAIN 分析执行计划
EXPLAIN SELECT * FROM user WHERE name = 'Tom';
```

关注：`type`（ALL < index < range < ref < const）、`key`、`rows`、`Extra`

### 优化技巧

1. 避免 `SELECT *`
2. 用覆盖索引避免回表
3. 分页优化：`WHERE id > last_id LIMIT 10` 替代 `OFFSET`
4. 批量插入替代单条循环
5. 合理使用联合索引（最左前缀）

## 高频面试题

1. **B+ 树和 B 树的区别？为什么 MySQL 用 B+ 树？**
2. **什么是回表？怎么避免？**
3. **MVCC 原理？RC 和 RR 的 Read View 有什么区别？**
4. **MySQL 怎么解决幻读？**（MVCC + Next-Key Lock）
5. **慢查询怎么优化？**
