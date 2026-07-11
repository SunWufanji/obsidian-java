# MySQL 高可用

## 1. 如何实现 MySQL 的读写分离？

### 什么是读写分离

**读写分离：** 将数据库的读操作和写操作分离到不同的数据库服务器上，提高系统的并发能力和可用性。

想象一个图书馆，如果只有一个窗口，借书和还书都要排队。如果分成两个窗口，一个专门借书，一个专门还书，效率就高多了。

### 读写分离的架构

```
应用程序
   ↓
┌─────────────────┐
│   读写分离中间件   │
│  (MyCAT/ShardingSphere)
└─────────────────┘
   ↓          ↓
写操作      读操作
   ↓          ↓
主库(Master)  从库(Slave)
   ↓          ↓
数据同步 →  从库1、从库2、从库3
```

### 读写分离的实现方式

**① 应用层实现**

在应用代码中判断是读操作还是写操作，分别连接不同的数据库。

```java
// 写操作连接主库
DataSource masterDataSource = getMasterDataSource();
Connection conn = masterDataSource.getConnection();
conn.executeUpdate("INSERT INTO user VALUES (1, '张三')");

// 读操作连接从库
DataSource slaveDataSource = getSlaveDataSource();
Connection conn = slaveDataSource.getConnection();
ResultSet rs = conn.executeQuery("SELECT * FROM user WHERE id = 1");
```

**优点：**
- 实现简单
- 灵活控制

**缺点：**
- 代码侵入性强
- 维护成本高

**② 中间件实现（推荐）**

使用数据库中间件（如 MyCAT、ShardingSphere）自动路由读写请求。

```yaml
# ShardingSphere 配置
spring:
  shardingsphere:
    datasource:
      names: master,slave0,slave1
      master:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://master:3306/test
      slave0:
        jdbc-url: jdbc:mysql://slave0:3306/test
      slave1:
        jdbc-url: jdbc:mysql://slave1:3306/test
    rules:
      readwrite-splitting:
        data-sources:
          myds:
            write-data-source-name: master
            read-data-source-names: slave0,slave1
            load-balancer-name: round-robin
```

**优点：**
- 对应用透明
- 支持负载均衡
- 支持故障转移

**③ 代理层实现**

使用数据库代理（如 ProxySQL、MaxScale）。

```sql
-- ProxySQL 配置
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (1, 'master', 3306);
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (2, 'slave1', 3306);
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (2, 'slave2', 3306);

-- 配置读写分离规则
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup) 
VALUES (1, 1, '^SELECT.*FOR UPDATE$', 1);  -- 写操作到主库
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup) 
VALUES (2, 1, '^SELECT', 2);  -- 读操作到从库
```

### 读写分离的负载均衡策略

**① 轮询（Round Robin）**

按顺序依次分配到每个从库。

```
请求1 → 从库1
请求2 → 从库2
请求3 → 从库3
请求4 → 从库1
```

**② 随机（Random）**

随机选择一个从库。

**③ 权重（Weight）**

根据从库的性能分配不同的权重。

```yaml
slave0: weight=3  # 性能好，分配更多请求
slave1: weight=1  # 性能差，分配较少请求
```

**④ 最少连接（Least Connections）**

选择当前连接数最少的从库。

### 读写分离的注意事项

**① 主从延迟问题**

```java
// 问题：刚写入的数据，立即读取可能读不到
userService.createUser(user);  // 写入主库
User user = userService.getUser(userId);  // 从从库读取，可能还没同步

// 解决方案1：强制读主库
@ReadFromMaster
User user = userService.getUser(userId);

// 解决方案2：延迟读取
Thread.sleep(100);  // 等待主从同步
User user = userService.getUser(userId);

// 解决方案3：使用缓存
cache.put(userId, user);  // 写入缓存
User user = cache.get(userId);  // 从缓存读取
```

**② 事务一致性问题**

```java
// 事务中的读操作应该读主库，保证一致性
@Transactional
public void transfer(int fromId, int toId, int amount) {
    // 这些读操作应该读主库，不能读从库
    Account from = accountDao.getById(fromId);  // 强制读主库
    Account to = accountDao.getById(toId);      // 强制读主库
    
    from.setBalance(from.getBalance() - amount);
    to.setBalance(to.getBalance() + amount);
    
    accountDao.update(from);
    accountDao.update(to);
}
```

---

## 2. MySQL 主从复制的原理是啥？

### 什么是主从复制

**主从复制：** 将主库的数据变更自动同步到从库，实现数据的备份和读写分离。

### 主从复制的原理

```
主库(Master)
   ↓
写入 binlog
   ↓
从库(Slave) 的 I/O 线程读取 binlog
   ↓
写入 relay log（中继日志）
   ↓
从库的 SQL 线程读取 relay log
   ↓
执行 SQL，更新数据
```

### 主从复制的三个线程

**① 主库的 binlog dump 线程**

- 读取主库的 binlog
- 发送给从库的 I/O 线程

**② 从库的 I/O 线程**

- 连接主库
- 读取主库的 binlog
- 写入从库的 relay log

**③ 从库的 SQL 线程**

- 读取 relay log
- 执行 SQL 语句
- 更新从库数据

### 主从复制的配置

**① 主库配置**

```ini
# my.cnf
[mysqld]
# 开启 binlog
log-bin=mysql-bin
# 服务器 ID（唯一）
server-id=1
# binlog 格式（推荐 ROW）
binlog-format=ROW
```

```sql
-- 创建复制用户
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

-- 查看主库状态
SHOW MASTER STATUS;
-- 记录 File 和 Position
```

**② 从库配置**

```ini
# my.cnf
[mysqld]
# 服务器 ID（唯一）
server-id=2
# 开启 relay log
relay-log=mysql-relay-bin
# 只读模式
read-only=1
```

```sql
-- 配置主库信息
CHANGE MASTER TO
  MASTER_HOST='master_host',
  MASTER_USER='repl',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

-- 启动复制
START SLAVE;

-- 查看从库状态
SHOW SLAVE STATUS\G
```

### 主从复制的模式

**① 异步复制（默认）**

```
主库写入 binlog → 返回客户端 → 从库异步同步
```

**特点：**
- 性能最好
- 可能丢失数据（主库崩溃，binlog 未同步到从库）

**② 半同步复制**

```
主库写入 binlog → 至少一个从库接收到 binlog → 返回客户端
```

**配置：**
```sql
-- 主库安装半同步插件
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;

-- 从库安装半同步插件
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

**特点：**
- 数据更安全
- 性能略低于异步复制

**③ 全同步复制（MySQL Cluster）**

```
主库写入 binlog → 所有从库都接收到 binlog → 返回客户端
```

**特点：**
- 数据最安全
- 性能最差

---

## 3. MySQL 主从同步延时问题如何解决？

### 什么是主从延迟

**主从延迟：** 主库的数据变更同步到从库需要时间，导致从库的数据落后于主库。

### 主从延迟的原因

**① 从库性能差**

- 从库硬件配置低
- 从库负载高

**② 主库写入量大**

- 主库 TPS 很高
- 从库的 SQL 线程来不及执行

**③ 大事务**

- 主库执行了一个大事务（如更新 100 万行）
- 从库需要很长时间才能执行完

**④ 网络延迟**

- 主从之间的网络延迟高

### 如何监控主从延迟

```sql
-- 查看从库状态
SHOW SLAVE STATUS\G

-- 关键字段
Seconds_Behind_Master: 5  -- 延迟秒数（0 表示无延迟）
```

### 主从延迟的解决方案

**① 升级从库硬件**

- 使用 SSD
- 增加内存
- 使用更快的 CPU

**② 优化从库配置**

```ini
# my.cnf
[mysqld]
# 增加 SQL 线程数（MySQL 5.7+）
slave-parallel-type=LOGICAL_CLOCK
slave-parallel-workers=4

# 增加 Buffer Pool
innodb_buffer_pool_size=8G
```

**③ 避免大事务**

```sql
-- ❌ 不好（大事务）
BEGIN;
UPDATE user SET status = 1;  -- 更新 100 万行
COMMIT;

-- ✅ 好（分批更新）
BEGIN;
UPDATE user SET status = 1 WHERE id BETWEEN 1 AND 10000;
COMMIT;

BEGIN;
UPDATE user SET status = 1 WHERE id BETWEEN 10001 AND 20000;
COMMIT;
```

**④ 使用并行复制**

MySQL 5.7+ 支持并行复制，多个 SQL 线程并行执行。

```sql
-- 开启并行复制
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 4;
```

**⑤ 强制读主库**

对于实时性要求高的查询，强制读主库。

```java
@ReadFromMaster
public User getUser(int userId) {
    return userDao.getById(userId);
}
```

**⑥ 使用缓存**

```java
// 写入时更新缓存
public void createUser(User user) {
    userDao.insert(user);
    cache.put(user.getId(), user);  // 更新缓存
}

// 读取时先查缓存
public User getUser(int userId) {
    User user = cache.get(userId);
    if (user == null) {
        user = userDao.getById(userId);  // 从数据库读取
        cache.put(userId, user);
    }
    return user;
}
```

---

## 4. 分库分表是什么？

### 什么是分库分表

**分库分表：** 将一个大表拆分成多个小表，分散到不同的数据库中，提高系统的并发能力和存储能力。

### 为什么需要分库分表

**① 单表数据量过大**

- 单表超过 1000 万行，查询性能下降
- 索引过大，维护成本高

**② 单库并发量过大**

- 单库 TPS 达到瓶颈
- 连接数不够

**③ 单库存储容量不够**

- 单库存储空间不足

### 分库分表的策略

**① 垂直分库**

**按业务模块拆分数据库。**

```
原来：
  ┌─────────────┐
  │   数据库     │
  │  - 用户表    │
  │  - 订单表    │
  │  - 商品表    │
  └─────────────┘

垂直分库后：
  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │ 用户库   │  │ 订单库   │  │ 商品库   │
  │ - 用户表 │  │ - 订单表 │  │ - 商品表 │
  └─────────┘  └─────────┘  └─────────┘
```

**优点：**
- 降低单库压力
- 业务隔离

**缺点：**
- 跨库 JOIN 困难
- 分布式事务

**② 垂直分表**

**按字段拆分表。**

```
原来：
  user 表：id, name, age, avatar, description

垂直分表后：
  user_base 表：id, name, age
  user_ext 表：id, avatar, description
```

**优点：**
- 减少单表字段数
- 提高查询性能

**③ 水平分库**

**按数据拆分到多个数据库。**

```
原来：
  ┌─────────────┐
  │   数据库     │
  │  - user 表   │
  │    (1000万行)│
  └─────────────┘

水平分库后：
  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │ 数据库1  │  │ 数据库2  │  │ 数据库3  │
  │ - user表 │  │ - user表 │  │ - user表 │
  │ (300万行)│  │ (300万行)│  │ (400万行)│
  └─────────┘  └─────────┘  └─────────┘
```

**④ 水平分表**

**按数据拆分到多个表。**

```
原来：
  user 表（1000万行）

水平分表后：
  user_0 表（250万行）
  user_1 表（250万行）
  user_2 表（250万行）
  user_3 表（250万行）
```

### 分库分表的分片规则

**① 按范围分片**

```
user_id 1-1000000    → user_0
user_id 1000001-2000000 → user_1
user_id 2000001-3000000 → user_2
```

**优点：**
- 实现简单
- 扩容方便

**缺点：**
- 数据分布不均匀

**② 按哈希分片**

```
user_id % 4 = 0 → user_0
user_id % 4 = 1 → user_1
user_id % 4 = 2 → user_2
user_id % 4 = 3 → user_3
```

**优点：**
- 数据分布均匀

**缺点：**
- 扩容困难（需要重新分片）

**③ 按地理位置分片**

```
北京用户 → db_beijing
上海用户 → db_shanghai
广州用户 → db_guangzhou
```

**④ 按时间分片**

```
2024年订单 → order_2024
2025年订单 → order_2025
2026年订单 → order_2026
```

### 分库分表的问题

**① 跨库 JOIN**

```sql
-- 原来：单库 JOIN
SELECT u.name, o.order_no
FROM user u
JOIN order o ON u.id = o.user_id;

-- 分库后：无法 JOIN
-- 解决方案：应用层 JOIN
List<User> users = userDao.getUsers();
List<Order> orders = orderDao.getOrders();
// 在应用层关联
```

**② 分布式事务**

```java
// 原来：单库事务
@Transactional
public void createOrder(Order order) {
    userDao.updateBalance(order.getUserId(), -order.getAmount());
    orderDao.insert(order);
}

// 分库后：跨库事务
// 解决方案：使用分布式事务（Seata）或最终一致性
```

**③ 分布式 ID**

```sql
-- 原来：自增 ID
CREATE TABLE user (
    id INT AUTO_INCREMENT PRIMARY KEY
);

-- 分库后：无法使用自增 ID
-- 解决方案：雪花算法、UUID、号段模式
```

**④ 分页查询**

```sql
-- 原来：单库分页
SELECT * FROM user ORDER BY id LIMIT 10 OFFSET 100;

-- 分库后：需要从每个库查询，再合并排序
-- 性能较差
```

---

## 5. 分布式 ID 你学习到的都有哪些？

### 什么是分布式 ID

**分布式 ID：** 在分布式系统中，生成全局唯一的 ID。

### 分布式 ID 的要求

**① 全局唯一**

不能重复。

**② 趋势递增**

方便排序，提高数据库插入性能。

**③ 高性能**

生成速度快。

**④ 高可用**

不能因为 ID 生成服务挂掉而影响业务。

### 分布式 ID 的实现方案

**① UUID**

```java
String id = UUID.randomUUID().toString();
// 输出：550e8400-e29b-41d4-a716-446655440000
```

**优点：**
- 实现简单
- 性能高
- 本地生成，无需网络请求

**缺点：**
- 不是递增的
- 字符串类型，占用空间大
- 无序，插入数据库性能差

**② 数据库自增 ID**

```sql
CREATE TABLE id_generator (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    stub CHAR(1) NOT NULL DEFAULT ''
) ENGINE=MyISAM;

-- 获取 ID
INSERT INTO id_generator (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```

**优点：**
- 实现简单
- 递增有序

**缺点：**
- 性能瓶颈（单点）
- 可用性问题（数据库挂了就无法生成 ID）

**③ 号段模式**

```java
// 一次性从数据库获取一批 ID（如 1-1000）
// 应用本地分配 ID，用完再获取下一批
public class SegmentIdGenerator {
    private long currentId = 0;
    private long maxId = 0;
    
    public synchronized long nextId() {
        if (currentId >= maxId) {
            // 从数据库获取下一批 ID
            Segment segment = dao.getNextSegment();
            currentId = segment.getMinId();
            maxId = segment.getMaxId();
        }
        return ++currentId;
    }
}
```

**优点：**
- 性能高（减少数据库访问）
- 递增有序

**缺点：**
- 应用重启会浪费一批 ID

**④ 雪花算法（Snowflake）**

```
64 位 ID 结构：
┌─────────────────────────────────────────────────────────────┐
│ 1位符号位 │ 41位时间戳 │ 10位机器ID │ 12位序列号 │
│    0     │ 时间戳     │ 机器ID    │ 序列号    │
└─────────────────────────────────────────────────────────────┘
```

**实现：**
```java
public class SnowflakeIdGenerator {
    private long workerId;  // 机器 ID（0-1023）
    private long sequence = 0L;  // 序列号（0-4095）
    private long lastTimestamp = -1L;
    
    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("时钟回拨");
        }
        
        if (timestamp == lastTimestamp) {
            // 同一毫秒内，序列号自增
            sequence = (sequence + 1) & 4095;
            if (sequence == 0) {
                // 序列号用完，等待下一毫秒
                timestamp = waitNextMillis(lastTimestamp);
            }
        } else {
            // 不同毫秒，序列号重置
            sequence = 0L;
        }
        
        lastTimestamp = timestamp;
        
        // 组装 ID
        return ((timestamp - 1288834974657L) << 22)
             | (workerId << 12)
             | sequence;
    }
}
```

**优点：**
- 性能高（本地生成）
- 趋势递增
- 不依赖数据库

**缺点：**
- 时钟回拨问题
- 需要分配机器 ID

**⑤ Redis 生成 ID**

```java
public long nextId() {
    return redisTemplate.opsForValue().increment("id_generator");
}
```

**优点：**
- 性能高
- 递增有序

**缺点：**
- 依赖 Redis
- Redis 挂了无法生成 ID

**⑥ 美团 Leaf**

Leaf 是美团开源的分布式 ID 生成系统，支持号段模式和雪花算法。

```java
// 号段模式
long id = leafSegmentService.getId("order");

// 雪花算法
long id = leafSnowflakeService.getId();
```

---

## 6. MySQL 分布式锁如何实现？

### 什么是分布式锁

**分布式锁：** 在分布式系统中，多个节点需要访问共享资源时，通过锁机制保证同一时刻只有一个节点能访问。

**使用场景：**
- 防止重复下单
- 防止库存超卖
- 定时任务防止重复执行
- 分布式事务

### MySQL 实现分布式锁的方式

**① 基于唯一索引**

```sql
-- 创建锁表
CREATE TABLE distributed_lock (
    lock_name VARCHAR(64) NOT NULL COMMENT '锁名称',
    lock_value VARCHAR(64) NOT NULL COMMENT '锁持有者',
    expire_time BIGINT NOT NULL COMMENT '过期时间',
    PRIMARY KEY (lock_name)
) ENGINE=InnoDB;

-- 获取锁（插入记录）
INSERT INTO distributed_lock (lock_name, lock_value, expire_time)
VALUES ('order_lock', 'server1', UNIX_TIMESTAMP() + 30);

-- 释放锁（删除记录）
DELETE FROM distributed_lock WHERE lock_name = 'order_lock' AND lock_value = 'server1';
```

**优点：**
- 实现简单
- 利用数据库唯一索引保证互斥

**缺点：**
- 性能较差（数据库压力大）
- 需要处理锁超时问题
- 单点故障（数据库挂了锁就失效）

**② 基于 FOR UPDATE**

```sql
-- 创建锁表
CREATE TABLE distributed_lock (
    id INT PRIMARY KEY,
    lock_name VARCHAR(64) NOT NULL,
    lock_value VARCHAR(64),
    expire_time BIGINT
) ENGINE=InnoDB;

-- 插入一条记录
INSERT INTO distributed_lock (id, lock_name) VALUES (1, 'order_lock');

-- 获取锁
BEGIN;
SELECT * FROM distributed_lock WHERE id = 1 FOR UPDATE;
-- 执行业务逻辑
COMMIT;  -- 释放锁
```

**优点：**
- 阻塞等待，自动排队
- 事务提交自动释放锁

**缺点：**
- 长事务会导致锁等待时间长
- 数据库压力大

**③ 基于乐观锁（版本号）**

```sql
-- 创建锁表
CREATE TABLE distributed_lock (
    lock_name VARCHAR(64) PRIMARY KEY,
    version INT NOT NULL DEFAULT 0,
    lock_value VARCHAR(64),
    expire_time BIGINT
) ENGINE=InnoDB;

-- 获取锁（更新版本号）
UPDATE distributed_lock 
SET version = version + 1, lock_value = 'server1', expire_time = UNIX_TIMESTAMP() + 30
WHERE lock_name = 'order_lock' AND (lock_value IS NULL OR expire_time < UNIX_TIMESTAMP());

-- 检查影响行数
-- 如果影响行数为 1，说明获取锁成功
-- 如果影响行数为 0，说明获取锁失败
```

**优点：**
- 不阻塞，性能较好
- 适合冲突较少的场景

**缺点：**
- 需要重试机制
- 不适合高并发场景

### MySQL 分布式锁的问题

**① 性能问题**

数据库压力大，不适合高并发场景。

**② 单点故障**

数据库挂了，锁就失效了。

**③ 锁超时问题**

业务执行时间超过锁超时时间，锁被自动释放。

**④ 不可重入**

同一个线程无法重复获取锁。

### MySQL 分布式锁 vs Redis 分布式锁

| 特性 | MySQL 分布式锁 | Redis 分布式锁 |
|------|---------------|---------------|
| **性能** | 较差 | 很好 |
| **实现复杂度** | 简单 | 中等 |
| **可靠性** | 依赖数据库 | 依赖 Redis |
| **适用场景** | 低并发 | 高并发 |

**推荐：** 高并发场景使用 Redis 分布式锁，低并发场景可以使用 MySQL 分布式锁。

---

## 7. MySQL 数据迁移有哪些方案？

### 什么是数据迁移

**数据迁移：** 将数据从一个数据库迁移到另一个数据库，或者从旧表迁移到新表。

**常见场景：**
- 数据库升级
- 分库分表
- 数据中心迁移
- 表结构变更

### 数据迁移的方案

**① mysqldump + mysql**

**最简单的迁移方式，适合小数据量。**

```bash
# 导出数据
mysqldump -h source_host -u root -p database_name > backup.sql

# 导入数据
mysql -h target_host -u root -p database_name < backup.sql
```

**优点：**
- 实现简单
- 支持跨版本迁移

**缺点：**
- 导出导入期间数据库不可用
- 大数据量迁移时间长

**② SELECT INTO OUTFILE + LOAD DATA INFILE**

**适合大数据量迁移。**

```sql
-- 导出数据
SELECT * INTO OUTFILE '/tmp/employee.csv'
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM employee;

-- 导入数据
LOAD DATA INFILE '/tmp/employee.csv'
INTO TABLE employee
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

**优点：**
- 速度快
- 适合大数据量

**缺点：**
- 需要文件系统权限
- 不支持跨服务器

**③ 主从复制迁移**

**适合在线迁移，不停机。**

```
旧库（主库）
   ↓
配置主从复制
   ↓
新库（从库）
   ↓
数据同步完成
   ↓
切换应用到新库
```

**步骤：**
```sql
-- 1. 在旧库配置主库
-- my.cnf
[mysqld]
log-bin=mysql-bin
server-id=1

-- 2. 在新库配置从库
-- my.cnf
[mysqld]
server-id=2

-- 3. 配置主从复制
CHANGE MASTER TO
  MASTER_HOST='old_host',
  MASTER_USER='repl',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

START SLAVE;

-- 4. 等待数据同步完成
SHOW SLAVE STATUS\G

-- 5. 切换应用到新库
-- 6. 停止主从复制
STOP SLAVE;
```

**优点：**
- 在线迁移，不停机
- 数据一致性好
- 可以回滚

**缺点：**
- 配置复杂
- 需要主从同步时间

**④ 双写方案**

**适合分库分表场景。**

```java
// 同时写入旧库和新库
public void createUser(User user) {
    // 写入旧库
    oldUserDao.insert(user);
    
    // 写入新库
    try {
        newUserDao.insert(user);
    } catch (Exception e) {
        log.error("写入新库失败", e);
    }
}

// 先读新库，读不到再读旧库
public User getUser(int userId) {
    User user = newUserDao.getById(userId);
    if (user == null) {
        user = oldUserDao.getById(userId);
    }
    return user;
}
```

**迁移步骤：**
1. 双写（旧库 + 新库）
2. 迁移历史数据
3. 切换读取（先读新库，读不到再读旧库）
4. 验证数据一致性
5. 停止写入旧库

**优点：**
- 在线迁移
- 可以逐步切换
- 可以回滚

**缺点：**
- 代码侵入性强
- 需要保证双写一致性

**⑤ 使用数据迁移工具**

**使用专业的数据迁移工具。**

- **Canal**：阿里开源，基于 binlog 的增量数据同步工具
- **DataX**：阿里开源，支持多种数据源的离线数据同步工具
- **Debezium**：基于 CDC 的数据变更捕获工具
- **AWS DMS**：AWS 数据库迁移服务
- **gh-ost**：GitHub 开源的在线表结构变更工具

**Canal 示例：**
```
旧库
  ↓
Canal 监听 binlog
  ↓
解析 binlog
  ↓
写入新库
```

**优点：**
- 功能强大
- 支持多种数据源
- 支持增量同步

**缺点：**
- 学习成本高
- 需要额外部署

### 数据迁移的注意事项

**① 数据一致性**

```sql
-- 迁移前后数据量对比
SELECT COUNT(*) FROM old_table;
SELECT COUNT(*) FROM new_table;

-- 数据校验
SELECT * FROM old_table WHERE id NOT IN (SELECT id FROM new_table);
```

**② 性能影响**

- 避免高峰期迁移
- 分批迁移，避免一次性迁移大量数据
- 使用限流，避免影响线上业务

**③ 回滚方案**

- 保留旧库数据
- 准备回滚脚本
- 灰度切换，逐步验证

**④ 索引和约束**

```sql
-- 先迁移数据，再创建索引（提高导入速度）
-- 1. 删除索引
ALTER TABLE employee DROP INDEX idx_dept_id;

-- 2. 导入数据
LOAD DATA INFILE '/tmp/employee.csv' INTO TABLE employee;

-- 3. 重建索引
ALTER TABLE employee ADD INDEX idx_dept_id (dept_id);
```


**Q: MySQL 分布式锁如何实现？**

A: MySQL 分布式锁有三种实现方式：
1. **基于唯一索引**：插入记录获取锁，删除记录释放锁
2. **基于 FOR UPDATE**：使用行锁，事务提交自动释放
3. **基于乐观锁**：使用版本号，更新时检查版本号

推荐高并发场景使用 Redis 分布式锁，低并发场景可以使用 MySQL 分布式锁。

**Q: MySQL 数据迁移有哪些方案？**

A: 常见的数据迁移方案：
1. **mysqldump + mysql**：适合小数据量，简单但停机时间长
2. **SELECT INTO OUTFILE + LOAD DATA INFILE**：适合大数据量，速度快
3. **主从复制迁移**：在线迁移，不停机，数据一致性好
4. **双写方案**：适合分库分表，可以逐步切换
5. **数据迁移工具**：Canal、DataX、Debezium 等

推荐使用主从复制或数据迁移工具进行在线迁移。

---


## 8. MySQL 常见的高可用架构方案有哪些？


---

## 8. MySQL 中如何实现高可用架构？

### 在 MySQL 中实现高可用架构的常见方式

**① 主从复制**

一个主数据库服务器和一个或多个从数据库服务器。

```
主库（Master）
   ↓ binlog 复制
从库（Slave1、Slave2、Slave3）
```

**特点：**
- 主库负责写入
- 从库负责读取
- 实现读写分离
- 需要手动故障转移

**适用场景：**
- 读多写少的业务
- 对可用性要求不高
- 成本有限

**② 集群部署**

如 MySQL Cluster 或 Galera Cluster，提供故障转移和负载均衡。

```
节点1 ←→ 节点2 ←→ 节点3
  ↓        ↓        ↓
  多主架构，数据同步
```

**特点：**
- 多主架构
- 自动故障转移
- 数据强一致性
- 负载均衡

**适用场景：**
- 对可用性要求高
- 需要多点写入
- 金融、支付等场景

**③ 读写分离**

通过分离读写操作来提高性能和可用性。

```
应用
  ↓
读写分离中间件（ShardingSphere/ProxySQL）
  ↓          ↓
写操作      读操作
  ↓          ↓
主库      从库1、从库2
```

**特点：**
- 写操作到主库
- 读操作到从库
- 提高并发能力
- 对应用透明

**适用场景：**
- 读多写少
- 需要提高并发
- 需要负载均衡

**④ 自动故障转移**

使用代理或中间件自动处理主服务器故障。

```
主库（Master）
   ↓
MHA Manager / Keepalived
   ↓
自动检测故障 → 提升从库为主库
```

**特点：**
- 自动检测故障
- 自动切换主库
- 减少停机时间
- 提高可用性

**适用场景：**
- 不能接受长时间停机
- 需要自动化运维
- 7×24 小时服务

### 高可用架构选择建议

| 需求 | 推荐方案 |
|------|---------|
| **读多写少** | 主从复制 + 读写分离 |
| **高可用要求** | MHA 或 MGR |
| **多点写入** | Galera Cluster 或 MGR（多主模式） |
| **自动故障转移** | MHA、MGR、Keepalived |
| **成本有限** | 主从复制 + Keepalived |
| **金融级可靠性** | MGR 或 Galera Cluster |

### 什么是高可用架构

**高可用架构：** 通过冗余、故障转移等技术，保证系统在部分组件故障时仍能正常提供服务。

**高可用的核心指标：**
- **可用性（Availability）**：系统正常运行时间占比（如 99.99%）
- **RTO（Recovery Time Objective）**：故障恢复时间目标
- **RPO（Recovery Point Objective）**：数据丢失容忍度

### MySQL 高可用架构方案

**① 主从复制（Master-Slave Replication）**

**最基础的高可用方案。**

```
主库（Master）
   ↓ binlog
从库（Slave）
```

**特点：**
- 实现简单
- 读写分离
- 故障需要手动切换

**优点：**
- 配置简单
- 成本低

**缺点：**
- 主库单点故障
- 需要手动故障转移
- 可能丢失数据

**适用场景：**
- 读多写少
- 对可用性要求不高
- 预算有限

**② MHA（Master High Availability）**

**自动故障转移的主从复制方案。**

```
主库（Master）
   ↓
MHA Manager（监控和故障转移）
   ↓
从库1、从库2、从库3
```

**工作流程：**
1. MHA Manager 监控主库
2. 主库故障时，自动选择最新的从库
3. 提升为新主库
4. 其他从库指向新主库

**特点：**
- 自动故障转移
- 数据一致性好
- 故障切换时间短（10-30秒）

**优点：**
- 自动化程度高
- 数据丢失少
- 成熟稳定

**缺点：**
- 需要额外的 MHA Manager 节点
- 配置相对复杂
- 仍然是主从架构

**适用场景：**
- 对可用性要求高
- 需要自动故障转移
- 传统主从架构升级

**③ MGR（MySQL Group Replication）**

**MySQL 官方的多主复制方案。**

```
节点1 ←→ 节点2 ←→ 节点3
  ↓        ↓        ↓
  Paxos 协议保证一致性
```

**特点：**
- 多主架构（可配置单主或多主）
- 基于 Paxos 协议
- 自动故障检测和转移
- 数据强一致性

**单主模式：**
- 只有一个节点可写
- 其他节点只读
- 主节点故障自动选举新主

**多主模式：**
- 所有节点都可写
- 冲突检测和解决
- 适合分布式写入

**优点：**
- 官方支持
- 数据强一致性
- 自动故障转移
- 支持多主写入

**缺点：**
- 性能开销大
- 网络延迟敏感
- 配置复杂

**适用场景：**
- 对数据一致性要求极高
- 需要多主写入
- 金融、支付等场景

**④ Galera Cluster（PXC）**

**基于 Galera 的多主同步复制方案。**

```
节点1 ←→ 节点2 ←→ 节点3
  ↓        ↓        ↓
  Galera 复制协议
```

**特点：**
- 多主架构，所有节点可读可写
- 同步复制，数据强一致性
- 自动故障检测和节点剔除
- 无主从概念

**优点：**
- 真正的多主架构
- 数据强一致性
- 自动故障转移
- 无数据丢失

**缺点：**
- 性能开销大
- 写入冲突需要处理
- 网络要求高

**适用场景：**
- 需要多点写入
- 对数据一致性要求高
- 地理分布式部署

**⑤ MySQL + Keepalived（VIP 漂移）**

**通过虚拟 IP 实现故障转移。**

```
VIP（虚拟 IP）
   ↓
主库（Master）+ Keepalived
   ↓
从库（Slave）+ Keepalived
```

**工作流程：**
1. VIP 绑定到主库
2. 主库故障时，Keepalived 检测到
3. VIP 漂移到从库
4. 从库提升为主库

**优点：**
- 实现简单
- 切换快速
- 对应用透明

**缺点：**
- 需要手动提升从库
- 可能出现脑裂
- 数据一致性保证弱

**适用场景：**
- 简单的高可用需求
- 预算有限
- 对数据一致性要求不高

**⑥ MySQL + ProxySQL（中间件）**

**通过中间件实现读写分离和故障转移。**

```
应用
  ↓
ProxySQL（中间件）
  ↓        ↓
主库    从库1、从库2
```

**特点：**
- 读写分离
- 连接池
- 查询路由
- 故障检测

**优点：**
- 对应用透明
- 支持读写分离
- 连接池管理
- 查询缓存

**缺点：**
- 增加中间层
- 单点故障（ProxySQL）
- 性能开销

**适用场景：**
- 需要读写分离
- 需要连接池管理
- 需要查询路由

### 高可用架构对比

| 方案 | 可用性 | 数据一致性 | 自动故障转移 | 复杂度 | 成本 | 适用场景 |
|------|--------|-----------|-------------|--------|------|---------|
| **主从复制** | 低 | 弱 | 否 | 低 | 低 | 读多写少 |
| **MHA** | 中 | 中 | 是 | 中 | 中 | 传统架构升级 |
| **MGR** | 高 | 强 | 是 | 高 | 高 | 金融、支付 |
| **Galera** | 高 | 强 | 是 | 高 | 高 | 多点写入 |
| **Keepalived** | 中 | 弱 | 半自动 | 低 | 低 | 简单高可用 |
| **ProxySQL** | 中 | 中 | 是 | 中 | 中 | 读写分离 |

### 如何选择高可用方案

**① 根据可用性要求**

- **99.9%（三个九）**：主从复制 + 手动切换
- **99.99%（四个九）**：MHA 或 Keepalived
- **99.999%（五个九）**：MGR 或 Galera

**② 根据数据一致性要求**

- **最终一致性**：主从复制、MHA
- **强一致性**：MGR、Galera

**③ 根据写入模式**

- **单点写入**：主从复制、MHA、MGR（单主模式）
- **多点写入**：MGR（多主模式）、Galera

**④ 根据预算**

- **低预算**：主从复制、Keepalived
- **中预算**：MHA、ProxySQL
- **高预算**：MGR、Galera

---

## 10. 市面上主流数据库有哪些分类？

### 数据库的分类

**按数据模型分类：**

**① 关系型数据库（RDBMS）**

**使用表格存储数据，支持 SQL 查询。**

**代表产品：**
- **MySQL**：开源、社区活跃、生态丰富
- **PostgreSQL**：功能强大、扩展性好、支持 JSON
- **Oracle**：企业级、功能全面、价格昂贵
- **SQL Server**：微软产品、与 .NET 集成好
- **MariaDB**：MySQL 的分支、兼容 MySQL

**特点：**
- ACID 事务
- SQL 标准
- 数据一致性强
- 适合结构化数据

**适用场景：**
- 金融、电商、ERP
- 需要事务保证
- 复杂查询

**② NoSQL 数据库**

**非关系型数据库，不使用 SQL。**

**键值数据库（Key-Value）：**
- **Redis**：内存数据库、高性能、支持多种数据结构
- **Memcached**：纯内存缓存、简单快速
- **Etcd**：分布式键值存储、用于配置管理

**特点：**
- 简单快速
- 高性能
- 适合缓存

**文档数据库（Document）：**
- **MongoDB**：最流行的文档数据库、支持 JSON
- **CouchDB**：支持 HTTP API、离线同步
- **Elasticsearch**：搜索引擎、全文检索

**特点：**
- 灵活的数据结构
- 适合半结构化数据
- 易于扩展

**列式数据库（Column-Family）：**
- **HBase**：基于 Hadoop、大数据场景
- **Cassandra**：分布式、高可用、线性扩展
- **ClickHouse**：OLAP、实时分析

**特点：**
- 列式存储
- 压缩率高
- 适合分析查询

**图数据库（Graph）：**
- **Neo4j**：最流行的图数据库
- **JanusGraph**：分布式图数据库
- **ArangoDB**：多模型数据库

**特点：**
- 适合关系复杂的数据
- 图查询高效
- 社交网络、推荐系统

**③ NewSQL 数据库**

**结合了 RDBMS 和 NoSQL 的优点。**

**代表产品：**
- **TiDB**：分布式、兼容 MySQL、HTAP
- **CockroachDB**：分布式、强一致性、云原生
- **Google Spanner**：全球分布式、强一致性
- **OceanBase**：阿里开源、金融级、高可用

**特点：**
- 支持 SQL
- 分布式架构
- 水平扩展
- ACID 事务

**适用场景：**
- 大规模数据
- 需要 SQL 和事务
- 云原生应用

**④ 时序数据库（Time-Series）**

**专门存储时间序列数据。**

**代表产品：**
- **InfluxDB**：开源、高性能
- **TimescaleDB**：基于 PostgreSQL
- **Prometheus**：监控系统、时序数据库

**特点：**
- 时间戳索引
- 高写入性能
- 数据压缩

**适用场景：**
- 监控数据
- IoT 数据
- 日志分析

### 数据库选型建议

**① 传统业务系统**
- 首选：MySQL、PostgreSQL
- 理由：成熟稳定、生态丰富、人才充足

**② 大数据分析**
- 首选：ClickHouse、HBase
- 理由：列式存储、高性能、适合 OLAP

**③ 缓存场景**
- 首选：Redis、Memcached
- 理由：内存存储、高性能

**④ 搜索场景**
- 首选：Elasticsearch
- 理由：全文检索、分布式

**⑤ 分布式事务**
- 首选：TiDB、CockroachDB
- 理由：分布式、支持 SQL、ACID

**⑥ 社交网络**
- 首选：Neo4j、MongoDB
- 理由：图数据库、灵活的数据结构

---

## 11. MySQL 和分布式数据库的对比分析

### MySQL vs 分布式数据库

**① 架构对比**

**MySQL（单机架构）：**
```
应用
  ↓
MySQL（单机）
  ↓
磁盘
```

**分布式数据库（如 TiDB）：**
```
应用
  ↓
TiDB Server（无状态）
  ↓
PD（调度）
  ↓
TiKV（存储节点1、2、3...）
```

**② 扩展性对比**

| 特性 | MySQL | 分布式数据库 |
|------|-------|-------------|
| **垂直扩展** | 升级硬件（CPU、内存、磁盘） | 支持 |
| **水平扩展** | 需要分库分表 | 自动分片、透明扩展 |
| **扩展上限** | 单机硬件限制 | 几乎无限 |
| **扩展成本** | 高（硬件升级） | 低（增加节点） |

**③ 可用性对比**

| 特性 | MySQL | 分布式数据库 |
|------|-------|-------------|
| **单点故障** | 是（需要主从复制） | 否（多副本） |
| **故障转移** | 手动或半自动（MHA） | 自动 |
| **数据一致性** | 最终一致性 | 强一致性 |
| **RTO** | 分钟级 | 秒级 |
| **RPO** | 可能丢失数据 | 不丢失数据 |

**④ 性能对比**

| 特性 | MySQL | 分布式数据库 |
|------|-------|-------------|
| **单机性能** | 高 | 中（网络开销） |
| **集群性能** | 需要分库分表 | 线性扩展 |
| **读性能** | 主从复制 | 多副本读 |
| **写性能** | 单主写入 | 分布式写入 |
| **事务性能** | 高 | 中（分布式事务） |

**⑤ 功能对比**

| 特性 | MySQL | 分布式数据库 |
|------|-------|-------------|
| **SQL 支持** | 完整 | 部分（兼容 MySQL） |
| **事务** | ACID | ACID |
| **索引** | B+ 树 | LSM 树 |
| **分布式事务** | 不支持 | 支持 |
| **跨节点 JOIN** | 不支持 | 支持 |

**⑥ 运维对比**

| 特性 | MySQL | 分布式数据库 |
|------|-------|-------------|
| **部署复杂度** | 低 | 高 |
| **运维成本** | 低 | 高 |
| **监控** | 简单 | 复杂 |
| **备份恢复** | 简单 | 复杂 |
| **升级** | 简单 | 复杂 |

**⑦ 成本对比**

| 特性 | MySQL | 分布式数据库 |
|------|-------|-------------|
| **硬件成本** | 高（单机高配） | 低（多台低配） |
| **软件成本** | 免费（开源） | 部分免费 |
| **人力成本** | 低（人才充足） | 高（人才稀缺） |
| **学习成本** | 低 | 高 |

### 什么时候选择 MySQL

**① 数据量不大（< 1TB）**

单机 MySQL 完全够用。

**② 业务简单**

不需要复杂的分布式特性。

**③ 预算有限**

MySQL 免费，运维成本低。

**④ 团队熟悉 MySQL**

学习成本低，上手快。

**⑤ 对性能要求高**

单机 MySQL 性能优于分布式数据库。

### 什么时候选择分布式数据库

**① 数据量大（> 10TB）**

单机 MySQL 无法支撑。

**② 需要水平扩展**

业务增长快，需要动态扩容。

**③ 高可用要求**

需要自动故障转移，不能接受停机。

**④ 跨地域部署**

需要多地域多活。

**⑤ 分布式事务**

需要跨节点的 ACID 事务。

### MySQL 到分布式数据库的迁移

**① 评估是否需要迁移**

- 数据量是否超过单机限制
- 是否需要水平扩展
- 是否需要高可用
- 团队是否有能力运维

**② 选择合适的分布式数据库**

- **TiDB**：兼容 MySQL、HTAP、开源
- **CockroachDB**：强一致性、云原生
- **OceanBase**：金融级、高可用

**③ 迁移方案**

**方案 1：双写 + 数据同步**
```
应用
  ↓ 双写
MySQL + 分布式数据库
  ↓
逐步切换读流量
  ↓
停止写入 MySQL
```

**方案 2：使用数据同步工具**
```
MySQL
  ↓ Canal/DTS
分布式数据库
  ↓
验证数据一致性
  ↓
切换应用
```

**④ 迁移注意事项**

- SQL 兼容性（部分语法可能不支持）
- 性能差异（分布式事务性能较低）
- 运维复杂度（需要学习新的运维工具）
- 成本增加（硬件、人力）

### 总结

**MySQL 的优势：**
- 成熟稳定
- 生态丰富
- 人才充足
- 成本低

**分布式数据库的优势：**
- 水平扩展
- 高可用
- 分布式事务
- 云原生

**选择建议：**
- 中小型业务：MySQL
- 大型业务：分布式数据库
- 混合使用：MySQL（OLTP）+ ClickHouse（OLAP）

## 面试简答版（更新）

**读写分离：** 写操作到主库，读操作到从库，使用中间件实现

**主从复制：** binlog dump 线程、I/O 线程、SQL 线程，三种模式：异步、半同步、全同步

**主从延迟：** 升级硬件、并行复制、避免大事务、强制读主库、使用缓存

**分库分表：** 垂直分库、垂直分表、水平分库、水平分表，分片规则：范围、哈希、地理、时间

**分布式 ID：** UUID、数据库自增、号段模式、雪花算法、Redis、美团 Leaf

**分布式锁：** 唯一索引、FOR UPDATE、乐观锁，推荐 Redis 分布式锁

**数据迁移：** mysqldump、主从复制、双写方案、Canal/DataX 工具
