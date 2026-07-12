# Spring Boot

> 本页吸收“Spring 刷题题库”中 Spring Boot 相关题目。与 Spring 核心共用的事务、MVC、Security 等内容通过 [[01-Spring核心与Spring MVC]] 回顾，避免重复。

## 1. Spring Boot 是什么？与 Spring 有什么区别？

Spring Boot 不是替代 Spring，而是建立在 Spring 之上的快速开发方案。它通过自动配置、Starter、内嵌服务器和外部化配置减少样板代码。Spring 更像能力集合，Boot 负责把常见组合按约定装配好。

它的直接收益是：依赖版本由 BOM/Starter 协调、Web 应用可打成可执行 Jar 并使用内嵌 Tomcat/Jetty/Undertow、默认配置可运行但仍可覆盖，同时自带健康检查和指标等生产能力。所谓“约定优于配置”不是不能配置，而是先给合理默认值。

## 2. Spring Boot 自动配置原理是什么？

启动类上的 `@SpringBootApplication` 包含 `@EnableAutoConfiguration`。Boot 会导入候选自动配置类，再根据类路径、已有 Bean、配置属性等 `@Conditional` 条件决定是否生效；生效后向容器注册默认 Bean，开发者配置或自定义 Bean 可覆盖默认行为。

排查自动配置时，按“是否引入依赖 → 条件是否满足 → 是否已有同类 Bean → 属性是否正确”检查；必要时查看 Condition Evaluation Report。常用条件包括 `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty`、`@ConditionalOnWebApplication`。

⚠️ 自动配置不是“无条件创建一切”，而是“满足条件才按约定配置”。

## 3. @SpringBootApplication 包含哪些注解？

它组合了 `@SpringBootConfiguration`（配置类）、`@EnableAutoConfiguration`（启用自动配置）和 `@ComponentScan`（组件扫描）。扫描默认从启动类所在包及其子包开始，所以启动类一般放在项目根包。

## 4. Starter 是什么？工作原理是什么？

Starter 是一组场景依赖描述，例如 `spring-boot-starter-web`。它通过依赖传递引入常用库，并配套自动配置：Boot 启动时读取自动配置声明，满足条件后装配对应组件。它解决的是依赖版本组合和初始化配置繁琐的问题。

原理上，Starter 负责把一组经过兼容性管理的传递依赖带进来，自动配置负责按条件创建 Bean。旧版 Boot 常通过 `META-INF/spring.factories` 声明自动配置；Boot 3 主要使用 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。两者的核心都是“发现候选配置 → 条件匹配 → 注入容器”，不要只背某一个文件名。

## 5. 如何自定义 Starter？

通常拆分为自动配置模块和 Starter 依赖模块：在自动配置中用 `@Configuration`、`@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@EnableConfigurationProperties` 定义可选 Bean；在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 声明自动配置类；Starter 模块只负责聚合依赖。重点是“条件化、可覆盖、配置可绑定”。

## 6. application.yml、properties 与配置优先级如何理解？

两种格式都能配置；YAML 层级更清晰，properties 更直接。配置可来自默认文件、Profile 文件、环境变量、命令行参数、外部文件等，通常越靠近运行环境、显式程度越高的配置优先级越高。不要死记每一项顺序，遇到覆盖问题用 Actuator 的环境端点或启动日志确认。

同一键被多处定义时，命令行和环境变量通常可覆盖打包内配置；精确优先级会随 Boot 版本和加载方式变化。团队实践是：仓库提交默认值和非敏感配置，环境差异与密钥交给部署平台或配置中心。

## 7. @Value 与 @ConfigurationProperties 如何选择？

少量、单个值可用 `@Value`；一组有层级的业务配置优先 `@ConfigurationProperties`，它支持类型绑定、校验、元数据提示，维护性更好。敏感配置不要写入仓库，应通过环境变量、配置中心或密钥管理注入。

## 8. Profile 的作用是什么？

Profile 用于区分开发、测试、生产等环境配置与 Bean。通过 `spring.profiles.active` 激活，配置文件可使用 `application-dev.yml` 等约定。它解决的是“同一份代码在不同环境连接不同资源、开关不同功能”的问题。

可通过配置文件、环境变量、JVM 参数或命令行激活；`@Profile("prod")` 可让某个 Bean 仅在指定环境注册。生产环境最好由部署平台传入激活项，避免把 `spring.profiles.active=dev` 固化在通用包内。

## 9. CommandLineRunner 与 ApplicationRunner 有什么区别？

二者都在 Spring Boot 启动完成后执行初始化逻辑。`CommandLineRunner` 接收字符串数组，`ApplicationRunner` 接收解析后的 `ApplicationArguments`。多个 Runner 可用 `@Order` 排序。⚠️ 不要把长期阻塞任务放进 Runner，否则会影响应用就绪。

Runner 适合初始化演示数据、校验启动依赖或执行一次性迁移，不适合承担常驻消费循环。带 `--key=value` 这类选项时，`ApplicationRunner` 能通过 `getOptionNames()`、`getOptionValues()` 更方便地区分选项参数和普通参数。

## 10. Spring Boot 如何统一处理异常？

推荐用 `@RestControllerAdvice` 和 `@ExceptionHandler` 统一返回错误码、提示语、请求标识等。参数校验异常、业务异常、系统异常分别处理；生产环境不要返回堆栈和内部实现细节。错误响应格式要稳定，便于前端和监控系统处理。

需要复用 Spring MVC 的默认异常处理时，可继承 `ResponseEntityExceptionHandler`；校验场景可读取 `BindingResult`，或统一处理 `MethodArgumentNotValidException`。`@ResponseStatus` 适合为明确的异常绑定 HTTP 状态，但复杂接口仍应统一错误响应体，避免前端面对多套格式。

## 11. Spring Boot 的数据访问与事务如何配置？

Boot 可依据依赖和配置自动创建 `DataSource`、JPA、MyBatis 等相关 Bean。业务事务仍用 `@Transactional`，原理与注意点见 [[01-Spring核心与Spring MVC#12. Spring 的声明式事务原理是什么？]]。数据源、连接池、SQL 日志和迁移工具应按环境配置，不要把生产账号密码写进 YAML。

数据访问层要明确边界：JPA 适合领域模型和常规 CRUD，MyBatis 适合 SQL 需要精细掌控的场景，二者可以共存但应避免同一业务随意混用。数据库迁移推荐采用 Flyway 或 Liquibase 等可版本化工具，而不是靠手工执行 SQL。

### 数据访问配置追问

连接池要配置最大连接数、超时与校验策略，并与数据库容量匹配；生产中应开启必要 SQL/慢查询观测而不是长期打印所有 SQL。事务方法要通过 Spring 代理调用，数据源和事务管理器必须对应，否则可能出现“代码写了事务但没有回滚”。

## 12. JPA 与 Hibernate 是什么关系？如何优化 JPA？

JPA 是 Java 持久化规范，Hibernate 是常用实现之一。优化重点不是盲目调注解：避免 N+1 查询（抓取策略、`join fetch`、批量查询）、分页时避免不必要的关联抓取、为高频条件建索引、必要时改用原生 SQL 或 MyBatis。

### JPA 性能检查

实体关联默认策略、序列化时访问懒加载字段、循环引用、分页加 `fetch join` 都可能产生意外 SQL 或错误结果。优化的顺序应是：看 SQL 与执行计划 → 修正查询与索引 → 控制抓取范围 → 再考虑缓存，而不是一开始就调大连接池。

## 13. Spring Cache 如何使用？

通过 `@EnableCaching` 启用缓存，常用 `@Cacheable` 查缓存、`@CachePut` 更新缓存、`@CacheEvict` 删除缓存。缓存键要能区分数据版本和查询条件；更新数据库后要处理缓存一致性。单机可用 Caffeine，多实例通常使用 Redis 等共享缓存。

缓存注解同样依赖代理，因此同类内部调用可能不生效。高并发热点还要额外防缓存穿透、击穿和雪崩；缓存不是数据库事务的一部分，更新策略要明确“先更新库还是先删缓存”以及失败补偿方案。

## 14. 如何实现异步任务与定时任务？

异步使用 `@EnableAsync + @Async`，定时任务使用 `@EnableScheduling + @Scheduled`。异步线程池必须显式配置容量、队列、拒绝策略和异常处理；多实例定时任务需要分布式锁或调度平台，否则会重复执行。

`@Async` 返回 `void` 时异常不会自动抛回调用方，应配置 `AsyncUncaughtExceptionHandler` 或改用 `Future`/`CompletableFuture` 获取结果。定时任务也要记录执行时长、幂等键和失败告警，避免静默失败。

### 缓存一致性追问

`@Cacheable` 命中缓存时目标方法不会执行；`@CachePut` 总会执行并写缓存；`@CacheEvict` 用于删除。多实例场景还要考虑本地缓存失效广播；热点 key 需要考虑过期时间随机化、互斥重建或逻辑过期等方案。

## 15. WebSocket 在 Spring Boot 中怎么用？

WebSocket 适合实时通知、聊天和行情推送等服务端主动推送场景。Boot 可配置 WebSocket 端点，或使用 STOMP；生产中还要考虑鉴权、连接数、心跳、消息顺序、断线重连与多节点消息广播。

### WebSocket 生产注意点

长连接并不等于无限连接：需要限制单用户连接数、消息大小和空闲时间，使用心跳清理失效连接。多节点部署时连接在不同实例上，广播消息需要 Redis、MQ 等共享通道；鉴权应在握手或消息协议层完成。

## 16. Spring Boot 如何处理文件上传、静态资源和国际化？

上传使用 `MultipartFile`，要限制大小、校验类型/内容、使用随机文件名并避免直接暴露存储路径；静态资源可由默认目录或资源映射提供；国际化通过 `MessageSource` 按 Locale 返回不同文本。⚠️ 文件上传是安全入口，必须防路径穿越和伪造文件类型。

### 文件与资源安全

上传文件应存到对象存储或受控目录，下载通过业务权限校验后以受控响应输出；禁止根据用户输入直接拼接文件路径。静态资源应设置缓存策略，国际化资源文件应与业务错误码统一维护。

## 17. Actuator 是什么？生产环境如何安全使用？

Actuator 提供健康检查（`health`）、指标（`metrics`）、环境（`env`）、动态日志级别（`loggers`）、线程信息（`threaddump`）、应用信息（`info`）、审计事件等运维端点；HTTP 请求跟踪在新版本中通常以 `httpexchanges` 等端点或观测能力呈现。它可对接监控系统。生产只暴露必要端点，使用独立管理端口或网络隔离并加认证授权；`env`、配置和线程信息等敏感端点不能随意公开。

## 18. 如何测试和保护 Spring Boot 应用？

测试分层进行：单元测试关注业务类，`@WebMvcTest` 测 MVC 层，`@DataJpaTest` 测持久层，`@SpringBootTest` 做必要集成测试。安全方面至少做到依赖漏洞扫描、及时升级、HTTPS、输入校验、认证授权、CSRF/XSS 防护、日志脱敏和限流；不能只依赖框架默认配置。

依赖漏洞扫描可使用 Snyk、Dependabot、OWASP Dependency-Check 等任一适配团队流程的工具；工具告警仍需结合可达性和版本修复策略判断，不能只“扫描一次”就结束。

### 监控落地

健康检查至少区分存活（liveness）和就绪（readiness）：前者决定是否重启进程，后者决定是否接流量。指标应输出到 Prometheus、OpenTelemetry 等观测体系并建立告警；不要把敏感的 `env`、heapdump、线程栈端点暴露给公网。

## 19. Spring、Spring Boot、Spring Cloud 的关系是什么？

Spring 是基础生态；Spring Boot 在其上简化单个应用的创建、配置和运行；Spring Cloud 在 Boot 基础上提供微服务治理能力，如注册发现、配置中心、网关、熔断和链路追踪。可记为：Spring 提供底座，Boot 提升开发效率，Cloud 解决分布式协作。

## 20. 如何自定义 Spring Boot 启动 Banner？

在 `src/main/resources` 下放置 `banner.txt`，或通过 `SpringApplication#setBanner` 设置自定义 Banner；可用 `spring.main.banner-mode=off` 关闭。它只影响启动展示，不参与业务能力，面试时说明“可通过资源文件或代码自定义”即可。

也可用 `spring.banner.location` 指定 Banner 文件位置。旧资料中的 `spring.banner.enabled=false` 在新版本中更推荐改为 `spring.main.banner-mode=off`。

## 21. Spring Boot 的核心启动注解是什么？

核心注解是 `@SpringBootApplication`，由 `@SpringBootConfiguration`、`@EnableAutoConfiguration`、`@ComponentScan` 组合而来，分别承担配置声明、自动配置和组件扫描职责。详见第 3 题；项目启动类的位置会影响扫描范围。

需要排除某项自动配置时，可在 `@SpringBootApplication(exclude = XxxAutoConfiguration.class)` 指定，但应先确认是否只是缺少正确配置；盲目排除会造成后续依赖 Bean 缺失。
