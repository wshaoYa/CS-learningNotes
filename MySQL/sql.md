# Mysql随笔（待后续整理）

（除最基本的知识外，刷题查漏补缺遇到的一些sql技巧记录）

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

### 去重

DISTINCT

```sql
select distinct name
from xx
order by name
```

### 计数

COUNT()

```sql
select count(xxx) as cnt 
from t
group by xxx
```

### **in、not in**

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

## 连接

### 内连接

```sql
select * from T1 inner join T2 on T1.userid=T2.userid
```

### **左连接**

left join xx

on xx   （on后还可跟AND OR等限制）

以左边表各列为基准

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

CROSS JOIN

```sql
SELECT 
    *
FROM
    Students s
CROSS JOIN
    Subjects sub
```

## 排序

order by

默认为升序 asc

可手动声明降序 desc

```sql
ORDER BY s.xx, sub.xx;
```

## **聚合**函数

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

datediff(日期1, 日期2)

得到的结果是日期1与日期2相差的天数。 如果日期1比日期2大，结果为正；如果日期1比日期2小，结果为负。

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