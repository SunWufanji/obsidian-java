# Redis 客户端

## 3.1 Jedis 快速入门

Jedis 是 Redis 官方推荐的 Java 客户端，API 与 Redis 命令同名，上手简单。

官网：https://github.com/redis/jedis

### 引入依赖

```xml
<!--jedis-->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.7.0</version>
</dependency>
<!--单元测试-->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.7.0</version>
    <scope>test</scope>
</dependency>
```

### 建立连接 & 测试

```java
public class JedisTest {

    private Jedis jedis;

    @BeforeEach
    void setUp() {
        // 建立连接
        jedis = new Jedis("127.0.0.1", 6379);
        // 设置密码
        jedis.auth("123321");
        // 选择数据库（默认 0 号库）
        jedis.select(0);
    }

    @Test
    void testString() {
        // 存入数据
        String result = jedis.set("name", "张三");
        System.out.println("result = " + result);  // OK
        // 获取数据
        String name = jedis.get("name");
        System.out.println("name = " + name);       // 张三
    }

    @Test
    void testHash() {
        // 插入 hash 数据
        jedis.hset("user:1", "name", "Jack");
        jedis.hset("user:1", "age", "21");
        // 获取所有字段
        Map<String, String> map = jedis.hgetAll("user:1");
        System.out.println(map);  // {name=Jack, age=21}
    }

    @AfterEach
    void tearDown() {
        if (jedis != null) {
            jedis.close();
        }
    }
}
```

---

## 3.2 Jedis 连接池

Jedis 本身**线程不安全**，频繁创建和销毁连接也有性能损耗，推荐使用**连接池**。

```java
public class JedisConnectionFactory {

    private static final JedisPool jedisPool;

    static {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(8);       // 最大连接数
        poolConfig.setMaxIdle(8);        // 最大空闲连接
        poolConfig.setMinIdle(0);        // 最小空闲连接
        poolConfig.setMaxWaitMillis(1000); // 等待超时时间

        jedisPool = new JedisPool(
            poolConfig,
            "127.0.0.1",  // Redis 地址
            6379,          // 端口
            1000,          // 连接超时
            "123321"       // 密码
        );
    }

    public static Jedis getJedis() {
        return jedisPool.getResource();
    }
}
```

使用时从连接池获取连接，用完调用 `close()` 归还（不是真正关闭）：

```java
@BeforeEach
void setUp() {
    jedis = JedisConnectionFactory.getJedis();
}

@AfterEach
void tearDown() {
    if (jedis != null) {
        jedis.close(); // 归还连接池
    }
}
```

---

## 3.3 SpringDataRedis 快速入门

SpringDataRedis 是 Spring 对 Redis 客户端的封装，提供统一的 `RedisTemplate` API，底层支持 Jedis 和 Lettuce。

官网：https://spring.io/projects/spring-data-redis

**主要功能：**
- 统一 API，屏蔽底层客户端差异
- 支持发布订阅
- 支持哨兵和集群
- 支持响应式编程（Lettuce）
- 支持多种序列化方式

### 引入依赖

```xml
<!--redis 依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!--连接池依赖-->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
<!--Jackson（用于 JSON 序列化）-->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

### 配置 Redis

```yaml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password: 123321
    lettuce:
      pool:
        max-active: 8   # 最大连接数
        max-idle: 8     # 最大空闲连接
        min-idle: 0     # 最小空闲连接
        max-wait: 100ms # 等待超时
```

### 注入并使用 RedisTemplate

```java
@SpringBootTest
class RedisTest {

    @Autowired
    private RedisTemplate redisTemplate;

    @Test
    void testString() {
        // 写入 String 数据
        redisTemplate.opsForValue().set("name", "张三");
        // 读取 String 数据
        Object name = redisTemplate.opsForValue().get("name");
        System.out.println("name = " + name);
    }
}
```

### RedisTemplate 的 API 分类

| 方法 | 对应数据类型 |
|------|------------|
| `opsForValue()` | String |
| `opsForHash()` | Hash |
| `opsForList()` | List |
| `opsForSet()` | Set |
| `opsForZSet()` | SortedSet |

---

## 3.4 RedisSerializer 配置

### 默认序列化的问题

RedisTemplate 默认使用 **JDK 序列化**，存入 Redis 的数据是这样的：

```
\xac\xed\x00\x05t\x00\x06\xe5\xbc\xa0\xe4\xb8\x89
```

缺点：
- **可读性差**，在 Redis 客户端看不懂
- **内存占用大**，JDK 序列化体积比 JSON 大很多

### 自定义序列化（JSON）

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        // JSON 序列化工具
        GenericJackson2JsonRedisSerializer jsonSerializer =
            new GenericJackson2JsonRedisSerializer();

        // key 使用 String 序列化
        template.setKeySerializer(RedisSerializer.string());
        template.setHashKeySerializer(RedisSerializer.string());

        // value 使用 JSON 序列化
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);

        return template;
    }
}
```

配置后存入 Redis 的数据变成可读的 JSON：

```json
{
  "@class": "com.heima.pojo.User",
  "name": "张三",
  "age": 21
}
```

⚠️ **注意**：JSON 序列化会在数据中记录 `@class` 类名，用于反序列化时还原对象类型，但这会带来额外的内存开销。

---

## 3.5 StringRedisTemplate

### 为什么用 StringRedisTemplate

为了避免 JSON 序列化带来的 `@class` 额外开销，可以使用 `StringRedisTemplate`：

- key 和 value 都用 **String 序列化**
- 存储 Java 对象时，**手动完成序列化和反序列化**
- 不会写入 `@class` 信息，节省内存

### 使用方式

```java
@SpringBootTest
class StringRedisTemplateTest {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    // Jackson 序列化工具
    private static final ObjectMapper mapper = new ObjectMapper();

    @Test
    void testSaveUser() throws JsonProcessingException {
        User user = new User("张三", 21);

        // 手动序列化为 JSON 字符串
        String json = mapper.writeValueAsString(user);

        // 存入 Redis
        stringRedisTemplate.opsForValue().set("user:1", json);

        // 从 Redis 读取
        String jsonUser = stringRedisTemplate.opsForValue().get("user:1");

        // 手动反序列化为 Java 对象
        User result = mapper.readValue(jsonUser, User.class);
        System.out.println("result = " + result);
    }

    @Test
    void testHash() {
        // Hash 操作
        stringRedisTemplate.opsForHash().put("user:2", "name", "李四");
        stringRedisTemplate.opsForHash().put("user:2", "age", "25");

        Map<Object, Object> entries = stringRedisTemplate.opsForHash().entries("user:2");
        System.out.println(entries);  // {name=李四, age=25}
    }
}
```

### 两种方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| `RedisTemplate` + JSON 序列化 | 自动序列化/反序列化，使用方便 | 存储 `@class` 信息，内存占用稍大 |
| `StringRedisTemplate` | 无额外信息，节省内存 | 需要手动序列化/反序列化 |

**实际开发推荐 `StringRedisTemplate`**，手动序列化虽然多几行代码，但更灵活，内存更省。

---

## 面试简答

**Q: Jedis 和 Lettuce 的区别？**

A:
- Jedis：同步阻塞，线程不安全，需要连接池，API 与 Redis 命令同名
- Lettuce：异步非阻塞，线程安全，基于 Netty，SpringBoot 默认使用 Lettuce

**Q: RedisTemplate 默认序列化有什么问题？**

A: 默认使用 JDK 序列化，存入 Redis 的数据可读性差、内存占用大。推荐改为 JSON 序列化（`GenericJackson2JsonRedisSerializer`），或者直接用 `StringRedisTemplate` 手动序列化。

**Q: 为什么推荐用连接池？**

A: Jedis 线程不安全，每次操作都新建连接开销大。连接池复用连接，避免频繁建立/断开的性能损耗，同时控制最大连接数防止资源耗尽。
