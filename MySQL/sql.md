# Mysql随笔（待后续整理）

## 条件判断

### null

```mysql
is null
is not null
```

### 单条件判断（IF）

类似于三目运算符

`if(xx,a,b)`

- xx成立则a
- 不成立则b

例子：

```mysql
if(activity_type="start",-timestamp,timestamp)
```

等价于

```mysql
        CASE
            WHEN activity_type = 'start' THEN -timestamp
            ELSE timestamp
        END
```

### 多条件判断（CASE WHEN）

CASE 语句遍历条件并在满足第一个条件时返回一个值（如 IF-THEN-ELSE 语句）。 因此，一旦条件为真，它将停止读取并返回结果。

如果没有条件为真，它将返回 ELSE 子句中的值。

如果没有ELSE部分且没有条件为真，则返回NULL。

```mysql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    WHEN conditionN THEN resultN
    ELSE result
END
```

| 参数                                    | 描述                                        |
| :-------------------------------------- | :------------------------------------------ |
| *condition1, condition2, ...conditionN* | 必需。条件。 它们的评估顺序与列出的顺序相同 |
| *result1, result2, ...resultN*          | 必需。条件为真时返回的值                    |

**实例**

```mysql
SELECT CustomerName, City, Country
FROM Customers
ORDER BY
(CASE
    WHEN City IS NULL THEN Country
    ELSE City
END);
```

### IFNULL

`IFNULL`函数的语法：

```mysql
IFNULL(expression_1,expression_2);
```

如果`expression_1`不为`NULL`，则`IFNULL`函数返回`expression_1`; 否则返回`expression_2`的结果。

注意：`expression_1`可以是某个表的某个字段，也可以是select查出来的某个字段这个整体直接当做`expression1`等等

[176. 第二高的薪水 - 力扣（LeetCode）](https://leetcode.cn/problems/second-highest-salary/description/?envType=study-plan-v2&envId=sql-free-50)

## 查询

### 去重（DISTINCT）

从表中查询数据时，可能会收到重复的行记录。为了删除这些重复行，可以在`SELECT`语句中使用`DISTINCT`子句。

```mysql
SELECT DISTINCT
    columns
FROM
    table_name
WHERE
    where_conditions;
```

### 计数（COUNT）

`COUNT()`函数返回表中的行数。 `COUNT()`函数允许您对表中符合特定条件的所有行进行计数。

```mysql
COUNT(expression)
```

```mysql
select count(xxx) as cnt 
from t
group by xxx
```

### LIMIT

使用`LIMIT`子句来约束结果集中的行数。`LIMIT`子句接受一个或两个参数。两个参数的值必须为零或正整数

下面说明了**两个参数**的`LIMIT`子句语法：

```mysql
SELECT 
    column1,column2,...
FROM
    table
LIMIT offset , count;
```

我们来查看`LIMIT`子句参数：

- `offset`参数指定要返回的第一行的偏移量。第一行的偏移量为`0`，而不是`1`。(可选)
- `count`指定要返回的最大行数。（必选）

当您使用带有**一个参数**的`LIMIT`子句时，此参数将用于确定从结果集的开头返回的最大行数。

```mysql
SELECT 
    column1,column2,...
FROM
    table
LIMIT count;
```

### **IN、NOT IN**

```mysql
select *
from t
where xx not in (
  select xxx
  from xxx
)
```

``````mysql
select *
from t
where xx in (
  select xxx
  from xxx
)
``````

**踩坑**

当使用 `IN` 或 `NOT IN` 操作符时，需要提供一个值列表，例如 `SELECT column_name(s) FROM table_name WHERE column_name IN (value1,value2,...);`。

试图将一个临时表(WITH AS)（`sameInsur` 和 `sameLoc`）用作值列表，这是不允许的！！

这时可以再嵌套一层SELECT取出值列表 `(SELECT * FROM sameInsur)`

### 子查询

MySQL子查询称为内部查询，而包含子查询的查询称为外部查询。 子查询可以在使用表达式的任何地方使用，并且**必须在括号中关闭！！**同时也可保障代码逻辑清晰易读 ovo

当查询执行时，首先执行子查询并返回一个结果集。然后，将此结果集作为外部查询的输入。

![img](http://www.yiibai.com/uploads/images/201707/1807/257110745_37696.png)

### BETWEEN

`BETWEEN` 运算符选择给定范围内的值。 这些值可以是数字、文本或日期。

`BETWEEN` 运算符具有包容性：包括开始值和结束值。（左右都是闭区间）

```mysql
SELECT column_name(s)
FROM table_name
WHERE column_name BETWEEN value1 AND value2;
```

### 模糊查询（LIKE）

`LIKE` 运算符在 `WHERE` 子句中用于搜索列中的指定模式。

有两个通配符经常与 `LIKE` 运算符一起使用：

- 百分号 (%) 表示零个、一个或多个字符
- 下划线符号 (_) 代表一个，单个字符

百分号和下划线也可以组合使用！

**LIKE 语法**

```mysql
SELECT column1, column2, ...
FROM table_name
WHERE columnN LIKE pattern;
```

**提示：**您还可以使用 `AND` 或 `OR` 运算符组合任意数量的条件。

以下是一些示例，展示了带有 '%' 和 '_' 通配符的不同 `LIKE` 运算符：

| LIKE 运算符                    | 描述                                       |
| :----------------------------- | :----------------------------------------- |
| WHERE CustomerName LIKE 'a%'   | 查找以"a"开头的任何值                      |
| WHERE CustomerName LIKE '%a'   | 查找以"a"结尾的任何值                      |
| WHERE CustomerName LIKE '%or%' | 查找在任何位置有"或"的任何值               |
| WHERE CustomerName LIKE '_r%'  | 查找第二个位置有"r"的任何值                |
| WHERE CustomerName LIKE 'a_%'  | 查找以"a"开头且长度至少为 2 个字符的任何值 |
| WHERE CustomerName LIKE 'a__%' | 查找以"a"开头且长度至少为 3 个字符的任何值 |
| WHERE ContactName LIKE 'a%o'   | 查找以"a"开头并以"o"结尾的任何值           |

## 删除

`DELETE`语句用于删除表中已有的记录。

```mysql
DELETE 
FROM xx 
WHERE xx;
```

**注意：**删除表中的记录时要小心！ 请注意 `DELETE` 语句中的 `WHERE` 子句。 `WHERE` 子句指定应该删除哪些记录。 如果省略`WHERE`子句，表中的所有记录都会被删除！

### 自连接-筛选删除

在DELETE官方文档中，给出了这一用法，比如下面这个DELETE语句👇

````mysql
DELETE t1 
FROM t1 
LEFT JOIN t2 
	ON t1.id=t2.id 
WHERE t2.id IS NULL;
````

这种DELETE方式很陌生，竟然和SELETE的写法类似。它涉及到t1和t2两张表，DELETE t1表示要删除t1的一些记录，具体删哪些，就看WHERE条件，满足就删；

这里删的是t1表中，跟t2匹配不上的那些记录。

所以，官方sql中，DELETE p1就表示从p1表中删除满足WHERE条件的记录。

**实例**

```mysql
DELETE p1
FROM Person AS p1 ,Person AS p2
WHERE p1.id > p2.id AND
  p1.email = p2.email
```

[196. 删除重复的电子邮箱 - 力扣（LeetCode）](https://leetcode.cn/problems/delete-duplicate-emails/solutions/219860/dui-guan-fang-ti-jie-zhong-delete-he-de-jie-shi-by/?envType=study-plan-v2&envId=sql-free-50)

## 连接

### 内连接（INNER JOIN）

等价于多表连接通过where来限制筛选条件

join等价于inner join内连接

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201007112127683.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ0NTQ1Mzg=,size_16,color_FFFFFF,t_70)

```mysql
# 内连接(取两表交集)
SELECT * FROM tab1 a inner join tab2 b on a.age = b.age
# 等同于
SELECT * FROM tab1 a ,tab2 b where  a.age = b.age
# 等同于
SELECT * FROM tab1 a join tab2 b on a.age = b.age
```

### **左连接**（LEFT JOIN）

left join xx

on xx   （on后还可跟AND OR等限制），

以左边表各列为基准

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201007113347885.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ0NTQ1Mzg=,size_16,color_FFFFFF,t_70)

```mysql
select 
  xx
from
  Employees eN
left join
  EmployeeUNI eU
on   # 此处的on也可用using(id)代替简化
  eN.id = eU.id  
```

#### on 条件与where 条件

- **使用位置**
  - on 条件位置在join后面
  - where 条件在join 与on完成的后面

- **使用对象**
  - on 的使用对象是被关联表
  - where的使用对象可以是主表，也可以是关联表

- **选择与使用**
  - 主表条件筛选：只能在where后面使用。
  - 被关联表，如果是想缩小join范围，可以放置到on后面。如果是关联后再查询，可以放置到where 后面。 
  - 如果left join 中，where条件有对被关联表的 关联字段的 非空查询，与使用inner join的效果后，在进行where 筛选的效果是一样的。不能起到left join的作用。

**执行顺序问题**

先执行on条件筛选， 在进行join， 最后进行where 筛选

### 笛卡尔积连接（交叉连接）

其实是数学领域的概念，就是对两个集合做乘法（暴力组合）

遍历左表的每一行数据，用左表每一行数据分别于与右表的每一行数据做关联

CROSS JOIN

```mysql
SELECT 
    *
FROM
    Students s
CROSS JOIN
    Subjects sub
```

### 联合查询（Union）

UNION 操作符用于连接两个以上的 SELECT 语句的结果组合到一个结果集合中。多个 SELECT 语句会删除重复的数据。

#### 语法

MySQL UNION 操作符语法格式：

```mysql
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions]
UNION [ALL | DISTINCT]
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions];
```

#### 参数

- **expression1, expression2, ... expression_n**: 要检索的列。

- **tables:** 要检索的数据表。

- **WHERE conditions:** 可选， 检索条件。

- **DISTINCT:** 可选，删除结果集中重复的数据。默认情况下 UNION 操作符已经删除了重复数据，所以 DISTINCT 修饰符对结果没啥影响。

- **ALL:** 可选，返回所有结果集，包含重复数据。

  - **UNION 语句**：用于将不同表中相同列中查询的数据展示出来；（不包括重复数据）

    **UNION ALL 语句**：用于将不同表中相同列中查询的数据展示出来；（包括重复数据）

#### 实例

以下语句组合了从表`t1`和`t2`表返回的结果集：

```mysql
(SELECT id
FROM t1)

UNION

(SELECT id
FROM t2) 
```

最终结果集包含查询返回的单独结果集的不同值：

```mysql
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
|  4 |
+----+
4 rows in set (0.00 sec) 
```

因为值为2和3的行是重复的，所以`UNION`操作员将其删除并仅保留不同的行。 以下维基中说明了来自`t1`和`t2`表的两个结果集的并集：

![MySQL UNION](http://www.begtut.com/wp-content/uploads/2019/08/MySQL-UNION.png)

如果使用`UNION ALL`，则重复行（如果可用）将保留在结果中。因为`UNION ALL`不需要处理重复项，所以它的执行速度比 `UNION DISTINCT` 快。

```mysql
SELECT id
FROM t1
UNION ALL
SELECT id
FROM t2; 
```

运行结果：

```mysql
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
|  2 |
|  3 |
|  4 |
+----+
6 rows in set (0.00 sec) 
```

如您所见，由于`UNION ALL`操作，重复项出现在组合结果集中 。

## 分组（GROUP BY)

GROUP BY 语句根据一个或多个列对结果集进行分组。

在分组的列上我们可以使用 COUNT, SUM, AVG,等函数。

可依据多个属性组合进行分组（例如：`group by sex, agex`）

## 排序（ORDER BY）

ORDER BY 

默认为升序 ASC

可手动声明降序 DESC

可依据多个属性组合进行排序（例如：`order by s1.name DESC, s2.age ASC,s3.score DESC`）

```mysql
ORDER BY s.xx, sub.xx,sub.yy;
```

## **聚合**函数

- **特殊用法：**聚合函数可以进行NULL/空值的转换	
  - 高质量题解解释：[619. 只出现一次的最大数字 - 力扣（LeetCode）](https://leetcode.cn/problems/biggest-single-number/solutions/683252/dang-biao-ge-wei-kong-shi-ru-he-fan-hui-6qpzg/?envType=study-plan-v2&envId=sql-free-50)

### AVG

用于计算一组值或表达式的平均值。

**注意：**表达式很重要，使用起来很方便某些场景。

例如：

- avg（条件）相当于sum（if（条件，1，0））/count(全体)
-  sum（if（条件，N，0））/count(全体) 可用 N*avg（条件）代替

```mysql
AVG(DISTINCT expression)
```

### SUM

计算一组值或表达式的总和

**注意：**表达式很重要，使用起来很方便某些场景。

```mysql
SUM(DISTINCT expression)
```

### HAVING 子句

在group by 之后对一些聚合函数计算出来的结果（例count、avg、sum等）的进一步筛选

where 优先级高于 group by 无法在group by之后进行筛选使用。聚合函数 having 优先级低于 group by，可用于group by之后的筛选。

```mysql
SELECT name 
FROM Employee
WHERE id IN (
  SELECT managerId
  FROM Employee
  GROUP BY managerId
  HAVING COUNT(managerId)>=5
)
```

### MOD

`MOD`函数返回一个数字除以另一个数字的余数。

以下3种方式均可取余

```mysql
MOD(x, y)

x MOD y

x % y
```

## 函数

### 日期函数

#### **日期数据间隔**

DATEDIFF() 函数返回两个日期之间的天数。

**语法**

```mysql
DATEDIFF(date1,date2)
```

*date1* 和 *date2* 参数是合法的日期或日期/时间表达式。（可以是表示时间的正确格式的字符串）

**注释：**只有值的日期部分参与计算。



datediff(日期1, 日期2)

得到的结果是日期1与日期2相差的天数。 如果日期1比日期2大，结果为正（大多少天）；如果日期1比日期2小，结果为负（小多少天）。

```mysql
datediff(w2.date,w1.date)
```

#### 显示日期格式

DATE_FORMAT() 函数用于以不同的格式显示日期/时间数据。

```mysql
DATE_FORMAT(date,format)
```

*date* 参数是合法的日期。*format* 规定日期/时间的输出格式。

可以使用的格式有：

| 格式 | 描述                                           |
| :--- | :--------------------------------------------- |
| %a   | 缩写星期名                                     |
| %b   | 缩写月名                                       |
| %c   | 月，数值                                       |
| %D   | 带有英文前缀的月中的天                         |
| %d   | 月的天，数值(00-31)                            |
| %e   | 月的天，数值(0-31)                             |
| %f   | 微秒                                           |
| %H   | 小时 (00-23)                                   |
| %h   | 小时 (01-12)                                   |
| %I   | 小时 (01-12)                                   |
| %i   | 分钟，数值(00-59)                              |
| %j   | 年的天 (001-366)                               |
| %k   | 小时 (0-23)                                    |
| %l   | 小时 (1-12)                                    |
| %M   | 月名                                           |
| %m   | 月，数值(00-12)                                |
| %p   | AM 或 PM                                       |
| %r   | 时间，12-小时（hh:mm:ss AM 或 PM）             |
| %S   | 秒(00-59)                                      |
| %s   | 秒(00-59)                                      |
| %T   | 时间, 24-小时 (hh:mm:ss)                       |
| %U   | 周 (00-53) 星期日是一周的第一天                |
| %u   | 周 (00-53) 星期一是一周的第一天                |
| %V   | 周 (01-53) 星期日是一周的第一天，与 %X 使用    |
| %v   | 周 (01-53) 星期一是一周的第一天，与 %x 使用    |
| %W   | 星期名                                         |
| %w   | 周的天 （0=星期日, 6=星期六）                  |
| %X   | 年，其中的星期日是周的第一天，4 位，与 %V 使用 |
| %x   | 年，其中的星期一是周的第一天，4 位，与 %v 使用 |
| %Y   | 年，4 位                                       |
| %y   | 年，2 位                                       |

**实例**

下面的脚本使用 DATE_FORMAT() 函数来显示不同的格式。我们使用 NOW() 来获得当前的日期/时间：

```mysql
DATE_FORMAT(NOW(),'%b %d %Y %h:%i %p')
DATE_FORMAT(NOW(),'%m-%d-%Y')
DATE_FORMAT(NOW(),'%d %b %y')
DATE_FORMAT(NOW(),'%d %b %Y %T:%f')
```

结果类似：

```mysql
Dec 29 2008 11:45 PM
12-29-2008
29 Dec 08
29 Dec 2008 16:25:46.635
```

### 字符串函数

#### **长度**

```mysql
where CHAR_LENGTH(xxx)>15
where CHAR_LENGTH(xxx)>=15
where CHAR_LENGTH(xxx)!=15
where CHAR_LENGTH(xxx)=15
```

#### 拼接

CONCAT() 函数将两个或多个表达式相加。

**语法**

```mysql
CONCAT(expression1, expression2, expression3,...)
```

**参数值**

| 参数                                          | 描述                                                         |
| :-------------------------------------------- | :----------------------------------------------------------- |
| *expression1, expression2, expression3, etc.* | 必需。要加在一起的表达式。**注意：** 如果任何表达式为NULL值，则返回NULL |

**实例**

将几个字符串加在一起：

```mysql
SELECT CONCAT("SQL ", "Tutorial ", "is ", "fun!") AS ConcatenatedString;
```

#### 命令汇总

- `CONCAT(str1, str2) :` 拼接字符串
- `UPPER(str) :` 字符串变成大写
- `LOWER(str) :` 字符串变成小写
- `LENGTH(str) :` 获取字符串的长度
- `LEFT(str,len) :` 获取字符串左边 len 个字符
- `RIGHT(str,len) :` 获取字符串右边 len 个字符
- `SUBSTR(str,start,len) :` 获取 str 中从 start 开始的 len 个字符
- `LOCATE(substr, str, start)`：
  - 判断substr在str中第一次出现的位置，找不到则输出0。start为起始位置，默认为1。（mysql中字符串起始下标从1开始计算）
- `GROUP_CONCAT`:
  - 以指定分隔符指定排序 拼接多个字符串
  - `group_concat([DISTINCT] 要连接的字段 [Order BY ASC/DESC 排序字段] [Separator '分隔符'])`
  - 默认为升序ASC，分隔符为","
  - [MYSQL GROUP_CONCAT函数 - 简书 (jianshu.com)](https://www.jianshu.com/p/7a1df0ce6d00)
  - [1484. 按日期分组销售产品 - 力扣（LeetCode）](https://leetcode.cn/problems/group-sold-products-by-the-date/description/?envType=study-plan-v2&envId=sql-free-50)

### 数值函数

#### 四舍五入

```mysql
ROUND(number, decimals)
```

| 参数       | 描述                                                         |
| :--------- | :----------------------------------------------------------- |
| *number*   | 必需。要四舍五入的数字                                       |
| *decimals* | 可选。*number* 要四舍五入的小数位数。 如果省略，则返回整数（无小数） |

#### **最小值**

MIN() 函数返回一组值中的最小值。

```mysql
MIN(expression)
```

| 参数         | 描述                           |
| :----------- | :----------------------------- |
| *expression* | 必需。数值（可以是字段或公式） |

#### 最大值

MAX() 函数返回一组值中的最大值。

```mysql
MAX(*expression*)
```

| 参数         | 描述                           |
| :----------- | :----------------------------- |
| *expression* | 必需。数值（可以是字段或公式） |

### 窗口函数

[MySQL 窗口函数 | 新手教程 (begtut.com)](https://www.begtut.com/mysql/mysql-window-functions.html)

调用窗口函数的一般语法如下：

```mysql
window_function_name(expression) 
    OVER (
        [partition_defintion]
        [order_definition]
        [frame_definition]
    ) 
```

在这个语法中：

- 首先，指定窗口函数名称，后跟表达式。
- 其次，指定`OVER`具有三个可能元素的子句：分区定义，顺序定义和帧定义。

`OVER`子句后面的开括号和右括号是强制性的，即使没有表达式，例如：

```mysql
window_function_name(expression) OVER()
```

**实例**

```mysql
+----------------+-------------+--------+
| sales_employee | fiscal_year | sale   |
+----------------+-------------+--------+
| Alice          |        2016 | 150.00 |
| Alice          |        2017 | 100.00 |
| Alice          |        2018 | 200.00 |
| Bob            |        2016 | 100.00 |
| Bob            |        2017 | 150.00 |
| Bob            |        2018 | 200.00 |
| John           |        2016 | 200.00 |
| John           |        2017 | 150.00 |
| John           |        2018 | 250.00 |
+----------------+-------------+--------+
9 rows in set (0.01 sec)
```

```mysql
SELECT 
    fiscal_year, 
    sales_employee,
    sale,
    SUM(sale) OVER (PARTITION BY fiscal_year) total_sales
FROM
    sales; 
    
    
+-------------+----------------+--------+-------------+
| fiscal_year | sales_employee | sale   | total_sales |
+-------------+----------------+--------+-------------+
|        2016 | Alice          | 150.00 |      450.00 |
|        2016 | Bob            | 100.00 |      450.00 |
|        2016 | John           | 200.00 |      450.00 |
|        2017 | Alice          | 100.00 |      400.00 |
|        2017 | Bob            | 150.00 |      400.00 |
|        2017 | John           | 150.00 |      400.00 |
|        2018 | Alice          | 200.00 |      650.00 |
|        2018 | Bob            | 200.00 |      650.00 |
|        2018 | John           | 250.00 |      650.00 |
+-------------+----------------+--------+-------------+
9 rows in set (0.02 sec)。
```

## 临时表（WITH AS）

如果一整句查询中**多个子查询都需要使用同一个子查询**的结果，那么就可以用with as，将共用的子查询提取出来，加个别名。后面查询语句可以直接用，对于大量复杂的SQL语句起到了很好的优化作用。

使得大规模的sql语句可读可维护，结构更清晰

**注意：**

- 相当于一个临时表，但是不同于视图，不会存储起来，要与select配合使用。
- 同一个select前可以有多个临时表，写一个with就可以，用逗号隔开，最后一个with语句不要用逗号。
- with子句要用括号括起来。
- 当使用 `IN` 或 `NOT IN` 操作符时，需要提供一个值列表，例如 `SELECT column_name(s) FROM table_name WHERE column_name IN (value1,value2,...);`。然而在你的查询中，你试图将一个临时表（`sameInsur` 和 `sameLoc`）用作值列表，这是不允许的！！
  - 这时可以再嵌套一层取出值列表 `(SELECT * FROM sameInsur)`

**实例：**

```mysql
WITH p1 AS (
  SELECT DISTINCT product_id
  FROM Products
),
p2 AS ( 
  SELECT product_id,new_price
  FROM Products
  WHERE (product_id,change_date) IN (
    xxx
  )
)

SELECT p1.product_id,IFNULL(p2.new_price,10) AS price
FROM  p1
LEFT JOIN p2
  ON p1.product_id  = p2.product_id
```

## 别名

通过使用 SQL，可以为表名称或列名称指定别名。

基本上，创建别名是为了让列名称的可读性更强。

### 列的 SQL 别名语法

```mysql
SELECT column_name AS alias_name
FROM table_name;
```

注意：`column_name `也可以传入一个字符串值，如“Low salary”，则会新增一列数据，列名为`alias_name`，值全为`column_name`

### 表的 SQL 别名语法

```mysql
SELECT column_name(s)
FROM table_name AS alias_name;
```