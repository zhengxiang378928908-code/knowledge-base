# RocketMQ 深入

> 你的变电 3.0 项目中有 29 个消费者、34 个 Topic，面试官一定会深问。

---

## 一、核心架构

```
Producer → Broker (Master/Slave) → Consumer
               ↑
           NameServer（路由注册中心）
```

- **NameServer**：无状态，存储 Broker 路由信息，Broker 每 30 秒心跳
- **Broker**：消息存储和转发，Master 写入，Slave 同步
- **Producer**：消息发送者，从 NameServer 获取路由
- **Consumer**：消息消费者，支持 Push 和 Pull 模式

### 核心存储

- **CommitLog**：所有消息顺序写入，保证写入性能（顺序 IO）
- **ConsumeQueue**：消费队列索引（Topic + QueueId），存储 CommitLog offset
- **IndexFile**：消息索引，支持按 key / 时间查询

---

## 二、消息类型

### 2.1 普通消息

```java
// 同步发送（等待 Broker 确认）
SendResult result = producer.send(message);

// 异步发送（回调通知）
producer.send(message, new SendCallback() {
    public void onSuccess(SendResult result) { }
    public void onException(Throwable e) { }
});

// 单向发送（不关心结果，如日志）
producer.sendOneway(message);
```

### 2.2 顺序消息

- 全局顺序：Topic 只有一个 Queue → 吞吐低
- 分区顺序：同一业务 ID 的消息发到同一个 Queue

```java
// Producer 端：同一 orderId 路由到同一 Queue
producer.send(message, (mqs, msg, arg) -> {
    int index = arg.hashCode() % mqs.size();
    return mqs.get(Math.abs(index));
}, orderId);

// Consumer 端：使用 MessageListenerOrderly（单线程消费每个 Queue）
```

### 2.3 延迟消息

```java
message.setDelayTimeLevel(3);  // level 3 = 10s
// 延迟级别：1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

### 2.4 事务消息

```
1. Producer 发送 Half Message（对 Consumer 不可见）
2. Broker 返回确认
3. Producer 执行本地事务
4. 根据本地事务结果：
   - Commit → 消息对 Consumer 可见
   - Rollback → 消息丢弃
5. 如果没有回复（网络问题），Broker 定期回查 Producer
```

---

## 三、消息可靠性（防丢失）

### 3.1 三个环节

| 环节 | 丢失原因 | 解决方案 |
|------|---------|---------|
| Producer → Broker | 网络异常 | 同步发送 + 重试（默认 3 次） |
| Broker 存储 | 宕机 | 同步刷盘 + 主从同步 |
| Broker → Consumer | 消费失败 | 手动 ACK，消费成功才返回 SUCCESS |

### 3.2 你项目中的做法

**Producer 端：**
- 使用同步发送，失败重试
- ACL 认证保证安全

**Consumer 端：**
```java
@RocketMQMessageListener(
    topic = RocketMqConfig.TOPIC_JIKONG_FAULT_MESSAGE,
    consumerGroup = "GID_JIKONG_FAULT_MESSAGE"
)
public class FaultTripRecordConsumer implements RocketMQListener<MessageExt> {
    @Override
    public void onMessage(MessageExt messageExt) {
        // 1. Redis 消息去重
        if (!messageDedupUtil.tryAcquireByMessage(topic, message, 3L * 60)) {
            return;
        }
        try {
            // 2. 业务处理
            boolean result = service.processRocketMqMessage(message);
            if (result) {
                // 3. 标记成功（10分钟窗口）
                messageDedupUtil.markMessageSuccess(topic, message, 10L * 60);
            }
        } catch (Exception e) {
            // 4. 重试 3 次后放弃
            if (messageExt.getReconsumeTimes() >= 3) {
                log.error("Max retries reached");
                return;
            }
            throw e;  // 触发 RocketMQ 重试
        }
    }
}
```

---

## 四、重复消费问题

### 4.1 为什么会重复

- Consumer 处理成功但 ACK 失败（网络抖动）
- Consumer 重平衡导致 offset 回退
- Producer 重试导致消息重复发送

### 4.2 解决方案：幂等

**方案 1：数据库唯一约束**
- 消息中包含业务唯一 ID
- 数据库层面做 unique 约束，重复插入会报错

**方案 2：Redis 去重（你项目的方案）**
```java
// MessageDedupUtil
// 消费前：SET message_uuid NX EX 180（3 分钟窗口）
// 如果 SET 成功 → 是新消息，继续处理
// 如果 SET 失败 → 重复消息，跳过

// 成功后：SET message_uuid "done" EX 600（10 分钟窗口）
// 扩大窗口，防止短暂重复
```

**方案 3：状态机**
- 业务有状态流转（如 待处理 → 处理中 → 已完成）
- 只有正确的状态才能流转，重复消息不会影响已完成的状态

---

## 五、消息积压

### 5.1 原因

- 消费者处理速度 < 生产速度
- 消费者异常/宕机
- 下游服务响应慢

### 5.2 解决方案

**短期应急：**
1. 增加消费者实例（注意 Queue 数量限制）
2. 消费者做批量消费
3. 跳过非关键消息

**长期优化：**
1. 消费者内部多线程处理
2. 增加 Queue 数量（需要在 Broker 端配置）
3. 优化消费逻辑（异步 IO、批量写库）

---

## 六、高频面试题

### Q1：RocketMQ 和 Kafka 的区别？

| 特性 | RocketMQ | Kafka |
|------|----------|-------|
| 语言 | Java | Scala + Java |
| 延迟消息 | 原生支持 | 不支持 |
| 事务消息 | 原生支持 | 支持但复杂 |
| 消息查询 | 支持按 key/时间查询 | 不支持 |
| 消息回溯 | 支持按时间点回溯 | 支持按 offset |
| 吞吐量 | 十万级 | 百万级 |
| 适用场景 | 业务消息、事务消息 | 大数据、日志流 |

**你项目选 RocketMQ 的原因**：需要延迟消息、消息查询、事务消息等特性，业务可靠性要求高于吞吐量。

### Q2：RocketMQ 的消费模式？

- **集群消费（Clustering）**：同组内负载均衡，每条消息只被一个消费者消费（默认）
- **广播消费（Broadcasting）**：同组内每个消费者都消费全量消息

### Q3：消费者拉模式还是推模式？

RocketMQ 的 Push 模式底层还是 Pull：
- 长轮询：Consumer 发送 Pull 请求，Broker 没有新消息时 hold 住连接（默认 15 秒）
- 有新消息立即返回
- 兼顾实时性和服务端压力

### Q4：Broker 主从同步机制？

- **同步复制（SYNC_MASTER）**：Master 写入后等 Slave 同步完成才返回成功（强一致）
- **异步复制（ASYNC_MASTER）**：Master 写入后立即返回（可能丢数据，性能好）

你的项目配置：
```yaml
rocketmq:
  name-server: 192.168.0.230:9876
  producer:
    group: PMS_PRODUCER_GROUP
```

---

## 七、你项目中的 Topic 设计

```java
// 部分 Topic 常量（共 34 个）
TOPIC_TRANSFORMER_OVERLOAD     // 主变过载告警
TOPIC_SWITCHGEAR_HIGH_LOAD     // 开关柜高负载告警
TOPIC_CIRCUIT_BREAKER_POSITION_INFO  // 断路器位置变化
TOPIC_FAULT_TRIP_RECORD        // 故障跳闸记录（关键）
TOPIC_JIKONG_FAULT_MESSAGE     // 集控故障消息
TOPIC_VOLTAGE_ABNORMAL_ALARM   // 电压异常告警
```

**设计原则：**
- 按业务场景划分 Topic（不是按数据类型）
- 关键告警和普通统计分开 Topic（隔离影响）
- 每个 Topic 对应独立的 ConsumerGroup
