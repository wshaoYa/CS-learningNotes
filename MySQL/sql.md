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

```sql
select distinct name
from xx
order by name
```

### 计数

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

**注意：**表达式很重要，使用起来很方便某些场景

```mysql
AVG(DISTINCT expression)
```

### SUM

计算一组值或表达式的总和

```mysql
SUM(DISTINCT expression)
```

### HAVING

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

## 特殊函数

### **字符串长度**

```sql
where CHAR_LENGTH(xxx)>15
where CHAR_LENGTH(xxx)>=15
where CHAR_LENGTH(xxx)!=15
where CHAR_LENGTH(xxx)=15
```

### **日期数据间隔**

datediff(日期1, 日期2)

得到的结果是日期1与日期2相差的天数。 如果日期1比日期2大，结果为正；如果日期1比日期2小，结果为负。

```sql
datediff(w2.date,w1.date)
```

### 四舍五入

```mysql
ROUND(number, decimals)
```

| 参数       | 描述                                                         |
| :--------- | :----------------------------------------------------------- |
| *number*   | 必需。要四舍五入的数字                                       |
| *decimals* | 可选。*number* 要四舍五入的小数位数。 如果省略，则返回整数（无小数） |

