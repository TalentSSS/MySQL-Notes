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
    -> SELECT last_name, first_name FROM president;
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

存储函数和存储过程隶属于某个数据库。如果要想创建、执行、删除存储函数或存储过程，需要拥有一些权限。

服务器会根据系统变量(log_bin_trust_function_creators)的设置来决定某些权限的授予和撤销。

如果服务器启动了二进制日志功能，存储函数还需要服从一些限制条件来保证二进制日志在执行备份和复制操作时的安全性。

#### 4.2.2.2 存储过程的参数类型

存储过程的参数分为3种类型。

* IN参数，指定输入参数。
* OUT参数，指定输出参数
* INOUT参数，允许调用者向过程传递一个值，然后再取回一个值

显式指定参数类型的方法是，在参数列表里的参数名前面使用IN、OUT或INOUT。如果没 有为参数指定类型，则其默认类型为IN。

使用OUT或INOUT参数的方法是，在调用过程时指定一个变量名。过程可以设置参数值， 相应的变量将在过程返回时获得那个值。如果想让某个存储过程返回多个结果值，可以用OUT和INOUT。下面这个过程演示了OUT参数的用法。它可以分别统计出stude卫表里的男生和女生人数

```mysql
CREATE PROCEDURE count_students_by_sex(OUT p_male INT, OUT p_female INT) 
BEGIN
SET p_male = (SELECT COUNT(*) FROM student WHERE sex='M');
SET p_female = (SELECT COUNT(*) FROM student WHERE sex='F');
END;
```

在调用此过程时，把各个参数替换成相应的用户定义变量。此过程将把统计值放到这些参数里。在它返回之后，这些变量会包含那些统计值：

```mysql
mysql> CALL count_students_by_sex(@male_count, @female_count); 
mysql> SELECT @male_count, @female_count;
```

如果在另一个存储程序里调用count_students_by_sex() , 那么在这个程序中定义的局部变量或参数可以作为参数传递给count_students_by_sex()。

关键字IN、OUT和INOUT都不能用于存储函数、触发器和事件。对于存储函数，所有参数 都像IN参数。

### 4.2.3 触发器

触发器是与特定表相关联的存储过程，其定义会在执行表的INSERT、DELETE或UPDATE 语句时，被自动激活。触发器可被设置成语句处理各行之前或之后激活。触发器的定义包含有一条会在触发器被激活时执行的语句。

触发器具有以下几个好处。

* 触发器可以检查或修改将被插入或用来更新行的那些新的数据值。可以利用触发器来实现数据完整性约束，用来对输入数据进行过滤。

* 触发器可以基于某个表达式来为列提供默认值，甚至可以为那些只能使用常量默认值进行定义的列类型提供值。

* 触发器可以在行删除或更新之前先检查行的当前内容。这种能力能完成许多任务，如记录已有行的更改情况。

使用CREATE TRIGGER语句创建触发器。

在触发器的定义里，需要指明：

* 触发它 的语句类型(INSERT、UPDATE或DELETE)；
* 以及是在行被修改之前触发，还是在之后触发。

触发器创建语句的基本语法如下所示：

```mysql
CREATE TRIGGER trigger_name				#触发器的名字
	{BEFORE | AFTER}					#触发器激活的时机
    {INSERT | UPDATE | DELETE}			#激活触发器的语句                  
	ON tbl_name							#关联表
	FOR EACH ROW trigger_stmt;			#触发器内容
```

其中，trigger_name是触发器的名字；tbl_name是与触发器相关联的那个表的名字。在给触发器命名时，最好在名字中体现触发器的用途和关联的表，如bi_tbl_name表示的是 一个与tbl_name表有关的BEFORE INSERT触发器；而ai_tbl_name表示的则是一个与 tbl_name表有关的AFTER_INSERT触发器。

trigger_stmt是触发器的语句体，即在触发器被激活时需要执行的语句。在触发器的语句体里，可以使用NEW.*col_name*语法来引用在INSERT或UPDATE触发器里将被插入或修改的那个新行里的列。类似地，OLD.*col_name*语法可以用来引用在DELETE或UPDATE触发器里将被删除或修改的原行里的列。如果想要用BEFORE触发器改变列值，而且想在值存储到表中之前改变它，那么可以使用SET NEW.*col_name* = value。

在下面这个示例里，表t的INSERT语句有一个名为bi_t的触发器。其中，表t有一个整 型列percent，用于存储百分比值（范围为0-100）；另外还有一个DATETIME列。该触发器使 用的是BEFORE, 因此它可以在数据值被插入进表之前对它们进行检查。

```mysql
mysql> CREATE TABLE t(percent INT, dt DATETIME);
mysql> delimiter $
mysql> CREATE TRIGGER bi_t BEFORE INSERT ON t
	->	 FOR EACH ROW BEGIN
	->	   IF NEW.percent< 0 THEN
	->	   	 SET NEW.percent= 0;
    ->     ELSEIF NEW.percent> 100 THEN
    ->       SET NEW.percent= 100;
    ->	   END IF;
   	->   NEW.dt = CURRENT_TIMESTAMP(); 
    ->   END $ 
mysql> delimiter ;
```

这个触发器将完成以下两个动作。

* 如果要插入的百分比值超出了0-100的范围，那么这个触发器将把该值转换成最靠近端点的那个值。
* 这个触发器将自动为那个DATETIME列提供一个CURRENT_TIMESTAMP值。

来看看这个触发器是如何工作的，先往表里插入几行，然后检索表的内容： 

```mysql
mysql> INSERT INTO t (percen七） VALUES(-2); DO SLEEP(2); 
mysql> INSERT INTO t (percent) VALUES(30); DO SLEEP(2);
mysql> INSERT INTO t (percent) VALUES(120);
mysql> SELECT* FROM t;
```

### 4.2.4 事件

MySQL有一个事件调度器，它可以定时激活多个数据库操作。事件就是一个与计划相关联的存储程序。计划会定义事件执行的时间或次数，并且还可以定义事件何时强行退出。事件非常适合于执行那些无人值守的系统管理任务，如汇总报告定期更新、旧数据过期清理或者日志表轮换等。

一个完成行过期处理操作的事件例子。

默认情况下，事件调度器并不会运行，因此必须先启用它才能使用事件。请把下面两行内 容放到某个在服务器启动时会被读取的选项文件里

```
[mysqld]
event_scheduler=ON
```

如果想在运行时查看邓件调度器的状态，那么可以使用下面这条语句： 

```mysql
SHOW VARIABLES LlKE'event_scheduler';
```

如果想在运行时停止或启动事件调度器，那么可以更改event_scheduler系统变量

```mysql
SET GLOBAL event_scheduler = OFF;
SET GLOBAL event_scheduler = ON;
```

如果你停止了调度器，所有事件都不会运行。也可以让事件调度器保持运行，但禁用各个事件。**如果在服务器启动时把变量event_scheduler设置成DISABLED, 那么在运行时只能查     看其状态，而不能更改它。你可以创建事件，但它们不会执行**。                                                         事件调度器会把执行情况写到服务器的错误日志里，通过它，你可以查行到小件调度器 在做什么。

创建事件的方法是，使用CREATE EVENT语句，其基本语法如下：

```mysql
CREATE EVENT evenc_name
	ON SCHEDULE
		{AT datetime I EVERY expr interval [STARTS datetime] [ENDS datetime]} DO event_stmt
```

事件属于数据库，因此你必须要拥有数据库的EVENT权限才能创建或删除其触发器。

如果想创建一个只执行一次的串件，则应该使用AT调度类型，而不要使用EVERY。创建一个在一个小时后只执行一次的事件：

```mysql
CREATE EVENT one_shot
	ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 HOUR 
	DO ..
```

如果要禁用某个事件，让它不再执行，或者要重新激活某个被禁用的事件，可以使用ALTER EVENT:

```mysql
ALTER EVENT evenc_name DISABLE;
ALTER EVENT evenc_name ENABLE;
```

## 4.3 视图和存储程序的安全性

在定义视图、存储程序时，需要定义一些以后执行的对象，如果以后执行这些操作的用户不是之前创建它们的用户，就需要考虑：服务器在执行时 应该使用什么样的安全上下文来检查访问权限呢？也就是说，应该应用哪个账户的权限呢？

默认情况下，服务器会使用定义该对象的那个用户的账户。假设我定义了一个存储过程p()，用于访问我的表。如果我把p()的EXECUTE权限授予你，那么你便可以使用CALL p()来调用该过程，进而访问到我的表，因为它是以我的权限来运行的。这种类型的安全上下文有利有弊： 

* 有利的一面

    精心编写的存储程序可以**把表开放给那些无法直接访问它们的用户**， 并可对访问进行控制

* 有弊的一面是：如果某个用户**创建了某个可以访问到敏感数据的存储程序**，但却忘记了其他可以调用该对象的人也会访问到那些敏感数据，则会造成**安全漏洞**。

在定义存储程序或视图时，显式指定定义者的方法是，在该对象的CREATE语句里包含一 个DEFINER= account子句。这样，这里列出的账户就能跟定义者完全一样，从而达到在执行时检查访问权限的目的。例如：

```mysql
CREATE DEFINER='sampdb'@'localhost'PROCEDURE count_students()
	SELECT COUNT(*) FROM student;
```

如果你拥有SUPER权限，则可以使用任何一种语法正确的账户名来作为DEFINER值。

如果你没有SUPER权限，则只能把定义者设置为你自己的账户。既可以使用完整的账户名，也可以使用CURRENT_USER。 

对于视图和存储例程（包括存储函数和存储过程），还可以使用SQL SECURITY特性，用于 实现对执行时访问检查的附加控制。SQL SECURITY特性的允许值为DEFINER (以定义者的权 限执行）或INVOKER (以对象调用者的权限执行）。 

适合使用SQL SECURITY INVOKER的场合是：只想以调用者所拥的权限来执行视图或存储程序。例如，下面这个视图将访问mysql数据库里的某个表，但是是以调用者的权限来运行。这样 一来，如果调用者本人无权访问mysql.user表，那么即使他访问这个视图，也无法突破权限限制。

```mysql
CREATE SQL SECURITY INVOKER VIEW v
	AS SELECT CONCAT(User,'@',Host) AS Account, Password FROM mysql.user;
```

服务器会自动调用触发器和事件，所以这里的"调用者"概念对它们并不适用。因此，它 们没有SQL SECURITY特性，总是以定义者的权限来执行。

如果某个视图或存储程序以定义者的权限来运行，但那个定义者账户并不存在，那么会出 现一个错误。

在视图或存储程序里，CURRENT_USER()函数会默认返回与该对象的DEFINER属性相对应的账户。对于视图和定义时带有SQL SECURITY INVOKER特性的存储例程，CURRENT_USER()会返回调用用户的账户。



