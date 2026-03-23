# Spring 框架

## IoC（控制反转）

- 对象的创建和依赖关系由 Spring 容器管理，而非手动 new
- **DI（依赖注入）** 是 IoC 的实现方式

### 注入方式

1. **构造器注入**（推荐）— 不可变，强制依赖
2. **Setter 注入** — 可选依赖
3. **字段注入**（`@Autowired`）— 不推荐，不利于测试

### Bean 作用域

| 作用域 | 说明 |
|--------|------|
| singleton | 默认，全局唯一 |
| prototype | 每次请求创建新实例 |
| request | 每个 HTTP 请求一个 |
| session | 每个 HTTP 会话一个 |

### Bean 生命周期

```
实例化 → 属性注入 → BeanNameAware → BeanFactoryAware
→ ApplicationContextAware → BeanPostProcessor.before
→ InitializingBean.afterPropertiesSet → @PostConstruct / init-method
→ BeanPostProcessor.after → 使用
→ @PreDestroy / destroy-method → DisposableBean.destroy
```

## AOP（面向切面编程）

### 核心概念

- **切面（Aspect）**：横切关注点的模块化
- **连接点（JoinPoint）**：可以拦截的点（方法执行）
- **切点（Pointcut）**：匹配连接点的表达式
- **通知（Advice）**：before / after / around / afterReturning / afterThrowing
- **织入（Weaving）**：将切面应用到目标对象的过程

### 实现原理

- **JDK 动态代理**：基于接口（Proxy + InvocationHandler）
- **CGLIB 代理**：基于子类继承（无需接口）
- Spring 默认：有接口用 JDK，无接口用 CGLIB

## Spring MVC

### 请求处理流程

```
请求 → DispatcherServlet → HandlerMapping 找到 Handler
→ HandlerAdapter 调用 Controller → 返回 ModelAndView
→ ViewResolver 解析视图 → 渲染响应
```

## Spring Boot

### 自动配置原理

1. `@SpringBootApplication` 包含 `@EnableAutoConfiguration`
2. 按版本加载配置类：
   - Boot 2.x：`spring.factories`
   - Boot 3.x+：`AutoConfiguration.imports`
3. `@Conditional` 系列注解决定是否生效
4. 开发者可通过 `application.yml` 覆盖默认配置

### 常用注解

| 注解 | 用途 |
|------|------|
| `@RestController` | @Controller + @ResponseBody |
| `@RequestMapping` | 映射请求路径 |
| `@Autowired` | 自动注入依赖 |
| `@Value` | 注入配置值 |
| `@ConfigurationProperties` | 绑定配置类 |
| `@Transactional` | 声明式事务 |

## 事务

### @Transactional

- 默认传播行为：`REQUIRED`（加入当前事务，没有就新建）
- 默认回滚：`RuntimeException` 和 `Error`
- `rollbackFor = Exception.class` 可扩大回滚范围

### 事务失效场景

1. 标在非 `public` 方法上的事务在标准代理模式下通常不要依赖其生效
2. 自调用（同类方法调用，未经过代理）
3. 异常被 catch 吞掉
4. 数据库不支持事务（MyISAM）
5. 传播行为设置不当

### 传播行为

| 传播行为 | 说明 |
|----------|------|
| REQUIRED | 有事务就加入，没有就新建（默认） |
| REQUIRES_NEW | 总是新建，挂起当前事务 |
| NESTED | 嵌套事务（savepoint） |
| SUPPORTS | 有就加入，没有就非事务执行 |
| NOT_SUPPORTED | 非事务执行，挂起当前事务 |

## 循环依赖

Spring 通过三级缓存解决 singleton 的循环依赖：

1. `singletonObjects` — 完成品
2. `earlySingletonObjects` — 半成品（已实例化，未填充属性）
3. `singletonFactories` — 对象工厂（用于创建代理）

> 构造器注入的循环依赖无法解决，需用 `@Lazy` 打破。

## 高频面试题

1. **Spring IoC 容器的初始化过程？**
   - 加载配置（XML / 注解扫描）→ BeanDefinition 注册 → 实例化 → 属性注入 → 初始化（Aware 回调、BeanPostProcessor、init-method）→ 就绪

2. **BeanFactory 和 ApplicationContext 区别？**
   - BeanFactory：最基础的容器，懒加载
   - ApplicationContext：BeanFactory 的子接口，启动时预加载所有单例 Bean，支持国际化、事件、AOP 等

3. **AOP 是怎么实现的？JDK 动态代理 vs CGLIB？**
   - JDK 动态代理：基于接口，Proxy + InvocationHandler，目标类必须实现接口
   - CGLIB：基于继承，生成目标类的子类，不需要接口但不能代理 final 类/方法
   - Spring 默认：有接口用 JDK，无接口用 CGLIB；Spring Boot 2.x 起默认全用 CGLIB

4. **@Transactional 失效的场景？**（非 public 方法、同类自调用绕过代理、异常被 catch 吞掉、数据库引擎不支持、传播行为设置不当）

5. **Spring Boot 自动配置原理？**
   - `@SpringBootApplication` → `@EnableAutoConfiguration` → 加载 `spring.factories`（Boot 2.x）或 `AutoConfiguration.imports`（Boot 3.x）→ `@Conditional` 按条件决定是否生效

6. **循环依赖怎么解决？**
   - 三级缓存：singletonObjects（完成品）→ earlySingletonObjects（半成品）→ singletonFactories（对象工厂，用于创建代理）
   - 构造器注入的循环依赖无法解决，用 `@Lazy` 打破
