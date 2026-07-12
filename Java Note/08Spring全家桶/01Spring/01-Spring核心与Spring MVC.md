# Spring 核心与 Spring MVC

> 本页吸收“Spring 刷题题库”中 Spring、MVC、Security 相关题目；同义题（如 IoC/DI、Bean 生命周期、AOP、事务）统一合并，避免重复。

## 1. 什么是 Spring，解决了什么问题？

Spring 的核心价值是把对象的创建、组装和通用能力交给框架处理，让业务代码专注于业务。它通过 IoC 管理对象，通过 AOP 承载事务、日志、安全等横切能力，并提供 MVC、数据访问等模块。

**面试一句话：**Spring 是一个轻量级 Java 企业开发框架，核心是 IoC 和 AOP，目的是解耦并复用基础能力。

从使用角度看，Spring 负责三层事情：用 IoC/DI 管对象，用 AOP/事务处理横切能力，用 MVC、数据访问、事件、国际化等模块解决常见企业开发问题。它不是要求所有能力都用，而是提供一套能按需组合的基础设施。

### 面试展开

Spring 的优势不是“帮你写业务”，而是提供统一的对象管理、事务、Web、数据访问与可扩展机制。实际项目里 Controller、Service、Repository 通常都是 Bean；日志、权限、事务不应散落在每个业务方法中，而应交给 AOP/框架基础设施统一处理。

## 2. 什么是 Bean？Bean 有哪些作用域？

Bean 是被 Spring IoC 容器创建和管理的对象。常用作用域：`singleton`（默认、容器内一个实例）、`prototype`（每次获取新实例）、Web 环境的 `request`、`session`、`application` 和 `websocket`。`globalSession` 主要服务于早期 Portlet 场景，现代 Servlet 应用通常无需使用。

⚠️ `singleton` 指“Spring 容器中单例”，不等于天然线程安全；无状态 Bean 通常安全，有可变共享字段时要自行处理并发。

### 作用域追问

`singleton` 由容器负责完整生命周期；`prototype` 每次获取新对象，但容器完成初始化后通常不再跟踪它的销毁。Web 作用域仅在 Web ApplicationContext 中有效：`request` 随一次请求存在，`session` 随会话存在，`application` 随 ServletContext 存在，`websocket` 随 WebSocket 会话存在。把用户态数据塞进单例字段是常见并发 bug。

## 3. Bean 的完整生命周期是怎样的？

典型流程是：实例化 → 注入属性 → 执行 `*Aware` 回调 → `BeanPostProcessor#postProcessBeforeInitialization` → 初始化（`@PostConstruct`、`afterPropertiesSet`、`init-method`）→ `postProcessAfterInitialization`（AOP 代理通常在此产生）→ 使用 → 容器关闭时执行 `@PreDestroy`、`destroy`、`destroy-method`。

### 按截图展开的关键顺序

1. **实例化 Bean**：创建对象。
2. **属性填充**：完成依赖注入。
3. **Aware 回调**：若实现了 `BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`，会依次获得 Bean 名称、`BeanFactory`、`ApplicationContext` 等容器信息。
4. **前置处理**：调用 `BeanPostProcessor#postProcessBeforeInitialization`。
5. **初始化**：可依次涉及 `@PostConstruct`、`InitializingBean#afterPropertiesSet` 与自定义 `init-method`。
6. **后置处理**：调用 `BeanPostProcessor#postProcessAfterInitialization`；AOP 代理通常在这一阶段包装完成。
7. **使用与销毁**：Bean 可被业务代码使用；容器关闭时依次执行 `@PreDestroy`、`DisposableBean#destroy` 与自定义 `destroy-method`。

**记忆：**生出来、装配好、前置加工、初始化、后置加工、使用、销毁。

### 易错点

`BeanFactoryPostProcessor` 处理的是 **BeanDefinition 元数据**，不是某一个 Bean 的初始化回调；`BeanPostProcessor` 才是对具体 Bean 初始化前后加工。AOP 代理通常与后置处理有关，因此注入到其他 Bean 的可能已经是代理对象。

## 4. IoC 是什么？IoC 容器的职责是什么？

IoC（控制反转）不是业务对象自己 `new` 依赖，而是把对象控制权交给容器。容器负责读取配置、创建 Bean、注入依赖、管理生命周期，并提供事件、资源、国际化等基础设施。

它的本质是依赖方向反转：业务类依赖抽象，组装工作由容器完成，因此替换实现和测试更容易。

容器通常先读取 XML、Java 配置或组件扫描结果形成 BeanDefinition，再按依赖关系创建、装配和初始化 Bean。业务代码通过注入使用依赖，而不是频繁调用 `getBean()`；后者会让业务代码重新依赖容器，只有框架扩展等少数场景才应使用。

### 容器类型

常见容器接口是 `BeanFactory` 与 `ApplicationContext`。后者是日常应用入口，除 IoC 外还提供事件、资源、国际化、环境抽象。IoC 的重点不是“所有对象都必须交给 Spring”，而是把需要被替换、协作复杂、需要事务/AOP 的对象交给容器管理。

## 5. DI 是什么？与 IoC 有什么区别？

DI（依赖注入）是 IoC 的具体实现方式：容器把一个对象所需的依赖注入进去。IoC 是思想，DI 是落地手段。

常见注入方式：构造器注入、Setter 注入、字段注入、方法参数注入。优先使用**构造器注入**：依赖明确、可以使用 `final`、便于单元测试；Setter 适合可选依赖；字段注入不利于测试且隐藏依赖。

当存在多个同类型 Bean 时，用 `@Qualifier` 指定名称，或用 `@Primary` 指定默认候选。DI 的价值不是“少写 `new`”，而是让接口与实现解耦，例如测试时能把真实仓储替换成 Mock 实现。

### 注入方式如何选

构造器注入适合必需依赖，能在创建时暴露循环依赖与缺失依赖；Setter 注入适合可选依赖或需要后续变更的属性；字段注入写得最少，但依赖不可见、难以单测。XML、注解、Java 配置只是描述装配关系的不同方式，运行时仍由容器完成注入。

## 6. Spring 如何解决循环依赖？为什么推荐三级缓存？

单例字段注入的循环依赖可借助三级缓存解决：一级缓存存完整 Bean，二级缓存存提前暴露的早期 Bean，三级缓存存 `ObjectFactory`。提前暴露工厂能在需要时生成早期引用；若 Bean 需要 AOP，工厂还能保证注入的是同一个早期代理。

⚠️ 构造器循环依赖无法靠此机制解决；`prototype` 循环依赖也不支持。业务上更应通过拆分职责消除循环依赖。

### 为什么是三级而不是二级

三级缓存保存的是创建早期引用的工厂，而不是直接保存早期对象。这样只有确实发生循环依赖时才创建早期代理，并能让“注入给依赖方的引用”和最终容器中的 AOP 代理保持一致；二级缓存难以同时兼顾延迟创建与代理一致性。

## 7. BeanFactory 与 ApplicationContext 有什么区别？

`BeanFactory` 是最基础的 IoC 容器，提供 Bean 获取与管理能力；`ApplicationContext` 在其基础上增加国际化、事件发布、资源加载、环境配置，以及多数单例 Bean 的预实例化。日常 Spring Boot 应用使用的是 `ApplicationContext`。

从取舍看，`BeanFactory` 更偏基础和按需获取，启动资源压力小；`ApplicationContext` 启动时会预实例化大多数单例 Bean，启动成本和内存占用更高，但能更早发现配置错误、运行时首次访问更稳定。不要把“ApplicationContext 一定加载所有 Bean”绝对化：懒加载、`prototype` 等仍有例外。

### 面试回答模板

`BeanFactory` 是基础 Bean 工厂，偏向按需获取；`ApplicationContext` 是企业应用常用上下文，增加事件、国际化、资源加载和环境能力，并预实例化大多数单例 Bean。两者不是“谁淘汰谁”，而是抽象层次不同；Spring Boot 默认使用 ApplicationContext。

## 8. 什么是 AOP？核心术语有哪些？

AOP 用来把日志、事务、权限、监控这类“多处都要做”的横切关注点从业务代码中抽离。`Aspect` 是切面，`Join Point` 是可增强位置，`Pointcut` 是匹配规则，`Advice` 是通知，`Target` 是目标对象，`Proxy` 是代理对象，`Weaving` 是织入过程；`Introduction`（引介）可为现有类型引入新的接口能力。

常用通知有 `@Before`、`@After`、`@AfterReturning`、`@AfterThrowing`、`@Around`；其中环绕通知能决定是否调用目标方法，能力最强。

实现方式分为运行时代理和织入两类：Spring AOP 主要使用运行时 JDK 动态代理或 CGLIB；AspectJ 还支持编译期织入和类加载期织入。JDK 代理要求目标实现接口，核心是 `Proxy` 与 `InvocationHandler`；CGLIB 通过生成子类代理，因此不能代理 `final` 类或 `final` 方法。

### AOP 的真实边界

Spring AOP 的连接点主要是 **Spring Bean 的方法执行**，不能像 AspectJ 那样随意拦截字段读写、构造器等。切面适合稳定的通用规则；把复杂业务判断塞进切面会让调用链难追踪，应留在 Service 中。

## 9. Spring AOP 与 AspectJ 有什么区别？

Spring AOP 基于运行时代理，主要拦截 Spring Bean 的方法执行，适合绝大多数业务切面；AspectJ 是更完整的 AOP 体系，可在编译期或类加载期织入，能覆盖构造器、字段等更多连接点。

Spring AOP 选择 JDK 动态代理（有接口）或 CGLIB（无接口）。⚠️ `final` 类/方法不能被 CGLIB 重写；同类内部调用不会经过代理，常导致 `@Transactional`、AOP 失效。

### 对比结论

Spring AOP 通常是运行时代理，配置和调试成本低，覆盖面有限；AspectJ 可编译期或类加载期织入，连接点更丰富，适合需要深度织入的场景。不要把 AspectJ 简化为“静态代理”，它的关键是织入时机和能力范围不同。

## 10. Spring 中常见设计模式有哪些？

工厂模式：`BeanFactory`、`FactoryBean`；单例模式：默认单例 Bean；代理模式：AOP；模板方法：`JdbcTemplate`、`RestTemplate`；观察者模式：事件监听；策略模式：资源、排序、不同实现选择；适配器模式：MVC 的 `HandlerAdapter`。

另外，Spring 的 `Resource` 包装、请求/响应包装等场景也能看到装饰器模式。面试不必把每个类强行归类，重点是说明模式如何让框架做到可扩展、可替换。

面试不要只报名称，最好补一句“它解决什么”：例如 `JdbcTemplate` 固化 JDBC 流程，让开发者只写 SQL 和结果映射。

### 设计模式要能说出用途

工厂模式负责创建与获取对象，代理模式让事务/AOP 在不改业务代码的前提下增强方法，模板模式固定资源处理流程，观察者模式让事件发布者和监听者解耦，适配器模式让 MVC 支持不同类型处理器。面试要优先讲“解决的问题”，不要只背类名。

## 11. Spring 事件和监听器如何工作？

发布者通过 `ApplicationEventPublisher` 发布事件，监听器用 `@EventListener` 或实现 `ApplicationListener` 接收。它适合把“下单成功后发通知、记日志”等非核心动作解耦。默认同步执行；需要异步要配合 `@Async` 或消息队列，并处理失败重试与事务边界。

### 事件使用边界

事件适合同一应用内的解耦协作。若监听动作必须与主业务一起成功，需关注事务时机，可使用 `@TransactionalEventListener` 指定在提交后执行；跨服务、要求可靠投递的场景则应使用消息队列与幂等消费，而不是只靠本地事件。

## 12. Spring 的声明式事务原理是什么？

`@Transactional` 本质是 AOP：代理在目标方法前开启或加入事务，正常返回时提交，异常时按规则回滚。真正的连接、提交、回滚由 `PlatformTransactionManager` 协调；JDBC 常用 `DataSourceTransactionManager`，JPA 常用 `JpaTransactionManager`。

除传播行为外，还可配置隔离级别来应对脏读、不可重复读、幻读。默认通常只对运行时异常和 `Error` 回滚；受检异常需要通过 `rollbackFor` 显式指定。隔离级别和回滚规则是事务语义的一部分，不应只背 `@Transactional` 注解。

**面试一句话：**声明式事务是代理拦截方法，再委托事务管理器绑定、提交或回滚当前线程资源。

### 事务属性要一起回答

一次事务定义通常包含传播行为、隔离级别、超时、只读标记和回滚规则。`readOnly=true` 是优化提示，不是禁止写入的安全机制；超时能避免连接长期占用；隔离级别最终还受数据库实现影响。

## 13. 事务的传播行为有哪些？

最常用的 `REQUIRED`：有事务就加入，没有就新建。`REQUIRES_NEW`：挂起外层事务并新建；`NESTED`：在当前事务内建立保存点；`SUPPORTS`、`NOT_SUPPORTED`、`NEVER`、`MANDATORY` 分别对应支持、非事务、禁止事务、必须已有事务。

⚠️ `REQUIRES_NEW` 的内外事务彼此独立；`NESTED` 依赖保存点，仍受外层最终提交/回滚影响。

实际选择口诀：绝大多数业务用 `REQUIRED`；必须独立提交的审计/日志才考虑 `REQUIRES_NEW`；需要局部回滚且数据源支持保存点时再考虑 `NESTED`。不要为“看起来高级”滥用新事务，它会增加连接占用和一致性复杂度。

### 传播行为对比

`REQUIRED` 是默认并加入外层事务；`SUPPORTS` 有就加入、无就非事务；`MANDATORY` 强制要求已有事务；`REQUIRES_NEW` 挂起外层并新开；`NOT_SUPPORTED` 挂起外层后非事务执行；`NEVER` 有事务即报错；`NESTED` 基于保存点实现局部回滚。传播行为解决的是“方法嵌套时事务边界如何组合”，不是数据库隔离级别。

## 14. 哪些情况会导致 @Transactional 失效？

常见原因：同类内部调用绕过代理；方法不是 `public`（默认代理配置下）；异常被吞掉；抛出受检异常但未配置 `rollbackFor`；Bean 不由 Spring 管理；事务方法在初始化阶段调用；使用了不受代理支持的 `final` 限制。排查时先确认“调用是否经过 Spring 代理”。

### 排查事务不生效的顺序

先看调用是否经过代理（自调用最常见）；再看方法可见性、Bean 是否受 Spring 管理、异常是否被吞掉或类型不匹配；最后检查事务管理器、数据源、数据库引擎和传播行为。不要把所有问题都归因于 `@Transactional` 注解“没扫描到”。

## 15. Spring MVC 的请求处理流程是什么？

请求进入 `DispatcherServlet`，再由 `HandlerMapping` 找到处理器，由 `HandlerAdapter` 调用 Controller；Controller 返回 `ModelAndView`、视图名或响应体；视图场景交给 `ViewResolver` 渲染，`@ResponseBody`/`@RestController` 则由 `HttpMessageConverter` 写回 JSON。`HandlerAdapter` 的意义是让 DispatcherServlet 不必绑定某一种 Controller 实现。

**记忆：**前端控制器找处理器、适配器调用、视图解析或消息转换返回。

常见九大协作组件包括：`HandlerMapping`、`HandlerAdapter`、`HandlerExceptionResolver`、`ViewResolver`、`RequestToViewNameTranslator`、`LocaleResolver`、`ThemeResolver`、`MultipartResolver`、`FlashMapManager`。并非每个请求都会显式用到全部组件；`DispatcherServlet` 的职责是协调它们，而不是自己承担业务逻辑。

### 请求链路中的异常

Controller 抛出异常后，`HandlerExceptionResolver` 会尝试把异常解析成响应；文件上传由 `MultipartResolver` 在进入 Controller 前处理；国际化解析和 Flash 属性也由专门组件协作。理解这条链路后，拦截器、全局异常和 JSON 返回的位置就不会混淆。

## 16. @Controller、@RestController、@RequestMapping 有什么区别？

`@Controller` 用于 MVC 控制器，返回字符串默认会被当作视图名；`@RestController` 等于 `@Controller + @ResponseBody`，返回值直接写入响应体，常用于 REST API。`@RequestMapping` 可定义路径、HTTP 方法、参数和请求头条件；`@GetMapping` 等是它的语义化快捷写法。

### 映射细节

`@RequestMapping` 可标在类上作为 URI 前缀，也可标在方法上声明路径、方法、请求头、参数等条件；`@GetMapping`、`@PostMapping` 等是指定 HTTP 方法的语义化快捷注解。前后端分离接口通常用 `@RestController`，服务端渲染页面才更多使用 `@Controller` + 视图名。

## 17. @RequestParam、@PathVariable、@RequestBody 如何选择？

`@RequestParam` 获取查询参数或表单参数，如 `/users?page=1`；`@PathVariable` 获取资源路径变量，如 `/users/{id}`；`@RequestBody` 把 JSON 请求体反序列化为对象。REST 风格中，资源标识优先放路径，筛选/分页放查询参数，复杂写入数据放请求体。

### 参数绑定易错点

`@RequestParam` 默认来自 query/form 参数，常用于分页、筛选等；`@PathVariable` 代表资源身份；`@RequestBody` 消费整个请求体，通常一个方法只应有一个。路径变量和查询参数都是字符串到目标类型的转换，格式不合法会在参数绑定阶段报错，应进入统一异常处理。

## 18. Spring MVC 如何返回 JSON、做参数校验和统一异常处理？

返回 JSON 依赖 `HttpMessageConverter`（常见 Jackson）；参数对象标注 `@Valid`/`@Validated` 并在字段上加校验注解；使用 `@RestControllerAdvice + @ExceptionHandler` 统一转换异常为约定响应体。

⚠️ 校验结果要显式处理 `BindingResult` 或让异常进入全局处理器；不要把数据库异常原样返回给前端。

### 统一响应建议

成功响应、参数错误、业务异常、未认证、无权限、系统异常应有稳定的错误码和结构。全局处理器只负责“翻译异常”，不要在其中吞掉日志或把数据库/堆栈细节返回客户端。JSON 的实际序列化由 `HttpMessageConverter` 完成，Spring MVC 常用 Jackson 的 `MappingJackson2HttpMessageConverter`。

## 19. Filter、HandlerInterceptor、AOP 有什么区别？

Filter 属于 Servlet 规范，最靠近 HTTP 容器，适合编码、CORS、通用请求包装；Interceptor 位于 Spring MVC 调用 Controller 前后，适合登录校验、接口审计；AOP 面向方法调用，适合事务、日志和权限等业务横切逻辑。三者不是替代关系，按作用层次选择。

Interceptor 可通过实现 `HandlerInterceptor` 的 `preHandle`、`postHandle`、`afterCompletion` 并在 `WebMvcConfigurer#addInterceptors` 注册；它能拿到 `Handler`，因而比 Filter 更了解将要执行的 Controller。Filter 可通过容器或 `FilterRegistrationBean` 注册，通常覆盖所有经过 Servlet 容器的请求。

### 如何选择

编码、请求包装、CORS 等 Servlet 层问题用 Filter；只针对 MVC Handler 的鉴权、审计、限速预处理用 Interceptor；方法级横切规则如事务、耗时统计用 AOP。Interceptor 的 `preHandle`、`postHandle`、`afterCompletion` 分别对应处理前、视图渲染前、请求完成后。

## 20. Spring Security 的认证与授权流程是什么？

请求经过安全过滤器链，认证阶段确认“你是谁”（账号密码、Token、OAuth2 等）并生成 `Authentication`；授权阶段根据角色、权限或表达式判断“你能做什么”。认证结果通常存入 `SecurityContext`。账号密码场景中，`UserDetailsService` 负责按用户名加载用户信息，`PasswordEncoder` 负责密码哈希与匹配；密码绝不能明文存储。前后端分离项目常采用无状态 Token，并配置 CORS、CSRF 策略和异常响应。

### 安全链路追问

认证成功不代表一定有权限：认证建立 `Authentication`，授权再根据 URL、角色、权限或方法注解决定放行。401 通常表示未认证或凭证无效，403 通常表示已认证但无权限。使用 JWT 也仍需要处理签名校验、过期、撤销策略、权限变更和密钥轮换。

## 21. OAuth2 是什么？

OAuth2 是授权框架，不等于登录协议本身。它让用户授权第三方在限定范围内访问资源，而不必交出密码。常见角色：资源所有者、客户端、授权服务器、资源服务器；常见授权方式包含授权码模式。实际项目中常与 OpenID Connect 组合完成身份认证。

当前 Web 与移动端通常优先采用“授权码 + PKCE”；不要把已经不推荐的密码模式用于新系统。OAuth2 解决授权委托，OIDC 在其上补充身份层；二者经常一起出现，但含义不同。

### OAuth2 角色与流程

资源所有者授权客户端，授权服务器签发访问令牌，客户端携带令牌访问资源服务器。令牌应限制 scope、有效期与受众；资源服务器只接受可信签发方的令牌。前后端分离场景不能把 client secret 放进浏览器或移动端。

## 22. 如何理解 REST API，以及接口如何保证安全？

REST 以资源为中心，用 URI 标识资源、HTTP 方法表达操作（GET 查、POST 建、PUT/PATCH 改、DELETE 删），保持无状态并使用合适状态码。接口安全需使用 HTTPS、认证授权、输入校验、限流、防重放与日志审计；写接口要考虑幂等性，敏感信息不能放 URL 或日志。

**安全（safe）和幂等不要混淆：**GET、HEAD 约定上不改变服务端资源，因此是安全方法；PUT、DELETE 通常是幂等的，但会改变资源，不能称为安全；POST 通常既不安全也不幂等。REST 的无状态是指服务器不依赖某个请求之外的会话上下文来理解当前请求，不是说服务端不能有数据库或缓存。

### REST 接口检查清单

URI 用名词表示资源，HTTP 状态码表达结果；分页、排序、过滤尽量通过查询参数表达；写操作要有鉴权、校验、幂等与审计；全站使用 HTTPS。REST 是一种架构风格，不是“URL 里没有动词”就自动 RESTful。

## 23. Spring Data JPA 的工作原理是什么？

Spring Data JPA 在 JPA 规范之上提供 Repository 抽象。应用定义实体和 Repository 接口后，Spring 在启动时为接口创建代理；代理根据方法名推导查询，或执行 `@Query` 指定的 JPQL/SQL，再交给 JPA 实现（如 Hibernate）完成 EntityManager、SQL 与结果映射工作。

⚠️ Repository 很省代码，但复杂查询仍要关注 SQL、索引和 N+1 问题，不能把性能问题藏在方法名后面。

### Repository 的边界

派生查询适合简单条件；复杂多表、性能敏感查询应明确写 `@Query`、Specification、原生 SQL 或转用 MyBatis。实体状态、一级缓存、延迟加载、事务边界共同决定最终 SQL，排查性能一定要看真实 SQL 与执行计划。

## 24. Spring 如何实现国际化（i18n）？

配置 `MessageSource` 和不同语言的资源文件（如 `messages_zh_CN.properties`、`messages_en_US.properties`），再依据请求 Locale 解析并获取对应文案。Web 场景可结合 `LocaleResolver`、请求头或参数切换语言。国际化的对象是文案，不应把业务分支复制成多套语言代码。

### 国际化实践

把面向用户的文本抽到资源文件，通过 message key 获取，避免在代码中散落中文/英文常量。Locale 可来自 `Accept-Language`、Cookie、Session 或参数解析器；接口错误消息也应使用同一套 message key，便于多语言维护。

## 25. JdbcTemplate 是什么，解决了什么问题？

`JdbcTemplate` 通常由 `DataSource` 创建/注入，它把 JDBC 中获取连接、创建语句、执行、关闭资源、转换异常这些固定流程封装起来。常用 `query`、`update` 执行查询和更新；查询结果可交给 `RowMapper` 逐行映射，或由 `ResultSetExtractor` 整体处理。它仍是直接操作 JDBC，适合简单、可控的 SQL 场景；复杂动态 SQL 可考虑 [[01-MyBatis]]。

### JdbcTemplate 使用边界

它负责连接获取、资源关闭与异常翻译，但不会替你设计 SQL 或索引。参数必须使用占位符绑定，不能字符串拼接；批量写入可使用 `batchUpdate`；事务边界仍由 Spring 事务管理器控制。需要动态 SQL 或复杂映射时，使用 [[01-MyBatis]] 往往更直观。

## 26. Spring 的数据访问异常层次结构有什么价值？

Spring 把不同数据库和 ORM 的底层异常翻译为统一、非受检的 `DataAccessException` 层次结构。`DataRetrievalFailureException` 常表示数据获取失败，`InvalidDataAccessResourceUsageException` 常表示 SQL/资源使用不当，此外还有数据完整性冲突、乐观锁失败、查询结果不正确等类别。业务代码因此不必依赖某个 JDBC 驱动的异常类型，也更容易判断异常是否可重试。

### 为什么异常翻译重要

底层 JDBC、Hibernate、JPA 的异常类型各不相同，业务层若直接依赖它们就会被实现绑定。统一的 `DataAccessException` 是非受检异常，既便于事务默认回滚，也能按“可重试、完整性冲突、资源错误”等语义分类处理。

## 27. Spring MVC 与 Spring WebFlux 有什么区别？

Spring MVC 基于 Servlet API，通常采用“一请求一线程”的阻塞式编程模型，生态成熟且适合大多数 CRUD 应用；WebFlux 基于 Reactor，支持非阻塞 I/O 和背压，适合大量慢 I/O、长连接、高并发场景。

⚠️ WebFlux 不是性能开关：内部仍调用阻塞 JDBC、文件 I/O 或同步远程调用时，会堵塞事件循环，反而得不偿失。

### 选型原则

MVC 对阻塞式 Servlet、JDBC、传统 CRUD 非常合适；WebFlux 的价值在于端到端非阻塞链路和高连接数，而不是把同步项目改成 `Mono`/`Flux` 就更快。一个应用可以同时理解两者，但不要在同一请求链里随意混用阻塞与事件循环线程。

## 28. Spring 中的 Template 模式体现在哪里？

模板方法模式把稳定流程固定在父类或模板中，把变化点交给回调。`JdbcTemplate` 固化 JDBC 流程，`RestTemplate`/WebClient 封装请求流程，事务模板 `TransactionTemplate` 封装事务边界。模板负责资源的打开和关闭、统一异常转换与常规流程，调用者只实现变化部分；因此既减少样板代码，也避免遗漏资源释放。

### 模板方法的本质

模板类把“变化少且容易出错”的资源处理流程封装起来，再通过回调暴露“变化多”的 SQL、映射或业务步骤。这样既避免重复 `try/finally`，又统一异常转换；调用者仍需要为回调逻辑负责。

## 29. Spring Security 过滤器链是什么？

Spring Security 把认证、鉴权、异常转换、CSRF 等安全工作拆成多个 Filter，按顺序组成过滤器链。请求先经过链，再进入 `DispatcherServlet`；其中某个过滤器可以直接拒绝请求或建立认证上下文。排查 401/403 时，应先看请求命中了哪条 `SecurityFilterChain`、认证信息是否建立、授权规则是否匹配。

自定义认证 Filter 应插入在合适的内置 Filter 前后，而不是随意注册到 Servlet Filter 链；否则可能绕过异常处理或认证上下文。配置时用 `HttpSecurity` 建立 `SecurityFilterChain`，再通过 `addFilterBefore`/`addFilterAfter` 明确顺序。

### Filter 链调试

出现登录接口也被拦截、Token 已带却仍 401、异常响应格式不统一等问题时，优先确认请求匹配到的 `SecurityFilterChain` 及 Filter 顺序。安全 Filter 在 `DispatcherServlet` 前执行，因此 Controller 的 `@ExceptionHandler` 不一定能处理安全链路中的异常，通常还需要配置认证/授权失败处理器。

## 30. 如何使用 AOP 实现日志记录？

用切点匹配 Controller 或 Service 方法，在 `@Around` 通知中记录方法名、关键参数、耗时、结果或异常，并调用 `proceed()` 执行目标方法。日志要注意脱敏、控制体积和避免重复记录；不要在切面里序列化巨大对象或读取一次性请求流。

使用注解方式时，切面类标记 `@Aspect + @Component`，配置类启用 `@EnableAspectJAutoProxy`（Spring Boot 的 AOP Starter 通常会完成相关自动配置），再用 `@Pointcut` 复用匹配表达式。`@Around` 中必须调用 `proceed()` 才会执行目标方法；漏调会导致业务方法根本不执行。

### 日志切面实践

建议记录 traceId、方法、关键业务标识、耗时和异常类型；密码、身份证、Token、完整请求体必须脱敏或禁止记录。日志切点应尽量限定在业务包，避免把框架内部方法全部打出；性能统计可配合 Micrometer，而不是只依赖手写日志。

## 31. Spring 有哪些配置和装配方式？如何声明 Bean？

配置方式包括 XML、Java 配置（`@Configuration + @Bean`）和注解扫描。常用组件注解有 `@Component`、`@Service`、`@Repository`、`@Controller`；装配可通过构造器、`@Autowired`、`@Qualifier` 和 `@Primary` 解决候选 Bean 选择问题。现代项目优先 Java 配置和构造器注入，XML 主要用于维护旧项目。

内部 Bean 是仅作为另一个 Bean 属性或构造器参数存在的匿名 Bean，常见于旧 XML 中嵌套的 `<bean>`；它不能像顶级 Bean 那样按名字复用，通常可视为随所属 Bean 创建的内部对象。新项目更常用 Java 对象组合或独立 Bean 取代它。

## 32. Spring 事务有哪些实现方式？各有什么特点？

声明式事务用 `@Transactional`，侵入业务代码少，是默认首选；编程式事务使用 `TransactionTemplate` 或直接调用 `PlatformTransactionManager`，能更精细控制局部边界，但代码更繁琐。事务管理的收益是统一边界、传播、隔离和回滚规则；前提是数据库、连接池和调用路径确实处在同一事务上下文中。

`PlatformTransactionManager` 是事务管理抽象：负责取得事务状态、提交和回滚，并协调连接等资源。JDBC 常用 `DataSourceTransactionManager`；JPA 常用 `JpaTransactionManager`。选择实现必须与实际持久化技术匹配，否则注解看似生效却无法正确管理资源。

## 33. Spring MVC 与 Struts2 有什么异同？

两者都是 Web MVC 框架，都能做请求分发、参数绑定和视图渲染。Spring MVC 以 `DispatcherServlet` 为前端控制器，方法级 Controller 更贴近 Spring 生态；Struts2 以 Filter 为核心，Action 对象模型不同。新项目通常优先 Spring MVC，因为生态、整合与维护成本更有优势。
