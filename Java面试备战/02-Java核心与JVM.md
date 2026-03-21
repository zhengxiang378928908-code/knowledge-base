# Java 核心与 JVM

> 8 年经验必考中高级题，结合你简历中 OOM 排查、跑批优化的经历来答。

---

## 一、JVM 内存模型

### 1.1 JVM 运行时数据区

```
┌─────────────────────────────────────────────┐
│                  JVM 内存                    │
├──────────┬──────────┬───────────────────────┤
│  线程私有  │  线程共享  │                       │
├──────────┼──────────┤                       │
│ 虚拟机栈   │  堆 Heap  │  ← 对象实例、数组       │
│ 本地方法栈  │  方法区    │  ← 类信息、常量池、静态变量│
│ 程序计数器  │          │                       │
└──────────┴──────────┴───────────────────────┘
```

- **堆（Heap）**：对象实例分配区域，GC 主战场
  - 新生代（Eden + S0 + S1）：新对象分配，Minor GC
  - 老年代：长期存活对象，Major GC / Full GC
- **虚拟机栈**：每个方法调用创建一个栈帧（局部变量表、操作数栈、动态链接、返回地址）
- **方法区（元空间 Metaspace）**：JDK8+ 用本地内存替代永久代，存放类元数据
- **程序计数器**：线程执行位置指示器，唯一不会 OOM 的区域

### 1.2 对象创建过程

1. 类加载检查 → 2. 分配内存（指针碰撞 / 空闲列表）→ 3. 初始化零值 → 4. 设置对象头（Mark Word + 类型指针）→ 5. 执行 `<init>` 构造方法

### 1.3 对象内存布局

- **对象头**：Mark Word（锁状态、HashCode、GC 年龄）+ 类型指针
- **实例数据**：字段值
- **对齐填充**：保证 8 字节整数倍

---

## 二、GC 垃圾回收

### 2.1 如何判断对象是否存活？

- **引用计数法**：简单但无法解决循环引用（Java 不用）
- **可达性分析**：从 GC Roots 出发，不可达的对象就是垃圾
- **GC Roots 包括**：虚拟机栈中引用的对象、静态变量引用、常量引用、JNI 引用、活跃线程

### 2.2 垃圾回收算法

| 算法 | 原理 | 优缺点 | 使用场景 |
|------|------|--------|---------|
| 标记-清除 | 标记存活对象，清除未标记 | 有碎片 | CMS 老年代 |
| 标记-复制 | 存活对象复制到另一半 | 无碎片，浪费空间 | 新生代（Eden→Survivor） |
| 标记-整理 | 存活对象向一端移动 | 无碎片，移动成本 | 老年代 |
| 分代收集 | 新生代用复制，老年代用标记整理 | 综合最优 | 主流 JVM |

### 2.3 常用垃圾收集器

| 收集器 | 代 | 算法 | 特点 |
|--------|---|------|------|
| Serial | 新生代 | 复制 | 单线程，STW，适合客户端 |
| ParNew | 新生代 | 复制 | Serial 多线程版，配合 CMS |
| Parallel Scavenge | 新生代 | 复制 | 吞吐量优先 |
| CMS | 老年代 | 标记-清除 | 低延迟，4 阶段（初始标记→并发标记→重新标记→并发清除） |
| G1 | 全堆 | 分区+复制+标记整理 | JDK9 默认，可预测停顿时间 |
| ZGC | 全堆 | 着色指针+读屏障 | 亚毫秒级停顿，JDK15+ |

### 2.4 G1 收集器重点

- 将堆划分为多个 Region（1-32MB），每个 Region 可以是 Eden/Survivor/Old/Humongous
- Mixed GC：同时回收新生代和部分老年代 Region
- 可通过 `-XX:MaxGCPauseMillis` 设置期望停顿时间
- 适用于大堆（6GB+）场景

---

## 三、JVM 调优（结合你的项目经验）

### 3.1 OOM 排查思路（你简历上写了处理 OOM）

**面试回答模板：**

> 在邮储银行风控项目中遇到过跑批 OOM，排查过程如下：

1. **确认 OOM 类型**：
   - `java.lang.OutOfMemoryError: Java heap space` → 堆内存不足
   - `java.lang.OutOfMemoryError: Metaspace` → 类加载过多
   - `java.lang.OutOfMemoryError: GC overhead limit exceeded` → GC 回收效率低

2. **获取 heap dump**：
   - 启动参数加 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof`
   - 或 `jmap -dump:format=b,file=heapdump.hprof <pid>`

3. **分析 dump**：
   - 用 MAT（Memory Analyzer Tool）打开
   - 看 Dominator Tree 找最大对象
   - 看 Leak Suspects Report 找泄漏路径

4. **我的案例**：
   - MAT 发现一个 ArrayList 持有几十万条记录（跑批一次性加载全量）
   - 解决：改为分批查询（每批 1000 条），处理完及时释放引用
   - 效果：内存峰值从几个 G 降到几百 MB

### 3.2 常用 JVM 参数

```bash
# 堆大小
-Xms4g -Xmx4g          # 初始/最大堆（建议设相同，避免动态扩缩）

# 新生代
-Xmn2g                  # 新生代大小
-XX:SurvivorRatio=8     # Eden:S0:S1 = 8:1:1

# 元空间
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m

# GC 选择
-XX:+UseG1GC            # 使用 G1
-XX:MaxGCPauseMillis=200 # 期望停顿时间

# 日志
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/path/gc.log

# OOM dump
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/
```

### 3.3 CPU 飙升排查（你简历上写了）

```bash
# 1. 找到 CPU 最高的进程
top

# 2. 找到该进程中 CPU 最高的线程
top -H -p <pid>

# 3. 线程 ID 转十六进制
printf "%x\n" <tid>

# 4. 用 jstack 抓线程堆栈，搜索该线程
jstack <pid> | grep <tid_hex> -A 30

# 5. 分析堆栈，定位代码位置
```

常见原因：死循环、频繁 Full GC、锁争用

---

## 四、Java 并发编程

### 4.1 synchronized 与 ReentrantLock 对比

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现 | JVM 内置（monitorenter/monitorexit） | API 层面（AQS） |
| 锁类型 | 非公平 | 公平/非公平可选 |
| 可中断 | 不可中断 | `lockInterruptibly()` 可中断 |
| 超时获取 | 不支持 | `tryLock(timeout)` |
| 条件变量 | 单一 wait/notify | 多个 Condition |
| 自动释放 | 是（异常自动释放） | 否（必须 finally 中 unlock） |

**你项目中的应用**：定时任务 Starter 中使用 `synchronized` 保护 `refreshScheduledTasks` 方法，因为不需要超时和中断特性，synchronized 更简洁。

### 4.2 volatile 关键字

- **可见性**：修改立即对其他线程可见（禁止缓存优化）
- **有序性**：禁止指令重排序（内存屏障）
- **不保证原子性**：`volatile int count; count++` 不是线程安全的

**你项目中的应用**：`ApplicationShutdownListener` 中用 `AtomicBoolean`（底层 volatile + CAS）实现全局关闭标志。

### 4.3 线程池

**核心参数：**
```java
new ThreadPoolExecutor(
    corePoolSize,      // 核心线程数
    maximumPoolSize,   // 最大线程数
    keepAliveTime,     // 非核心线程空闲存活时间
    unit,              // 时间单位
    workQueue,         // 任务队列
    threadFactory,     // 线程工厂
    handler            // 拒绝策略
);
```

**执行流程：**
1. 线程数 < corePoolSize → 创建新线程
2. 线程数 >= corePoolSize → 任务入队列
3. 队列满 + 线程数 < maximumPoolSize → 创建新线程
4. 队列满 + 线程数 >= maximumPoolSize → 执行拒绝策略

**拒绝策略：**
- AbortPolicy（默认）：抛异常
- CallerRunsPolicy：调用者线程执行
- DiscardPolicy：静默丢弃
- DiscardOldestPolicy：丢弃队列最老的任务

**你项目中的配置：**
```java
// 手动任务执行器 - CachedThreadPool（任务不频繁，数量不确定）
@Bean("manualTaskExecutor")
public ExecutorService manualTaskExecutor() {
    return Executors.newCachedThreadPool();
}

// SMS 通知执行器 - 固定线程池（限制并发量）
@Bean("smsExecutor")
public Executor smsExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(2);
    executor.setMaxPoolSize(4);
    executor.setQueueCapacity(200);
    executor.setThreadNamePrefix("sms-async-");
    return executor;
}
```

**面试追问：为什么不建议用 Executors 创建线程池？**
- `newFixedThreadPool` / `newSingleThreadExecutor`：队列用的是无界 LinkedBlockingQueue，可能导致 OOM
- `newCachedThreadPool`：最大线程数是 Integer.MAX_VALUE，可能创建大量线程
- 建议用 `ThreadPoolExecutor` 手动指定参数

### 4.4 ConcurrentHashMap

**JDK 1.7**：Segment 分段锁，每个 Segment 是一个小的 HashMap + ReentrantLock

**JDK 1.8**：
- 数据结构：Node 数组 + 链表 + 红黑树（同 HashMap）
- 锁粒度：对每个桶的头节点加 synchronized
- CAS + synchronized：put 时如果桶为空用 CAS，不为空用 synchronized
- 扩容：支持多线程并发扩容（transfer）

**你项目中的应用：**
```java
// 定时任务注册表 - 跟踪运行中的任务
public static final Map<Long, Thread> RUNNING_TASKS = new ConcurrentHashMap<>();

// 动态任务调度 - 管理 ScheduledFuture
private final Map<String, ScheduledFuture<?>> scheduledFutures = new ConcurrentHashMap<>();
```

### 4.5 ThreadLocal

- 线程本地变量，每个线程独立副本
- 底层：每个 Thread 持有一个 ThreadLocalMap，key 是 ThreadLocal 弱引用，value 是值
- **内存泄漏风险**：key 被 GC 回收后变成 null，value 还在 → 用完必须 `remove()`

**你项目中的应用：**
```java
// 定时任务上下文
public static final ThreadLocal<ScheduledTaskContext> taskContextHolder = new ThreadLocal<>();

// 手动执行标记
public static final ThreadLocal<Boolean> IS_MANUAL_EXECUTION = ThreadLocal.withInitial(() -> false);

// 使用后必须清理
finally {
    taskContextHolder.remove();
    ManualExecutionContext.IS_MANUAL_EXECUTION.remove();
}
```

---

## 五、集合框架

### 5.1 HashMap 底层原理

**JDK 1.8：**
- 数据结构：数组 + 链表 + 红黑树
- 默认容量 16，负载因子 0.75
- 链表长度 > 8 且数组长度 >= 64 → 转红黑树
- 红黑树节点 < 6 → 退化为链表

**put 流程：**
1. 对 key 做 hash（高 16 位异或低 16 位，减少碰撞）
2. `(n-1) & hash` 计算数组下标
3. 如果为空，直接放入
4. 如果不为空，判断 key 是否相等（equals）
5. 相等则覆盖 value
6. 不相等则追加到链表/红黑树

**扩容：**
- 元素数 > 容量 × 负载因子 → 扩容为原来 2 倍
- JDK 1.8 优化：扩容时元素要么在原位置，要么在原位置 + 旧容量

**为什么线程不安全？**
- JDK 1.7 扩容时链表头插法可能形成环（死循环）
- JDK 1.8 多线程 put 可能丢数据（覆盖）

### 5.2 ArrayList vs LinkedList

| 特性 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层 | Object 数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头部插入 | O(n)（移动元素） | O(1) |
| 尾部插入 | 均摊 O(1) | O(1) |
| 内存占用 | 紧凑（连续内存） | 多（每个节点额外两个指针） |
| 适用场景 | 读多写少 | 频繁头部插入删除 |

实际开发中 ArrayList 用得更多，因为 CPU 缓存友好（连续内存）。

---

## 六、Java 基础高频

### 6.1 String / StringBuilder / StringBuffer

- String：不可变，每次修改创建新对象
- StringBuilder：可变，非线程安全，性能好
- StringBuffer：可变，线程安全（synchronized），性能略差

### 6.2 == 和 equals

- `==`：基本类型比较值，引用类型比较地址
- `equals`：默认比较地址（Object），String/Integer 等重写为值比较
- 重写 equals 必须重写 hashCode

### 6.3 接口 vs 抽象类

| 特性 | 接口 | 抽象类 |
|------|------|--------|
| 多继承 | 支持 | 不支持 |
| 构造方法 | 无 | 有 |
| 成员变量 | 只有 static final 常量 | 可以有实例变量 |
| 方法 | JDK8+ 支持 default 方法 | 可以有普通方法 |
| 设计理念 | "能做什么"（行为契约） | "是什么"（代码复用） |

### 6.4 异常体系

```
Throwable
├── Error（不可恢复：OOM、StackOverflow）
└── Exception
    ├── RuntimeException（非受检：NullPointer、IndexOutOfBounds、ClassCast）
    └── 受检异常（必须 try-catch / throws：IOException、SQLException）
```
