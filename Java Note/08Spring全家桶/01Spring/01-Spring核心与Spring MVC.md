# Spring 核心与 Spring MVC

> 本页吸收“Spring 刷题题库”中 Spring、MVC、Security 相关题目；同义题（如 IoC/DI、Bean 生命周期、AOP、事务）统一合并，避免重复。

## Spring 知识地图（先知道都是什么）

把 Spring 想成 Java 后端项目的“总后勤”。业务代码只关心下单、查用户、扣库存；Spring 帮你管理对象、连接数据库、处理 HTTP 请求、加事务、做权限校验等通用工作。

| 模块 | 它解决什么问题 | 大白话理解 |
| --- | --- | --- |
| IoC / Bean | 对象谁创建、谁管理 | 不用到处 `new`，对象交给 Spring 统一管理。 |
| DI | 对象之间怎么合作 | Spring 自动把 `Repository` 交给需要它的 `Service`。 |
| Bean 生命周期 / 作用域 | 对象何时创建、能活多久 | Spring 负责对象从“出生、上岗”到“下班销毁”。 |
| AOP | 日志、权限、事务等重复代码 | 给业务方法统一外挂一层自动流程。 |
| 事务 | 多步数据库操作如何一起成功/失败 | 转账的扣钱和加钱，要么都成功，要么都撤销。 |
| Spring MVC | 浏览器请求如何到 Controller | DispatcherServlet 像总调度台，把请求送到正确接口。 |
| 数据访问 | 怎么更方便操作数据库 | JdbcTemplate、JPA 等减少 JDBC 样板代码。 |
| Spring Security | 登录、认证、权限 | 请求先过安检：你是谁、你有没有权限访问。 |
| 事件 | 模块之间如何解耦通知 | 下单后通知积分、消息等，不必让订单代码直接依赖所有模块。 |
| 国际化 | 同一功能显示不同语言 | 中文用户看到中文，英文用户看到英文，业务逻辑不复制。 |

### 推荐理解顺序

先学 **Bean → IoC → DI**，知道对象怎么被管理；再学 **AOP → 事务**，知道通用能力如何自动加到业务上；接着学 **MVC → 数据访问 → Security**，把一个完整后端接口串起来；最后补事件、国际化、WebFlux 等扩展能力。

## 1. 什么是 Spring，解决了什么问题？
![[Pasted image 20260712163925.png|508]]
Spring 的核心价值是把对象的创建、组装和通用能力交给框架处理，让业务代码专注于业务。它通过 IoC 管理对象，通过 AOP 承载事务、日志、安全等横切能力，并提供 MVC、数据访问等模块。

**面试一句话：**Spring 是一个轻量级 Java 企业开发框架，核心是 IoC 和 AOP，目的是解耦并复用基础能力。

从使用角度看，Spring 负责三层事情：用 IoC/DI 管对象，用 AOP/事务处理横切能力，用 MVC、数据访问、事件、国际化等模块解决常见企业开发问题。它不是要求所有能力都用，而是提供一套能按需组合的基础设施。

### Spring 常用术语速查

| 术语 | 大白话解释 |
| --- | --- |
| `Bean` | 被 Spring 容器创建、组装和管理的 Java 对象。 |
| `Bean id` / Bean name | Bean 在容器里的名字，相当于员工编号；用来区分和获取 Bean。默认常由类名生成，也可显式指定。 |
| `IoC` | 控制反转：对象不再自己 `new` 依赖，交给 Spring 管。 |
| `DI` | 依赖注入：Spring 把一个对象需要的依赖对象交给它。 |
| `IoC 容器` | 管理 Bean 的“总管”，常见是 `ApplicationContext`。 |
| `BeanDefinition` | Spring 读取到的 Bean 配置说明书，记录类、作用域、依赖、初始化方法等信息。 |
| `Component Scan` | 扫描带 `@Component`、`@Service` 等注解的类，并注册为 Bean。 |
| `@Autowired` | 让 Spring 按类型自动注入依赖。 |
| `@Qualifier` / `@Primary` | 多个同类型 Bean 时，指定用哪一个 / 指定默认用哪一个。 |
| `Scope` | Bean 的作用范围，例如单例、每次新建、每个请求一个。 |
| `AOP` | 给多个业务方法统一外挂日志、事务、权限等流程。 |
| `Proxy` | Spring 创建的代理替身，外部通常先调用它，再由它决定是否执行 AOP 流程。 |
| `TransactionManager` | 事务管理员，负责开启、提交、回滚事务。 |
| `DispatcherServlet` | Spring MVC 的总入口，负责把 HTTP 请求分发给对应 Controller。 |
| `ApplicationContext` | 功能更完整的 IoC 容器，支持 Bean、事件、国际化、资源加载等。 |

⚠️ `id` 不是对象的 Java 内存地址，也不是数据库主键；它只是 Bean 在 Spring 容器中的标识。实际项目里通常不手动按 id 获取 Bean，而是优先通过构造器注入。

### 面试展开

Spring 的优势不是“帮你写业务”，而是提供统一的对象管理、事务、Web、数据访问与可扩展机制。实际项目里 Controller、Service、Repository 通常都是 Bean；日志、权限、事务不应散落在每个业务方法中，而应交给 AOP/框架基础设施统一处理。

## 2. 什么是 Bean？Bean 有哪些作用域？
![[Pasted image 20260712163900.png|515]]
### 先用大白话理解
![[Pasted image 20260712164505.png|485]]

可以把 Spring 想成一家公司，Bean 就是被这家公司统一管理的“员工”。

- 普通 Java 对象：你自己 `new` 出来、自己给它找依赖、自己决定什么时候不用。
- Spring Bean：你告诉 Spring“这个类需要被管理”，Spring 负责创建它、把它需要的对象注入进去、在合适的时机初始化，并在容器关闭时清理它。

例如 `OrderService` 需要 `OrderRepository`。不使用 Spring 时，通常要自己写 `new OrderRepository()` 再传给 `OrderService`；使用 Spring 后，两者都交给容器，Spring 会创建并组装好它们。**所以 Bean 本质上仍是 Java 对象，只是它的生命周期和依赖关系由 Spring IoC 容器管理。**

Bean 是被 Spring IoC 容器创建和管理的对象。常用作用域：`singleton`（默认、容器内一个实例）、`prototype`（每次获取新实例）、Web 环境的 `request`、`session`、`application` 和 `websocket`。`globalSession` 主要服务于早期 Portlet 场景，现代 Servlet 应用通常无需使用。

⚠️ `singleton` 指“Spring 容器中单例”，不等于天然线程安全；无状态 Bean 通常安全，有可变共享字段时要自行处理并发。

### 作用域追问
![[Pasted image 20260712164545.png|455]]
`singleton` 由容器负责完整生命周期；`prototype` 每次获取新对象，但容器完成初始化后通常不再跟踪它的销毁。Web 作用域仅在 Web ApplicationContext 中有效：`request` 随一次请求存在，`session` 随会话存在，`application` 随 ServletContext 存在，`websocket` 随 WebSocket 会话存在。把用户态数据塞进单例字段是常见并发 bug。

### 单例 Bean 为什么可能线程不安全

Web 服务通常会让多个请求线程同时调用同一个单例 Service。若 Service 只有方法参数、局部变量和注入的线程安全依赖，每个线程用自己的局部数据，通常安全；若它把“当前用户、当前订单、临时查询结果、计数器”等放进**可变成员变量或静态变量**，多个线程就会互相覆盖数据。

```java
@Service
class OrderService {
    private Long currentUserId; // ❌ 多请求共享，可能串数据

    public void create(Long userId) {
        this.currentUserId = userId;
    }
}
```

更安全的写法是把 `userId` 放在方法参数或局部变量中，而不是 Bean 字段中。

### 有状态、无状态与 Prototype 的真实关系

| 类型 | 含义 | 线程安全结论 |
| --- | --- | --- |
| 无状态 Bean | 不保存会随请求变化的实例字段 | 单例复用通常安全，最常见 |
| 有状态 Bean | 保存可变的请求/用户/业务中间状态 | 单例下容易并发冲突，需要重新设计或隔离状态 |
| Prototype Bean | 每次从容器获取时创建新实例 | 能减少“同一实例共享”的问题，但不自动保证内部依赖、集合或静态变量线程安全 |

⚠️ 不要把 Prototype 当成线程安全万能药：若 Prototype 被注入单例 Bean，它通常只在注入时创建一次；需要每次动态获取时要用 `ObjectProvider`、`@Lookup` 等方式。多数业务 Service 最好的做法仍是保持无状态。

### ThreadLocal 能做什么，不能做什么

`ThreadLocal` 能让同一线程保存自己的上下文，例如请求链路 ID、当前租户信息；它不是共享状态的通用替代品。在线程池环境必须在请求结束后 `remove()`，否则线程复用可能造成数据泄漏。优先使用显式方法参数传递业务数据，只有确实是线程上下文时才使用 ThreadLocal。

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

### 先用大白话理解

依赖注入就是：**一个对象需要帮手时，不自己去创建帮手，而是让 Spring 把帮手送过来。**

例如 `OrderService` 需要 `OrderRepository`。不用 Spring 时，`OrderService` 里会自己 `new OrderRepository()`；用了 DI 后，`OrderService` 只声明“我需要 Repository”，具体实现由 Spring 创建后注入。这样未来把 MySQL 实现替换成测试 Mock，不必改业务类。

DI（依赖注入）是 IoC 的具体实现方式：容器把一个对象所需的依赖注入进去。IoC 是思想，DI 是落地手段。

常见注入方式：构造器注入、Setter 注入、字段注入、方法参数注入。优先使用**构造器注入**：依赖明确、可以使用 `final`、便于单元测试；Setter 适合可选依赖；字段注入不利于测试且隐藏依赖。

当存在多个同类型 Bean 时，用 `@Qualifier` 指定名称，或用 `@Primary` 指定默认候选。DI 的价值不是“少写 `new`”，而是让接口与实现解耦，例如测试时能把真实仓储替换成 Mock 实现。

### 注入方式怎么选
![[Pasted image 20260712164131.png|531]]

| 方式        | 写法特点                   | 适合场景                        |
| --------- | ---------------------- | --------------------------- |
| 构造器注入     | 依赖放在构造器参数中             | **首选**；必需依赖、可用 `final`、便于测试 |
| Setter 注入 | 通过 `setXxx` 注入         | 可选依赖、运行时允许替换的依赖             |
| 字段注入      | 直接标在成员变量               | 写法短，但依赖隐藏、单测不便，不推荐作为默认选择    |
| 方法参数注入    | 在 `@Bean` 方法或配置方法参数中注入 | Java 配置、装配多个 Bean 时常用       |

截图里提到的“接口注入”不是 Spring 项目中最常规的 DI 分类。`BeanNameAware`、`ApplicationContextAware` 等接口用于让 Bean 感知容器信息，属于特殊回调能力；业务依赖仍应优先构造器注入。

### 注入方式如何选

构造器注入适合必需依赖，能在创建时暴露循环依赖与缺失依赖；Setter 注入适合可选依赖或需要后续变更的属性；字段注入写得最少，但依赖不可见、难以单测。XML、注解、Java 配置只是描述装配关系的不同方式，运行时仍由容器完成注入。

## 6. Spring 如何解决循环依赖？为什么推荐三级缓存？

### 先用大白话理解

假设 A 需要 B，B 也需要 A：

```text
A 创建时：先给我 B
B 创建时：先给我 A
```

两边都在等对方“完全创建好”，就会卡住。Spring 的解法不是一次把对象全部做完再交出去，而是：**对象刚创建出外壳时，先把“早期引用”临时交给对方，等属性填充和初始化结束后再换成完整对象。**

单例字段注入的循环依赖可借助三级缓存解决：一级缓存存完整 Bean，二级缓存存提前暴露的早期 Bean，三级缓存存 `ObjectFactory`。提前暴露工厂能在需要时生成早期引用；若 Bean 需要 AOP，工厂还能保证注入的是同一个早期代理。

⚠️ 构造器循环依赖无法靠此机制解决；`prototype` 循环依赖也不支持。业务上更应通过拆分职责消除循环依赖。

### 为什么是三级而不是二级

三级缓存保存的是创建早期引用的工厂，而不是直接保存早期对象。这样只有确实发生循环依赖时才创建早期代理，并能让“注入给依赖方的引用”和最终容器中的 AOP 代理保持一致；二级缓存难以同时兼顾延迟创建与代理一致性。

### 截图中的三种情况

1. **构造器循环依赖**：A 的构造器必须拿到 B，B 的构造器又必须拿到 A。对象外壳都还没创建，无法提前暴露，通常抛出 `BeanCurrentlyInCreationException`。
2. **单例 Bean 的 Setter / 字段循环依赖**：默认单例、且可先实例化再注入属性时，Spring 可以用三级缓存处理。
3. **`prototype` 循环依赖**：每次都要新建对象，容器不维护可复用的单例缓存，通常无法处理。

### 单例 Setter 循环依赖流程

以 `A → B → A` 为例：

1. **实例化 A**：执行 `createBeanInstance`，此时只是创建 A 的对象外壳，还没填充依赖。
2. **提前暴露 A**：把能生成 A 早期引用的 `ObjectFactory` 放入三级缓存 `singletonFactories`。
3. **填充 A 属性**：发现 A 依赖 B，于是开始创建 B。
4. **实例化并填充 B**：B 又依赖 A，此时一级缓存没有完整 A，二级缓存也没有早期 A，于是从三级缓存取出 A 的工厂，得到 A 的早期引用。
5. **完成 B**：把 A 的早期引用注入 B，B 初始化完成，进入一级缓存 `singletonObjects`。
6. **完成 A**：把完整的 B 注入 A，A 初始化完成，最终进入一级缓存；临时二级/三级缓存记录被清理。

### 三个缓存各放什么

| 缓存 | 名称 | 放的内容 |
| --- | --- | --- |
| 一级 | `singletonObjects` | 完整初始化完成的单例 Bean |
| 二级 | `earlySingletonObjects` | 已提前暴露、但未完全初始化的早期 Bean 引用 |
| 三级 | `singletonFactories` | 生成早期引用的 `ObjectFactory`，必要时可返回早期 AOP 代理 |

⚠️ 三级缓存是框架兜底机制，不是设计循环依赖的理由。业务类互相依赖通常说明职责没拆好，优先抽取第三个服务、引入接口或重新划分职责；不要为了让 Spring “能启动”而保留复杂环。

## 7. BeanFactory 与 ApplicationContext 有什么区别？

### 先用大白话理解

两者都能管理 Bean。`BeanFactory` 像最基础的“对象仓库”：能按需拿到和创建对象；`ApplicationContext` 像完整的“应用管理中心”：除了对象，还提供事件通知、国际化、资源读取、环境配置等企业应用常用能力。

日常 Spring Boot 项目基本使用 `ApplicationContext`，不是因为 BeanFactory 没用，而是 Boot 应用通常需要完整的基础设施。

![[Pasted image 20260712154731.png|529]]
`BeanFactory` 是最基础的 IoC 容器，提供 Bean 获取与管理能力；`ApplicationContext` 在其基础上增加国际化、事件发布、资源加载、环境配置，以及多数单例 Bean 的预实例化。日常 Spring Boot 应用使用的是 `ApplicationContext`。

从取舍看，`BeanFactory` 更偏基础和按需获取，启动资源压力小；`ApplicationContext` 启动时会预实例化大多数单例 Bean，启动成本和内存占用更高，但能更早发现配置错误、运行时首次访问更稳定。不要把“ApplicationContext 一定加载所有 Bean”绝对化：懒加载、`prototype` 等仍有例外。

### 面试回答模板

`BeanFactory` 是基础 Bean 工厂，偏向按需获取；`ApplicationContext` 是企业应用常用上下文，增加事件、国际化、资源加载和环境能力，并预实例化大多数单例 Bean。两者不是“谁淘汰谁”，而是抽象层次不同；Spring Boot 默认使用 ApplicationContext。

### 截图中的五点对比

| 对比点 | BeanFactory | ApplicationContext |
| --- | --- | --- |
| Bean 实例化时机 | 通常偏按需获取/创建 | 通常在启动时预实例化大多数单例 Bean |
| 功能范围 | 基础 Bean 管理 | Bean 管理 + 国际化、事件、资源加载等 |
| AOP 与常用集成 | 能作为基础容器，但常需额外组合配置 | 提供更完整的应用上下文支持，日常使用更方便 |
| 环境抽象 | 基础能力为主 | 可通过 `Environment` 读取 Profile、配置属性、环境变量 |
| 注解/组件扫描 | 可支持，但通常需要额外注册相关后处理器 | 常见上下文会整合组件扫描、注解配置等能力 |

⚠️ “ApplicationContext 启动时实例化所有 Bean”是为了理解而做的简化：懒加载 Bean、`prototype` Bean 等例外并不会按同样方式预创建。也不要把“支持 AOP”理解成 BeanFactory 完全不能用 AOP，真正差异是上下文提供的整合能力与默认体验。

### 优缺点如何正确理解

BeanFactory 的基础、按需特征在资源非常受限或框架底层场景有意义；ApplicationContext 提前创建大多数单例 Bean，启动阶段会多花时间和内存，却能更早暴露配置错误、在运行期首次使用时更稳定。现代 Spring Boot 应用通常直接选择 ApplicationContext，不需要为了“省内存”刻意退回 BeanFactory。

## 8. 什么是 AOP？核心术语有哪些？

### 先用大白话理解

可以把 AOP 理解为：**不给业务代码动手术，而是在它执行的前后统一加一层“自动流程”。**

例如下单、退款、查询等很多方法都要做“记录日志、检查权限、开启事务、统计耗时”。如果每个方法都手写这些代码，业务代码会重复且难维护。AOP 会先按规则找到这些方法，再在执行前、执行后或抛异常时自动加入通用逻辑；业务方法本身仍只关注“下单成功没有、退款规则对不对”。

类比一下：业务方法像餐厅后厨做菜；AOP 像统一的前台和质检流程——进厨房前登记、做完后结账统计、出问题时上报。它不负责炒菜，却能让每道菜都遵守同一套流程。

### AOP 术语翻译成大白话

| 术语 | 大白话 | 餐厅类比 |
| --- | --- | --- |
| `Target`（目标对象） | 原本要执行的业务对象 | 后厨/厨师 |
| `Join Point`（连接点） | 理论上可以插入额外动作的位置 | 一道菜开始做、做完、出错的时刻 |
| `Pointcut`（切点） | 规定“到底挑哪些位置加流程”的筛选规则 | “所有退款菜品都要质检”这条规则 |
| `Advice`（通知） | 真正要执行的额外动作 | 登记、质检、记账这件事 |
| `Aspect`（切面） | 一整套规则加动作的组合 | “质检制度”本身：挑谁检查 + 怎么检查 |
| `Proxy`（代理对象） | Spring 包装后的替身，外部实际调用它 | 前台接单后再交给后厨 |
| `Weaving`（织入） | 把额外流程接进业务调用链的过程 | 把质检环节接入出餐流程 |
| `Introduction`（引介） | 给已有对象额外增加接口能力 | 给普通厨师额外挂上“可接收质检”的能力 |

**最容易混：**`Pointcut` 是“选谁”，`Advice` 是“做什么”，`Aspect` 是“选谁 + 做什么”合起来的一整套方案。

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
![[Pasted image 20260712155641.png|501]]
工厂模式：`BeanFactory`、`FactoryBean`；单例模式：默认单例 Bean；代理模式：AOP；模板方法：`JdbcTemplate`、`RestTemplate`；观察者模式：事件监听；策略模式：资源、排序、不同实现选择；适配器模式：MVC 的 `HandlerAdapter`。

另外，Spring 的 `Resource` 包装、请求/响应包装等场景也能看到装饰器模式。面试不必把每个类强行归类，重点是说明模式如何让框架做到可扩展、可替换。

面试不要只报名称，最好补一句“它解决什么”：例如 `JdbcTemplate` 固化 JDBC 流程，让开发者只写 SQL 和结果映射。

### 设计模式要能说出用途

工厂模式负责创建与获取对象，代理模式让事务/AOP 在不改业务代码的前提下增强方法，模板模式固定资源处理流程，观察者模式让事件发布者和监听者解耦，适配器模式让 MVC 支持不同类型处理器。面试要优先讲“解决的问题”，不要只背类名。
0
## 11. Spring 事件和监听器如何工作？

发布者通过 `ApplicationEventPublisher` 发布事件，监听器用 `@EventListener` 或实现 `ApplicationListener` 接收。它适合把“下单成功后发通知、记日志”等非核心动作解耦。默认同步执行；需要异步要配合 `@Async` 或消息队列，并处理失败重试与事务边界。

### 事件使用边界

事件适合同一应用内的解耦协作。若监听动作必须与主业务一起成功，需关注事务时机，可使用 `@TransactionalEventListener` 指定在提交后执行；跨服务、要求可靠投递的场景则应使用消息队列与幂等消费，而不是只靠本地事件。

## 12. 如何实现事务Spring 的声明式事务原理是什么？

### 先用大白话理解
![[Pasted image 20260712161715.png|504]]
事务就是“要么整组成功，要么整组失败”。

比如转账包含扣减 A 账户余额和增加 B 账户余额：只扣钱不加钱是不允许的。Spring 事务会在这组操作开始前先记账，全部成功就提交；中途出错就回滚到开始前的状态。

`@Transactional` 的价值是：你只需要标记“这个方法是一组操作”，Spring 在背后自动安排开启、提交和回滚，不用每个业务方法手写事务控制代码。

`@Transactional` 本质是 AOP：代理在目标方法前开启或加入事务，正常返回时提交，异常时按规则回滚。真正的连接、提交、回滚由 `PlatformTransactionManager` 协调；JDBC 常用 `DataSourceTransactionManager`，JPA 常用 `JpaTransactionManager`。

除传播行为外，还可配置隔离级别来应对脏读、不可重复读、幻读。默认通常只对运行时异常和 `Error` 回滚；受检异常需要通过 `rollbackFor` 显式指定。隔离级别和回滚规则是事务语义的一部分，不应只背 `@Transactional` 注解。

**面试一句话：**声明式事务是代理拦截方法，再委托事务管理器绑定、提交或回滚当前线程资源。

### 按截图理解五个关键点

1. **声明式事务**：`@Transactional` 或 XML 声明事务边界；最常用，业务代码最干净。
2. **传播行为**：决定事务方法互相调用时，是加入旧事务还是新开事务；详见 [[01-Spring核心与Spring MVC#13. 事务的传播行为有哪些？]]。
3. **事务管理器**：真正管理连接和提交/回滚的组件；JDBC、JPA 等技术对应不同实现。
4. **异常回滚**：默认通常对运行时异常和 `Error` 回滚；受检异常可用 `rollbackFor` 指定。异常若被业务代码吞掉，事务可能无法感知而正常提交。
5. **编程式事务**：使用 `TransactionTemplate` 或 `PlatformTransactionManager` 手动划分边界；需要细粒度动态控制时使用，详见 [[01-Spring核心与Spring MVC#32. Spring 事务有哪些实现方式？各有什么特点？]]。

```java
@Transactional(rollbackFor = Exception.class)
public void transfer(Long from, Long to, BigDecimal amount) {
    accountRepository.decrease(from, amount);
    accountRepository.increase(to, amount);
}
```

上例中第二步抛出匹配回滚规则的异常时，第一步的扣款也会回滚。前提是两个操作使用同一事务管理器和同一事务上下文。

### 事务属性要一起回答

一次事务定义通常包含传播行为、隔离级别、超时、只读标记和回滚规则。`readOnly=true` 是优化提示，不是禁止写入的安全机制；超时能避免连接长期占用；隔离级别最终还受数据库实现影响。

## 13. 事务的传播行为有哪些？
![[Pasted image 20260712150209.png|535]]
最常用的 `REQUIRED`：有事务就加入，没有就新建。`REQUIRES_NEW`：挂起外层事务并新建；`NESTED`：在当前事务内建立保存点；`SUPPORTS`、`NOT_SUPPORTED`、`NEVER`、`MANDATORY` 分别对应支持、非事务、禁止事务、必须已有事务。

⚠️ `REQUIRES_NEW` 的内外事务彼此独立；`NESTED` 依赖保存点，仍受外层最终提交/回滚影响。

实际选择口诀：绝大多数业务用 `REQUIRED`；必须独立提交的审计/日志才考虑 `REQUIRES_NEW`；需要局部回滚且数据源支持保存点时再考虑 `NESTED`。不要为“看起来高级”滥用新事务，它会增加连接占用和一致性复杂度。

### 传播行为对比
![[Pasted image 20260712165450.png|465]]
`REQUIRED` 是默认并加入外层事务；`SUPPORTS` 有就加入、无就非事务；`MANDATORY` 强制要求已有事务；`REQUIRES_NEW` 挂起外层并新开；`NOT_SUPPORTED` 挂起外层后非事务执行；`NEVER` 有事务即报错；`NESTED` 基于保存点实现局部回滚。传播行为解决的是“方法嵌套时事务边界如何组合”，不是数据库隔离级别。

## 14. 哪些情况会导致 @Transactional 失效？

常见原因：同类内部调用绕过代理；方法不是 `public`（默认代理配置下）；异常被吞掉；抛出受检异常但未配置 `rollbackFor`；Bean 不由 Spring 管理；事务方法在初始化阶段调用；使用了不受代理支持的 `final` 限制。排查时先确认“调用是否经过 Spring 代理”。

### 排查事务不生效的顺序

先看调用是否经过代理（自调用最常见）；再看方法可见性、Bean 是否受 Spring 管理、异常是否被吞掉或类型不匹配；最后检查事务管理器、数据源、数据库引擎和传播行为。不要把所有问题都归因于 `@Transactional` 注解“没扫描到”。

## 15. Spring MVC 的请求处理流程是什么？
![[Pasted image 20260712161930.png|500]]
### 先用大白话理解 DispatcherServlet

`DispatcherServlet` 是 Spring MVC 的“总调度台”或“前台总机”。浏览器的请求先到它这里；它不负责下单、查库等业务，而是负责找到应该处理这件事的 Controller，安排调用，然后把结果交给视图或 JSON 转换器返回。

类比医院：DispatcherServlet 是导诊台，`HandlerMapping` 是挂号分诊规则，`HandlerAdapter` 是能调用不同医生的助手，Controller 是真正看病的医生，ViewResolver 是把医生结论转换成具体报告的部门。

请求进入 `DispatcherServlet`，再由 `HandlerMapping` 找到处理器，由 `HandlerAdapter` 调用 Controller；Controller 返回 `ModelAndView`、视图名或响应体；视图场景交给 `ViewResolver` 渲染，`@ResponseBody`/`@RestController` 则由 `HttpMessageConverter` 写回 JSON。`HandlerAdapter` 的意义是让 DispatcherServlet 不必绑定某一种 Controller 实现。

**记忆：**前端控制器找处理器、适配器调用、视图解析或消息转换返回。

常见九大协作组件包括：`HandlerMapping`、`HandlerAdapter`、`HandlerExceptionResolver`、`ViewResolver`、`RequestToViewNameTranslator`、`LocaleResolver`、`ThemeResolver`、`MultipartResolver`、`FlashMapManager`。并非每个请求都会显式用到全部组件；`DispatcherServlet` 的职责是协调它们，而不是自己承担业务逻辑。

### 按截图走一遍完整流程

1. **接收请求**：`DispatcherServlet` 接收 HTTP 请求并读取 URL、方法、参数等。
2. **查找处理器**：`HandlerMapping` 根据 URL 等规则找到对应 Handler，通常就是 Controller 的某个方法。
3. **选择适配器**：`HandlerAdapter` 选择合适的调用方式，完成参数解析、类型转换等准备。
4. **执行业务**：适配器调用 Controller；Controller 再调用 Service 等业务层。
5. **得到结果**：Controller 可返回 `ModelAndView`、视图名、普通对象或 `ResponseEntity`。
6. **解析视图或消息转换**：页面场景由 `ViewResolver` 找到实际 View；前后端分离场景由 `HttpMessageConverter` 转为 JSON。
7. **写回响应**：View 渲染页面或消息转换器写入响应体，最终返回浏览器。

⚠️ `@RestController` 返回 JSON 时，通常不会走 `ViewResolver`；异常发生时会优先交给 `HandlerExceptionResolver` / 全局异常处理器，而不是继续正常视图渲染。

### 请求链路中的异常

Controller 抛出异常后，`HandlerExceptionResolver` 会尝试把异常解析成响应；文件上传由 `MultipartResolver` 在进入 Controller 前处理；国际化解析和 Flash 属性也由专门组件协作。理解这条链路后，拦截器、全局异常和 JSON 返回的位置就不会混淆。
![[Pasted image 20260712165513.png]]

## 16. @Controller、@RestController、@RequestMapping 有什么区别？

`@Controller` 用于 MVC 控制器，返回字符串默认会被当作视图名；`@RestController` 等于 `@Controller + @ResponseBody`，返回值直接写入响应体，常用于 REST API。`@RequestMapping` 可定义路径、HTTP 方法、参数和请求头条件；`@GetMapping` 等是它的语义化快捷写法。

### 映射细节

`@RequestMapping` 可标在类上作为 URI 前缀，也可标在方法上声明路径、方法、请求头、参数等条件；`@GetMapping`、`@PostMapping` 等是指定 HTTP 方法的语义化快捷注解。前后端分离接口通常用 `@RestController`，服务端渲染页面才更多使用 `@Controller` + 视图名。

Spring MVC 也支持实现旧式 `Controller` 或 `HttpRequestHandler` 接口来作为 Handler，但现代项目几乎都优先使用注解式 Controller：方法粒度更细、参数绑定更自然、可读性更好。`@RestController` 的响应内容由内容协商和 `HttpMessageConverter` 决定，常见场景会按客户端 `Accept` 请求头选择 JSON 等格式。

## 17. @RequestParam、@PathVariable、@RequestBody 如何选择？

`@RequestParam` 获取查询参数或表单参数，如 `/users?page=1`；`@PathVariable` 获取资源路径变量，如 `/users/{id}`；`@RequestBody` 把 JSON 请求体反序列化为对象。REST 风格中，资源标识优先放路径，筛选/分页放查询参数，复杂写入数据放请求体。

### 参数绑定易错点

`@RequestParam` 默认来自 query/form 参数，常用于分页、筛选等；`@PathVariable` 代表资源身份；`@RequestBody` 消费整个请求体，通常一个方法只应有一个。路径变量和查询参数都是字符串到目标类型的转换，格式不合法会在参数绑定阶段报错，应进入统一异常处理。

## 18.异常 Spring MVC 如何返回 JSON、做参数校验和统一异常处理？
![[Pasted image 20260712160208.png]]
### 先用大白话理解异常处理

异常处理不是“出了错打印一句话”，而是给所有接口准备统一的故障出口。

例如下单时库存不足、参数格式错误、数据库异常，如果每个 Controller 都自己 `try/catch`，返回格式会乱成一团。全局异常处理像客服中心：不同部门报来的问题，统一翻译成前端能理解的错误码、提示语和 HTTP 状态码；真正的堆栈细节留在服务端日志。

返回 JSON 依赖 `HttpMessageConverter`（常见 Jackson）；参数对象标注 `@Valid`/`@Validated` 并在字段上加校验注解；使用 `@RestControllerAdvice + @ExceptionHandler` 统一转换异常为约定响应体。

⚠️ 校验结果要显式处理 `BindingResult` 或让异常进入全局处理器；不要把数据库异常原样返回给前端。

### 统一响应建议

成功响应、参数错误、业务异常、未认证、无权限、系统异常应有稳定的错误码和结构。全局处理器只负责“翻译异常”，不要在其中吞掉日志或把数据库/堆栈细节返回客户端。JSON 的实际序列化由 `HttpMessageConverter` 完成，Spring MVC 常用 Jackson 的 `MappingJackson2HttpMessageConverter`。

### 截图中的五种方式怎么用

1. **`@RestControllerAdvice + @ExceptionHandler`**：最常用的全局异常处理组合，适合统一返回 JSON。
2. **`ResponseEntityExceptionHandler`**：Spring MVC 常见异常的扩展基类，适合覆盖默认处理逻辑。
3. **自定义业务异常**：例如 `BusinessException`，业务层在库存不足、状态不允许时抛出，让全局处理器转换成业务错误码。
4. **`BindingResult` / 校验异常**：接收参数后处理校验错误；也可统一处理 `MethodArgumentNotValidException` 等异常。
5. **`@ResponseStatus`**：给简单异常绑定状态码；复杂项目通常仍由全局处理器统一响应体。

**推荐顺序：**Controller 只做参数接收 → Service 抛语义明确的业务异常 → Advice 统一记录、转换、返回。不要在每一层都捕获后再包装一次，容易丢失堆栈和错误语义。

## 19. Filter、HandlerInterceptor、AOP 有什么区别？
![[Pasted image 20260712163503.png|508]]
### 先用大白话理解

三者都能“在业务代码执行前后做点事”，但站的位置不同。

- **Filter**：站在 Web 容器的大门口，像小区门卫；请求刚进来时就能处理。
- **Interceptor**：进入 Spring MVC 后的关卡，像前台；它知道这次请求要找哪个 Controller。
- **AOP**：站在业务方法调用旁边，像统一的业务流程助手；不局限于 HTTP 请求。

Filter 属于 Servlet 规范，最靠近 HTTP 容器，适合编码、CORS、通用请求包装；Interceptor 位于 Spring MVC 调用 Controller 前后，适合登录校验、接口审计；AOP 面向方法调用，适合事务、日志和权限等业务横切逻辑。三者不是替代关系，按作用层次选择。

### 截图四点对比

| 维度 | Filter | HandlerInterceptor |
| --- | --- | --- |
| 所在层次 | Servlet 容器层 | Spring MVC 层 |
| 执行位置 | 请求最前端，`DispatcherServlet` 之前 | Controller 调用前后 |
| 覆盖范围 | 所有经过容器的请求 | 仅 Spring MVC 映射到 Handler 的请求 |
| 能否了解 Controller | 不能直接拿到 Controller 方法元数据 | 可以拿到 `Handler`，通常能判断目标 Controller/方法 |
| 常见注册方式 | 容器配置、注解、`FilterRegistrationBean` | `WebMvcConfigurer#addInterceptors` |
| 典型用途 | 编码、CORS、请求包装、通用安全头 | 登录校验、接口权限、审计、重复提交拦截 |

⚠️ 认证机制若使用 Spring Security，优先放入它的 Security Filter Chain，而不是自己再写一个普通 Filter 造成顺序混乱。

Interceptor 可通过实现 `HandlerInterceptor` 的 `preHandle`、`postHandle`、`afterCompletion` 并在 `WebMvcConfigurer#addInterceptors` 注册；它能拿到 `Handler`，因而比 Filter 更了解将要执行的 Controller。Filter 可通过容器或 `FilterRegistrationBean` 注册，通常覆盖所有经过 Servlet 容器的请求。

### HandlerInterceptor 的三个方法

| 方法 | 什么时候执行 | 适合做什么 |
| --- | --- | --- |
| `preHandle` | Controller 方法执行前 | 登录校验、权限校验、接口限流、记录开始时间；返回 `false` 可直接中断请求。 |
| `postHandle` | Controller 正常执行后、视图渲染前 | 给 `ModelAndView` 补公共数据、调整视图信息；前后端 JSON 接口中通常使用较少。 |
| `afterCompletion` | 请求完成后（视图渲染结束后） | 清理 ThreadLocal、记录最终耗时、释放请求级资源、统一审计收尾。 |

它实现的是 **Spring MVC 请求级** 的横切处理：适合围绕 HTTP 请求做事。事务、Service 方法日志等不依赖 HTTP 的问题，仍更适合 AOP；编码、CORS、请求包装等最外层问题更适合 Filter。

⚠️ `preHandle` 中如果拒绝请求，要自己保证响应格式一致；`afterCompletion` 即使出现异常也可能执行，因此清理 ThreadLocal 等操作应放这里或 `finally` 中，避免线程池复用时数据串到下一个请求。

### 如何选择

编码、请求包装、CORS 等 Servlet 层问题用 Filter；只针对 MVC Handler 的鉴权、审计、限速预处理用 Interceptor；方法级横切规则如事务、耗时统计用 AOP。Interceptor 的 `preHandle`、`postHandle`、`afterCompletion` 分别对应处理前、视图渲染前、请求完成后。

## 20. Spring Security 的认证与授权流程是什么？

请求经过安全过滤器链，认证阶段确认“你是谁”（账号密码、Token、OAuth2 等）并生成 `Authentication`；授权阶段根据角色、权限或表达式判断“你能做什么”。认证结果通常存入 `SecurityContext`。账号密码场景中，`UserDetailsService` 负责按用户名加载用户信息，`PasswordEncoder` 负责密码哈希与匹配；密码绝不能明文存储。前后端分离项目常采用无状态 Token，并配置 CORS、CSRF 策略和异常响应。

### 安全链路追问

认证成功不代表一定有权限：认证建立 `Authentication`，授权再根据 URL、角色、权限或方法注解决定放行。401 通常表示未认证或凭证无效，403 通常表示已认证但无权限。使用 JWT 也仍需要处理签名校验、过期、撤销策略、权限变更和密钥轮换。

## 21. OAuth2 是什么？

### 先用大白话理解

OAuth2 解决的不是“用户怎么登录”，而是“用户怎样授权某个应用在有限范围内替自己访问资源”。

例如你用微信/Google 登录第三方网站：第三方网站不应该拿到你的密码；你在授权页面确认后，授权服务器发给它一个有范围、有效期的 Token。第三方拿 Token 调用资源服务器，只能做被授权的事情。

OAuth2 是授权框架，不等于登录协议本身。它让用户授权第三方在限定范围内访问资源，而不必交出密码。常见角色：资源所有者、客户端、授权服务器、资源服务器；常见授权方式包含授权码模式。实际项目中常与 OpenID Connect 组合完成身份认证。

当前 Web 与移动端通常优先采用“授权码 + PKCE”；不要把已经不推荐的密码模式用于新系统。OAuth2 解决授权委托，OIDC 在其上补充身份层；二者经常一起出现，但含义不同。

### 截图知识点的当前版本整理

| 概念 | 含义与当前建议 |
| --- | --- |
| 授权码模式 | 用户跳转授权服务器确认授权，客户端再用授权码换 Token；新 Web/移动端首选，移动端加 PKCE。 |
| 简化模式 | 早期浏览器模式，Token 暴露风险更高；新项目不推荐。 |
| 密码模式 | 用户把账号密码直接交给客户端；新项目不推荐，应改用授权码 + PKCE。 |
| 客户端凭证模式 | 没有最终用户，服务调用服务时用客户端身份换 Token；仍有实际用途。 |
| Access Token | 调用资源 API 的短期凭证，应校验签名、过期时间、受众和权限范围。 |
| Refresh Token | 用于换取新的 Access Token，生命周期更长，必须更严格保护和支持撤销。 |

Spring Security 可以分别作为 OAuth2 Client、Resource Server 或 Authorization Server 集成。资源服务器负责校验 Token；授权服务器负责认证用户/客户端并签发 Token。不要把“Resource Server”和“Authorization Server”混成同一个角色。

### OAuth2 角色与流程

资源所有者授权客户端，授权服务器签发访问令牌，客户端携带令牌访问资源服务器。令牌应限制 scope、有效期与受众；资源服务器只接受可信签发方的令牌。前后端分离场景不能把 client secret 放进浏览器或移动端。

## 22. 如何理解 REST API，以及接口如何保证安全？

### 先用大白话理解 REST

REST 可以理解成“**用统一的 HTTP 规则操作资源**”。

把后端想成图书馆：用户、订单、图书都是资源；URL 像资源地址，HTTP 方法像你要对它做的动作。看到接口时，不用先背文档，也能猜出大意：

| 目的 | REST 风格示例 | 含义 |
| --- | --- | --- |
| 查一本书 | `GET /books/100` | 获取 id 为 100 的书 |
| 查书列表 | `GET /books?page=1` | 获取图书列表与分页 |
| 新增一本书 | `POST /books` | 创建资源 |
| 修改一本书 | `PUT /books/100` 或 `PATCH /books/100` | 更新资源 |
| 删除一本书 | `DELETE /books/100` | 删除资源 |

REST 不是“返回 JSON 就叫 REST”，也不是“URL 不写动词就一定 RESTful”；它更强调资源地址、统一 HTTP 方法、无状态交互和清晰状态码这些约定。

REST 以资源为中心，用 URI 标识资源、HTTP 方法表达操作（GET 查、POST 建、PUT/PATCH 改、DELETE 删），保持无状态并使用合适状态码。接口安全需使用 HTTPS、认证授权、输入校验、限流、防重放与日志审计；写接口要考虑幂等性，敏感信息不能放 URL 或日志。

**安全（safe）和幂等不要混淆：**GET、HEAD 约定上不改变服务端资源，因此是安全方法；PUT、DELETE 通常是幂等的，但会改变资源，不能称为安全；POST 通常既不安全也不幂等。REST 的无状态是指服务器不依赖某个请求之外的会话上下文来理解当前请求，不是说服务端不能有数据库或缓存。

### 截图四个追问一次讲清

1. **REST 是什么**：一种 API 设计风格，核心是围绕资源与 HTTP 语义协作；JSON/XML 只是数据表现形式。
2. **什么是安全操作**：安全（safe）指不修改服务端资源，通常是 GET、HEAD；它不是指“不会被攻击”。
3. **什么是无状态**：每个请求都要带齐处理所需的认证和参数，服务端不依赖上一个请求的临时会话状态来理解它。
4. **REST 安全吗**：REST 本身不自动安全。需要 HTTPS、认证（Token/Session 等）、授权、输入校验、限流、审计、敏感数据保护等共同实现。

### REST 接口检查清单

URI 用名词表示资源，HTTP 状态码表达结果；分页、排序、过滤尽量通过查询参数表达；写操作要有鉴权、校验、幂等与审计；全站使用 HTTPS。REST 是一种架构风格，不是“URL 里没有动词”就自动 RESTful。

## 23. Spring Data JPA 的工作原理是什么？

### 先用大白话理解

以前操作数据库，你要自己写 SQL、拿连接、执行 SQL、把查询结果一列列装进对象。Spring Data JPA 的思路是：**常规增删改查你只要写“我要操作哪种数据、按什么条件找”，它帮你把大部分重复工作补齐。**

例如用户表对应 `User` 实体。你定义一个 `UserRepository` 接口，并写 `findByName(String name)`；Spring Data JPA 会在运行时为这个接口生成实现，理解成“按 name 查询 User”，再交给 JPA/Hibernate 生成并执行 SQL。你写的是“查询意图”，框架负责常规实现。

Spring Data JPA 在 JPA 规范之上提供 Repository 抽象。应用定义实体和 Repository 接口后，Spring 在启动时为接口创建代理；代理根据方法名推导查询，或执行 `@Query` 指定的 JPQL/SQL，再交给 JPA 实现（如 Hibernate）完成 EntityManager、SQL 与结果映射工作。

### 截图中的五点对应什么

1. **Repository 接口**：开发者定义接口，Spring Data JPA 生成代理实现；常见基类是 `JpaRepository`。
2. **查询方法命名解析**：`findByNameAndStatus` 这类方法名会被解析为查询条件；复杂条件不宜无限堆在方法名里。
3. **实体管理**：`@Entity` 把 Java 类与表建立映射，`@Id` 标识主键；JPA 通过持久化上下文跟踪实体状态变化。
4. **事务管理**：读写操作通常在 Spring 事务中完成，事务提交时 JPA 可能把受管实体的变化自动同步到数据库（脏检查）。
5. **Hibernate 的位置**：JPA 是规范，Hibernate 是常见实现；Spring Data JPA 不是替代 Hibernate，而是再向上提供 Repository 这一层便利抽象。

⚠️ Repository 很省代码，但复杂查询仍要关注 SQL、索引和 N+1 问题，不能把性能问题藏在方法名后面。

### Repository 的边界

派生查询适合简单条件；复杂多表、性能敏感查询应明确写 `@Query`、Specification、原生 SQL 或转用 MyBatis。实体状态、一级缓存、延迟加载、事务边界共同决定最终 SQL，排查性能一定要看真实 SQL 与执行计划。

## 24. Spring 如何实现国际化（i18n）？

### 先用大白话理解

国际化（i18n）就是：**程序逻辑不变，只根据用户所在语言环境换显示的文字。**

例如登录接口无论中文、英文用户访问，校验账号密码的代码完全一样；区别只在返回文案。我们不把“用户名或密码错误”写死在 Java 代码里，而是给它取一个 key，例如 `login.failed`，再准备不同语言的翻译文件。

```properties
# messages_zh_CN.properties
login.failed=用户名或密码错误

# messages_en_US.properties
login.failed=Invalid username or password
```

![[Pasted image 20260712150358.png]]
配置 `MessageSource` 和不同语言的资源文件（如 `messages_zh_CN.properties`、`messages_en_US.properties`），再依据请求 Locale 解析并获取对应文案。Web 场景可结合 `LocaleResolver`、请求头或参数切换语言。国际化的对象是文案，不应把业务分支复制成多套语言代码。

### 截图中的组件分别做什么

1. **消息资源文件**：像一本多语言词典；同一个 key 在不同文件里有不同翻译。
2. **`MessageSource`**：查词典的工具。你传入 key、参数和语言，它返回对应文案。
3. **`ApplicationContext`**：Spring 的应用上下文本身通常就能作为 `MessageSource` 使用，因此可以从容器中获取国际化消息。
4. **`Locale`**：用户当前的语言/地区，例如 `zh_CN`、`en_US`。
5. **`LocaleResolver`**：判断该用哪个 Locale 的规则，可从 `Accept-Language` 请求头、Cookie、Session 或请求参数中解析。

⚠️ 国际化只解决“显示什么语言”，不等于按国家复制业务规则、金额格式或时区处理；后几项还需要分别使用 `NumberFormat`、时区和日期时间 API 处理。

### 国际化实践

把面向用户的文本抽到资源文件，通过 message key 获取，避免在代码中散落中文/英文常量。Locale 可来自 `Accept-Language`、Cookie、Session 或参数解析器；接口错误消息也应使用同一套 message key，便于多语言维护。

## 25. JdbcTemplate 是什么，解决了什么问题？

### 先用大白话理解

不用 JdbcTemplate 时，写一次查询通常要重复做很多事：拿数据库连接、创建 SQL 语句、设置参数、执行、循环读取结果、关闭连接和处理异常。真正和业务有关的往往只有两件：**写什么 SQL、把结果变成什么对象。**

JdbcTemplate 就像一个数据库操作助手：你把 SQL 和参数交给它，它负责固定的流程和善后工作。它没有替你“自动猜业务 SQL”，而是把 JDBC 最容易重复、最容易忘关资源的部分统一封装。

![[Pasted image 20260712150612.png]]
`JdbcTemplate` 通常由 `DataSource` 创建/注入，它把 JDBC 中获取连接、创建语句、执行、关闭资源、转换异常这些固定流程封装起来。常用 `query`、`update` 执行查询和更新；查询结果可交给 `RowMapper` 逐行映射，或由 `ResultSetExtractor` 整体处理。它仍是直接操作 JDBC，适合简单、可控的 SQL 场景；复杂动态 SQL 可考虑 [[01-MyBatis]]。

### 截图中的四步分别做什么

1. **`DataSource`**：数据库连接的来源，通常背后接的是连接池；它保存 URL、用户名、密码、连接池配置等。
2. **`JdbcTemplate`**：拿到 `DataSource` 后的执行工具，负责借连接、执行、归还连接和异常转换。
3. **`query` / `update`**：`query` 用于查数据，`update` 用于新增、修改、删除；参数用 `?` 占位符绑定，避免字符串拼接 SQL。
4. **结果映射**：`RowMapper` 是“每一行转一个对象”，适合普通列表查询；`ResultSetExtractor` 是“拿到整个结果集自己处理”，适合需要组合、分组的复杂结果。

```java
List<User> users = jdbcTemplate.query(
    "select id, name from user where status = ?",
    (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name")),
    1
);
```

上面你只关心 SQL、参数和如何把一行记录变成 `User`；连接和资源释放由 JdbcTemplate 处理。

### JdbcTemplate 使用边界

它负责连接获取、资源关闭与异常翻译，但不会替你设计 SQL 或索引。参数必须使用占位符绑定，不能字符串拼接；批量写入可使用 `batchUpdate`；事务边界仍由 Spring 事务管理器控制。需要动态 SQL 或复杂映射时，使用 [[01-MyBatis]] 往往更直观。

## 26. Spring 的数据访问异常层次结构有什么价值？

### 先用大白话理解

不同数据库技术报错时“说的话”不一样：JDBC 驱动可能抛一种异常，Hibernate 又抛另一种。若业务代码直接处理它们，就会被具体技术绑死，换 ORM 或数据库时要改很多判断。

Spring 相当于一个翻译员：把底层的各种数据库异常翻译成统一的 `DataAccessException` 家族。业务层不必关心异常来自 MySQL、JDBC 还是 Hibernate，而是关注业务含义：是“数据取不到”、还是“SQL 用错了”、还是“数据冲突了”。

Spring 把不同数据库和 ORM 的底层异常翻译为统一、非受检的 `DataAccessException` 层次结构。`DataRetrievalFailureException` 常表示数据获取失败，`InvalidDataAccessResourceUsageException` 常表示 SQL/资源使用不当，此外还有数据完整性冲突、乐观锁失败、查询结果不正确等类别。业务代码因此不必依赖某个 JDBC 驱动的异常类型，也更容易判断异常是否可重试。

### 截图中的层次怎么记

1. **`DataAccessException`**：所有 Spring 数据访问异常的总父类。
2. **`DataRetrievalFailureException`**：想取数据却取失败，例如关联对象不存在、读取资源失败。
3. **`InvalidDataAccessResourceUsageException`**：数据访问方式不对，例如 SQL 语法、表名列名、资源使用不正确。
4. **其他常见语义**：唯一键/外键冲突通常属于数据完整性异常；乐观锁冲突表示数据已被别人修改；查询本应一条却返回多条也有对应异常。
5. **异常转换**：JdbcTemplate、ORM 集成层等会在底层异常抛出后自动做翻译，业务代码通常不需要手动转换。

⚠️ 统一异常不代表所有异常都该吞掉。是否重试要看异常类型和业务幂等性；唯一键冲突通常需要返回业务提示，连接超时才可能适合有限重试。

### 为什么异常翻译重要

底层 JDBC、Hibernate、JPA 的异常类型各不相同，业务层若直接依赖它们就会被实现绑定。统一的 `DataAccessException` 是非受检异常，既便于事务默认回滚，也能按“可重试、完整性冲突、资源错误”等语义分类处理。

## 27. Spring MVC 与 Spring WebFlux 有什么区别？

### 先用大白话理解

两者都是 Spring 写 Web 接口的方式，区别在于“等慢操作时，线程怎么办”。

- **Spring MVC**：像一个服务员专门服务一桌客人。查数据库、调远程服务时，这个服务员会等结果回来，期间不能接别桌。
- **WebFlux**：像服务员把“等结果”这件事登记后先去服务别桌，结果回来再继续处理。因此同样数量的线程能应对更多“等待网络/IO”的连接。

这不代表 WebFlux 天生更快。普通后台 CRUD 使用 MVC 已经足够成熟；只有请求很多、连接保持很久、且调用链大部分都是非阻塞 I/O 时，WebFlux 才更能发挥优势。

Spring MVC 基于 Servlet API，通常采用“一请求一线程”的阻塞式编程模型，生态成熟且适合大多数 CRUD 应用；WebFlux 基于 Reactor，支持非阻塞 I/O 和背压，适合大量慢 I/O、长连接、高并发场景。

⚠️ WebFlux 不是性能开关：内部仍调用阻塞 JDBC、文件 I/O 或同步远程调用时，会堵塞事件循环，反而得不偿失。

### 对比速记

| 维度 | Spring MVC | Spring WebFlux |
| --- | --- | --- |
| 编程模型 | Servlet、同步阻塞 | Reactor、响应式、非阻塞 |
| 常见返回值 | `String`、对象、`ModelAndView` | `Mono<T>`、`Flux<T>` |
| I/O | 传统阻塞 I/O | 非阻塞 I/O、支持背压 |
| 适合场景 | 常规管理后台、传统 CRUD | 网关、流式数据、长连接、大量慢 I/O |
| 最大误区 | 线程多不等于无限并发 | 使用阻塞 JDBC 会破坏非阻塞链路 |

**面试回答：**MVC 和 WebFlux 不是替代关系。选 MVC 还是 WebFlux，要看整个调用链是否非阻塞、团队熟悉度和业务连接模型，而不是只看“并发高不高”。

### 选型原则

MVC 对阻塞式 Servlet、JDBC、传统 CRUD 非常合适；WebFlux 的价值在于端到端非阻塞链路和高连接数，而不是把同步项目改成 `Mono`/`Flux` 就更快。一个应用可以同时理解两者，但不要在同一请求链里随意混用阻塞与事件循环线程。

## 28. Spring 中的 Template 模式体现在哪里？

### 先用大白话理解

Template 模式就是：**把每次都一样、又容易出错的步骤交给模板；把真正不同的业务步骤留给你填写。**

例如 JDBC 查询时，“拿连接 → 执行 → 关闭连接 → 翻译异常”几乎每次都一样；真正不同的是 SQL 和结果怎么转对象。`JdbcTemplate` 把前一部分固定好，你只提供 SQL 和行映射逻辑。

模板方法模式把稳定流程固定在父类或模板中，把变化点交给回调。`JdbcTemplate` 固化 JDBC 流程，`RestTemplate`/WebClient 封装请求流程，事务模板 `TransactionTemplate` 封装事务边界。模板负责资源的打开和关闭、统一异常转换与常规流程，调用者只实现变化部分；因此既减少样板代码，也避免遗漏资源释放。

### 它解决的真实问题

1. **资源管理**：连接、流、会话等不容易忘记关闭。
2. **异常统一**：不同底层异常能翻译成统一异常体系。
3. **代码复用**：固定流程写一次，避免每个业务方法复制 `try/catch/finally`。
4. **API 更短**：开发者只提交变化部分，代码更聚焦业务。

⚠️ 模板不是万能 ORM。它减少流程样板代码，不会替你自动设计 SQL、索引和事务边界。

### 模板方法的本质

模板类把“变化少且容易出错”的资源处理流程封装起来，再通过回调暴露“变化多”的 SQL、映射或业务步骤。这样既避免重复 `try/finally`，又统一异常转换；调用者仍需要为回调逻辑负责。

## 29. Spring Security 过滤器链是什么？

### 先用大白话理解

请求还没到 Controller 前，Spring Security 会先让它经过一串“安检门”。每道门只负责一件事：有没有登录凭证、Token 对不对、有没有访问这个接口的权限、CSRF 是否通过等。任意一道门不通过，请求就会被拦下，不会进入业务代码。

所以过滤器链的价值是：**安全流程统一放在入口，不让每个 Controller 手写一遍登录和权限判断。**

Spring Security 把认证、鉴权、异常转换、CSRF 等安全工作拆成多个 Filter，按顺序组成过滤器链。请求先经过链，再进入 `DispatcherServlet`；其中某个过滤器可以直接拒绝请求或建立认证上下文。排查 401/403 时，应先看请求命中了哪条 `SecurityFilterChain`、认证信息是否建立、授权规则是否匹配。

自定义认证 Filter 应插入在合适的内置 Filter 前后，而不是随意注册到 Servlet Filter 链；否则可能绕过异常处理或认证上下文。配置时用 `HttpSecurity` 建立 `SecurityFilterChain`，再通过 `addFilterBefore`/`addFilterAfter` 明确顺序。

### 流程与易错点

请求 → 选择匹配的 `SecurityFilterChain` → 认证 Filter 解析账号密码/Token → 成功后把 `Authentication` 放入 `SecurityContext` → 授权 Filter 判断当前请求所需角色或权限 → 放行给 `DispatcherServlet` 和 Controller。

- **401**：通常是没有认证或凭证无效。
- **403**：通常是已经认证，但权限不足。
- Security Filter 在 MVC Controller 之前执行，因此普通 `@ControllerAdvice` 未必能处理安全链路异常；认证失败和授权失败通常要分别配置处理器。
- 多个 `SecurityFilterChain` 时，请求会按匹配规则选择其中一条；顺序配置错误会导致“明明配置了却不生效”。

### Filter 链调试

出现登录接口也被拦截、Token 已带却仍 401、异常响应格式不统一等问题时，优先确认请求匹配到的 `SecurityFilterChain` 及 Filter 顺序。安全 Filter 在 `DispatcherServlet` 前执行，因此 Controller 的 `@ExceptionHandler` 不一定能处理安全链路中的异常，通常还需要配置认证/授权失败处理器。

## 30. 如何使用 AOP 实现日志记录？
![[Pasted image 20260712154614.png]]
用切点匹配 Controller 或 Service 方法，在 `@Around` 通知中记录方法名、关键参数、耗时、结果或异常，并调用 `proceed()` 执行目标方法。日志要注意脱敏、控制体积和避免重复记录；不要在切面里序列化巨大对象或读取一次性请求流。

使用注解方式时，切面类标记 `@Aspect + @Component`，配置类启用 `@EnableAspectJAutoProxy`（Spring Boot 的 AOP Starter 通常会完成相关自动配置），再用 `@Pointcut` 复用匹配表达式。`@Around` 中必须调用 `proceed()` 才会执行目标方法；漏调会导致业务方法根本不执行。

### 日志切面实践

建议记录 traceId、方法、关键业务标识、耗时和异常类型；密码、身份证、Token、完整请求体必须脱敏或禁止记录。日志切点应尽量限定在业务包，避免把框架内部方法全部打出；性能统计可配合 Micrometer，而不是只依赖手写日志。

## 31. Spring 有哪些配置和装配方式？如何声明 Bean？

### 先用大白话理解

Spring 配置就是告诉容器三件事：**哪些对象要交给你管理、这些对象依赖谁、遇到多个候选对象该选谁。**

写法可以不同，但目的相同：XML 像一份单独的对象清单；注解像直接贴在类上的“请 Spring 管我”标签；Java 配置则像用 Java 代码写对象装配说明书。

配置方式包括 XML、Java 配置（`@Configuration + @Bean`）和注解扫描。常用组件注解有 `@Component`、`@Service`、`@Repository`、`@Controller`；装配可通过构造器、`@Autowired`、`@Qualifier` 和 `@Primary` 解决候选 Bean 选择问题。现代项目优先 Java 配置和构造器注入，XML 主要用于维护旧项目。

### 三种配置方式对比

| 方式 | 怎么写 | 适合什么情况 |
| --- | --- | --- |
| XML 配置 | `<bean id="userService" class="..."/>` | 旧项目、第三方框架遗留配置 |
| 注解配置 | `@Component`、`@Service` 等 + 组件扫描 | 日常业务类，最常见 |
| Java 配置 | `@Configuration` 类中的 `@Bean` 方法 | 需要显式构造第三方对象、复杂装配、条件化配置 |

```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService(UserRepository repository) {
        return new UserService(repository);
    }
}
```

这里 `@Bean` 相当于 XML 的 `<bean>`：返回对象会被放入 Spring 容器；方法参数 `UserRepository` 会由容器自动注入。

### 什么是 Spring 装配

Bean 装配就是：**Spring 按配置和依赖关系，把多个 Bean 连接成可以工作的对象网络。**

例如 `OrderService` 依赖 `OrderRepository`：容器先创建 Repository，再创建 Service，并把 Repository 注入进去。Service 不需要知道 Repository 是怎么创建出来的，只管使用接口即可。

### 截图中的 XML 自动装配模式

这些主要是旧 XML 项目中的 `autowire` 模式，了解即可：

| 模式 | 含义 | 现在是否推荐 |
| --- | --- | --- |
| `no` | 默认，不自动装配，显式写 `<property>` / `<constructor-arg>` | 旧 XML 中可用 |
| `byName` | 按属性名找同名 Bean | 不推荐，依赖名称，重构容易出问题 |
| `byType` | 按类型找 Bean | 多个同类型 Bean 时会歧义 |
| `constructor` | 按构造器参数类型匹配 | 概念上接近现代构造器注入 |
| `autodetect` | 自动猜测构造器或 byType | 已过时，不应在新项目使用 |

现代 Spring 项目更推荐：**构造器注入 + `@Qualifier` / `@Primary` 明确选择**。这样依赖最清楚，不依赖 Bean 名字猜测。

### 装配限制与同名 Bean

- 基本类型、`String` 等简单值通常通过配置属性、`@Value` 或 `@ConfigurationProperties` 提供，不适合靠自动装配猜测。
- 显式声明的依赖会覆盖自动装配结果，这是旧 XML 中需要注意的规则。
- 同类型 Bean 有多个时，先用 `@Qualifier` 指定；若有默认实现，用 `@Primary`。
- 同名 Bean 可能因为配置重复、组件扫描与 `@Bean` 重名而发生覆盖或启动失败，具体行为受 Spring Boot 版本和 `allow-bean-definition-overriding` 配置影响。新项目应避免同名，靠清晰命名和显式限定消除歧义。

### 关于截图里的 XML 注解开关

`<context:annotation-config/>` 是传统 XML 项目中启用一部分注解处理器的方式。现代 Spring Boot 项目通常通过 `@SpringBootApplication` 的组件扫描和自动配置完成，不需要手写这段 XML；不要把旧项目配置照搬到新项目。

内部 Bean 是仅作为另一个 Bean 属性或构造器参数存在的匿名 Bean，常见于旧 XML 中嵌套的 `<bean>`；它不能像顶级 Bean 那样按名字复用，通常可视为随所属 Bean 创建的内部对象。新项目更常用 Java 对象组合或独立 Bean 取代它。

## 32. Spring 事务有哪些实现方式？各有什么特点？
![[Pasted image 20260712162131.png|481]]
### 先用大白话理解 TransactionManager

把一次转账想成“扣钱 + 加钱”两个动作：它们不能只完成一半。业务代码负责说“这两个动作是一组”；`TransactionManager` 负责真正下达命令——开始事务、把同一个数据库连接交给这一组操作、最后决定提交还是回滚。

所以 `@Transactional` 不是自己直接操作数据库事务。它通过 AOP 拦截方法，再委托 `TransactionManager` 完成事务控制。

声明式事务用 `@Transactional`，侵入业务代码少，是默认首选；编程式事务使用 `TransactionTemplate` 或直接调用 `PlatformTransactionManager`，能更精细控制局部边界，但代码更繁琐。事务管理的收益是统一边界、传播、隔离和回滚规则；前提是数据库、连接池和调用路径确实处在同一事务上下文中。

`PlatformTransactionManager` 是事务管理抽象：负责取得事务状态、提交和回滚，并协调连接等资源。JDBC 常用 `DataSourceTransactionManager`；JPA 常用 `JpaTransactionManager`。选择实现必须与实际持久化技术匹配，否则注解看似生效却无法正确管理资源。

旧项目若直接使用 Hibernate，可能看到 `HibernateTransactionManager`；它们都是 TransactionManager 的不同实现，区别在于管理哪种持久化资源，不是事务语义不同。

### 截图中的五项职责

1. **事务控制**：开始、提交、回滚事务。
2. **事务状态管理**：维护当前事务是否新建、是否只读、是否已完成、是否需要回滚等状态。
3. **资源管理**：把数据库连接、JPA `EntityManager` 等资源绑定到当前线程，保证同一事务内的操作使用同一份资源。
4. **不同实现**：JDBC 用 `DataSourceTransactionManager`，JPA 用 `JpaTransactionManager`，旧 Hibernate 项目可能用 `HibernateTransactionManager`；分布式/JTA 场景还有对应实现。
5. **与 Spring 集成**：声明式事务由 `@Transactional` 驱动，编程式事务由 `TransactionTemplate` 或直接 API 驱动，底层最终都依赖事务管理器。

### 声明式和编程式怎么选

| 方式 | 怎么写 | 适合什么情况 |
| --- | --- | --- |
| 声明式事务 | 方法/类上加 `@Transactional` | 大多数业务方法，代码干净、边界清晰 |
| 编程式事务 | `TransactionTemplate` 或手动调用管理器 | 事务边界需要在一个方法内动态拆分、条件非常复杂 |

⚠️ 同一个业务事务不能随意混用多个数据源或多个事务管理器；跨多个资源时要明确采用哪种一致性方案，不能以为一个 `@Transactional` 自动解决分布式事务。

## 33. Spring MVC 与 Struts2 有什么异同？

两者都是 Web MVC 框架，都能做请求分发、参数绑定和视图渲染。Spring MVC 以 `DispatcherServlet` 为前端控制器，方法级 Controller 更贴近 Spring 生态；Struts2 以 Filter 为核心，Action 对象模型不同。新项目通常优先 Spring MVC，因为生态、整合与维护成本更有优势。

| 对比点 | Spring MVC | Struts2 |
| --- | --- | --- |
| 入口 | `DispatcherServlet`（Servlet） | 核心 Filter |
| 开发粒度 | 方法级 Controller | 类级 Action |
| 参数传递 | 通过方法参数、参数解析器绑定 | 常通过 Action 类属性接收 |
| 对象使用 | Controller 通常可作为单例使用 | Action 为避免并发状态问题通常按请求创建 |
| 视图/数据 | `ModelAndView`、Model、Request 等 | 值栈（Value Stack）和 OGNL |
| 当前项目选择 | Spring 生态主流 | 多见于维护旧项目 |

理解重点不是死记 JSP/JSTL 或 OGNL 细节，而是知道两者的请求入口、对象模型和数据传递方式不同。新项目一般不再选择 Struts2。
