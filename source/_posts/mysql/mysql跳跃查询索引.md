---
title: MySQL 8.0 索引跳跃扫描
tag: mysql
---

# MySQL 8.0 索引跳跃扫描

[**MySQL**](https://cloud.tencent.com/product/cdb?from=10680) **8.0 实现了Index skip scan，翻译过来就是索引跳跃扫描。**熟悉ORACLE的朋友是不是发现越来越像ORACLE了？再者，熟悉MySQL 5.7 的朋友是不是觉得这个很类似当时优化器的选项MRR？好了，先具体说下什么 ISS，我后面全部用 ISS 简称。

**考虑以下的场景：**

表t1有一个联合索引idx_u1(rank1,rank2)，但是查询的时候却没有rank1这列，只有rank2。比如，select * from t1 where rank2 = 30。

那以前遇到这样的情况，如果没有针对rank2这列单独建立普通索引，这条SQL怎么着都是走的FULL TABLE SCAN。

ISS 就是在这样的场景下产生的。**ISS 可以在查询过滤组合索引不包括最左列的情况下，走索引扫描，而不必要单独建立额外的索引。**因为毕竟额外的索引对写开销很大，能省则省。

**还是拿刚才的例子来讲，假设：**

表t1的两个字段rank1,rank2。有这样的记录，

![img](.\media\1620.jpg)

我们给出的SQL是，

```javascript
select * from t1 where rank2 >400,
```

复制

那MySQL通过ISS把这条SQL变为，

```javascript
select * from t1 where rank1=1 and rank2 > 400union allselect * from t1 where rank1 = 5 and rank2 > 400;
```

复制

可以看出来，MySQL其实内部自己把左边的列做了一次DISTINCT，完了加进去。

**我们拿实际的例子来看下。假设：**

还是刚才描述那张表，rank1字段值的distinct值比较少。查询计划的对比，

![img](.\media\1621.jpg)

关闭 ISS，

很显然，ISS 扫描的行数要比之前的少很多。

**ISS其实恰好适合在这种左边字段的唯一值较少的情况下，效率来的高。**比如性别，状态等等。

**那假设：rank1字段的distinct值比较多呢？**

我们重新造了点数据，这次，rank1的唯一值个数有快上万个。

![img](.\media\1623.jpg)

我们来再次看一遍这样SQL的执行计划，

![img](.\media\1624.jpg)

![img](.\media\1625.jpg)

这次我们发现，无论如何MySQL也不会选择 ISS，而选了FULL INDEX SCAN。

那这样的场景就必须给rank2加一个单独索引了。

![img](.\media\1626.jpg)

**Index Skip Scan限制条件：**

1. 查询只能涉及一张表，多表关联无法使用该特性
2. 查询SQL不能使用 GROUP BY 或者 DISTINCT子句
3. **查询字段必须是索引中的字段**
4. 组合索引形式：([A_1, …, A_k,] B_1, …, B_m, C [, D_1, …, D_n])，A，D 可以为空，但是B ,C 不能为空

**参数开关：**

通过设置参数optimizer_switch中的skip_scan=on/off，来控制是否打开该优化策略，默认打开。

那来总结下 ISS 就是一句话：**ISS 其实就是MySQL 8.0推出的适合联合索引左边列唯一值较少的情况的一种优化策略。**