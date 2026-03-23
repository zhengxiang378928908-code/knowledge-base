# JVM

## 内存区域

```
线程私有：
├── 程序计数器（PC）— 当前执行的字节码行号
├── 虚拟机栈（VM Stack）— 方法调用栈帧（局部变量表、操作数栈、动态链接、返回地址）
└── 本地方法栈（Native Method Stack）

线程共享：
├── 堆（Heap）— 对象实例、数组
│   ├── 新生代（Eden + S0 + S1）
│   └── 老年代
└── 方法区 / 元空间（Metaspace）— 类信息、常量池、静态变量
```

## 垃圾回收

### 判断对象是否可回收

1. **引用计数法**（不用，有循环引用问题）
2. **可达性分析**（GC Roots → 引用链）

GC Roots 包括：
- 栈帧中的局部变量
- 静态变量
- 常量
- JNI 引用

### 四种引用

| 引用类型 | 回收时机 | 典型用途 |
|----------|----------|----------|
| 强引用 | 不回收 | 普通赋值 |
| 软引用 | 内存不足时 | 缓存 |
| 弱引用 | 下次 GC | WeakHashMap, ThreadLocalMap |
| 虚引用 | 随时 | 追踪对象被回收的时机 |

### GC 算法

- **标记-清除**：产生碎片
- **标记-整理**：移动存活对象，无碎片
- **复制算法**：新生代 Eden → Survivor
- **分代收集**：新生代用复制，老年代用标记-整理

### 常见垃圾收集器

| 收集器 | 区域 | 特点 |
|--------|------|------|
| Serial | 新生代 | 单线程，STW |
| ParNew | 新生代 | 多线程版 Serial |
| Parallel Scavenge | 新生代 | 吞吐量优先 |
| CMS | 老年代 | 低延迟，并发标记 |
| G1 | 全堆 | 分区收集，可预测停顿 |
| ZGC | 全堆 | 超低延迟（< 1ms），Java 15+ |

### G1 收集器

- 堆划分为等大的 Region（1-32MB）
- 优先回收垃圾最多的 Region（Garbage First）
- Mixed GC：新生代 + 部分老年代
- 可通过 `-XX:MaxGCPauseMillis` 设定停顿目标

## 类加载

### 类加载过程

```
加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
```

### 双亲委派模型

```
BootstrapClassLoader（rt.jar）
    ↑
ExtClassLoader（ext/*.jar）
    ↑
AppClassLoader（classpath）
    ↑
自定义 ClassLoader
```

- 先委派父加载器加载，父加载器找不到才自己加载
- **目的**：防止核心类被篡改，保证类的唯一性

### 打破双亲委派

- SPI 机制（JDBC、JNDI）→ Thread.getContextClassLoader()
- OSGi / 热部署
- Tomcat：每个 Web 应用独立 ClassLoader

## JVM 调优

> 核心目标：减少 GC 次数、降低 GC 停顿时间、避免 OOM / 内存泄漏

### 面试回答结构

我一般从三个维度分析 JVM：
1. 内存结构（堆/新生代/老年代）
2. GC 行为（频率、停顿、算法）
3. 对象特征（生命周期、创建速率）

通过监控 + GC 日志定位问题，再进行参数调优。

### 常用参数

```bash
-Xms512m          # 初始堆
-Xmx2g            # 最大堆
-Xmn256m          # 新生代大小
-XX:MetaspaceSize=256m
-XX:+UseG1GC      # 使用 G1
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/dump.hprof
-XX:+PrintGCDetails   # GC 日志（JDK8）
-Xlog:gc*             # GC 日志（JDK9+）
```

### 调优思路

**第一步：先看 GC 日志，不盲调**
- GC 是否频繁
- 单次 GC 是否耗时过长

**Minor GC 频繁**
- 原因：新生代太小 / 对象创建过多
- 优化：调大新生代（`-Xmn`）、减少短命对象创建

**Full GC 频繁**
- 原因：老年代满 / 对象晋升过快 / 内存泄漏
- 优化：增大堆（`-Xmx`）、调整新生代比例、dump + MAT 分析泄漏

**GC 停顿时间长**
- 优化：换用 G1 / ZGC

### 调优步骤（高分回答模板）

1. 监控（CPU / 内存 / GC 日志）
2. 判断问题类型（频繁 GC or 停顿过长）
3. 调整内存结构（堆大小、新生代比例）
4. 排查内存泄漏（jmap dump → MAT 分析引用链）
5. 选择合适 GC 算法（G1 / ZGC）
6. 压测验证效果

### 常用诊断工具

| 工具 | 用途 |
|------|------|
| jps | 查看 Java 进程 |
| jstat | GC 统计 |
| jmap | 堆 dump |
| jstack | 线程 dump |
| jconsole / VisualVM | 可视化监控 |
| Arthas | 在线诊断神器 |
| MAT | 分析内存泄漏（引用链） |

### MAT 内存泄漏排查实战

**获取堆 dump**

```bash
# 手动 dump
jmap -dump:format=b,file=/tmp/dump.hprof <pid>

# OOM 时自动 dump（推荐提前配置）
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/dump.hprof
```

**MAT 核心功能**

| 功能 | 作用 |
|------|------|
| Leak Suspects Report | 自动分析疑似泄漏点（打开 dump 后自动弹出） |
| Dominator Tree | 按对象占用内存从大到小排列，快速定位大对象 |
| Histogram | 按类统计对象数量和内存占用 |
| Path to GC Roots | 查看引用链，找出谁持有对象导致无法回收 |

**排查流程**

```
打开 dump → Leak Suspects（看自动报告）
  → Dominator Tree（找最大对象）
  → 右键可疑对象 → Path to GC Roots → exclude weak references
  → 顺着引用链定位泄漏代码
```

**面试话术**：Full GC 后内存不下降，怀疑内存泄漏。用 `jmap` dump 堆内存，MAT 打开后看 Leak Suspects 报告，再通过 Dominator Tree 找到占用最大的对象，右键查看 Path to GC Roots，顺着引用链定位到泄漏的代码位置。

## 高频面试题

1. **JVM 内存模型和内存区域的区别？**（JMM 是并发规范，内存区域是运行时结构）
2. **什么时候触发 Full GC？**
3. **OOM 排查思路？**
4. **G1 和 CMS 区别？**
5. **类加载器和双亲委派模型？什么场景会打破？**
6. **如何判断内存泄漏？** Full GC 后内存不下降 → MAT 查看引用链
7. **对象为什么进入老年代？** Survivor 放不下 / 年龄达到阈值（默认 15）
8. **G1 优势？** 无碎片、可预测停顿、Region 粒度回收
