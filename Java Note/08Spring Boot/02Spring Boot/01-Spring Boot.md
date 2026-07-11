# Spring Boot

> 本页吸收“Spring 刷题题库”中 Spring Boot 相关题目。与 Spring 核心共用的事务、MVC、Security 等内容通过 [[01Spring/01-Spring核心与Spring MVC]] 回顾，避免重复。

## 1. Spring Boot 是什么？与 Spring 有什么区别？

Spring Boot 不是替代 Spring，而是建立在 Spring 之上的快速开发方案。它通过自动配置、Starter、内嵌服务器和外部化配置减少样板代码。Spring 更像能力集合，Boot 负责把常见组合按约定装配好。

## 2. Spring Boot 自动配置原理是什么？

启动类上的 `@SpringBootApplication` 包含 `@EnableAutoConfiguration`。Boot 会导入候选自动配置类，再根据类路径、已有 Bean、配置属性等 `@Conditional` 条件决定是否生效；生效后向容器注册默认 Bean，开发者配置或自定义 Bean 可覆盖默认行为。

⚠️ 自动配置不是“无条件创建一切”，而是“满足条件才按约定配置”。

## 3. @SpringBootApplication 包含哪些注解？

它组合了 `@SpringBootConfiguration`（配置类）、`@EnableAutoConfiguration`（启用自动配置）和 `@ComponentScan`（组件扫描）。扫描默认从启动类所在包及其子包开始，所以启动类一般放在项目根包。

## 4. Starter 是什么？工作原理是什么？

Starter 是一组场景依赖描述，例如 `spring-boot-starter-web`。它通过依赖传递引入常用库，并配套自动配置：Boot 启动时读取自动配置声明，满足条件后装配对应组件。它解决的是依赖版本组合和初始化配置繁琐的问题。

## 5. 如何自定义 Starter？

通常拆分为自动配置模块和 Starter 依赖模块：在自动配置中用 `@Configuration`、`@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@EnableConfigurationProperties` 定义可选 Bean；在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 声明自动配置类；Starter 模块只负责聚合依赖。重点是“条件化、可覆盖、配置可绑定”。

## 6. application.yml、properties 与配置优先级如何理解？

两种格式都能配置；YAML 层级更清晰，properties 更直接。配置可来自默认文件、Profile 文件、环境变量、命令行参数、外部文件等，通常越靠近运行环境、显式程度越高的配置优先级越高。不要死记每一项顺序，遇到覆盖问题用 Actuator 的环境端点或启动日志确认。

## 7. @Value 与 @ConfigurationProperties 如何选择？

少量、单个值可用 `@Value`；一组有层级的业务配置优先 `@ConfigurationProperties`，它支持类型绑定、校验、元数据提示，维护性更好。敏感配置不要写入仓库，应通过环境变量、配置中心或密钥管理注入。

## 8. Profile 的作用是什么？

Profile 用于区分开发、测试、生产等环境配置与 Bean。通过 `spring.profiles.active` 激活，配置文件可使用 `application-dev.yml` 等约定。它解决的是“同一份代码在不同环境连接不同资源、开关不同功能”的问题。

## 9. CommandLineRunner 与 ApplicationRunner 有什么区别？

二者都在 Spring Boot 启动完成后执行初始化逻辑。`CommandLineRunner` 接收字符串数组，`ApplicationRunner` 接收解析后的 `ApplicationArguments`。多个 Runner 可用 `@Order` 排序。⚠️ 不要把长期阻塞任务放进 Runner，否则会影响应用就绪。

## 10. Spring Boot 如何统一处理异常？

推荐用 `@RestControllerAdvice` 和 `@ExceptionHandler` 统一返回错误码、提示语、请求标识等。参数校验异常、业务异常、系统异常分别处理；生产环境不要返回堆栈和内部实现细节。错误响应格式要稳定，便于前端和监控系统处理。

## 11. Spring Boot 的数据访问与事务如何配置？

Boot 可依据依赖和配置自动创建 `DataSource`、JPA、MyBatis 等相关 Bean。业务事务仍用 `@Transactional`，原理与注意点见 [[01Spring/01-Spring核心与Spring MVC#12. Spring 的声明式事务原理是什么？]]。数据源、连接池、SQL 日志和迁移工具应按环境配置，不要把生产账号密码写进 YAML。

## 12. JPA 与 Hibernate 是什么关系？如何优化 JPA？

JPA 是 Java 持久化规范，Hibernate 是常用实现之一。优化重点不是盲目调注解：避免 N+1 查询（抓取策略、`join fetch`、批量查询）、分页时避免不必要的关联抓取、为高频条件建索引、必要时改用原生 SQL 或 MyBatis。

## 13. Spring Cache 如何使用？

通过 `@EnableCaching` 启用缓存，常用 `@Cacheable` 查缓存、`@CachePut` 更新缓存、`@CacheEvict` 删除缓存。缓存键要能区分数据版本和查询条件；更新数据库后要处理缓存一致性。单机可用 Caffeine，多实例通常使用 Redis 等共享缓存。

## 14. 如何实现异步任务与定时任务？

异步使用 `@EnableAsync + @Async`，定时任务使用 `@EnableScheduling + @Scheduled`。异步线程池必须显式配置容量、队列、拒绝策略和异常处理；多实例定时任务需要分布式锁或调度平台，否则会重复执行。

## 15. WebSocket 在 Spring Boot 中怎么用？

WebSocket 适合实时通知、聊天和行情推送等服务端主动推送场景。Boot 可配置 WebSocket 端点，或使用 STOMP；生产中还要考虑鉴权、连接数、心跳、消息顺序、断线重连与多节点消息广播。

## 16. Spring Boot 如何处理文件上传、静态资源和国际化？

上传使用 `MultipartFile`，要限制大小、校验类型/内容、使用随机文件名并避免直接暴露存储路径；静态资源可由默认目录或资源映射提供；国际化通过 `MessageSource` 按 Locale 返回不同文本。⚠️ 文件上传是安全入口，必须防路径穿越和伪造文件类型。

## 17. Actuator 是什么？生产环境如何安全使用？

Actuator 提供健康检查、指标、环境、日志级别等运维端点，可对接监控系统。生产只暴露必要端点，使用独立管理端口或网络隔离并加认证授权；`env`、配置和线程信息等敏感端点不能随意公开。

## 18. 如何测试和保护 Spring Boot 应用？

测试分层进行：单元测试关注业务类，`@WebMvcTest` 测 MVC 层，`@DataJpaTest` 测持久层，`@SpringBootTest` 做必要集成测试。安全方面至少做到依赖漏洞扫描、及时升级、HTTPS、输入校验、认证授权、CSRF/XSS 防护、日志脱敏和限流；不能只依赖框架默认配置。

## 19. Spring、Spring Boot、Spring Cloud 的关系是什么？

Spring 是基础生态；Spring Boot 在其上简化单个应用的创建、配置和运行；Spring Cloud 在 Boot 基础上提供微服务治理能力，如注册发现、配置中心、网关、熔断和链路追踪。可记为：Spring 提供底座，Boot 提升开发效率，Cloud 解决分布式协作。

## 20. 如何自定义 Spring Boot 启动 Banner？

在 `src/main/resources` 下放置 `banner.txt`，或通过 `SpringApplication#setBanner` 设置自定义 Banner；可用 `spring.main.banner-mode=off` 关闭。它只影响启动展示，不参与业务能力，面试时说明“可通过资源文件或代码自定义”即可。

## 21. Spring Boot 的核心启动注解是什么？

核心注解是 `@SpringBootApplication`，由 `@SpringBootConfiguration`、`@EnableAutoConfiguration`、`@ComponentScan` 组合而来，分别承担配置声明、自动配置和组件扫描职责。详见第 3 题；项目启动类的位置会影响扫描范围。
