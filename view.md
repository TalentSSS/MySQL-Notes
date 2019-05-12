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

## 4.2 使用存储程序

存储程序

* 存储函数
* 存储过程
* 触发器
* 事件

在了解如何编写存储程序之前要会写复合语句

### 4.2.1 复合语句和语句分隔符

存储程序可以包含多条SQL语句，并且可以使用多种类型的构建形式，如局部变量、循环和嵌套块。这些特性都需要使用复合语句支持。

这种语句由一个BEGIN和END块构成，其中可以包含任意数量的语句。

```mysql
CREATE PROCEDURE greetings ()
BEGIN
	# 77 = 16 for username + 60 for hostname + 1 for '@'
	DECLARE user CHAR(77) CHARACTER SET utf8
	SET user = (SELECR CURRENT_USER());
	IF INSTR(user, '@') > 0 THEN
		SET user = SUBSTRING_INDEX(user, '@', 1);
	END IF
	IF user = '' THEN	# 匿名用户
		SET user = 'earthling';
	END IF
	SELECT CONCAT('Greetings, ', user, '!') AS greeting;
END;
```

对于复合语句，在块内的语句之间必须使用分号";"进行分隔。但由于";"同时又是客户端程序mysql的默认语句分隔符，因此在尝试使用mysql定义存储程序的时候会有冲突。

这可以使用delimeter命令重新定义mysql的默认语句分隔符，从而保证它不会出现在这个存储程序的定义里。在结束后再把结束符定义成分号。

```mysql
mysql> delimeter $
mysql> CREATE PROCEDURE show_times()
	-> BEGIN
	->   SELECT CURRENT_TIMESTAMP AS 'Local Time';
	->	 SELECT UTC_TIMESTAMP AS 'UTC TIME';
	-> END $
	-> delimeter ;
	-> CALL show_times();
```

复合语句不一定要用在复杂的存储程序里，不管程序体里有没有语句，都可以使用复合语句

```mysql
BEGIN
  DO SLEEP(1);
END;
...
CREATE PROCEDURE do_nothing()
BEGIN
END;
```

### 4.2.2 存储函数和存储过程

存储函数可以用CREATE FUNCTION创建，存储过程可以用CREATE PROCEDURE语句来创建

基本语法

```mysql
CREATE FUNCTION func_name([param_list])
  RETURNS type
  routine_stmt

CREATE PROCEDURE proc_name([param_list])
  routine_stmt
```

下面的实例给出了如何使用函数来进行一个子查询，从而确定有多少位总统出生于给定年份

```mysql
mysql> delimiter $
mysql> CREATE FUNCTION count_born_in_year(p_year INT)
	-> RETURNS INT
	-> READS SQL DATA
	-> BEGIN
	->   RETURN (SELECT COUNT(*) FROM president WHERE YEAR(birth) = p_year);
	-> END $
mysql> delimiter ;
```

这个函数有一条用来表明其返回值数据类型的RETURNS子句，以及一个用来计算那个值的函数体。**函数体至少要包含一条RETURN语句，用来向调用者返回一个值**。

可以像使用内建函数调用，直接输入参数调用

```mysql
mysql> SELECT count_born_in_year(1908);
```

存储过程与存储函数很相似，区别在于**存储过程没有返回值**

```mysql
mysql> delimiter $
mysql> CREATE PROCEDURE show_born_in_year(p_year INT)
	-> BEGIN
	->   SELECT first_name, last_name, birth, death
	->	 FROM president
	->	 WHERE YEAR(birth) = p_year;
mysql> delimiter ;
```

存储过程不能用在表达式里，只能通过CALL语句来调用

```mysql
mysql> CALL show_born_in_year(1908);
```

#### 4.2.2.1 存储函数和存储过程的权限

存储函数和存储过程属于某个数据库，要创建它们必须拥有数据库的CREATE ROUTINE权限。创建存储例程时，没有EXECUTE和ALTER ROUTINE权限。mysql服务器会做一些自动权限授予，以便可以执行或删除该例程。如果不想有这种自动化权限授予/撤销，那就可以将系统变量automatic_sp_privileges设为0。





 



