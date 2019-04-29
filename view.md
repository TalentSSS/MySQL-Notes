# 4 视图和存储程序

**视图**(*view*)，一种类型的存储对象。是一个虚拟表，提供了一种简单的运行复杂查询的方式。

**存储程序**(*stored program*)，一种类型的存储对象。形式多样，有些可以按需调用，而其他则可以修改表或达到预定时间时自动执行。

* 存储函数(*stored function*)

    在表达式里返回某个计算结果

* 存储过程(*stored procedure*)

    不直接返回结果，可以用来完成一般的运算，生成可以传递到客户端的结果集

* 触发器(*trigger*)

    与表关联，在使用INSERT、DELETE或UPDATE语句进行修改时自动执行

* 事件(*event*)

    根据计划在预定时刻自动执行

## 4.1 使用视图

视图是一个虚表，是在表或者其他视图的基础上，使用SELECT语句来定义的。查询视图就等效于查询定义它的那条语句。

定义一个视图，只检索想要的列

```sql
CREATE VIEW vpres AS
SELECT last_name, first_name, city, state FROM president;
```

注意创建视图时，**必须拥有对视图的CREATE VIEW权限，拥有对SELECT语句所选列的操作权限**。

通过视图只能看到你想看到的那些列。视图也可以像表那样，在查询时使用WHERE、ORDER BY、LIMIT等子句。

```mysql
mysql> SELECT * FROM vpres WHERE last_name = 'Adams';
```

默认情况下视图里的列与其SELECT语句里列出的输出列名相同。也可以使用别名

```mysql
mysql> CREATE VIEW vpres2 (ln, fn) AS
    -> SELECT last_name, first_name, FROM president;
```

使用别名后引用视图，就必须使用括号里给出的列名。

视图可以隐藏复杂的SELECT语句，从而轻松地查询到重要的输出信息

```MySQL
CREATE VIEW grade_stats AS
SELECT
grade_event.date, grade_event.category,
MIN(score.score) AS minimum,
MAX(score.score) AS maximum,
MAX(score.score)-MIN(score.score)+1 AS span,
SUM(score.score) AS total,
AVG(score.score) AS average,
COUNT(score.score) AS count,
FROM score INNER JOIN grade_event
ON score.event_id = grade_event.event_id
GROUP BY grade_event.date;
```

通过这个视图进行查询会执行连接操作，并检索计算的结果会变得很简单

```MySQL
mysql> SELECT * FROM grade_stats;
```





