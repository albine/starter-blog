---
title: SQL Query Problems Hoard
date: 2020-05-09T07:07:22.911Z
image: /images/lutetia-to-roma.jpg
tags:
  - MySQL
  - SQL
draft: false
---
我们用sakira库来练习：

1. Top N问题：在每个分组中进行排序，并取前N个

在分好组后，我们需要row_number或者rank函数，MySQL的话似乎是8.0里才有。我们用自定义变量实现一个。

例子：对每个演员，输出其总票房最高的前三名电影。

```sql
set @rank := 0;
# 要注意@type的数据类型
set @type := 1; 
select tmp.actor_id, tmp.film_id, tmp.total
from (
    select a.actor.id, f.film_id, sum(p.amount) as total,
    @rank := if(@type = a.actor_id, @rank := @rank + 1, 1) as rnk,
    @type := a.actor_id
    from actor a 
        join film_actor fa on a.actor_id = fa.actor_id
        join film f on fa.film_id = f.film_id
        join inventory i on f.film_id = i.film_id
        join rental r on i.inventory_id = r.inventory_id
        join payment p on r.rental_id = p.rental_id
    # 分组
    group by a.actor_id, f.film_id 
    order by a.actor_id, total desc
) as tmp
where tmp.rnk <= 3;
```

2. `group_concat`
   
3. 做集合的差A - B

用A left join b，然后应用b为空的条件
```sql
select r1.*
from (
    select st.*
    from Student st join Score s1 on st.s_id = s1.s_id
    where s1.c_id = '01') as r1 left join (
        select st.s_id
        from Student st join Score s1 on st.s_id = s1.s_id
        where s1.c_id = '02') as r2
on r1.s_id = r2.s_id
where r2.s_id is null;
```

4. 分页
分页问题就是从排序好的序列中找出第N个到第N+S个。
首先要排序
1. 要排序的列必须加索引
2. 排序条件必须加主键（否则相等的行顺序不确定）
offset方法
对于较小的页数，可以`limit @size offset @discard`或者`limit @discard, @size`
优点：
1. 可以直接找到目标页
缺点：
1. 由于sql标准规定offset工作的方式是先排序再把@discard这么多行扔掉，于是在页数较大的时候性能很差。
seek方法
我们用前一页最后一项的值来查询其后@size行
```sql
select ...
from ...
where ( 
    distance > ? or (distance = ? and shop_id < ?)
)
order by distance asc, shop_id asc
limit ?
```
优点：
1. 性能好
2. 特别适合无限滚动的逻辑
缺点：
1. 没法直接跳到指定页
2. “上一页”的操作得重新写（反向排序，用本页第一个值查询）

具体实现要和需求确定。
1. 最大页数，超过该页数不可浏览
2. 只有当前页的后10页可直接跳转

注意各种框架生成的分页代码往往都是offset方法
mybatis pagehelper:
https://github.com/pagehelper/Mybatis-PageHelper/issues/474

参考：https://use-the-index-luke.com/no-offset