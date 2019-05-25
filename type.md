# 3 数据类型

MySQL里的每一个数值都拥有一个类型，37.4、'abc'

使用CREATE TABLE语句建表时，就必须为这个表里的每个列定义一种类型

```mysql
CREATE TABLE mytbl
{
	int_col		INT,		# 整数值列
	str_col		CHAR(20),	# 字符串值列
	date_col 	DATE		# 日期值列
};
```

看似普通的操作中几乎都隐藏着数据类型的概念

```mysql
INSERT INTO mytbl(int_col, str_col, date_col)
VALUES(14, CONCAT('a', 'b'), 20130815);
```

这条语隐含着丰富的数据类型信息

* 把整数`14`赋值给整型列`int_col`
* 把字符串`'a'`和`'b'`传递给字符串拼接函数`CONCAT()`，然后将返回的字符串值`'ab'`赋给字符串列`str_col`
* 把整数值`20130815`赋值给日期列`date_col`，这里出现了类型不匹配的问题，但整数值可以解释为日期值，MySQL会进行自动类型转换

有必要透彻了解MySQL是如何处理数据。

## 3.1 数据值类别

MySQL支持多种常规类别的数据值，数值、字符串值、日期/时间(时态值)、空间值、NULL值

### 3.1.1 数值

48、193.62、-2.378E12之类的值。

#### 3.1.1.1 精确值数和近似值数

MySQL支持对精确值数的精确计算，以及对近似值数的近似运算。

整数可以表示成十进制或者十六进制两种格式。十进制整数由一个不含小数点的数字序列构成。十进制值会被默认为字符串，但在数值计算的时候，十六进制常量会被视为64位整数。

小数部分的精确值由3个部分组成：一个数字序列、一个小数点和另一个数字序列。

近似值采用科学计数法表示的浮点数，有一个底数、一个指数，1.58E5、-1.58E5、1.58E-5、-1.58E-5。

**十六进制数不能用科学计数法**。

精确值计算结果也是精确的，只要没超出精度范围，就不会丢失精度。

近似值计算是近似的，会有舍入误差。

MySQL按照如下规则确定表达式使用精确计算还是近似计算

* 表达式有近似值，就被当成浮点数（近似值）算
* 有整数精确值，就用BIGINT(64位)精度计算
* 只包含精确值，但有小数部分，就以具有65位精度的DECIMAL算法来进行计算
* 如果必须将字符串转换为一个数才能算，那字符串会被转化为双精度浮点值。然后按照前面的规则做近似计算

#### 3.1.1.2 位域值(BIT值)

b'val'或0bval，val由一个或多个二进制(0或1)构成，b'1001'或0b1001表示十进制9。

BIT值会被显示为一个二进制串，不过输出不友好。

使用+0或CAST转化为一个整数

```mysql
mysql> SELECT b'1001' + 0, CAST(b'1001' AS UNSIGNED);
```

### 3.1.2 字符串值

'Madison, Wisconson'、'patient shows improvement'、'12345'

字符串两端可以是单引号也可以是双引号，尽量用单引号。

原因：

* SQL语言标准规定；能更好地移植到其他数据库引擎
* 如果用了ANSI_QUOTES模式，MySQL会将双引号处理成将标识符引起来的符号，而不会只当成字符串引起来的符号

转义字符

| 转义序列 |          含义          |
| :------: | :--------------------: |
|    \0    |     NUL(零值字节)      |
|   \\'    |         单引号         |
|   \\"    |         双引号         |
|    \b    |         退格符         |
|    \n    |         换行符         |
|    \r    |         回车符         |
|    \t    |         制表符         |
|   \\\    |         反斜线         |
|    \z    | Ctrl+Z(Windows中的EOF) |

嵌入引号的多种方法

* 用重复两次的方式嵌入引号

    ```
    'I can''t'
    "He said, ""I told you so."""
    ```

* 嵌入引号与字符串引号不同

    ```
    "I can't"
    'He said, "I told you so."'
    ```

* 对嵌入引号使用反斜线转义，不受字符串引号限制

    ```
    'I can\'t'
    "I can\'t"
    "He said, \"I told you so.\""
    'He said, \"I told you so.\"'
    ```

用双引号写字符串值，还可以用十六进制记法

* 标准的SQL记法X'val'

    val由多对十六进制数字(包括"0"~"9"和"a"~"f")组成。前导字符"X"和字符串里的十六进制数字字符("a"~"f")都不区分大小写

    ```mysql
    mysql> SELECT X'4A', x'4a';
    ```

    都输出字母`J`。

    字符串上下文中每两个十六进制数字都会被解释为一个8为数字字节值，范围是0~255。

    ```mysql
    mysql> SELECT X'61626364', X'61626364'+0;
    ```

* 以`"0x"`开头，后面跟一个或多个十六进制数字

    `x`只能是小写的，与X'val'记法相似，`0x`值会被默认解释为字符串

    ```mysql
    mysql> SELECT 0x61626364, 0x61626364+0;
    ```

#### 3.1.2.1 字符串类型与字符集支持

字符串值可以分为两类，即二进制串和非二进制串

* 二进制串是一组字节序列。对这些字节的解释不牵涉任何字符集概念，没有特殊的比较或排序属性。比较操作基于各字节的数值逐个字节实现，所有字节都有意义，包括结尾空格。
* 非二进制串，与字符集相关。字符集决定了：哪些字符能用、MySQL会如何解释字符串的内容。默认的字符集和排序规则为latin1何latin_swedish-ci。

字符单位在占用存储空间方面存在差异，即不同字符集单字符占有的字节空间不同。

用下面两条语句来看服务器上有哪些字符集，对应的排序规则是什么

```mysql
mysql> SHOW CHARACTER SET;
mysql> SHOW COLLATION;
```

从输出内容可以观察到，每种排序规则都捆绑在某个特定字符集上，每个特定字符集可以有多种排序规则。排序规则由**字符集名、语言名和一个附加的后缀构成**。

Utf8_iceland_ci，字符集utf8，iceland排序规则，case insensitive不区分大小写。

后缀的含义

* `_ci`不区分大小写
* `_cs`区分大小写
* `_bin`二进制排序规则，比较操作基于数字字符编码值进行

二进制串和非二进制串的不同排序特性

* 二进制串逐字节比较，结果取决于每个字节的数值大小
* 非二进制串按字符进行比较，相对打下取决于用的字符集排序规则

排序规则会对以下操作造成影响

* 比较运算：<、<=、=、< >、>=、>和LIKE
* 排序：ORDER BY、MIN()和MAX()。
* 分组：GROUP BY和DISTINCT

使用`CHARSET()`或`COLLATION()`来确定某个字符串的字符集合排序规则。

引号里的字符串会根据服务器的当前设置来解释。默认的字符集和排序规则分别为`latin1`和`latin1_swedish_ci`。

```mysql
mysql> SELECT CHARSET(x'0123'), COLLATION(X'0123');
```

有两种记法约定可以用于将某个字符串常量强制解释为某种指定的字符集

* _charset 记法，字符集引导符

    `_latin2 'abc'`

* N'str'

    等价于`_utf8'str'`，在N(不区分大小写)的后面必须紧跟一个引号形式的字符串，不能有任何空白

引导符记法可用于引号形式的字符串或十六进制常量，但不能用与字符串表达式或列值。可以用CONVERT()函数将任意字符串转换为另一种指定字符集形式的字符串

```mysql
CONVERT(str USING charset);
```

引导符与CONVERT()函数的不同

```mysql
mysql> SET @s1 = _ucs2 'ABCD';
mysql> SET @s2 = CONVERT('ABCD' USING ucs2);
```

假设当前字符集为`latin1`（单字节字符集）。第一条语句把字符串`'ABCD'`中的每一对字符解释为一个双字节的`ucs2`字符，结果为包含两个字符的`ucs2`字符。第二条语句把字符串`'ABCD'`中的每个字符转换为相应的`ucs2`字符，结果为包含4个字符的`ucs2`字符串。

长度有两种

* CHAR_LENGTH()函数，按照字符来测量
* LENGTH()函数，按照字节来测量

```mysql
mysql> SELECT CHAR_LENGTH(@s1), LENGTH(@s1), CHAR_LENGTH(@s2), LENGTH(@s2);
```

这里有一个非常容易弄错的问题，**二进制串与使用某种二进制排序规则的非二进制串**的区别

* 二进制串没有字符集的概念。它会被**解释为字节**，并且比较的是单字节的数字代码
* 使用了二进制排序规则的非二进制串，会被**解释成字符**，并且比较的是它们的数字字符值，这种值通常基于每个字符多个字节算出

展示二进制和非二进制串在大小写方面的区别，创建一个二进制串和一个使用了某种二进制排序规则的非二进制串，然后把它们传递给UPPER()函数

```mysql
mysql> SET @s1 = BINARY 'abcd';
mysql> SET @s2 = _latin1 'abcd' COLLATE latin1_bin;
mysql> SELECT UPPER(@s1), UPPER(@s2);
```

输出结果发现函数没有把二进制字符串转换大小写。因为二进制字符串没有字符集概念，就没有大小写概念。如果要传给UPPER()或LOWER()这样的需要指定字符集的函数，必须先转成非二进制串

```mysql
mysql> SELECT @s1, UPPER(CONVERT(@s1 USING latin1));
```

#### 3.1.2.2 字符集相关的系统变量

MySQL服务器有几个系统变量，它们设计字符集支持的各个方面。大部分与字符集有关，其余与排序规则有关。每个排序规则变量都有一个相应的字符集变量与之相连。

* 有些字符集表示的是服务器或当前数据库的属性

    character_set_system、character_set_server和collation_system、character_set_database和collation_database

* 其他的字符集变量影响客户和服务器之间的通信

    character_set_client、character_set_results、character_set_connection、character_set_filesystem

绝大多数字符集和排序规则都被默认设置成了相同的值

```mysql
mysql> SHOW VARIABLE LIKE 'character\_set\_%';
mysql> SHOW VARIABLE LIKE 'collation\_%';
```

如果某个用户客户端想用另一种字符集来与服务器通信，那就要修改与通信有关的变量，例如想用`utf8`，则需要

```mysql
mysql> SET character_set_client = utf8;
mysql> SET character_set_results = utf8;
mysql> SET character_set_connection = utf8;
```

使用SET NAMES语句可以更方便地达到同样效果。

```mysql
mysql> SET NAMES 'utf8';
```

### 3.1.3 时态(日期/时间)值

时态值包括日期值或时间值，如'2012-06-17'或'12:30:43'。

MySQL支持将日期时间合并在一起，即'2012-06-17 12:30:43'。MySQL是按照"年-月-日"的顺序来表示日期的，输入也必须是这样的顺序(标准的SQL格式)。可以利用DATE_FORMAT()函数按任意方式显示日期值。

组合后的日期时间值，允许在日期和时间之间加一个字符"T"，但不能用空格。

### 3.1.4 空间值

很少用

### 3.1.5 布尔值

表达式里，零为假，任何非零、非NULL值为真。

布尔常量TRUE和FALSE分别当做1和0，不区分大小写。

### 3.1.6 NULL值

NULL即一种没有类型的值，用来表示"无值"、"未知值"、"缺失值"、"超界"、"不适用"和"不在其中"等。可以将NULL插入表中、从表里检索、测试某个值是不是NULL。但不能对NULL进行算术运算。

使用NULL不用引号，不用区分大小写。\N(区分大小写)也当做NULL

```mysql
mysql> SELECT \N, ISNULL(\N);
```

## 3.2 MySQL数据类型

在使用CREATE TABLE语句创建表时，必须为每一个列定义一种类型。列的数据类型比一般的分类(如"数"或"字符串")更具体。同时，它也决定了MySQL将如何对待那些值。

对于一种数据，要考虑以下几个特点

* 它可以表示何种类型的值
* 这种类型的值要占用多少存储空间
* 值的长度是固定，还是可变的？
* MySQL会如何比较这种类型的值，如何对它进行排序
* 是否可以对这种类型进行索引

### 3.2.1 数据类型概述

* 数字类型

    包括整数、定点数、浮点数和位值，除BIT外，其他数都可以有正负号，也可以不带正负号。有种特殊的属性，可以让列自动生成一组有序的整数或浮点值。在你需要一组唯一标识编号的时候，可以使用它。

* 字符串类型

    可以容纳任何内容，甚至是用于表示图像和声音的二进制数据。可以按大小比较，可以进行模式匹配。

* 时态类型

    日期、时间、时间戳，不需要完整日期时，可以使用专用于表示年的类型。

### 3.2.2 表定义里的特殊列类型

调用CREATE TABLE语句创建表，列出表中所有的列

```mysql
CREATE TABLE mytbl
(
	f	FLOAT(10,4),
	c	CHAR(15) NOT NULL DEFAULT 'none',
	i	TINYINT UNSIGNED NULL
);
```

每个列都有一个名字、一个类型，以及一些有可能有的可选属性，列定义语法如下

*col_name col_type [type_attrs] [general_ attrs]*

*col_name*，列名，要求满足MySQL标识符的命名规范，2.2节

*col_type*，列类型，表明这一列可以用来容纳何种类型的值。有些类型**说明符会表明存储在列里的值所允许的最大长度**。而对于其他的类型说明符，所允许的长度则**隐含在它们的名称里**。CHAR(10)，指定了最多拥有10个字符。TINYTEXT隐含表示列的最大长度为255个字符。有些类型说明符允许指定一个最大显示宽度(用于显示值的字符有多少个)。

在列的数据类型后面，除了指定多个**通用属性**，还可以**指定一些类型特有的可选属性**。这些属性的作用是对该类型做进一步的修饰和限定，它们会对MySQL处理这个列的方式产生影响。

* 所允许的类型特有属性取决于具体的数据类型。数字类型才有UNSIGNED和ZEROFILL，非二进制串类型才有CHARACTER SET和COLLATE属性。
* 通用属性，NULL或NOT NULL表明某列是否允许NULL值。

如果一个列有多个属性，建议吧数据特有类型属性放在通用类型前面。

### 3.2.3 指定列的默认值

指定列的默认值可以通过

* DEFAULT添加

    除了BOLB、TEXT、空间类型、具有AUTO_INCREMENT属性的列，还可以指定DEFAULT *def_value*子句，用来表明：当创建新的行时，如果没有显式地指定某个值，那么该列将被赋值为默认值*def_value*。除了TIMESTAMP和DATETIME列有限制，*dev_value*都应该是一个常量。

* 没有显式的DEFAULT语句

    * 允许为NULL

        默认为NULL

    * 不允许为NULL

        没有默认值，如果往里插新行，没指定具体值，就会影响MySQL对该列的处理。

        * 如果没有启动SQL严格模式，设置成类型隐含的默认值
        * 启动了SQL严格模式
            * 如果表是事务性的，就会出错，中止执行然后回滚
            * 非事务性，如果这行是该语句插入的第一行，那就可以选择让语句中止执行，或者选择把这列设置为隐含值，并发警告

        隐含的默认值取决于数据类型

        * 数字列，默认值为0，AUTO_INCREMENT列，默认值为下一个列序号
        * 大多数时态类型，默认值为该类型的"零"值，TIMESTAMP列比较特殊
        * 字符串类型列，默认为空串。对ENUM，默认是枚举集里的第一个元素。对于SET，不允许包含NULL值，那默认是空集

使用SHOW CREATE TABLE tbl_name语句可以看到表里哪些列有DEFAULT子句，以及默认值是啥。

### 3.2.4 数字数据类型

* 精确值

    * 整数类型，存放没有小数的数字
    * `DECIMAL`，保存带有小数部分的精确值

* 浮点类型(近似值)

    * 单精度
    * 双精度

    精度要求不严格，数值很大，达到DECIMAL无法表示，就用浮点类型

* BIT类型，位域值

带小数部分的值可以赋值给一个整型列，但会使用"四舍五入"规则近似成整数。

数据类型使用要考虑**取值范围**、**存储空间**。

#### 3.2.4.1 精确值数字数据类型

精确值数据类型包括**整数类型**和**定点DECIMAL**类型。

MySQL中的整数类型包括`TINYINT`、`SMALLINT`、`MEDIUMINT`、`INT(INTEGER)`和`BIGINT`。可以将整数定义为`UNSIGNED`，来禁止出现负数。定义整型列时，可以指定一个可选的显式宽度*M*，但与值如何存储无关。

`DECIMAL`列可以被定义成`UNSIGNED`，但`DECIMAL`不会扩大数据范围，只会砍掉负数部分。`DECIMAL`可以包含一个最大有效位数*M*和一个小数位数*D*，*M*的范围是1~65，*D*的范围是0~30，且不超过*M*。

```
DECIMAL = DECIMAL(10) = DECIMAL(10,0)
DECIMAL(n) = DECIMAL(n,0)
```

#### 3.2.4.2 近似值数字数据类型

`FLOAT`和`DOUBLE`(`DOUBLE PRECISION`)，`REAL`是`DOUBLE`的同义词，如果启用了SQL的`REAL_AS_DEFAULT`模式，就会成为`FLOAT`的同义词。

浮点也可以被定义为`UNSIGNED`，负数部分将被砍掉。

浮点类型可以包含一个最大有效位数*M*和一个小数位数*D*，*M*的范围是1~255，*D*的范围是0~30，且不超过*M*。*M*和*D*都是可选的，如果定义列的时候省略了，这些值会按照硬件支持的最大精度来存。

还可以使用FLOAT(*p*)这样的语法来控制精度。*p*的取值范围0~53，0~24单精度，25~53双精度。

#### 3.2.4.3 BIT 数据类型

BIT列的定义里可以包含一个可选的最大宽度*M*，表示按二进制位计算的"宽度"。*M*的取值必须是一个1~64的整数。

默认情况下，从BIT列检索出来的值不能打印显示。要显示一个位域值的可打印形式，要加上0或者中CAST()函数

```mysql
mysql> CREATE TABLE t (b BIT(3));
mysql> INSERT INTO t (b) VALUES(0), (b'11'), (b'101'), (b'111');
mysql> SELECT b+0, CAST(b AS UNSIGNED) FROM t;
```

用二进制表示法显示这些位域值或以二进制形式计算后的结果，可以用BIN()函数。

```mysql
mysql> SELECT BIN(b), BIN(b & b'101'), BIN(b | b'101') FROM t;
```

八进制和十六进制可以用OCT()和HEX()

#### 3.2.4.4 数字数据类型的属性

* UNSIGNED

    防止出现负值，适用于除了BIT以外的所有数字类型。

* ZEROFILL

    适用于除了BIT意外的所有数字类型。会在列里显示值的前面填充若干0，达到显示宽度。

* AUTO_INCREMENT

    用于整数或浮点型。

    插入NULL到AUTO_INCREMENT列时，从1开始，每新增一行加1。如果表里删了几行，有可能受影响，即序列值会不会再次使用，取决于存储引擎。

    **每个表最多只能有一个AUTO_INCREMENT列**，该列应该有**NOT NULL**约束，并且必须被**索引**。

    ```mysql
    CREATE TABLE ai (i INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY);
    CREATE TABLE ai (i INT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE);
    CREATE TABLE ai (i INT UNSIGNED NOT NULL AUTO_INCREMENT, PRIMARY KEY(i));
    CREATE TABLE ai (i INT UNSIGNED NOT NULL AUTO_INCREMENT, UNIQUE(i));
    ```

    前两种形式将索引信息指定为列定义的一部分，后两种形式指定为CREATE TABLE语句的一个独立子句。如果索引只包含AUTO_INCREMENT，可以不用独立子句，如果还涉及其他列，就要用独立子句。

* NULL和NOT NULL

* DEFAULT

 #### 3.2.4.5 选择数字数据类型

* 选择数据类型的时候要根据实际的需求，取值范围、取值精度
* 数据的截断规则与SQL模式
* 浮点数精度，取舍情况

### 3.2.5 字符串数据类型

字符串可以用于表示任何值。

使用字符串时，要指定定义字符串值列的类型，以及这些类型的最大大小和存储空间要求。

要分清楚哪些类型用来存二进制串(字符串)，哪些类型用来存非二进制串，如BINARY(20)容纳20个字节，CHAR(20)容纳20个字符

| 二进制串类型 | 非二进制串类型 |
| :----------: | :------------: |
|    BINARY    |      CHAR      |
|  VARBINARY   |    VARCHAR     |
|     BLOB     |      TEXT      |

所有的非二进制串类型，包括ENUM和SET类型，都可以指定具体的字符集和排序规则。不同列可以指定不同字符集。

BINARY和CHAR都是固定长度的字符串类型，MySQL会给每个值分配同样数量的存储空间，对那些比列的长度短的值可以进行补齐。BINARY用0x00补齐，CHAR用空格补齐。CHAR(M)列占的空间跟列表示的字符集的最卡可能字符串有关。

其他字符串长度可变。处理长度可变数据时，MySQL会把数据内容和数据长度都存起来。这些额外的"长度字节"前缀，会被当作无符号整数来对待。

对VARBINARY和VARCHAR类型，如果列值的最大字节长度小于256，那么其长度前缀将占用1个字节，否则将占用2个字节。(长度前缀用于记录长度，如果大于256则需要多一个字节)

对于ENUM和SET类型，其列定义里都有一个合法字符串值的列表，但ENUM和SET的值在内部都被存储为数字。在未启用严格模式的情况下，如果把某个未出现在列表里的值存储到ENUM或SET类列里，会导致该值被转换为一个空串('')，严格模式下会导致错误。

#### 3.2.5.1 CHAR和VARCHAR数据类型

用于保存非二进制串，有字符集和排序规则。

CHAR和VARCHAR区别

* CHAR长度固定，VARCHAR长度可变
* CHAR列检索出的值，尾部空格会被移除。(如果值长度小于M个字符)
* 对于VARCHAR列，其尾部空格在存储和检索时都会被保留

在定义CHAR列时，最大长度M的取值范围是0~255。省略就是1，可以有CHAR(0)。

对于VARCHAR(M)，其中的M在语法上的取值范围是1~65535，实际能够容纳的最大字符个数肯定小于65535(行的最大长度为65535个字节)。潜在的影响

* 长VARCHAR列需要2个字节存放字符串的长度，最终长度不能超过行的总长。
* 使用多字节字符可以减少字符个数，从而使字符总长度不会超过最大行长度。
* 表里的其他列也会减少行里VARCHAR列的可用空间量。

使用CHAR和VARCHAR这两种数据类型的原则

* 如果所有值的长度都是M的字符，那VARCHAR(M)会比CHAR(M)列多占据一些存储空间(存记录值的长度)。如果长短不一，那用VARCHAR类型可以节省空格占用的存储空间。CHAR(M)列始终会占用M个字符的空间，即使其值为空或NULL也一样。
* 如果用MyISAM表，各个值的长度差别不大，那用CHAR会比VARCHAR好，因为MyISAM存储引擎对固定长度行的处理效率，比对长度可变行的高。

#### 3.2.5.2 BINARY和VARBINARY数据类型

BINARY和VARBINARY类型与CHAR和VARCHAR相似，不同之处

* CHAR和VARCHAR存储非二进制字符串，有字符串和排序规则。
* BINARY和VARBINARY都是用来存字节的二进制类型，没有字符集和排序规则，比较操作依据的是字节值大小。

对BINARY(M)列，存储那些长度短于M个字节的值时，会用0x00字节补齐。检索的时候不会去掉。VARBINARY存的时候不补齐，检索页不去除内容。

#### 3.2.5.3 BLOB和TEXT数据类型

"blob"指二进制大对象(binary large object)，实际上是一个能够存放任何内容的容器，多达4GB。如果要存的信息可能会急剧膨胀，或者各行长短差异大，那就很适合用BLOB类型。（TINYBLOB、BLOB、MEDIUMBLOB和LONGBLOB）

TEXT类型家族，包括TINYTEXT、TEXT、MEDIUMTEXT和LONGTEXT。相应的BLOB类型有很多相似之处，但TEXT类型存储非二进制串，即存储字符，与字符集和排序规则关联。

TEXT类型的最大长度与BLOB类型相同，都以字节为单位。

BLOB和TEXT列能否被索引，具体取决于所使用的存储引擎

* InnoDB和MyISAM都支持对BLOB和TEXT列进行索引。但必须制定一个前缀长度。但FULLTEXT基于索引列完整内容为基础，所以指定的前缀长度会被忽略。
* MEMORY表不支持BLOB和TEXT索引，因为MEMORY引擎根本不支持BLOB和TEXT列。

对于BLOB或TEXT列，需要注意

* BLOB和TEXT列在长度方面差异很大，在多次执行删除和修改操作后，表里会有很多碎片。如果用的是MyISAM表，来存，可以定期运行OPTIMIZE TABLE来减少碎片、改善性能。
* max_sort_length系统变量指定了只有最前面的max_sort_length个字节会被用到。
* 对于数据量非常大的值，要配置MySQL服务器，增大max_allowed_packet参数的值。

#### 3.2.5.4 ENUM和SET数据类型

ENUM和SET都只能从一个固定的(预先定义好的)字符串列表里取值。

两种类型的主要区别：ENUM列值必须包含且只能包含一个值列表成员，而SET列值则允许包含任意多个值列表成员(可以为空，也可以是全体成员)。

做选择

```mysql
employees ENUM('less than 100', '100-500', '501-1500', 'more than 1500')
color ENUM('red', 'green', 'blue', 'black')
size ENUM('S', 'M', 'L', 'XL', 'XXL')
vote ENUM('Yes', 'No', 'Undecided')
```

创建SET列时，同样需要为它指定一个所允许的集合成员列表。

用一个SET来表示这些配料

```mysql
SET('pepperoni', 'sausage', 'mushrooms', 'onions', 'ripe olives')
```

于是，特定的SET值代表了顾客实际所选择的配料

```
'pepperoni, mushrooms'
'sausage, onions'
'sausage, mushrooms, ripe olives'
'onions'
''
```

SET列允许有空串这样的值。

SET列的定义，可以写成以逗号分隔的单个字符串列表，表示所有的集合成员。SET列的值必须是一个字符串。如果这个值由集合里的多个成员构成，则必须用逗号把各个成员分隔开。也就是说不应该把一个含逗号的字符串用作SET的成员。

在为ENUM或SET列定义合法取值列表时，必须考虑到以下几个因素

* 这个列表确定了列的所有允许值
* 如果ENUM或SET列**带有一个不区分大小写的排序规则**，那么在插入合法值时也不用区分大小写。但是，当在检索ENUM或SET列里的数据时，它们将按照列定义的合法值列表里的字母大小写形式来显示。如果这个列带有区分大小写的排序规则或二进制排序规则，那么在插入数据时，必须按照严格按照列定义里的大小写。
* ENUM定义里的值的顺序就是排序所用的顺序。SET定义里的值的排序也确定了排序顺序，只是关系更复杂些，因为其列值可能包含多个集合成员。
* 当MySQL显示某个由多个集合成员构成的SET值时，这些成员的列出顺序由它们在SET列定义里的顺序确定。

在创建ENUM和SET类型时，需要以字符串的形式列出枚举和集合成员，因此它们都被归类为字符串类型。但它们内部存储形式是数字，可以把它们当成数字对待，即ENUM和SET类型要比其他字符串类型有更好的处理性能。

* ENUM

    MySQL从1开始对ENUM列定义里的成员进行顺序编号(0为出错代码，字符串形式是空串)。ENUM列占用空间由枚举个数决定，1字节能表示256个值，2字节能表示65536个值。出错代码是枚举的隐含成员。如果非法值放入枚举，MySQL会赋值为那个出错成员。

    以字符串和数字方式检索ENUM值的情况

    ```mysql
    mysql> CREATE TABLE e_table (e ENUM('jane', 'fred', 'will', 'marcia'));
    mysql> INSERT INTO e_table
    	-> VALUES('jane'), ('fred'), ('will'), ('marcia'), (NULL);
    mysql> SELECT e, e+0, e+1, e*3 FROM e_table;
    ```

    按名字或编号来对ENUM成员进行比较

    ```mysql
    mysql> SELECT e FROM e_table WHERE e='will';
    mysql> SELECT e FROM e_table WHERE e=3;
    ```

    可以把空串赋给一个非零值，但要注意，如果插入了一个错误成员，那它内部就是错误代码0，它被检索出来也是空串

    ```mysql
    mysql> CREATE TABLE t (e ENUM('a', '', 'b'));
    mysql> INSERT INTO t VALUES('a'), (''), ('b'), ('x');	# x是非法值
    mysql> SELECT e, e+0 FROM t;
    ```

* SET

    SET列的数字表示与ENUM列稍有不同，SET成员并没有按照顺序编号。每个SET成员对应着SET值里的一个二进制位。第一个成员对应于第0位，第二个成员对应于第1位，SET成员的数字值都是2的幂。空串对应SET数字值为0。

    SET最多只能有64个成员。1~8个、9~16个、17~24个、25~32个、33~64个，对应占用空间为1、2、3、4或8个字节。

    一个SET值可以由多个集合成员组成

    ```mysql
    mysql> CREATE TABLE s_table (s SET('table', 'lamp', 'chair', 'stool'));
    mysql> INSERT INTO s_table
    	-> VALUES('table'), ('lamp'), ('chair'), ('stool'), (''), (NULL);
    mysql> SELECT s, s+0, BIN(s+0) FROM s_table;
    ```

    如果把'chair, couch, table'赋值给s_table表里的s列，那会发生两件事

    * 'couch'会被剔除，因为它不是集合成员
    * 检索的时候将显示'table, chair'，即按照列定义里的顺序对子字符串自动重排。即使把某个值多次赋给一个SET列，那在检索的时候也只会看到一个'lamp'。

ENUM和SET列的排序和索引都是按照列值的内部值(即数字值)执行的。

如果有一组固定值，你想按照某种顺序对它们进行排序，那么可以利用ENUM类型的排序特性。如果想让某个ENUM列按照常规字母表顺序排序，可以先用CAST()函数把这个列转换为一个非ENUM的字符串，然后排序。

#### 3.2.5.5 字符串数据类型属性

CHARACTER SET(或CHARSET)和COLLATE，适用于CHAR、VARCHAR、TEXT、ENUM和SET，即所有包含字符串的类型。

指定这两个属性时一定要注意，不管在列、表和数据这三个级别中哪个级别

* 服务器必须支持该字符集(用SHOW CHARACTER SET查)
* 如果这两个同时用，要确保它们兼容
* 定义了CHARACTER SET但没有COLLATION，就用默认的排序规则
* 定义了COLLATION但没有CHARACTER SET，那具体字符集就用排序规则名的第一部分确定

查询某个表字符集信息

```mysql
mysql> SHOW CREATE TABLE mytbl;
```

查询排序规则信息

```mysql
mysql> SHOW FULL COLUMNS FROM mytbl;
```

在MySQL处理字符列定义时，会依次根据下述原则为它确定使用什么字符集

* 如果列定义里指定了字符集，那么使用这个字符集
* 否则，如果表的定义里有字符集选项，那使用这个字符集
* 否则，把数据库的字符集作为表的默认字符集，该字符集还将成为列的默认字符集；如果以前没有为数据库显式指定字符集，那数据库的字符集将沿用服务器的字符集。

字符集名binary比较特殊，将某个非二进制串列指定为binary字符集时，相当于把该列定义为相应的二进制串类型，以下几对列定义等价

```mysql
c1 CHAR(10) CHARACTER SET binary
c1 BINARY(10)

c2 VARCHAR(10) CHARACTER SET binary
c2 VARBINARY(10)

c3 TEXT CHARACTER SET binary
c3 BLOB
```

如果为某个二进制串列指定了CHARACTER SET binary，会被忽略。如果把binary字符集赋值为表选项，那么它将应用到所有在其定义里没有指定任何字符集信息的字符串列

对于字符列定义的属性，MySQL提供了多种简写

* ASCII，即CHARACTER SET latin1

* UNICODE，即CHARACTER SET ucs2

* 对非二进制串列、ENUM列或SET列，如果指定BINARY属性，即等同于在指定该列字符集的二进制排序规则，以下定义等效

    ```mysql
    c1 CHAR(10) BINARY
    c2 CHAR(10) CHARACTER SET latin1 BINARY
    c3 CHAR(10) CHARACTER SET latin1 COLLATE latin_bin
    ```

任何字符串类型都能用通用属性NULL或NOT NULL。如果没指定，就是NULL。但字符串声明为NOT NULL并不是说它不能存' '，NULL与' '还是有区别的。

可以用DEFAULT子句，为除了BLOB和TEXT类型外的其他字符串数据类型指定默认值。

#### 3.2.5.6 选择字符串数据类型

选择字符串数据类型，要考虑这几个问题

1. 表示字符数据还是二进制数据？

2. 比较操作要区分大小写吗？(字符串与排序规则)

3. 想少占点空间吗？(用不用可变长类型)

4. 是不是从固定的某些值里取？(ENUM、SET)

5. 尾部填充值重要吗？

    尾部填充值的问题

    | 数据类型           | 存储     | 检索   | 结果                 |
    | ------------------ | -------- | ------ | -------------------- |
    | CHAR               | 填充空格 | 去掉   | 检索值无尾部填充     |
    | BINARY             | 填充0x00 | 不处理 | 检索值会保留尾部填充 |
    | VARCHAR, VARBINARY | 不处理   | 不处理 | 尾部填充无变化       |
    | TEXT, BLOB         | 不处理   | 不处理 | 尾部填充无变化       |

    启用PAD_CHAR_TO_FULL_LENGTH可以让检索出来的CHAR列值保留尾部空格。

### 3.2.6 时态(日期/时间)数据类型

MySQL提供了多种存储时态值的类型：DATE、TIME、DATE、TIMESTAMP和YEAR。

MySQL5.6版本里，这些类型有了多项重要改进

* 对数据类型TIME、DATETIME和TIMESTAMP，MySQL5.6.4增加了对小数秒的支持，可选小数部分多达6位(微秒)精度
* MySQL5.6.5引入扩展支持，自动把当前时间戳作为初始值并进行更新。
* MySQL5.6.6丢弃了对YEAR(2)的支持，允许创建像YEAR(4)这样的列。

有兴趣可以了解下时态数据类型的取值范围。

如果要声明包含**小数秒部分的时态类型列**，就需要把定义写成*type_name(fsp)*，其中type_name为TIME、DATETIME或TIMESTAMP，*fsp*为小数秒精度，范围是0~6，默认为0。

```mysql
t1 TIME(3)	# 允许3位小数
t2 TIME(6)	# 允许6位小数
```

注意不同时态的数据类型存储空间要求，还有带有小数秒部分具有的额外存储空间要求。

当为某种日期/时间类型插入非法值时，默认存储为一个"零"值。如果要把非法值处理为错误，就需要设置相应的SQL模式。"零"值也是那些声明时带有NOT NULL属性的日期/时间类型列的默认值。

按照SQL和ISO 8601规范的要求，MySQL的日期表示顺序为"年-月-日"。为了满足检索显示要求，用*DATE_FORMAT()*函数和*TIME_FORMAT()*函数来显示各种格式的日期和时间。

#### 3.2.6.1 DATE、TIME和DATETIME数据类型

DATE：日期值，'*CCYY-MM-DD*'

TIME：时间值，'*hh:mm:ss[.uuuuuu]*'

DATETIME：日期和时间的组合，'*CCYY-MM-DD hh:mm:ss[.uuuuuu]*'

从MySQL 5.6.5起，DATETIME列会自动把当期时间戳作为初始值，进行更新。

如果把DATE值赋值给DATETIME列，那么MySQL会自动把时间部分补足为'00:00:00'。反过来，如果DATETIME赋值给DATE或TIME列，MySQL会把不相干部分去掉。

从TIME到DATETIME，依赖具体的MySQL版本；从MySQL5.6.4起，当前日期会添加上时间。

在MySQL里，DATETIME的时间部分表示的是一天里的时间，必须在'00:00:00'~'23:59:59'范围里。但TIME表示的是一段逝去的时间，TIME可以是负值，可以大于'23:59:59'。

插入TIME最好插完整，不要'30'或'12:30'这样插。'12:30'到底是'12:30:00'还是'00:12:30'傻傻分不清。

#### 3.2.6.2 TIMESTAMP数据类型

TIMESTAMP时态数据类型，存日期和时间的组合值。

取值范围：'1970-01-01 00:00:00[.000000]'~'2038-01-19 03:14:07[.999999]'

MySQL 5.6.4之前，TIMESTAMP值允许有小数部分，但在存储时会被丢弃。

TIMESTAMP的取值范围跟Unix时间相关，规定1970年的第一天为"零日"，也称作"纪元"。每一个TIMESTAMP值，MySQL会用4个字节来把它存储为**自纪元以来总共逝去的秒数**。

1970年的起始确定了TIMESTAMP类型的取值范围下限值。取值范围上限值则与4个字节所能表示的最大Unix时间相对应。

MySQL会按世界标准时间(Universal Coordinated Time, UTC)来存储TIMESTAMP值。保存这样的值时，服务器会把它从会话时区转换为UTC。以后检索该值时，服务器又会把它从UTC转换回会话时区，从而让你看到与你存储结果一样的时间值。不同客户端在不同时区连接服务器，并检索这个值，看到的值是设置为其所设置时区的那个值。

尝试更改时区

```mysql
mysql> CREATE TABLE t (ts TIMESTAMP);
mysql> SET time_zone = '+00:00';	# 将时区设置为UTC
mysql> INSERT INTO t VALUES('2000-01-01 00:00:00');
mysql> SELECT ts FROM t;
```

修改下时区

```mysql
mysql> SET time_zone = '+03:00';	# 将时区前调3小时
mysql> SELECT ts FROM t;
```

TIMESTAMP列会自动把当前时间戳作为初始值，并进行更新。此外，如果在定义TIMESTAMP列时为了允许NULL值而带有NULL属性，那么当把NULL存储到该列时，该列值会被设置为当前时间戳。

#### 3.2.6.3 YEAR数据类型

YEAR是单字节数据类型，用来提高年值的表示效率。YEAR的取值范围是1901~2155年。

如果只会用到日期里的年份，如出生年份、政府选举年份等，那用YEAR就够了。如果不需要完整的日期值，那用YEAR会比用其他日期类型更省存储空间。

在声明YEAR列时，可以指定一个显示宽度*M*，*M*只能为4或2

YEAR(2)只显示最后两位数，并且这种类型实际只能存储从1970年到2069年之间的值。因此避免使用YEAR(2)，用YEAR(4)来代替。

MySQL会把2位YEAR值转换成4位值，97和14会变成1997和2014。但00会变成0000而不是2000。

TINYINT类型的存储空间占用量与YEAR类型(只有1个字节)的一样，但取值范围不一样。

使用SMALLINT(占用2个字节)，能够覆盖YEAR类型所能表示的年份范围。

#### 3.2.6.4 时态数据类型的属性

时态列的定义可以包含通用属性NULL或NOT NULL。如果都不指定，默认为NULL，TIMESTAMP除外，默认值为NOT NULL。

也可以用DEFAULT子句来设定默认值。

大部分情况下默认值都必须为常量，除了TIMESTAMP和DATETIME，不能用像CURRENT_TIMESTAMP这样的函数来将DATETIME列的默认值设置为"当前日期和时间"。TIMESTAMP和DATETIME之所以特殊，是因为它们的**默认值可以为当前日期和时间**。

其他类型也想这样，可以在每次创建新行之后显式设置为CURRENT_TIMESTAMP。也可以用TIMESTAMP或DATETIME列来代替；或者设置一个触发器，设置该列的初始化。

#### 3.2.6.5 时态类型的小数秒功能

TIME、DATETIME和TIMESTAMP类型的声明语法中，允许设置一个可选的小数秒精度(*fsp*)，精度值最高可达6位数字。*fsp*必须是0~6，0表示没有小数部分，而6则表示精度为微秒。

对于带时态参数的函数，接受或返回的时态值中都带有小数秒部分。例如CURTIME()返回不带小数部分当前时间，但CURTIME(3)返回的时间就包括了一个精度高达千分之一秒的小数秒部分

```mysql
mysql> SELECT CURTIME(), CURTIME(3);
```

#### 3.2.6.6 时态类型的自动特性

TIMESTAMP和DATETIME可以有**自动初始化**和**自动更新**特性。

* 自动初始化

    对新行，如果在INSERT语句里省略了这两种类型的列，那么列会被设置为当前时间戳

* 自动更新

    对已有的行，当把任何其他列更改为不同值时，这两种类型的列都会被更新为当前时间戳。

可以通过指定TIMESTAMP列的语法，来设置它的默认值和自动更新

*col_name* TIMESTAMP [DEFAULT *default_value*] [ON UPDATE CURRENT_TIMESTAMP]

如果要让TIMESTAMP或DATETIME列**没有自动初始化或自动更新的特性**，那执行插入或更新操作时，可以**显式设置**成所期望的值。

TIMESTAMP和DATETIME列的定义可以包含**NULL或NOT NULL属性**。TIMESTAMP的默认属性是NOT NULL，当把列设置成NULL时，MySQL会把它设置为当前时间戳。如果是NULL那设置成NULL就是NULL。

* 新增行时，列被设置为当前时间戳，插入行后列会保持它原来的值，直到显式更改

    ```mysql
    CREATE TABLE t1
    {
    	ts_created TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    	# ... other columns ...
    }
    ```

* 两个分别用来存储创建时间和最后修改时间的列

    ```mysql
    CREATE TABLE t2
    {
    	ts_created TIMESTAMP DEFAULT 0,
    	ts_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    						  ON UPDATE CURRENT_TIMESTAMP,
    	# ... other columns ...
    }
    ```

    插入一个新行，就把两个TIMSTAMP列都设为NULL，就可以把它们设成插入时的时间戳，如果要更新已经有的行，那就可以不用管。ts_modified会被自动更新为修改时间戳。

#### 3.2.6.7 处理时态值

MySQL可以正确解释各种格式的日期和时间列的输入值，包括字符串形式和数值形式。

* DATETIME和TIMESTAMP支持的格式

    '*CCYY-MM-DD hh:mm:ss[.uuuuuu]*'

    '*YY-MM-DD hh:mm:ss[.uuuuuu]*'

    '*CCYYMMDDhhmmss[.uuuuuu]*'

    *CCYYMMDDhhmmss[.uuuuuu]*

    *YYMMDDhhmmss[.uuuuuu]*

    DATE、TIME和YEAR类型值，也有类似表示。

* 对于带分隔符的字符串格式，分隔符可以随意。但尽量遵循习惯。
* 日期、时间前面有前导0的情况，MySQL有多种不同解释方式，具体取决于是用字符串还是数值，填的时候尽量写完整。
* 一般情况下DATE、DATETIME和TIMSTAMP之间可以随意相互赋值，但注意赋值过程中的数据丢失（丢失部分时间）和填充（补零）。
* 注意各个类型时间的取值范围。

MySQL提供了很多处理日期和时间的函数，参考**附录C.2.5 日期/时间函数**

#### 3.2.6.8 解释模糊年份值

2位数字的年份值会被转换成4位数字

* 在00~69之间的值，被转化为2000~2069
* 在70~99之间的值，被转化为1970~1999

年份要写就写完整。

### 3.3 MySQL如何处理无效数据值

MySQL有几种SQL模式可以拒绝"坏"值，并且会在遇到"坏"值时抛出一个错误。

* MySQL处理越界值和非正常值的规则
    * 对数值或TIME列，超出合法取值范围的那些值将被截断到取值范围最近的那个端点，并存储结果值
    * 对除TIME以外的其他时态类型，非法值会被转换成与该类型相一致的"零"值
    * 对字符串列，过长的字符串将被截断到该列的最大长度
    * 给ENUM或SET类型列进行赋值时，需要根据列定义里给出的合法取值列表进行

    如果在执行INSERT、REPLACE、UPDATE、LOAD DATA和ALTER TABLE等语句时发生了这些转换，那MySQL会给出警告信息。可以使用SHOW WARNINGS查看警告信息内容。

* 如果要在插入更新数据的时候进行更严格的检查，那可以启用两种SQL模式的一种

    ```mysql
    mysql> SET sql_mode = 'STRICT_ALL_TABLES';
    mysql> SET sql_mode = 'STRICT_TRANS_TABLES';
    ```

* 是否支持事务？

    对支持事务的表，也两种模式都是：如果发现某个值无效或缺失，那结果会有一个错误，语句会中止执行并回滚。

    对不支持事务的表有下面的效果
    * 如果插入或修改第一个行的时候，发现某个值无效或者缺失，就会错误终止执行，什么都没发生
    * 插入或修改多个行的语句，第一行之后出错，那就看是哪种严格模式
        * STRICT_ALL_TABLES模式下，抛出一个错误，语句停止执行，导致"部分更新"
        * STRICT_TRANS_TABLES模式下，因为之前的修改没法撤销，会把无效值转化为最接近的合法值。缺失值设置成数据类型的隐式默认值

* MySQL可以执行更加严格的检查

    * ERROR_FOR_DIVISION_BY_ZERO：阻止以零为除数的情况产生的数值进入数据库
    * NO_ZERO_DATE：严格模式下，阻止"零"日期值进入数据库
    * NO_ZERO_IN_DATE：严格模式下，阻止月或日部分为零的不完整日期值进入数据库

* 如何设置？

    想让所有存储引擎启动严格模式，并检查"除零"错误

    ```mysql
    mysql> SET sql_mode = 'STRICT_ALL_TABLES, ERROR_FOR_DIVISION_BY_ZERO';
    ```

    启用严格模式，以及所有附加限制，使用TRANDITIONAL，默认含义是"启用两种严格模式，外加一大堆的其他限制"

    ```mysql
    mysql> SET sql_mode = 'TRANDITIONAL';
    ```

## 3.4 处理序列

为达到标识的目的，许多应用都需要生成唯一编号。 MySQL提供唯一编号的机制是使用MySQL提供唯一编号的机制是使用AUTO_INCREMENT 列属性，自动生成序列编号。

MySQL使用不同的存储引擎，处理 AUTO_INCREMENT列的方式有所不同。

探讨两个问题

* 特定的引擎AUTO_INCREMENT列如何工作
* 不使用AUTO_INCREMENT列的情况下，如何生存序列

### 3.4.1 通用的AUTO_INCREMENT属性

 AUTO_INCREMENT列必须按照以下条件进行定义。

* 每个表只能有一个列具有AUTO_INCREMENT属性，并且它应该为整数数据类型。（也支持浮点类型，但很少用）。

* 列必须建立索引。最常见的情况是使用PRIMARY KEY或UNIQUE索引，但是也允许使用不唯一的索引。

* 列必须拥有NOT NULL约束条件。(没有就默认是NOT NULL)

    在创建之后，AUTO_INCREMENT列将具有以下行为

    * 把NULL值插入AUTO_INCREMENT列将引发MySQL自动生成下一个序列编号，并把它 插入列。在某些场合，根据所用存储引擎的不同，可以显式地设置或重置下一个序号，或者重复使用已从序列顶端删除的那些序号值。
    * 要获得最近生成的序号值，可以调用LAST_INSERT_ID()函数。没有生成过AUTO_INCREMENT值，那么LAST_INSERT_ID()将返回0。

    LAST_INSERT_ID()特性

    * LAST_INSERT_ID()只会依赖于与服务器的当前会话连接所生成的AUTO_INCREMENT值。你可以生成一个序号，接着在同一个会话连接里调用LAST_INSERT_ID()来检索它。即使其他客户在此期间生成了它们自己的序号值，返回的也是当前会话连接所生成的AUTO_INCREMENT值。
    * 一次插人多个行的INSERT语句，将生成多个AUTO_INCREMENT值，LAST_INSERT_ID() 只会返回其中的第一个。
    * 如果使用INSERT DELAYED，那么要直到实际插入行时，才会生成AUTO_INCREMENT值。这时不能依靠LAST_INSERT_ID()来返回序号值了。

* 在插入一行时，如果不为AUTO_INCREMENT列指定值，则等同于向该列插入一个NULL 值。
* 默认情况下，把0插入AUTO_INCREMENT列，等效于插入NULL值。但如果启用了SQL的 NO_AUTO_VALUE_ON_ZERO模式，那么插入0则会导致存储值为0，而非下一个序号值。   
* 如果要插入一行，并为某个拥有唯一索引的AUTO_INCREMENT列指定一个既不为 NULL, 也不为0的值，那么可能发生键重复错误，或从该值开始继续编号。使用UPDATE命令也会有类似的现象。
* 对于某些存储引擎，从序列顶端删除的值可以被重用。如果把包含最大 AUTO_INCREMENT列值的那行删除，那么在下次生成新值时可以重用这个最大值。如果 把表里的所有记录都删除，那么所有值都可以重用，并且这个序列会重新从1开始。
* 如果根据AUTO_INCREMENT列的值，**使用REPLACE来更新行**，那么这个行的 AUTO_INCREMENT值将保持不变。如果根据其他具有PRIMARY KEY或UNIQUE索引的列 的值，使用REPLACE命令来更新行，那么当你把AUTO_INCREMENT列设置为NULL，或者把它设置为0，而且没有启用NO_AUTO_VALUE_ON_ZERO时，该列的值将被更新为一个新的序号值。

###   3.4.2 存储引擎特有的AUTO_INCREMENT属性

MyISAM为序列处理提供了最大的灵活性。

#### 3.4.2.1 MylSAM表的AUTO_INCREMENT列

MyISAM存储引擎拥有以下AUTO_INCREMENT特征。 

* MyISAM表里的序列一般是单调的。在一个自动生成的序列里，一般都是严格递增的；被删除之后，不会被重用。如果最大值是143, 而你删除了包含这个值的行，那么MySQL生成的下一个值仍然是144。不过，这种行为存在以下两种例外情况。 
    * 如果使用TRUNCATE TABLE清空了表，那么计数器将被重置为从1开始。
    * 如果在表里使用了复合索引来生成多个序列，那么从序列顶端删除的那些值将被重 用。（此项技术随后将被讨论到。）

* MyISAM序列将**默认从1开始**，不过你可以在CREATE TABLE语句里，通过 AUTO_INCREMENT = n选项**显式地指定初始值**。

* 可以使用ALTER TABLE来更改某个已有MyISAM表的当前序列计数器。如果想要重用那些**从序列顶端删除**了的值，你也可以用这个语句把编号计数器设置到最低，最低只能是表里当前的最大计数值。

    ```mysql
    ALTER TABLE mytbl AUTO_INCREMENT = 10;
    ```

* 不能使用AUTO_INCREMENT选项来把当前计数值设置得比表里当前的最大计数值还小。 

MyISAM存储引擎支持在同一个表里使用复合（多列）索引，以创建多个相互独立的序列。   为利用这个特性，为表创建一个由多列组成的PRIMARY KEY或UNIQUE索引，并把包含 AUTO_INCREMENT的那个列作为其中的最后一个。**对于该索引最左边的列构成的每一个相异键， AUTO_INCREMENT列将生成一组彼此互不干扰的序列值**。

案例，有一个名为bugs的表，你需要用它来同时记录多个软件项目的bug报告，该表的定义如下：

```mysql
CREATE TABLE bugs
(
    proj_name	VARCHAR(20)	NOT NULL,	# 标识项目名
    bug_id	INT UNSIGNED NOT NULL AUTO_INCREMENT,	# 存放bug描述
    description VARCHAR(100),
    PRIMARY KEY (proj_name, bug_id)
) ENGINE= MYISAM; 
```

bug_id列是一 个AUTO_INCREMENT列，通过创建一个与proj_name列相关联的索引，可以为各个项目分别生 成一个互不干扰的序列编号。把下面儿行输到表里，记录Super Browser项目的3个bug, 以及SpamSquisher项目的2个bug:

```mysql
mysql> INSERT INTO bugs (proj_name,description)
	-> VALUES('SuperBrowser','crashes when displaying complex tables');
mysql> INSERT INTO bugs (proj_name,description)
	-> VALUES('SuperBrowser','image scaling does not work');
mysql> INSERT INTO bugs (proj_name,description)
	-> VALUES('SpamSquisher','fails to block known blacklisted domains');
mysql> INSERT INTO bugs (proj_name,description)
	-> VALUES('SpamSquisher', 'fails to respect whitelist addresses');
mysql> INSERT INTO bugs (proj_name,description)
	-> VALUES('SuperBrowser','background patterns not displayed');
```

这个表为每一个项目的bug_id值进行了单独的编号，整个过程与这些项目的行输入顺序 毫无关系。在输入另一个项目的行之前，你不用先全部输入某个项目的所有行。

如果使用复合索引来创建多个序列，那么从各个序列顶端删除的值都可以被重用。

#### 3.4.2.2 lnnoDB表的AUTO_INCREMENT列

lnnoDB存储引擎拥有以下AUTO_INCREMENT特征。

* 在CREATE TABLE语句里，可以使用AUTO_INCREMENT = n表选项来设置初始序列值， 并且在表创建之后，还可以使用ALTER TABLE来进行更改。 

* 从序列顶端删除的值通常不能再重用。不过，如果使用TRUNCATE TABLE清空表，那么 序列将被重置，并重新从1开始编号。

    此外，在满足后面几个条件时也可以重用

    * 首先，在首次为AUTO_INCREMENT列生成序号值时，lnnoDB会把列的当前最大值加上1, 得到的结果作为新生成的序号值（如果表此前为空，那么新的序号值为1)。
    * 其次，为满足生成后续序号值的需要，InnoDB是**在内存里维护这个计数器**，它**并未存储在表内部**。这意味着，如果从这个序列的顶端删除了某些值，然后又重启了服务器，那么这些删除的值将会被重用。
    * 重启服务器还将取消在CREATE TABLE或ALTER TABLE语句里使用AUTO_INCREMENT表选项所带来的效果。 

* 如果生成AUTO_INCREMENT值的事务被回滚，那么在序列里可能会出现断裂。

* 在表里，不能使用复合索引生成多个独立的序列。   

#### 3.4.2.3 MEMORY表的AUTO_INCREMENT列

 MEMORY存储引擎拥有以下AUTO_INCREMENT特征。

* 在CREATE TABLE语句里，可以使用AUTO_INCREMENT = n表选项来设置初始序列值， 并且在表创建之后，还可以使用ALTER TABLE来进行更改。   

* 从序列顶端删除的值通常不能再重用。不过，如果使用TRUNCATE TABLE清空表，那么 序列将被重置，并重新从1开始编号。   

* 在表里，不能使用复合索引生成多个独立的序列。   

### 3.4.3 使用AUTO_INCREMENT列需要考虑的问题

在使用AUTO_INCREMENT列时，为避免陷入不必要的麻烦，请牢记以下几点。

* AUTO_INCREMENT机制的主要用途是生成一个正整数序列，不支持非正数。因此，可以把AUTO_INCREMENT列定义为UNSIGNED类型。对于整型列，使用UNSIGNED可以在达到该数据类型的范围上限前，获得两倍的序列编号。   

* 把AUTO_INCREMENT添加到列的定义里，**并不能得到无穷尽的序列编号**。   AUTO_INCREMENT序列总是会受到底层数据类型的取值范围的约束。一旦达到上限，应用程序便会因“键重复”错误而失败。

* 使用**TRUNCATE TABLE语句**来清除某个表的内容，可以把该表的计数序列重置为重新从1开始。即使对于那些通常不会重用AUTO_INCREMENT值的存储引擎，也是如此。之所以会重置序列，是因为MySQL需要对整表的删除操作进行优化。只要有可能，它会快速丢弃全部的行和索引，并且从头开始重建该表，而不是一次删除一行。这样做将导致序列编号信息出现丢失。如果想在删除所有行的同时把序列信息保留下来，那么可以使用带有一个永远为真的WHERE子句的DELETE语句来阻止这个优化操作的执行。该语句会迫使MySQL针对每行计算一次条件，然后单独地删除每行：

    ```mysql
    DELETE FROM cbl_name WHERE TRUE;
    ```

### 3.4.4 AUTO_INCREMENT列的使用提示

本节将讨论使用AUTO_INCREMENT列时可以用到的一些实用技术。   

#### 3.4.4.1 为表增加一个序列编号列

假设，你创建并填充了一个表：

```mysql
mysql> CREATE TABLE t (c CHAR(lO));
mysql> INSERT INTO t VALUES('a')'('b')'('c');
mysql> SELECT* FROM t;
```

接着为这个表增加一个序列编号列。

```mysql
mysgl> ALTER TABLE t ADD i INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY;
mysgl> SELECT* FROM t;
```

#### 3.4.4.2 重置已有列的序列编号

如果表已经有了一个AUTO_INCREMENT列，但是你想要对其进行重新编号，以便消除因删 除行而在序列里产生的断裂。

实现这一目标的最简单办法是：先删除该列，然后再重新添加它。 

当MySQL添加列时，会自动分配新的序列号。   假设，有这样一个表t, 其中的1为一个AUTO_INCREMENT列：

```mysql
mysql> CREATE TABLE t(c CHAR(lO), i INT UNSIGNED AUTO_INCREMENT 
	-> NOT NULL PRIMARY KEY);
mysql> INSERT INTO t (c)
	-> VALUES ('a')'('b')'('C')'('d')'('e')'('f')'('g')'('h')'('i')'('j')'('k');
mysql> DELETE FROM t WHERE C IN('a','d','f','g','j');
mysql> SELECT* FROM t;
```

下面的ALTER TABLE语句将依次删除列，再重加它，并对列重新编号：

```mysql
mysql> ALTER TABLE t
	-> DROP PRIMARY KEY,
	-> DROP i,
    -> ADD i INT UNSIGNED NOT NULL AUTO INCREMENT PRIMARY KEY,
    -> AUTO_INCREMENT = 1;
mysql> SELECT * FROM t;
```

AUTO_INCREMENT = 1子句将把序列的起始编号重置为1。对于MyISAM、MEMORY或 InnoDB表，可以把序列的起始编号设置为不等于1的其他数。对于其他的存储引擎，则可以忽略AUTO_INCREMENT子句，这个序列将从1开始编号。

虽然可以轻易地重新设置列的编号，但通常没有必要这么做。MySQL并不会在意序列里是否存在有断裂，并且在重置序列编号后也不会获得任何性能上的提高。此外，如果在**另一个表里有行引用了AUTO_INCREMENT列里的值，那么调整该列的编号将破坏这两个表之间的对应关系**。

### 3.4.5 在无AUTO_INCREMENT的情况下生成序列

如果在插入或修改一个列时使用了LAST_INSERT_ID(expr)函数，那么下次不带参数调用LAST_INSERT_ID()时，它会返回表达式expr的值。

MySQL会把表达式expr当作是一个新生成的AUTO_INCREMENT值。因此，你可以先创建一个序列编号，之后检索它，同时**不用担心这个编号值会受到其他客户端程序活动的影响**。（因为LAST_INSERT_ID()函数是客户端专用的，即使在UPDATE和SELECT两个操作之间的时间间隔里，其他客户端程序又生成多个序列编号，你也会检索到正确值。）

这种策略的一种用途是，创建一个只有一个行的表，其中包含一个在每次需要该序列里下一个值时都会进行更新的值

```mysql
CREATE TABLE seq_table (seq INT UNSIGNED NOT NULL);
INSERT INTO seq_table VALUES(0);
```

上面这些语句会创建一个只有一个行的seq_table表，其中seq的值为0。为生成下一个 序列编号，并对它进行检索，可以这样做：

```mysql
UPDATE seq_table SET seq= LAST_INSERT_ID(seq+l);
SELECT LAST_INSERT_ID();
```

上面的UPDATE语句将检索seq列的当前值，并把它加上1，从而产生该序列的下一个编号。

利用LAST_INSERT_ID(seq+1)生成的新编号值，与AUTO_INCREMENT值很相像，因此可以通 过不带参数调用LAST_INSERT_ID ()的方式来检索它。

利用这种方法，**可以生成步长不为1(甚至可以为负值）的序列编号**。可以递增也可以递减。      

也可以生成一个从任意值开始编号的序列，只要给seq列设置一个合适的初始值就行。

以上描述了如何利用一个只包含一行的表来建立一个计数器。

如果需要多个计数器，则可以这样做：首先给这个表增加一列，用作计数器的标识符；然后在这个表里为每个 计数器单独增加一行。

* 假设，有一个网站要实现"某网页已被访问过n次"的页计数器。创建一个具有两个列的表：一列用于保存计数器的唯一标识，而另一列保存计数器的当前值。

    可以使用LAST_INSERTED_ID()函数，但需要根据计数器的名字来确定它应该应用于哪一行。例如，可以用下面的语句来创建这样一个表：

    ```mysql
    CREATE TABLE counter (
        name VARCHAR(255) CHARACTER SET latinl COLLATE latinl_general_cs NOT NULL,
        value INT UNSIGNED,
        PRIMARY KEY (name)
    );
    ```

    其中，name列是计数器名。为了避免计数器的名字出现重复，被定义成了PRIMARY KEY。

    在使用counter表时，INSERT ... ON DUPLICATE KEY UPDATE语句可以为一个未曾统计过的页面插入一个新行，也可以更新已有页面的计数值。

    通过使用LAST_INSERT_ID(expr)函数来生成计数值，可以在更新当前计数器后，轻易地将其值检索出来。

    例如，为了初始化或递增网站主页的计数器，然后检索计数器显示结 果，可以像下面这样做：

    ```mysql
    INSERT INTO counter (name, value) VALUES ('index. html', LAST_INSERT_ID (1)) ON DUPLICATE KEY UPDATE value= LAST_INSERT_ID(value+l);
    SELECT LAST_INSERT_ID();
    ```

     在不使用LAST_INSERT_ID()函数的情况下，另一种递增已有页面计数器的方法是：

    ```mysql
    UPDATE counter SET value= value+l WHERE name='index.html';
    SELECT value FROM counter WHERE name='index.html';
    ```

    不过，在执行UPDATE语句之后，且在执行SELECT语句之前，如果有另一个客户端递增了 这个计数器，那么value值就不准确了。可以有两种解决方案

    * 通过LOCK TABLE和UNLOCK TABLE把这两条语句围起来
    * 选用支持事务处理的存储引擎来创建这个表，并且在一 个事务里更新这个表

    这两种办法都可以在你使用这个计数器时，阻断其他客户端。

    但是使用LAST_INSERT_ID ()函数可以更容易地完成这件事情。因为计数器的值是客户端专用的，所以你总是能获得你刚插入的那个值，而不会获得来自其他客户端插入的值；并且，你不用为了达到排斥其他客户端的目的，而使用表锁定或事务来把代码弄得很复杂。 

## 3.5 表达式计算和类型转换

表达式包含子项和运算符，并且经过计算可以产生具体值。

子项：包含值，如常数、函数调用、表列引用，以及标量子查询。

这些值可以用**不同种类的运算符**（如算术运算符或比较运算符）组合在一起，表达式的子项可以用括号进行分组。

绝大部分情况下，表达式都是出现在输出列的列表里，以及SELECT语句的WHERE子句里。表达式也可以出现在 DELETE和UPDATE语句的WHERE子句里、INSERT语句的VALUE()子句里等。

当MySQL遇到表达式时，它会计算这个表达式，并产生一个结果。

表达式的计算过程可能涉及类型转换。

本节讨论如何在MySQL里编写表达式，讨论在表达式计算过程中各种类型转换的规则。

### 3.5.1 编写表达式

表达式可以很简单，最简单的就是一个常量，如数字0和字符串'abc'。

表达式可以使用函数调用。（具体怎么调用？参数怎么分隔？函数名与括号空格？）

表达式里可以包含对表列的引用。对于最简单的情形，当可以根据上下文清楚地知道列所     属的那个表时，只需给出该列的名字即可。如果还不清楚应该使用那个数据库、哪个表、，则需要在列名的前面加上正确的**数据库名**、**表名**进行限定。如果无法知道应该使用哪一个数据库，则需要在表名前面加上数据库名进行限定。最稳妥的方法就是使用这种完整的限定形式。

在表达式里，标量子查询可以用于提供单个值。这种子查询必须用栝号括起来：

```mysql
SELECT * FROM president WHERE birth= (SELECT MAX(birth) FROM president);
```

可以根据所有这些类型的值（常量、函数调用、列引用和子查询）组合成各种更加复杂的表达式。

#### 3.5.1.1 运算符类型

可以使用各种运算符把表达式的各个子项组合在一起。

运算符主要分为以下几种

* 算术运算符：加法、减法、乘法、除法(/、DIV)和取模

    当两个操作数都是整数时，对于运算符＋、－和＊，整个运算过程所使用的都是BIGINT     (64位）整数值。如果两个操作数都是整数，且其中有一个是无符号数，那么结果也将是无符号数。

    对于除DIV以外的所有算术运算符，如果某个操作数是近似值，那么整个计算过程会遵循双精度浮点运算规则。

    如果某个整数运算涉及了很大的值（如超出了64位所能表示的范围），那么得到的结果是不可预测的。尽量避免使用超过63位的整数。

* 逻辑运算符：AND和&&、OR和||、XOR、NOT、!

    跟C语言里的用法一样。

    在用||时要注意，这个在标准SQL里代表字符串拼接，但在MySQL里进行的是逻辑OR运算，如果想用标准SQL那就要设置SQL为PIPES_AS_CONCAT模式。

* 按位运算符：完成二进制位的与、或和异或、移位运算。

* 比较运算符：用来测试数字和字符串的相对大小或词法顺序。

模式匹配允许我们在不指定精确字面值的情况下查找值。MySQL提供了两种类型的模式匹 配。

* 一种是利用LIKE运算符以及通配符"%" (能匹配任意字符序列）和"\_" (只能匹配单个 字符）实现的SQL模式匹配。

* 另一种是利用REGEXP运算符和正则表达式（它与grep、sed和 vi等UNIX程序所使用的正则表达式很相似）实现的模式匹配。必须使用其中的一种运算符来完成模式匹配，并且不能使用＝运算符。

要颠倒模式匹配的含义，可以使用运算符NOT LIKE或NOT REGEXP。

这两种类型的模式匹配，除了使用的运算符和模式字符不同之外，还有以下两个重要的差异。 

* SQL的LIKE模式只用于匹配**整个字符串**。正则表达式REGEXP可以匹配**字符串的任何部分**。    

* LIKE运算符是**多字节安全**的。REGEXP运算符**只能正确地处理单字节字符集**，并且不会考虑排序规则。

与LIKE运算符配合使用的模式，可以包括通配符"%"和"\_"。

#### 3.5.1.2 运算符优先级

下面按优先级从高到低的顺序列出了各个运算符。同一行的运算符拥有相同的优先级。优先级高的运算符会在优先级低的运算符之前进行计算。优先级相同的运算符则按从左至右的顺序依次进行计算。

```
INTERVAL
BINARY	COLLATE
!
-（负号）	~（按位求反）                                                           
*	/	DIV	%	MOD      
+	-
<<	>>
＆
|
<	<=	=	<=>	<>	!=	>=	>	IN	IS	LIKE	REGEXP	RLIKE
BETWEEN	CASE	WHEN	THEN	ELSE
NOT
AND	&&
XOR
OR	||
:=
```

更改表达式子项的计算顺序，可以使用括号。                                                       

#### 3.5.1.3 表达式里的Null值

在表达式里使用NULL值时要小心，因为其结果往往会出乎你的意料。下面这些点帮助你避免意外情况的发生。     

* 当把NULL用作算术运算符或位运算符的操作数时，其计算结果皆为NULL                                                   

* 当把NULL与逻辑运算符一起使用时，除非其计算结果可以确定，否则其结果皆为NULL

    ```
    1 AND NULL	->	NULL
    1 OR NULL	->	1
    0 AND NULL	->	0
    0 OR NULL	-> NULL     
    ```

* 当把NULL用作比较运算符或模式匹配运算符的操作数时，其计算结果皆为NULL，不过， 运算符＜＝＞、IS NULL和IS NOT NULL例外，因为它们都是专门用于处理NULL值的

    ```
    1 = NULL	->	NULL
    NULL = NULL	->	NULL
    1 <=> NULL	->	0
    NULL LIKE '%'	->	NULL
    NULL REGEXP '.*'	-> NULL
    NULL <=> NULL	->	1
    1 IS NULL	-> 0
    NULL IS NULL	-> 1
    ```

* 当把NULL用作函数参数时，函数通常会返回NULL。不过，那些专门用于处理NULL参数 的函数例外。例如

    * IFNULL()便可以处理NULL参数，并且会根据具体情况返回真值或假值。 另外，
    * STRCMP()期望的是非NULL参数；如果把NULL作为参数传递给它，那么它返回的结果将为NULL, 而不是真值或假值。

* 在排序运算中，NULL值会一起排序。按升序排序在最前面；以降序排序时在最后面。

### 3.5.2 类型转换

只要某个值的类型与上下文所要求的类型不相符，MySQL就会根据操作的类型自动进行类   型转换。发生自动类型转换的场合有以下几种。 

* 把操作数转换成适合运算符计算的相应类型。

* 把由数参数转换成函数所期望的类型。   

* 把某个值转换成适合于存储在列里的另一种类型。

另外，还可以利用类型转换运算符或函数来完成显式类型转换。

可以根据系统变量character_set_connection和collation_connection所指定的字符集和排序规则，将数字转换成字符串或时态值。只有character_set_connection为binary 时，转换结果才会是一个二进制串；否则，结果为非二进制串。

默认情况下，MySQL会尽量把值转换为表达式所需要的类型，而不是产生错误。根据具体上下文的不同，MySQL会让值在三种通用类别（即数字、字符串、日期／时间）之间 进行转换。

不过，值并不是总能从一种类型转换成另外一种类型。如果某个值在类型转换后， 看起来并不像目标类型的合法值，那么这个转换就是失败的。例如

* 把看起来并不像数字的某个值（如字符串'abc')转换为数字类型，那么最终得到的结果将为0。如果某个看起来并不像日期或时间的值转换为日期或时间类型，那么最终得到的结果将为该类型的“零”值。
* 把字符串，abc'转换成日期类型，则得到的结果为“零“日期：'0000-00-00'。

另外，由于任何值都可以处理成字符串，因此把一个值转换成字符串，在通常情况下都不会有问题。

在数据输入操作期间，为防止把非法值转换成最相近的合法值，可以启用严格模式，以便把此类问题当作错误报告出来。

MySQL还会执行某些微小的类型转换。如果在整数上下文中使用浮点数值，那么浮点数值 会被转换（通过四舍五入）。反之亦可，整数可以毫无问题地当作浮点数来使用。

除非上下文明确地表明需要一个数字，否则十六进制常量会被当作二进制串来对待。

在字符串上下文里，每两个十六进制数字会被转换成一个字符，最终结果将为一个字符串，如下列示例所示：

```
0x61	->	'a'
0x61 + 0	->	97
X'61'	->	'a'
X'61'+ 0	->	97
CONCAT (Ox61)	->	'a'
CONCAT(Ox61 + 0)	->	'97'
CONCAT (X'61')	-> 'a'
CONCAT(X'61'+ 0)	->	'97'
```

对于比较操作中的十六进制常量，上下文会根据具体情况来确定是把它当成二进制串，还是把它当成数字。

使用字符串引导符或CONVERT()，可以把十六进制常量强制转换成一个非二进制串。

有些运算符会把操作数强制转换成它所期望的类型，而不管该操作数原来是什么类型。 (数字字符串相加)

对于"字符串到数字"的转换，只是字符串里的某个地方包含一个数字还远远不够。MySQL并不会检查整个字符串，以期望找到某个数字；它只会检查字符串的开头。如果字符串的开头 部分不是数字，那么转换的结果即为0。MySQL的"字符串到数字"的转换规则，适用于把看起来像数字的字符串转换为数字值。

位运算符甚至比算术运算符还严格。它们不仅要求操作数得是数字，而且还要求它们是整     数（根据情况进行相应的类型转换）。这意味着，对于像0.3这样的小数，即使它为非零值， 也不会被当成逻辑真值。因为当它转换成整数时，所得结果为0。

模式匹配运算符希望操作的是字符串。这意味着，可以把MySQL的模式匹配运算符用于数     字，因为在查找匹配的过程中，它会把这些数字转换成字符串。                                                       大小比较运算符(<、<=、＝等）跟上下文密切相关，也就是说，它们会根据操作数的类型     来进行计算。两个操作数都是数字，那就比较数字相对大小；两个操作数都是字符串，那就按照词法进行比较；两个操作数类型不同，那就按数字方式比较。

在进行比较运算时，MySQL会根据需要按以下规则对操作数进行类型转换。

* 除<=>运算符以外，所有涉及NULL值的比较结果都为NULL。（另外，＜＝＞与＝很像，只 是NULL<=> NULL的结果为真，而NULL= NULL的结果为NULL。)
* 如果两个操作数都是字符串，那么它们将按字符串进行词法比较。
    * 对于二进制串，会按字节逐个比较各个对应字节的数值。
    * 对于非二进制串，会使用表示字符串的字符集的排序序列，逐个字符地进行比较。如果字符串使用了不同的字符集，那么这个比较可能导致错误或者产生毫无意义的结果。对于非二进制串和二进制串之间的比较，按二进制串方式进行。
* 如果两个操作数都是整数，那么它们将按整数方式进行数字比较。
* 与数字进行比较的十六进制常量，会按二进制串进行比较。
* 除IN()以外，如果有一个操作数为TIMESTAMP或DATET工ME, 而另一个操作数为常量， 那么两个操作数会按TIMESTAMP值进行比较。这种做法很适合于各种ODBC应用程序 的比较操作。
* 如果有一个操作数为小数，而另一个操作数为小数或整数，那么两个操作数会按小数 进行比较；否则，按双精度浮点值进行比较。
* 除上述情况以外，比较操作中的操作数都将按双精度浮点值进行数字比较。请注意， 字符串与数字之间的比较也属千这种情况。字符串将被转换成一个双精度数，如果字符 串看起来不像数字，那么转换结果将为0。例如，'14.3'将被转换成14.3, 但'L4.3' 会被转换成0。

#### 3.5.2.1 时态值的解释规则

根据表达式中上下文的需要，MySQL会把字符串和数字自动转换成日期/时间值，或者进行相反转换。在数字上下文中，日期/时间值将自动转换成数字；在日期或时间上下文中，数字将自动转换成日期或时间。当把某个值赋给日期或时间类型列，或者当某个函数需要日期或时间值时，会发生到日期或时间类型的转换。在比较操作中，一般的规则是把日期/时间值当作字符串来比较。    

#### 3.5.2.2 测试或强制类型转换

如果想看看某个表达式会发生什么样的类型转换，那么可以在SELECT语句里计算这个表 达式，并检查其结果

```mysql
mysql> SELECT X'41', X'41'+ 0;
```

如果无法直观地看出表达式的类型，那么可以把查询结果放到一个新表里，然后查看这个表的定义

```mysql
mysql> CREATE TABLE t SELECT X'41'AS coll, X'41'+ 0 AS co12;
mysql> DESCRIBE t;
```

对于那些用来删改行的DELETE或UPDATE语句，提前对表达式计算进行测试特别有用，因 为你会想要确保这些语句将只会对你想要进行删改的行产生影响。有一种检查表达式的方式是， 在执行DELETE或UPDATE语句之前，先运行一条带有相同WHERE子句的SELECT语句，以确认该子句选中的行正是你想要的。

在对某个值的使用方式不太确定时，可以利用MySQL的类型转换机制，把表达式强制转换 为某个特定类型的值；或者调用函数执行你需要的转换。下面列出了多项有用的类型转换技术。

* 往表达式里添加子项+0或者+0.0，将其强制转换成数值
* 如果要去掉某个数字的小数部分，那么可以使用函数FLOOR()或CAST()。如果想为某个整数增加小数部分，那么可以让它加上一个带有所需小数位数的精确零值           
* 如果要带有四舍五入，则需要使用ROUND()来代替CAST()
* 使用CAST()或CONCAT()，可以把一个值转换成字符串
* 使用HEX()，可以把数字转换成十六进制字符串，也可以把HEX()用于字符串，把字符串转换成一个由十六进制数字构成的字符串，其中每两个数字对应于原字符串里的一个字节
* 使用ASCII(), 可以把一个单字节字符转换成它的ASCII编码值；如果想根据ASCII码反向转换成字符，则需要使用CHAR()函数
* 使用DATE_ADD()或INTERVAL运算时，会把字符串或者数字强制转换成日期
* 一般情况下，可以采用让日期值加上一个0的方式，把它转换成数字形式
* 带有时间部分的时态值可以转换成带有微秒部分的值
* 如果想去掉小数秒部分，可以把值转换成整数            
* 如果想把字符串从一个字符集转换到另一个，可以使用CONVERT()。如果想检查结果是否 有所要的字符集，可以使用CHARSET()函数
* 在字符串的前面带上字符集引导符，并不会导致该字符串发生转换，但MySQL会把它解 释成那个引导符所指示的字符集
* 如果想确定某个给定十六进制UCS-2字符所对应的UTF-8字符的十六进制值，可以组合使 用函数CONVERT()和HEX()
* 如果想更改字符串的排序规则，可以使用COLLATE运算符。如果想检查结果是否有所要的排序规则，可以使用COLLATION()函数
* 字符集和排序规则必须是兼容的。如果它们不兼容，则可以先用CONVERT()来转换字符集，然后再用COLLATE来更改排序规则
* 如果想把二进制串转换成具有给定字符集的非二进制串，则可以使用CONVERT()
* 另外，对于用引号引起来的二进制串或十六进制串，可以用字符集引导符来更改MySQL 对这种二进制串的解释                   
* 如果想把非二进制串强制转换成二进制串，则可以使用BINARY关键字

## 3.6 选择数据类型

简单地选用某种表示，而 弃用另一种表示，这种做法往往预示着在存储要求、查询处理和整体性能方面存在问题。下面列出了几个在为列挑选类型时需要思考的问题。

* 列要存放什么类型的值？

    如果使用另一种更适合的类型，则有可能获得更好的性能。不过，对要处理的值的类型进行评估，尤其是要评估那些源自其他人的数据的值， 并不是件轻而易举的事情。如果你正在为别人设计表，那么把"列将存放什么样的数据"这样 一个问题询问清楚将非常重要。一定要多问，并获得必要的信息，以便做出最佳的选择。

* 值是否都在某个特定区间内？

    如果它们都是整数，那么是否总为非负值？如果是，则可以考 虑选用UNSIGNED。如果它们是字符串，那么是否总是来自某个固定且有限的值集合？如果是， 则使用ENUM或SET更为合适。

    数据类型的取值范围和空间占用率是相互影响的。实际需要使用多”大”的类型才算合适 呢？对于数字，可以选择一种取值范围有限的小类型，也可以选择一种取值范围很大的大类型。 对千字符串，应该根据它们的长短来选择。如果要存储的值所包含的字符不超过10个，那么不 要选择CHAR(255)。   

* 在性能和效率方面要考虑哪些问题？

    有些类型处理起来的效率比其他类型要高。数字操作的执行速度通常都比字符串操作要快。短字符串的比较速度，比长字符串要快，而且磁盘开销也 更小。对于MyISAM表，长度固定的行的性能，比长度可变的行更好。

尽管你很想在创建表时就做出最好的数据类型选择，但实际选择的类型可能并不是最优的， 可以利用ALTER TABLE语句，把它更改成一种更好的类型。

### 3.6.1 列要存放什么类型的值

选择数据类型首先需要考虑的事情。必须了解数据的本质，才能选出合适的类型。

要从描述的事务中抽象出数据，根据不同数据的特定来选择类型。

用什么数据类型描述降雨量？

身高用什么数据类型比较好？

用什么数据类型表示货币？

电话号码、信用卡号、社保号用什么数据类型？

保存日期信息需要时间部分吗？

### 3.6.2 所有值是否都在某个特定的区间内

如果你已确定了一个大致范畴，计划从中选择列的数据类型，那么多考虑一下所要表示的 值的范围，会有助把于选择缩小到该范畴里的某个特定类型上。

* 通过整数值的范围选择整数数据类型

    * 如果取值0-1000, 那么可以选择SMALLINT-BIGINT之 间的任何一种类型
    * 如果取值范围超过了200万，则不能使用SMALLINT, 可以选择的类型变为 从MEDIUM工NT到BIGINT之间的某一种

    选用足以满足使用目的的最小类型，那么将可以最小化表所使用的存储空间。还将获得更好的性能，较短的列的处理速度更快。当读取较短的值时，所需的磁盘读写操作会更少；还可以把更多的键值放入内存索引缓冲区里， 进而加快索引搜索的速度。如果数据类型不合适还可以使用ALTER TABLE让该列。

* 根据字符串类型的长度来选择字符串类型

    * 如果需要存储的字符串短千256个字符，那么可以 使用CHAR、VARCHAR或TINYTEXT
    * 如果需要存储更长一点的字符串，则可以选用VARCHAR或某种更长的TEXT类型 
    * 如果某个字符串列用于表示某种固定集合的值，那么可以考虑使用数据类型ENUM或SET。它们在内部都被表示为数字，对它们的操作都是按数字方式进行，这使它们的处理效率比其他字符串类型的更高。它们也比其他字符串类型更紧凑，从而可以节省更多空间。此外，如果启用了SQL的严格模式，那么还可以防止那些并未列在 允许值列表里的字符串进入数据库