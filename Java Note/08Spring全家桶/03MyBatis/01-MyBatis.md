# MyBatis

> 覆盖 MyBatis 的核心执行机制、映射、动态 SQL 与缓存；后续独立题目从第 1 题开始连续编号。

## MyBatis 是什么？

MyBatis 是 Java 后端常用的**持久层框架**：它负责把 Java 方法和数据库 SQL 连接起来。你在代码中调用 `userMapper.findById(1)`，MyBatis 会找到对应 SQL、把参数传入、执行查询，再把结果封装成 `User` 等 Java 对象。

### 先用大白话理解

它像 Java 与数据库之间的“翻译员”。Java 代码不会直接听懂 SQL，数据库也不会直接认识 `User` 对象；MyBatis 就负责双向翻译：

`Mapper 方法 → SQL → 数据库结果 → Java 对象`

### 实际开发中能做什么

常见调用链是：`Controller → Service → Mapper → MyBatis → MySQL`。例如订单列表的多条件筛选、多表关联查询、统计报表等场景，开发者把 SQL 明确写出来，MyBatis 负责执行和映射结果。

### 为什么使用它

- **SQL 可控**：复杂联表、分页、统计 SQL 可以自己精确编写和优化。
- **参数安全**：使用 `#{}` 绑定参数，通常能避免 SQL 注入。
- **对象映射**：把表的一行数据自动映射为 Java 对象。
- **开发方便**：Mapper 接口配合 XML 或注解即可操作数据库，不必手动写 JDBC 的连接、预编译、结果集关闭等模板代码。

⚠️ MyBatis 不是数据库，也不是 ORM 的“完全自动生成器”。它的核心优势是**你写 SQL、它帮你规范执行和映射**；SQL 的正确性、索引设计和慢查询优化仍然需要开发者负责。

### 和 Spring / Spring Boot 的关系

- **MyBatis**：提供 Mapper、SQL 映射、动态 SQL、缓存等数据库访问能力。
- **Spring**：负责管理 `DataSource`、事务和 Mapper 等对象。
- **Spring Boot**：通过 `mybatis-spring-boot-starter` 自动帮我们准备常见依赖和配置，让 MyBatis 更快运行起来。

后续题目会围绕“SQL 如何找到并执行、参数和结果如何映射、动态 SQL、缓存、分页和常见坑”逐步展开。

## 1. MyBatis 的一级缓存和二级缓存有什么区别？

### 先说结论

一级缓存是 **SqlSession 级别** 的本地缓存；二级缓存是 **Mapper namespace 级别** 的共享缓存。可以把一级缓存理解为“当前办事窗口桌上的便签”，窗口关了便签就没了；二级缓存像“整个部门共用的公告板”，多个窗口都可能读取。

| 对比项 | 一级缓存 | 二级缓存 |
| --- | --- | --- |
| 作用范围 | 一个 `SqlSession` | 同一 Mapper namespace |
| 默认状态 | 默认开启 | 默认关闭，需显式启用 |
| 存储位置 | 当前会话的本地缓存 | Mapper 对应的 `Cache` |
| 共享范围 | 不同 SqlSession 不共享 | 多个 SqlSession 可共享 |
| 失效时机 | `commit`、`rollback`、`close`、`clearCache()`，或执行更新类操作 | 更新/提交等会按配置清空相关 namespace 缓存 |

一级缓存常由 `PerpetualCache`（底层通常是 `HashMap`）承载；二级缓存也可用默认实现或接入第三方缓存。二级缓存若要跨序列化边界存对象，实体通常需要可序列化；具体要求还取决于 Cache 实现。

⚠️ 二级缓存不是“打开后数据库永远不会变”。多个服务实例、直接改库、跨 namespace 更新都会带来一致性风险。高频更新的核心业务数据通常优先使用 Redis 等专用缓存方案，并设计明确的失效策略。

### 面试简答

一级缓存随 SqlSession 生命周期存在，适合一次会话内重复查询；二级缓存按 Mapper 共享，能跨会话复用，但一致性更难保证，生产使用要谨慎。

## 2. MyBatis 一级缓存的原理是什么？

### 大白话

同一个 `SqlSession` 可以理解成“这一次和数据库打交道的工作台”。同一工作台刚查过用户 1，马上又查一次用户 1，MyBatis 会先翻工作台上的结果，不必再跑去数据库问一遍。

同一个 `SqlSession` 内多次执行**完全等价**的查询时，MyBatis 会优先查本地缓存；命中就直接返回，不再访问数据库。缓存键并不只是 SQL 文本，还会综合 `MappedStatement`、分页参数、最终 SQL、参数值和环境等生成 `CacheKey`，因此参数不同不会误命中。

执行链路是：Mapper 方法找到 `MappedStatement` → `Executor` 先查本地缓存 → 未命中则 JDBC 查库 → 将结果放入本地缓存。执行 `insert`、`update`、`delete` 后，为了避免读到旧数据，默认会清理本会话相关缓存；也可以调用 `sqlSession.clearCache()` 手动清理。

⚠️ 不要把一级缓存当成跨请求缓存：Web 项目通常一个请求/事务就是一个 SqlSession，关闭后缓存就没有了。

## 3. MyBatis 二级缓存的原理是什么？

### 大白话

一级缓存只放在一个人的桌上，二级缓存则像 `UserMapper` 部门的公共资料柜：不同 SqlSession 都可能从里面拿到同一份用户查询结果。共享更省查询，但“谁负责及时换掉旧资料”也更麻烦。

二级缓存以 Mapper namespace 为单位。例如 `UserMapper` 开启缓存后，多个 SqlSession 查询 `UserMapper` 的相同语句，有机会共享缓存结果。

查询通常仍先看一级缓存，未命中再看二级缓存，仍未命中才查数据库。查询结果在事务提交/会话关闭等合适时机写入二级缓存；执行增删改时会根据语句的 `flushCache` 等配置清空对应 namespace 的缓存，避免明显脏读。

可以在 Mapper XML 中使用 `<cache/>` 开启，也可替换 `Cache` 实现接入 Ehcache 等。⚠️ 不能把“多个 SqlSession 共享”误解为“全系统强一致”：跨服务、数据库被其他程序修改、多个 Mapper 关联更新时仍需自行保证失效和一致性。

## 4. `resultType` 和 `resultMap` 有什么区别？

### 大白话

查询结果就像一张数据库表格，MyBatis 要把它装进 Java 对象。表头和对象属性能自然对上时，直接告诉它“装成 User”（`resultType`）就行；对不上或里面还套着订单、角色等对象时，需要给它一张更详细的“装箱说明书”（`resultMap`）。

两者都用于把查询结果映射成 Java 对象。

- **`resultType`**：直接指定返回类型。列名与 Java 属性名能够自动对应时最省事，适合单表、字段简单的查询。
- **`resultMap`**：先定义一套映射规则，再在 SQL 中引用。它可处理列名与属性名不同、嵌套对象、一对一、一对多等复杂场景。

```xml
<!-- 简单映射：数据库列 user_name 可通过别名对应 userName -->
<select id="findById" resultType="com.example.User">
  select id, user_name as userName from user where id = #{id}
</select>

<!-- 复杂映射：显式描述列到属性的关系 -->
<resultMap id="userMap" type="com.example.User">
  <id property="id" column="user_id"/>
  <result property="userName" column="user_name"/>
</resultMap>
```

面试口诀：**简单对象用 `resultType`，字段不一致或有关联对象用 `resultMap`。**

## 5. 为什么说 MyBatis 是半自动 ORM？

### 大白话

它会帮你把“表的一行”变成“一个对象”，但不会替你决定业务 SQL 怎么写。像请了一个熟练的录入员：表格录入、格式转换它来做，查询条件、联表方式和性能优化仍要你自己定。

ORM 是“对象”和“关系型表”之间的映射。MyBatis 能把查询行映射为 JavaBean，也能处理关联对象，所以具备 ORM 能力；但 SQL 通常由开发者自己编写、自己优化，不像 JPA/Hibernate 那样主要由框架根据实体自动生成 SQL，因此常称为**半自动 ORM**。

好处是复杂 SQL 可控、调优直观；代价是 SQL 增多，需要维护 Mapper XML/注解。实际选型不是谁绝对更好：复杂报表、多表查询常偏向 MyBatis；简单标准 CRUD 可以考虑 JPA。

## 6. MyBatis 动态 SQL 有什么用？有哪些标签？

### 大白话

搜索页面的用户名、状态、时间可能只填其中几个。动态 SQL 就像点菜时按顾客实际选择组合菜品：传了状态才加状态条件，没传就不加，避免为了每种组合各写一条 SQL。

动态 SQL 用于根据参数拼出不同条件，解决“筛选项有时传、有时不传”的问题。例如订单查询可选状态、用户、时间范围；不用动态 SQL 就容易写出大量重复 SQL。

常用标签：

- `<if>`：条件成立才拼接片段。
- `<where>`：自动加 `WHERE`，并处理开头多余的 `AND` / `OR`。
- `<set>`：更新时自动处理多余逗号。
- `<foreach>`：遍历集合，常用于 `IN (...)`、批量插入。
- `<choose>` / `<when>` / `<otherwise>`：多选一分支。
- `<trim>`：自定义前后缀和要移除的字符。
- `<bind>`：创建绑定变量，例如模糊查询模式。

```xml
<select id="list" resultType="User">
  select id, user_name from user
  <where>
    <if test="name != null and name != ''">and user_name = #{name}</if>
    <if test="ids != null and ids.size() > 0">
      and id in <foreach collection="ids" item="id" open="(" separator="," close=")">#{id}</foreach>
    </if>
  </where>
</select>
```

⚠️ 动态 SQL 是“按规则组装 SQL”，不是把用户输入直接拼进去；值参数仍优先用 `#{}`。

## 7. MyBatis 如何获取数据库自动生成的主键？

### 大白话

新增用户时，`id` 往往是数据库自增生成的。主键回填就是插入成功后，MyBatis 把数据库刚发的“新编号”写回你传入的 Java 对象，后面保存关联订单就能直接使用。

MySQL 等支持 JDBC 自动生成键时，常用 `useGeneratedKeys="true"` 和 `keyProperty`：

```xml
<insert id="insertUser" useGeneratedKeys="true" keyProperty="id">
  insert into user(user_name) values(#{userName})
</insert>
```

插入成功后，生成的主键会回填到传入对象的 `id` 属性。若使用 Oracle 序列等场景，可使用 `<selectKey>` 在插入前后获取主键。

⚠️ `keyProperty` 写的是 **Java 属性名**，不是数据库列名；批量插入、不同驱动对回填支持也要实际验证。

## 8. 传统 JDBC 开发有什么问题？MyBatis 解决了什么？

### 大白话

不用 MyBatis 时，每个查询都像自己从仓库拿连接、打开箱子、一个格子一个格子读 `ResultSet`，最后还要记得关门。MyBatis 把这些重复体力活收起来，让业务代码集中在“查什么、返回什么”。

原生 JDBC 每次都要处理连接、预编译、参数设置、结果集遍历、异常和资源关闭；SQL 混在 Java 字符串中，改一条 SQL 往往要改代码、重新编译发布。结果集到 Java 对象的映射也有大量重复代码。

MyBatis 把 SQL 放在 XML 或注解中，用 Mapper 接口调用；它统一做参数绑定、结果映射、资源管理和异常转换。注意：连接池通常由数据源提供，MyBatis 并不替代数据库连接池；SQL 本身的质量仍由开发者负责。

## 9. `#{}` 和 `${}` 有什么区别？

### 大白话
![[Pasted image 20260712182412.png|645]]
`#{}` 像把参数放进密封的表单格里，数据库只把它当“数据”；`${}` 像把用户说的话直接写进 SQL 句子里，用户若故意夹带 `or 1=1`，SQL 的结构就可能被改掉。

- **`#{}`**：预编译参数占位符，最终交给 `PreparedStatement` 绑定值。普通查询条件、插入更新参数几乎都应使用它。
- **`${}`**：字符串直接拼接到 SQL 文本中，不会作为预编译参数绑定，存在 SQL 注入风险。

```xml
select * from user where id = #{id}       <!-- 安全绑定值 -->
select * from user order by ${sortColumn} <!-- 只能用于受控结构片段 -->
```

⚠️ `${}` 不是“给字符串加引号”的工具。若必须用于表名、字段名、排序方向等 SQL 结构，必须从后端白名单枚举后选择，绝不能直接接收前端输入。模糊查询可用 `concat('%', #{keyword}, '%')` 或 `<bind>`，不要用 `${keyword}` 拼接。

## 10. MyBatis 如何实现一对一、一对多关联查询？

### 大白话

查订单时顺便要客户信息是一对一；查订单时还要许多订单项是一对多。可以一次 join 把资料全拿回来，也可以先拿订单、需要时再去查明细；关键是别在列表里悄悄查出几十上百次额外 SQL。

两种常见方式：

1. **联合查询**：一条 `join` SQL 查出所有数据，在 `resultMap` 中使用 `<association>`（一对一）和 `<collection>`（一对多）组装对象图。优点是请求少；要正确配置 `<id>`，避免一对多结果重复组装。
2. **嵌套查询**：先查主对象，再通过 `<association select="...">` 或 `<collection select="...">` 触发子查询。写法直观，但列表场景容易出现 N+1 查询问题。

实际开发中，列表页优先评估联合查询、批量查后在内存组装，避免“查 100 个订单又查 100 次订单项”。

## 11. MyBatis 可以映射 Enum 枚举吗？

### 大白话

数据库里可能存 `PAID` 或数字 `1`，Java 里希望得到 `OrderStatus.PAID`，而不是到处自己写 `if (status == 1)`。`TypeHandler` 就是这两种表示之间的固定翻译规则。

可以。MyBatis 默认可按枚举名称或序号做基础处理，也可自定义 `TypeHandler`，把 Java 枚举与数据库的字符串/整数编码互转。

例如订单状态建议存稳定业务码 `PAID` 或 `1`，再通过 `TypeHandler` 转成 `OrderStatus.PAID`。⚠️ 不建议依赖 `ordinal()` 的位置编号：枚举顺序一改，历史数据可能被解释成错误状态。

## 12. Mapper 中如何传递多个参数？

### 大白话

一个查询可能同时要“姓名、部门、起止时间”。`@Param` 相当于给每个参数贴名字标签；DTO 则像把一整张搜索表单交给 Mapper，条件越来越多时更好维护。

常用方式如下：

| 方式 | 适用情况 | 建议 |
| --- | --- | --- |
| `@Param` | 少量、语义明确的参数 | **最常用**，SQL 可读性最好 |
| JavaBean / DTO | 条件较多、可扩展 | 查询对象最清晰 |
| `Map<String, Object>` | 条件高度动态 | 灵活但类型安全较弱 |
| `param1`/`param2` 或索引 | 临时写法 | 不推荐，改参数顺序易出错 |

```java
User find(@Param("userName") String userName, @Param("deptId") Long deptId);
```

XML 中使用 `#{userName}`、`#{deptId}`。当查询条件逐渐增多时，应优先建 `UserQuery` 这类 DTO，而不是继续堆很多 `@Param`。

## 13. MyBatis 中如何指定使用哪一种 Executor？

### 大白话

Executor 可以理解为 MyBatis 的“SQL 办事方式”。配置默认类型，就是告诉它：平时每条 SQL 都单独办，还是尽量复用办事材料，还是积攒一批更新后一起办。

Executor 是 MyBatis 实际执行 SQL 的执行器。可在全局配置中设置默认类型：

```xml
<settings>
  <setting name="defaultExecutorType" value="SIMPLE"/>
</settings>
```

这里的 `SIMPLE`、`REUSE`、`BATCH` 分别代表三种执行策略，具体差异见第 16 题。绝大部分普通业务保持默认 `SIMPLE` 即可；大量批量写入才评估 `BATCH`。

## 14. MyBatis 是否支持延迟加载？原理是什么？

### 大白话

你查一个订单时，未必立刻要看买家地址和所有订单项。延迟加载就像先拿订单封面，真正点开“订单项”时才去数据库拿详情；省不省要看你最后会不会真的点开它。

支持，常用于关联对象：先查订单本身，需要访问订单的用户或明细时再查关联数据。可通过 `lazyLoadingEnabled` 等配置开启，并结合 `fetchType="lazy"` 使用；具体可用项受版本和映射方式影响。

MyBatis 会为关联属性创建代理/延迟加载器，在真正访问该属性时触发关联 SQL。它的本质是“晚点查”，不是“不查”。

⚠️ 延迟加载能减少不需要的数据，却可能在遍历列表时触发 N+1 查询。不要机械开启；列表页常更适合 join 或批量预取。

## 15. 为什么需要预编译？

### 大白话

预编译是先把 SQL 的“句式”交给数据库准备好，之后只换具体参数。这样既不会让参数改变 SQL 语法，也避免相同模板反复做无意义解析。

JDBC 的 `PreparedStatement` 会先让数据库解析/编译 SQL 模板，再绑定不同参数执行。它的主要价值：

1. 参数与 SQL 结构分离，能显著降低 SQL 注入风险；
2. 相同 SQL 模板可复用解析/执行计划（实际复用效果取决于数据库、驱动和连接池配置）；
3. 参数类型处理更规范。

MyBatis 中 `#{}` 通常走预编译参数绑定；`${}` 是文本拼接，不具备上述安全性。

## 16. MyBatis 都有哪些 Executor？它们有什么区别？

### 大白话

`SIMPLE` 像每次都新开一张办事单；`REUSE` 像相同格式的单子留着下次继续用；`BATCH` 像把很多条更新单攒成一摞再交。普通 CRUD 用 SIMPLE，批量导入才重点考虑 BATCH。

Executor 的三种基本类型：

- **SIMPLE**：每次执行都创建新的 `Statement`，默认且最常见。
- **REUSE**：复用相同 SQL 的 `Statement`，减少重复创建。
- **BATCH**：把多次更新暂存并批量发送，适合批量插入/更新；查询不能靠它批处理。

`BATCH` 模式要主动在合适时机 `flushStatements()`，并注意异常时的回滚、内存占用和生成主键回填。三者的作用范围都在当前 `SqlSession`，不会跨请求永久复用。

## 17. MyBatis 如何为某一次操作指定 Executor？

### 大白话

全局默认是“公司的日常规定”，单次指定则是“这次导入十万条数据临时走批处理通道”。它不会影响其他正常请求。

除全局 `defaultExecutorType` 外，也可以只为某次会话指定执行器：

```java
try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
    // 批量写入
    session.flushStatements();
    session.commit();
}
```

这适合“平时仍用 SIMPLE，但导入任务需要 BATCH”的场景。⚠️ 不要把 `BATCH` 当成所有 SQL 的性能开关；它主要服务批量写，必须配合事务、分批提交和异常处理。

## 18. MyBatis 的框架架构和执行流程是怎样的？

### 大白话

你调用的 Mapper 接口像前台窗口；MyBatis 找到这次请求对应的 SQL 说明书，再派 Executor 通过 JDBC 去数据库办事，最后把查询到的一行行数据装回 Java 对象。这就是 MyBatis 从“方法调用”到“返回对象”的完整流水线。

启动阶段，MyBatis 读取 `mybatis-config.xml`、Mapper XML 或注解，解析为全局 `Configuration`；每条 SQL 会形成一个 `MappedStatement`，其中包含 SQL、参数映射、结果映射、缓存和刷新策略等信息。`MappedStatement` 就像“一条已经登记好的 SQL 说明书”。

运行阶段可记为：

`Mapper 代理 → MappedStatement → Executor → StatementHandler → JDBC → 数据库 → ResultSetHandler → Java 对象`

Mapper 接口本身通常没有实现类；MyBatis 为它创建动态代理。调用 `userMapper.findById()` 时，代理根据“接口全限定名 + 方法名”定位 SQL，随后执行并映射结果。

## 19. MyBatis 的 `like` 模糊查询应该怎么写？

### 大白话

用户输入“张”，你想查“张三”“小张”等名字，就需要在参数两边加 `%`。重点不是能不能查到，而是 `%` 应由 SQL 函数或绑定变量和 `#{}` 组合，不能把用户输入直接拼到 SQL 中。

推荐使用预编译参数拼接通配符：

```xml
select id, user_name from user
where user_name like concat('%', #{keyword}, '%')
```

也可以：

```xml
<bind name="pattern" value="'%' + keyword + '%'"/>
select id, user_name from user where user_name like #{pattern}
```

⚠️ 不推荐 `like '%${keyword}%'`，因为它会直接拼接用户输入，存在 SQL 注入风险。另一个性能点：`%关键词%` 通常无法很好利用普通 B+Tree 索引；数据量大要评估前缀匹配、全文索引或 Elasticsearch。

## 20. MyBatis 如何分页？分页插件的原理是什么？

### 大白话

分页不是“把所有数据查出来再截一段”，而是让数据库只返回当前页。分页插件像一个在 SQL 发出前工作的改写员，自动给原查询补上 `limit` 和统计总数的 SQL；但页码翻得太深，数据库仍然要跳过大量记录。

基础方案有三类：

1. 在 SQL 中使用数据库分页语法，如 MySQL 的 `limit #{offset}, #{size}`；最直观，通常优先。
2. `RowBounds`：逻辑分页，可能先取出较多数据再在内存截取，不适合大数据量。
3. 分页插件（如 PageHelper）：通过 MyBatis `Interceptor` 拦截执行过程，识别原 SQL 后改写为带 `limit/offset` 的分页 SQL，通常还会额外执行 `count` 查询。

⚠️ 深分页 `limit 1000000, 20` 会越来越慢；大表应使用覆盖索引、延迟关联或基于最后一条记录的**游标/键集分页**。分页插件只是改写 SQL，不能替代索引与 SQL 优化。
