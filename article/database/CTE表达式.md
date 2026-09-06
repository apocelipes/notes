CTE（公用表表达式）可以简化代码或者用了简化处理层级结构。

CTE分为普通的和递归的。

普通的CTE：

```sql
WITH cte_name AS (
  select * from table1 where column1 > 100;
)
select * from cte_name;
```

普通CTE表达式可以像表或者子查询一样用在各处。这可以减少重复代码的书写。

每个CTE只对当前的SQL语句有效。可以创建多个CTE：

```sql
WITH A AS (),
     B AS (), ...
```

CTE不可以嵌套定义。一些数据库禁止在CTE中使用ORDER BY（比如MSSQL），这点和子查询很像。

数据库不一定会对CTE表达式产生优化（有时会缓存CTE的结果或者优化为JOIN），因此让代码更简洁才是重点。

递归CTE：

```sql
WITH RECURSIVE cte_name AS (
    -- 锚点查询（Anchor Member）：初始条件，找到“根节点”
    SELECT ...
    FROM table_name
    WHERE ...

    UNION ALL -- 必须用union all

    -- 递归查询（Recursive Member）：每次循环调用的条件，关联 CTE 自身
    SELECT ...
    FROM table_name
    JOIN cte_name ON ...
)
SELECT * FROM cte_name;
```

Recursive Member必须引用CTE表达式，且只能在FROM或JOIN中引用。

执行顺序上，Anchor Member先执行产生结果，接着一遍遍执行Recursive Member直到某次查询返回结果为空时停止。

例子，打印1到100的平方：

```sql
WITH RECURSIVE numbers AS (
    SELECT 1 AS pow, 1 AS id
    UNION ALL
    SELECT (id+1)*(id+1) AS pow, id+1 AS id FROM numbers WHERE id < 100
)
SELECT pow FROM numbers;
```

例子，处理层级结构：

```sql
CREATE TABLE employees(
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name VARCHAR(50),
  manager_id INT NOT NULL
);

WITH RECURSIVE company_tree AS (
    SELECT 
        id, 
        name, 
        manager_id, 
        1 AS level
    FROM employees
    -- manager_id为0是最上级
    WHERE manager_id = 0

    UNION ALL

    -- 递归查询：用员工表 e 去匹配上一轮的表 t，找到上级的直属下属，层级 + 1
    SELECT 
        e.id, 
        e.name, 
        e.manager_id, 
        t.level + 1
    FROM employees e
    INNER JOIN company_tree t ON e.manager_id = t.id
)
SELECT id, name, manager_id, level FROM company_tree;

WITH RECURSIVE path_tree AS (
    -- 锚点查询：初始化路径为当前姓名
    SELECT 
        id, 
        name, 
        manager_id, 
        CAST(name AS CHAR(200)) AS path
    FROM employees
    WHERE manager_id = 0

    UNION ALL

    -- 递归查询：通过 -> 拼接下一级姓名
    SELECT 
        e.id, 
        e.name, 
        e.manager_id, 
        CONCAT(t.path, ' -> ', e.name)
    FROM employees e
    INNER JOIN path_tree t ON e.manager_id = t.id
)
SELECT id, path FROM path_tree;
```

可以有多个锚点，它们之间用union、union all等连接，只有最后一个union all后的部分会被当做递归member。

一部分数据库不允许递归成员部分出现GROUP BY或INNER JOIN以外的连接，各个数据情况不同。

CTE创建的临时表是没有索引的，因此数据量较大时可能有性能回退。

PostgreSQL 12之后支持手动控制是否缓存CTE结果：

```sql
WITH A AS MATERIALIZED (), -- 缓存
     B AS NOT MATERIALIZED () -- 不缓存
```

通常不应该使用这些选项，应该交给优化器自动判断，否则可能需要使用更多资源或者影响优化器判断。
