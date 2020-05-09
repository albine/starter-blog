---
title: Pecularities of MySQL User-defined Variables
date: 2020-05-09T07:07:22.911Z
image: /images/riot-photos-madrid-spain.jpg
tags:
  - MySQL
draft: false
---
*High Performance MySQL* 第253页有个有趣的例子：

```sql
set @rownum := 0;
select actor_id, (@rownum := @rownum + 1) as cnt
from actor
where @rownum <= 1
```

结果会输出多少行呢？乍一看似乎只有一行，实际上的结果：

| actor_id | cnt
| --- | --- |
| 58 | 1 |
| 92 | 2 |

我们可以看到第一次`WHERE`判断的时候，`@rownum`还等于0。这说明自定义变量更新的时间在`WHERE`判断之后。至于自定义变量更新到底是在什么时候，手册中写道：

> In a SELECT statement, each select expression is evaluated only when sent to the client. This means that in a HAVING, GROUP BY, or ORDER BY clause, referring to a variable that is assigned a value in the select expression list does not work as expected: [\[1\]](https://dev.mysql.com/doc/refman/5.7/en/user-variables.html)

我觉得这个“when sent to the client”不是很清楚。考虑SQL子句的执行顺序[\[3\]](https://www.eversql.com/sql-order-of-operations-sql-query-order-of-execution/)：

1. FROM
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT
6. DISTINCT
7. UNION
8. ORDER BY
9. LIMIT & OFFSET
我们不妨先认为自定义变量更新在所有这些子句执行之后。这样的话，上面这个结果还是很显然的。

那如果我们加上一个`ORDER BY`会怎么样呢？问题来了，`ORDER BY`哪一行呢？对于有索引的行，走索引的话是自带排序的。走   filesort [\[2\]](https://www.percona.com/blog/2009/03/05/what-does-using-filesort-mean-in-mysql/)的话，全表扫描完之后还要排序。

```sql
set @rownum := 0;
select actor_id, (@rownum := @rownum + 1) as cnt
from actor
where @rownum <= 1
order by first_name; # A. 这行没有索引，最终执行走Using where; Using filesort
# order by last_name; # B. 这行有索引，最终执行走Using where; Using index
```

A的输出：
| actor_id | cnt
| --- | --- |
|71	|1
|132|2
|165|3
|173|4
|125|5
|146|6
|29	|7
|...|...
一共输出200行，是不是还……蛮酷的😂。而B的输出和前面没什么区别：
| actor_id | cnt
| --- | --- |
| 58 | 1 |
| 92 | 2 |
让我们尝试着解释一下。

filesort的过程如下，参考[\[3\]](http://s.petrunia.net/blog/?p=24)。

1. “w/o filesort”
>开始循环（走第一张表的索引）
>
>按`WHERE`的条件进行过滤
>
>join剩下的表，`SELECT`到结果集
>
>结束循环
   
2. “Using filesort”
>开始循环（逐行扫描第一张表）
>
>按`WHERE`的条件进行过滤
>
>结束循环
>
>`ORDER BY`进行排序
>
>join剩下的表，`SELECT`到结果集

3. “Using temporary; Using filesort”
>开始循环（逐行join所有的表）
>
>按`WHERE`的条件进行过滤
>
>`SELECT`到临时表中
>
>结束循环
>
>`ORDER BY`进行排序，输出到结果集

所以问题的关键就在NLJ的循环里有没有更新变量。再次考虑手册中的
>when sent to the client

*High Performance MySQL* 第228页讲到：

> As soon as MySQL processes the last table and generates one row successfully, it can and should send that row to the client.

于是我们猜想，只要一行输出到结果集之后，变量就会更新。那么我们就能清楚地看到，A中的`WHERE`条件是在所有的act_id已经排序结束之后才更新的，于是选中了所有行。而B呢，在循环里就join然后输出结果了，于是`WHERE`条件也是每行都更新的。所以和前面不带排序的结果是一样的，因为它们实际上走了同一个索引，执行起来完全是一样的。

---

那么如果把更新自定义变量放到`WHERE`语句里会怎么样？如果用**last_name**列排序呢？

```sql
set @rownum := 0;
select actor_id, @rownum as cnt
from actor
where (@rownum := @rownum + 1) <= 1
order by first_name; # A. Using where; Using filesort
# order by last_name; # B. Using where; Using index
```
A:
| actor_id | cnt
| --- | --- |
| 1 | 200 |
B:
| actor_id | cnt
| --- | --- |
| 58 | 1 |
A的结果就更有趣了。因为是filesort，所以逐行扫描，第一行是满足`WHERE`条件的，然而之后的都不满足被过滤掉，然而变量的值还是一直在`WHERE`中更新。于是结束之后排序也没什么可排，只有第一行。再join变量的时候，变量已经成了200。

B的结果和A区别，还是前面所说循环里更新和循环外更新的区别，就不再赘述了。

书中还有如下的语句：

```sql
set @rownum := 0;
explain select actor_id, @rownum as cnt
from actor
where (@rownum := @rownum + 1) <= 1
order by first_name, least(0, @rownum := @rownum + 1);
```

结果是只会出现一行，cnt值是2。有趣的是`EXPLAIN`发现这次排序是 Using where; **Using temporary**; Using filesort ，这或许是因为`ORDER BY`中的条件不是常量[\[3\]](http://s.petrunia.net/blog/?p=24)。因为这一点，即使排序的这行有索引排序也不会用到[\[4\]](https://dev.mysql.com/doc/refman/8.0/en/order-by-optimization.html)。