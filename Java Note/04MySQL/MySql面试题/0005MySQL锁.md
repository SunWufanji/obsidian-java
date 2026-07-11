# MySQL 锁

## 1. MySQL 中都有哪些锁？

### 什么是锁

**锁是用来控制多个事务并发访问同一资源时的机制，保证数据的一致性。**

想象图书馆借书，如果两个人同时想借同一本书，就需要一个机制来协调，这就是锁。

### MySQL 锁的分类

**按锁的粒度分类：**

| 锁类型 | 粒度 | 特点 | 适用场景 |
|--------|------|------|---------|
| **表锁** | 整个表 | 开销小，加锁快，不会死锁，并发度低 | 读多写少，全表扫描 |
| **行锁** | 单行记录 | 开销大，加锁慢，会死锁，并发度高 | 写多读少，精确查询 |
| **页锁** | 数据页 | 介于表锁和行锁之间 | 很少使用 |

**按锁的类型分类：**

| 锁类型 | 英文 | 作用 | 兼容性 |
|--------|------|------|--------|
| **共享锁（S锁）** | Shared Lock | 读锁，允许多个事务同时读 | S锁之间兼容，S锁和X锁互斥 |
| **排他锁（X锁）** | Exclusive Lock | 写锁，只允许一个事务写 | X锁和任何锁都互斥 |

**按锁的算法分类（InnoDB）：**

| 锁类型                    | 作用        | 使用场景               |
| ---------------------- | --------- | ------------------ |
| **记录锁（Record Lock）**   | 锁定单行记录    | 精确匹配（WHERE id = 1） |
| **间隙锁（Gap Lock）**      | 锁定索引之间的间隙 | 防止幻读               |
| **临键锁（Next-Key Lock）** | 记录锁 + 间隙锁 | RR 隔离级别默认使用        |

**按锁的实现方式分类：**

| 锁类型 | 实现方式 | 特点 |
|--------|---------|------|
| **悲观锁** | 数据库锁机制 | 先加锁再操作 |
| **乐观锁** | 版本号或时间戳 | 先操作再检查冲突 |

---

## 2. 行级锁是什么？

### 什么是行锁

**行锁：** 只锁定某一行记录，其他行不受影响，并发度高。

**InnoDB 支持行锁，MyISAM 不支持。**

### 行锁的类型

**① 共享锁（S锁，读锁）**

```sql
-- 手动加共享锁
SELECT * FROM employee WHERE id = 1 LOCK IN SHARE MODE;
```

**特点：**
- 允许多个事务同时读取同一行
- 阻止其他事务修改该行（X锁）

**示例：**
```sql
-- 事务 A
BEGIN;
SELECT * FROM employee WHERE id = 1 LOCK IN SHARE MODE;  -- 加 S 锁

-- 事务 B
BEGIN;
SELECT * FROM employee WHERE id = 1 LOCK IN SHARE MODE;  -- 成功，S 锁兼容
UPDATE employee SET salary = 10000 WHERE id = 1;  -- 等待，S 锁和 X 锁互斥
```

**② 排他锁（X锁，写锁）**

```sql
-- 手动加排他锁
SELECT * FROM employee WHERE id = 1 FOR UPDATE;

-- 自动加排他锁
UPDATE employee SET salary = 10000 WHERE id = 1;
DELETE FROM employee WHERE id = 1;
```

**特点：**
- 只允许一个事务修改该行
- 阻止其他事务读取或修改该行

**示例：**
```sql
-- 事务 A
BEGIN;
SELECT * FROM employee WHERE id = 1 FOR UPDATE;  -- 加 X 锁

-- 事务 B
BEGIN;
SELECT * FROM employee WHERE id = 1 LOCK IN SHARE MODE;  -- 等待
UPDATE employee SET salary = 10000 WHERE id = 1;  -- 等待
```

### 行锁的三种算法

**① 记录锁（Record Lock）**

**锁定单行记录。**

```sql
-- 精确匹配，加记录锁
SELECT * FROM employee WHERE id = 1 FOR UPDATE;
```

**锁定范围：** 只锁定 id=1 这一行

**② 间隙锁（Gap Lock）**

**锁定索引之间的间隙，防止其他事务插入数据。**

```sql
-- 范围查询，加间隙锁
SELECT * FROM employee WHERE id > 10 AND id < 20 FOR UPDATE;
```

**锁定范围：** 锁定 (10, 20) 之间的间隙，防止插入 id=11, 12, ..., 19

**作用：** 防止幻读

**示例：**
```sql
-- 事务 A
BEGIN;
SELECT * FROM employee WHERE id > 10 AND id < 20 FOR UPDATE;  -- 加间隙锁

-- 事务 B
BEGIN;
INSERT INTO employee VALUES (15, '张三', 10000);  -- 等待，间隙锁阻止插入
```

**③ 临键锁（Next-Key Lock）**

**临键锁 = 记录锁 + 间隙锁**

**RR 隔离级别默认使用临键锁。**

```sql
-- 范围查询，加临键锁
SELECT * FROM employee WHERE id >= 10 AND id <= 20 FOR UPDATE;
```

**锁定范围：** 
- 记录锁：锁定 id=10, 20 这两行
- 间隙锁：锁定 (10, 20) 之间的间隙

**作用：** 防止幻读 + 防止修改

---

## 3. InnoDB 存储引擎行级锁的原理是什么？

### 行锁的实现原理

**InnoDB 的行锁是通过锁定索引实现的，不是锁定记录本身。**

**关键点：**
- 如果查询没有使用索引，会升级为表锁
- 如果使用了索引，只锁定索引对应的行

### 锁定索引，不是锁定记录

```sql
-- 假设 id 是主键索引
SELECT * FROM employee WHERE id = 1 FOR UPDATE;
```

**锁定过程：**
1. 在主键索引的 B+ 树中找到 id=1 的索引记录
2. 对该索引记录加 X 锁
3. 其他事务无法修改 id=1 的记录

### 没有索引会升级为表锁

```sql
-- 假设 name 没有索引
SELECT * FROM employee WHERE name = '张三' FOR UPDATE;
```

**锁定过程：**
1. 没有索引，需要全表扫描
2. 扫描过程中对所有记录加锁
3. 升级为表锁，锁定整个表

⚠️ **注意：** 这会严重影响并发性能！

### 索引失效也会升级为表锁

```sql
-- 假设 age 有索引，但使用了函数导致索引失效
SELECT * FROM employee WHERE age + 1 = 26 FOR UPDATE;
```

**锁定过程：**
1. 索引失效，需要全表扫描
2. 升级为表锁

### 不同索引锁定不同行

```sql
-- 假设 id 是主键索引，dept_id 是普通索引

-- 事务 A：通过主键索引锁定
BEGIN;
SELECT * FROM employee WHERE id = 1 FOR UPDATE;

-- 事务 B：通过普通索引锁定
BEGIN;
SELECT * FROM employee WHERE dept_id = 1 FOR UPDATE;
```

**锁定情况：**
- 事务 A 锁定主键索引的 id=1
- 事务 B 锁定普通索引的 dept_id=1
- 如果 id=1 的记录的 dept_id 也是 1，事务 B 会等待

---

## 4. MySQL 表级锁和行级锁的区别在哪？

### 表锁 vs 行锁

| 特性 | 表锁 | 行锁 |
|------|------|------|
| **锁粒度** | 整个表 | 单行记录 |
| **加锁开销** | 小 | 大 |
| **加锁速度** | 快 | 慢 |
| **锁冲突概率** | 高 | 低 |
| **并发度** | 低 | 高 |
| **死锁** | 不会 | 会 |
| **适用场景** | 读多写少，全表扫描 | 写多读少，精确查询 |
| **存储引擎** | MyISAM、InnoDB | InnoDB |

### 表锁的使用

**① 手动加表锁**

```sql
-- 加读锁（共享锁）
LOCK TABLES employee READ;
SELECT * FROM employee;
UNLOCK TABLES;

-- 加写锁（排他锁）
LOCK TABLES employee WRITE;
UPDATE employee SET salary = 10000 WHERE id = 1;
UNLOCK TABLES;
```

**② 自动加表锁**

```sql
-- MyISAM 引擎自动加表锁
-- InnoDB 引擎在没有索引时升级为表锁
```

### 表锁的特点

**① 读锁（共享锁）**
- 当前会话只能读，不能写
- 其他会话可以读，不能写

```sql
-- 会话 A
LOCK TABLES employee READ;
SELECT * FROM employee;  -- ✅ 成功
UPDATE employee SET salary = 10000 WHERE id = 1;  -- ❌ 失败

-- 会话 B
SELECT * FROM employee;  -- ✅ 成功
UPDATE employee SET salary = 10000 WHERE id = 1;  -- ⏳ 等待
```

**② 写锁（排他锁）**
- 当前会话可以读写
- 其他会话不能读写

```sql
-- 会话 A
LOCK TABLES employee WRITE;
SELECT * FROM employee;  -- ✅ 成功
UPDATE employee SET salary = 10000 WHERE id = 1;  -- ✅ 成功

-- 会话 B
SELECT * FROM employee;  -- ⏳ 等待
UPDATE employee SET salary = 10000 WHERE id = 1;  -- ⏳ 等待
```

### 行锁的优势

**① 并发度高**

```sql
-- 事务 A
BEGIN;
UPDATE employee SET salary = 10000 WHERE id = 1;  -- 锁定 id=1

-- 事务 B
BEGIN;
UPDATE employee SET salary = 20000 WHERE id = 2;  -- 锁定 id=2，不冲突
```

**② 锁冲突概率低**

只有访问同一行时才会冲突，不同行互不影响。

**③ 适合高并发场景**

电商、社交、金融等高并发场景都使用行锁。

---

## 5. MySQL 在什么情况下会发生死锁？

### 什么是死锁

**死锁：** 两个或多个事务互相等待对方释放锁，导致都无法继续执行。

**类比：** 两个人过独木桥，谁也不让谁，都卡在桥上。

### 死锁的四个必要条件

1. **互斥条件：** 资源只能被一个事务占用
2. **请求与保持条件：** 事务持有锁的同时，请求新的锁
3. **不剥夺条件：** 锁不能被强制剥夺，只能主动释放
4. **循环等待条件：** 形成环形等待链

### 死锁的常见场景

**① 不同顺序加锁**

```sql
-- 事务 A
BEGIN;
UPDATE employee SET salary = 10000 WHERE id = 1;  -- 锁定 id=1
UPDATE employee SET salary = 20000 WHERE id = 2;  -- 等待 id=2

-- 事务 B
BEGIN;
UPDATE employee SET salary = 30000 WHERE id = 2;  -- 锁定 id=2
UPDATE employee SET salary = 40000 WHERE id = 1;  -- 等待 id=1

-- 死锁！
```

**原因：** 事务 A 持有 id=1，等待 id=2；事务 B 持有 id=2，等待 id=1。

**② 间隙锁冲突**

```sql
-- 事务 A
BEGIN;
SELECT * FROM employee WHERE id > 10 AND id < 20 FOR UPDATE;  -- 锁定 (10, 20)
INSERT INTO employee VALUES (15, '张三', 10000);  -- 等待

-- 事务 B
BEGIN;
SELECT * FROM employee WHERE id > 15 AND id < 25 FOR UPDATE;  -- 锁定 (15, 25)
INSERT INTO employee VALUES (18, '李四', 20000);  -- 等待

-- 死锁！
```

**③ 索引不当导致锁升级**

```sql
-- 假设 name 没有索引

-- 事务 A
BEGIN;
UPDATE employee SET salary = 10000 WHERE name = '张三';  -- 表锁

-- 事务 B
BEGIN;
UPDATE employee SET salary = 20000 WHERE name = '李四';  -- 等待表锁

-- 如果事务 A 又需要等待事务 B 持有的其他锁，就会死锁
```

### 如何避免死锁

**① 按相同顺序加锁**

```sql
-- 事务 A 和事务 B 都按 id 升序加锁
BEGIN;
UPDATE employee SET salary = 10000 WHERE id = 1;
UPDATE employee SET salary = 20000 WHERE id = 2;
COMMIT;
```

**② 缩短事务时间**

```sql
-- ❌ 不好的做法（事务时间长）
BEGIN;
SELECT * FROM employee WHERE id = 1 FOR UPDATE;
-- 执行复杂业务逻辑（耗时）
UPDATE employee SET salary = 10000 WHERE id = 1;
COMMIT;

-- ✅ 好的做法（事务时间短）
-- 先执行业务逻辑
BEGIN;
UPDATE employee SET salary = 10000 WHERE id = 1;
COMMIT;
```

**③ 使用索引避免锁升级**

```sql
-- 确保查询条件有索引
CREATE INDEX idx_name ON employee(name);

-- 避免索引失效
SELECT * FROM employee WHERE name = '张三' FOR UPDATE;  -- ✅
SELECT * FROM employee WHERE name LIKE '%张三%' FOR UPDATE;  -- ❌
```

**④ 降低隔离级别**

```sql
-- 从 RR 降低到 RC，减少间隙锁
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**⑤ 设置锁等待超时**

```sql
-- 查看锁等待超时时间
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';  -- 默认 50 秒

-- 设置锁等待超时时间
SET innodb_lock_wait_timeout = 10;  -- 10 秒
```

### 死锁的检测和处理

**① 死锁检测**

InnoDB 会自动检测死锁，并回滚其中一个事务。

```sql
-- 查看最近的死锁信息
SHOW ENGINE INNODB STATUS;
```

**② 死锁日志**

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2026-05-09 16:00:00
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 5 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 10, OS thread handle 123456, query id 100 localhost root updating
UPDATE employee SET salary = 10000 WHERE id = 1

*** (2) TRANSACTION:
TRANSACTION 12346, ACTIVE 3 sec starting index read
mysql tables in use 1, locked 1
2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 11, OS thread handle 123457, query id 101 localhost root updating
UPDATE employee SET salary = 20000 WHERE id = 2

*** WE ROLL BACK TRANSACTION (2)
```

**③ 死锁回滚策略**

InnoDB 会选择回滚代价最小的事务（通常是修改行数最少的事务）。

---

## 6. MySQL 死锁问题可能是由哪些原因所导致的？

### 死锁的常见原因

**① 不同顺序访问资源**

```sql
-- 事务 A：先锁 1，再锁 2
-- 事务 B：先锁 2，再锁 1
-- 导致循环等待
```

**② 间隙锁冲突**

```sql
-- RR 隔离级别下，范围查询会加间隙锁
-- 多个事务的间隙锁可能互相冲突
```

**③ 索引不当**

```sql
-- 没有索引或索引失效，导致锁升级为表锁
-- 增加锁冲突概率
```

**④ 事务时间过长**

```sql
-- 事务持有锁的时间越长，死锁概率越高
```

**⑤ 并发度过高**

```sql
-- 并发事务越多，锁冲突概率越高
```

**⑥ 业务逻辑不合理**

```sql
-- 在事务中执行耗时操作（如调用外部 API）
-- 增加锁持有时间
```

---

## 7. 怎么排查死锁问题？

### 排查步骤

**① 查看死锁日志**

```sql
-- 查看最近的死锁信息
SHOW ENGINE INNODB STATUS;
```

**关键信息：**
- 死锁发生的时间
- 涉及的事务
- 持有的锁和等待的锁
- 回滚的事务

**② 分析死锁原因**

```sql
-- 查看事务持有的锁
SELECT * FROM information_schema.INNODB_LOCKS;

-- 查看事务等待的锁
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 查看当前事务
SELECT * FROM information_schema.INNODB_TRX;
```

**③ 定位业务代码**

根据死锁日志中的 SQL 语句，定位到业务代码。

**④ 优化业务逻辑**

- 调整加锁顺序
- 缩短事务时间
- 添加索引
- 降低隔离级别

---

## 8. MySQL 中悲观锁和乐观锁的实现原理及使用场景是什么？

### 悲观锁

**悲观锁：** 假设会发生冲突，先加锁再操作。

**实现方式：** 数据库锁机制（行锁、表锁）

```sql
-- 悲观锁示例
BEGIN;
SELECT * FROM employee WHERE id = 1 FOR UPDATE;  -- 加排他锁
UPDATE employee SET salary = salary - 100 WHERE id = 1;
COMMIT;
```

**特点：**
- 先加锁，再操作
- 阻止其他事务修改
- 适合写多读少的场景

**使用场景：**
- 库存扣减（防止超卖）
- 账户余额修改（防止透支）
- 秒杀系统

**示例：库存扣减**
```sql
BEGIN;
-- 加锁查询库存
SELECT stock FROM product WHERE id = 1 FOR UPDATE;

-- 检查库存
IF stock > 0 THEN
    -- 扣减库存
    UPDATE product SET stock = stock - 1 WHERE id = 1;
END IF;

COMMIT;
```

### 乐观锁

**乐观锁：** 假设不会发生冲突，先操作再检查冲突。

**实现方式：** 版本号或时间戳

**① 版本号实现**

```sql
-- 添加版本号字段
ALTER TABLE employee ADD COLUMN version INT DEFAULT 0;

-- 乐观锁示例
-- 1. 查询数据和版本号
SELECT id, salary, version FROM employee WHERE id = 1;
-- 假设查到：id=1, salary=10000, version=5

-- 2. 更新时检查版本号
UPDATE employee 
SET salary = 11000, version = version + 1 
WHERE id = 1 AND version = 5;

-- 3. 检查影响行数
-- 如果影响行数为 0，说明版本号已变，更新失败（有冲突）
-- 如果影响行数为 1，说明更新成功（无冲突）
```

**② 时间戳实现**

```sql
-- 添加时间戳字段
ALTER TABLE employee ADD COLUMN update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;

-- 乐观锁示例
-- 1. 查询数据和时间戳
SELECT id, salary, update_time FROM employee WHERE id = 1;
-- 假设查到：id=1, salary=10000, update_time='2026-05-09 16:00:00'

-- 2. 更新时检查时间戳
UPDATE employee 
SET salary = 11000 
WHERE id = 1 AND update_time = '2026-05-09 16:00:00';

-- 3. 检查影响行数
```

**特点：**
- 不加锁，先操作
- 通过版本号检查冲突
- 适合读多写少的场景

**使用场景：**
- 商品信息修改
- 用户资料更新
- 文章编辑

### 悲观锁 vs 乐观锁

| 特性 | 悲观锁 | 乐观锁 |
|------|--------|--------|
| **实现方式** | 数据库锁 | 版本号/时间戳 |
| **加锁时机** | 操作前 | 不加锁 |
| **冲突处理** | 阻塞等待 | 重试或失败 |
| **并发性能** | 低 | 高 |
| **适用场景** | 写多读少 | 读多写少 |
| **典型应用** | 库存扣减、账户余额 | 商品信息、用户资料 |

---

## 9. MySQL 中悲观锁和乐观锁的实现原理及使用场景是什么？

### 悲观锁和乐观锁的选择

**选择悲观锁的场景：**
- 写操作频繁
- 冲突概率高
- 对数据一致性要求极高
- 不能容忍重试

**选择乐观锁的场景：**
- 读操作频繁
- 冲突概率低
- 对性能要求高
- 可以容忍重试

**示例对比：**

**场景 1：库存扣减（悲观锁）**
```sql
-- 悲观锁：先锁定，再扣减
BEGIN;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

**场景 2：商品信息修改（乐观锁）**
```sql
-- 乐观锁：先修改，再检查版本号
SELECT id, name, version FROM product WHERE id = 1;
UPDATE product SET name = '新名称', version = version + 1 
WHERE id = 1 AND version = 5;
```

---

## 面试回答模板

**Q: MySQL 中都有哪些锁？**

A: MySQL 的锁可以从多个维度分类：
1. **按粒度**：表锁、行锁、页锁
2. **按类型**：共享锁（S锁）、排他锁（X锁）
3. **按算法**：记录锁、间隙锁、临键锁
4. **按实现**：悲观锁、乐观锁

**Q: 行锁和表锁的区别？**

A: 
- **表锁**：锁定整个表，开销小，并发度低，不会死锁
- **行锁**：锁定单行记录，开销大，并发度高，会死锁
- InnoDB 支持行锁，MyISAM 只支持表锁

**Q: 什么情况下会发生死锁？**

A: 死锁的常见场景：
1. 不同顺序访问资源（事务 A 先锁 1 再锁 2，事务 B 先锁 2 再锁 1）
2. 间隙锁冲突（RR 隔离级别下）
3. 索引不当导致锁升级
4. 事务时间过长

**Q: 如何避免死锁？**

A: 
1. 按相同顺序加锁
2. 缩短事务时间
3. 使用索引避免锁升级
4. 降低隔离级别（从 RR 到 RC）
5. 设置锁等待超时

**Q: 悲观锁和乐观锁的区别？**

A: 
- **悲观锁**：先加锁再操作，使用数据库锁机制，适合写多读少
- **乐观锁**：先操作再检查冲突，使用版本号或时间戳，适合读多写少

**Q: InnoDB 的行锁是如何实现的？**

A: InnoDB 的行锁是通过锁定索引实现的，不是锁定记录本身。如果查询没有使用索引，会升级为表锁。

---

## 面试简答版

**锁的分类：** 表锁、行锁、共享锁、排他锁、记录锁、间隙锁、临键锁

**表锁 vs 行锁：** 表锁粒度大并发低，行锁粒度小并发高

**死锁原因：** 不同顺序加锁、间隙锁冲突、索引不当、事务时间长

**避免死锁：** 相同顺序加锁、缩短事务、使用索引、降低隔离级别

**悲观锁：** 先加锁再操作，数据库锁机制，写多读少

**乐观锁：** 先操作再检查，版本号机制，读多写少

**行锁实现：** 锁定索引，没有索引升级为表锁
