# SQL 初级查询

## 1. 单表查询（SELECT、FROM、AS、DISTINCT）

### 基本查询
查询就是从表里拿数据，最简单的形式就是"我要看某个表的某些列"。

```sql
-- 查询所有员工的姓名和月薪
SELECT emp_name, salary FROM employee;

-- 查询所有列（不推荐，性能差）
SELECT * FROM employee;

-- 使用别名让结果更易读
SELECT emp_name AS 姓名, salary AS 月薪 FROM employee;
```

**输出结果：**
```
姓名    | 月薪
--------|--------
张三    | 25000.00
李四    | 18000.00
王五    | 15000.00
...
```

### DISTINCT 去重
想象你在统计"公司有哪些部门"，不想看到重复的部门编号。

```sql
-- 查询公司有哪些部门（去重）
SELECT DISTINCT dept_id FROM employee;
```

**输出结果：**
```
dept_id
-------
1
2
3
4
5
```

⚠️ **易错点：** DISTINCT 作用于所有查询列的组合，不是单独某一列。

```sql
-- 这会对 (dept_id, job_id) 组合去重，不是只对 dept_id 去重
SELECT DISTINCT dept_id, job_id FROM employee;
```

---

## 2. 查询条件（WHERE、IN、LIKE、IS NULL、AND/OR/NOT）

### WHERE 基本条件
就像筛选器，只要符合条件的数据。

```sql
-- 查询月薪大于 15000 的员工
SELECT emp_name, salary FROM employee WHERE salary > 15000;

-- 查询技术部（部门编号为 1）的员工
SELECT emp_name, dept_id FROM employee WHERE dept_id = 1;
```

### IN 多值匹配
想查"在这几个值里面的"，用 IN 比写一堆 OR 简洁。

```sql
-- 查询技术部和销售部的员工
SELECT emp_name, dept_id FROM employee WHERE dept_id IN (1, 2);

-- 等价于（但更啰嗦）
SELECT emp_name, dept_id FROM employee WHERE dept_id = 1 OR dept_id = 2;
```

### LIKE 模糊查询
用通配符匹配文本：
- `%` 代表任意多个字符（包括 0 个）
- `_` 代表单个字符

```sql
-- 查询姓"张"的员工
SELECT emp_name FROM employee WHERE emp_name LIKE '张%';

-- 查询邮箱是 company.com 的员工
SELECT emp_name, email FROM employee WHERE email LIKE '%@company.com';

-- 查询名字是两个字的员工
SELECT emp_name FROM employee WHERE emp_name LIKE '__';
```

### IS NULL 空值判断
⚠️ **重要：** NULL 不能用 `=` 判断，必须用 `IS NULL` 或 `IS NOT NULL`。

```sql
-- 查询没有经理的员工（高层管理者）
SELECT emp_name, manager FROM employee WHERE manager IS NULL;

-- 错误写法（查不出结果）
SELECT emp_name FROM employee WHERE manager = NULL;  -- ❌
```

**为什么？** NULL 表示"未知"，任何值和 NULL 比较结果都是 NULL（不是 true 也不是 false）。

### AND/OR/NOT 组合条件
```sql
-- 查询技术部月薪大于 12000 的员工
SELECT emp_name, salary FROM employee 
WHERE dept_id = 1 AND salary > 12000;

-- 查询月薪大于 15000 或有奖金大于 3000 的员工
SELECT emp_name, salary, bonus FROM employee 
WHERE salary > 15000 OR bonus > 3000;

-- 查询不在技术部的员工
SELECT emp_name, dept_id FROM employee WHERE dept_id != 1;
-- 或者
SELECT emp_name, dept_id FROM employee WHERE NOT dept_id = 1;
```

---

## 3. 结果排序（ORDER BY、ASC/DESC、中文排序、空值排序）

### 基本排序
```sql
-- 按月薪从低到高排序
SELECT emp_name, salary FROM employee ORDER BY salary ASC;

-- 按月薪从高到低排序（DESC 降序）
SELECT emp_name, salary FROM employee ORDER BY salary DESC;

-- 默认是升序，ASC 可以省略
SELECT emp_name, salary FROM employee ORDER BY salary;
```

### 多列排序
先按第一列排，相同的再按第二列排。

```sql
-- 先按部门分组，同部门内按月薪降序
SELECT emp_name, dept_id, salary FROM employee 
ORDER BY dept_id ASC, salary DESC;
```

**输出结果：**
```
emp_name | dept_id | salary
---------|---------|--------
张三     | 1       | 25000.00
李四     | 1       | 18000.00
王五     | 1       | 15000.00
周八     | 2       | 22000.00
吴九     | 2       | 10000.00
...
```

### 中文排序
中文按拼音排序需要指定字符集。

```sql
-- 按姓名拼音排序
SELECT emp_name FROM employee ORDER BY CONVERT(emp_name USING gbk);
```

### 空值排序
⚠️ **注意：** NULL 值在排序中的位置：
- MySQL 中 NULL 被认为是最小值（ASC 时排最前，DESC 时排最后）

```sql
-- 查询所有员工按经理编号排序（NULL 会排在最前面）
SELECT emp_name, manager FROM employee ORDER BY manager ASC;
```

---

## 4. 限定数量（FETCH、LIMIT、OFFSET）

### LIMIT 限制返回行数
想看"前几名"或者"分页查询"时用。

```sql
-- 查询月薪最高的 3 名员工
SELECT emp_name, salary FROM employee 
ORDER BY salary DESC 
LIMIT 3;

-- 分页查询：跳过前 5 条，取 3 条（第 6-8 条）
SELECT emp_name, salary FROM employee 
ORDER BY salary DESC 
LIMIT 3 OFFSET 5;

-- 简写形式：LIMIT offset, count
SELECT emp_name, salary FROM employee 
ORDER BY salary DESC 
LIMIT 5, 3;  -- 跳过 5 条，取 3 条
```

**输出结果（前 3 名）：**
```
emp_name | salary
---------|--------
张三     | 25000.00
周八     | 22000.00
李四     | 18000.00
```

⚠️ **分页场景：** 每页 10 条，查第 3 页：`LIMIT 10 OFFSET 20`（跳过前 20 条）

### FETCH（标准 SQL 语法）
FETCH 是 SQL 标准中限制返回行数的语句，功能和 LIMIT 一样，但语法更规范。

```sql
-- 查询月薪最高的 3 名员工
SELECT emp_name, salary FROM employee 
ORDER BY salary DESC 
FETCH NEXT 3 ROWS ONLY;

-- 分页查询：跳过前 5 条，取 3 条
SELECT emp_name, salary FROM employee 
ORDER BY salary DESC 
OFFSET 5 ROWS
FETCH NEXT 3 ROWS ONLY;

-- 查询排名第 4-6 的员工
SELECT emp_name, salary FROM employee 
ORDER BY salary DESC 
OFFSET 3 ROWS           -- 跳过前 3 名
FETCH NEXT 3 ROWS ONLY; -- 取第 4-6 名

-- 包含并列排名（WITH TIES）
-- 如果第 3 名有多人工资相同，都返回
SELECT emp_name, salary FROM employee 
ORDER BY salary DESC 
FETCH NEXT 3 ROWS WITH TIES;
```

⚠️ **FETCH vs LIMIT：**
- FETCH 是 SQL 标准语法，LIMIT 是 MySQL 方言
- FETCH 必须配合 ORDER BY 使用
- MySQL 8.0+ 才支持 FETCH，之前版本只能用 LIMIT
- `NEXT` 可以替换为 `FIRST`（意思一样）
- `ROWS` 可以写成 `ROW`（单数形式）

---

## 5. 常见函数（字符函数、数字函数、日期函数）

### 字符函数
```sql
-- CONCAT 拼接字符串
SELECT CONCAT(emp_name, '(', dept_id, ')') AS 员工信息 FROM employee;
-- 输出：张三(1)

-- LENGTH 字符串长度
SELECT emp_name, LENGTH(emp_name) AS 名字长度 FROM employee;

-- UPPER/LOWER 大小写转换
SELECT UPPER(email) FROM employee;

-- SUBSTRING 截取子串（从第 1 位开始取 3 个字符）
SELECT SUBSTRING(emp_name, 1, 3) FROM employee;

-- TRIM 去除空格
SELECT TRIM('  hello  ');  -- 输出：hello
```

### 数字函数
```sql
-- ROUND 四舍五入
SELECT emp_name, ROUND(salary * 1.1, 2) AS 涨薪后 FROM employee;

-- CEIL 向上取整，FLOOR 向下取整
SELECT CEIL(12.3);   -- 输出：13
SELECT FLOOR(12.8);  -- 输出：12

-- ABS 绝对值
SELECT ABS(-100);  -- 输出：100

-- MOD 取余
SELECT MOD(10, 3);  -- 输出：1
```

### 日期函数
```sql
-- NOW 当前日期时间
SELECT NOW();  -- 输出：2026-05-08 19:25:11

-- CURDATE 当前日期
SELECT CURDATE();  -- 输出：2026-05-08

-- YEAR/MONTH/DAY 提取日期部分
SELECT emp_name, YEAR(hire_date) AS 入职年份 FROM employee;

-- DATEDIFF 日期差（天数）
SELECT emp_name, DATEDIFF(NOW(), hire_date) AS 入职天数 FROM employee;

-- DATE_ADD 日期加减
SELECT DATE_ADD(NOW(), INTERVAL 7 DAY);  -- 7 天后
SELECT DATE_ADD(NOW(), INTERVAL -1 MONTH);  -- 1 个月前
```

**实际例子：查询入职超过 3 年的员工**
```sql
SELECT emp_name, hire_date FROM employee 
WHERE DATEDIFF(NOW(), hire_date) > 365 * 3;
```

---

## 6. CASE 表达式（CASE、WHEN、THEN、ELSE、END）

### 什么是 CASE
就像编程里的 if-else，根据不同条件返回不同结果。

### 简单 CASE（等值判断）
```sql
-- 根据性别显示中文
SELECT emp_name,
    CASE sex
        WHEN '男' THEN '先生'
        WHEN '女' THEN '女士'
        ELSE '未知'
    END AS 称呼
FROM employee;
```

**输出结果：**
```
emp_name | 称呼
---------|-----
张三     | 先生
李四     | 女士
王五     | 先生
```

### 搜索 CASE（条件判断）
```sql
-- 根据月薪划分等级
SELECT emp_name, salary,
    CASE
        WHEN salary >= 20000 THEN '高薪'
        WHEN salary >= 15000 THEN '中等'
        WHEN salary >= 10000 THEN '一般'
        ELSE '较低'
    END AS 薪资等级
FROM employee;
```

**输出结果：**
```
emp_name | salary   | 薪资等级
---------|----------|--------
张三     | 25000.00 | 高薪
李四     | 18000.00 | 中等
王五     | 15000.00 | 中等
赵六     | 12000.00 | 一般
```

⚠️ **注意：** CASE 是表达式，不是语句，必须写在 SELECT、WHERE、ORDER BY 等子句中。

---

## 7. 分组汇总（GROUP BY、AVG、SUM、COUNT、MAX/MIN、HAVING）

### 为什么要分组
想统计"每个部门的平均工资"、"每个职位有多少人"，就需要分组。

### 基本分组
```sql
-- 统计每个部门的人数
SELECT dept_id, COUNT(*) AS 人数 FROM employee GROUP BY dept_id;
```

**输出结果：**
```
dept_id | 人数
--------|-----
1       | 5
2       | 3
3       | 2
4       | 1
5       | 1
```

### 常用聚合函数
```sql
-- 每个部门的平均工资、总工资、最高工资、最低工资
SELECT 
    dept_id,
    COUNT(*) AS 人数,
    AVG(salary) AS 平均工资,
    SUM(salary) AS 总工资,
    MAX(salary) AS 最高工资,
    MIN(salary) AS 最低工资
FROM employee 
GROUP BY dept_id;
```

**输出结果：**
```
dept_id | 人数 | 平均工资  | 总工资   | 最高工资  | 最低工资
--------|-----|----------|---------|----------|----------
1       | 5   | 15600.00 | 78000.00| 25000.00 | 8000.00
2       | 3   | 13666.67 | 41000.00| 22000.00 | 9000.00
```

### HAVING 过滤分组
⚠️ **WHERE vs HAVING：**
- WHERE：分组前过滤（过滤行）
- HAVING：分组后过滤（过滤分组）

```sql
-- 查询平均工资大于 15000 的部门
SELECT dept_id, AVG(salary) AS 平均工资 
FROM employee 
GROUP BY dept_id 
HAVING AVG(salary) > 15000;

-- 查询人数大于 3 的部门
SELECT dept_id, COUNT(*) AS 人数 
FROM employee 
GROUP BY dept_id 
HAVING COUNT(*) > 3;
```

### 完整查询顺序
```sql
SELECT dept_id, AVG(salary) AS 平均工资
FROM employee
WHERE salary > 10000          -- 1. 先过滤：只看月薪 > 10000 的员工
GROUP BY dept_id              -- 2. 再分组：按部门分组
HAVING AVG(salary) > 15000    -- 3. 过滤分组：只要平均工资 > 15000 的部门
ORDER BY 平均工资 DESC;        -- 4. 最后排序
```

⚠️ **易错点：** SELECT 中出现的非聚合列必须在 GROUP BY 中。

```sql
-- ❌ 错误：emp_name 不在 GROUP BY 中
SELECT dept_id, emp_name, AVG(salary) FROM employee GROUP BY dept_id;

-- ✅ 正确
SELECT dept_id, AVG(salary) FROM employee GROUP BY dept_id;
```

---

## 面试回答模板

**Q: SQL 查询的执行顺序是什么？**

A: SQL 查询的逻辑执行顺序是：
1. FROM - 确定数据来源
2. WHERE - 过滤行
3. GROUP BY - 分组
4. HAVING - 过滤分组
5. SELECT - 选择列
6. ORDER BY - 排序
7. LIMIT - 限制返回行数

注意：WHERE 在分组前过滤，HAVING 在分组后过滤。

**Q: COUNT(*) 和 COUNT(列名) 有什么区别？**

A: 
- `COUNT(*)` 统计所有行数，包括 NULL
- `COUNT(列名)` 统计该列非 NULL 的行数
- 例如：`COUNT(manager)` 不会统计 manager 为 NULL 的员工

**Q: DISTINCT 和 GROUP BY 的区别？**

A:
- DISTINCT 用于去重，返回唯一值
- GROUP BY 用于分组聚合，通常配合聚合函数使用
- 单纯去重时，DISTINCT 更简洁；需要统计时，用 GROUP BY
