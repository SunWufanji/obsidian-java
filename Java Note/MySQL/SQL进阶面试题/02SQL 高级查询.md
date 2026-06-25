# SQL 高级查询

## 1. 连接查询（INNER JOIN、LEFT/RIGHT/FULL JOIN、CROSS JOIN、self join）

### 为什么需要连接
数据分散在多个表里，想把它们拼起来看。比如员工表有部门编号，但部门名称在部门表里，需要连接才能看到"张三在技术部"。

### INNER JOIN（内连接）
只保留两边都匹配上的数据，像两个圆的交集。

```sql
-- 查询员工姓名和所属部门名称
SELECT e.emp_name, d.dept_name
FROM employee e
INNER JOIN department d ON e.dept_id = d.dept_id;
```

**输出结果：**
```
emp_name | dept_name
---------|----------
张三     | 技术部
李四     | 技术部
王五     | 技术部
周八     | 销售部
...
```

⚠️ **注意：** 如果员工的 dept_id 为 NULL 或在 department 表中不存在，该员工不会出现在结果中。

### LEFT JOIN（左连接）
保留左表所有数据，右表匹配不上的显示 NULL。

```sql
-- 查询所有员工及其部门（包括没有部门的员工）
SELECT e.emp_name, d.dept_name
FROM employee e
LEFT JOIN department d ON e.dept_id = d.dept_id;
```

如果某个员工的 dept_id 不存在，dept_name 会显示 NULL。

### RIGHT JOIN（右连接）
保留右表所有数据，左表匹配不上的显示 NULL。

```sql
-- 查询所有部门及其员工（包括没有员工的部门）
SELECT d.dept_name, e.emp_name
FROM employee e
RIGHT JOIN department d ON e.dept_id = d.dept_id;
```

### FULL JOIN（全连接）
保留两边所有数据，匹配不上的都显示 NULL。

⚠️ **MySQL 不支持 FULL JOIN**，需要用 UNION 模拟：

```sql
-- MySQL 中模拟全连接
SELECT e.emp_name, d.dept_name
FROM employee e
LEFT JOIN department d ON e.dept_id = d.dept_id
UNION
SELECT e.emp_name, d.dept_name
FROM employee e
RIGHT JOIN department d ON e.dept_id = d.dept_id;
```

### CROSS JOIN（交叉连接）
笛卡尔积，左表每一行和右表每一行组合。

```sql
-- 每个员工和每个部门的组合（一般没啥实际用途）
SELECT e.emp_name, d.dept_name
FROM employee e
CROSS JOIN department d;
```

如果 employee 有 12 行，department 有 5 行，结果是 12 × 5 = 60 行。

### self join（自连接）
表和自己连接，常用于查询层级关系。

```sql
-- 查询每个员工及其经理的姓名
SELECT e.emp_name AS 员工, m.emp_name AS 经理
FROM employee e
LEFT JOIN employee m ON e.manager = m.emp_id;
```

**输出结果：**
```
员工   | 经理
-------|------
张三   | NULL
李四   | 张三
王五   | 李四
赵六   | 李四
```

---

## 2. 集合运算（UNION、UNION ALL、INTERSECT、EXCEPT）

### UNION（并集去重）
合并两个查询结果，自动去重。

```sql
-- 查询月薪大于 20000 或奖金大于 3000 的员工（去重）
SELECT emp_name, salary FROM employee WHERE salary > 20000
UNION
SELECT emp_name, salary FROM employee WHERE bonus > 3000;
```

⚠️ **要求：** 两个查询的列数和类型必须一致。

### UNION ALL（并集不去重）
合并两个查询结果，保留重复行，性能比 UNION 好。

```sql
-- 合并结果但不去重
SELECT emp_name FROM employee WHERE dept_id = 1
UNION ALL
SELECT emp_name FROM employee WHERE salary > 15000;
```

### INTERSECT（交集）
⚠️ **MySQL 不支持 INTERSECT**，需要用 INNER JOIN 或 IN 模拟：

```sql
-- 查询既在技术部又月薪大于 15000 的员工
SELECT emp_name FROM employee 
WHERE dept_id = 1 AND salary > 15000;

-- 或者用 IN
SELECT emp_name FROM employee WHERE dept_id = 1
AND emp_name IN (SELECT emp_name FROM employee WHERE salary > 15000);
```

### EXCEPT（差集）
⚠️ **MySQL 不支持 EXCEPT**，需要用 NOT IN 或 LEFT JOIN 模拟：

```sql
-- 查询在技术部但月薪不大于 15000 的员工
SELECT emp_name FROM employee 
WHERE dept_id = 1 AND salary <= 15000;

-- 或者用 NOT IN
SELECT emp_name FROM employee WHERE dept_id = 1
AND emp_name NOT IN (SELECT emp_name FROM employee WHERE salary > 15000);
```

---

## 3. 子查询（标量子查询、行子查询、表子查询、关联子查询、EXISTS）

### 什么是子查询
查询里套查询，内层查询的结果给外层用。

### 标量子查询（返回单个值）
```sql
-- 查询月薪高于平均工资的员工
SELECT emp_name, salary 
FROM employee 
WHERE salary > (SELECT AVG(salary) FROM employee);
```

### 行子查询（返回一行多列）
```sql
-- 查询与张三同部门同职位的员工
SELECT emp_name FROM employee
WHERE (dept_id, job_id) = (
    SELECT dept_id, job_id FROM employee WHERE emp_name = '张三'
);
```

### 表子查询（返回多行多列）
```sql
-- 查询每个部门工资最高的员工
SELECT e.emp_name, e.dept_id, e.salary
FROM employee e
WHERE (e.dept_id, e.salary) IN (
    SELECT dept_id, MAX(salary) 
    FROM employee 
    GROUP BY dept_id
);
```

### 关联子查询（子查询引用外层表）
```sql
-- 查询工资高于本部门平均工资的员工
SELECT e1.emp_name, e1.dept_id, e1.salary
FROM employee e1
WHERE e1.salary > (
    SELECT AVG(e2.salary) 
    FROM employee e2 
    WHERE e2.dept_id = e1.dept_id
);
```

⚠️ **性能问题：** 关联子查询会对外层每一行都执行一次，性能较差，能用 JOIN 就用 JOIN。

### EXISTS（存在性检查）
检查子查询是否返回结果，返回 TRUE/FALSE。

```sql
-- 查询有员工的部门
SELECT d.dept_name
FROM department d
WHERE EXISTS (
    SELECT 1 FROM employee e WHERE e.dept_id = d.dept_id
);

-- 查询没有员工的部门
SELECT d.dept_name
FROM department d
WHERE NOT EXISTS (
    SELECT 1 FROM employee e WHERE e.dept_id = d.dept_id
);
```

⚠️ **EXISTS vs IN：**
- EXISTS 只关心是否有结果，不关心具体值，性能通常更好
- IN 需要返回具体值列表

---

## 4. 高级分组（ROLLUP、CUBE、GROUPING SETS、GROUPING）

### ROLLUP（分层汇总）
从右到左逐层汇总，常用于生成小计和总计。

```sql
-- 按部门和职位分组，并生成小计和总计
SELECT dept_id, job_id, COUNT(*) AS 人数
FROM employee
GROUP BY dept_id, job_id WITH ROLLUP;
```

**输出结果：**
```
dept_id | job_id | 人数
--------|--------|-----
1       | 1      | 1
1       | 2      | 1
1       | 3      | 1
1       | NULL   | 5    -- 部门 1 小计
2       | 1      | 1
2       | 6      | 2
2       | NULL   | 3    -- 部门 2 小计
NULL    | NULL   | 12   -- 总计
```

### CUBE（多维汇总）
⚠️ **MySQL 不支持 CUBE**，PostgreSQL 支持。

CUBE 会生成所有维度组合的汇总。

### GROUPING SETS（自定义分组）
⚠️ **MySQL 不支持 GROUPING SETS**，PostgreSQL 支持。

可以在一个查询中指定多个分组维度。

### GROUPING 函数
判断某列是否是汇总行。

```sql
SELECT 
    dept_id,
    job_id,
    COUNT(*) AS 人数,
    GROUPING(dept_id) AS is_dept_total,
    GROUPING(job_id) AS is_job_total
FROM employee
GROUP BY dept_id, job_id WITH ROLLUP;
```

GROUPING 返回 1 表示该列是汇总行，返回 0 表示是实际分组。

---

## 5. 通用表表达式（普通 CTE、递归 CTE）

### 什么是 CTE
CTE（Common Table Expression）就是给子查询起个名字，让 SQL 更易读。

### 普通 CTE
```sql
-- 查询高薪员工（月薪 > 15000）及其部门
WITH high_salary_emp AS (
    SELECT emp_id, emp_name, dept_id, salary
    FROM employee
    WHERE salary > 15000
)
SELECT h.emp_name, d.dept_name, h.salary
FROM high_salary_emp h
JOIN department d ON h.dept_id = d.dept_id;
```

**好处：** 比嵌套子查询更清晰，可以多次引用。

### 递归 CTE
用于处理层级数据，比如组织架构、评论树。

```sql
-- 查询张三及其所有下属（递归）
WITH RECURSIVE emp_tree AS (
    -- 初始查询：找到张三
    SELECT emp_id, emp_name, manager, 1 AS level
    FROM employee
    WHERE emp_name = '张三'
    
    UNION ALL
    
    -- 递归查询：找下属
    SELECT e.emp_id, e.emp_name, e.manager, et.level + 1
    FROM employee e
    JOIN emp_tree et ON e.manager = et.emp_id
)
SELECT emp_name, level FROM emp_tree;
```

**输出结果：**
```
emp_name | level
---------|------
张三     | 1
李四     | 2
王五     | 3
赵六     | 3
孙七     | 3
```

⚠️ **注意：** 递归 CTE 必须有终止条件，否则会无限循环。

---

## 6. 窗口函数（ROW_NUMBER、RANK、DENSE_RANK、FIRST_VALUE、LAST_VALUE、LEAD、LAG）

### 什么是窗口函数
在不改变行数的情况下，对每一行计算聚合值或排名。和 GROUP BY 不同，窗口函数不会合并行。

### ROW_NUMBER（行号）
给每一行分配唯一的序号。

```sql
-- 按月薪降序排列，给每个员工编号
SELECT 
    emp_name, 
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num
FROM employee;
```

**输出结果：**
```
emp_name | salary   | row_num
---------|----------|--------
张三     | 25000.00 | 1
周八     | 22000.00 | 2
李四     | 18000.00 | 3
```

### RANK 和 DENSE_RANK（排名）
- RANK：相同值排名相同，下一个排名跳号（1, 2, 2, 4）
- DENSE_RANK：相同值排名相同，下一个排名不跳号（1, 2, 2, 3）

```sql
-- 按月薪排名
SELECT 
    emp_name, 
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank_num,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank_num
FROM employee;
```

### 分区窗口（PARTITION BY）
在每个分组内独立计算窗口函数。

```sql
-- 每个部门内按月薪排名
SELECT 
    emp_name, 
    dept_id,
    salary,
    RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dept_rank
FROM employee;
```

**输出结果：**
```
emp_name | dept_id | salary   | dept_rank
---------|---------|----------|----------
张三     | 1       | 25000.00 | 1
李四     | 1       | 18000.00 | 2
王五     | 1       | 15000.00 | 3
周八     | 2       | 22000.00 | 1
吴九     | 2       | 10000.00 | 2
```

### FIRST_VALUE 和 LAST_VALUE
获取窗口内第一个或最后一个值。

```sql
-- 查询每个部门工资最高的员工姓名
SELECT DISTINCT
    dept_id,
    FIRST_VALUE(emp_name) OVER (PARTITION BY dept_id ORDER BY salary DESC) AS highest_paid
FROM employee;
```

### LEAD 和 LAG（前后行）
- LAG：获取前一行的值
- LEAD：获取后一行的值

```sql
-- 查询每个员工和下一个员工的工资差
SELECT 
    emp_name,
    salary,
    LEAD(salary) OVER (ORDER BY salary DESC) AS next_salary,
    salary - LEAD(salary) OVER (ORDER BY salary DESC) AS salary_diff
FROM employee;
```

**输出结果：**
```
emp_name | salary   | next_salary | salary_diff
---------|----------|-------------|------------
张三     | 25000.00 | 22000.00    | 3000.00
周八     | 22000.00 | 18000.00    | 4000.00
李四     | 18000.00 | 17000.00    | 1000.00
```

---

## 面试回答模板

**Q: INNER JOIN 和 LEFT JOIN 的区别？**

A: 
- INNER JOIN 只返回两表都匹配的数据
- LEFT JOIN 返回左表所有数据，右表匹配不上的显示 NULL
- 使用场景：需要保留左表所有记录时用 LEFT JOIN，只要匹配数据时用 INNER JOIN

**Q: 子查询和 JOIN 的性能对比？**

A:
- JOIN 通常性能更好，优化器可以选择最优执行计划
- 关联子查询性能最差，会对外层每一行都执行一次
- 建议：能用 JOIN 就用 JOIN，避免关联子查询

**Q: 窗口函数和 GROUP BY 的区别？**

A:
- GROUP BY 会合并行，返回每组一行
- 窗口函数不改变行数，为每一行计算聚合值
- 使用场景：需要保留明细数据同时显示聚合值时用窗口函数

**Q: RANK 和 DENSE_RANK 的区别？**

A:
- RANK 相同值排名相同，下一个排名跳号（1, 2, 2, 4）
- DENSE_RANK 相同值排名相同，下一个排名连续（1, 2, 2, 3）
- 使用场景：需要连续排名用 DENSE_RANK，否则用 RANK

**Q: 什么时候用 EXISTS 而不是 IN？**

A:
- EXISTS 只检查是否存在，不返回具体值，性能通常更好
- IN 需要返回完整结果集
- 使用场景：子查询返回大量数据时优先用 EXISTS
