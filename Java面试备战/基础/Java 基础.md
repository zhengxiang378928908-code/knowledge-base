# Java 基础

## 数据类型

### 基本类型

| 类型 | 字节 | 默认值 | 范围 |
|------|------|--------|------|
| byte | 1 | 0 | -128 ~ 127 |
| short | 2 | 0 | -32768 ~ 32767 |
| int | 4 | 0 | -2^31 ~ 2^31-1 |
| long | 8 | 0L | -2^63 ~ 2^63-1 |
| float | 4 | 0.0f | IEEE 754 |
| double | 8 | 0.0d | IEEE 754 |
| char | 2 | '\u0000' | 0 ~ 65535 |
| boolean | - | false | true / false |

### 自动装箱与拆箱

```java
Integer a = 127;  // 自动装箱 Integer.valueOf(127)
int b = a;        // 自动拆箱 a.intValue()

// 缓存陷阱：-128 ~ 127 范围内的 Integer 会缓存
Integer x = 127, y = 127;
x == y  // true（同一个缓存对象）

Integer m = 128, n = 128;
m == n  // false（不同对象）
```

## String

- **不可变性**：`String` 内部用 `final byte[]` 存储（Java 9+），创建后不可修改
- **字符串常量池**：字面量存在常量池中，`new String()` 在堆上创建新对象
- `String` vs `StringBuilder` vs `StringBuffer`：
  - `String`：不可变，线程安全
  - `StringBuilder`：可变，非线程安全，性能好
  - `StringBuffer`：可变，线程安全（synchronized），性能差

```java
String s1 = "hello";           // 常量池
String s2 = new String("hello"); // 堆
s1 == s2      // false
s1.equals(s2) // true
s1 == s2.intern() // true
```

## == 与 equals

- `==`：比较引用地址（基本类型比较值）
- `equals()`：默认同 `==`，String/Integer 等重写为值比较
- 重写 `equals()` 必须同时重写 `hashCode()`

## 面向对象

### 三大特性

1. **封装**：隐藏实现细节，暴露接口
2. **继承**：子类继承父类的属性和方法（Java 单继承）
3. **多态**：同一接口不同实现，运行时动态绑定

### 抽象类 vs 接口

| 对比项 | 抽象类 | 接口 |
|--------|--------|------|
| 实例化 | 不能 | 不能 |
| 构造方法 | 可以有 | 不能有 |
| 成员变量 | 任意 | 只能 public static final |
| 方法 | 可以有实现 | Java 8 起可以有 default/static 方法 |
| 继承/实现 | 单继承 | 多实现 |

## 异常体系

```
Throwable
├── Error（不可恢复：OOM、StackOverflow）
└── Exception
    ├── RuntimeException（非受检：NullPointer、ArrayIndex、ClassCast）
    └── 受检异常（IOException、SQLException，必须 try-catch 或 throws）
```

### try-with-resources

```java
try (var is = new FileInputStream("f.txt")) {
    // 自动关闭，实现 AutoCloseable 接口的资源
} catch (IOException e) {
    e.printStackTrace();
}
```

## 反射

```java
Class<?> clazz = Class.forName("com.example.User");
Object obj = clazz.getDeclaredConstructor().newInstance();
Method m = clazz.getDeclaredMethod("getName");
m.setAccessible(true);
String name = (String) m.invoke(obj);
```

**应用场景**：Spring IoC、MyBatis、动态代理、序列化框架

## 泛型

- **类型擦除**：编译期检查，运行时擦除为 Object
- `<? extends T>`：上界通配符，只读
- `<? super T>`：下界通配符，只写
- PECS 原则：Producer Extends, Consumer Super

## 高频面试题

1. **final、finally、finalize 的区别？**
2. **String 为什么不可变？不可变有什么好处？**
3. **Java 是值传递还是引用传递？**（值传递，对象传的是引用的副本）
4. **深拷贝 vs 浅拷贝？**
5. **Object 类有哪些方法？**（equals, hashCode, toString, clone, wait, notify, finalize, getClass）
