# MySQL 日志

## 1. 慢查询日志的用途是什么？

### 什么是慢查询日志

**慢查询日志（Slow Query Log）：** 记录执行时间超过阈值的 SQL 语句。

**作用：** 帮助定位性能瓶颈，优化慢 SQL。

### 慢查询日志的配置

```sql
-- 查看慢查询日志是否开启
SHOW VARIABLES LIKE 'slow_query_log';

-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;

-- 查看慢查询阈值（默认 10 秒）
SHOW VARIABLES LIKE 'long_query_time';

-- 设置慢查询阈值（单位：秒）
SET GLOBAL long_query_time = 2;  -- 超过 2 秒的查询会被记录

-- 查看慢查询日志文件路径
SHOW VARIABLES LIKE 'slow_query_log_file';
```

### 慢查询日志的内容

```
# Time: 2026-05-09T16:00:00.123456Z
# User@Host: root[root] @ localhost []
# Query_time: 3.123456  Lock_time: 0.000123  Rows_sent: 100  Rows_examined: 10000
SET timestamp=1715270400;
SELECT * FROM employee WHERE name LIKE '%张%';
```

**关键字段：**
- `Query_time`：查询执行时间
- `Lock_time`：锁等待时间
- `Rows_sent`：返回的行数
- `Rows_examined`：扫描的行数

### 分析慢查询日志

**① 使用 mysqldumpslow 工具**

```bash
# 查看最慢的 10 条 SQL
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 查看访问次数最多的 10 条 SQL
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log

# 查看返回记录最多的 10 条 SQL
mysqldumpslow -s r -t 10 /var/log/mysql/slow.log
```

**② 使用 pt-query-digest 工具**

```bash
# 分析慢查询日志
pt-query-digest /var/log/mysql/slow.log
```

### 慢查询日志的使用场景

**① 定位性能瓶颈**

找出执行时间最长的 SQL，优先优化。

**② 发现全表扫描**

`Rows_examined` 很大但 `Rows_sent` 很小，说明扫描了很多无用数据。

**③ 监控生产环境**

定期分析慢查询日志，及时发现性能问题。

⚠️ **注意：** 慢查询日志会影响性能，生产环境建议只在需要时开启。

---

## 2. 事务的原子性和 undolog 之间的关系？

### undo log 是什么

**undo log（回滚日志）：** 记录事务修改前的数据（旧值），用于事务回滚和 MVCC。

### undo log 如何实现原子性

**原子性：** 事务要么全部成功，要么全部失败。

**实现方式：** 通过 undo log 记录旧值，回滚时恢复数据。

**示例：**
```sql
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;  -- 旧值 600
UPDATE account SET balance = 600 WHERE id = 2;  -- 旧值 500
ROLLBACK;  -- 用 undo log 恢复：id=1 恢复成 600，id=2 恢复成 500
```

### undo log 的工作流程

**① 事务开始**
```sql
BEGIN;
```

**② 修改数据前，先写 undo log**
```
UPDATE account SET balance = 500 WHERE id = 1;

写 undo log:
  - 表名: account
  - 主键: id=1
  - 旧值: balance=600
  - 操作: UPDATE
```

**③ 修改数据**
```
在 Buffer Pool 中修改:
  id=1, balance=500
```

**④ 事务回滚**
```sql
ROLLBACK;

读取 undo log:
  - 表名: account
  - 主键: id=1
  - 旧值: balance=600

恢复数据:
  UPDATE account SET balance = 600 WHERE id = 1;
```

### undo log 的存储

**① undo log 存储在哪里？**

- MySQL 5.6 之前：存储在系统表空间（ibdata1）
- MySQL 5.6 之后：存储在独立的 undo 表空间

```sql
-- 查看 undo 表空间
SHOW VARIABLES LIKE 'innodb_undo_tablespaces';
```

**② undo log 什么时候删除？**

- 事务提交后，undo log 不会立即删除
- 需要等待所有使用该版本的事务结束（MVCC）
- 由后台线程（purge 线程）定期清理

### undo log 的两个作用

**① 实现原子性（回滚）**

```sql
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
ROLLBACK;  -- 用 undo log 恢复
```

**② 实现 MVCC（多版本并发控制）**

```sql
-- 事务 A
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 读到 600

-- 事务 B
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
COMMIT;

-- 事务 A
SELECT balance FROM account WHERE id = 1;  -- 还是读到 600（通过 undo log）
```

---

## 3. binlog 的原理有了解吗？

### 什么是 binlog

**binlog（归档日志）：** 记录所有修改数据的 SQL 语句或行变更，用于主从复制和数据恢复。

**特点：**
- Server 层实现，所有存储引擎都可以使用
- 逻辑日志，记录 SQL 语句或行变更
- 追加写入，不会覆盖

### binlog 的三种格式

**① STATEMENT 格式**

**记录 SQL 语句本身。**

```sql
-- binlog 记录
UPDATE account SET balance = balance - 100 WHERE id = 1;
```

**优点：**
- 日志量小
- 可读性好

**缺点：**
- 某些函数会导致主从不一致（如 NOW()、UUID()）

```sql
-- 主库执行
UPDATE account SET update_time = NOW() WHERE id = 1;
-- 主库：update_time = 2026-05-09 16:00:00

-- 从库执行（可能晚几秒）
UPDATE account SET update_time = NOW() WHERE id = 1;
-- 从库：update_time = 2026-05-09 16:00:05

-- 主从不一致！
```

**② ROW 格式（推荐）**

**记录每一行的变更。**

```sql
-- binlog 记录
UPDATE account SET balance = 500 WHERE id = 1;
-- 记录：id=1, balance 从 600 改为 500
```

**优点：**
- 不会出现主从不一致
- 可以精确恢复数据

**缺点：**
- 日志量大（每行都记录）

**③ MIXED 格式**

**自动选择 STATEMENT 或 ROW。**

- 一般情况用 STATEMENT
- 可能导致不一致时用 ROW

### binlog 的写入流程

```
事务执行
  ↓
写 redo log (prepare)
  ↓
写 binlog
  ↓
写 redo log (commit)
  ↓
事务提交
```

**两阶段提交：** 保证 redo log 和 binlog 的一致性。

### binlog 的配置

```sql
-- 查看 binlog 是否开启
SHOW VARIABLES LIKE 'log_bin';

-- 查看 binlog 格式
SHOW VARIABLES LIKE 'binlog_format';

-- 设置 binlog 格式
SET GLOBAL binlog_format = 'ROW';

-- 查看 binlog 文件列表
SHOW BINARY LOGS;

-- 查看当前使用的 binlog 文件
SHOW MASTER STATUS;
```

### binlog 的使用场景

**① 主从复制**

```
主库写入 binlog
  ↓
从库读取 binlog
  ↓
从库执行 binlog 中的操作
  ↓
主从数据一致
```

**② 数据恢复**

```bash
# 恢复 binlog
mysqlbinlog --start-datetime="2026-05-09 15:00:00" \
            --stop-datetime="2026-05-09 16:00:00" \
            mysql-bin.000001 | mysql -u root -p
```

**③ 数据审计**

查看谁在什么时候修改了什么数据。

---

## 4. 事务的持久性如何实现？

### 持久性是什么

**持久性（Durability）：** 事务一旦提交，对数据库的改变是永久的，即使系统崩溃也不会丢失。

### redo log 实现持久性

**redo log（重做日志）：** 记录事务修改后的数据（新值），用于崩溃恢复。

**为什么需要 redo log？**

- 数据修改先在内存（Buffer Pool）中进行
- 如果每次修改都立即刷盘，性能太差（随机 I/O）
- redo log 先记录修改操作，后台异步刷盘（顺序 I/O）

### redo log 的工作流程

```
事务开始
  ↓
修改 Buffer Pool 中的数据
  ↓
写 redo log（顺序写，快）
  ↓
事务提交
  ↓
后台异步刷盘（随机写，慢）
  ↓
MySQL 崩溃
  ↓
重启后读取 redo log
  ↓
恢复数据
```

**示例：**
```sql
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
COMMIT;

-- 写 redo log: 将 id=1 的 balance 改为 500
-- 数据还在内存中，未刷盘

-- MySQL 崩溃

-- 重启后读取 redo log
-- 恢复数据: id=1, balance=500
```

### redo log 的特点

**① 循环写入**

redo log 是固定大小的，写满后会覆盖旧的日志。

```
┌─────────────────────────────────┐
│  redo log 文件（固定大小）        │
│  ib_logfile0, ib_logfile1       │
│                                 │
│  write pos ──→ 当前写入位置      │
│  checkpoint ──→ 已刷盘位置       │
└─────────────────────────────────┘
```

**② 顺序写入**

redo log 是顺序写入，性能比随机写入快很多。

**③ 崩溃恢复**

MySQL 重启后，读取 redo log，恢复未刷盘的数据。

### redo log 的配置

```sql
-- 查看 redo log 文件大小
SHOW VARIABLES LIKE 'innodb_log_file_size';

-- 查看 redo log 文件数量
SHOW VARIABLES LIKE 'innodb_log_files_in_group';

-- 查看 redo log 刷盘策略
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
```

**innodb_flush_log_at_trx_commit 参数：**

| 值 | 含义 | 性能 | 安全性 |
|----|------|------|--------|
| **0** | 每秒刷盘一次 | 最高 | 最低（可能丢失 1 秒数据） |
| **1** | 每次事务提交都刷盘 | 最低 | 最高（不会丢失数据） |
| **2** | 每次事务提交写到 OS 缓存，每秒刷盘 | 中等 | 中等（MySQL 崩溃不丢失，OS 崩溃可能丢失） |

**推荐：** 生产环境设置为 1，保证数据不丢失。

---

## 5. redolog 的使用场景都有哪些？

### redo log 的使用场景

**① 崩溃恢复**

MySQL 崩溃重启后，用 redo log 恢复未刷盘的数据。

```sql
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
COMMIT;

-- 数据在内存中，未刷盘
-- MySQL 崩溃
-- 重启后读取 redo log，恢复数据
```

**② 提高写入性能**

先写 redo log（顺序 I/O），后台异步刷盘（随机 I/O）。

```
写 redo log: 顺序写，快（几毫秒）
刷盘: 随机写，慢（几十毫秒）
```

**③ 保证持久性**

事务提交后，redo log 已刷盘，数据不会丢失。

---

## 6. binlog、redolog、undolog 有什么区别？

### 三大日志对比

| 特性 | redo log | binlog | undo log |
|------|----------|--------|----------|
| **层级** | InnoDB 引擎层 | Server 层 | InnoDB 引擎层 |
| **作用** | 崩溃恢复，保证持久性 | 主从复制，数据恢复 | 事务回滚，MVCC |
| **记录内容** | 物理日志（页的修改） | 逻辑日志（SQL 或行变更） | 逻辑日志（旧值） |
| **写入方式** | 循环写入（固定大小） | 追加写入（不覆盖） | 随事务写入 |
| **刷盘时机** | 事务提交时 | 事务提交时 | 事务开始时 |
| **使用场景** | 崩溃恢复 | 主从复制、数据恢复 | 事务回滚、MVCC |

### redo log vs binlog

**① 层级不同**
- redo log：InnoDB 引擎层，只有 InnoDB 有
- binlog：Server 层，所有存储引擎都有

**② 作用不同**
- redo log：崩溃恢复，保证持久性
- binlog：主从复制，数据恢复

**③ 记录内容不同**
- redo log：物理日志，记录"在某个数据页上做了什么修改"
- binlog：逻辑日志，记录 SQL 语句或行变更

**④ 写入方式不同**
- redo log：循环写入，固定大小，写满后覆盖
- binlog：追加写入，不覆盖，写满后创建新文件

### redo log vs undo log

**① 作用不同**
- redo log：崩溃恢复，保证持久性（记录新值）
- undo log：事务回滚，MVCC（记录旧值）

**② 记录内容不同**
- redo log：记录修改后的数据（新值）
- undo log：记录修改前的数据（旧值）

**③ 使用场景不同**
- redo log：MySQL 崩溃后恢复数据
- undo log：事务回滚，读取历史版本

### 两阶段提交

**为什么需要两阶段提交？**

保证 redo log 和 binlog 的一致性。

**流程：**
```
事务执行
  ↓
写 redo log (prepare)  -- 第一阶段
  ↓
写 binlog
  ↓
写 redo log (commit)   -- 第二阶段
  ↓
事务提交
```

**如果没有两阶段提交会怎样？**

**场景 1：先写 redo log，再写 binlog**
```
写 redo log ✅
MySQL 崩溃 💥
写 binlog ❌

结果：
- 主库数据已修改（通过 redo log 恢复）
- 从库数据未修改（binlog 未写入）
- 主从不一致！
```

**场景 2：先写 binlog，再写 redo log**
```
写 binlog ✅
MySQL 崩溃 💥
写 redo log ❌

结果：
- 主库数据未修改（redo log 未写入）
- 从库数据已修改（binlog 已写入）
- 主从不一致！
```

**两阶段提交解决方案：**
```
写 redo log (prepare) ✅
写 binlog ✅
MySQL 崩溃 💥
写 redo log (commit) ❌

重启后：
- 检查 redo log 状态
- 如果是 prepare 状态，检查 binlog 是否完整
- 如果 binlog 完整，提交事务
- 如果 binlog 不完整，回滚事务
```

---

## 7. 说一下两阶段提交？

### 什么是两阶段提交

**两阶段提交（Two-Phase Commit）：** 保证 redo log 和 binlog 的一致性。

### 两阶段提交的流程

**阶段 1：Prepare**
```
事务执行
  ↓
写 redo log (prepare 状态)
  ↓
写 binlog
```

**阶段 2：Commit**
```
写 redo log (commit 状态)
  ↓
事务提交
```

### 崩溃恢复的处理

**场景 1：redo log 处于 prepare 状态，binlog 完整**
```
写 redo log (prepare) ✅
写 binlog ✅
MySQL 崩溃 💥

重启后：
- 检查 redo log 状态：prepare
- 检查 binlog：完整
- 提交事务（将 redo log 改为 commit）
```

**场景 2：redo log 处于 prepare 状态，binlog 不完整**
```
写 redo log (prepare) ✅
写 binlog 一半 💥
MySQL 崩溃

重启后：
- 检查 redo log 状态：prepare
- 检查 binlog：不完整
- 回滚事务（用 undo log 恢复）
```

**场景 3：redo log 处于 commit 状态**
```
写 redo log (prepare) ✅
写 binlog ✅
写 redo log (commit) ✅
MySQL 崩溃 💥

重启后：
- 检查 redo log 状态：commit
- 事务已提交，不需要处理
```

### 两阶段提交的意义

**保证 redo log 和 binlog 的一致性：**
- redo log 用于崩溃恢复
- binlog 用于主从复制
- 两者必须一致，否则主从数据不一致

---

## 面试回答模板

**Q: 慢查询日志的用途是什么？**

A: 慢查询日志记录执行时间超过阈值的 SQL 语句，用于定位性能瓶颈。可以通过 `long_query_time` 设置阈值，使用 `mysqldumpslow` 或 `pt-query-digest` 工具分析日志。

**Q: undo log 的作用是什么？**

A: undo log 有两个作用：
1. **实现原子性**：记录旧值，事务回滚时恢复数据
2. **实现 MVCC**：读取历史版本，实现可重复读

**Q: binlog 的作用是什么？**

A: binlog 有两个作用：
1. **主从复制**：从库读取主库的 binlog 同步数据
2. **数据恢复**：误删数据后，用 binlog 恢复

binlog 有三种格式：STATEMENT、ROW、MIXED，推荐使用 ROW 格式。

**Q: redo log 的作用是什么？**

A: redo log 用于崩溃恢复，保证持久性。数据修改先在内存中进行，写 redo log 后事务提交，后台异步刷盘。MySQL 崩溃重启后，用 redo log 恢复未刷盘的数据。

**Q: redo log 和 binlog 的区别？**

A: 
- **层级**：redo log 是 InnoDB 引擎层，binlog 是 Server 层
- **作用**：redo log 用于崩溃恢复，binlog 用于主从复制
- **记录内容**：redo log 是物理日志，binlog 是逻辑日志
- **写入方式**：redo log 循环写入，binlog 追加写入

**Q: 什么是两阶段提交？**

A: 两阶段提交保证 redo log 和 binlog 的一致性：
1. **Prepare 阶段**：写 redo log（prepare 状态）和 binlog
2. **Commit 阶段**：写 redo log（commit 状态）

如果 MySQL 崩溃，重启后检查 redo log 状态和 binlog 完整性，决定提交或回滚事务。

---

## 面试简答版

**慢查询日志：** 记录慢 SQL，用于性能优化

**undo log：** 记录旧值，实现原子性和 MVCC

**binlog：** 记录 SQL 或行变更，用于主从复制和数据恢复

**redo log：** 记录新值，用于崩溃恢复，保证持久性

**redo log vs binlog：** 层级不同、作用不同、记录内容不同、写入方式不同

**两阶段提交：** 先写 redo log（prepare）和 binlog，再写 redo log（commit），保证一致性
