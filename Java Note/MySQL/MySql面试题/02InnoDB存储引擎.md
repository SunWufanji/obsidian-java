# InnoDB 存储引擎

## 1. MySQL 行记录的存储格式有哪些？

### 什么是行格式
行格式就是一行数据在磁盘上的存储方式。就像你整理衣柜，可以选择叠放、挂起、卷起来，MySQL 也有不同的方式来存储一行数据。

### InnoDB 的四种行格式

| 行格式 | 特点 | 适用场景 |
|--------|------|---------|
| **Compact** | 紧凑存储，节省空间 | MySQL 5.1+ 默认格式 |
| **Redundant** | 冗余存储，兼容老版本 | MySQL 5.0 之前的格式 |
| **Dynamic** | 动态存储，大字段溢出 | MySQL 5.7+ 默认格式 |
| **Compressed** | 压缩存储，节省磁盘 | 数据量大、读多写少 |

```sql
-- 查看表的行格式
SHOW TABLE STATUS LIKE 'employee';

-- 创建表时指定行格式
CREATE TABLE test (
    id INT PRIMARY KEY,
    name VARCHAR(100)
) ROW_FORMAT=DYNAMIC;

-- 修改表的行格式
ALTER TABLE employee ROW_FORMAT=COMPACT;
```

### Compact 行格式（重点）

**存储结构：**
```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│  变长字段长度  │  NULL 标志位  │   记录头信息   │   实际数据    │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

**① 变长字段长度列表**
- 记录 VARCHAR、TEXT 等变长字段的实际长度
- 逆序存放（方便从前往后读）

**② NULL 标志位**
- 用位图标记哪些列是 NULL
- 1 个字节可以标记 8 个列

**③ 记录头信息（5 字节）**
- deleted_flag：是否删除
- min_rec_flag：是否是 B+ 树非叶子节点的最小记录
- n_owned：当前记录拥有的记录数
- heap_no：在页中的位置
- record_type：记录类型（0=普通，1=B+树非叶节点，2=最小记录，3=最大记录）
- next_record：下一条记录的相对位置

**④ 实际数据**
- 隐藏列：row_id（6字节）、trx_id（6字节）、roll_pointer（7字节）
- 用户定义的列数据

**示例：**
```sql
CREATE TABLE user (
    id INT PRIMARY KEY,
    name VARCHAR(20),
    age INT,
    email VARCHAR(50)
) ROW_FORMAT=COMPACT;

INSERT INTO user VALUES (1, '张三', 25, 'zhangsan@qq.com');
```

**存储示意：**
```
变长字段长度: [15, 6]  -- email 15字节，name 6字节（逆序）
NULL标志位: 00000000   -- 都不是 NULL
记录头: [5字节]
隐藏列: row_id(6) + trx_id(6) + roll_pointer(7)
实际数据: 1 + '张三' + 25 + 'zhangsan@qq.com'
```

### Dynamic 行格式（MySQL 5.7+ 默认）

**和 Compact 的区别：**
- 大字段（TEXT、BLOB、长 VARCHAR）完全溢出到溢出页
- 行记录中只保留 20 字节指针

**Compact 的处理：**
- 前 768 字节存在行记录中
- 剩余部分存到溢出页

**Dynamic 的处理：**
- 整个字段都存到溢出页
- 行记录中只保留指针

```sql
-- Dynamic 格式更适合大字段
CREATE TABLE article (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT  -- 大字段会溢出
) ROW_FORMAT=DYNAMIC;
```

### Compressed 行格式

**特点：**
- 在 Dynamic 基础上增加压缩
- 使用 zlib 算法压缩
- 节省磁盘空间，但增加 CPU 开销

```sql
-- 适合归档数据、日志表
CREATE TABLE logs (
    id INT PRIMARY KEY,
    log_content TEXT
) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;
```

⚠️ **注意：** 压缩格式需要设置 `innodb_file_per_table=ON`

---

## 2. MySQL 中 NULL 值如何存储？

### NULL 的存储方式

**不占用实际数据空间，只占用 NULL 标志位。**

```sql
CREATE TABLE test (
    id INT PRIMARY KEY,
    name VARCHAR(20),
    age INT,
    email VARCHAR(50)
) ROW_FORMAT=COMPACT;

-- 插入包含 NULL 的数据
INSERT INTO test VALUES (1, '张三', NULL, NULL);
```

**存储示意：**
```
变长字段长度: [6]      -- 只有 name 有长度
NULL标志位: 00001100   -- age 和 email 是 NULL（从右往左数第3、4位是1）
记录头: [5字节]
实际数据: 1 + '张三'   -- age 和 email 不占空间
```

### NULL 的影响

**① 索引影响**
```sql
-- NULL 值可以存在索引中
CREATE INDEX idx_age ON test(age);

-- 查询 NULL 值
SELECT * FROM test WHERE age IS NULL;  -- 可以走索引
```

**② 统计影响**
```sql
-- COUNT(*) 统计所有行
SELECT COUNT(*) FROM test;  -- 包括 NULL

-- COUNT(列名) 不统计 NULL
SELECT COUNT(age) FROM test;  -- 不包括 age 为 NULL 的行
```

**③ 比较影响**
```sql
-- NULL 和任何值比较都是 NULL（不是 FALSE）
SELECT * FROM test WHERE age = NULL;     -- ❌ 查不到结果
SELECT * FROM test WHERE age IS NULL;    -- ✅ 正确写法

-- NULL 参与运算结果是 NULL
SELECT age + 10 FROM test;  -- age 是 NULL，结果也是 NULL
```

⚠️ **建议：** 尽量避免使用 NULL，可以用默认值代替（如 0、空字符串）

---

## 3. char 和 varchar 的区别是什么？

### 核心区别

| 特性 | CHAR | VARCHAR |
|------|------|---------|
| **长度** | 固定长度 | 可变长度 |
| **存储方式** | 不足部分用空格填充 | 实际长度 + 1-2字节长度前缀 |
| **空间效率** | 浪费空间 | 节省空间 |
| **查询效率** | 稍快（不需要计算长度） | 稍慢 |
| **适用场景** | 长度固定的数据 | 长度变化大的数据 |

### 存储示例

```sql
CREATE TABLE test (
    char_col CHAR(10),
    varchar_col VARCHAR(10)
);

INSERT INTO test VALUES ('abc', 'abc');
```

**存储对比：**
```
CHAR(10):
- 存储内容: 'abc       ' (补7个空格，总共10字节)
- 读取时自动去掉尾部空格

VARCHAR(10):
- 存储内容: [3] + 'abc' (1字节长度 + 3字节数据 = 4字节)
- 长度前缀: 1字节(长度≤255) 或 2字节(长度>255)
```

### 长度前缀规则

**VARCHAR 长度前缀：**
- VARCHAR(255) 及以下：1 字节长度前缀
- VARCHAR(256) 及以上：2 字节长度前缀

```sql
-- 1 字节长度前缀
CREATE TABLE t1 (name VARCHAR(255));  -- 最多 1 + 255 = 256 字节

-- 2 字节长度前缀
CREATE TABLE t2 (name VARCHAR(1000)); -- 最多 2 + 1000 = 1002 字节
```

### 使用场景

**CHAR 适合：**
- 长度固定的数据：手机号、身份证号、MD5 值
- 长度变化小的数据：性别（'男'/'女'）、状态码

```sql
CREATE TABLE user (
    id INT PRIMARY KEY,
    phone CHAR(11),        -- 手机号固定11位
    id_card CHAR(18),      -- 身份证固定18位
    gender CHAR(1)         -- 性别固定1位
);
```

**VARCHAR 适合：**
- 长度变化大的数据：姓名、地址、邮箱
- 长度不确定的数据：评论、描述

```sql
CREATE TABLE user (
    id INT PRIMARY KEY,
    name VARCHAR(50),      -- 姓名长度不固定
    email VARCHAR(100),    -- 邮箱长度不固定
    address VARCHAR(200)   -- 地址长度不固定
);
```

### 性能对比

**CHAR 的优势：**
- 不需要记录长度，读取更快
- 适合频繁更新的场景（不会产生碎片）

**VARCHAR 的优势：**
- 节省存储空间
- 适合长度变化大的场景

⚠️ **注意：** CHAR 在 UTF-8 编码下，CHAR(10) 最多占用 30 字节（每个字符最多3字节）

---

## 4. 一个数据页可以被太致划分为 7 个部分，分别是哪些？

### 什么是数据页
InnoDB 以页为单位管理存储空间，一个页默认 16KB。就像书本以"页"为单位，MySQL 以"数据页"为单位读写数据。

```sql
-- 查看页大小
SHOW VARIABLES LIKE 'innodb_page_size';  -- 默认 16384 字节 = 16KB
```

### 数据页的 7 个部分

```
┌─────────────────────────────────────┐
│  File Header (文件头，38字节)         │  页的通用信息
├─────────────────────────────────────┤
│  Page Header (页头，56字节)          │  页的状态信息
├─────────────────────────────────────┤
│  Infimum + Supremum (26字节)        │  最小/最大虚拟记录
├─────────────────────────────────────┤
│  User Records (用户记录)             │  实际存储的行记录
├─────────────────────────────────────┤
│  Free Space (空闲空间)               │  未使用的空间
├─────────────────────────────────────┤
│  Page Directory (页目录)             │  记录的相对位置
├─────────────────────────────────────┤
│  File Trailer (文件尾，8字节)        │  校验页是否完整
└─────────────────────────────────────┘
```

### ① File Header（文件头，38字节）

**作用：** 记录页的通用信息

**关键字段：**
- `FIL_PAGE_OFFSET`：页号（4字节）
- `FIL_PAGE_TYPE`：页类型（2字节）
  - 0x45BF：数据页
  - 0x45BE：索引页
- `FIL_PAGE_PREV`：上一页的页号（4字节）
- `FIL_PAGE_NEXT`：下一页的页号（4字节）
- `FIL_PAGE_LSN`：页最后被修改的日志序列号（8字节）

**作用：** 通过 PREV 和 NEXT 形成双向链表，方便范围查询。

### ② Page Header（页头，56字节）

**作用：** 记录页的状态信息

**关键字段：**
- `PAGE_N_RECS`：页中记录的数量（2字节）
- `PAGE_FREE`：空闲空间的起始位置（2字节）
- `PAGE_GARBAGE`：已删除记录占用的字节数（2字节）
- `PAGE_LAST_INSERT`：最后插入记录的位置（2字节）
- `PAGE_DIRECTION`：插入方向（1字节）
  - 0x01：向右（递增）
  - 0x02：向左（递减）
- `PAGE_N_DIRECTION`：连续同方向插入的记录数（2字节）

### ③ Infimum + Supremum（26字节）

**作用：** 两条虚拟记录，标记页中记录的边界

- **Infimum**：最小记录（比任何记录都小）
- **Supremum**：最大记录（比任何记录都大）

**为什么需要？**
- 方便遍历：从 Infimum 开始，通过 next_record 指针遍历所有记录
- 统一处理：不需要特殊处理第一条和最后一条记录

```
Infimum → 记录1 → 记录2 → 记录3 → Supremum
```

### ④ User Records（用户记录）

**作用：** 实际存储的行记录

- 记录按主键顺序排列（单向链表）
- 通过记录头的 `next_record` 指针连接

```sql
-- 插入数据
INSERT INTO user VALUES (1, '张三'), (3, '李四'), (2, '王五');

-- 存储顺序（按主键排序）
Infimum → (1,'张三') → (2,'王五') → (3,'李四') → Supremum
```

### ⑤ Free Space（空闲空间）

**作用：** 未使用的空间，用于插入新记录

- 新记录从 Free Space 开始分配
- 删除记录后，空间加入到空闲链表

### ⑥ Page Directory（页目录）

**作用：** 记录的"目录"，加速查找

**工作原理：**
- 将记录分组（每组 4-8 条记录）
- 记录每组最后一条记录的偏移量
- 二分查找定位到组，再顺序查找

```
页目录: [100, 200, 300, 400]
       ↓    ↓    ↓    ↓
记录:  1-4  5-8  9-12 13-16
```

**查找过程：**
```sql
SELECT * FROM user WHERE id = 10;

1. 二分查找页目录: 10 在 [300, 400) 区间
2. 顺序遍历第3组: 找到 id=10 的记录
```

### ⑦ File Trailer（文件尾，8字节）

**作用：** 校验页是否完整

- 前 4 字节：页的校验和
- 后 4 字节：页的 LSN 后 4 字节

**为什么需要？**
- 检测页是否损坏（校验和不匹配）
- 检测页是否写入完整（LSN 不匹配）

---

## 5. Buffer Pool 是什么？主要是针对查询的优化还是写入的优化？

### 什么是 Buffer Pool

**Buffer Pool 是 InnoDB 的内存缓存池，用于缓存数据页和索引页。**

想象你在图书馆看书，每次都从书架拿书太慢，所以你在桌上放几本常看的书。Buffer Pool 就是 MySQL 的"桌子"，把常用的数据放在内存里，避免频繁读磁盘。

```sql
-- 查看 Buffer Pool 大小
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- 设置 Buffer Pool 大小（建议设置为物理内存的 50%-80%）
SET GLOBAL innodb_buffer_pool_size = 1073741824;  -- 1GB
```

### Buffer Pool 的作用

**① 读优化（主要）**
- 缓存数据页，避免频繁读磁盘
- 缓存索引页，加速索引查询

**② 写优化（次要）**
- 缓存修改的数据页，批量刷盘
- 减少随机 I/O，提升写入性能

### Buffer Pool 的结构

```
┌─────────────────────────────────────┐
│         Buffer Pool (内存)           │
├─────────────────────────────────────┤
│  数据页缓存 (Data Pages)             │  缓存表数据
├─────────────────────────────────────┤
│  索引页缓存 (Index Pages)            │  缓存索引数据
├─────────────────────────────────────┤
│  插入缓冲 (Insert Buffer)            │  缓存非聚簇索引的插入
├─────────────────────────────────────┤
│  自适应哈希索引 (Adaptive Hash)      │  加速等值查询
├─────────────────────────────────────┤
│  锁信息 (Lock Info)                  │  缓存锁信息
├─────────────────────────────────────┤
│  数据字典 (Data Dictionary)          │  缓存表结构信息
└─────────────────────────────────────┘
```

### 读取流程

```sql
SELECT * FROM user WHERE id = 1;
```

**① 查询 Buffer Pool**
- 如果数据页在 Buffer Pool 中 → 直接返回（命中）
- 如果不在 → 从磁盘读取，放入 Buffer Pool（未命中）

**② LRU 淘汰策略**
- Buffer Pool 满了，淘汰最少使用的页
- 使用改进的 LRU 算法（分为 young 区和 old 区）

```
┌──────────────────────────────────┐
│  Young 区 (热数据，5/8)           │  频繁访问的页
├──────────────────────────────────┤
│  Old 区 (冷数据，3/8)             │  新读入的页
└──────────────────────────────────┘
```

**为什么分区？**
- 防止全表扫描污染 Buffer Pool
- 新读入的页先放 Old 区，访问多次才进入 Young 区

### 写入流程

```sql
UPDATE user SET name = '李四' WHERE id = 1;
```

**① 修改 Buffer Pool**
- 在内存中修改数据页
- 标记为脏页（Dirty Page）

**② 写 redo log**
- 记录修改操作到 redo log
- 保证崩溃恢复

**③ 后台刷盘**
- 脏页不是立即刷盘，而是后台异步刷盘
- 减少随机 I/O，提升性能

**刷盘时机：**
- Buffer Pool 空间不足
- redo log 写满
- 定期刷盘（每秒一次）
- MySQL 正常关闭

### Buffer Pool 的优化建议

**① 设置合适的大小**
```sql
-- 建议设置为物理内存的 50%-80%
SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB
```

**② 监控命中率**
```sql
-- 查看 Buffer Pool 状态
SHOW ENGINE INNODB STATUS;

-- 计算命中率
命中率 = (Innodb_buffer_pool_read_requests - Innodb_buffer_pool_reads) 
       / Innodb_buffer_pool_read_requests

-- 命中率应该 > 99%
```

**③ 预热 Buffer Pool**
```sql
-- MySQL 重启后，Buffer Pool 是空的
-- 可以手动预热常用数据
SELECT * FROM user LIMIT 10000;
```

---

## 6. Change Buffer 是什么？它的作用是什么？使用场景？

### 什么是 Change Buffer

**Change Buffer 是 InnoDB 的写缓冲区，用于缓存对非聚簇索引的修改操作。**

想象你在整理书架，每次插入一本书都要找到正确位置很麻烦。Change Buffer 就是先把书堆在一边，等有空了再一起整理。

### 为什么需要 Change Buffer

**问题：** 更新非聚簇索引时，需要先读取索引页到内存，再修改，最后刷盘。如果索引页不在 Buffer Pool 中，就需要随机读磁盘，性能很差。

**解决：** Change Buffer 先把修改操作缓存起来，等索引页被读取到 Buffer Pool 时，再合并（Merge）修改。

### Change Buffer 的工作流程

```sql
-- 插入数据
INSERT INTO user (id, name, email) VALUES (100, '张三', 'zhangsan@qq.com');

-- 假设 email 有非聚簇索引
CREATE INDEX idx_email ON user(email);
```

**① 插入聚簇索引（主键索引）**
- 直接插入到 Buffer Pool 中的数据页

**② 插入非聚簇索引（email 索引）**
- 如果索引页在 Buffer Pool 中 → 直接插入
- 如果索引页不在 Buffer Pool 中 → 写入 Change Buffer

**③ 后台合并（Merge）**
- 当索引页被读取到 Buffer Pool 时，合并 Change Buffer 中的修改
- 或者后台线程定期合并

### Change Buffer 的使用场景

**✅ 适合场景：**
- 写多读少的表
- 非聚簇索引较多的表
- 索引页不在 Buffer Pool 中的情况

**❌ 不适合场景：**
- 唯一索引（需要检查唯一性，必须读取索引页）
- 读多写少的表（索引页经常在 Buffer Pool 中）

```sql
-- Change Buffer 只对普通索引有效
CREATE INDEX idx_email ON user(email);  -- ✅ 可以使用 Change Buffer

-- 唯一索引不能使用 Change Buffer
CREATE UNIQUE INDEX idx_email ON user(email);  -- ❌ 不能使用 Change Buffer
```

### Change Buffer 的配置

```sql
-- 查看 Change Buffer 大小（占 Buffer Pool 的比例）
SHOW VARIABLES LIKE 'innodb_change_buffer_max_size';  -- 默认 25%

-- 设置 Change Buffer 大小
SET GLOBAL innodb_change_buffer_max_size = 50;  -- 占 Buffer Pool 的 50%

-- 查看 Change Buffer 使用情况
SHOW ENGINE INNODB STATUS;
```

### Change Buffer vs Buffer Pool

| 特性 | Buffer Pool | Change Buffer |
|------|-------------|---------------|
| **作用** | 缓存数据页和索引页 | 缓存非聚簇索引的修改 |
| **优化对象** | 读和写 | 写 |
| **适用场景** | 所有查询和更新 | 写多读少的非聚簇索引 |
| **大小** | 独立配置 | 占 Buffer Pool 的一部分 |

---

## 7. MySQL 中 InnoDB 和 MyISAM 存储引擎的区别是什么？

### InnoDB 和 MyISAM 是 MySQL 中最常用的两种存储引擎

**① 事务支持**

InnoDB 支持事务处理，包括提交和回滚，而 MyISAM 不支持事务。

```sql
-- InnoDB 支持事务
CREATE TABLE orders (
    id INT PRIMARY KEY,
    amount DECIMAL(10,2)
) ENGINE=InnoDB;

BEGIN;
INSERT INTO orders VALUES (1, 100);
ROLLBACK;  -- 可以回滚

-- MyISAM 不支持事务
CREATE TABLE logs (
    id INT PRIMARY KEY,
    message TEXT
) ENGINE=MyISAM;

-- 无法使用事务
```

**② 表锁定和行锁定**

InnoDB 支持行级锁定和外键约束，适合高并发操作，而 MyISAM 仅支持表级锁定。

```sql
-- InnoDB 行锁
UPDATE orders SET amount = 200 WHERE id = 1;  -- 只锁定 id=1 这一行

-- MyISAM 表锁
UPDATE logs SET message = 'new' WHERE id = 1;  -- 锁定整个表
```

**③ 数据完整性**

InnoDB 提供了对数据完整性的支持，包括外键约束，而 MyISAM 则不支持。

```sql
-- InnoDB 支持外键
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB;

-- MyISAM 不支持外键
CREATE TABLE logs (
    id INT PRIMARY KEY,
    user_id INT
    -- 无法创建外键约束
) ENGINE=MyISAM;
```

**④ 存储限制**

MyISAM 支持的文件大小通常比 InnoDB 要大。

**⑤ 内存使用**

InnoDB 通常需要更多的内存和存储空间来维护其事务日志和缓冲池。

**⑥ 恢复能力**

InnoDB 能够更好地处理崩溃后的恢复。

### InnoDB vs MyISAM 对比表

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 | 不支持 |
| **锁机制** | 行锁 | 表锁 |
| **外键约束** | 支持 | 不支持 |
| **崩溃恢复** | 支持（redo log） | 不支持 |
| **MVCC** | 支持 | 不支持 |
| **全文索引** | 支持（5.6+） | 支持 |
| **存储空间** | 较大 | 较小 |
| **查询性能** | 较慢 | 较快（简单查询） |
| **写入性能** | 较快（行锁） | 较慢（表锁） |
| **适用场景** | 事务、高并发写入 | 只读、日志记录 |

---

## 面试回答模板

**Q: MySQL 行记录的存储格式有哪些？**

A: InnoDB 有四种行格式：
1. **Compact**：紧凑存储，MySQL 5.1+ 默认格式
2. **Redundant**：冗余存储，兼容老版本
3. **Dynamic**：动态存储，MySQL 5.7+ 默认，大字段完全溢出
4. **Compressed**：压缩存储，节省磁盘空间

推荐使用 Dynamic 格式，对大字段处理更好。

**Q: char 和 varchar 的区别？**

A: 
- **CHAR** 是固定长度，不足部分用空格填充，适合长度固定的数据（如手机号）
- **VARCHAR** 是可变长度，需要 1-2 字节记录长度，适合长度变化大的数据（如姓名）
- CHAR 查询稍快但浪费空间，VARCHAR 节省空间但需要计算长度

**Q: Buffer Pool 是什么？**

A: Buffer Pool 是 InnoDB 的内存缓存池，用于缓存数据页和索引页。主要优化读操作，避免频繁读磁盘。同时也优化写操作，通过批量刷盘减少随机 I/O。建议设置为物理内存的 50%-80%。

**Q: Change Buffer 的作用是什么？**

A: Change Buffer 用于缓存对非聚簇索引的修改操作。当索引页不在 Buffer Pool 中时，先把修改缓存起来，等索引页被读取时再合并。适合写多读少的场景，但不支持唯一索引。

---

## 面试简答版

**行格式：** Compact、Redundant、Dynamic、Compressed，推荐 Dynamic

**NULL 存储：** 只占 NULL 标志位，不占实际数据空间

**CHAR vs VARCHAR：** CHAR 固定长度，VARCHAR 可变长度

**数据页结构：** File Header、Page Header、Infimum+Supremum、User Records、Free Space、Page Directory、File Trailer

**Buffer Pool：** 内存缓存池，缓存数据页和索引页，主要优化读操作

**Change Buffer：** 缓存非聚簇索引的修改，优化写操作，适合写多读少场景
