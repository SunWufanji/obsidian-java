# Spring Boot

> 本页吸收“Spring 刷题题库”中 Spring Boot 相关题目。与 Spring 核心共用的事务、MVC、Security 等内容通过 [[01-Spring核心与Spring MVC]] 回顾，避免重复。

## 1. Spring Boot 是什么？与 Spring 有什么区别？

### 先用大白话理解

Spring 像一大套积木：IoC、AOP、MVC、事务、数据访问等能力都有，但传统项目需要自己挑选、组合和配置。Spring Boot 像把常用积木预先装成“开箱能跑的套装”：它仍然使用 Spring 的能力，只是帮你处理依赖版本、默认配置和启动方式。

所以 Spring Boot **不是替代 Spring，也不是脱离 Spring 的新框架**；它是让你更省配置、更快开发 Spring 应用的方式。

Spring Boot 不是替代 Spring，而是建立在 Spring 之上的快速开发方案。它通过自动配置、Starter、内嵌服务器和外部化配置减少样板代码。Spring 更像能力集合，Boot 负责把常见组合按约定装配好。

它的直接收益是：依赖版本由 BOM/Starter 协调、Web 应用可打成可执行 Jar 并使用内嵌 Tomcat/Jetty/Undertow、默认配置可运行但仍可覆盖，同时自带健康检查和指标等生产能力。所谓“约定优于配置”不是不能配置，而是先给合理默认值。

### 传统 Spring 与 Spring Boot 对比

| 对比点 | 传统 Spring 应用 | Spring Boot |
| --- | --- | --- |
| 核心能力 | 提供 IoC、AOP、MVC、事务等模块 | 直接复用这些 Spring 能力 |
| 配置方式 | 常需手动组合依赖、XML/Java 配置、Servlet 容器配置 | 自动配置 + Starter + 外部化配置 |
| 依赖管理 | 自己处理多个依赖版本兼容 | Starter/BOM 管理常用依赖组合和版本 |
| Web 运行方式 | 常打 WAR 部署到外部 Tomcat 等容器 | 常打可执行 JAR，内嵌 Tomcat/Jetty/Undertow |
| 默认配置 | 需要更多显式配置 | 约定优于配置，可按需覆盖 |
| 生产能力 | 自行整合监控、健康检查等 | Actuator 等能力更容易接入 |

### 面试易错点

1. Spring Boot 不是“只能做微服务”；单体应用也非常适合。
2. Spring Boot 不是“没有配置”；它只是提供默认配置，仍可通过 `application.yml`、自定义 Bean 等覆盖。
3. 内嵌服务器不等于不能部署到外部容器，只是可执行 JAR 往往更方便。

### 看到“Spring Boot 如何使用某工具”时怎么理解

很多题目里的能力并不是 Boot 独有：Spring MVC、事务、WebSocket、国际化属于 Spring 生态；JPA、RabbitMQ、Elasticsearch、JMS 是外部规范或中间件。**Boot 的作用通常是把依赖、默认 Bean 和常见配置先准备好，不是重新实现这些能力。**

| 能力 | 原本是谁提供能力 | Spring Boot 简化了什么 |
| --- | --- | --- |
| 数据访问 | Spring JDBC/JPA/MyBatis 等集成 | Starter、`DataSource`、事务管理器自动配置 |
| WebSocket | Spring WebSocket | Starter、常见容器配置和属性支持 |
| RabbitMQ / JMS | 消息中间件与 Spring Messaging | Starter、连接工厂、Template、监听器容器配置 |
| Elasticsearch | Elasticsearch + Spring Data 集成 | Starter、客户端连接和 Repository 自动装配 |
| 国际化 | Spring `MessageSource` / MVC | 默认消息资源配置和 Web Locale 支持 |
| 文件上传 | Spring MVC Multipart | Web Starter、上传大小等属性配置 |

判断口诀：**Spring 解决“能不能做”，Boot 解决“能不能少配点、快点跑起来”。** 但当需求复杂时，Boot 仍需要你显式配置安全、连接池、消息可靠性、索引、路由等关键细节。

## 2. Spring Boot 自动配置原理是什么？
![[Pasted image 20260712145953.png]]
启动类上的 `@SpringBootApplication` 包含 `@EnableAutoConfiguration`。Boot 会导入候选自动配置类，再根据类路径、已有 Bean、配置属性等 `@Conditional` 条件决定是否生效；生效后向容器注册默认 Bean，开发者配置或自定义 Bean 可覆盖默认行为。

排查自动配置时，按“是否引入依赖 → 条件是否满足 → 是否已有同类 Bean → 属性是否正确”检查；必要时查看 Condition Evaluation Report。常用条件包括 `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty`、`@ConditionalOnWebApplication`。

⚠️ 自动配置 7不是“无条件创建一切”，而是“满足条件才按约定配置”。

### 自动配置顺序与版本提示

自动配置之间有先后依赖时，可通过 `@AutoConfigureBefore`、`@AutoConfigureAfter` 或排序相关机制声明顺序；但应用代码不应依赖某个内部自动配置类的具体加载顺序。旧资料常说自动配置写在 `spring.factories`，Boot 3 主要使用 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，核心仍是“候选配置 + 条件匹配 + 属性覆盖”。

### 通俗理解与实践

自动配置像“看到厨房里有咖啡机，就自动准备咖啡相关工具”。实际开发中你引入 `starter-data-jpa` 并填写数据库地址，Boot 才会自动创建数据源、JPA 相关 Bean；如果你自己声明同类 Bean，Boot 通常会退让，使用你的实现。

## 3. @SpringBootApplication 包含哪些注解？

它组合了 `@SpringBootConfiguration`（配置类）、`@EnableAutoConfiguration`（启用自动配置）和 `@ComponentScan`（组件扫描）。扫描默认从启动类所在包及其子包开始，所以启动类一般放在项目根包。

需要排除某项自动配置时，可在 `@SpringBootApplication(exclude = XxxAutoConfiguration.class)` 指定，但应先确认是否只是缺少正确配置；盲目排除会造成后续依赖 Bean 缺失。

## 4. Starter 是什么？工作原理是什么？
![[Pasted image 20260712174725.png]]
Starter 是一组场景依赖描述，例如 `spring-boot-starter-web`。它通过依赖传递引入常用库，并配套自动配置：Boot 启动时读取自动配置声明，满足条件后装配对应组件。它解决的是依赖版本组合和初始化配置繁琐的问题。

原理上，Starter 负责把一组经过兼容性管理的传递依赖带进来，自动配置负责按条件创建 Bean。旧版 Boot 常通过 `META-INF/spring.factories` 声明自动配置；Boot 3 主要使用 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。两者的核心都是“发现候选配置 → 条件匹配 → 注入容器”，不要只背某一个文件名。

### 通俗理解与实践

Starter 像一份“功能套餐”：想做 Web 就引 `spring-boot-starter-web`，不用自己逐个找 MVC、JSON、内嵌服务器的依赖。实际项目选 Starter 时要只引需要的套餐，避免为了方便引入大量无用依赖和冲突组件。

## 5. 如何自定义 Starter？

通常拆分为自动配置模块和 Starter 依赖模块：在自动配置中用 `@Configuration`、`@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@EnableConfigurationProperties` 定义可选 Bean；在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 声明自动配置类；Starter 模块只负责聚合依赖。重点是“条件化、可覆盖、配置可绑定”。

## 6. application.yml、properties 与配置优先级如何理解？

两种格式都能配置；YAML 层级更清晰，properties 更直接。配置可来自默认文件、Profile 文件、环境变量、命令行参数、外部文件等，通常越靠近运行环境、显式程度越高的配置优先级越高。不要死记每一项顺序，遇到覆盖问题用 Actuator 的环境端点或启动日志确认。

同一键被多处定义时，命令行和环境变量通常可覆盖打包内配置；精确优先级会随 Boot 版本和加载方式变化。团队实践是：仓库提交默认值和非敏感配置，环境差异与密钥交给部署平台或配置中心。

## 7. @Value 与 @ConfigurationProperties 如何选择？

少量、单个值可用 `@Value`；一组有层级的业务配置优先 `@ConfigurationProperties`，它支持类型绑定、校验、元数据提示，维护性更好。敏感配置不要写入仓库，应通过环境变量、配置中心或密钥管理注入。

### 通俗理解与实践

配置管理像给同一程序准备不同环境的“说明书”。本地开发连测试库，生产连生产库；代码不变，只通过 `application-dev.yml`、环境变量或启动参数换连接地址和开关。敏感密码不要写进 Git，应由部署平台/密钥系统注入。

## 8. Profile 的作用是什么？

![[Pasted image 20260712174634.png|491]]
![[Pasted image 20260712155820.png|536]]
Profile 用于区分开发、测试、生产等环境配置与 Bean。通过 `spring.profiles.active` 激活，配置文件可使用 `application-dev.yml` 等约定。它解决的是“同一份代码在不同环境连接不同资源、开关不同功能”的问题。

可通过配置文件、环境变量、JVM 参数或命令行激活；`@Profile("prod")` 可让某个 Bean 仅在指定环境注册。生产环境最好由部署平台传入激活项，避免把 `spring.profiles.active=dev` 固化在通用包内。

### 通俗理解与实践

Profile 就像启动时选择“开发模式、测试模式、生产模式”。实际项目最常见用途是切换数据库、日志级别、第三方地址和某些功能开关；不要把业务逻辑分叉成大量 Profile，否则会难以测试和维护。

## 9. CommandLineRunner 与 ApplicationRunner 有什么区别？

### 先用大白话理解

它们都是“应用已经启动好之后，马上执行一次的启动钩子”。

例如启动后初始化演示数据、检查关键配置、打印启动信息、执行一次性数据迁移，都可以放在 Runner 里。它不是定时任务，也不是后台常驻线程；任务执行太久会让应用迟迟无法真正就绪。

二者都在 Spring Boot 启动完成后执行初始化逻辑。`CommandLineRunner` 接收字符串数组，`ApplicationRunner` 接收解析后的 `ApplicationArguments`。多个 Runner 可用 `@Order` 排序。⚠️ 不要把长期阻塞任务放进 Runner，否则会影响应用就绪。

Runner 适合初始化演示数据、校验启动依赖或执行一次性迁移，不适合承担常驻消费循环。带 `--key=value` 这类选项时，`ApplicationRunner` 能通过 `getOptionNames()`、`getOptionValues()` 更方便地区分选项参数和普通参数。

### 区别与选择

| 项目 | `CommandLineRunner` | `ApplicationRunner` |
| --- | --- | --- |
| `run` 参数 | `String... args` | `ApplicationArguments args` |
| 参数解析 | 自己解析字符串 | 可区分选项参数、非选项参数 |
| 适合 | 参数很简单 | 需要读取 `--key=value` 等复杂启动参数 |

例如命令行为 `--region=cn --dry-run file1` 时，`ApplicationArguments` 可直接取得 `region` 的值和非选项参数 `file1`。多个 Runner 用 `@Order` 决定先后；若初始化失败应让启动失败或明确报警，不能悄悄吞掉异常。

## 10. Spring Boot 如何统一处理异常？
![[Pasted image 20260712174826.png|611]]
### 先用大白话理解

异常处理就是给接口准备统一的“故障出口”。

用户下单失败，可能是参数不合法、库存不足、没有权限、数据库异常。若每个 Controller 自己 `try/catch`，有的返回字符串、有的返回 500、有的把堆栈返回给用户，前端会很难处理。Spring Boot 推荐把异常集中交给一个全局处理器：对用户返回统一格式，对开发者记录完整日志。

推荐用 `@RestControllerAdvice` 和 `@ExceptionHandler` 统一返回错误码、提示语、请求标识等。参数校验异常、业务异常、系统异常分别处理；生产环境不要返回堆栈和内部实现细节。错误响应格式要稳定，便于前端和监控系统处理。

需要复用 Spring MVC 的默认异常处理时，可继承 `ResponseEntityExceptionHandler`；校验场景可读取 `BindingResult`，或统一处理 `MethodArgumentNotValidException`。`@ResponseStatus` 适合为明确的异常绑定 HTTP 状态，但复杂接口仍应统一错误响应体，避免前端面对多套格式。

### 截图五步整理成完整处理链

1. **自定义业务异常**：业务规则不满足时抛出例如 `BusinessException`，携带稳定业务错误码；不要用 `NullPointerException` 表达“库存不足”。
2. **全局处理器**：使用 `@RestControllerAdvice` 声明统一异常处理类，它会作用于 Controller 层。
3. **按类型分流**：用多个 `@ExceptionHandler` 分别处理业务异常、参数校验异常、权限异常、兜底系统异常。
4. **构造 HTTP 响应**：用 `ResponseEntity` 或统一响应对象返回状态码、错误码、用户提示、请求 ID 和时间；不要返回数据库异常或完整堆栈。
5. **记录日志与监控**：服务端日志记录异常堆栈、请求路径、traceId、关键业务参数（注意脱敏）；严重系统异常再触发告警。

```java
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException.class)
    ResponseEntity<ApiResponse<Void>> handleBusiness(BusinessException e) {
        return ResponseEntity.badRequest()
            .body(ApiResponse.fail(e.getCode(), e.getMessage()));
    }
}
```

**推荐分层：**Controller 接收请求和校验 → Service 抛业务异常 → Advice 统一翻译响应。异常处理器不应承担正常业务分支，也不要为了“接口永不报错”把所有异常都返回 200。

### 错误页与兜底错误处理

前后端分离 API 通常通过 `@RestControllerAdvice` 返回 JSON 错误体；传统服务端渲染应用可在静态/模板错误目录提供 `404.html`、`5xx.html` 等错误页。需要完全接管默认错误响应时可以定制 Boot 的错误处理入口，但优先把业务异常和 MVC 异常放在 Advice 中统一处理，避免无必要地覆盖框架默认错误机制。

### 通俗理解与实践

统一异常处理像接口的“统一客服台”：库存不足返回业务错误，参数格式错返回校验错误，系统故障返回通用提示。实际代码中 Controller 不应到处 `try/catch`，而是让 Service 抛语义异常，再由 Advice 统一变成前端约定的 JSON。

## 11. Spring Boot 的数据访问与事务如何配置以及事务管理？

### 先用大白话理解
![[Pasted image 20260712171423.png|505]]
Spring Boot 管事务的目标是：你告诉它“这几个数据库操作必须绑在一起”，它帮你准备事务管理器并在方法前后执行开启、提交或回滚。

例如创建订单时要“扣库存、写订单、扣优惠券”。其中一步失败，其他步骤也要撤销；在方法上加 `@Transactional`，Spring 便会通过代理把这组操作包进同一个事务。

Boot 可依据依赖和配置自动创建 `DataSource`、JPA、MyBatis 等相关 Bean。业务事务仍用 `@Transactional`，原理与注意点见 [[01-Spring核心与Spring MVC#12. Spring 的声明式事务原理是什么？]]。数据源、连接池、SQL 日志和迁移工具应按环境配置，不要把生产账号密码写进 YAML。

### 截图四个关键点

1. **`@EnableTransactionManagement`**：启用注解驱动事务的开关。Spring Boot 在常见数据访问场景中通常已通过自动配置准备好相关能力，日常项目往往不需要手写；自定义配置或非 Boot 项目才更可能显式使用。
2. **`@Transactional`**：标在类或方法上声明事务，可配置 `propagation`、`isolation`、`timeout`、`readOnly`、`rollbackFor` 等属性。方法级配置通常优先级更高。
3. **事务管理器**：JDBC 场景常用 `DataSourceTransactionManager`，JPA 场景常用 `JpaTransactionManager`。Boot 会根据当前依赖和 Bean 自动配置合适实现；多数据源时必须明确指定。
4. **回滚规则**：默认通常对运行时异常和 `Error` 回滚；受检异常需要用 `rollbackFor` 明确指定。异常若被捕获后不再抛出，事务可能会正常提交。

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    rollbackFor = Exception.class
)
public void createOrder(CreateOrderCommand cmd) {
    inventoryService.deduct(cmd);
    orderRepository.save(...);
}
```

⚠️ `@Transactional` 依赖 Spring 代理：同类内部调用、方法不经过 Spring 管理、多个事务管理器未指定、异步线程切换等场景都可能让事务不按预期生效。详细传播与失效原因见 [[01-Spring核心与Spring MVC#12. Spring 的声明式事务原理是什么？]]。

数据访问层要明确边界：JPA 适合领域模型和常规 CRUD，MyBatis 适合 SQL 需要精细掌控的场景，二者可以共存但应避免同一业务随意混用。数据库迁移推荐采用 Flyway 或 Liquibase 等可版本化工具，而不是靠手工执行 SQL。

### 数据访问配置追问
![[Pasted image 20260712172836.png]]
连接池要配置最大连接数、超时与校验策略，并与数据库容量匹配；生产中应开启必要 SQL/慢查询观测而不是长期打印所有 SQL。事务方法要通过 Spring 代理调用，数据源和事务管理器必须对应，否则可能出现“代码写了事务但没有回滚”。

## 12. JPA 与 Hibernate 是什么关系？如何优化 JPA？
![[Pasted image 20260712174132.png]]
### 先用大白话理解

JPA 和 Hibernate 不是竞争关系，更像“交通规则”和“具体汽车”。

- **JPA**：规定 Java 对象怎样映射表、怎样保存查询数据的一套标准接口。
- **Hibernate**：按这套标准真正把对象操作翻译成 SQL 并执行的常见实现。

你项目中写 `@Entity`、`JpaRepository`、JPQL 等 JPA 风格代码，底层通常由 Hibernate 完成实际工作。这样未来理论上可以替换实现；但若使用 Hibernate 独有 API，就会和 Hibernate 绑定更深。

JPA 是 Java 持久化规范，Hibernate 是常用实现之一。优化重点不是盲目调注解：避免 N+1 查询（抓取策略、`join fetch`、批量查询）、分页时避免不必要的关联抓取、为高频条件建索引、必要时改用原生 SQL 或 MyBatis。

### 截图中的四点对比
![[Pasted image 20260712175315.png]]

| 维度 | JPA | Hibernate |
| --- | --- | --- |
| 身份 | ORM 持久化规范/API | JPA 的常见实现框架 |
| 抽象层次 | 标准化接口，便于屏蔽实现差异 | 功能更丰富，也有自己的扩展 API |
| Spring Boot 使用 | 通常通过 Spring Data JPA 编程 | 引入 JPA Starter 后常被自动作为 provider 使用 |
| 查询语言 | JPQL：面向实体和字段的标准查询语言 | HQL：与 JPQL 很相近，并有 Hibernate 扩展 |

### Spring Boot 里实际发生什么

引入 `spring-boot-starter-data-jpa` → Boot 根据依赖自动配置数据源、`EntityManagerFactory`、事务管理器等 → Hibernate 扫描实体并作为 JPA provider 工作 → 你通过 `JpaRepository` 或 `EntityManager` 操作实体。

⚠️ JPQL/HQL 都是面向**实体名和属性名**，不是直接写表名和列名；复杂报表、性能敏感 SQL 不必强行用方法名或 JPQL，可选择原生 SQL 或 MyBatis。

### JPA 性能检查

实体关联默认策略、序列化时访问懒加载字段、循环引用、分页加 `fetch join` 都可能产生意外 SQL 或错误结果。优化的顺序应是：看 SQL 与执行计划 → 修正查询与索引 → 控制抓取范围 → 再考虑缓存，而不是一开始就调大连接池。

### 通俗理解与实践

JPA 像“按 Java 对象操作数据库”，Hibernate 是实际翻译并执行 SQL 的引擎。实际代码里简单 CRUD 用 Repository 很省事；一旦发现 N+1、慢 SQL 或复杂报表，就要看真实 SQL，必要时改 `join fetch`、原生 SQL 或 MyBatis。

## 13. Spring Cache 缓存如何使用？
![[Pasted image 20260712172052.png|609]]
通过 `@EnableCaching` 启用缓存，常用 `@Cacheable` 查缓存、`@CachePut` 更新缓存、`@CacheEvict` 删除缓存。缓存键要能区分数据版本和查询条件；更新数据库后要处理缓存一致性。单机可用 Caffeine，多实例通常使用 Redis 等共享缓存。

缓存实现可按场景选择：`ConcurrentMapCache` 适合演示，Caffeine 适合单机本地高性能缓存，Redis 适合多实例共享缓存；Ehcache 也可用于本地缓存。缓存注解中可通过 `key` 指定键规则、通过 `condition` / `unless` 决定何时缓存；但规则应尽量简单可追踪，避免同一份数据出现多个难以失效的 key。

缓存注解同样依赖代理，因此同类内部调用可能不生效。高并发热点还要额外防缓存穿透、击穿和雪崩；缓存不是数据库事务的一部分，更新策略要明确“先更新库还是先删缓存”以及失败补偿方案。

## 14. 如何实现异步任务与定时任务？

### 先用大白话理解异步

异步就是“先把用户眼前最重要的事办完，其他慢事放到后台”。

例如用户注册：保存用户信息必须先成功；但欢迎邮件、注册送券通知、统计日志不必让用户盯着页面等几秒。主线程保存完用户后立刻返回“注册成功”，后台线程再慢慢发邮件，这就是异步。

**适合异步：**发邮件/短信、生成图片缩略图、导出文件、非核心通知、数据统计、调用耗时但可重试的第三方服务。

**不适合异步：**调用方必须立刻拿到结果的操作；必须和主数据库事务强一致提交/回滚的核心操作；无法接受任务丢失却又没有可靠队列保障的操作。

### 截图补充：定时与异步执行器

定时任务用 `@EnableScheduling + @Scheduled`，可用固定间隔、固定频率或 cron 表达式定义；需要更精细控制时配置 `TaskScheduler`。异步方法用 `@EnableAsync + @Async`，并显式配置 `TaskExecutor` 的线程数、队列和拒绝策略。多实例部署时同一个定时任务会在每台机器都执行，必须用分布式锁、调度平台或任务分片避免重复。

异步使用 `@EnableAsync + @Async`，定时任务使用 `@EnableScheduling + @Scheduled`。异步线程池必须显式配置容量、队列、拒绝策略和异常处理；多实例定时任务需要分布式锁或调度平台，否则会重复执行。

`@Async` 返回 `void` 时异常不会自动抛回调用方，应配置 `AsyncUncaughtExceptionHandler` 或改用 `Future` / `CompletableFuture` 获取结果。定时任务也要记录执行时长、幂等键和失败告警，避免静默失败。

```java
@Service
class NoticeService {
    @Async
    public void sendWelcomeEmail(String email) {
        mailClient.send(email, "欢迎注册"); // 后台线程执行
    }
}

// 注册接口：保存成功后不等待邮件发送
userRepository.save(user);
noticeService.sendWelcomeEmail(user.getEmail());
return "注册成功";
```

### 代码里真正要注意什么

1. `@Async` 方法必须通过 Spring 代理调用；同一个类里自己调用自己通常不会异步。
2. 默认线程池不一定适合生产，应配置 `TaskExecutor` 的核心线程、最大线程、队列和拒绝策略。
3. 主任务提交数据库事务后再异步发通知更稳妥；必要时使用 `@TransactionalEventListener` 或消息队列，避免事务回滚了却已经发出“成功通知”。
4. 异步不是无限并发。线程池满了、下游邮件服务慢了都可能积压，必须监控队列长度、失败数和耗时。

### 缓存一致性追问

`@Cacheable` 命中缓存时目标方法不会执行；`@CachePut` 总会执行并写缓存；`@CacheEvict` 用于删除。多实例场景还要考虑本地缓存失效广播；热点 key 需要考虑过期时间随机化、互斥重建或逻辑过期等方案。

### 通俗理解与实践

异步像把“用户不必等”的任务交给后台同事：注册后发邮件、下单后发通知、导出文件。实际使用必须配置线程池和失败监控；核心扣款、库存等强一致操作不能简单丢给 `@Async`。

## 15. WebSocket 在 Spring Boot 中怎么用？

### 截图补充：原生 WebSocket 的最小组成

引入 `spring-boot-starter-websocket` 后，可通过 `WebSocketConfigurer` 注册端点、通过 `WebSocketHandler` 接收和发送消息；也可使用 STOMP 走更高层的订阅/发布模型。无论哪种方式，连接建立（握手）和消息处理都要做鉴权，避免任何人随意连接或订阅敏感频道。

WebSocket 适合实时通知、聊天和行情推送等服务端主动推送场景。Boot 可配置 WebSocket 端点，或使用 STOMP；生产中还要考虑鉴权、连接数、心跳、消息顺序、断线重连与多节点消息广播。

### WebSocket 生产注意点

长连接并不等于无限连接：需要限制单用户连接数、消息大小和空闲时间，使用心跳清理失效连接。多节点部署时连接在不同实例上，广播消息需要 Redis、MQ 等共享通道；鉴权应在握手或消息协议层完成。

## 16. Spring Boot 如何处理文件上传、静态资源和国际化？
![[Pasted image 20260712173052.png|582]]
### 截图补充：上传与国际化配置
![[Pasted image 20260712173112.png|557]]
上传接口通常使用 `@RequestParam MultipartFile file` 接收文件；通过 `spring.servlet.multipart.max-file-size`、`spring.servlet.multipart.max-request-size` 限制大小，并统一处理超限和类型不支持异常。国际化通常在 `resources` 下放 `messages_zh_CN.properties`、`messages_en.properties` 等资源文件，由 `MessageSource` 与 `LocaleResolver` 根据请求语言选择对应文案。

Boot 可通过 `spring.messages.basename=messages` 指定消息资源的基础名；例如基名为 `messages` 时，会按当前 Locale 自动选择 `messages_zh_CN.properties` 或 `messages_en.properties`。真正项目中除了页面提示语，还应统一维护错误码文案、日期金额格式与时区，避免只翻译页面文字却让接口错误信息仍是单一语言。

上传使用 `MultipartFile`，要限制大小、校验类型/内容、使用随机文件名并避免直接暴露存储路径；静态资源可由默认目录或资源映射提供；国际化通过 `MessageSource` 按 Locale 返回不同文本。⚠️ 文件上传是安全入口，必须防路径穿越和伪造文件类型。

### 文件与资源安全

上传文件应存到对象存储或受控目录，下载通过业务权限校验后以受控响应输出；禁止根据用户输入直接拼接文件路径。静态资源应设置缓存策略，国际化资源文件应与业务错误码统一维护。

## 17. Actuator 是什么？生产环境如何安全使用？
![[Pasted image 20260712175226.png]]
### 先用大白话理解

Actuator 就像给应用安装了一套“体检仪表盘”。用户访问 `/orders` 是在使用业务功能；运维、监控系统则通过 Actuator 了解“服务是否存活、数据库是否连得上、请求是否变慢、线程是否堆积”。它不直接解决业务问题，但能让你知道业务系统出了什么问题。

Actuator 提供健康检查（`health`）、指标（`metrics`）、环境（`env`）、动态日志级别（`loggers`）、线程信息（`threaddump`）、应用信息（`info`）、审计事件等运维端点；HTTP 请求跟踪在新版本中通常以 `httpexchanges` 等端点或观测能力呈现。它可对接监控系统。生产只暴露必要端点，使用独立管理端口或网络隔离并加认证授权；`env`、配置和线程信息等敏感端点不能随意公开。

### 截图中的端点分别看什么

1. ** `health` **：健康检查，例如应用、数据库、Redis 是否可用；常用于 Kubernetes/负载均衡决定是否继续把流量发给实例。
2. ** `metrics` **：各种数值指标，例如 JVM 内存、GC、线程数、HTTP 请求耗时；通常交给 Prometheus、Grafana 等监控系统展示和告警。
3. ** `env` **：当前环境属性与配置来源，排查“为什么配置没生效”很有用，但也最可能泄露账号、密钥等敏感信息。
4. ** `loggers` **：运行时查看或临时调整日志级别，排查问题后应及时恢复，不能长期把全局日志调成 DEBUG。
5. ** `threaddump` **：线程快照，用于排查死锁、线程池耗尽、请求卡住等问题。
6. **HTTP Trace / `httpexchanges` **：近期 HTTP 交换记录；新版本实现和默认暴露策略可能不同，不能把请求头、Token、用户数据直接暴露出去。
7. ** `info` **：展示应用版本、构建号、Git 提交号等自定义信息。
8. ** `auditevents` **：记录登录成功/失败等审计事件；是否有数据取决于实际安全与审计配置。

### 生产使用原则

只暴露真正需要的端点，例如健康检查和指标；管理端点应使用独立端口、内网访问或认证授权保护。不要把 `env`、heapdump、线程栈、配置详情直接开放到公网，否则它们可能成为攻击者收集系统信息的入口。

### 通俗理解与实践

Actuator 像应用的体检仪表盘。实际部署中健康检查给 Kubernetes/负载均衡判断是否接流量，指标交给 Prometheus/Grafana 告警；`env`、线程栈等敏感端点只开放给内网和运维人员。

## 18. 如何测试和保护 Spring Boot 应用？

测试分层进行：单元测试关注业务类，`@WebMvcTest` 测 MVC 层，`@DataJpaTest` 测持久层，`@SpringBootTest` 做必要集成测试。安全方面至少做到依赖漏洞扫描、及时升级、HTTPS、输入校验、认证授权、CSRF/XSS 防护、日志脱敏和限流；不能只依赖框架默认配置。

依赖漏洞扫描可使用 Snyk、Dependabot、OWASP Dependency-Check 等任一适配团队流程的工具；工具告警仍需结合可达性和版本修复策略判断，不能只“扫描一次”就结束。

### 监控落地

健康检查至少区分存活（liveness）和就绪（readiness）：前者决定是否重启进程，后者决定是否接流量。指标应输出到 Prometheus、OpenTelemetry 等观测体系并建立告警；不要把敏感的 `env`、heapdump、线程栈端点暴露给公网。

## 19. Spring、Spring Boot、Spring Cloud 的关系是什么？

Spring 是基础生态；Spring Boot 在其上简化单个应用的创建、配置和运行；Spring Cloud 在 Boot 基础上提供微服务治理能力，如注册发现、配置中心、网关、熔断和链路追踪。可记为：Spring 提供底座，Boot 提升开发效率，Cloud 解决分布式协作。

## 20. 如何自定义 Spring Boot 启动 Banner？

### 先用大白话理解

Banner 就是 Spring Boot 刚启动时打印在控制台上的“开机欢迎页”。它不会影响接口、数据库、事务或性能；主要作用是让日志里一眼看出当前启动的是哪个项目、哪个版本、哪个环境。

最简单的做法是在 `src/main/resources` 下创建 `banner.txt`：

```text
  My Order Service
  version: 1.0.0
  environment: dev
```

应用启动时 Spring Boot 会自动读取并打印它。也可以放 ASCII 字符画，但生产环境更建议展示简洁、有用的信息，而不是太长的图案淹没启动日志。

在 `src/main/resources` 下放置 `banner.txt`，或通过 `SpringApplication#setBanner` 设置自定义 Banner；可用 `spring.main.banner-mode=off` 关闭。它只影响启动展示，不参与业务能力，面试时说明“可通过资源文件或代码自定义”即可。

也可用 `spring.banner.location` 指定 Banner 文件位置。旧资料中的 `spring.banner.enabled=false` 在新版本中更推荐改为 `spring.main.banner-mode=off`。

### 截图中的五种操作

1. **创建文件**：默认放在 `src/main/resources/banner.txt`，Spring Boot 会自动发现。
2. **写自定义文字**：可写普通文本、ASCII 字符画、版本号等；不要写敏感信息。
3. **指定位置**：使用 `spring.banner.location=classpath:my-banner.txt` 指向其他 Banner 文件。
4. **代码方式**：实现 `Banner` 接口或调用 `SpringApplication#setBanner(...)`，适合需要根据运行环境动态生成内容的场景。
5. **关闭 Banner**：当前常用 `spring.main.banner-mode=off`；`console` 表示打印到控制台，`log` 表示写到日志系统。

```java
SpringApplication app = new SpringApplication(Application.class);
app.setBanner((environment, sourceClass, out) ->
    out.println("Order Service started"));
app.run(args);
```

⚠️ Banner 只在启动时执行一次。需要持续记录版本、环境和运行状态时，应使用日志、`/actuator/info` 或监控系统，而不是依赖 Banner。

## 21. Spring Boot 如何实现多数据源配置？

### 先用大白话理解

多数据源就是一个应用同时连接多个数据库或多个库实例。例如订单数据在订单库、用户数据在用户库；程序必须明确“这次操作到底用哪个连接、哪个事务管理器”，不能让 Spring 猜。

### 基本配置思路

1. 在配置文件为每个数据源定义独立前缀、URL、账号和连接池参数。
2. 为每个数据源创建 `DataSource` Bean；若其中一个是默认数据源，用 `@Primary` 标明。
3. JDBC 场景为每个库配置对应 `JdbcTemplate`；JPA 场景还需各自的 `EntityManagerFactory` 和 Repository 扫描范围。
4. 每个数据源配置匹配的事务管理器，例如 `DataSourceTransactionManager` 或 `JpaTransactionManager`。
5. 注入或声明事务时使用 `@Qualifier` / `transactionManager` 明确选择。

```java
@Transactional(transactionManager = "orderTransactionManager")
public void createOrder() { /* 操作订单库 */ }
```

⚠️ 两个本地数据源上的操作并不会因为写了一个 `@Transactional` 就自动成为分布式事务。跨库一致性要使用合适的分布式事务/最终一致性方案，或重新划分数据边界。

## 22. Spring Boot 中的 AOP 如何工作？

### 先用大白话理解

Boot 中的 AOP 仍是“给业务方法统一外挂流程”：例如所有 Service 方法记录耗时、事务方法前后控制提交回滚。Boot 的便利在于，引入 `spring-boot-starter-aop` 后，常见 AOP 基础配置会自动准备好。

### 使用步骤

1. 引入 AOP Starter。
2. 用 `@Aspect + @Component` 声明切面。
3. 用 `@Pointcut` 定义哪些方法需要增强。
4. 用 `@Before`、`@AfterReturning`、`@AfterThrowing`、`@Around` 写通知。
5. Spring 通过代理把通知织入目标 Bean 的方法调用。

```java
@Aspect
@Component
class CostAspect {
    @Around("execution(* com.example.service..*(..))")
    Object record(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try { return pjp.proceed(); }
        finally { log.info("cost={}ms", System.currentTimeMillis() - start); }
    }
}
```

⚠️ `@EnableAspectJAutoProxy` 在 Boot 常见 AOP Starter 场景通常不必手写；AOP 仍受代理限制，同类内部调用、`final` 限制等问题见 [[01-Spring核心与Spring MVC#8. 什么是 AOP？核心术语有哪些？]]。

## 23. Spring Boot 如何实现微服务架构以及单体应用的区别？
![[Pasted image 20260712172135.png|649]]
### 先用大白话理解

微服务不是“把一个项目拆成很多小项目”就结束了。拆开后会多出新问题：服务地址怎么找、配置怎么统一、请求怎么路由、一个服务故障会不会拖垮其他服务、调用链如何排查。

Spring Boot 用来快速开发每个独立服务；Spring Cloud/相关生态用来解决服务之间的协作和治理。

### 微服务与单体应用对比

| 维度 | 单体应用 | 微服务 |
| --- | --- | --- |
| 结构 | 所有功能部署在一个应用中 | 按业务边界拆为多个独立服务 |
| 部署与扩容 | 一次部署整个应用，整体扩容 | 服务可独立部署、独立扩容 |
| 开发与运维 | 开始简单，调试和部署成本低 | 需要注册、网关、监控、链路追踪等配套治理 |
| 故障影响 | 一个模块问题可能影响整个应用 | 可做隔离和降级，但网络故障、调用失败会增加 |
| 适用场景 | 团队小、业务稳定、规模不大 | 模块边界清晰，确有独立发布、扩容和多团队协作需求 |

微服务不是“更高级的单体”。单体像一家店把收银、后厨、配送都放在一起，开店快、管理简单；微服务像把这些拆成多家专门店，能各自扩张，却需要配送、调度和统一管理。项目早期通常先做好模块化单体，业务和团队复杂到确有拆分收益时再逐步演进。

### 常见能力

- **服务注册与发现**：Nacos、Consul、Eureka 等让服务实例登记并互相发现。
- **集中配置**：Config/Nacos 等集中管理不同环境配置。
- **网关与路由**：Spring Cloud Gateway 作为统一入口，处理路由、鉴权、限流等。
- **负载均衡与调用容错**：实例选择、超时、重试、熔断、降级。
- **可观测性**：日志、指标、链路追踪帮助定位跨服务问题。

截图里的 Zuul、Hystrix、Sleuth/Zipkin 是常见历史组件；新项目通常优先 Gateway、Resilience4j 和 Micrometer Tracing/Zipkin 等当前生态组合。微服务只有在服务边界、独立部署和团队协作确实需要时才值得引入。

## 24. Spring Boot 应用如何实现安全控制**OAuth 2**？
![[Pasted image 20260712175017.png|594]]
### 先用大白话理解
![[Pasted image 20260712173827.png|566]]
安全控制像接口入口的安检：先确认“你是谁”（认证），再判断“你能做什么”（授权），最后保护传输和输入，防止别人偷看、篡改或攻击。

### Spring Security 的常见落点

1. 引入 `spring-boot-starter-security`。
2. 配置 `SecurityFilterChain` 指定哪些路径公开、哪些必须认证、需要什么权限。
3. 配置认证来源，例如数据库用户、JWT、OAuth2/OIDC。
4. 用 `@PreAuthorize` 等方法级注解补充细粒度权限。
5. 根据前后端架构正确处理 CSRF、CORS、密码哈希、401/403 响应。

⚠️ `WebSecurityConfigurerAdapter` 是旧版写法，当前推荐声明 `SecurityFilterChain` Bean。CSRF 不是“永远关闭”；Cookie/Session 浏览器应用通常需要考虑，纯无状态 Token API 才按风险评估配置。还必须使用 HTTPS、密码加密、输入校验、依赖漏洞治理和日志脱敏。

### OAuth2 / OIDC 集成位置

Boot 中做 OAuth2 通常不是自己从零实现授权服务器，而是配置为 OAuth2 Client（第三方登录）、Resource Server（校验 JWT/Access Token），或接入专门授权服务器。客户端信息、授权服务器地址、回调地址等通过配置文件提供；资源接口再用 Security 规则和方法权限保护。详细协议见 [[01-Spring核心与Spring MVC#21. OAuth2 是什么？]]。

### 通俗理解与实践

安全控制像接口安检：先确认身份，再检查权限。实际代码里用 `SecurityFilterChain` 定义公开路径和受保护路径，用密码哈希、JWT/OAuth2、`@PreAuthorize` 控制访问；安全配置不能只停在“引入 Starter”。

## 25. Spring Boot 如何集成和使用 RabbitMQ？

### 先用大白话理解

RabbitMQ 像消息邮局：生产者把消息交给邮局，消费者按自己的速度领取处理。它适合把“下单成功后发通知、扣积分、写分析日志”等非核心动作从主请求中拆出去，避免用户一直等待。

### 基本步骤

1. 引入 `spring-boot-starter-amqp`。
2. 配置 RabbitMQ 地址、端口、账号、虚拟主机等连接信息。
3. 声明 Exchange、Queue、Binding，明确消息如何路由到队列。
4. 使用 `RabbitTemplate` 发送消息。
5. 使用 `@RabbitListener` 消费消息，并配置 JSON 消息转换器。

```java
rabbitTemplate.convertAndSend("order.exchange", "order.created", orderEvent);

@RabbitListener(queues = "order.created.queue")
public void consume(OrderCreatedEvent event) { /* 幂等处理 */ }
```

⚠️ 真正生产难点在可靠性：消息确认、失败重试、死信队列、消费者幂等、重复消费、消息顺序和监控告警都要设计。消息队列不是“发出去就一定处理成功”。

## 26. Spring Boot 如何实现 RESTful API？

### 先用大白话理解

Spring Boot 做 REST API 就是用 `@RestController` 把 HTTP 请求映射到方法，再把 Java 对象自动转成 JSON 返回。它复用了 Spring MVC，只是 Boot 帮你准备了 Web 依赖、内嵌服务器、Jackson 等常用默认配置。

### 一个接口通常包含

1. `@RestController`：返回响应体而非视图页面。
2. `@GetMapping`、`@PostMapping` 等：把 URL 和 HTTP 方法映射到处理方法。
3. `@PathVariable`、`@RequestParam`、`@RequestBody`：接收路径、查询参数和 JSON 请求体。
4. `@Valid` / `@Validated`：校验输入。
5. `@RestControllerAdvice`：统一异常和错误响应。
6. `HttpMessageConverter`（通常 Jackson）：对象与 JSON 转换。

REST 风格、状态码、无状态、安全与幂等见 [[01-Spring核心与Spring MVC#22. 如何理解 REST API，以及接口如何保证安全？]]。Boot 层要额外关注跨域、异常格式、OpenAPI 文档、日志追踪和接口版本兼容性。

## 27. Spring Boot 如何集成和使用 Elasticsearch？

### 先用大白话理解

Elasticsearch 是专门做搜索和分析的引擎，不是用来替代 MySQL 的普通事务数据库。商品搜索、全文检索、日志检索、复杂筛选排序这类场景更适合它；订单、余额等强一致核心数据仍通常保存在关系型数据库。

### 基本接入步骤

1. 引入与当前 Elasticsearch 服务端版本兼容的 Spring Data Elasticsearch 依赖。
2. 在 `application.yml` 配置 Elasticsearch 地址、认证和超时等连接信息。
3. 定义映射到索引的文档实体，并使用 Repository 或 `ElasticsearchOperations` / `ElasticsearchRestTemplate` 操作。
4. 简单查询可用 Repository 派生方法；复杂全文检索、聚合、分页排序使用查询 API 或原生 DSL。
5. 建立数据库与索引的数据同步策略，例如写库后异步更新索引、定时重建或使用消息队列。

⚠️ 最难的不是“连上 ES”，而是索引映射、版本兼容和数据同步一致性。不要在一次用户请求中把数据库写入和 ES 更新当成天然原子操作。

## 28. Spring Boot 如何配置和使用 JMS？

### 先用大白话理解

JMS 是 Java 的消息规范；ActiveMQ 等是具体消息中间件实现。它和 RabbitMQ 一样适合把耗时的异步工作从主请求里拆出来，只是协议与生态不同。

### 基本接入步骤

1. 引入对应 JMS Provider 依赖，例如 ActiveMQ 相关 Starter。
2. 在配置文件中提供连接工厂地址、账号、队列或主题等参数。
3. 使用 `JmsTemplate` 发送消息到 Queue 或 Topic。
4. 使用 `@JmsListener` 声明消费者，异步监听消息。
5. 按业务配置事务、持久化、确认、重试与死信处理。

```java
jmsTemplate.convertAndSend("order.queue", orderEvent);

@JmsListener(destination = "order.queue")
public void consume(OrderEvent event) { /* 幂等处理 */ }
```

**Queue** 通常是一条消息由一个消费者处理；**Topic** 是发布订阅模式，一条消息可被多个订阅者接收。生产环境同样要处理重复消费、失败重试、消息持久化和监控，不能只关注发送 API。

## 29. Spring 与 Spring Boot 的能力接入有什么区别？

### 先给结论

Spring 提供核心能力和底层机制；Spring Boot 不会重新发明这些能力，而是通过 Starter、自动配置、内嵌服务器和外部化配置，让常见能力更快接入、默认可运行。

**记忆：Spring 解决“能不能做”，Spring Boot 解决“能不能少配点、快点跑起来”。**

### 能力接入对比表

| 能力 | Spring 中通常要关注什么 | Spring Boot 简化了什么 | 仍需要自己处理什么 |
| --- | --- | --- | --- |
| 数据访问 | 数据源、ORM/JDBC、事务管理器 | Starter、`DataSource`、常见事务 Bean 自动配置 | 多数据源、连接池参数、慢 SQL、事务边界 |
| 缓存 | 缓存抽象、缓存实现、键和失效策略 | Cache Starter、常用缓存管理器配置 | 缓存一致性、穿透/击穿/雪崩、多实例失效 |
| AOP | 代理、切面、通知、切点 | AOP Starter 准备常见代理能力 | 切点范围、同类调用失效、日志脱敏 |
| WebSocket | 端点、Handler、消息收发 | WebSocket Starter、常见容器支持 | 鉴权、心跳、连接数、多节点广播 |
| 国际化 | `MessageSource`、`LocaleResolver` | 默认消息资源与 Web 配置支持 | 文案治理、时区、金额/日期格式 |
| 文件上传 | Multipart 解析与请求处理 | Web Starter、上传大小属性 | 文件类型校验、路径穿越、对象存储、权限 |
| RabbitMQ / JMS | 连接工厂、Template、监听容器 | AMQP/JMS Starter、Template 和监听器配置 | 确认、重试、死信、幂等、顺序、监控 |
| Elasticsearch | 客户端、索引映射、查询 API | Spring Data 集成与连接配置 | 索引设计、版本兼容、数据库与索引同步 |
| 安全 | Spring Security 认证授权机制 | Security Starter、默认安全链 | 安全策略、JWT/OAuth2、CSRF/CORS、密钥和权限模型 |
| REST API | Spring MVC、JSON 转换、异常处理 | Web Starter、Jackson、内嵌服务器 | API 设计、状态码、校验、错误格式、版本兼容 |

### 使用这张表的方法

以后看到“Spring Boot 如何使用 X”这类题，先问三件事：

1. X 本身是谁提供的能力？是 Spring、Spring 生态还是外部中间件？
2. Boot 通过哪个 Starter / 自动配置帮我少配了什么？
3. 哪些生产关键细节仍必须由我显式设计？

这张表随本模块后续题目持续补充。

## 30. Spring Boot 如何处理跨域资源共享（CORS）？

### 先用大白话理解

浏览器有同源策略：前端在 `http://localhost:3000`，后端在 `http://localhost:8080` 时，浏览器会先问后端“你允许这个来源访问吗？”CORS 就是后端返回的允许规则。它是浏览器安全机制，不是后端接口本身的登录认证。

### 常见配置方式

1. **全局配置**：实现 `WebMvcConfigurer#addCorsMappings`，为一组路径统一允许来源、方法、请求头、凭证和预检缓存时间。
2. **局部配置**：在 Controller 或方法上使用 `@CrossOrigin`，适合少数特殊接口。
3. **Spring Security 集成**：启用 Security 后，必须在 Security 的过滤器链中启用/复用 CORS 配置，否则 MVC 的 CORS 配置可能在安全链前被拦截。

⚠️ 不要在生产中简单写 `allowedOrigins("*")` 并同时允许凭证；应明确允许的前端域名、方法和请求头。CORS 只控制浏览器跨域，不替代 Token、权限、HTTPS 和 CSRF 防护。

## 31. Spring Boot 如何实现数据校验？

### 先用大白话理解

数据校验就是在请求进入业务逻辑前先检查“这份表单能不能用”。例如手机号不能为空、年龄必须在范围内、密码长度必须足够。这样非法数据不会跑到 Service 或数据库后才报错。

### 使用步骤

1. 引入 Bean Validation 支持（常见为 `spring-boot-starter-validation`）。
2. 在 DTO 字段上使用 `@NotBlank`、`@NotNull`、`@Size`、`@Pattern`、`@Email` 等注解。
3. Controller 参数上加 `@Valid` 或 `@Validated` 触发校验。
4. 全局处理 `MethodArgumentNotValidException` 等异常，返回统一错误信息。
5. 复杂规则实现自定义 `ConstraintValidator`，不要把跨字段复杂校验硬塞进单字段注解。

```java
record CreateUserRequest(
    @NotBlank(message = "用户名不能为空") String name,
    @Email(message = "邮箱格式不正确") String email
) {}
```

⚠️ 校验是接口边界的第一道防线，但不能替代数据库约束、权限校验和业务规则校验。

## 32. Spring Boot 应用如何实现日志管理？

### 先用大白话理解

日志像应用的飞行记录仪：出问题时告诉你“谁在什么时候调用了什么、失败在哪里、耗时多久”。它不是越多越好，而是要能定位问题且不泄露敏感信息。

Spring Boot 通常使用 SLF4J 门面配合 Logback 默认实现。可在 `application.yml` 中配置包级日志级别；生产常用 INFO/WARN，排查特定问题时临时调高某个包的 DEBUG，再及时恢复。

### 生产实践

- 使用 `log.info("orderId={}, status={}", id, status)` 这种参数化日志，不用字符串拼接。
- 按日期/大小滚动并保留归档，避免单个日志文件无限增长。
- 记录 traceId、请求路径、关键业务 ID、耗时和异常堆栈；密码、Token、身份证等必须脱敏。
- 需要集中检索时可接入 ELK/OpenSearch 等日志平台。
- 不要用日志替代监控指标和告警；严重异常应有告警闭环。

## 33. Spring Boot 如何集成和使用 GraphQL？

### 先用大白话理解

REST 通常由后端决定每个接口返回什么字段；GraphQL 让客户端声明“我这次只要哪些字段”，由一个 GraphQL Endpoint 按查询内容返回数据。它适合页面需要组合多个资源、字段需求变化较多的场景。

### 基本组成

1. 引入 Spring for GraphQL / Boot Starter。
2. 定义 Schema：类型（Type）、查询（Query）、修改（Mutation）以及字段关系。
3. 编写 Controller / Resolver，把 Schema 字段映射到业务查询方法。
4. 配置 GraphQL Endpoint、鉴权、异常处理和查询复杂度限制。
5. 处理 N+1 查询，可使用 DataLoader 批量加载关联数据。

⚠️ GraphQL 不会自动替代 REST：简单 CRUD、文件上传、缓存友好的公共接口通常 REST 更直接；GraphQL 更需要防止查询过深、字段越权和 N+1 性能问题。

## 34. Spring Boot 如何管理应用配置？

### 先用大白话理解

应用配置就是“同一套程序在不同地方运行时，要带哪份说明书”。

代码不应该写死数据库地址、端口、第三方密钥、日志级别。开发环境可能连本地数据库，测试环境连测试库，生产环境连生产库；程序代码相同，只是启动时读取的配置不同。

### 配置从哪里来

1. **`application.yml` / `application.properties`**：最常见的默认配置文件，适合端口、数据库、日志等通用配置。
2. **`@Value`**：取一个简单配置值，例如 `${server.port}`；适合少量单独配置。
3. **`@ConfigurationProperties`**：把一组同前缀配置绑定成对象，适合数据库、第三方服务等成组配置，类型安全且便于校验。
4. **Profile 文件**：`application-dev.yml`、`application-prod.yml` 等保存环境差异。
5. **命令行参数**：启动时显式覆盖，例如 `--server.port=9090`。
6. **环境变量**：部署平台、Docker、Kubernetes 常用；适合环境差异和敏感配置注入。

```yaml
app:
  payment:
    timeout: 3000
    base-url: https://api.example.com
```

```java
@ConfigurationProperties(prefix = "app.payment")
public record PaymentProperties(Duration timeout, URI baseUrl) {}
```

### 覆盖与实践原则

通常更外部、更明确的配置能覆盖打包内默认配置，例如命令行、环境变量覆盖 `application.yml`。具体优先级受 Boot 版本和加载方式影响，出现“配置没生效”时用 Actuator 环境端点或启动日志确认，不要硬背全部顺序。

⚠️ 密码、Token、数据库账号不应提交到 Git；本地可用环境变量或忽略的私有文件，生产应使用部署平台的密钥管理、配置中心或 Secret。业务配置集中成 `@ConfigurationProperties`，不要在各个类里散落几十个 `@Value`。
