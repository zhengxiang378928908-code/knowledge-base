# 社招高频题 + SQL 高频题 + 手写代码基础题

> 这篇文档适合中小公司和常规 Java 后端社招复习。
> 使用方式：先过一遍 100 道高频题，再刷 SQL 题，最后练手写代码题。

---

## 一、100 道社招高频题 + 答案

### 1. Java 基础

1. `==` 和 `equals()` 的区别？
答：`==` 比较基本类型的值或对象引用地址；`equals()` 比较对象内容，前提是类重写了该方法。

2. `hashCode()` 和 `equals()` 为什么要一起重写？
答：哈希容器先用 `hashCode()` 定位桶，再用 `equals()` 判等，只重写一个会导致去重和查找异常。

3. `String` 为什么不可变？
答：便于字符串常量池复用、线程安全、缓存哈希值，也更适合作为 `HashMap` 的 key。

4. `StringBuilder` 和 `StringBuffer` 的区别？
答：`StringBuilder` 线程不安全但性能高；`StringBuffer` 线程安全但性能相对低。

5. `final`、`finally`、`finalize` 的区别？
答：`final` 是关键字；`finally` 是异常处理代码块；`finalize` 是已过时的对象回收前回调方法。

6. 重载和重写的区别？
答：重载发生在同一个类中，方法名相同参数不同；重写发生在父子类之间，方法签名相同。

7. 接口和抽象类的区别？
答：接口更偏行为规范和多实现；抽象类更偏模板复用和公共实现沉淀。

8. 基本类型和包装类型的区别？
答：基本类型直接存值；包装类型是对象，可为 `null`，并提供很多工具方法。

9. 自动装箱和拆箱是什么？
答：基本类型和包装类型之间的自动转换，例如 `int` 和 `Integer` 之间的转换。

10. `Integer` 缓存范围是多少？
答：默认缓存 `-128` 到 `127`。

11. `Object` 常用方法有哪些？
答：`equals()`、`hashCode()`、`toString()`、`wait()`、`notify()`、`notifyAll()`、`clone()`。

12. `try-catch-finally` 的执行顺序是什么？
答：先执行 `try`，发生异常则进入 `catch`，最后通常执行 `finally`。

13. `throw` 和 `throws` 的区别？
答：`throw` 是在方法体内主动抛出异常对象；`throws` 是在方法声明处声明可能抛出的异常。

14. checked 异常和 unchecked 异常的区别？
答：checked 异常编译期强制处理；unchecked 异常一般指运行时异常，可不显式捕获。

15. `finally` 一定会执行吗？
答：通常会执行，但进程被强制退出、机器宕机、调用 `System.exit()` 等情况不会。

16. `static` 能修饰什么？
答：变量、方法、代码块、内部类。

17. 静态变量和实例变量的区别？
答：静态变量属于类，所有对象共享；实例变量属于对象，每个对象一份。

18. `this` 和 `super` 的区别？
答：`this` 表示当前对象；`super` 表示父类对象引用语义，用于访问父类成员。

19. 深拷贝和浅拷贝的区别？
答：浅拷贝只复制对象本身和引用地址；深拷贝连引用对象内容一起复制。

20. Java 是值传递还是引用传递？
答：Java 只有值传递，对象参数传递的是引用变量的副本。

### 2. 集合

21. `List`、`Set`、`Map` 的区别？
答：`List` 有序可重复；`Set` 无重复；`Map` 存键值对。

22. `ArrayList` 和 `LinkedList` 的区别？
答：`ArrayList` 底层是数组，查询快；`LinkedList` 底层是链表，特定场景插删更方便。

23. `ArrayList` 的扩容机制是什么？
答：JDK 8 中一般扩容为原容量的 1.5 倍。

24. `Vector` 和 `ArrayList` 的区别？
答：`Vector` 线程安全但性能低；`ArrayList` 线程不安全但更常用。

25. `HashMap` 底层结构是什么？
答：JDK 1.8 是数组 + 链表 + 红黑树。

26. JDK 1.8 `HashMap` 为什么引入红黑树？
答：当哈希冲突严重时，链表查找会退化成 O(n)，红黑树能优化到 O(logn)。

27. 链表什么时候转红黑树？
答：桶内元素个数至少 8 且数组容量至少 64。

28. `HashMap` 为什么线程不安全？
答：多线程 put 和 resize 时可能导致数据覆盖、结构异常、读取不一致。

29. `Hashtable` 和 `HashMap` 的区别？
答：`Hashtable` 线程安全且不允许 `null`；`HashMap` 线程不安全且允许 `null`。

30. `ConcurrentHashMap` 如何保证线程安全？
答：JDK 1.8 中主要通过 CAS + `synchronized` 控制桶级并发。

31. `HashSet` 为什么元素不能重复？
答：底层依赖 `HashMap`，通过 `hashCode()` 和 `equals()` 判重。

32. `TreeMap` 和 `HashMap` 的区别？
答：`TreeMap` 基于红黑树，按 key 有序；`HashMap` 无序。

33. `LinkedHashMap` 的特点是什么？
答：保留插入顺序或访问顺序，常用于实现 LRU 缓存。

34. fail-fast 是什么？
答：遍历集合时如果被结构性修改，会快速抛出 `ConcurrentModificationException`。

35. `Iterator` 和 `ListIterator` 的区别？
答：`ListIterator` 支持双向遍历，还支持在遍历时修改列表。

### 3. 并发

36. 创建线程的方式有哪些？
答：继承 `Thread`、实现 `Runnable`、实现 `Callable`、使用线程池。

37. `Runnable` 和 `Callable` 的区别？
答：`Callable` 有返回值且可抛异常；`Runnable` 没有返回值。

38. `Future` 的作用是什么？
答：表示异步执行结果，可用于阻塞获取、取消任务、判断任务状态。

39. `sleep()` 和 `wait()` 的区别？
答：`sleep()` 不释放锁；`wait()` 会释放锁，且必须在同步代码中使用。

40. `notify()` 和 `notifyAll()` 的区别？
答：`notify()` 唤醒一个等待线程；`notifyAll()` 唤醒全部等待线程。

41. `synchronized` 锁的是什么？
答：实例方法锁当前对象；静态方法锁类对象；同步块锁传入的对象。

42. `volatile` 的作用是什么？
答：保证变量可见性和一定的有序性，但不能保证复合操作的原子性。

43. 为什么 `volatile` 不能保证 `i++` 安全？
答：`i++` 包含读取、计算、写回三个步骤，不是原子操作。

44. CAS 是什么？
答：CAS 是比较并交换，是一种无锁原子更新机制。

45. CAS 有什么问题？
答：ABA 问题、自旋开销大、只能保证单变量原子性。

46. 什么是 AQS？
答：AQS 是抽象队列同步器，是很多并发工具类的底层实现基础。

47. `ReentrantLock` 和 `synchronized` 的区别？
答：`ReentrantLock` 更灵活，支持可中断、可超时、公平锁；`synchronized` 语法更简单。

48. `ThreadLocal` 的作用是什么？
答：为每个线程维护独立变量副本，适合存储线程上下文信息。

49. `ThreadLocal` 为什么可能内存泄漏？
答：线程池线程长时间存活时，key 被回收后 value 可能残留在线程本地 map 中。

50. `CountDownLatch` 和 `CyclicBarrier` 的区别？
答：前者是倒计时器，一次性；后者是栅栏，可循环复用。

51. `Semaphore` 的作用是什么？
答：控制同时访问某资源的线程数量。

52. 线程池核心参数有哪些？
答：核心线程数、最大线程数、空闲线程存活时间、阻塞队列、线程工厂、拒绝策略。

53. 常见拒绝策略有哪些？
答：直接抛异常、调用者执行、丢弃当前任务、丢弃队列中最旧任务。

54. 为什么不推荐直接使用 `Executors`？
答：默认线程池参数可能导致无界队列堆积或线程数失控，引发 OOM。

55. 线程池执行流程是什么？
答：先用核心线程，核心满了进队列，队列满了再扩容非核心线程，最后触发拒绝策略。

### 4. JVM

56. JVM 内存结构有哪些？
答：堆、虚拟机栈、方法区、程序计数器、本地方法栈。

57. 堆和栈的区别？
答：堆主要存对象；栈主要存方法调用信息和局部变量。

58. 对象一般分配在哪里？
答：通常分配在堆中的新生代 Eden 区。

59. 新生代为什么分 Eden 和 Survivor？
答：为了提高对象回收效率，减少内存碎片和复制成本。

60. 什么对象会进入老年代？
答：长期存活对象、大对象、经过多次 GC 仍存活的对象。

61. 常见 GC 算法有哪些？
答：标记清除、复制、标记整理、分代收集。

62. Minor GC 和 Full GC 的区别？
答：Minor GC 主要回收新生代；Full GC 通常涉及老年代，停顿时间更长。

63. Full GC 常见触发条件有哪些？
答：老年代空间不足、元空间不足、显示调用 `System.gc()` 等。

64. 类加载过程是什么？
答：加载、验证、准备、解析、初始化。

65. 双亲委派模型是什么？
答：类加载时优先委派父加载器，避免重复加载和核心类被业务类替换。

66. 常见 OOM 类型有哪些？
答：堆内存溢出、栈溢出、元空间溢出、直接内存溢出。

67. 如何排查内存泄漏？
答：通过 GC 日志、heap dump、MAT 分析大对象和引用链定位问题。

68. `jstack` 有什么用？
答：查看线程堆栈，排查死锁、线程阻塞、CPU 飙高问题。

69. `jmap` 有什么用？
答：查看堆使用情况，或导出堆转储文件供离线分析。

70. CMS 和 G1 的区别？
答：G1 更适合大堆和可预测停顿场景，按 Region 回收；CMS 是老一代低停顿回收器。

### 5. Spring / Spring Boot

71. IOC 是什么？
答：对象创建和依赖关系由 Spring 容器统一管理。

72. AOP 是什么？
答：把日志、事务、权限等横切逻辑从业务代码中抽离并统一增强。

73. Bean 生命周期大致过程是什么？
答：实例化、属性注入、初始化、使用、销毁。

74. `@Autowired` 和 `@Resource` 的区别？
答：`@Autowired` 默认按类型注入；`@Resource` 默认按名称注入。

75. Spring 如何解决循环依赖？
答：单例 setter 注入场景下，通过三级缓存提前暴露对象引用解决。

76. 为什么构造器循环依赖很难处理？
答：因为对象都还没创建完成，无法提前暴露可用引用。

77. `@Transactional` 的底层原理是什么？
答：基于 AOP 代理，在目标方法前后控制事务的开启、提交和回滚。

78. `@Transactional` 失效场景有哪些？
答：同类内部调用、非 `public` 方法、异常被吞掉、对象不是 Spring 管理。

79. 常见事务传播行为有哪些？
答：`REQUIRED`、`REQUIRES_NEW`、`SUPPORTS`、`MANDATORY` 等。

80. Spring Boot 自动配置原理是什么？
答：通过自动配置类和条件注解，在满足条件时自动装配 Bean。

### 6. MyBatis / MySQL

81. MyBatis 中 `#{}` 和 `${}` 的区别？
答：`#{}` 是预编译占位符，更安全；`${}` 是字符串拼接，容易引发 SQL 注入。

82. MyBatis 一级缓存和二级缓存的区别？
答：一级缓存是 SqlSession 级别；二级缓存是 Mapper 级别。

83. 什么是 SQL 注入？
答：恶意输入被拼入 SQL，导致查询语义被篡改。

84. 索引底层为什么用 B+ 树？
答：更适合磁盘读写，范围查询和排序性能更好。

85. 聚簇索引和非聚簇索引的区别？
答：聚簇索引叶子节点存整行数据；非聚簇索引叶子节点通常存主键或地址。

86. 联合索引最左匹配原则是什么？
答：查询条件尽量从联合索引最左边字段开始使用。

87. 索引失效常见原因有哪些？
答：对索引列做函数运算、隐式类型转换、前导模糊查询、范围后再用后续列。

88. 事务四大特性是什么？
答：原子性、一致性、隔离性、持久性。

89. 事务隔离级别有哪些？
答：读未提交、读已提交、可重复读、串行化。

90. 脏读、不可重复读、幻读的区别？
答：脏读读到未提交数据；不可重复读是同一行前后值不同；幻读是范围查询前后条数变化。

### 7. Redis / MQ / 分布式 / 排查

91. Redis 为什么快？
答：基于内存、单线程减少锁竞争、I/O 多路复用、数据结构高效。

92. Redis 持久化方式有哪些？
答：RDB 和 AOF。

93. 缓存穿透、击穿、雪崩怎么解决？
答：穿透用布隆过滤器或缓存空值；击穿用互斥锁；雪崩用过期时间打散和降级兜底。

94. Redis 分布式锁怎么实现？
答：通常用 `SET key value NX PX`，释放锁时校验 value 防止误删。

95. 为什么要用消息队列？
答：为了系统解耦、异步处理、削峰填谷、提升系统容错性。

96. 如何保证消息不丢失？
答：生产确认、Broker 持久化、消费确认、失败重试和补偿。

97. 如何保证消息幂等？
答：通过唯一业务键、去重表、状态机、防重校验等手段。

98. CAP 理论是什么？
答：一致性、可用性、分区容错性三者无法在分布式系统中同时完全满足。

99. 接口慢怎么排查？
答：先看监控和日志，再查线程池、慢 SQL、缓存命中率和下游接口耗时。

100. 线上 CPU 飙高怎么排查？
答：先用 `top` 找高 CPU 进程，必要时用 `jps -l` 确认 Java 应用，再用 `top -H -p <pid>` 找高 CPU 线程，把线程 ID 转十六进制后结合 `jstack` 看线程栈，最后定位是死循环、频繁 GC、锁竞争还是热点代码。

---

## 二、SQL 高频题

> 建议这部分配合 MySQL 本地库或在线练习平台一起刷。

### 1. 基础查询

1. 查询学生表中所有学生信息。
```sql
SELECT * FROM student;
```

2. 查询姓名和年龄两列。
```sql
SELECT name, age FROM student;
```

3. 查询年龄大于 18 岁的学生。
```sql
SELECT * FROM student WHERE age > 18;
```

4. 查询姓名以 `张` 开头的学生。
```sql
SELECT * FROM student WHERE name LIKE '张%';
```

5. 查询年龄在 18 到 25 之间的学生。
```sql
SELECT * FROM student WHERE age BETWEEN 18 AND 25;
```

### 2. 排序与分页

6. 按年龄降序查询学生。
```sql
SELECT * FROM student ORDER BY age DESC;
```

7. 查询前 10 条数据。
```sql
SELECT * FROM student LIMIT 10;
```

8. 查询第 2 页，每页 10 条。
```sql
SELECT * FROM student LIMIT 10, 10;
```

9. 按创建时间倒序，名称正序排序。
```sql
SELECT * FROM student ORDER BY create_time DESC, name ASC;
```

10. 深分页为什么慢？
答：因为 `LIMIT offset, size` 偏移量越大，扫描和丢弃的行越多，性能变差。

### 3. 聚合与分组

11. 统计学生总数。
```sql
SELECT COUNT(*) FROM student;
```

12. 统计每个班级的人数。
```sql
SELECT class_id, COUNT(*) AS total
FROM student
GROUP BY class_id;
```

13. 查询每个班级平均年龄。
```sql
SELECT class_id, AVG(age) AS avg_age
FROM student
GROUP BY class_id;
```

14. 查询人数大于 30 的班级。
```sql
SELECT class_id, COUNT(*) AS total
FROM student
GROUP BY class_id
HAVING COUNT(*) > 30;
```

15. `WHERE` 和 `HAVING` 的区别？
答：`WHERE` 在分组前过滤；`HAVING` 在分组后过滤聚合结果。

### 4. 连接查询

16. 查询学生及其班级名称。
```sql
SELECT s.id, s.name, c.class_name
FROM student s
LEFT JOIN class c ON s.class_id = c.id;
```

17. `INNER JOIN` 和 `LEFT JOIN` 的区别？
答：`INNER JOIN` 只返回匹配上的数据；`LEFT JOIN` 保留左表全部数据。

18. 查询没有班级的学生。
```sql
SELECT s.*
FROM student s
LEFT JOIN class c ON s.class_id = c.id
WHERE c.id IS NULL;
```

19. 三表关联时优化重点是什么？
答：确保关联字段有索引，减少中间结果集，尽量让过滤条件尽早生效。

20. 什么是笛卡尔积？
答：没有正确关联条件时，多表每条记录互相组合，结果数爆炸。

### 5. 子查询与去重

21. 查询年龄大于平均年龄的学生。
```sql
SELECT *
FROM student
WHERE age > (SELECT AVG(age) FROM student);
```

22. 查询每个班年龄最大的学生。
```sql
SELECT s.*
FROM student s
JOIN (
    SELECT class_id, MAX(age) AS max_age
    FROM student
    GROUP BY class_id
) t ON s.class_id = t.class_id AND s.age = t.max_age;
```

23. 查询去重后的班级编号。
```sql
SELECT DISTINCT class_id FROM student;
```

24. `DISTINCT` 和 `GROUP BY` 的区别？
答：都能去重，但 `GROUP BY` 更适合分组统计，`DISTINCT` 更适合简单去重。

25. `EXISTS` 和 `IN` 的区别？
答：`EXISTS` 常用于关联判断，命中即返回；`IN` 更适合小结果集匹配。

### 6. 更新、删除、事务

26. 给某个学生年龄加 1。
```sql
UPDATE student
SET age = age + 1
WHERE id = 1;
```

27. 删除年龄小于 10 的学生。
```sql
DELETE FROM student
WHERE age < 10;
```

28. 事务的四大特性是什么？
答：原子性、一致性、隔离性、持久性。

29. MySQL 默认隔离级别是什么？
答：InnoDB 默认是可重复读。

30. 乐观锁和悲观锁的区别？
答：乐观锁假设冲突少，通常靠版本号；悲观锁假设冲突多，先加锁再操作。

### 7. 索引与优化

31. 索引为什么能加快查询？
答：因为索引能减少全表扫描，快速定位到符合条件的数据范围。

32. 什么字段适合建索引？
答：高频查询条件、关联字段、排序字段、区分度较高字段。

33. 什么字段不适合建索引？
答：更新非常频繁、区分度低、字段很长且很少查询的列。

34. 联合索引 `(a,b,c)` 能支持哪些查询？
答：通常支持 `a`、`a+b`、`a+b+c`，以及以 `a` 开头的部分组合。

35. 为什么 `LIKE '%abc'` 容易失效？
答：前导 `%` 无法利用有序索引做前缀匹配。

36. 如何看 SQL 是否走索引？
答：通过 `EXPLAIN` 查看执行计划。

37. `EXPLAIN` 里重点看哪些字段？
答：`type`、`key`、`rows`、`extra`。

38. `type=all` 一般表示什么？
答：表示全表扫描，通常性能较差。

39. 覆盖索引是什么？
答：查询字段都能从索引中拿到，不需要回表。

40. 回表是什么？
答：先通过二级索引定位主键，再回聚簇索引查完整数据。

---

## 三、手写代码基础题

> 这部分建议自己先写，再看答案。

### 1. 字符串与数组

1. 反转字符串。
```java
public String reverse(String s) {
    return new StringBuilder(s).reverse().toString();
}
```

2. 判断字符串是否是回文串。
```java
public boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) {
            return false;
        }
        left++;
        right--;
    }
    return true;
}
```

3. 数组中找最大值。
```java
public int max(int[] nums) {
    int max = nums[0];
    for (int num : nums) {
        if (num > max) {
            max = num;
        }
    }
    return max;
}
```

4. 数组去重。
```java
public int[] distinct(int[] nums) {
    return java.util.Arrays.stream(nums).distinct().toArray();
}
```

5. 两数之和。
```java
public int[] twoSum(int[] nums, int target) {
    java.util.Map<Integer, Integer> map = new java.util.HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int need = target - nums[i];
        if (map.containsKey(need)) {
            return new int[]{map.get(need), i};
        }
        map.put(nums[i], i);
    }
    return new int[0];
}
```

### 2. 链表

6. 反转单链表。
```java
public ListNode reverseList(ListNode head) {
    ListNode prev = null;
    ListNode curr = head;
    while (curr != null) {
        ListNode next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}
```

7. 判断链表是否有环。
```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            return true;
        }
    }
    return false;
}
```

8. 合并两个有序链表。
```java
public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0);
    ListNode curr = dummy;
    while (l1 != null && l2 != null) {
        if (l1.val < l2.val) {
            curr.next = l1;
            l1 = l1.next;
        } else {
            curr.next = l2;
            l2 = l2.next;
        }
        curr = curr.next;
    }
    curr.next = (l1 != null) ? l1 : l2;
    return dummy.next;
}
```

9. 删除链表倒数第 N 个节点。
```java
public ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode fast = dummy, slow = dummy;
    for (int i = 0; i <= n; i++) {
        fast = fast.next;
    }
    while (fast != null) {
        fast = fast.next;
        slow = slow.next;
    }
    slow.next = slow.next.next;
    return dummy.next;
}
```

10. 找链表中间节点。
```java
public ListNode middleNode(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow;
}
```

### 3. 栈、队列、哈希

11. 用栈判断括号是否有效。
```java
public boolean isValid(String s) {
    java.util.Stack<Character> stack = new java.util.Stack<>();
    for (char c : s.toCharArray()) {
        if (c == '(') stack.push(')');
        else if (c == '[') stack.push(']');
        else if (c == '{') stack.push('}');
        else if (stack.isEmpty() || stack.pop() != c) return false;
    }
    return stack.isEmpty();
}
```

12. 用两个栈实现队列。
答：一个栈负责入队，一个栈负责出队；出队时如果出队栈为空，就把入队栈全部倒过去。

13. 统计字符串中每个字符出现次数。
```java
public java.util.Map<Character, Integer> countChars(String s) {
    java.util.Map<Character, Integer> map = new java.util.HashMap<>();
    for (char c : s.toCharArray()) {
        map.put(c, map.getOrDefault(c, 0) + 1);
    }
    return map;
}
```

14. 找出数组中出现次数最多的元素。
答：用 `HashMap` 计数，再遍历 map 找最大值。

15. 判断两个字符串是否是字母异位词。
```java
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) {
        return false;
    }
    int[] count = new int[26];
    for (char c : s.toCharArray()) count[c - 'a']++;
    for (char c : t.toCharArray()) count[c - 'a']--;
    for (int n : count) {
        if (n != 0) {
            return false;
        }
    }
    return true;
}
```

### 4. 排序与查找

16. 写一个冒泡排序。
```java
public void bubbleSort(int[] nums) {
    for (int i = 0; i < nums.length - 1; i++) {
        for (int j = 0; j < nums.length - 1 - i; j++) {
            if (nums[j] > nums[j + 1]) {
                int temp = nums[j];
                nums[j] = nums[j + 1];
                nums[j + 1] = temp;
            }
        }
    }
}
```

17. 二分查找。
```java
public int binarySearch(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) return mid;
        if (nums[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

18. 快速排序的核心思想是什么？
答：选一个基准值，把数组分成小于基准和大于基准两部分，再递归处理。

19. 归并排序的核心思想是什么？
答：分治，把数组拆分成更小子数组，分别排序后再合并。

20. 如何在有序数组中找第一个大于等于 target 的位置？
答：用二分查找的边界写法，最终返回左边界。

### 5. 二叉树

21. 二叉树前序遍历。
```java
public void preorder(TreeNode root) {
    if (root == null) return;
    System.out.println(root.val);
    preorder(root.left);
    preorder(root.right);
}
```

22. 二叉树层序遍历。
```java
public java.util.List<java.util.List<Integer>> levelOrder(TreeNode root) {
    java.util.List<java.util.List<Integer>> result = new java.util.ArrayList<>();
    if (root == null) return result;
    java.util.Queue<TreeNode> queue = new java.util.LinkedList<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        int size = queue.size();
        java.util.List<Integer> level = new java.util.ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(level);
    }
    return result;
}
```

23. 判断二叉树是否对称。
答：递归比较左子树的左节点和右子树的右节点，以及左子树的右节点和右子树的左节点。

24. 求二叉树最大深度。
```java
public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    return Math.max(maxDepth(root.left), maxDepth(root.right)) + 1;
}
```

25. 判断是否是二叉搜索树。
答：中序遍历应该严格递增，或递归校验每个节点的取值范围。

### 6. 动态规划与常见题

26. 斐波那契数列。
```java
public int fib(int n) {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

27. 爬楼梯问题。
答：`dp[i] = dp[i - 1] + dp[i - 2]`。

28. 最长连续子数组和。
答：用 Kadane 算法，状态转移是 `curr = max(num, curr + num)`。

29. 零钱兑换的思路是什么？
答：典型完全背包问题，状态表示凑到金额 `i` 的最少硬币数。

30. 最长公共子序列的思路是什么？
答：二维动态规划，比较两个字符串当前位置字符是否相等。

---

## 四、复习建议

### 1. 适合中小公司的准备顺序

1. 先背熟前 50 道 Java / 集合 / 并发 / Spring 高频题。
2. 再刷 SQL 高频题里的 `GROUP BY`、`JOIN`、`索引`、`事务`。
3. 最后练手写代码基础题里的链表、二叉树、二分、哈希和动态规划。

### 2. 一周冲刺建议

1. 第 1-2 天：刷高频题 1-50。
2. 第 3-4 天：刷高频题 51-100。
3. 第 5 天：把 SQL 题全部手敲一遍。
4. 第 6-7 天：把手写代码题自己写一遍，不看答案。

### 3. 面试回答原则

1. 先说结论。
2. 再说原理。
3. 最后结合项目经验。
