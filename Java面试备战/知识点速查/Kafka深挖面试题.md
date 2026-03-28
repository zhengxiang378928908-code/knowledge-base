# Kafka 深挖面试题 — 口语版

> 适用场景：面试岗位技术栈包含 Kafka，面试官可能围绕 Kafka 深入追问。
> 答题风格：用自然语言讲机制，不念源码。

---

## 一、Kafka 为什么吞吐量这么高？

> Kafka 高吞吐主要靠四个设计：
>
> **第一，顺序写磁盘。** 消息追加写入 Partition 对应的日志文件，不做随机写，磁盘顺序写的速度接近内存。
>
> **第二，Page Cache。** Kafka 不自己管内存缓存，直接利用操作系统的页缓存。写消息时先写到页缓存，操作系统异步刷盘；读消息时如果页缓存命中，直接返回，不走磁盘。
>
> **第三，零拷贝（Zero Copy）。** 消费者拉消息时，Kafka 用 sendfile 系统调用，数据从磁盘到网卡不经过用户态，减少了两次内存拷贝。
>
> **第四，批量和压缩。** Producer 端会把多条消息攒成一个 batch 再发送，还可以用 snappy/lz4 压缩，减少网络 IO。

---

## 二、Kafka 怎么保证消息不丢？

> 从三个环节讲：
>
> **Producer 端：** 设置 `acks=all`，等 ISR 里所有副本都确认了才算发送成功。再加上 `retries` 重试配置，网络抖动时自动重发。
>
> **Broker 端：** 每个 Partition 配多个副本，`replication.factor` 至少 3。设置 `min.insync.replicas=2`，这样即使一个副本挂了，只要还有 2 个在 ISR 里，消息就不会丢。
>
> **Consumer 端：** 关闭自动提交 offset（`enable.auto.commit=false`），业务处理成功后手动提交。这样消费者挂了重启，会从上次提交的 offset 重新消费，最多重复不会丢。

---

## 三、ISR 机制详解

> 每个 Partition 有一个 Leader 和若干 Follower。ISR 就是和 Leader 保持同步的副本集合。
>
> **同步标准：** Follower 持续从 Leader 拉取数据，如果落后太多（由 `replica.lag.time.max.ms` 控制，默认 30 秒内没追上），就会被踢出 ISR。追上了之后可以重新加入。
>
> **和 acks 的关系：**
> - `acks=1`：Leader 写成功就返回，Leader 挂了可能丢
> - `acks=all`：ISR 里所有副本都确认才返回，配合 `min.insync.replicas` 使用最可靠
>
> **Leader 选举：** Leader 挂了，从 ISR 里选一个 Follower 当新 Leader。如果 ISR 为空，要看 `unclean.leader.election.enable` 配置，开了的话可以从非 ISR 副本选，但可能丢数据。

---

## 四、Consumer Rebalance 详解

> **什么时候触发：**
> - 消费者加入或退出 Consumer Group
> - 消费者心跳超时被判定为掉线
> - Topic 的 Partition 数量变了
> - 消费者主动调用 unsubscribe
>
> **Rebalance 的问题：**
> - 期间整个 Consumer Group 暂停消费
> - 如果 Rebalance 频繁，会导致消费延迟严重
>
> **怎么减少 Rebalance：**
> - `session.timeout.ms` 适当调大（比如 25 秒），避免偶尔 GC 停顿被误判
> - `heartbeat.interval.ms` 设为 session timeout 的 1/3（比如 8 秒）
> - `max.poll.interval.ms` 根据业务处理时间调大，避免消费太慢被踢
> - 消费逻辑里不要做太重的事，重操作异步处理

---

## 五、Kafka 消息顺序性怎么保证？

> **Partition 内有序，跨 Partition 无序。**
>
> 如果业务要求同一个订单的消息有序，就把订单 ID 作为消息 key，Kafka 会按 key hash 到同一个 Partition，消费端单线程顺序处理就能保证顺序。
>
> **注意：** 如果消费端用多线程处理同一个 Partition 的消息，顺序就没了。要保证顺序就只能单线程消费，或者按业务 key 分发到不同的有序队列再并行。

---

## 六、Kafka 和 RocketMQ 最核心的区别？

> **一句话版本：** Kafka 是为大数据高吞吐设计的，RocketMQ 是为业务消息可靠性设计的。
>
> **展开版本：**
>
> | 维度 | Kafka | RocketMQ |
> |------|-------|----------|
> | 吞吐量 | 百万级 | 十万级 |
> | 延迟消息 | 不支持 | 原生支持 |
> | 事务消息 | 偏 exactly-once 语义 | 半消息 + 回查，真正的业务事务 |
> | 消息查询 | 不支持按 key 查 | 支持按 key/时间查 |
> | 消费模型 | 一个 Partition 只能被组内一个消费者消费 | 更灵活 |
> | 运维依赖 | ZooKeeper / KRaft | NameServer，更轻量 |
> | 典型场景 | 日志采集、埋点、流计算 | 订单、支付、告警、工单 |

---

## 七、Kafka 消息积压怎么处理？

> **先判断原因：**
> - 消费者挂了 → 先恢复消费者
> - 消费太慢 → 优化消费逻辑
> - 流量突增 → 扩容
>
> **短期应急：**
> 1. 增加消费者实例，但不能超过 Partition 数（超了也没用，多余的会空闲）
> 2. 如果消费者数已经等于 Partition 数还不够，先增加 Partition
> 3. 非核心消息可以临时跳过或转存到另一个 Topic 后续慢慢消费
>
> **长期优化：**
> 1. 消费端批量处理（攒一批再写库）
> 2. 消费端多线程处理（注意顺序性要求）
> 3. 提前规划好 Partition 数，一般 Partition 数 = 期望吞吐 / 单消费者处理能力

---

## 八、Kafka 的 Exactly-Once 语义怎么实现？

> Kafka 从 0.11 版本开始支持 Exactly-Once，靠两个机制：
>
> **幂等 Producer：**
> - 设置 `enable.idempotence=true`
> - Broker 会给每个 Producer 分配一个 PID，每条消息带序列号
> - Broker 收到重复序列号的消息直接丢弃
> - 这只能保证**单 Partition 单会话**内不重复
>
> **事务：**
> - 设置 `transactional.id`，Producer 可以把多个 Partition 的写入和 Consumer offset 提交放在一个事务里
> - 要么全部成功，要么全部回滚
> - Consumer 端设置 `isolation.level=read_committed`，只读已提交的消息
>
> **面试话术：** 幂等解决的是单分区内 Producer 重试导致的重复；事务解决的是跨分区的原子写入。两个一起用才是完整的 Exactly-Once。

---

## 九、Kafka 高频追问清单

1. **为什么 Kafka 快？**（顺序写 + Page Cache + 零拷贝 + 批量压缩）
2. **acks=0/1/all 分别什么含义？**（不等确认 / Leader 确认 / ISR 全确认）
3. **ISR 是什么？副本落后了怎么办？**（同步副本集合，落后踢出，追上重新加入）
4. **Leader 挂了怎么办？**（从 ISR 选新 Leader）
5. **Rebalance 是什么？怎么减少？**（消费者变动触发，调大超时参数 + 优化消费速度）
6. **怎么保证不丢消息？**（acks=all + 多副本 + 手动提交 offset）
7. **怎么保证顺序？**（同 key 同 Partition + 单线程消费）
8. **消息积压怎么办？**（加消费者 + 加 Partition + 优化消费逻辑）
9. **Exactly-Once 怎么实现？**（幂等 Producer + 事务）
10. **Kafka 和 RocketMQ 怎么选？**（大数据用 Kafka，业务消息用 RocketMQ）
