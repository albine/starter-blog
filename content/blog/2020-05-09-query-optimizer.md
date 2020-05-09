---
title: a Query Optimizer Review
date: 2020-05-09T07:07:22.911Z
image: /images/lutetia-to-roma.jpg
tags:
  - MySQL
  - Database
  - Query Optimizer
draft: false
---

从SQL语句开始：
```sql
SELECT salary
from instructor
where salary < 75000
```
前处理器负责把SQL语句转化成关系代数式：
$$\sigma_{salary < 75000}(\Pi(instructor))$$
或者
$$\Pi_{salary}(\sigma_{salary < 75000}(instructor))$$
然后查询优化器QO负责从中间表示产生出较优的执行计划EP。