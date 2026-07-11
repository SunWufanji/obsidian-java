# Redis 常见命令

## 2.1 Redis 数据结构介绍

Redis 是典型的 key-value 数据库，key 一般是字符串，value 支持多种数据类型：

| 数据类型      | 说明                    | 典型应用         |
| --------- | --------------------- | ------------ |
| String    | 字符串，可存文本、数字、JSON      | 缓存、计数器、分布式锁  |
| Hash      | 键值对集合，类似 Java HashMap | 存储对象         |
| List      | 有序可重复列表，类似 LinkedList | 消息队列、朋友圈点赞列表 |
| Set       | 无序不重复集合，类似 HashSet    | 标签、共同好友      |
| SortedSet | 有序不重复集合，带 score 排序    | 排行榜          |

官方命令文档：https://redis.io/commands

---

## 2.2 通用命令

通用命令适用于所有数据类型：

```bash
# 查看所有符合模板的 key（生产环境慎用，会阻塞）
KEYS pattern
KEYS *          # 查看所有 key
KEYS user:*     # 查看所有 user 开头的 key

# 删除指定 key（可批量）
DEL key [key ...]
DEL name age    # 删除 name 和 age 两个 key

# 判断 key 是否存在，存在返回 1，不存在返回 0
EXISTS key

# 设置 key 的过期时间（秒）
EXPIRE key seconds
EXPIRE name 20  # name 这个 key 20 秒后过期

# 查看 key 的剩余有效期（秒）
# 返回 -1 表示永不过期，-2 表示 key 不存在
TTL key

# 查看 key 的数据类型
TYPE key
```

**示例：**
```bash
127.0.0.1:6379> set name zhangsan
OK
127.0.0.1:6379> exists name
(integer) 1
127.0.0.1:6379> expire name 10
(integer) 1
127.0.0.1:6379> ttl name
(integer) 8
127.0.0.1:6379> ttl name
(integer) -2   # 已过期，key 不存在
```

### Key 的命名规范

Redis 没有 Table 概念，用 `:` 分隔的层级结构来区分不同业务的 key：

```
项目名:业务名:类型:id
```

例如：
- `heima:user:1` → 用户 ID 为 1 的数据
- `heima:product:1` → 商品 ID 为 1 的数据

```bash
set heima:user:1 '{"id":1,"name":"Jack","age":21}'
set heima:product:1 '{"id":1,"name":"小米11","price":4999}'
```

图形化客户端会自动将相同前缀的 key 显示为层级结构，一目了然。

---

## 2.3 String 类型

### 三种存储格式

| 格式 | 说明 | 示例 |
|-----|------|------|
| string | 普通字符串 | `"hello"` |
| int | 整数，可自增自减 | `100` |
| float | 浮点数，可自增自减 | `3.14` |

底层都是字节数组，最大不超过 **512MB**。

### 常见命令

```bash
# 添加或修改键值对
SET key value

# 获取值
GET key

# 批量设置
MSET key1 value1 key2 value2 ...

# 批量获取
MGET key1 key2 ...

# 整数自增 1
INCR key

# 整数自增指定步长
INCRBY key increment
INCRBY num 5    # num 加 5

# 浮点数自增指定步长
INCRBYFLOAT key increment

# key 不存在时才设置（分布式锁常用）
SETNX key value

# 设置键值对并指定过期时间（秒）
SETEX key seconds value

# SET 的增强版，可以同时设置过期时间和 NX 条件
SET key value [EX seconds] [NX]
SET name zhangsan EX 30 NX
```

**示例：**
```bash
127.0.0.1:6379> set age 18
OK
127.0.0.1:6379> incr age
(integer) 19
127.0.0.1:6379> incrby age 5
(integer) 24
127.0.0.1:6379> mset k1 v1 k2 v2 k3 v3
OK
127.0.0.1:6379> mget k1 k2 k3
1) "v1"
2) "v2"
3) "v3"
```

---

## 2.4 Hash 类型

### 为什么用 Hash

String 存对象需要序列化为 JSON，修改某个字段时要整体读写：

```
String: key="user:1"  value='{"name":"Jack","age":21}'
→ 修改 age 需要：GET → 反序列化 → 修改 → 序列化 → SET
```

Hash 可以将对象的每个字段独立存储，直接针对单个字段 CRUD：

```
Hash: key="user:1"  field="name" value="Jack"
                    field="age"  value="21"
→ 修改 age 只需：HSET user:1 age 22
```

### 常见命令

```bash
# 设置 hash 中某个 field 的值
HSET key field value [field value ...]
HSET user:1 name Jack age 21

# 获取 hash 中某个 field 的值
HGET key field
HGET user:1 name

# 批量获取多个 field 的值
HMGET key field1 field2 ...

# 获取 hash 中所有 field 和 value
HGETALL key

# 获取 hash 中所有 field
HKEYS key

# 获取 hash 中所有 value
HVALS key

# 让 hash 中某个 field 的值自增
HINCRBY key field increment

# field 不存在时才设置
HSETNX key field value
```

**示例：**
```bash
127.0.0.1:6379> hset user:1 name Jack age 21 email jack@example.com
(integer) 3
127.0.0.1:6379> hget user:1 name
"Jack"
127.0.0.1:6379> hgetall user:1
1) "name"
2) "Jack"
3) "age"
4) "21"
5) "email"
6) "jack@example.com"
127.0.0.1:6379> hincrby user:1 age 1
(integer) 22
```

---

## 2.5 List 类型

### 特点

Redis 的 List 类似 Java 的 **LinkedList**（双向链表）：

- **有序**
- **元素可以重复**
- **插入和删除快**（O(1)）
- **查询速度一般**（O(n)）

### 常见命令

```bash
# 向列表左侧插入元素
LPUSH key element [element ...]

# 向列表右侧插入元素
RPUSH key element [element ...]

# 移除并返回列表左侧第一个元素
LPOP key

# 移除并返回列表右侧第一个元素
RPOP key

# 返回指定范围内的元素（0 是第一个，-1 是最后一个）
LRANGE key start end
LRANGE mylist 0 -1   # 获取所有元素

# 获取列表长度
LLEN key

# 阻塞式弹出，没有元素时等待指定时间（秒），而不是直接返回 nil
BLPOP key [key ...] timeout
BRPOP key [key ...] timeout
```

**示例：**
```bash
127.0.0.1:6379> rpush mylist a b c
(integer) 3
127.0.0.1:6379> lpush mylist x
(integer) 4
127.0.0.1:6379> lrange mylist 0 -1
1) "x"
2) "a"
3) "b"
4) "c"
127.0.0.1:6379> lpop mylist
"x"
```

### 用 List 模拟队列和栈

```
队列（先进先出）：RPUSH 入队 + LPOP 出队
栈（先进后出）：RPUSH 入栈 + RPOP 出栈
```

---

## 2.6 Set 类型

### 特点

Redis 的 Set 类似 Java 的 **HashSet**（value 为 null 的 HashMap）：

- **无序**
- **元素不可重复**
- **查找快**（O(1)）
- **支持交集、并集、差集**

### 常见命令

```bash
# 添加元素
SADD key member [member ...]

# 移除元素
SREM key member [member ...]

# 获取 set 中元素个数
SCARD key

# 判断元素是否在 set 中
SISMEMBER key member

# 获取 set 中所有元素
SMEMBERS key

# 求两个 set 的交集
SINTER key1 key2 ...

# 求两个 set 的差集（key1 有但 key2 没有的）
SDIFF key1 key2 ...

# 求两个 set 的并集
SUNION key1 key2 ...
```

**示例（共同好友）：**
```bash
# 张三的好友
127.0.0.1:6379> sadd zhangsan lisi wangwu zhaoliu
(integer) 3

# 李四的好友
127.0.0.1:6379> sadd lisi wangwu mazi ergou
(integer) 3

# 共同好友（交集）
127.0.0.1:6379> sinter zhangsan lisi
1) "wangwu"

# 张三有但李四没有的好友（差集）
127.0.0.1:6379> sdiff zhangsan lisi
1) "lisi"
2) "zhaoliu"
```

---

## 2.7 SortedSet 类型

### 特点

SortedSet 是**可排序的 Set**，每个元素带有一个 **score（分数）** 属性，基于 score 排序。底层是**跳表 + 哈希表**。

- **可排序**
- **元素不重复**
- **查询速度快**

常用于**排行榜**场景。

### 常见命令

```bash
# 添加元素，如果已存在则更新 score
ZADD key score member [score member ...]
ZADD ranking 85 Jack 89 Lucy 95 Tom

# 删除元素
ZREM key member

# 获取元素的 score
ZSCORE key member

# 获取元素的排名（从 0 开始，升序）
ZRANK key member

# 获取元素的排名（降序）
ZREVRANK key member

# 获取 SortedSet 中元素个数
ZCARD key

# 统计 score 在指定范围内的元素个数
ZCOUNT key min max

# 让元素的 score 自增
ZINCRBY key increment member

# 按 score 升序，获取指定排名范围内的元素
ZRANGE key min max [WITHSCORES]

# 按 score 降序获取（Z 后加 REV）
ZREVRANGE key min max

# 按 score 范围获取元素
ZRANGEBYSCORE key min max

# 求差集、交集、并集
ZDIFF key1 key2
ZINTER key1 key2
ZUNION key1 key2
```

**示例（排行榜）：**
```bash
127.0.0.1:6379> zadd ranking 85 Jack 89 Lucy 82 Rose 95 Tom 78 Jerry
(integer) 5

# 查询前 3 名（降序）
127.0.0.1:6379> zrevrange ranking 0 2 withscores
1) "Tom"
2) "95"
3) "Lucy"
4) "89"
5) "Jack"
6) "85"

# 查询 Jack 的排名（降序，从 0 开始）
127.0.0.1:6379> zrevrank ranking Jack
(integer) 2

# 给 Jack 加 2 分
127.0.0.1:6379> zincrby ranking 2 Jack
"87"
```

---

## 面试简答

**Q: Redis 有哪些数据类型？**

A: 5 种基本类型：String、Hash、List、Set、SortedSet。还有 3 种特殊类型：BitMap（位图）、HyperLogLog（基数统计）、GEO（地理坐标）。

**Q: String、Hash 存储对象各有什么优缺点？**

A:
- String 存 JSON：简单，但修改某个字段需要整体读写，内存占用稍大
- Hash 存字段：可以针对单个字段 CRUD，更灵活，但不适合存嵌套对象

**Q: List 和 SortedSet 的区别？**

A: List 是有序可重复的链表，按插入顺序排列，适合消息队列；SortedSet 是有序不重复集合，按 score 排序，适合排行榜。
