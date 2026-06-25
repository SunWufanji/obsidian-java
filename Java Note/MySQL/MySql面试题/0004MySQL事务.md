# MySQL 事务（重点）

## 1. MySQL 里面的 ACID 是如何实现的？

### 什么是 ACID

**ACID 是事务的四大特性，保证数据的可靠性。**

| 特性 | 英文 | 含义 | 实现方式 |
|------|------|------|---------|
| **原子性** | Atomicity | 要么全做，要么全不做 | undo log |
| **一致性** | Consistency | 数据从一个一致状态到另一个一致状态 | 其他三个特性保证 |
| **隔离性** | Isolation | 事务之间互不干扰 | MVCC + 锁 |
| **持久性** | Durability | 提交后永久保存 | redo log |

### 原子性（Atomicity）- undo log

**原子性：** 事务是不可分割的最小单位，要么全部成功，要么全部失败。

**实现方式：** undo log（回滚日志）

```sql
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 张三转账
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 李四收款
ROLLBACK;  -- 回滚，两条 UPDATE 都撤销
```

**undo log 的作用：**
1. 记录修改前的数据（旧值）
2. 事务回滚时，用 undo log 恢复数据
3. 实现 MVCC（多版本并发控制）

**示例：**
```
事务开始
  ↓
UPDATE account SET balance = 500 WHERE id = 1;
  ↓
写 undo log: id=1, balance=600 (旧值)
  ↓
修改数据: id=1, balance=500 (新值)
  ↓
ROLLBACK
  ↓
读取 undo log: id=1, balance=600
  ↓
恢复数据: id=1, balance=600
```

### 一致性（Consistency）

**一致性：** 数据库从一个一致状态转换到另一个一致状态。

**什么是一致状态？**
- 满足所有约束条件（主键、外键、唯一索引）
- 满足业务规则（如转账前后总金额不变）

**示例：**
```sql
-- 转账前：张三 600 元，李四 400 元，总计 1000 元
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 张三 500 元
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 李四 500 元
COMMIT;
-- 转账后：张三 500 元，李四 500 元，总计 1000 元（一致）
```

**一致性由其他三个特性保证：**
- 原子性：保证要么全做，要么全不做
- 隔离性：保证事务之间不互相干扰
- 持久性：保证提交后数据不丢失

### 隔离性（Isolation）- MVCC + 锁

**隔离性：** 多个事务并发执行时，互不干扰。

**实现方式：** MVCC（多版本并发控制）+ 锁

**① MVCC（读不加锁）**
- 每行数据有多个版本
- 读取时根据事务的快照读取对应版本
- 不需要加锁，提高并发性能

**② 锁（写加锁）**
- 行锁：锁定某一行
- 表锁：锁定整个表
- 间隙锁：锁定索引之间的间隙

```sql
-- 事务 A
BEGIN;
SELECT * FROM account WHERE id = 1;  -- 快照读，不加锁
UPDATE account SET balance = 500 WHERE id = 1;  -- 加行锁
COMMIT;

-- 事务 B（并发执行）
BEGIN;
SELECT * FROM account WHERE id = 1;  -- 快照读，读到旧版本
UPDATE account SET balance = 600 WHERE id = 1;  -- 等待事务 A 释放锁
COMMIT;
```

### 持久性（Durability）- redo log

**持久性：** 事务一旦提交，对数据库的改变是永久的。

**实现方式：** redo log（重做日志）

**为什么需要 redo log？**
- 数据修改先在内存（Buffer Pool）中进行
- 如果每次修改都立即刷盘，性能太差
- redo log 先记录修改操作，后台异步刷盘

**redo log 的作用：**
1. 记录修改操作（新值）
2. MySQL 崩溃重启后，用 redo log 恢复数据
3. 保证持久性

**示例：**
```
事务开始
  ↓
UPDATE account SET balance = 500 WHERE id = 1;
  ↓
修改 Buffer Pool: id=1, balance=500
  ↓
写 redo log: 将 id=1 的 balance 改为 500
  ↓
COMMIT
  ↓
redo log 刷盘（顺序写，快）
  ↓
后台异步刷盘（随机写，慢）
  ↓
MySQL 崩溃
  ↓
重启后读取 redo log
  ↓
恢复数据: id=1, balance=500
```

### ACID 实现总结

| 特性 | 实现方式 | 作用 |
|------|---------|------|
| 原子性 | undo log | 记录旧值，回滚时恢复 |
| 一致性 | 其他三个特性 | 保证数据符合约束和业务规则 |
| 隔离性 | MVCC + 锁 | 读不加锁（MVCC），写加锁 |
| 持久性 | redo log | 记录新值，崩溃后恢复 |

---

## 2. 说一下 MySQL 事务隔离级别都有哪些？

### 什么是事务隔离级别

**事务隔离级别：** 控制事务之间的可见性，解决并发问题。

### 并发事务的三大问题

**① 脏读（Dirty Read）**
- 读到其他事务未提交的数据

```sql
-- 事务 A
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
-- 未提交

-- 事务 B
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 读到 500（脏读）
COMMIT;

-- 事务 A
ROLLBACK;  -- 回滚，balance 恢复成 600
```

**问题：** 事务 B 读到的 500 是无效数据。

**② 不可重复读（Non-Repeatable Read）**
- 同一事务中，多次读取同一数据，结果不一致

```sql
-- 事务 A
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 读到 600

-- 事务 B
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
COMMIT;

-- 事务 A
SELECT balance FROM account WHERE id = 1;  -- 读到 500（不可重复读）
COMMIT;
```

**问题：** 事务 A 两次读取结果不一致。

**③ 幻读（Phantom Read）**
- 同一事务中，多次查询，结果集的行数不一致

```sql
-- 事务 A
BEGIN;
SELECT * FROM account WHERE balance > 500;  -- 查到 2 行

-- 事务 B
BEGIN;
INSERT INTO account VALUES (3, '王五', 600);
COMMIT;

-- 事务 A
SELECT * FROM account WHERE balance > 500;  -- 查到 3 行（幻读）
COMMIT;
```

**问题：** 事务 A 两次查询结果集不一致。

### MySQL 的四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|---------|------|-----------|------|---------|
| **读未提交（RU）** | ✅ 可能 | ✅ 可能 | ✅ 可能 | 不加锁 |
| **读已提交（RC）** | ❌ 不可能 | ✅ 可能 | ✅ 可能 | MVCC（每次读取最新快照） |
| **可重复读（RR）** | ❌ 不可能 | ❌ 不可能 | ⚠️ 部分解决 | MVCC（读取事务开始时的快照） |
| **串行化（S）** | ❌ 不可能 | ❌ 不可能 | ❌ 不可能 | 加锁（读写都加锁） |

⚠️ **MySQL 默认隔离级别：** 可重复读（RR）

### ① 读未提交（Read Uncommitted）

**特点：** 不加锁，性能最好，但会出现脏读。

```sql
-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- 事务 A
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
-- 未提交

-- 事务 B
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 读到 500（脏读）
COMMIT;
```

**使用场景：** 几乎不用，数据不可靠。

### ② 读已提交（Read Committed）

**特点：** 只能读到已提交的数据，解决脏读。

**实现方式：** MVCC，每次读取都生成新的快照。

```sql
-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 事务 A
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 读到 600

-- 事务 B
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
COMMIT;

-- 事务 A
SELECT balance FROM account WHERE id = 1;  -- 读到 500（不可重复读）
COMMIT;
```

**使用场景：** Oracle、PostgreSQL 默认隔离级别。

### ③ 可重复读（Repeatable Read）

**特点：** 同一事务中，多次读取结果一致，解决不可重复读。

**实现方式：** MVCC，读取事务开始时的快照。

```sql
-- 设置隔离级别（MySQL 默认）
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 事务 A
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 读到 600

-- 事务 B
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
COMMIT;

-- 事务 A
SELECT balance FROM account WHERE id = 1;  -- 还是读到 600（可重复读）
COMMIT;
```

**幻读问题：**
- 快照读（普通 SELECT）：不会出现幻读
- 当前读（SELECT FOR UPDATE）：可能出现幻读，需要间隙锁解决

```sql
-- 事务 A
BEGIN;
SELECT * FROM account WHERE balance > 500;  -- 快照读，查到 2 行

-- 事务 B
BEGIN;
INSERT INTO account VALUES (3, '王五', 600);
COMMIT;

-- 事务 A
SELECT * FROM account WHERE balance > 500;  -- 快照读，还是 2 行（不会幻读）
SELECT * FROM account WHERE balance > 500 FOR UPDATE;  -- 当前读，查到 3 行（幻读）
COMMIT;
```

**使用场景：** MySQL 默认隔离级别，适合大部分场景。

### ④ 串行化（Serializable）

**特点：** 读写都加锁，完全避免并发问题，但性能最差。

```sql
-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 事务 A
BEGIN;
SELECT * FROM account WHERE id = 1;  -- 加共享锁（S 锁）

-- 事务 B
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;  -- 等待事务 A 释放锁
COMMIT;

-- 事务 A
COMMIT;  -- 释放锁
```

**使用场景：** 对数据一致性要求极高的场景（如金融系统）。

### 查看和设置隔离级别

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置会话隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 设置全局隔离级别
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

---

## 3. RC 隔离级别和 RR 隔离级别在不同场景下的应用？

### RC（读已提交）的特点

**优点：**
- 不会出现脏读
- 锁的持有时间短，并发性能好
- 不会出现间隙锁，减少死锁

**缺点：**
- 会出现不可重复读
- 会出现幻读

**适用场景：**
- 对数据一致性要求不高
- 并发量大，需要高性能
- 不需要可重复读的场景

**示例场景：**
```sql
-- 电商系统：查询商品库存
-- 不需要可重复读，只要读到最新的库存即可
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

BEGIN;
SELECT stock FROM product WHERE id = 1;  -- 读到 100
-- 其他事务可能修改库存
SELECT stock FROM product WHERE id = 1;  -- 读到 95（不可重复读，但符合业务需求）
COMMIT;
```

### RR（可重复读）的特点

**优点：**
- 不会出现脏读
- 不会出现不可重复读
- 快照读不会出现幻读

**缺点：**
- 使用间隙锁，可能增加死锁
- 并发性能略低于 RC

**适用场景：**
- 对数据一致性要求高
- 需要可重复读的场景
- 需要避免幻读的场景

**示例场景：**
```sql
-- 银行系统：转账
-- 需要可重复读，保证转账过程中余额不变
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 读到 600
-- 计算转账金额
UPDATE account SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

### RC vs RR 对比

| 特性 | RC（读已提交） | RR（可重复读） |
|------|---------------|---------------|
| 脏读 | ❌ 不会 | ❌ 不会 |
| 不可重复读 | ✅ 会 | ❌ 不会 |
| 幻读 | ✅ 会 | ⚠️ 快照读不会，当前读会 |
| 间隙锁 | ❌ 不使用 | ✅ 使用 |
| 死锁概率 | 低 | 高 |
| 并发性能 | 高 | 中 |
| 适用场景 | 高并发、低一致性 | 高一致性、中并发 |

### 实际应用建议

**使用 RC 的场景：**
- 互联网应用（电商、社交、内容平台）
- 读多写少的场景
- 对实时性要求高的场景

**使用 RR 的场景：**
- 金融系统（银行、支付）
- 对数据一致性要求极高的场景
- 需要避免幻读的场景

---

## 4. MVCC 是如何实现的？

### 什么是 MVCC

**MVCC（Multi-Version Concurrency Control）：** 多版本并发控制。

**核心思想：** 每行数据有多个版本，读取时根据事务的快照读取对应版本，不需要加锁。

**好处：**
- 读不加锁，写不阻塞读
- 提高并发性能

### MVCC 的实现原理

**① 隐藏列**

每行数据有三个隐藏列：

| 隐藏列 | 大小 | 作用 |
|--------|------|------|
| **DB_TRX_ID** | 6 字节 | 最后修改该行的事务 ID |
| **DB_ROLL_PTR** | 7 字节 | 回滚指针，指向 undo log |
| **DB_ROW_ID** | 6 字节 | 行 ID（没有主键时使用） |

**示例：**
```
id | name | balance | DB_TRX_ID | DB_ROLL_PTR
---|------|---------|-----------|-------------
1  | 张三 | 600     | 100       | 0x1234
```

**② undo log 版本链**

每次修改都会生成一个新版本，通过 DB_ROLL_PTR 形成版本链。

```
当前版本: id=1, balance=500, TRX_ID=102
   ↓ (DB_ROLL_PTR)
旧版本1: id=1, balance=600, TRX_ID=101
   ↓ (DB_ROLL_PTR)
旧版本2: id=1, balance=700, TRX_ID=100
```

**③ Read View（读视图）**

**Read View 记录当前活跃的事务 ID 列表，用于判断哪些版本可见。**

**Read View 包含：**
- `m_ids`：当前活跃的事务 ID 列表
- `min_trx_id`：最小的活跃事务 ID
- `max_trx_id`：下一个要分配的事务 ID
- `creator_trx_id`：创建 Read View 的事务 ID

**可见性判断规则：**
```
如果 DB_TRX_ID < min_trx_id:
    可见（该版本在所有活跃事务之前提交）

如果 DB_TRX_ID >= max_trx_id:
    不可见（该版本在 Read View 创建之后生成）

如果 min_trx_id <= DB_TRX_ID < max_trx_id:
    如果 DB_TRX_ID 在 m_ids 中:
        不可见（该版本由活跃事务生成，未提交）
    否则:
        可见（该版本已提交）
```

### MVCC 的工作流程

**示例：**
```sql
-- 事务 100
BEGIN;
UPDATE account SET balance = 600 WHERE id = 1;
COMMIT;

-- 事务 101
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
-- 未提交

-- 事务 102（读取）
BEGIN;
SELECT balance FROM account WHERE id = 1;
```

**步骤：**

**① 事务 102 创建 Read View**
```
m_ids: [101]  -- 事务 101 活跃
min_trx_id: 101
max_trx_id: 103
creator_trx_id: 102
```

**② 查找可见版本**
```
当前版本: balance=500, TRX_ID=101
  ↓ 判断：101 在 m_ids 中，不可见
  ↓ 沿着 DB_ROLL_PTR 找旧版本
旧版本: balance=600, TRX_ID=100
  ↓ 判断：100 < min_trx_id，可见
  ↓ 返回 balance=600
```

**③ 事务 102 读到 balance=600**

### RC 和 RR 的 MVCC 区别

**RC（读已提交）：**
- 每次读取都生成新的 Read View
- 能读到其他事务已提交的数据

**RR（可重复读）：**
- 只在事务开始时生成一次 Read View
- 读取的是事务开始时的快照

**示例：**
```sql
-- 事务 A
BEGIN;
SELECT balance FROM account WHERE id = 1;  -- 读到 600

-- 事务 B
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
COMMIT;

-- 事务 A
SELECT balance FROM account WHERE id = 1;
-- RC：读到 500（生成新的 Read View）
-- RR：读到 600（使用旧的 Read View）
```

---

## 5. RR 隔离级别是否彻底解决了幻读问题？

### 答案：没有彻底解决

**RR 隔离级别：**
- 快照读（普通 SELECT）：不会出现幻读
- 当前读（SELECT FOR UPDATE、UPDATE、DELETE）：可能出现幻读

### 快照读 vs 当前读

**① 快照读（Snapshot Read）**
- 读取的是 MVCC 的快照版本
- 不加锁
- 不会出现幻读

```sql
-- 事务 A
BEGIN;
SELECT * FROM account WHERE balance > 500;  -- 快照读，查到 2 行

-- 事务 B
BEGIN;
INSERT INTO account VALUES (3, '王五', 600);
COMMIT;

-- 事务 A
SELECT * FROM account WHERE balance > 500;  -- 快照读，还是 2 行（不会幻读）
COMMIT;
```

**② 当前读（Current Read）**
- 读取的是最新版本
- 加锁（共享锁或排他锁）
- 可能出现幻读

```sql
-- 事务 A
BEGIN;
SELECT * FROM account WHERE balance > 500 FOR UPDATE;  -- 当前读，查到 2 行

-- 事务 B
BEGIN;
INSERT INTO account VALUES (3, '王五', 600);
COMMIT;

-- 事务 A
SELECT * FROM account WHERE balance > 500 FOR UPDATE;  -- 当前读，查到 3 行（幻读）
COMMIT;
```

### 如何解决当前读的幻读？

**使用间隙锁（Gap Lock）**

**间隙锁：** 锁定索引之间的间隙，防止其他事务插入数据。

```sql
-- 事务 A
BEGIN;
SELECT * FROM account WHERE balance > 500 FOR UPDATE;  -- 加间隙锁

-- 事务 B
BEGIN;
INSERT INTO account VALUES (3, '王五', 600);  -- 等待间隙锁释放
COMMIT;

-- 事务 A
SELECT * FROM account WHERE balance > 500 FOR UPDATE;  -- 还是 2 行（不会幻读）
COMMIT;
```

**间隙锁的范围：**
```
索引值: 100, 200, 300, 400
间隙: (-∞, 100), (100, 200), (200, 300), (300, 400), (400, +∞)

SELECT * FROM account WHERE balance > 200 FOR UPDATE;
-- 锁定间隙: (200, 300), (300, 400), (400, +∞)
```

### 间隙锁的问题

**① 增加死锁概率**

```sql
-- 事务 A
BEGIN;
SELECT * FROM account WHERE balance > 200 FOR UPDATE;  -- 锁定 (200, +∞)

-- 事务 B
BEGIN;
SELECT * FROM account WHERE balance > 300 FOR UPDATE;  -- 锁定 (300, +∞)

-- 事务 A
INSERT INTO account VALUES (4, '赵六', 350);  -- 等待事务 B 释放 (300, +∞)

-- 事务 B
INSERT INTO account VALUES (5, '孙七', 250);  -- 等待事务 A 释放 (200, +∞)

-- 死锁！
```

**② 降低并发性能**

间隙锁会阻塞其他事务的插入操作，降低并发性能。

### 总结

| 场景 | 是否幻读 | 解决方式 |
|------|---------|---------|
| 快照读（SELECT） | ❌ 不会 | MVCC |
| 当前读（SELECT FOR UPDATE） | ✅ 会 | 间隙锁 |
| 串行化隔离级别 | ❌ 不会 | 读写都加锁 |

---

## 面试回答模板

**Q: MySQL 里面的 ACID 是如何实现的？**

A: 
- **原子性**：undo log 记录旧值，回滚时恢复
- **一致性**：由其他三个特性保证
- **隔离性**：MVCC（读不加锁）+ 锁（写加锁）
- **持久性**：redo log 记录新值，崩溃后恢复

**Q: 说一下 MySQL 事务隔离级别都有哪些？**

A: MySQL 有四种隔离级别：
1. **读未提交（RU）**：不加锁，会出现脏读
2. **读已提交（RC）**：MVCC，每次读取生成新快照，会出现不可重复读
3. **可重复读（RR）**：MVCC，只在事务开始时生成快照，MySQL 默认隔离级别
4. **串行化（S）**：读写都加锁，完全避免并发问题

**Q: RC 和 RR 的区别？**

A: 
- **RC**：每次读取都生成新的 Read View，能读到其他事务已提交的数据，会出现不可重复读
- **RR**：只在事务开始时生成一次 Read View，读取的是事务开始时的快照，不会出现不可重复读

**Q: MVCC 是如何实现的？**

A: MVCC 通过三个机制实现：
1. **隐藏列**：DB_TRX_ID（事务 ID）、DB_ROLL_PTR（回滚指针）
2. **undo log 版本链**：每次修改生成新版本，形成版本链
3. **Read View**：记录活跃事务 ID，判断哪些版本可见

**Q: RR 隔离级别是否彻底解决了幻读？**

A: 没有彻底解决。
- **快照读**（普通 SELECT）：不会出现幻读，通过 MVCC 实现
- **当前读**（SELECT FOR UPDATE）：可能出现幻读，需要间隙锁解决

**Q: 什么是间隙锁？**

A: 间隙锁是锁定索引之间的间隙，防止其他事务插入数据，用于解决当前读的幻读问题。但会增加死锁概率，降低并发性能。

---

## 面试简答版

**ACID 实现：**
- 原子性：undo log
- 一致性：其他三个特性保证
- 隔离性：MVCC + 锁
- 持久性：redo log

**隔离级别：** RU、RC、RR、S，MySQL 默认 RR

**RC vs RR：** RC 每次读取生成新快照，RR 只在事务开始时生成快照

**MVCC：** 隐藏列 + undo log 版本链 + Read View

**幻读：** 快照读不会，当前读会，需要间隙锁解决

**间隙锁：** 锁定索引间隙，防止插入，解决幻读，但增加死锁
