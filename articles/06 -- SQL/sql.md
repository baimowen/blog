# SQL

[^tag]: sql



# 基本概念



## 主键(PK)



**唯一且非空**，可通过 `SERIAL` 或 `BIGSERIAL` 配合主键生成自增主键



```sql
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);  -- 设置主键方式一

CREATE TABLE association (
    student_id INT,
    course_id INT,
    PRIMARY KEY (student_id, course_id)
);  -- 设置主键方式二
```



## 外键(FK)



建立**表与表**之间的联系



```sql
CREATE TABLE association (
    student_id INT REFERENCES students(id),
    course_id INT REFERENCES courses(id) 
);
```



可以通过 `ON DELETE/UPDATE CASCADE` 设置**级联**操作确保数据一致性



```sql
CREATE TABLE association (
    student_id INT REFERENCES students(id) ON DELETE CASCADE,
    course_id INT REFERENCES courses(id)
);
```



**级联**：父表进行删除/更新操作时，子表自动进行相应操作。保证数据一致性

*   `ON DELETE CASCADE`：删除父表记录，自动删除相关子表记录
*   `ON UPDATE CASCADE`：更新父表主键，自动更新相关子表外键
*   `ON DELETE SET NULL`：删除父表记录，将相关子表外键的值设为NULL
*   `ON DELETE RESTRIC`：若子表中存在相关记录则不允许删除父表记录



外键方向的记忆技巧：

-   **"子表指向父表"**：外键总是在<u>子表(association)中定义</u>，并指向<u>父表(students/courses)的主键</u>。**总是由子表的普通字段指向父表的主键/唯一键字段**

    >   student(id)不存在association(student_id)一定不存在；student(id)存在association(student_id)不一定存在

-   **"箭头方向"**：在关系图中，箭头从外键列指向被引用的主键列

-   **"依赖关系"**：association表依赖于students和courses表的存在



## 索引



**提高查询效率**，略微影响插入和更新性能



```sql
CREATE INDEX idx_name ON student(name)
```

>   可以在索引中添加约束，如：`UNIQUE`、`NOT NULL`、`DEFAULT`、`CHECK`



## 视图



**保存查询定义**，每次访问视图存在查询开销。虚拟表不储存实际数据，访问时执行定义的查询



```sql
CREATE VIEW active_students AS
SELECT student_id
FROM association
WHERE course_id = '';
```



## sql操作



### 对数据库



#### 创建



```sql
CREATE DATABASE dbname;
```



#### 查看



```SQL
\l;  -- psql
SELECT dataname FROM pg_database;  -- SQL
SELECT * FROM pg_database WHERE dataname = 'dbname';  -- 查看特定数据库详情
```



#### 连接



```sql
\c dbname;  -- psql
```



#### 重命名



```sql
ALTER DATABASE oldname RENAME TO newname;
```



#### 备份/恢复



```shell
# 导出数据库结构(不含数据)
pg_dump -U username -d dbname -s -f structure.sql

# 导出完整数据库
pg_dump -U username -d dbname -Fc -f backup.dump

# 从备份恢复
pg_restore -U username -d newdb backup.dump
```



#### 其他



```sql
-- 重命名数据库
ALTER DATABASE dbname RENAME TO newname;

-- 修改所有者
ALTER DATABASE dbname OWNER TO new_owner;

-- 修改连接限制
ALTER DATABASE dbname WITH CONNECTION LIMIT 50;

-- 修改默认配置参数
ALTER DATABASE dbname SET config_parameter = value;

-- 将数据库设为模板
UPDATE pg_database SET datistemplate = true WHERE datname = 'template_db';

-- 使用模板创建数据库
CREATE DATABASE newdb TEMPLATE template_db;
```



### 对数据表



#### 创建



```sql
CREATE TYPE gender_type ENUM('male', 'female');  -- 创建枚举类型
CREATE TABLE tbname (
    id INT NOT NULL PRIMARY KEY,
    name VARCHAR NOT NULL,
    gender gender_type
);  -- 创建普通表

CREATE TABLE (
    id SERIAL PRIMARY KEY,  -- 创建自增主键
    subject_id INT,
    score INT
);

CREATE TABLE child_table (
    child_column datatype
) INHERITS (parent_table);  -- 创建子表继承父表
```



#### 查看



```sql
\dt;  -- 列出当前数据库下所有数据表
\d tbname;  -- 查看表结构
\d+ tbname;  -- 更详细的信息
```



#### 修改表结构



```sql
ALTER TABLE tbname ADD COLUMN column_name datatype [constraints];  -- 添加字段
ALTER TABLE tbname DROP COLUMN column_name [CASCADE];  -- 删除字段
ALTER TABLE tbname ALTER COLUMN column_name TYPE new_datatype;  -- 修改字段类型
ALTER TABLE tbname RENAME COLUMN old_name TO new_name;  -- 重命名字段

ALTER TABLE tbname ADD CONSTRAINT constraint_name PRIMARY KEY (column);  -- 添加主键约束
ALTER TABLE tbname ADD CONSTRAINT constraint_name UNIQUE (column);  -- 添加唯一约束
ALTER TABLE tbname ADD CONSTRAINT constraint_name CHECK (condition);  -- 添加check约束
ALTER TABLE tbname ADD CONSTRAINT constraint_name FOREIGN KEY (column) REFERENCES other_table(column);  -- 添加外键

ALTER TABLE tbname RENAME TO new_tbname;  -- 重命名表
```



#### 删除



```sql
DROP TABLE tbname;
```



### CRUD操作



CRUD即数据库中基本的四种操作：增删改查

*   C：**创建**  -- INSERT
*   R：**读取/查询**  -- SELECT
*   U：**更新/修改**  -- UPDATE
*   D：**删除**  -- DELETE



#### INSERT



```sql
-- 指定插入字段顺序 
INSERT INTO
    tbname (name, id)
VALUES
    ('张三', 1), 
    ('李四', 2);
-- 默认字段顺序
INSERT INTO tbname VALUES (), ();
-- 计算插值
INSERT INTO tbname VALUES ((SELECT MAX(id) FROM tbname)+1,'王五')
```



#### UPDATE



```sql
-- 更新单个字段
UPDATE tbname
set name = '赵六'
where name = '王五';  -- 不使用where指定会更新所有记录
-- 更新多个字段
UPDATE tbname
set id = , name = ''
FROM tbname_x/(SELECT ... FROM ...)
WHERE name = '';
```



#### DELETE



```sql
DELETE FROM tbname WHERE ...;  -- 删除符合条件的记录
```



>   [!warning]
>
>   `DELETE` 不会立即释放空间



>   [!note]
>
>   可以在 `DELETE` 后使用 `OPTIMIZE/ANALYZE TABLE tbname` 来释放空间



#### SELECT



获取数据的**基本操作**



##### BASIC



```sql
SELECT * FROM tbname;  -- 查找所有数据
SELECT id, name FROM tbname;  -- 查找指定字段的数据
```



##### WHERE



条件**筛选**



```sql
SELECT id, name FROM tbname WHERE id in (1,3);
```

>   [!note]
>
>   SQL中的“不等于”可以使用 `<>` 或 `!=`



>   [!tip]
>
>   [HAVING](####HAVING)



##### AND OR NOT



基本逻辑关系符：`AND`、`OR`、`NOT`



```sql
SELECT
    id, name 
FROM
    dbname 
WHERE
    id > 0 AND id != 3;
```



##### LIKE



基于**匹配**的条件筛选



```sql
SELECT * FROM db WHERE name LIKE '%张三%';
```



>   [!note]
>
>   ```
>   LIKE` 匹配对大小写敏感，若忽略大小写需使用 `ILIKE
>   ```
>
>   `%` 匹配任意字符串，`_` 匹配单个字符
>
>   可以使用 `~` 匹配**正则表达式**



##### CASE



依据条件**分类**进行筛选



```sql SELECT name
CASE
    WHEN score < 60 THEN
        'unpass'
    ELSE
        'pass'
    END AS situation
FROM dbname.score;
```



>   [!note]
>
>   ```
>   CASE` 支持多个 `WHEN`，并且不一定需要 `ELSE
>   ```
>
>   对于未覆盖的情况会返回 `NULL`



##### OPERATION ON DATE



SQL中 `timestamp` 时间戳固定格式为 `YYYY-MM-DD HH:MM:SS.nnnnnn`

>   [!tip]
>
>   时间戳可以进行计算



```sql
WHERE data >= '2025-08-18 17:38:00';  -- 比较
SELECT 
TIMESTAMP '2012-08-31 02：00：00' - TIMESTAMP '2012-07-30 01:00:00'
AS interval;  -- 时间相减得到时间间隔
```



>   `DATA` 相应类型具有许多函数如 `EXTRACT`、`DATE_TRUNC` 等



##### DISTINCT



**去重**



```sql
SELECT DISTINCT name FROM dbname.tbname
```

从 *tbname* 表中选取 *name* 列去重后的结果



##### ORDER BY & LIMIT



**排序**和**限制**输出



```sql
SELECT id,name,score FROM dbname.score
ORDER BY score DESC  -- 根据 score 列排序
LIMIT 10  -- 限制输出十条记录
```



##### GROUP BY



**分组**



```sql
SELECT
    sc.id AS stu_id, tb.name AS stu_name, SUM(sc.score) AS total_score
FROM
    tbname tb JOIN score sc
    ON sc.id = tb.id  
GROUP BY stu_id
ORDER BY total_score DESC;
```



### JOIN



将 `JOIN` 连接的**两张表**的记录逐一取出，依照 `ON` 的条件进行比对，**满足条件**的放入一张新的临时表

>   [!tip]
>
>   可以通过 `AS` 为表取别名



#### INNER JOIN



**内**连接



**同时存在**两张表内的数据（**交集**：A∩B）



```sql
SELECT *
FROM
    dbname.tbname tb
    INNER JOIN dbname.score sc
        ON tb.id = sc.id
WHERE tb.name = '张三';
```



>   [!tip]
>
>   隐式内连接：
>
>   ```sql
>   SELECT *
>   FROM
>      dbname.tbname tb
>      dbname.score sc
>   WHERE tb.id = sc.id
>   ```



#### LEFT JOIN



**左**连接



取 左表全部数据 和 右表中包含在左表的数据



#### RIGHT JOIN



**右**连接



取 右表全部数据 和 左表中包含在右表的数据



#### UNION



**全**连接



**合并**查询结果（即取两表中全部数据）



```sql
SELECT * FROM dbname.tbname UNION SELECT * FROM dbname.score
```



### Aggregates



**聚合函数**



1.   `COUNT([DISTINCT] column_name)`：计数
2.   `SUM([DISTINCT] column_name)`：求和
3.   `AVG()`：平均值
4.   `MIN/MAX()`：最小/最大值
5.   `STDDEV()`：标准差
6.   `VARIANCE()`方差
7.   ...



#### HAVING



>   [!note]
>
>   PostgresSQL `WHERE` 中不可以使用聚合函数，但是可以使用 `HAVING` 达成相似目的



```sql
-- 基本用法
SELECT column1, aggregate_function(column2)
FROM table_name
GROUP BY column1
HAVING condition;

-- 实例：筛选总分大于300的学生
SELECT 
    id AS student_id,
    name AS student_name,
    SUM(score) AS total_score
FROM scores
GROUP BY id, name
HAVING SUM(score) > 300  -- 或使用 `HAVING total_score`
ORDER BY total_score DESC;

-- WHERE先筛选原始数据，HAVING再筛选分组结果
SELECT 
    id AS student_id,
    name AS student_name,
    SUM(score) AS total_score,
    AVG(score) AS average_score
FROM scores
WHERE score >= 60  -- 只计算及格分数
GROUP BY id, name
HAVING AVG(score) > 75  -- 平均分大于75
ORDER BY average_score DESC;
```



>   [!note]
>
>   `WHERE`：在**分组前**对<u>原始数据</u>进行筛选
>
>   `HAVING`：在**分组后**对<u>聚合结果</u>进行筛选



# SQL注入



在**输入字段中插入 SQL 语句**



实例：

假设存在一个登录功能其 SQL 查询如下：

```sql
SELECT * FROM users WHERE username = 'user' AND passwd = 'pass';
```

若用户输入 `username` 为 `' OR ' '1' = '1`，则生成的 SQL 查询变为：

```sql
SELECT * FROM users WHERE username = '' OR '1' = '1' AND passwd = 'pass';
```

因为 `'1' = '1'` 为真，查询会绕过验证导致攻击者可以直接登录系统



>   [!note]
>
>   **预防措施**：
>
>   *   使用 **ORM（对象关系映射）**
>   *   **加密**数据，使用 **hash** 等
>   *   对输入进行**转义**/使用**正则表达式**进行匹配
>   *   **参数化** SQL



# MORE ABOUT...

CTEs、Window Function、ORM
