# MySQL 索引

## 1. MySQL 索引的底层结构是什么？

### 什么是索引
索引就像书的目录，帮你快速找到想要的内容。没有索引，MySQL 就得从头到尾翻一遍表（全表扫描）。

### MySQL 索引使用 B+ 树

**为什么不用其他数据结构？**

| 数据结构 | 查询时间 | 为什么不用 |
|---------|---------|-----------|
| 数组 | O(n) | 太慢，需要遍历 |
| 哈希表 | O(1) | 不支持范围查询 |
| 二叉搜索树 | O(log n) | 树太高，磁盘 I/O 次数多 |
| 红黑树 | O(log n) | 树太高，磁盘 I/O 次数多 |
| **B+ 树** | **O(log n)** | **树矮胖，I/O 少，支持范围查询** |

### B+ 树的特点

```
                    [10, 20, 30]           ← 非叶子节点（只存索引）
                   /    |    |    \
                  /     |    |     \
         [1,5,8] [10,15,18] [20,25,28] [30,35,38]  ← 叶子节点（存数据）
            ↔        ↔         ↔          ↔         ← 叶子节点形成链表
```

**① 所有数据都在叶子节点**
- 非叶子节点只存索引，不存数据
- 叶子节点存完整的行数据（聚簇索引）或主键值（非聚簇索引）

**② 叶子节点形成有序链表**
- 方便范围查询（如 `WHERE id BETWEEN 10 AND 30`）
- 方便排序（`ORDER BY`）

**③ 树矮胖，I/O 次数少**
- 一个节点可以存很多索引（通常几百个）
- 3 层 B+ 树可以存储上千万条数据

**示例：**
```sql
-- 创建索引
CREATE INDEX idx_age ON employee(age);

-- 查询时走索引
SELECT * FROM employee WHERE age = 25;
```

**查询过程：**
1. 从根节点开始，找到 25 在哪个区间
2. 到对应的子节点，继续查找
3. 最终到叶子节点，找到 age=25 的数据

**磁盘 I/O 次数 = 树的高度**（通常 3-4 次）

### B+ 树 vs B 树

| 特性 | B 树 | B+ 树 |
|------|------|-------|
| 数据存储位置 | 所有节点都存数据 | 只有叶子节点存数据 |
| 叶子节点链表 | 无 | 有 |
| 范围查询 | 需要中序遍历 | 直接遍历叶子节点链表 |
| 单个节点存储量 | 少（因为存数据） | 多（只存索引） |

**为什么 MySQL 选择 B+ 树？**
- 范围查询更快（叶子节点链表）
- 树更矮（单个节点存更多索引）
- 查询性能更稳定（都要到叶子节点）

---

## 2. 为什么 MySQL 选择使用 B+ 树作为索引的数据结构？

### 核心原因：减少磁盘 I/O

**磁盘 I/O 是数据库的性能瓶颈。**

- 内存访问：纳秒级
- 磁盘访问：毫秒级（慢 10 万倍）

**B+ 树的优势：**

**① 树矮胖，I/O 次数少**

假设一个节点 16KB（InnoDB 页大小），索引列是 INT（4 字节），指针 6 字节：
- 一个节点可以存：16KB / (4+6) ≈ 1600 个索引
- 3 层 B+ 树可以存：1600 × 1600 × 1600 ≈ 40 亿条数据

**查询 40 亿条数据，只需要 3 次磁盘 I/O！**

**② 支持范围查询**

```sql
-- 范围查询
SELECT * FROM employee WHERE age BETWEEN 20 AND 30;
```

B+ 树的叶子节点是有序链表，直接遍历即可。

**③ 支持排序**

```sql
-- 排序查询
SELECT * FROM employee ORDER BY age;
```

叶子节点本身就是有序的，不需要额外排序。

**④ 查询性能稳定**

- 所有查询都要到叶子节点
- 查询时间复杂度稳定在 O(log n)

### 为什么不用哈希表？

**哈希表的问题：**
- ❌ 不支持范围查询（`BETWEEN`、`>`、`<`）
- ❌ 不支持排序（`ORDER BY`）
- ❌ 不支持最左前缀匹配（`LIKE 'abc%'`）
- ❌ 哈希冲突需要额外处理

```sql
-- 这些查询哈希索引都不支持
SELECT * FROM employee WHERE age > 25;        -- ❌
SELECT * FROM employee WHERE age BETWEEN 20 AND 30;  -- ❌
SELECT * FROM employee ORDER BY age;          -- ❌
SELECT * FROM employee WHERE name LIKE '张%';  -- ❌
```

⚠️ **注意：** MySQL 的 Memory 引擎支持哈希索引，但 InnoDB 只支持 B+ 树索引。

---

## 3. 为什么使用索引能够加速查询？

### 没有索引的查询（全表扫描）

```sql
SELECT * FROM employee WHERE age = 25;
```

**执行过程：**
1. 从第一行开始，逐行检查 age 是否等于 25
2. 扫描完所有行，返回结果

**时间复杂度：** O(n)，n 是表的行数

**示例：** 100 万行数据，需要扫描 100 万行

### 有索引的查询

```sql
CREATE INDEX idx_age ON employee(age);
SELECT * FROM employee WHERE age = 25;
```

**执行过程：**
1. 在 B+ 树索引中查找 age=25（3-4 次磁盘 I/O）
2. 找到对应的数据页，返回结果

**时间复杂度：** O(log n)

**示例：** 100 万行数据，只需要 3-4 次磁盘 I/O

### 性能对比

| 数据量 | 全表扫描 | 索引查询 | 提升倍数 |
|--------|---------|---------|---------|
| 1 万 | 10,000 次 | 3 次 | 3,333 倍 |
| 100 万 | 1,000,000 次 | 4 次 | 250,000 倍 |
| 1 亿 | 100,000,000 次 | 5 次 | 20,000,000 倍 |

### 索引加速的本质

**① 减少扫描的数据量**
- 全表扫描：扫描所有行
- 索引查询：只扫描匹配的行

**② 减少磁盘 I/O**
- 全表扫描：读取所有数据页
- 索引查询：只读取索引页和匹配的数据页

**③ 避免排序**
- 索引本身是有序的，`ORDER BY` 可以直接利用索引顺序

---

## 4. MySQL 索引是通过什么形式来存储的？

### InnoDB 的两种索引

**① 聚簇索引（Clustered Index）**
- 索引和数据存在一起
- 叶子节点存储完整的行数据
- 一个表只有一个聚簇索引（主键索引）

**② 非聚簇索引（Secondary Index）**
- 索引和数据分开存储
- 叶子节点存储主键值
- 一个表可以有多个非聚簇索引

### 聚簇索引（主键索引）

```sql
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    age INT
);
```

**存储结构：**
```
                    [10, 20, 30]           ← 非叶子节点（只存主键）
                   /    |    |    \
                  /     |    |     \
    [1,张三,25] [10,李四,30] [20,王五,28] [30,赵六,35]  ← 叶子节点（存完整行数据）
```

**特点：**
- 数据按主键顺序存储
- 查询主键时，直接返回完整行数据
- 不需要回表

```sql
-- 走聚簇索引，不需要回表
SELECT * FROM employee WHERE id = 10;
```

### 非聚簇索引（辅助索引）

```sql
CREATE INDEX idx_age ON employee(age);
```

**存储结构：**
```
                    [20, 30, 40]           ← 非叶子节点（只存 age）
                   /    |    |    \
                  /     |    |     \
         [25→1] [30→10] [28→20] [35→30]   ← 叶子节点（存 age + 主键 id）
```

**特点：**
- 叶子节点只存索引列和主键值
- 查询时需要回表（根据主键再查一次聚簇索引）

```sql
-- 走非聚簇索引，需要回表
SELECT * FROM employee WHERE age = 25;
```

**查询过程：**
1. 在 age 索引中找到 age=25 的记录，得到主键 id=1
2. 根据主键 id=1，在聚簇索引中查找完整行数据（回表）

### 回表的性能影响

**什么是回表？**
- 先在非聚簇索引中找到主键
- 再根据主键在聚簇索引中查找完整数据

**回表的开销：**
- 需要两次 B+ 树查询
- 增加磁盘 I/O

**如何避免回表？** 使用覆盖索引

```sql
-- 需要回表（查询 name 列）
SELECT id, name, age FROM employee WHERE age = 25;

-- 不需要回表（只查询索引列）
SELECT id, age FROM employee WHERE age = 25;

-- 创建覆盖索引，避免回表
CREATE INDEX idx_age_name ON employee(age, name);
SELECT id, name, age FROM employee WHERE age = 25;  -- 不需要回表
```

---

## 5. MySQL 联合索引的结构是怎样的？

### 什么是联合索引

**联合索引（复合索引）：** 对多个列建立的索引

```sql
CREATE INDEX idx_dept_age ON employee(dept_id, age);
```

### 联合索引的存储结构

**B+ 树按照索引列的顺序排序：**

```
                [部门1,年龄30]
               /              \
    [部门1,年龄25]          [部门2,年龄28]
    [部门1,年龄30]          [部门2,年龄35]
    [部门1,年龄35]          [部门3,年龄25]
```

**排序规则：**
1. 先按第一列（dept_id）排序
2. 第一列相同时，按第二列（age）排序

**类比：** 就像字典，先按拼音首字母排序，首字母相同再按第二个字母排序。

### 最左前缀原则

**联合索引 (dept_id, age) 可以支持的查询：**

```sql
-- ✅ 走索引（使用 dept_id）
SELECT * FROM employee WHERE dept_id = 1;

-- ✅ 走索引（使用 dept_id 和 age）
SELECT * FROM employee WHERE dept_id = 1 AND age = 25;

-- ✅ 走索引（使用 dept_id，age 用于过滤）
SELECT * FROM employee WHERE dept_id = 1 AND age > 25;

-- ❌ 不走索引（没有使用 dept_id）
SELECT * FROM employee WHERE age = 25;
```

**为什么？**
- 联合索引是先按 dept_id 排序的
- 如果不指定 dept_id，就无法利用索引的有序性

**类比：** 字典按拼音排序，如果你只知道第二个字母，无法快速查找。

### 索引覆盖

**联合索引可以避免回表：**

```sql
-- 创建联合索引
CREATE INDEX idx_dept_age_name ON employee(dept_id, age, name);

-- 查询的列都在索引中，不需要回表
SELECT dept_id, age, name FROM employee WHERE dept_id = 1;
```

### 联合索引的设计原则

**① 区分度高的列放前面**

```sql
-- ✅ 好的设计（dept_id 区分度高）
CREATE INDEX idx_dept_age ON employee(dept_id, age);

-- ❌ 不好的设计（age 区分度低）
CREATE INDEX idx_age_dept ON employee(age, dept_id);
```

**② 常用的查询条件放前面**

```sql
-- 如果经常查询 dept_id，少查询 age
CREATE INDEX idx_dept_age ON employee(dept_id, age);

-- 如果经常查询 age，少查询 dept_id
CREATE INDEX idx_age_dept ON employee(age, dept_id);
```

**③ 尽量使用联合索引，减少单列索引**

```sql
-- ❌ 不好的设计（两个单列索引）
CREATE INDEX idx_dept ON employee(dept_id);
CREATE INDEX idx_age ON employee(age);

-- ✅ 好的设计（一个联合索引）
CREATE INDEX idx_dept_age ON employee(dept_id, age);
```

---

## 6. MySQL 建立索引在什么情况下会失效？

### 索引失效的常见场景

**① 违反最左前缀原则**

```sql
-- 联合索引 (dept_id, age)
CREATE INDEX idx_dept_age ON employee(dept_id, age);

-- ❌ 索引失效（没有使用 dept_id）
SELECT * FROM employee WHERE age = 25;

-- ✅ 索引生效
SELECT * FROM employee WHERE dept_id = 1 AND age = 25;
```

**② 在索引列上使用函数**

```sql
-- ❌ 索引失效（对 age 使用了函数）
SELECT * FROM employee WHERE YEAR(hire_date) = 2020;

-- ✅ 索引生效（改写查询）
SELECT * FROM employee WHERE hire_date >= '2020-01-01' AND hire_date < '2021-01-01';
```

**③ 在索引列上进行计算**

```sql
-- ❌ 索引失效（对 age 进行了计算）
SELECT * FROM employee WHERE age + 1 = 26;

-- ✅ 索引生效（改写查询）
SELECT * FROM employee WHERE age = 25;
```

**④ 使用 != 或 <> 操作符**

```sql
-- ❌ 索引失效（使用了 !=）
SELECT * FROM employee WHERE dept_id != 1;

-- ✅ 索引生效（改写查询）
SELECT * FROM employee WHERE dept_id > 1 OR dept_id < 1;
```

**⚠️ 注意：** 主键使用 != 不会失效

**⑤ 使用 IS NULL 或 IS NOT NULL**

```sql
-- ❌ 可能失效（取决于 NULL 值的比例）
SELECT * FROM employee WHERE manager IS NULL;

-- ✅ 避免使用 NULL，用默认值代替
ALTER TABLE employee MODIFY manager INT NOT NULL DEFAULT 0;
```

**⑥ 使用 LIKE 以通配符开头**

```sql
-- ❌ 索引失效（通配符在开头）
SELECT * FROM employee WHERE name LIKE '%张%';

-- ✅ 索引生效（通配符在结尾）
SELECT * FROM employee WHERE name LIKE '张%';
```

**⑦ 使用 OR 连接条件**

```sql
-- ❌ 索引失效（OR 连接的列没有索引）
SELECT * FROM employee WHERE dept_id = 1 OR name = '张三';

-- ✅ 索引生效（OR 连接的列都有索引）
CREATE INDEX idx_name ON employee(name);
SELECT * FROM employee WHERE dept_id = 1 OR name = '张三';
```

**⑧ 类型隐式转换**

```sql
-- ❌ 索引失效（phone 是 VARCHAR，但查询用了数字）
SELECT * FROM employee WHERE phone = 13800138000;

-- ✅ 索引生效（使用字符串）
SELECT * FROM employee WHERE phone = '13800138000';
```

**⑨ 范围查询后的列索引失效**

```sql
-- 联合索引 (dept_id, age, name)
CREATE INDEX idx_dept_age_name ON employee(dept_id, age, name);

-- age 使用了范围查询，name 的索引失效
SELECT * FROM employee WHERE dept_id = 1 AND age > 25 AND name = '张三';
```

**⑩ 优化器认为全表扫描更快**

```sql
-- 如果表数据量很小，或者查询结果占比很大，优化器可能选择全表扫描
SELECT * FROM employee WHERE dept_id = 1;  -- 如果 90% 的数据都是 dept_id=1
```

---

## 7. MySQL 中优化器是如何进行索引的选择？

### 什么是优化器

**优化器（Optimizer）：** MySQL 的"大脑"，负责选择最优的执行计划。

```sql
-- 查看执行计划
EXPLAIN SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

### 优化器的选择依据

**① 索引的区分度（Cardinality）**

```sql
-- 查看索引的区分度
SHOW INDEX FROM employee;
```

**Cardinality：** 索引列的唯一值数量
- 区分度高：Cardinality 接近表的行数（如主键）
- 区分度低：Cardinality 远小于表的行数（如性别）

**优化器倾向于选择区分度高的索引。**

**② 索引的选择性**

**选择性 = Cardinality / 表的行数**

- 选择性越高，索引越有效
- 选择性 = 1：最好（如主键）
- 选择性 < 0.1：可能不如全表扫描

**③ 查询的数据量**

```sql
-- 查询少量数据，走索引
SELECT * FROM employee WHERE dept_id = 1;  -- 假设只有 10 行

-- 查询大量数据，可能全表扫描
SELECT * FROM employee WHERE dept_id = 1;  -- 假设有 90 万行
```

**为什么？**
- 走索引需要回表，如果数据量大，回表的开销可能比全表扫描还大

**④ 索引的覆盖情况**

```sql
-- 覆盖索引，不需要回表
SELECT id, dept_id FROM employee WHERE dept_id = 1;

-- 非覆盖索引，需要回表
SELECT * FROM employee WHERE dept_id = 1;
```

**优化器倾向于选择覆盖索引。**

### 强制使用索引

```sql
-- 强制使用索引
SELECT * FROM employee FORCE INDEX(idx_dept_id) WHERE dept_id = 1;

-- 忽略索引
SELECT * FROM employee IGNORE INDEX(idx_dept_id) WHERE dept_id = 1;

-- 建议使用索引（优化器可能不采纳）
SELECT * FROM employee USE INDEX(idx_dept_id) WHERE dept_id = 1;
```

⚠️ **注意：** 一般不建议强制使用索引，优化器通常比人更聪明。

### 优化器选择错误的情况

**① 统计信息不准确**

```sql
-- 更新统计信息
ANALYZE TABLE employee;
```

**② 索引区分度变化**

```sql
-- 重建索引
ALTER TABLE employee DROP INDEX idx_dept_id;
CREATE INDEX idx_dept_id ON employee(dept_id);
```

---

## 8. 优化器什么时候执行索引选择？

### SQL 执行流程

```
客户端发送 SQL
   ↓
连接器：验证身份
   ↓
解析器：词法分析 + 语法分析
   ↓
优化器：生成执行计划（选择索引）  ← 在这里选择索引
   ↓
执行器：调用存储引擎
   ↓
存储引擎：读取数据
```

### 优化器的工作

**① 生成多个执行计划**

```sql
SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

**可能的执行计划：**
- 方案 1：使用 dept_id 索引
- 方案 2：使用 age 索引
- 方案 3：使用联合索引 (dept_id, age)
- 方案 4：全表扫描

**② 评估每个计划的成本**

**成本包括：**
- CPU 成本：比较、排序的开销
- I/O 成本：读取数据页的开销

**③ 选择成本最低的计划**

```sql
-- 查看优化器的选择
EXPLAIN SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

---

## 9. 哪些情况下不适合建立索引？

**① 数据量很小的表**

```sql
-- 表只有几百行，全表扫描更快
CREATE TABLE config (
    id INT PRIMARY KEY,
    key VARCHAR(50),
    value VARCHAR(100)
);
```

**② 频繁更新的列**

```sql
-- 每次更新都要维护索引，开销大
CREATE INDEX idx_update_time ON employee(update_time);
```

**③ 区分度很低的列**

```sql
-- 性别只有两个值，索引没用
CREATE INDEX idx_gender ON employee(gender);  -- ❌
```

**④ 查询中很少使用的列**

```sql
-- 如果从不查询 hobby，不需要索引
CREATE INDEX idx_hobby ON employee(hobby);  -- ❌
```

**⑤ 大字段列**

```sql
-- TEXT、BLOB 字段不适合建索引
CREATE INDEX idx_content ON article(content);  -- ❌

-- 可以对前缀建索引
CREATE INDEX idx_content ON article(content(100));  -- ✅
```

---

## 10. 哪些情况下适合建立索引？

**① 主键列**

```sql
-- 主键自动创建聚簇索引
CREATE TABLE employee (
    id INT PRIMARY KEY
);
```

**② WHERE 条件列**

```sql
-- 经常作为查询条件的列
CREATE INDEX idx_dept_id ON employee(dept_id);
SELECT * FROM employee WHERE dept_id = 1;
```

**③ ORDER BY 排序列**

```sql
-- 经常排序的列
CREATE INDEX idx_age ON employee(age);
SELECT * FROM employee ORDER BY age;
```

**④ JOIN 连接列**

```sql
-- 经常作为连接条件的列
CREATE INDEX idx_dept_id ON employee(dept_id);
SELECT * FROM employee e JOIN department d ON e.dept_id = d.dept_id;
```

**⑤ GROUP BY 分组列**

```sql
-- 经常分组的列
CREATE INDEX idx_dept_id ON employee(dept_id);
SELECT dept_id, COUNT(*) FROM employee GROUP BY dept_id;
```

**⑥ 区分度高的列**

```sql
-- 区分度高的列（如邮箱、手机号）
CREATE INDEX idx_email ON employee(email);
```

---
## 11. MySQL 的索引覆盖是什么及其优点？

### 什么是索引覆盖

**索引覆盖（Covering Index）：** 指一个查询语句的所有字段都从同一个索引中直接获取，而不需要回表查询数据。

想象你在图书馆查书，如果目录上就有你需要的所有信息（书名、作者、出版年份），你就不需要去书架上翻书了。索引覆盖就是这个道理。

### 索引覆盖的优点

**① 减少 I/O 操作**

由于数据可以直接从索引中获取，减少了对数据表的访问。

```sql
-- 没有索引覆盖（需要回表）
SELECT id, name, age, salary FROM employee WHERE dept_id = 1;

-- 有索引覆盖（不需要回表）
CREATE INDEX idx_dept_id_name_age_salary ON employee(dept_id, name, age, salary);
SELECT id, name, age, salary FROM employee WHERE dept_id = 1;
```

**② 提高查询效率**

索引通常比数据表小，读取索引更快。

**③ 减少磁盘空间的使用**

因为避免了数据文件的访问。

**④ 优化查询计划**

使得数据库优化器可以选择更有效的执行计划。

### 索引覆盖的示例

**场景 1：查询索引列**

```sql
-- 创建索引
CREATE INDEX idx_dept_age ON employee(dept_id, age);

-- 查询索引列（索引覆盖）
SELECT dept_id, age FROM employee WHERE dept_id = 1;

-- EXPLAIN 结果
-- Extra: Using index（说明使用了索引覆盖）
```

**场景 2：查询包含主键**

```sql
-- 创建索引
CREATE INDEX idx_dept_id ON employee(dept_id);

-- 查询索引列 + 主键（索引覆盖）
SELECT id, dept_id FROM employee WHERE dept_id = 1;

-- 非聚簇索引的叶子节点包含主键值，所以也是索引覆盖
```

**场景 3：统计查询**

```sql
-- 创建索引
CREATE INDEX idx_dept_id ON employee(dept_id);

-- 统计查询（索引覆盖）
SELECT COUNT(*) FROM employee WHERE dept_id = 1;

-- 只需要扫描索引，不需要访问数据表
```

### 如何判断是否使用了索引覆盖

```sql
EXPLAIN SELECT dept_id, age FROM employee WHERE dept_id = 1;
```

**查看 Extra 字段：**
- `Using index`：使用了索引覆盖
- `Using index condition`：使用了索引，但需要回表
- 没有 `Using index`：没有使用索引覆盖

### 索引覆盖的设计建议

**① 根据查询需求设计索引**

```sql
-- 经常查询 dept_id, name, age
CREATE INDEX idx_dept_name_age ON employee(dept_id, name, age);

-- 查询时使用索引覆盖
SELECT dept_id, name, age FROM employee WHERE dept_id = 1;
```

**② 不要盲目添加列**

```sql
-- ❌ 不好（索引太大）
CREATE INDEX idx_all ON employee(dept_id, name, age, salary, email, phone, address);

-- ✅ 好（只包含常用列）
CREATE INDEX idx_dept_name_age ON employee(dept_id, name, age);
```

**③ 考虑索引大小**

- 索引列越多，索引越大
- 索引越大，维护成本越高
- 需要在查询性能和维护成本之间权衡

### 索引覆盖 vs 回表

**回表查询：**
```sql
-- 创建索引
CREATE INDEX idx_dept_id ON employee(dept_id);

-- 查询非索引列（需要回表）
SELECT * FROM employee WHERE dept_id = 1;

-- 执行过程：
-- 1. 在 dept_id 索引中找到 dept_id=1 的记录，得到主键 id
-- 2. 根据主键 id 在聚簇索引中查找完整行数据（回表）
```

**索引覆盖：**
```sql
-- 创建覆盖索引
CREATE INDEX idx_dept_name ON employee(dept_id, name);

-- 查询索引列（不需要回表）
SELECT id, dept_id, name FROM employee WHERE dept_id = 1;

-- 执行过程：
-- 1. 在 idx_dept_name 索引中找到 dept_id=1 的记录
-- 2. 直接从索引中获取 dept_id 和 name（不需要回表）
```

**性能对比：**
- 回表查询：需要 2 次 B+ 树查询（索引 + 聚簇索引）
- 索引覆盖：只需要 1 次 B+ 树查询（索引）

### 实际应用案例

**案例 1：分页查询优化**

```sql
-- ❌ 慢（需要回表）
SELECT * FROM employee ORDER BY dept_id LIMIT 10000, 10;

-- ✅ 快（索引覆盖 + 延迟关联）
SELECT e.* FROM employee e
INNER JOIN (
    SELECT id FROM employee ORDER BY dept_id LIMIT 10000, 10
) t ON e.id = t.id;
```

**案例 2：统计查询优化**

```sql
-- 创建索引
CREATE INDEX idx_dept_id ON employee(dept_id);

-- 统计查询（索引覆盖）
SELECT dept_id, COUNT(*) FROM employee GROUP BY dept_id;

-- 只需要扫描索引，不需要访问数据表
```

### 总结

使用索引覆盖是提高数据库查询性能的有效手段之一。

**关键点：**
1. 查询的所有字段都在索引中
2. 不需要回表查询数据
3. 减少 I/O 操作，提高查询效率
4. 需要在查询性能和索引维护成本之间权衡


## 12. MySQL 中的索引类型及其用途

### MySQL 的主要索引类型

MySQL 中的主要索引类型及其用途包括：

**① 主键索引（PRIMARY KEY）**

用于唯一标识表中的每一行，并优化数据的访问速度。

```sql
-- 创建主键索引
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    age INT
);

-- 或者在创建表后添加
ALTER TABLE employee ADD PRIMARY KEY (id);
```

**特点：**
- 唯一且不能为 NULL
- 一个表只能有一个主键
- 自动创建聚簇索引
- 查询速度最快

**② 唯一索引（UNIQUE）**

确保数据列中的所有值都是唯一的。

```sql
-- 创建唯一索引
CREATE UNIQUE INDEX idx_email ON employee(email);

-- 或者在创建表时定义
CREATE TABLE employee (
    id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE,
    name VARCHAR(50)
);
```

**特点：**
- 列值必须唯一
- 可以有多个唯一索引
- 允许 NULL 值（但只能有一个 NULL）
- 防止重复数据插入

**③ 普通索引（INDEX）**

最基本的索引类型，用于加速对数据的查询。

```sql
-- 创建普通索引
CREATE INDEX idx_name ON employee(name);

-- 或者
ALTER TABLE employee ADD INDEX idx_age (age);
```

**特点：**
- 没有唯一性限制
- 可以有重复值
- 可以为 NULL
- 最常用的索引类型

**④ 全文索引（FULLTEXT）**

用于全文搜索，特别是在大量文本数据中搜索特定词汇。

```sql
-- 创建全文索引
CREATE FULLTEXT INDEX idx_content ON article(content);

-- 使用全文索引查询
SELECT * FROM article 
WHERE MATCH(content) AGAINST('MySQL 索引' IN NATURAL LANGUAGE MODE);
```

**特点：**
- 只能用于 CHAR、VARCHAR、TEXT 类型
- 适合大文本搜索
- 支持自然语言搜索和布尔搜索
- InnoDB 和 MyISAM 都支持

**使用场景：**
- 文章内容搜索
- 商品描述搜索
- 评论搜索

**⑤ 组合索引（COMPOSITE INDEX）**

包含多个列，用于优化多列的查询条件。

```sql
-- 创建组合索引
CREATE INDEX idx_dept_age ON employee(dept_id, age);

-- 查询时使用组合索引
SELECT * FROM employee WHERE dept_id = 1 AND age = 25;
```

**特点：**
- 遵循最左前缀原则
- 可以减少索引数量
- 提高多条件查询性能

**最左前缀原则：**
```sql
-- 创建组合索引 (dept_id, age, name)
CREATE INDEX idx_dept_age_name ON employee(dept_id, age, name);

-- ✅ 走索引（使用 dept_id）
SELECT * FROM employee WHERE dept_id = 1;

-- ✅ 走索引（使用 dept_id, age）
SELECT * FROM employee WHERE dept_id = 1 AND age = 25;

-- ✅ 走索引（使用 dept_id, age, name）
SELECT * FROM employee WHERE dept_id = 1 AND age = 25 AND name = '张三';

-- ❌ 不走索引（跳过了 dept_id）
SELECT * FROM employee WHERE age = 25;

-- ❌ 不走索引（跳过了 dept_id）
SELECT * FROM employee WHERE name = '张三';
```

**⑥ 空间索引（SPATIAL）**

用于地理空间数据的索引。

```sql
-- 创建空间索引
CREATE TABLE locations (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    location POINT NOT NULL,
    SPATIAL INDEX idx_location (location)
) ENGINE=MyISAM;

-- 插入地理数据
INSERT INTO locations (id, name, location) 
VALUES (1, '北京', ST_GeomFromText('POINT(116.4074 39.9042)'));

-- 查询附近的位置
SELECT name FROM locations
WHERE ST_Distance_Sphere(location, ST_GeomFromText('POINT(116.4074 39.9042)')) < 1000;
```

**特点：**
- 用于 GEOMETRY、POINT、LINESTRING、POLYGON 等空间数据类型
- 支持空间查询（距离、包含、相交等）
- MyISAM 和 InnoDB（MySQL 5.7+）支持

**使用场景：**
- 地图应用
- 位置服务
- 地理信息系统（GIS）

### 索引类型对比

| 索引类型 | 唯一性 | NULL 值 | 数量限制 | 主要用途 |
|---------|--------|---------|---------|---------|
| **主键索引** | 必须唯一 | 不允许 | 一个表只能有一个 | 唯一标识每一行 |
| **唯一索引** | 必须唯一 | 允许一个 NULL | 可以有多个 | 保证列值唯一 |
| **普通索引** | 可以重复 | 允许 | 可以有多个 | 加速查询 |
| **全文索引** | 可以重复 | 允许 | 可以有多个 | 全文搜索 |
| **组合索引** | 可以重复 | 允许 | 可以有多个 | 多列查询优化 |
| **空间索引** | 可以重复 | 不允许 | 可以有多个 | 地理空间查询 |

### 如何选择索引类型

**① 主键索引**
- 每个表都应该有主键
- 通常使用自增 ID

**② 唯一索引**
- 需要保证唯一性的列（如邮箱、手机号）
- 防止重复数据

**③ 普通索引**
- 经常作为查询条件的列
- WHERE、ORDER BY、GROUP BY 的列

**④ 全文索引**
- 需要进行全文搜索的文本列
- 文章内容、商品描述等

**⑤ 组合索引**
- 经常一起出现在查询条件中的多个列
- 减少索引数量，提高查询效率

**⑥ 空间索引**
- 地理位置数据
- 需要进行空间查询的场景

---

## 13. 了解索引下推吗？

### 什么是索引下推

**索引下推（Index Condition Pushdown，ICP）：** 是 MySQL 5.6 引入的一种优化技术，可以在索引遍历过程中，直接对索引中包含的字段先做判断，过滤掉不符合条件的记录，减少回表次数。

### MySQL 5.6 之前的查询流程（没有索引下推）

假设有一个联合索引 `(zipcode, lastname, address)`，查询语句如下：

```sql
SELECT * FROM people_table 
WHERE zipcode='95054' 
  AND lastname LIKE '%etrunia%' 
  AND address LIKE '%Main Street%';
```

**没有索引下推的执行流程：**

1. **通过索引找到 zipcode='95054' 的所有记录**
   - 在索引中定位到 zipcode='95054' 的记录
   - 获取这些记录的主键值

2. **回表查询完整数据**
   - 根据主键值回表，读取完整的行数据
   - 返回到 MySQL 服务端

3. **在 MySQL 服务端判断其他条件**
   - 判断 `lastname LIKE '%etrunia%'`
   - 判断 `address LIKE '%Main Street%'`
   - 如果都符合条件，则返回给客户端

**问题：** 即使 lastname 和 address 不符合条件，也需要回表读取完整数据，造成大量无效的回表操作。

### MySQL 5.6 之后的查询流程（有索引下推）

**使用索引下推的执行流程：**

1. **通过索引找到 zipcode='95054' 的记录**
   - 在索引中定位到 zipcode='95054' 的记录

2. **在索引中直接判断其他条件（索引下推）**
   - 判断 `lastname LIKE '%etrunia%'`
   - 判断 `address LIKE '%Main Street%'`
   - 如果不符合条件，直接 reject 掉，不回表

3. **只对符合条件的记录回表**
   - 只有同时满足三个条件的记录才回表
   - 根据主键值读取完整的行数据

**优点：** 大大减少了回表次数，提高了查询性能。

### 索引下推的优化效果

**① 减少回表次数**

```sql
-- 假设 zipcode='95054' 有 1000 条记录
-- 其中只有 10 条同时满足 lastname 和 address 条件

-- 没有索引下推：回表 1000 次
-- 有索引下推：回表 10 次
```

**② 在 InnoDB 中只针对二级索引有效**

- 聚簇索引（主键索引）本身就包含完整数据，不需要回表
- 索引下推只对二级索引（非聚簇索引）有优化效果

### 如何开启索引下推

**MySQL 5.6+ 默认开启**

```sql
-- 查看是否开启索引下推
SHOW VARIABLES LIKE 'optimizer_switch';
-- 查找 index_condition_pushdown=on

-- 关闭索引下推（不推荐）
SET optimizer_switch = 'index_condition_pushdown=off';

-- 开启索引下推
SET optimizer_switch = 'index_condition_pushdown=on';
```

### 如何判断是否使用了索引下推

使用 EXPLAIN 查看执行计划：

```sql
EXPLAIN SELECT * FROM people_table 
WHERE zipcode='95054' 
  AND lastname LIKE '%etrunia%' 
  AND address LIKE '%Main Street%';
```

**查看 Extra 字段：**
- `Using index condition`：使用了索引下推
- `Using where`：没有使用索引下推，在 MySQL 服务端过滤

### 索引下推的使用场景

**✅ 适合使用索引下推：**

1. **联合索引 + 范围查询**

```sql
-- 联合索引 (age, name)
CREATE INDEX idx_age_name ON employee(age, name);

-- 使用索引下推
SELECT * FROM employee WHERE age > 20 AND name LIKE '张%';

-- age > 20 走索引范围扫描
-- name LIKE '张%' 在索引中直接判断（索引下推）
```

2. **联合索引 + LIKE 查询**

```sql
-- 联合索引 (dept_id, name)
CREATE INDEX idx_dept_name ON employee(dept_id, name);

-- 使用索引下推
SELECT * FROM employee WHERE dept_id = 1 AND name LIKE '%张%';

-- dept_id = 1 走索引
-- name LIKE '%张%' 在索引中直接判断（索引下推）
```

**❌ 不适合使用索引下推：**

1. **主键索引（聚簇索引）**
   - 主键索引本身就包含完整数据，不需要回表

2. **覆盖索引**
   - 查询的所有字段都在索引中，不需要回表

3. **全表扫描**
   - 没有使用索引，无法使用索引下推

### 索引下推 vs 普通索引查询

**示例对比：**

```sql
-- 创建联合索引
CREATE INDEX idx_dept_age ON employee(dept_id, age);

-- 查询
SELECT * FROM employee WHERE dept_id = 1 AND age > 25;
```

**没有索引下推：**
```
1. 在索引中找到 dept_id=1 的所有记录（假设 1000 条）
2. 回表 1000 次，读取完整数据
3. 在 MySQL 服务端判断 age > 25（假设只有 100 条符合）
4. 返回 100 条数据

总回表次数：1000 次
```

**有索引下推：**
```
1. 在索引中找到 dept_id=1 的记录
2. 在索引中直接判断 age > 25（索引下推）
3. 只对符合条件的记录回表（100 条）
4. 返回 100 条数据

总回表次数：100 次
```

**性能提升：** 回表次数减少 90%

### 索引下推的限制

**① 只对二级索引有效**

```sql
-- ✅ 二级索引可以使用索引下推
CREATE INDEX idx_dept_id ON employee(dept_id);

-- ❌ 主键索引不需要索引下推
SELECT * FROM employee WHERE id > 100;
```

**② 不支持子查询**

```sql
-- ❌ 不支持索引下推
SELECT * FROM employee 
WHERE dept_id = 1 
  AND age > (SELECT AVG(age) FROM employee);
```

**③ 不支持存储函数**

```sql
-- ❌ 不支持索引下推
SELECT * FROM employee 
WHERE dept_id = 1 
  AND my_function(age) > 25;
```

### 实际案例

**案例：订单查询优化**

```sql
-- 创建联合索引
CREATE INDEX idx_user_time ON orders(user_id, create_time);

-- 查询用户最近 30 天的订单
SELECT * FROM orders 
WHERE user_id = 1 
  AND create_time >= '2026-04-10' 
  AND create_time < '2026-05-10';
```

**优化效果：**
- 没有索引下推：找到 user_id=1 的所有订单（假设 10000 条），全部回表，再过滤时间
- 有索引下推：在索引中直接过滤时间，只回表符合时间条件的订单（假设 100 条）
- 性能提升：回表次数减少 99%

### 总结

**索引下推的核心思想：**
- 把部分过滤条件"下推"到存储引擎层
- 在索引遍历过程中直接判断条件
- 减少回表次数，提高查询性能

**关键点：**
1. MySQL 5.6+ 默认开启
2. 只对二级索引有效
3. 可以减少回表次数
4. 通过 EXPLAIN 的 Extra 字段查看（Using index condition）

---

## 面试回答模板

**Q: MySQL 索引的底层结构是什么？**

A: MySQL 使用 B+ 树作为索引结构。B+ 树的特点是：
1. 所有数据都在叶子节点，非叶子节点只存索引
2. 叶子节点形成有序链表，支持范围查询
3. 树矮胖，3-4 层可以存储上千万数据，磁盘 I/O 次数少

**Q: 为什么使用索引能够加速查询？**

A: 索引通过 B+ 树结构，将查询时间复杂度从 O(n) 降低到 O(log n)。100 万行数据，全表扫描需要 100 万次，索引查询只需要 3-4 次磁盘 I/O。

**Q: 什么是聚簇索引和非聚簇索引？**

A: 
- **聚簇索引**：索引和数据存在一起，叶子节点存完整行数据，一个表只有一个（主键索引）
- **非聚簇索引**：索引和数据分开，叶子节点存主键值，查询时需要回表

**Q: 什么是最左前缀原则？**

A: 联合索引 (a, b, c) 按照列的顺序排序，查询时必须从最左边的列开始使用。如果跳过左边的列，索引会失效。例如，可以使用 (a)、(a,b)、(a,b,c)，但不能只使用 (b) 或 (c)。

**Q: 索引什么时候会失效？**

A: 常见的索引失效场景：
1. 违反最左前缀原则
2. 在索引列上使用函数或计算
3. 使用 LIKE 以通配符开头
4. 类型隐式转换
5. 使用 != 或 IS NOT NULL
6. 优化器认为全表扫描更快

**Q: 什么情况下不适合建索引？**

A: 
1. 数据量很小的表
2. 频繁更新的列
3. 区分度很低的列（如性别）
4. 查询中很少使用的列

---

## 面试简答版

**索引结构：** B+ 树，树矮胖，I/O 少，支持范围查询

**聚簇索引：** 索引和数据在一起，叶子节点存完整行数据

**非聚簇索引：** 索引和数据分开，叶子节点存主键值，需要回表

**最左前缀原则：** 联合索引必须从最左边的列开始使用

**索引失效：** 函数、计算、类型转换、LIKE 通配符开头、违反最左前缀

**覆盖索引：** 查询的列都在索引中，不需要回表

**回表：** 先在非聚簇索引中找主键，再根据主键查聚簇索引

---

