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
```

### 常用诊断工具

| 工具 | 用途 |
|------|------|
| jps | 查看 Java 进程 |
| jstat | GC 统计 |
| jmap | 堆 dump |
| jstack | 线程 dump |
| jconsole / VisualVM | 可视化监控 |
| Arthas | 在线诊断神器 |

## 高频面试题

1. **JVM 内存模型和内存区域的区别？**（JMM 是并发规范，内存区域是运行时结构）
2. **什么时候触发 Full GC？**
3. **OOM 排查思路？**
4. **G1 和 CMS 区别？**
5. **类加载器和双亲委派模型？什么场景会打破？**
