---
title: Query Optimizer Review
date: 2020-05-09T07:07:22.911Z
image: /images/lutetia-to-roma.jpg
tags:
  - MySQL
  - Database
  - Query Optimizer
draft: true
---
这篇文章是*Database System Concepts*第15、16章，*High Performance MySQL*第6章，*Architecture of a Databse System*第4节的总结笔记。
### 查询优化器干什么？
从SQL语句开始：
```sql
SELECT salary
from instructor
where salary < 75000
```
前处理器负责把SQL语句转化成某种中间表示，比如关系代数式：

$$\sigma_{salary < 75000}(\Pi(instructor))$$

或者

$$\Pi_{salary}(\sigma_{salary < 75000}(instructor))$$

然后**查询优化器Query Optimazor**负责从中间表示产生出较优的执行计划EP：比如在上面的例子里，选择算符是全表扫描还是有索引可用？我们把关系代数式加上操作叫做一个**执行原语 Evaluation Primitive**。一个**执行计划 Execution Plan**是执行原语的图。

然后**执行引擎**执行这个执行计划。
### 什么是优化？
我们优化的首要方向是时间。每个执行原语都需要一定时间执行，如CPU时间，磁盘I/O时间，网络时间等。不同的执行计划的执行时间不同，优化器需要找到时间较短的执行计划。

对于执行原语，可能的执行时间包括CPU，磁盘，网络等：
#### CPU时间

Database System Concepts里举Pg的例子（692页）：
> (i) a CPU cost per tuple, (ii) a CPU cost for processing each index entry (in addition to the I/O cost), and (iii) a CPU cost per operator or function

#### 磁盘I/O时间
当然最重要的还是I/O时间，分为**顺序传输block transfer**和**随机I/O寻址 random I/O access**两部分。下面是一些设备的特征时间参数：

|磁盘+接口类型|4Kb block transfer时间|random I/O access时间|
|---|---|---|
|magnetic|0.1ms|4ms
|mid-range SSD + SATA|10us|90us
|SSD + PCIe 3.0|2us|20-60us

然而响应时间受系统具体情况影响很大，比如：
1. 实际上现在都默认几个G的缓存，如果本次查询在内存里完成，就很快。
2. 磁盘具体的存储情况不同
3. 其他负载的竞争
   
所以优化器一般只优化**资源开销cost**，i.e. 多少次读盘，而非具体的响应时间。

我们会具体看看以下算符的开销。
1. 选择
2. 排序
3. 连接

对于执行计划，我们可以采用临时表materialization和流水线pipelining两种方法来从原语的开销计算执行计划的开销。

然后我们还需比较不同的执行计划。
