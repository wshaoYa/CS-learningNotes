# Mysql随笔（待后续整理）

## 条件判断

### null

```sql
is null
is not null
```

### 单条件判断（IF）

类似于三目运算符

`if(xx,a,b)`

- xx成立则a
- 不成立则b

例子：

```sql
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

```sql
IFNULL(expression_1,expression_2);
```

如果`expression_1`不为`NULL`，则`IFNULL`函数返回`expression_1`; 否则返回`expression_2`的结果。

## 查询

### 去重（DISTINCT）

从表中查询数据时，可能会收到重复的行记录。为了删除这些重复行，可以在`SELECT`语句中使用`DISTINCT`子句。

```sql
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

```sql
select count(xxx) as cnt 
from t
group by xxx
```

### LIMIT 子句

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

```sql
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

### 子查询

MySQL子查询称为内部查询，而包含子查询的查询称为外部查询。 子查询可以在使用表达式的任何地方使用，并且必须在括号中关闭。

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



## 连接

### 内连接（INNER JOIN）

等价于多表连接通过where来限制筛选条件

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201007112127683.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ0NTQ1Mzg=,size_16,color_FFFFFF,t_70)

```sql
# 内连接(取两表交集)
SELECT * FROM tab1 a inner join tab2 b on a.age = b.age
# 等同于
SELECT * FROM tab1 a ,tab2 b where  a.age = b.age
```

### **左连接**（LEFT JOIN）

left join xx

on xx   （on后还可跟AND OR等限制），

以左边表各列为基准

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201007113347885.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ0NTQ1Mzg=,size_16,color_FFFFFF,t_70)

```sql
select 
  xx
from
  Employees eN
left join
  EmployeeUNI eU
on   # 此处的on也可用using(id)代替简化
  eN.id = eU.id  
```

### 笛卡尔积连接（交叉连接）

其实是数学领域的概念，就是对两个集合做乘法（暴力组合）

遍历左表的每一行数据，用左表每一行数据分别于与右表的每一行数据做关联

CROSS JOIN

```sql
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

```
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
SELECT id
FROM t1
UNION
SELECT id
FROM t2; 
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

## 排序（ORDER BY）

ORDER BY 

默认为升序 ASC

可手动声明降序 DESC

```sql
ORDER BY s.xx, sub.xx;
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

```sql
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

```
DATEDIFF(date1,date2)
```

*date1* 和 *date2* 参数是合法的日期或日期/时间表达式。（可以是表示时间的正确格式的字符串）

**注释：**只有值的日期部分参与计算。



datediff(日期1, 日期2)

得到的结果是日期1与日期2相差的天数。 如果日期1比日期2大，结果为正（大多少天）；如果日期1比日期2小，结果为负（小多少天）。

```sql
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

```
DATE_FORMAT(NOW(),'%b %d %Y %h:%i %p')
DATE_FORMAT(NOW(),'%m-%d-%Y')
DATE_FORMAT(NOW(),'%d %b %y')
DATE_FORMAT(NOW(),'%d %b %Y %T:%f')
```

结果类似：

```
Dec 29 2008 11:45 PM
12-29-2008
29 Dec 08
29 Dec 2008 16:25:46.635
```

### 字符串函数

#### **字符串长度**

```sql
where CHAR_LENGTH(xxx)>15
where CHAR_LENGTH(xxx)>=15
where CHAR_LENGTH(xxx)!=15
where CHAR_LENGTH(xxx)=15
```

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