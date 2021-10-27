# MySql常用命令

## 查询相关

### 基本查询

```sql
SELECT * FROM <table_name>
```

### 条件查询

```sql
SELECT * FROM students WHERE score >= 80;
SELECT * FROM students WHERE score >= 80 AND gender = 'M';
SELECT * FROM students WHERE score >= 80 OR gender = 'M';
SELECT * FROM students WHERE NOT class_id = 2;
SELECT * FROM students WHERE (score < 80 OR score > 90) AND gender = 'M';
SELECT * FROM students WHERE score BETWEEN 60 AND 90
-- AND,OR可用于关联多个条件，NOT表示非，BETWEEN表示在某某之间，还可以用括号对条件进行组合
```

|         条件         |   表达式举例1   |   表达式举例2    |                       说明                        |
| :------------------: | :-------------: | :--------------: | :-----------------------------------------------: |
|    使用=判断相等     |   score = 80    |   name = 'abc'   |             字符串需要用单引号括起来              |
|    使用>判断大于     |   score > 80    |   name > 'abc'   | 字符串比较根据ASCII码，中文字符比较根据数据库设置 |
| 使用>=判断大于或相等 |   score >= 80   |  name >= 'abc'   |                                                   |
|    使用<判断小于     |   score < 80    |  name <= 'abc'   |                                                   |
| 使用<=判断小于或相等 |   score <= 80   |  name <= 'abc'   |                                                   |
|   使用<>判断不相等   |   score <> 80   |  name <> 'abc'   |                                                   |
|   使用LIKE判断相似   | name LIKE 'ab%' | name LIKE '%bc%' | %表示任意字符，例如'ab%'将匹配'ab'，'abc'，'abcd' |

### 投影查询

```sql
SELECT * FROM students
-- *表示查询表的所有列
SELECT id, score, name FROM students;
-- 返回指定列
SELECT id, score points, name FROM students;
-- 将score列重命名为points
```

### 排序

```sql
SELECT id, name, gender, score FROM students ORDER BY score;
-- 加上ORDER BY，将查询结果按score从低到高排序（升序）

SELECT id, name, gender, score FROM students ORDER BY score DESC;
-- 加上DESC，实现降序（从高到低排序），ASC表示升序，可以省略（一般默认升序）

SELECT id, name, gender, score FROM students ORDER BY score DESC, id;
-- 先对score降序排序，若score相同，再对id升序排序

SELECT id, name, gender, score
FROM students
WHERE class_id = 1
ORDER BY score DESC;
-- 带有WHERE的查询，ORDER BY要放在WHERE后面

```

### 分页查询

使用SELECT查询时，如果结果集数据量很大，比如几万行数据，放在一个页面显示的话数据量太大，此时可用分页显示，比如每次显示100条。

```sql
SELECT id, name, gender, score
FROM students
ORDER BY score DESC
LIMIT 3 OFFSET 0;
-- 每页3条记录，获取第一页的记录
-- LIMIT N 表示每页最多显示N条记录，OFFSET M表示从第几个数据开始显示，0为第一个数据

-- LIMIT 3等价于 LIMIT 3 OFFSET 0
-- LIMIT 15 OFFSET 30还可以简写成LIMIT 30, 15

```

分页查询的关键在于，首先要确定每页需要显示的结果数量`pageSize`（这里是3），然后根据当前页的索引`pageIndex`（从1开始），确定`LIMIT`和`OFFSET`应该设定的值：

- `LIMIT`总是设定为`pageSize`；
- `OFFSET`计算公式为`pageSize * (pageIndex - 1)`。

> 使用`LIMIT <M> OFFSET <N>`分页时，随着`N`越来越大，查询效率也会越来越低。

### 聚合查询

```sql
-- 查询表中一共有多少条记录
SELECT COUNT(*) num FROM students;
-- 聚合查询也可使用条件
SELECT COUNT(*) boys FROM students WHERE gender = 'M';
```

#### 聚合函数

| 函数 | 说明                                   |
| ---- | -------------------------------------- |
| SUM  | 计算某一列的合计值，该列必须为数值类型 |
| AVG  | 计算某一列的平均值，该列必须为数值类型 |
| MAX  | 计算某一列的最大值                     |
| MIN  | 计算某一列的最小值                     |

> 如果聚合查询的`WHERE`没有匹配到任何行，则`COUNT()`会返回0，而`SUM()`、`AVG()`、`MAX()`和`MIN()`会返回`NULL`

#### 分组

聚合查询只返回一个值，如果想同时查询不同类别的聚合，可以使用`GROUP BY`

```sql
SELECT class_id, COUNT(*) num 
FROM students 
GROUP BY class_id;
-- 按class_id分组，输出各组的记录数
```

> 聚合查询的列中，只能放入分组的列。如果在任一分组中，某列的内容有不同，会报错。
>
> **`GROUP BY`也必须放在`WHERE`之后**

```sql
SELECT class_id, gender, COUNT(*) num 
FROM students 
GROUP BY class_id, gender;
-- 按class_id, gender分组
```

### 多表查询

查询多张表的语法是：`SELECT * FROM <表1> <表2>`

```sql
SELECT * FROM students, classes;
```

> 这种一次查询两个表的数据，查询的结果也是一个二维表，它是`students`表和`classes`表的“乘积”，即`students`表的每一行与`classes`表的每一行都两两拼在一起返回。结果集的列数是`students`表和`classes`表的列数之和，行数是`students`表和`classes`表的**行数之积**。
>
> 这种多表查询又称笛卡尔查询，使用笛卡尔查询时要非常小心，由于结果集是目标表的行数乘积，对两个各自有100行记录的表进行笛卡尔查询将返回1万条记录，对两个各自有1万行记录的表进行笛卡尔查询将返回1亿条记录。

若两个表有相同的列名，一般使用投影为各字段设置别名，避免歧义

```sql
SELECT
    students.id sid, -- 为列名设置别名
    students.name,
    students.gender,
    students.score,
    classes.id cid,
    classes.name cname -- 为列名设置别名
FROM students, classes;
```

为方面使用，还可为表名设置别名，以简化代码

```sql
SELECT
    s.id sid,
    s.name,
    s.gender,
    s.score,
    c.id cid,
    c.name cname
FROM students s, classes c; -- 为表名设置别名
```

多表查询依旧可以搭配`WHERE`使用。但用处不大，直接跳到下一节，连接查询。

### 连接查询

#### 内连接（常用）

```sql
SELECT s.id, s.name, s.class_id, c.name class_name, s.gender, s.score
FROM students s -- students是主表
INNER JOIN classes c --需要连接的表
ON s.class_id = c.id; --条件
```

INNER JOIN只返回**同时存在于两张表**的行数据。

#### 外连接

- 外连接分为LEFT OUTER JOIN、RIGHT OUTER JOIN和FULL OUTER JOIN（左外、右外和全外）。
- 左外连接返回**左表（主表）都存在**的行，若某一行仅在左表存在，则以`NULL`填充剩下的字段。
- 同理，**右外连接返回右表都存在的行**，若某一行仅在右表存在，则以`NULL`填充剩下的字段。
- FULL OUTER JOIN会把两张表的所有记录全部选择出来，自动把对方不存在的列填充为`NULL`。

#### 连接查询语法

```sql
SELECT ... 
FROM <表1> 
JOIN <表2> 
ON <条件...>
WHERE <条件...>
ORDER BY <列>
```



## 修改相关（增删改）

### INSERT

```sql
INSERT INTO <表名> (字段1, 字段2, ...) VALUES (值1, 值2, ...);

-- 一次性添加多条记录
INSERT INTO students (class_id, name, gender, score) VALUES
  (1, '大宝', 'M', 87),
  (2, '二宝', 'M', 81);
```

- 没有列出的字段将使用其默认值，如自增主键
- 字段排序不必和表中字段顺序相同，但必须和值得顺序一致

### UPDATE

```sql
UPDATE <表名> SET 字段1=值1, 字段2=值2, ... WHERE ...;
-- SET后为修改的内容，WHERE为选择的条件

UPDATE students SET score=score+10 WHERE score<80;
-- 也可用表达式

UPDATE students SET score=100 WHERE id=999;
--当WHERE没有匹配到任何记录时，不会报错，即没有数据被更新
```

- 在使用`UPDATE`前，最好用`SELECT`看一下是否筛选出了期望的记录集

### DELETE

```sql
DELETE FROM <表名> WHERE ...;
```

- 同`UPDATE`，在使用`DELETE`前，最好用`SELECT`看一下是否筛选出了期望的记录集
- 如果运行`DELETE FROM <表名>`，则整个表的所有记录都会被删除

## 其他命令

#### 库表相关

- 创建索引

  ```mysql
  ALTER TABLE students
  ADD INDEX idx_score (score);
  
  ALTER TABLE students
  ADD INDEX idx_name_score (name, score);-- 索引可以包含多列
  ```

- 列出所有数据库

  ```mysql
  SHOW DATABASES;
  ```

- 创建新的数据库

  ```mysql
  CREATE DATABASE test;
  ```

- 删除数据库

  ```mysql
  DROP DATABASE test;
  ```

- 切换数据库

  ```mysql
  USE test;
  ```

- 列出当前数据库的所有表

  ```mysql
  SHOW TABLES;
  ```

- 查看一个表的结构

  ```mysql
  DESC students;
  ```

- 查看创建表的SQL语句

  ```mysql
  mysql> SHOW CREATE TABLE students;
  +----------+-------------------------------------------------------+
  | students | CREATE TABLE `students` (                             |
  |          |   `id` bigint(20) NOT NULL AUTO_INCREMENT,            |
  |          |   `class_id` bigint(20) NOT NULL,                     |
  |          |   `name` varchar(100) NOT NULL,                       |
  |          |   `gender` varchar(1) NOT NULL,                       |
  |          |   `score` int(11) NOT NULL,                           |
  |          |   PRIMARY KEY (`id`)                                  |
  |          | ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 |
  +----------+-------------------------------------------------------+
  1 row in set (0.00 sec)
  ```

- 创建表使用`CREATE TABLE`，删除表使用`DROP TABLE`

  ```mysql
  DROP TABLE students;
  ```

- 给表增加一列（给students表增加birth列，类型为`VARCHAR(10)`，不能为空

  ```mysql
  ALTER TABLE students ADD COLUMN birth VARCHAR(10) NOT NULL;
  ```

- 修改某一列（将birth列改名为birthday，类型改为`VARCHAR(20)`

  ```mysql
  ALTER TABLE students CHANGE COLUMN birth birthday VARCHAR(20) NOT NULL;
  ```

- 删除列

  ```mysql
  ALTER TABLE students DROP COLUMN birthday;
  ```

- 退出MySQL

  ```mysql
  EXIT
  ```

#### CRUD相关

- **插入或替换**，插入一条新记录（INSERT），但如果记录已经存在，就先删除原记录，再插入新记录。

  ```mysql
  REPLACE INTO students (id, class_id, name, gender, score) VALUES (1, 1, '小明', 'F', 99);
  -- 若主键id=1已经存在，则先删除原数据再插入新数据，否则直接插入
  ```

- **插入或更新**，插入一条新记录（INSERT），但如果记录已经存在，就更新该记录

  ```mysql
  INSERT INTO students (id, class_id, name, gender, score) 
  VALUES (1, 1, '小明', 'F', 99)  -- 若主键不存在则直接插入
  ON DUPLICATE KEY UPDATE name='小明', gender='F', score=99;
  -- 若主键存在，则更新UPDATE后的内容
  ```

- **插入或忽略**，插入一条新记录（INSERT），但如果记录已经存在，就啥事也不干直接忽略

  ```mysql
  INSERT IGNORE INTO students (id, class_id, name, gender, score) 
  VALUES (1, 1, '小明', 'F', 99);
  ```

#### 快照

如果想要对一个表进行快照，即复制一份当前表的数据到一个新表，可以结合`CREATE TABLE`和`SELECT`

```mysql
-- 对class_id=1的记录进行快照，并存储为新表students_of_class1:
CREATE TABLE students_of_class1 SELECT * FROM students WHERE class_id=1;
-- 新表结构与老表结构一致
```

#### 写入查询结果集

如果查询结果集需要写入到表中，可以结合`INSERT`和`SELECT`，将`SELECT`语句的结果集直接插入到指定表中。

```mysql
-- 创建统计成绩的表，用于记录各班平均成绩
CREATE TABLE statistics (
    id BIGINT NOT NULL AUTO_INCREMENT,
    class_id BIGINT NOT NULL,
    average DOUBLE NOT NULL,
    PRIMARY KEY (id)
);

INSERT INTO statistics (class_id, average) 
SELECT class_id, AVG(score) -- 确保INSERT的列和SELECT的列一一对应
FROM students 
GROUP BY class_id;
```

### 强制使用指定索引

在查询的时候，数据库系统会自动分析查询语句，并选择一个最合适的索引。但是很多时候，数据库系统的查询优化器并不一定总是能使用最优索引。如果我们知道如何选择索引，可以使用`FORCE INDEX`强制查询使用指定的索引。

```mysql
SELECT * FROM students 
FORCE INDEX (idx_class_id) -- 索引idx_class_id必须存在
WHERE class_id = 1 
ORDER BY id DESC;
```

## 事务

事务特性（ACID）：

- A：Atomic，原子性，将所有SQL作为原子工作单元执行，要么全部执行，要么全部不执行；
- C：Consistent，一致性，事务完成后，所有数据的状态都是一致的，即A账户只要减去了100，B账户则必定加上了100；
- I：Isolation，隔离性，如果有多个事务并发执行，每个事务作出的修改必须与其他事务隔离；
- D：Duration，持久性，即事务完成后，对数据库数据的修改被持久化存储。

对于单条SQL语句，数据库系统自动将其作为一个事务执行，这种事务被称为**隐式事务**。

要手动把多条SQL语句作为一个事务执行，使用`BEGIN`开启一个事务，使用`COMMIT`提交一个事务，这种事务被称为*显式事务*，例如，把上述的转账操作作为一个显式事务：

```mysql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

有些时候，我们希望主动让事务失败，这时，可以用`ROLLBACK`回滚事务，整个事务会失败：

```mysql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
ROLLBACK;
```

## 隔离级别

对于两个并发执行的事务，如果涉及到操作同一条记录的时候，可能会发生问题。因为并发操作会带来数据的不一致性，包括脏读、不可重复读、幻读等。数据库系统提供了隔离级别来让我们有针对性地选择事务的隔离级别，避免数据不一致的问题。

SQL标准定义了4种隔离级别，分别对应可能出现的数据不一致的情况：

| Isolation Level  | 脏读（Dirty Read） | 不可重复读（Non Repeatable Read） | 幻读（Phantom Read） |
| ---------------- | ------------------ | --------------------------------- | -------------------- |
| Read Uncommitted | Yes                | Yes                               | Yes                  |
| Read Committed   | -                  | Yes                               | Yes                  |
| Repeatable Read  | -                  | -                                 | Yes                  |
| Serializable     | -                  | -                                 | -                    |

#### Read Uncommitted

在这种隔离级别下，一个事务会读到另一个事务更新后但未提交的数据，如果另一个事务回滚，那么当前事务读到的数据就是脏数据，这就是脏读（Dirty Read）。如下图所示，4时刻和6时刻事务B将读到不同的数据。

| 时刻 | 事务A                                             | 事务B                                             |
| ---- | ------------------------------------------------- | ------------------------------------------------- |
| 1    | SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED; | SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED; |
| 2    | BEGIN;                                            | BEGIN;                                            |
| 3    | UPDATE students SET name = 'Bob' WHERE id = 1;    |                                                   |
| 4    |                                                   | SELECT * FROM students WHERE id = 1;              |
| 5    | ROLLBACK;                                         |                                                   |
| 6    |                                                   | SELECT * FROM students WHERE id = 1;              |
| 7    |                                                   | COMMIT;                                           |

#### Read committed

在Read Committed隔离级别下，一个事务可能会遇到不可重复读（Non Repeatable Read）的问题。

不可重复读是指，在一个事务内，<u>多次读同一数据</u>，在这个事务还没有结束时，如果另一个事务恰好修改了这个数据，那么，在第一个事务中，**两次读取的数据就可能不一致**。如下图所示，时刻3和时刻5读到的数据相同，因为事务A还没有提交，但时刻7读到的数据将会不同。

| 时刻 | 事务A                                           | 事务B                                           |
| ---- | ----------------------------------------------- | ----------------------------------------------- |
| 1    | SET TRANSACTION ISOLATION LEVEL READ COMMITTED; | SET TRANSACTION ISOLATION LEVEL READ COMMITTED; |
| 2    | BEGIN;                                          | BEGIN;                                          |
| 3    |                                                 | SELECT * FROM students WHERE id = 1;            |
| 4    | UPDATE students SET name = 'Bob' WHERE id = 1;  |                                                 |
| 5    |                                                 | SELECT * FROM students WHERE id = 1;            |
| 6    | COMMIT;                                         |                                                 |
| 7    |                                                 | SELECT * FROM students WHERE id = 1;            |
| 8    |                                                 | COMMIT;                                         |

#### Repeatable Read

在Repeatable Read隔离级别下，一个事务可能会遇到幻读（Phantom Read）的问题。

幻读是指，在一个事务中，第一次查询某条记录，发现没有，但是，当试图更新这条不存在的记录时，竟然能成功，并且，再次读取同一条记录，它就神奇地出现了。如下图所示，事务A插入了一条数据，但是时刻3和时刻6事务B都不能读到该数据，但却能**成功更新**这条数据，并且在**时刻8能够查到**这条数据，即幻读。

> 幻读就是没有读到的记录，以为不存在，但其实是可以更新成功的，并且，更新成功后，再次读取，就出现了。

| 时刻 | 事务A                                               | 事务B                                             |
| ---- | --------------------------------------------------- | ------------------------------------------------- |
| 1    | SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;    | SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;  |
| 2    | BEGIN;                                              | BEGIN;                                            |
| 3    |                                                     | SELECT * FROM students WHERE id = 99;             |
| 4    | INSERT INTO students (id, name) VALUES (99, 'Bob'); |                                                   |
| 5    | COMMIT;                                             |                                                   |
| 6    |                                                     | SELECT * FROM students WHERE id = 99;             |
| 7    |                                                     | UPDATE students SET name = 'Alice' WHERE id = 99; |
| 8    |                                                     | SELECT * FROM students WHERE id = 99;             |
| 9    |                                                     | COMMIT;                                           |

#### Serializable

Serializable是最严格的隔离级别。在Serializable隔离级别下，所有事务按照次序依次执行，因此，脏读、不可重复读、幻读都不会出现。

虽然Serializable隔离级别下的事务具有最高的安全性，但是，由于事务是串行执行，所以效率会大大下降，应用程序的性能会急剧降低。如果没有特别重要的情景，一般都不会使用Serializable隔离级别。

#### 默认隔离级别

如果没有指定隔离级别，数据库就会使用默认的隔离级别。在MySQL中，如果使用InnoDB，默认的隔离级别是Repeatable Read。

## InnoDB

需了解

<!--本文大部分内容搬运自廖雪峰的博客-->