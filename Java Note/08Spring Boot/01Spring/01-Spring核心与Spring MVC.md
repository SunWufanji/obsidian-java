# Spring 核心与 Spring MVC

> 本页吸收“Spring 刷题题库”中 Spring、MVC、Security 相关题目；同义题（如 IoC/DI、Bean 生命周期、AOP、事务）统一合并，避免重复。

## 1. 什么是 Spring，解决了什么问题？

Spring 的核心价值是把对象的创建、组装和通用能力交给框架处理，让业务代码专注于业务。它通过 IoC 管理对象，通过 AOP 承载事务、日志、安全等横切能力，并提供 MVC、数据访问等模块。

**面试一句话：**Spring 是一个轻量级 Java 企业开发框架，核心是 IoC 和 AOP，目的是解耦并复用基础能力。

## 2. 什么是 Bean？Bean 有哪些作用域？

Bean 是被 Spring IoC 容器创建和管理的对象。常用作用域：`singleton`（默认、容器内一个实例）、`prototype`（每次获取新实例）、Web 环境的 `request`、`session`、`application` 和 `websocket`。

⚠️ `singleton` 指“Spring 容器中单例”，不等于天然线程安全；无状态 Bean 通常安全，有可变共享字段时要自行处理并发。

## 3. Bean 的完整生命周期是怎样的？

典型流程是：实例化 → 注入属性 → 执行 `*Aware` 回调 → `BeanPostProcessor#postProcessBeforeInitialization` → 初始化（`@PostConstruct`、`afterPropertiesSet`、`init-method`）→ `postProcessAfterInitialization`（AOP 代理通常在此产生）→ 使用 → 容器关闭时执行 `@PreDestroy`、`destroy`、`destroy-method`。

**记忆：**生出来、装配好、前置加工、初始化、后置加工、使用、销毁。

## 4. IoC 是什么？IoC 容器的职责是什么？

IoC（控制反转）不是业务对象自己 `new` 依赖，而是把对象控制权交给容器。容器负责读取配置、创建 Bean、注入依赖、管理生命周期，并提供事件、资源、国际化等基础设施。

它的本质是依赖方向反转：业务类依赖抽象，组装工作由容器完成，因此替换实现和测试更容易。

## 5. DI 是什么？与 IoC 有什么区别？

DI（依赖注入）是 IoC 的具体实现方式：容器把一个对象所需的依赖注入进去。IoC 是思想，DI 是落地手段。

常见注入方式：构造器注入、Setter 注入、字段注入、方法参数注入。优先使用**构造器注入**：依赖明确、可以使用 `final`、便于单元测试；Setter 适合可选依赖；字段注入不利于测试且隐藏依赖。

## 6. Spring 如何解决循环依赖？为什么推荐三级缓存？

单例字段注入的循环依赖可借助三级缓存解决：一级缓存存完整 Bean，二级缓存存提前暴露的早期 Bean，三级缓存存 `ObjectFactory`。提前暴露工厂能在需要时生成早期引用；若 Bean 需要 AOP，工厂还能保证注入的是同一个早期代理。

⚠️ 构造器循环依赖无法靠此机制解决；`prototype` 循环依赖也不支持。业务上更应通过拆分职责消除循环依赖。

## 7. BeanFactory 与 ApplicationContext 有什么区别？

`BeanFactory` 是最基础的 IoC 容器，提供 Bean 获取与管理能力；`ApplicationContext` 在其基础上增加国际化、事件发布、资源加载、环境配置，以及多数单例 Bean 的预实例化。日常 Spring Boot 应用使用的是 `ApplicationContext`。

## 8. 什么是 AOP？核心术语有哪些？

AOP 用来把日志、事务、权限、监控这类“多处都要做”的横切关注点从业务代码中抽离。`Aspect` 是切面，`Join Point` 是可增强位置，`Pointcut` 是匹配规则，`Advice` 是通知，`Target` 是目标对象，`Proxy` 是代理对象，`Weaving` 是织入过程。

常用通知有 `@Before`、`@After`、`@AfterReturning`、`@AfterThrowing`、`@Around`；其中环绕通知能决定是否调用目标方法，能力最强。

## 9. Spring AOP 与 AspectJ 有什么区别？

Spring AOP 基于运行时代理，主要拦截 Spring Bean 的方法执行，适合绝大多数业务切面；AspectJ 是更完整的 AOP 体系，可在编译期或类加载期织入，能覆盖构造器、字段等更多连接点。

Spring AOP 选择 JDK 动态代理（有接口）或 CGLIB（无接口）。⚠️ `final` 类/方法不能被 CGLIB 重写；同类内部调用不会经过代理，常导致 `@Transactional`、AOP 失效。

## 10. Spring 中常见设计模式有哪些？

工厂模式：`BeanFactory`、`FactoryBean`；单例模式：默认单例 Bean；代理模式：AOP；模板方法：`JdbcTemplate`、`RestTemplate`；观察者模式：事件监听；策略模式：资源、排序、不同实现选择；适配器模式：MVC 的 `HandlerAdapter`。

面试不要只报名称，最好补一句“它解决什么”：例如 `JdbcTemplate` 固化 JDBC 流程，让开发者只写 SQL 和结果映射。

## 11. Spring 事件和监听器如何工作？

发布者通过 `ApplicationEventPublisher` 发布事件，监听器用 `@EventListener` 或实现 `ApplicationListener` 接收。它适合把“下单成功后发通知、记日志”等非核心动作解耦。默认同步执行；需要异步要配合 `@Async` 或消息队列，并处理失败重试与事务边界。

## 12. Spring 的声明式事务原理是什么？

`@Transactional` 本质是 AOP：代理在目标方法前开启或加入事务，正常返回时提交，异常时按规则回滚。真正的连接、提交、回滚由 `PlatformTransactionManager` 协调；JDBC 常用 `DataSourceTransactionManager`，JPA 常用 `JpaTransactionManager`。

**面试一句话：**声明式事务是代理拦截方法，再委托事务管理器绑定、提交或回滚当前线程资源。

## 13. 事务的传播行为有哪些？

最常用的 `REQUIRED`：有事务就加入，没有就新建。`REQUIRES_NEW`：挂起外层事务并新建；`NESTED`：在当前事务内建立保存点；`SUPPORTS`、`NOT_SUPPORTED`、`NEVER`、`MANDATORY` 分别对应支持、非事务、禁止事务、必须已有事务。

⚠️ `REQUIRES_NEW` 的内外事务彼此独立；`NESTED` 依赖保存点，仍受外层最终提交/回滚影响。

## 14. 哪些情况会导致 @Transactional 失效？

常见原因：同类内部调用绕过代理；方法不是 `public`（默认代理配置下）；异常被吞掉；抛出受检异常但未配置 `rollbackFor`；Bean 不由 Spring 管理；事务方法在初始化阶段调用；使用了不受代理支持的 `final` 限制。排查时先确认“调用是否经过 Spring 代理”。

## 15. Spring MVC 的请求处理流程是什么？

请求进入 `DispatcherServlet`，再由 `HandlerMapping` 找到处理器，由 `HandlerAdapter` 调用 Controller；Controller 返回 Model/View 或响应体；视图场景交给 `ViewResolver` 渲染，`@ResponseBody`/`@RestController` 则由 `HttpMessageConverter` 写回 JSON。

**记忆：**前端控制器找处理器、适配器调用、视图解析或消息转换返回。

## 16. @Controller、@RestController、@RequestMapping 有什么区别？

`@Controller` 用于 MVC 控制器，返回字符串默认会被当作视图名；`@RestController` 等于 `@Controller + @ResponseBody`，返回值直接写入响应体，常用于 REST API。`@RequestMapping` 可定义路径、HTTP 方法、参数和请求头条件；`@GetMapping` 等是它的语义化快捷写法。

## 17. @RequestParam、@PathVariable、@RequestBody 如何选择？

`@RequestParam` 获取查询参数或表单参数，如 `/users?page=1`；`@PathVariable` 获取资源路径变量，如 `/users/{id}`；`@RequestBody` 把 JSON 请求体反序列化为对象。REST 风格中，资源标识优先放路径，筛选/分页放查询参数，复杂写入数据放请求体。

## 18. Spring MVC 如何返回 JSON、做参数校验和统一异常处理？

返回 JSON 依赖 `HttpMessageConverter`（常见 Jackson）；参数对象标注 `@Valid`/`@Validated` 并在字段上加校验注解；使用 `@RestControllerAdvice + @ExceptionHandler` 统一转换异常为约定响应体。

⚠️ 校验结果要显式处理 `BindingResult` 或让异常进入全局处理器；不要把数据库异常原样返回给前端。

## 19. Filter、HandlerInterceptor、AOP 有什么区别？

Filter 属于 Servlet 规范，最靠近 HTTP 容器，适合编码、CORS、通用请求包装；Interceptor 位于 Spring MVC 调用 Controller 前后，适合登录校验、接口审计；AOP 面向方法调用，适合事务、日志和权限等业务横切逻辑。三者不是替代关系，按作用层次选择。

## 20. Spring Security 的认证与授权流程是什么？

请求经过安全过滤器链，认证阶段确认“你是谁”（账号密码、Token、OAuth2 等）并生成 `Authentication`；授权阶段根据角色、权限或表达式判断“你能做什么”。认证结果通常存入 `SecurityContext`。前后端分离项目常采用无状态 Token，并配置密码加密、CORS、CSRF 策略和异常响应。

## 21. OAuth2 是什么？

OAuth2 是授权框架，不等于登录协议本身。它让用户授权第三方在限定范围内访问资源，而不必交出密码。常见角色：资源所有者、客户端、授权服务器、资源服务器；常见授权方式包含授权码模式。实际项目中常与 OpenID Connect 组合完成身份认证。

## 22. 如何理解 REST API，以及接口如何保证安全？

REST 以资源为中心，用 URI 标识资源、HTTP 方法表达操作（GET 查、POST 建、PUT/PATCH 改、DELETE 删），保持无状态并使用合适状态码。接口安全需使用 HTTPS、认证授权、输入校验、限流、防重放与日志审计；写接口要考虑幂等性，敏感信息不能放 URL 或日志。

## 23. Spring Data JPA 的工作原理是什么？

Spring Data JPA 在 JPA 规范之上提供 Repository 抽象。应用定义实体和 Repository 接口后，Spring 在启动时为接口创建代理；代理根据方法名推导查询，或执行 `@Query` 指定的 JPQL/SQL，再交给 JPA 实现（如 Hibernate）完成 EntityManager、SQL 与结果映射工作。

⚠️ Repository 很省代码，但复杂查询仍要关注 SQL、索引和 N+1 问题，不能把性能问题藏在方法名后面。

## 24. Spring 如何实现国际化（i18n）？

配置 `MessageSource` 和不同语言的资源文件（如 `messages_zh_CN.properties`、`messages_en_US.properties`），再依据请求 Locale 解析并获取对应文案。Web 场景可结合 `LocaleResolver`、请求头或参数切换语言。国际化的对象是文案，不应把业务分支复制成多套语言代码。

## 25. JdbcTemplate 是什么，解决了什么问题？

`JdbcTemplate` 把 JDBC 中获取连接、创建语句、执行、关闭资源、转换异常这些固定流程封装起来，开发者只需提供 SQL、参数和 `RowMapper`。它仍是直接操作 JDBC，适合简单、可控的 SQL 场景；复杂动态 SQL 可考虑 [[03MyBatis/01-MyBatis]]。

## 26. Spring 的数据访问异常层次结构有什么价值？

Spring 把不同数据库和 ORM 的底层异常翻译为统一、非受检的 `DataAccessException` 层次结构，例如数据完整性冲突、乐观锁失败、查询结果不正确等。业务代码因此不必依赖某个 JDBC 驱动的异常类型，也更容易判断异常是否可重试。

## 27. Spring MVC 与 Spring WebFlux 有什么区别？

Spring MVC 基于 Servlet API，通常采用“一请求一线程”的阻塞式编程模型，生态成熟且适合大多数 CRUD 应用；WebFlux 基于 Reactor，支持非阻塞 I/O 和背压，适合大量慢 I/O、长连接、高并发场景。

⚠️ WebFlux 不是性能开关：内部仍调用阻塞 JDBC、文件 I/O 或同步远程调用时，会堵塞事件循环，反而得不偿失。

## 28. Spring 中的 Template 模式体现在哪里？

模板方法模式把稳定流程固定在父类或模板中，把变化点交给回调。`JdbcTemplate` 固化 JDBC 流程，`RestTemplate`/WebClient 封装请求流程，事务模板 `TransactionTemplate` 封装事务边界。它的好处是减少重复样板代码，同时保留关键业务步骤的可定制性。

## 29. Spring Security 过滤器链是什么？

Spring Security 把认证、鉴权、异常转换、CSRF 等安全工作拆成多个 Filter，按顺序组成过滤器链。请求先经过链，再进入 `DispatcherServlet`；其中某个过滤器可以直接拒绝请求或建立认证上下文。排查 401/403 时，应先看请求命中了哪条 `SecurityFilterChain`、认证信息是否建立、授权规则是否匹配。

## 30. 如何使用 AOP 实现日志记录？

用切点匹配 Controller 或 Service 方法，在 `@Around` 通知中记录方法名、关键参数、耗时、结果或异常，并调用 `proceed()` 执行目标方法。日志要注意脱敏、控制体积和避免重复记录；不要在切面里序列化巨大对象或读取一次性请求流。

## 31. Spring 有哪些配置和装配方式？如何声明 Bean？

配置方式包括 XML、Java 配置（`@Configuration + @Bean`）和注解扫描。常用组件注解有 `@Component`、`@Service`、`@Repository`、`@Controller`；装配可通过构造器、`@Autowired`、`@Qualifier` 和 `@Primary` 解决候选 Bean 选择问题。现代项目优先 Java 配置和构造器注入，XML 主要用于维护旧项目。

## 32. Spring 事务有哪些实现方式？各有什么特点？

声明式事务用 `@Transactional`，侵入业务代码少，是默认首选；编程式事务使用 `TransactionTemplate` 或直接调用 `PlatformTransactionManager`，能更精细控制局部边界，但代码更繁琐。事务管理的收益是统一边界、传播、隔离和回滚规则；前提是数据库、连接池和调用路径确实处在同一事务上下文中。

## 33. Spring MVC 与 Struts2 有什么异同？

两者都是 Web MVC 框架，都能做请求分发、参数绑定和视图渲染。Spring MVC 以 `DispatcherServlet` 为前端控制器，方法级 Controller 更贴近 Spring 生态；Struts2 以 Filter 为核心，Action 对象模型不同。新项目通常优先 Spring MVC，因为生态、整合与维护成本更有优势。
