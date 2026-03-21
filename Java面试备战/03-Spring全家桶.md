# Spring 全家桶

> 你的核心技术栈，必须烂熟于心。

---

## 一、Spring 核心

### 1.1 IoC 和 DI

- **IoC（控制反转）**：对象的创建和管理权交给 Spring 容器，而不是开发者手动 new
- **DI（依赖注入）**：容器在创建对象时自动注入其依赖，三种方式：
  - 构造器注入（推荐）
  - Setter 注入
  - 字段注入（`@Autowired`）

### 1.2 Bean 生命周期

```
1. 实例化（Instantiation）
   ↓
2. 属性填充（Populate properties / DI）
   ↓
3. Aware 接口回调（BeanNameAware、ApplicationContextAware 等）
   ↓
4. BeanPostProcessor#postProcessBeforeInitialization
   ↓
5. @PostConstruct / InitializingBean#afterPropertiesSet / init-method
   ↓
6. BeanPostProcessor#postProcessAfterInitialization（AOP 代理在此生成）
   ↓
7. Bean 就绪，放入容器
   ↓
8. 容器关闭时：@PreDestroy / DisposableBean#destroy / destroy-method
```

**你项目中用到的：**
- `CommandLineRunner`：Bean 创建完成后执行（定时任务扫描器 `ScheduledTaskScanner`、清理器 `TaskStateCleanupRunner`）
- `ApplicationListener<ContextClosedEvent>`：容器关闭时执行（`ApplicationShutdownListener`）

### 1.3 Bean 作用域

| 作用域 | 描述 |
|--------|------|
| singleton | 默认，容器中只有一个实例 |
| prototype | 每次获取创建新实例 |
| request | 每个 HTTP 请求一个实例 |
| session | 每个 HTTP Session 一个实例 |

### 1.4 循环依赖

**问题**：A 依赖 B，B 依赖 A

**Spring 三级缓存解决（仅限 singleton + setter/field 注入）：**

| 缓存 | 内容 | 作用 |
|------|------|------|
| 一级缓存 singletonObjects | 完整 Bean | 正常获取 Bean |
| 二级缓存 earlySingletonObjects | 早期引用（可能是代理） | 解决循环依赖 |
| 三级缓存 singletonFactories | ObjectFactory（创建早期引用的工厂） | 延迟代理生成 |

**流程：**
1. 创建 A → 实例化（未注入属性）→ 放入三级缓存
2. A 注入 B → 创建 B → B 注入 A → 从三级缓存获取 A 的早期引用 → B 完成
3. A 注入 B 完成 → A 完成

**构造器注入无法解决循环依赖**（实例化就依赖对方）→ 会报错

---

## 二、Spring AOP

### 2.1 AOP 概念

- **切面（Aspect）**：横切关注点的模块化（如日志、事务、监控）
- **连接点（JoinPoint）**：可以被增强的点（方法执行）
- **切入点（Pointcut）**：定义在哪些连接点上应用切面
- **通知（Advice）**：增强逻辑
  - `@Before`：方法执行前
  - `@After`：方法执行后（无论是否异常）
  - `@AfterReturning`：正常返回后
  - `@AfterThrowing`：抛异常后
  - `@Around`：环绕通知（最强大，可以控制是否执行目标方法）

### 2.2 AOP 实现原理

- **JDK 动态代理**：目标类实现了接口 → 基于 Proxy 类 + InvocationHandler
- **CGLIB 代理**：目标类没有接口 → 基于字节码生成子类

Spring Boot 2.x 默认使用 CGLIB（`spring.aop.proxy-target-class=true`）

### 2.3 你项目中的 AOP 应用

**定时任务监控切面：**
```java
@Aspect
@Component
public class ScheduledTaskAspect {
    @Around("@annotation(monitoredTask)")
    public Object monitorTask(ProceedingJoinPoint joinPoint,
                             MonitoredScheduledTask monitoredTask) throws Throwable {
        // 前置：创建日志记录、设置 ThreadLocal 上下文
        // 执行：joinPoint.proceed()
        // 后置：更新日志状态、记录耗时、清理上下文
    }
}
```

**Feign 失败补偿切面：**
```java
@Aspect
@Component
public class FeignFallbackAspect {
    @Around("@annotation(FallbackToDatabase)")
    public Object around(ProceedingJoinPoint point) {
        try {
            return point.proceed();  // 正常调用
        } catch (Exception e) {
            // 序列化参数并落库，等待重试
        }
    }
}
```

---

## 三、Spring Boot

### 3.1 自动装配原理

1. `@SpringBootApplication` 包含 `@EnableAutoConfiguration`
2. `@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 加载候选配置
3. `AutoConfigurationImportSelector` 读取自动配置候选：
   - Spring Boot 2.x：`META-INF/spring.factories`
   - Spring Boot 3.x+：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
4. 根据 `@ConditionalOnXxx` 条件注解过滤，满足条件的配置类生效

**你项目中的应用：**
定时任务 Starter 就是一个自动装配的 Spring Boot Starter：
```java
@Configuration
@ComponentScan("com.ysh.pms")
@MapperScan("com.ysh.pms.mapper")
public class PmsAutoConfiguration {
    // 引入这个 Starter 依赖后，自动扫描组件和 Mapper
}
```
如果项目是 Spring Boot 2.x，就在 `META-INF/spring.factories` 注册；如果是 Spring Boot 3.x+，则放到 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。

### 3.2 Spring Boot Starter 开发步骤

1. 创建 `xxx-spring-boot-starter` 模块
2. 编写自动配置类（`@Configuration` + `@ConditionalOnXxx`）
3. 按 Spring Boot 版本注册自动配置类：
   - 2.x：`META-INF/spring.factories`
   - 3.x+：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
4. 其他项目引入依赖即可自动生效

### 3.3 常用注解

| 注解 | 作用 |
|------|------|
| `@SpringBootApplication` | 启动类（= @Configuration + @EnableAutoConfiguration + @ComponentScan） |
| `@RestController` | = @Controller + @ResponseBody |
| `@RequestMapping` | URL 映射 |
| `@Value` | 注入配置值 |
| `@ConfigurationProperties` | 批量绑定配置 |
| `@ConditionalOnProperty` | 条件装配 |
| `@Async` | 异步执行 |
| `@Scheduled` | 定时任务 |

---

## 四、Spring Cloud Alibaba

### 4.1 Nacos

**服务注册与发现：**
- 服务启动时向 Nacos 注册（实例 IP + 端口）
- 消费者从 Nacos 获取服务实例列表
- 支持 AP 和 CP 两种模式：
  - 临时实例（默认）：AP 模式，客户端心跳维持，适合微服务
  - 永久实例：CP 模式，Raft 协议，适合基础组件

**配置中心：**
- 支持配置的动态刷新（`@RefreshScope`）
- 长轮询机制：客户端发起 30 秒长轮询，配置变更时立即返回
- 配置的 Namespace → Group → DataId 三级隔离

**你的项目配置：**
```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.0.230:8848
      config:
        server-addr: 192.168.0.230:8848
        file-extension: yaml
```

### 4.2 OpenFeign

**声明式 HTTP 客户端，用接口注解方式调用远程服务。**

**原理：**
1. `@FeignClient` 接口通过动态代理生成实现类
2. 方法调用时，解析注解构建 HTTP 请求
3. 通过负载均衡器选择服务实例
4. 发送 HTTP 请求并反序列化响应

**你项目中的用法：**
```java
@FeignClient(contextId = "orderGenerateFeignClient",
             name = "strategy-analysis-handler", path = "/")
public interface OrderGenerateFeignClient {
    @PostMapping("/order/generate")
    Result generateOrder(@RequestBody OrderRequest request);
}
```

**contextId 的作用**：当多个 FeignClient 指向同一个服务（name 相同）时，用 contextId 区分 Bean。

### 4.3 Gateway / Sentinel（了解即可）

- **Gateway**：API 网关，路由、鉴权、限流
- **Sentinel**：流量控制、熔断降级、系统保护

---

## 五、MyBatis

### 5.1 #{} 和 ${}

- `#{}`：预编译参数（PreparedStatement），防 SQL 注入，推荐使用
- `${}`：字符串拼接，有 SQL 注入风险，只在动态表名/列名时使用

### 5.2 一级缓存和二级缓存

| 特性 | 一级缓存 | 二级缓存 |
|------|---------|---------|
| 作用域 | SqlSession（默认开启） | namespace / Mapper |
| 生命周期 | Session 关闭失效 | 应用级别 |
| 失效条件 | 增删改操作 | 增删改操作 |
| 问题 | 多 SqlSession 间不共享 | 多表查询时可能脏读 |

**实际项目中一般不用二级缓存**，用 Redis 做缓存更可控。

### 5.3 MyBatis 插件机制

- 基于 JDK 动态代理，拦截 Executor、StatementHandler、ParameterHandler、ResultSetHandler 四大对象
- 常用插件：分页插件（PageHelper）、性能分析、SQL 打印

**你项目中的应用：**
- `@AutoCount` 自动分页注解 + AOP 实现自动 count 查询
- MyBatis-Plus 的分页插件

### 5.4 MyBatis-Plus 常用功能

- 通用 CRUD（BaseMapper）
- 条件构造器（QueryWrapper / LambdaQueryWrapper）
- 自动填充（createTime、updateTime）
- 逻辑删除
- 分页插件

---

## 六、Spring 事务

### 6.1 事务传播行为

| 传播行为 | 描述 |
|---------|------|
| REQUIRED（默认） | 当前有事务就加入，没有就新建 |
| REQUIRES_NEW | 挂起当前事务，新建事务 |
| NESTED | 嵌套事务（SavePoint），外层回滚内层也回滚 |
| SUPPORTS | 有事务就加入，没有就非事务执行 |
| NOT_SUPPORTED | 非事务执行，挂起当前事务 |
| MANDATORY | 必须在事务中调用，否则报错 |
| NEVER | 非事务执行，有事务就报错 |

### 6.2 @Transactional 失效场景

1. **标准代理模式下非 public 方法通常不要依赖事务生效**；最稳妥的是标在 `public` 方法上
2. **同类方法内部调用** → 绕过了代理，直接调用 this.method()
3. **异常被 catch 了** → Spring 检测不到异常，不回滚
4. **抛出非 RuntimeException** → 默认只回滚 RuntimeException（可配置 rollbackFor）
5. **数据库不支持事务** → 比如 MyISAM 引擎

### 6.3 你项目中的事务使用

```java
@Override
@DS("mysqldb")      // 指定数据源
@Transactional()    // 开启事务
public void run(String... args) throws Exception {
    // TaskStateCleanupRunner 中清理残留日志
    // 批量更新日志状态为 SYSTEM_SHUTTING_DOWN
}
```
