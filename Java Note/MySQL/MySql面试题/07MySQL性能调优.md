# MySQL 性能调优

## 1. 有做过数据库性能测试吗？

### 什么是数据库性能测试

**数据库性能测试：** 评估数据库在不同负载下的性能表现，找出性能瓶颈。

### 常用的性能测试工具

**① sysbench**

**最常用的 MySQL 性能测试工具。**

```bash
# 安装 sysbench
apt-get install sysbench

# 准备测试数据（100 万行）
sysbench /usr/share/sysbench/oltp_read_write.lua \
  --mysql-host=127.0.0.1 \
  --mysql-port=3306 \
  --mysql-user=root \
  --mysql-password=password \
  --mysql-db=test \
  --tables=10 \
  --table-size=100000 \
  prepare

# 运行测试（16 个线程，持续 60 秒）
sysbench /usr/share/sysbench/oltp_read_write.lua \
  --mysql-host=127.0.0.1 \
  --mysql-port=3306 \
  --mysql-user=root \
  --mysql-password=password \
  --mysql-db=test \
  --tables=10 \
  --table-size=100000 \
  --threads=16 \
  --time=60 \
  --report-interval=10 \
  run

# 清理测试数据
sysbench /usr/share/sysbench/oltp_read_write.lua \
  --mysql-host=127.0.0.1 \
  --mysql-db=test \
  cleanup
```

**测试结果：**
```
SQL statistics:
    queries performed:
        read:                            140000
        write:                           40000
        other:                           20000
        total:                           200000
    transactions:                        10000  (166.67 per sec.)
    queries:                             200000 (3333.33 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

Throughput:
    events/s (eps):                      166.67
    time elapsed:                        60.00s
    total number of events:              10000

Latency (ms):
         min:                            5.23
         avg:                            95.87
         max:                            523.45
         95th percentile:                189.23
```

**关键指标：**
- **TPS（Transactions Per Second）**：每秒事务数
- **QPS（Queries Per Second）**：每秒查询数
- **Latency**：延迟（平均、95 分位、99 分位）

**② mysqlslap**

**MySQL 自带的压测工具。**

```bash
# 并发测试
mysqlslap --concurrency=50 --iterations=10 \
  --number-int-cols=5 --number-char-cols=20 \
  --auto-generate-sql \
  --auto-generate-sql-add-autoincrement \
  --auto-generate-sql-load-type=mixed \
  --auto-generate-sql-write-number=1000 \
  --number-of-queries=10000 \
  --host=127.0.0.1 --user=root --password=password
```

**③ pt-query-digest**

**分析慢查询日志，找出性能瓶颈。**

```bash
# 分析慢查询日志
pt-query-digest /var/log/mysql/slow.log
```

### 性能测试的关键指标

**① 吞吐量（Throughput）**
- TPS：每秒事务数
- QPS：每秒查询数

**② 响应时间（Latency）**
- 平均响应时间
- 95 分位响应时间
- 99 分位响应时间

**③ 并发能力**
- 最大并发连接数
- 并发查询性能

**④ 资源使用率**
- CPU 使用率
- 内存使用率
- 磁盘 I/O
- 网络带宽

---

## 2. explain 都有哪些关键字段？

### 什么是 EXPLAIN

**EXPLAIN：** 查看 SQL 的执行计划，分析查询性能。

```sql
EXPLAIN SELECT * FROM employee WHERE dept_id = 1;
```

### EXPLAIN 的关键字段

**① id**

**查询的标识符，表示查询的执行顺序。**

- id 相同：从上往下执行
- id 不同：id 越大越先执行

```sql
EXPLAIN 
SELECT * FROM employee WHERE dept_id IN (
    SELECT dept_id FROM department WHERE dept_name = '技术部'
);
```

**结果：**
```
id | select_type | table      | type
---|-------------|------------|------
1  | PRIMARY     | employee   | ALL
2  | SUBQUERY    | department | ALL
```

**② select_type**

**查询的类型。**

| 类型 | 含义 |
|------|------|
| **SIMPLE** | 简单查询，不包含子查询或 UNION |
| **PRIMARY** | 最外层查询 |
| **SUBQUERY** | 子查询 |
| **DERIVED** | 派生表（FROM 子句中的子查询） |
| **UNION** | UNION 中的第二个或后面的查询 |

**③ table**

**查询的表名。**

**④ type（重要）**

**访问类型，性能从好到差：**

| 类型 | 含义 | 性能 |
|------|------|------|
| **system** | 表只有一行（系统表） | 最好 |
| **const** | 通过主键或唯一索引查询，最多返回一行 | 很好 |
| **eq_ref** | 唯一索引扫描，每次只返回一行 | 很好 |
| **ref** | 非唯一索引扫描，返回匹配的所有行 | 好 |
| **range** | 索引范围扫描（BETWEEN、>、<） | 一般 |
| **index** | 全索引扫描 | 差 |
| **ALL** | 全表扫描 | 最差 |

**示例：**
```sql
-- type = const（最好）
EXPLAIN SELECT * FROM employee WHERE id = 1;

-- type = ref（好）
EXPLAIN SELECT * FROM employee WHERE dept_id = 1;

-- type = range（一般）
EXPLAIN SELECT * FROM employee WHERE age BETWEEN 20 AND 30;

-- type = ALL（最差）
EXPLAIN SELECT * FROM employee WHERE name LIKE '%张%';
```

**⚠️ 优化目标：** 至少达到 range 级别，最好是 ref 或 const。

**⑤ possible_keys**

**可能使用的索引。**

**⑥ key（重要）**

**实际使用的索引。**

- 如果为 NULL，说明没有使用索引

**⑦ key_len**

**使用的索引长度（字节数）。**

- 越短越好
- 可以判断联合索引使用了几个列

**示例：**
```sql
-- 联合索引 (dept_id, age)
CREATE INDEX idx_dept_age ON employee(dept_id, age);

-- 只使用了 dept_id
EXPLAIN SELECT * FROM employee WHERE dept_id = 1;
-- key_len = 4（INT 类型 4 字节）

-- 使用了 dept_id 和 age
EXPLAIN SELECT * FROM employee WHERE dept_id = 1 AND age = 25;
-- key_len = 8（INT + INT = 8 字节）
```

**⑧ ref**

**显示索引的哪一列被使用了。**

**⑨ rows（重要）**

**估计需要扫描的行数。**

- 越少越好
- 不是精确值，是估算值

**⑩ Extra（重要）**

**额外信息，非常重要。**

| 值 | 含义 | 性能 |
|----|------|------|
| **Using index** | 使用覆盖索引，不需要回表 | 很好 |
| **Using where** | 使用 WHERE 过滤 | 一般 |
| **Using index condition** | 使用索引下推 | 好 |
| **Using filesort** | 使用文件排序，没有使用索引排序 | 差 |
| **Using temporary** | 使用临时表 | 差 |

**示例：**
```sql
-- Extra = Using index（覆盖索引，很好）
EXPLAIN SELECT id, dept_id FROM employee WHERE dept_id = 1;

-- Extra = Using filesort（文件排序，差）
EXPLAIN SELECT * FROM employee ORDER BY name;

-- Extra = Using temporary（临时表，差）
EXPLAIN SELECT dept_id, COUNT(*) FROM employee GROUP BY dept_id;
```

### EXPLAIN 的使用示例

```sql
EXPLAIN SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

**结果：**
```
id | select_type | table    | type  | possible_keys | key          | key_len | ref   | rows | Extra
---|-------------|----------|-------|---------------|--------------|---------|-------|------|-------
1  | SIMPLE      | employee | range | idx_dept_age  | idx_dept_age | 8       | NULL  | 100  | Using index condition
```

**分析：**
- **type = range**：索引范围扫描，性能一般
- **key = idx_dept_age**：使用了联合索引
- **key_len = 8**：使用了 dept_id 和 age 两列
- **rows = 100**：估计扫描 100 行
- **Extra = Using index condition**：使用了索引下推

---

## 3. 需要同学们掌握 MySQL 索引优化的常见思路

### 索引优化的核心思路

**① 为 WHERE、ORDER BY、GROUP BY 的列建立索引**

```sql
-- 经常作为查询条件
CREATE INDEX idx_dept_id ON employee(dept_id);

-- 经常排序
CREATE INDEX idx_age ON employee(age);

-- 经常分组
CREATE INDEX idx_dept_id ON employee(dept_id);
```

**② 使用覆盖索引，避免回表**

```sql
-- 需要回表
SELECT * FROM employee WHERE dept_id = 1;

-- 不需要回表（覆盖索引）
SELECT id, dept_id FROM employee WHERE dept_id = 1;

-- 创建覆盖索引
CREATE INDEX idx_dept_name ON employee(dept_id, name);
SELECT id, dept_id, name FROM employee WHERE dept_id = 1;
```

**③ 使用联合索引，遵循最左前缀原则**

```sql
-- 创建联合索引
CREATE INDEX idx_dept_age_name ON employee(dept_id, age, name);

-- ✅ 走索引（使用 dept_id）
SELECT * FROM employee WHERE dept_id = 1;

-- ✅ 走索引（使用 dept_id, age）
SELECT * FROM employee WHERE dept_id = 1 AND age = 25;

-- ✅ 走索引（使用 dept_id, age, name）
SELECT * FROM employee WHERE dept_id = 1 AND age = 25 AND name = '张三';

-- ❌ 不走索引（没有使用 dept_id）
SELECT * FROM employee WHERE age = 25;
```

**④ 避免索引失效**

```sql
-- ❌ 索引失效（使用函数）
SELECT * FROM employee WHERE YEAR(hire_date) = 2020;

-- ✅ 索引生效（改写查询）
SELECT * FROM employee WHERE hire_date >= '2020-01-01' AND hire_date < '2021-01-01';

-- ❌ 索引失效（使用计算）
SELECT * FROM employee WHERE age + 1 = 26;

-- ✅ 索引生效（改写查询）
SELECT * FROM employee WHERE age = 25;

-- ❌ 索引失效（LIKE 通配符开头）
SELECT * FROM employee WHERE name LIKE '%张%';

-- ✅ 索引生效（LIKE 通配符结尾）
SELECT * FROM employee WHERE name LIKE '张%';
```

**⑤ 选择区分度高的列建立索引**

```sql
-- ❌ 区分度低（性别只有两个值）
CREATE INDEX idx_gender ON employee(gender);

-- ✅ 区分度高（邮箱几乎唯一）
CREATE INDEX idx_email ON employee(email);
```

**区分度计算：**
```sql
SELECT COUNT(DISTINCT column_name) / COUNT(*) AS selectivity
FROM table_name;

-- 区分度 > 0.1 才适合建索引
```

**⑥ 控制索引数量**

- 索引越多，写入性能越差
- 每个表建议不超过 5 个索引
- 优先使用联合索引，减少单列索引

**⑦ 定期维护索引**

```sql
-- 分析表，更新统计信息
ANALYZE TABLE employee;

-- 优化表，重建索引
OPTIMIZE TABLE employee;

-- 查看索引使用情况
SELECT * FROM sys.schema_unused_indexes;
```

---

## 4. SQL 语句你应该怎么优化？

### SQL 优化的常见方法

**① 避免 SELECT ***

```sql
-- ❌ 不好（查询所有列）
SELECT * FROM employee WHERE dept_id = 1;

-- ✅ 好（只查询需要的列）
SELECT id, name, salary FROM employee WHERE dept_id = 1;
```

**② 使用 LIMIT 限制返回行数**

```sql
-- ❌ 不好（返回所有行）
SELECT * FROM employee WHERE dept_id = 1;

-- ✅ 好（只返回前 10 行）
SELECT * FROM employee WHERE dept_id = 1 LIMIT 10;
```

**③ 避免在 WHERE 中使用函数**

```sql
-- ❌ 不好（索引失效）
SELECT * FROM employee WHERE YEAR(hire_date) = 2020;

-- ✅ 好（索引生效）
SELECT * FROM employee WHERE hire_date >= '2020-01-01' AND hire_date < '2021-01-01';
```

**④ 使用 JOIN 代替子查询**

```sql
-- ❌ 不好（子查询）
SELECT * FROM employee WHERE dept_id IN (
    SELECT dept_id FROM department WHERE dept_name = '技术部'
);

-- ✅ 好（JOIN）
SELECT e.* FROM employee e
JOIN department d ON e.dept_id = d.dept_id
WHERE d.dept_name = '技术部';
```

**⑤ 避免使用 OR，改用 IN 或 UNION**

```sql
-- ❌ 不好（OR 可能导致索引失效）
SELECT * FROM employee WHERE dept_id = 1 OR dept_id = 2;

-- ✅ 好（IN）
SELECT * FROM employee WHERE dept_id IN (1, 2);

-- ✅ 好（UNION）
SELECT * FROM employee WHERE dept_id = 1
UNION
SELECT * FROM employee WHERE dept_id = 2;
```

**⑥ 使用 EXISTS 代替 IN**

```sql
-- ❌ 不好（IN，子查询返回大量数据）
SELECT * FROM employee WHERE dept_id IN (
    SELECT dept_id FROM department
);

-- ✅ 好（EXISTS，只检查是否存在）
SELECT * FROM employee e WHERE EXISTS (
    SELECT 1 FROM department d WHERE d.dept_id = e.dept_id
);
```

**⑦ 分页查询优化**

```sql
-- ❌ 不好（深分页，扫描大量数据）
SELECT * FROM employee ORDER BY id LIMIT 100000, 10;

-- ✅ 好（使用主键过滤）
SELECT * FROM employee WHERE id > 100000 ORDER BY id LIMIT 10;
```

**⑧ 批量操作代替单条操作**

```sql
-- ❌ 不好（单条插入）
INSERT INTO employee VALUES (1, '张三', 10000);
INSERT INTO employee VALUES (2, '李四', 20000);
INSERT INTO employee VALUES (3, '王五', 30000);

-- ✅ 好（批量插入）
INSERT INTO employee VALUES 
(1, '张三', 10000),
(2, '李四', 20000),
(3, '王五', 30000);
```

**⑨ 使用 UNION ALL 代替 UNION**

```sql
-- ❌ 不好（UNION 会去重，性能差）
SELECT * FROM employee WHERE dept_id = 1
UNION
SELECT * FROM employee WHERE dept_id = 2;

-- ✅ 好（UNION ALL 不去重，性能好）
SELECT * FROM employee WHERE dept_id = 1
UNION ALL
SELECT * FROM employee WHERE dept_id = 2;
```

**⑩ 避免大事务**

```sql
-- ❌ 不好（大事务，锁定时间长）
BEGIN;
UPDATE employee SET salary = salary * 1.1;  -- 更新所有行
COMMIT;

-- ✅ 好（分批更新）
BEGIN;
UPDATE employee SET salary = salary * 1.1 WHERE id BETWEEN 1 AND 1000;
COMMIT;

BEGIN;
UPDATE employee SET salary = salary * 1.1 WHERE id BETWEEN 1001 AND 2000;
COMMIT;
```

---

## 5. 如何提高 MySQL 连接池性能？

### 什么是连接池

**连接池：** 预先创建一定数量的数据库连接，复用连接，避免频繁创建和销毁连接的开销。

### 连接池的关键参数

**① 最小连接数（minIdle）**

- 连接池中保持的最小空闲连接数
- 设置过小：高峰期需要创建新连接，性能下降
- 设置过大：浪费资源

**推荐：** 根据平均并发量设置，通常 5-10 个

**② 最大连接数（maxActive）**

- 连接池中允许的最大连接数
- 设置过小：高峰期连接不够，请求等待
- 设置过大：数据库压力大，可能导致崩溃

**推荐：** 根据数据库最大连接数和应用实例数设置

```
maxActive = 数据库最大连接数 / 应用实例数
```

**③ 连接超时时间（maxWait）**

- 获取连接的最大等待时间
- 设置过小：容易超时
- 设置过大：请求等待时间长

**推荐：** 3-5 秒

**④ 空闲连接回收时间（minEvictableIdleTimeMillis）**

- 连接空闲多久后被回收
- 设置过小：频繁创建和销毁连接
- 设置过大：浪费资源

**推荐：** 30 分钟

**⑤ 连接有效性检测（testOnBorrow、testWhileIdle）**

- testOnBorrow：获取连接时检测是否有效
- testWhileIdle：空闲时检测是否有效

**推荐：** testWhileIdle = true，testOnBorrow = false

### 连接池配置示例（HikariCP）

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/test");
config.setUsername("root");
config.setPassword("password");

// 最小空闲连接数
config.setMinimumIdle(10);

// 最大连接数
config.setMaximumPoolSize(50);

// 连接超时时间（毫秒）
config.setConnectionTimeout(30000);

// 空闲连接最大存活时间（毫秒）
config.setIdleTimeout(600000);

// 连接最大存活时间（毫秒）
config.setMaxLifetime(1800000);

HikariDataSource dataSource = new HikariDataSource(config);
```

### 连接池优化建议

**① 合理设置连接数**

```
最大连接数 = ((核心数 * 2) + 磁盘数)
```

**② 使用连接池监控**

- 监控连接池使用率
- 监控连接等待时间
- 监控连接泄漏

**③ 避免连接泄漏**

```java
// ❌ 不好（连接泄漏）
Connection conn = dataSource.getConnection();
// 忘记关闭连接

// ✅ 好（使用 try-with-resources）
try (Connection conn = dataSource.getConnection()) {
    // 使用连接
}  // 自动关闭连接
```

**④ 使用长连接**

- 避免频繁创建和销毁连接
- 设置合理的连接最大存活时间

**⑤ 选择高性能连接池**

- HikariCP（推荐，性能最好）
- Druid（功能丰富，监控完善）
- C3P0（老牌连接池）

---

## 6. 面试的时候可以适当和面试官讨论 MySQL 性能优化的问题，比较加分

### MySQL 性能优化的综合思路

**① 硬件层面**
- 使用 SSD 代替 HDD
- 增加内存（Buffer Pool）
- 使用更快的 CPU

**② 配置层面**
- 调整 Buffer Pool 大小（物理内存的 50%-80%）
- 调整 redo log 大小
- 调整连接数

**③ 架构层面**
- 读写分离
- 分库分表
- 使用缓存（Redis）

**④ SQL 层面**
- 优化慢 SQL
- 添加索引
- 避免全表扫描

**⑤ 应用层面**
- 使用连接池
- 批量操作
- 异步处理

### 性能优化的实际案例

**案例 1：慢查询优化**

**问题：** 查询用户订单列表很慢

```sql
-- 慢 SQL
SELECT * FROM orders WHERE user_id = 1 ORDER BY create_time DESC LIMIT 10;
```

**分析：**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY create_time DESC LIMIT 10;

-- type = ALL（全表扫描）
-- Extra = Using filesort（文件排序）
```

**优化：**
```sql
-- 创建联合索引
CREATE INDEX idx_user_time ON orders(user_id, create_time);

-- 优化后
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY create_time DESC LIMIT 10;

-- type = ref（索引扫描）
-- Extra = Using index（覆盖索引）
```

**案例 2：深分页优化**

**问题：** 分页查询第 10000 页很慢

```sql
-- 慢 SQL
SELECT * FROM orders ORDER BY id LIMIT 100000, 10;
```

**优化：**
```sql
-- 使用主键过滤
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 10;
```

**案例 3：大表查询优化**

**问题：** 查询大表（1 亿行）很慢

**优化方案：**
- 分库分表
- 使用归档表
- 使用缓存

---


---

## 7. MySQL 优化器是什么，它如何工作？

### 什么是 MySQL 优化器

**MySQL 优化器：** 是 MySQL 服务层的一个核心组件，负责分析 SQL 语句并生成最优的执行计划。

想象你要从北京去上海，可以坐飞机、高铁、开车。优化器就像导航软件，帮你选择最快、最省钱的路线。

### 优化器的位置

```
连接层
  ↓
服务层
  ↓ 连接器
  ↓ 查询缓存（已废弃）
  ↓ 分析器（语法分析）
  ↓ 优化器 ← 在这里
  ↓ 执行器
  ↓
存储引擎层
```

### 优化器的工作流程

**① 接收解析后的 SQL**

分析器将 SQL 解析成语法树后，传递给优化器。

```sql
SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

**② 生成多个执行计划**

优化器会生成多种可能的执行方案：

```
方案 1：使用 dept_id 索引
  - 通过 dept_id 索引找到所有 dept_id=1 的记录
  - 再过滤 age > 25

方案 2：使用 age 索引
  - 通过 age 索引找到所有 age>25 的记录
  - 再过滤 dept_id = 1

方案 3：使用联合索引 (dept_id, age)
  - 直接通过联合索引找到符合条件的记录

方案 4：全表扫描
  - 扫描整个表，逐行判断条件
```

**③ 成本估算**

优化器会估算每个方案的成本（主要是 I/O 成本和 CPU 成本）。

```sql
-- 查看优化器的成本估算
EXPLAIN FORMAT=JSON 
SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

**成本计算因素：**
- **I/O 成本**：读取数据页的次数
- **CPU 成本**：比较、排序、计算的次数
- **索引选择性**：索引能过滤掉多少数据
- **表统计信息**：表的行数、索引的基数等

**④ 选择最优方案**

优化器选择成本最低的执行计划。

```sql
-- 查看优化器选择的执行计划
EXPLAIN SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

**⑤ 生成执行计划**

将选定的方案转换为执行器可以理解的指令。

### 优化器的优化策略

**① 索引选择**

选择最合适的索引。

```sql
-- 假设有两个索引：idx_dept_id 和 idx_age
CREATE INDEX idx_dept_id ON employee(dept_id);
CREATE INDEX idx_age ON employee(age);

-- 优化器会选择选择性更好的索引
SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

**② 表连接顺序优化**

对于多表连接，优化器会选择最优的连接顺序。

```sql
-- 三表连接
SELECT * FROM employee e
JOIN department d ON e.dept_id = d.dept_id
JOIN job j ON e.job_id = j.job_id
WHERE d.dept_name = '技术部';

-- 优化器会决定连接顺序：
-- 方案 1：employee → department → job
-- 方案 2：department → employee → job
-- 方案 3：job → employee → department
```

**连接顺序选择原则：**
- 小表驱动大表
- 有索引的表优先
- 过滤条件多的表优先

**③ 子查询优化**

将子查询转换为 JOIN。

```sql
-- 原始 SQL（子查询）
SELECT * FROM employee 
WHERE dept_id IN (SELECT dept_id FROM department WHERE dept_name = '技术部');

-- 优化器可能转换为 JOIN
SELECT e.* FROM employee e
JOIN department d ON e.dept_id = d.dept_id
WHERE d.dept_name = '技术部';
```

**④ 条件简化**

简化 WHERE 条件。

```sql
-- 原始 SQL
SELECT * FROM employee WHERE age > 20 AND age > 25;

-- 优化器简化为
SELECT * FROM employee WHERE age > 25;
```

**⑤ 常量折叠**

提前计算常量表达式。

```sql
-- 原始 SQL
SELECT * FROM employee WHERE age > 10 + 15;

-- 优化器简化为
SELECT * FROM employee WHERE age > 25;
```

**⑥ 外连接转内连接**

在某些情况下，将外连接转换为内连接。

```sql
-- 原始 SQL（左外连接）
SELECT * FROM employee e
LEFT JOIN department d ON e.dept_id = d.dept_id
WHERE d.dept_name = '技术部';

-- 优化器转换为内连接（因为 WHERE 条件过滤了 NULL）
SELECT * FROM employee e
JOIN department d ON e.dept_id = d.dept_id
WHERE d.dept_name = '技术部';
```

### 优化器的类型

**① 基于规则的优化器（RBO）**

根据预定义的规则选择执行计划。

**特点：**
- 简单、快速
- 不考虑数据分布
- 可能选择次优方案

**② 基于成本的优化器（CBO）**

根据成本估算选择执行计划（MySQL 使用）。

**特点：**
- 考虑数据分布
- 需要统计信息
- 更准确，但计算开销大

### 影响优化器的因素

**① 表统计信息**

```sql
-- 查看表统计信息
SHOW TABLE STATUS LIKE 'employee';

-- 更新统计信息
ANALYZE TABLE employee;
```

**统计信息包括：**
- 表的行数
- 索引的基数（不同值的数量）
- 数据分布

**② 索引统计信息**

```sql
-- 查看索引统计信息
SHOW INDEX FROM employee;
```

**关键字段：**
- `Cardinality`：索引中不同值的数量（基数）
- 基数越高，选择性越好

**③ MySQL 配置参数**

```sql
-- 查看优化器相关参数
SHOW VARIABLES LIKE 'optimizer%';

-- 优化器开关
SHOW VARIABLES LIKE 'optimizer_switch';
```

**常用参数：**
- `optimizer_search_depth`：优化器搜索深度
- `optimizer_prune_level`：是否启用剪枝优化
- `optimizer_switch`：各种优化策略的开关

### 如何干预优化器

**① 使用 FORCE INDEX**

强制使用某个索引。

```sql
-- 强制使用 idx_dept_id 索引
SELECT * FROM employee FORCE INDEX (idx_dept_id)
WHERE dept_id = 1 AND age > 25;
```

**② 使用 IGNORE INDEX**

忽略某个索引。

```sql
-- 忽略 idx_age 索引
SELECT * FROM employee IGNORE INDEX (idx_age)
WHERE dept_id = 1 AND age > 25;
```

**③ 使用 STRAIGHT_JOIN**

强制按照 SQL 中的表顺序连接。

```sql
-- 强制按 employee → department 的顺序连接
SELECT STRAIGHT_JOIN * FROM employee e
JOIN department d ON e.dept_id = d.dept_id;
```

**④ 使用 Hint**

```sql
-- 使用 Hint 提示优化器
SELECT /*+ INDEX(employee idx_dept_id) */ * 
FROM employee 
WHERE dept_id = 1;
```

### 优化器的局限性

**① 统计信息不准确**

```sql
-- 统计信息过时，导致优化器选择错误
-- 解决方案：定期更新统计信息
ANALYZE TABLE employee;
```

**② 复杂查询的成本估算不准**

```sql
-- 多表连接、子查询嵌套时，成本估算可能不准确
-- 解决方案：简化查询、拆分复杂查询
```

**③ 无法预测数据倾斜**

```sql
-- 数据分布不均匀时，优化器可能选择错误
-- 例如：dept_id = 1 的记录占 90%，但优化器不知道
SELECT * FROM employee WHERE dept_id = 1;
```

**④ 不考虑缓存**

```sql
-- 优化器不知道数据是否在 Buffer Pool 中
-- 可能选择了需要更多磁盘 I/O 的方案
```

### 优化器调优建议

**① 保持统计信息准确**

```sql
-- 定期更新统计信息
ANALYZE TABLE employee;

-- 或者开启自动更新
SET GLOBAL innodb_stats_auto_recalc = ON;
```

**② 创建合适的索引**

```sql
-- 根据查询条件创建索引
CREATE INDEX idx_dept_age ON employee(dept_id, age);
```

**③ 避免复杂查询**

```sql
-- ❌ 不好（复杂子查询）
SELECT * FROM employee WHERE dept_id IN (
    SELECT dept_id FROM department WHERE dept_name IN (
        SELECT dept_name FROM dept_config WHERE status = 1
    )
);

-- ✅ 好（简化为 JOIN）
SELECT e.* FROM employee e
JOIN department d ON e.dept_id = d.dept_id
JOIN dept_config dc ON d.dept_name = dc.dept_name
WHERE dc.status = 1;
```

**④ 使用 EXPLAIN 分析**

```sql
-- 查看执行计划
EXPLAIN SELECT * FROM employee WHERE dept_id = 1 AND age > 25;

-- 查看详细的成本信息
EXPLAIN FORMAT=JSON SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

---

## 7. MySQL 中，如何优化大表的查询性能？

### 优化大表查询性能的方法

**① 使用合适的索引**

确保查询中使用到的列已经被索引。

```sql
-- 为经常查询的列创建索引
CREATE INDEX idx_user_id ON orders(user_id);
CREATE INDEX idx_create_time ON orders(create_time);

-- 使用联合索引
CREATE INDEX idx_user_time ON orders(user_id, create_time);
```

**② 分区**

将大表分割成多个物理更小的表。

```sql
-- 按时间分区
CREATE TABLE orders (
    id INT,
    user_id INT,
    amount DECIMAL(10,2),
    create_time DATETIME
)
PARTITION BY RANGE (YEAR(create_time)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);
```

**③ 查询优化**

优化 SQL 查询语句，避免全表扫描。

```sql
-- ❌ 不好（全表扫描）
SELECT * FROM orders WHERE YEAR(create_time) = 2025;

-- ✅ 好（使用索引）
SELECT * FROM orders WHERE create_time >= '2025-01-01' AND create_time < '2026-01-01';
```

**④ 硬件优化**

增加内存和处理器速度，提高 I/O 性能。

---

## 8. MySQL 优化查询性能的常用方法有哪些？

### 优化查询性能的常用方法

**① 使用合适的索引**

为查询中常用的列和查询条件创建索引。

**② 优化查询语句**

确保 SQL 语句尽可能高效，避免使用函数和子查询。

**③ 合理设计表结构**

避免过多的列，合理分割大表。

**④ 使用缓存**

利用 MySQL 的查询缓存机制或应用层缓存（Redis）。

**⑤ 配置优化**

调整 MySQL 服务器配置，优化内存使用和查询执行。

**⑥ 定期维护数据库**

定期进行数据库表的优化和重建索引。

---

## 面试回答模板

**Q: 有做过数据库性能测试吗？**

A: 有，我使用过 sysbench 进行性能测试。主要测试 TPS、QPS、响应时间等指标。通过测试可以评估数据库在不同并发下的性能表现，找出性能瓶颈。

**Q: explain 都有哪些关键字段？**

A: EXPLAIN 的关键字段包括：
- **type**：访问类型，性能从好到差：const > eq_ref > ref > range > index > ALL
- **key**：实际使用的索引
- **rows**：估计扫描的行数
- **Extra**：额外信息，如 Using index（覆盖索引）、Using filesort（文件排序）

**Q: 如何优化 SQL？**

A: SQL 优化的常见方法：
1. 避免 SELECT *，只查询需要的列
2. 使用 LIMIT 限制返回行数
3. 避免在 WHERE 中使用函数
4. 使用 JOIN 代替子查询
5. 使用覆盖索引避免回表
6. 分页查询使用主键过滤

**Q: 如何提高连接池性能？**

A: 连接池优化建议：
1. 合理设置最小和最大连接数
2. 设置合理的连接超时时间
3. 使用连接有效性检测
4. 避免连接泄漏
5. 选择高性能连接池（如 HikariCP）

---

## 面试简答版

**性能测试工具：** sysbench、mysqlslap、pt-query-digest

**EXPLAIN 关键字段：** type、key、rows、Extra

**索引优化：** 覆盖索引、联合索引、最左前缀、避免索引失效

**SQL 优化：** 避免 SELECT *、使用 LIMIT、避免函数、JOIN 代替子查询

**连接池优化：** 合理设置连接数、连接超时、连接检测、避免泄漏
