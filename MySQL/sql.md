# Mysql随笔（待后续整理）

## 判断

### null

```sql
is null
is not null
```

### IF

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

### 联合查询（Union)

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

## 连接

### 内连接（INNER JOIN）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201007112127683.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ0NTQ1Mzg=,size_16,color_FFFFFF,t_70)

```sql
select * from T1 inner join T2 on T1.userid=T2.userid
```

### **左连接**（LEFT JOIN）

left join xx

on xx   （on后还可跟AND OR等限制）

以左边表各列为基准

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201007113347885.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ0NTQ1Mzg=,size_16,color_FFFFFF,t_70)

```sql
select 
  xx
from
  Employees eN
left join
  EmployeeUNI eU
on 
  eN.id = eU.id
```

### 笛卡尔积连接（交叉联接）

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

## 分组

GROUP BY 语句根据一个或多个列对结果集进行分组。

在分组的列上我们可以使用 COUNT, SUM, AVG,等函数。

## 排序

order by

默认为升序 asc

可手动声明降序 desc

```sql
ORDER BY s.xx, sub.xx;
```

## BETWEEN

`BETWEEN` 运算符选择给定范围内的值。 这些值可以是数字、文本或日期。

`BETWEEN` 运算符具有包容性：包括开始值和结束值。（左右都是闭区间）

```mysql
SELECT column_name(s)
FROM table_name
WHERE column_name BETWEEN value1 AND value2;
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