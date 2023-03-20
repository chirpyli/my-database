
### Greenplum执行计划分析
我们已经对PostgreSQL的执行计划比较熟了，那么在分布式数据库下，SQL是如何执行的呢，我们通过分析Greenplum来看一下MPP类型的数据库是如何进行处理的。

> Greenplum支持列存与ORCA优化器，这里我们先不去进行细致的分析，后续再进行分析。


#### Greenplum工作机制简析
我们在执行计划分析前，先看一下Greenplum的工作原理。在进行海量数据分析时，单机数据库能够承载的数据量是有限的，并且其计算能力也是受限与单机处理能力。为了进行海量的数据分析，其中一个思路就是一台机器不行，我就让更多的机器参与，将数据分布在多台机器上，多台机器同时进行运算，这样数据量和计算能力都得到了水平扩展。这就是MPP数据库的思想，分而治之。

GP是由一个master节点，多个segments节点构成，master节点负责执行计划解析以及分发、将segment节点的数据汇集等工作，segments承担数据存储与计算的工作。我们以最简单的表存储与查询为例：GP在建表时通过分布键将表切分分布在多个segment中，这个segment可以理解为一个PostgreSQL数据库实例，这样表的大小不再首先与单机PostgreSQL数据库能承载的大小，表数据非常大时，可通过增加节点数量解决。查询时，每个segment将各自节点数据进行查询，汇总在master节点，由master节点汇集返回给用户。

因为master节点对于我们理解GP的工作机制非常重要，对此我们先列出master节点的工作，方便我们接下来理解执行计划：
- 执行计划解析及分发
- 将子节点的数据汇集到一起
- 将所有segment的有序数据进行归并操作
- 聚合函数在master节点上进行最后的计算
- 需要有唯一的序列的功能（如开窗函数不带partiton by子句）

需要注意的是，Greenplum优化器有2个，一个是PG的优化器，另一个是GPORCA，关于ORCA我们后续再进行分析，这里先分析PG优化器的情况。
```sql
-- optimizer = on, 则表示开启了ORCA优化器
postgres=# show optimizer;
 optimizer 
-----------
 off
(1 row)
```

#### 分布表、分布键 
我们先建立一个分布表，
```sql
-- 建立分布表，分布键为a，分布方式默认为hash分布
postgres=# create table t1(a int primary key, b int) distributed by(a);
CREATE TABLE
-- 插入数据
postgres=# insert into t1 select s,s from generate_series(1,1000) s;
INSERT 0 1000

postgres=# create table t2(a int primary key, b int) distributed by(a);
CREATE TABLE
postgres=# insert into t2 select s,s from generate_series(1,1000) s;
INSERT 0 1000
```

#### 全表扫描

我们先看最简单的全表扫描的是如何实现的，其逻辑应该是什么呢？先在各segment节点进行顺序扫描，再在master节点进行汇总。我们看一下其执行计划：
```sql
postgres=# explain select * from t1;
                                   QUERY PLAN                                   
--------------------------------------------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)  (cost=0.00..21.02 rows=1001 width=8)
   ->  Seq Scan on t1  (cost=0.00..6.00 rows=500 width=8)
 Optimizer: Postgres query optimizer    -- PG优化器，非ORCA优化器
(3 rows)
```

这个是基础，相比PG的执行计划，多了一个`Gather Motion`的步骤，即聚合操作，在master节点上将segment节点的数据聚合起来。我们可以通过gp_segment_id，查看数据是位于那个segment上的：
```sql
-- 查询，可以看到，分布在不同节点的数据
postgres=# select gp_segment_id,* from t1 where mod(a, 100) = 0;
 gp_segment_id |  a   |  b   
---------------+------+------
             1 |    0 |    0
             1 |  200 |  200
             1 |  300 |  300
             1 |  400 |  400
             1 |  900 |  900
             0 |  100 |  100
             0 |  500 |  500
             0 |  600 |  600
             0 |  700 |  700
             0 |  800 |  800
             0 | 1000 | 1000
(11 rows)
```
多次执行，其结果的顺序并与固定不变的，原因是，master节点在聚合segment节点的数据时，不同segment节点数据到master节点的先后顺序不确定，master采用的策略是谁先到谁的数据先放在master上。
```sql
postgres=# select gp_segment_id,* from t1 where mod(a, 100) = 0;
 gp_segment_id |  a   |  b   
---------------+------+------
             0 |  100 |  100
             0 |  500 |  500
             0 |  600 |  600
             0 |  700 |  700
             0 |  800 |  800
             0 | 1000 | 1000
             1 |    0 |    0
             1 |  200 |  200
             1 |  300 |  300
             1 |  400 |  400
             1 |  900 |  900
(11 rows)

```

#### 分布式执行计划中Limit怎么实现
在分布式执行计划中，Limit怎么实现呢？先不看执行计划，自己先想一下，思路就有了，那就是，每个segment节点先执行一遍Limit，再聚合到master节点，在maste节点中再次执行一次Limit。我们看一下执行计划，确实如此。
```sql
postgres=# explain select gp_segment_id,* from t1 limit 5;
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Limit  (cost=0.60..0.71 rows=5 width=12)
   ->  Gather Motion 2:1  (slice1; segments: 2)  (cost=0.60..0.81 rows=10 width=12)
         ->  Limit  (cost=0.60..0.66 rows=5 width=12)
               ->  Seq Scan on t1  (cost=0.00..6.00 rows=500 width=12)
 Optimizer: Postgres query optimizer
(5 rows)
```
这次例子比较简单，却对我们理解分布式执行计划非常重要，因为其他的很多SQL操作也是同样的思路。比如聚合函数
。我们以count函数为例，其思路就是先在segment节点计算各自的count，再聚合到master节点，由master节点完成最后的`Finalize Aggregate`。
```sql
postgres=# explain select count(*) from t1;
                                    QUERY PLAN                                    
----------------------------------------------------------------------------------
 Finalize Aggregate  (cost=7.30..7.31 rows=1 width=8)
   ->  Gather Motion 2:1  (slice1; segments: 2)  (cost=7.26..7.30 rows=2 width=8)
         ->  Partial Aggregate  (cost=7.26..7.27 rows=1 width=8)
               ->  Seq Scan on t1  (cost=0.00..6.00 rows=500 width=0)
 Optimizer: Postgres query optimizer
(5 rows)

```

#### 排序Sort的执行计划
全表排序在分布式下如何试下呢？很容易让我们联想到“分而治之”的算法思想，当数据量非常大时，我们通常采用“分而治之”的思想，将数据分片，在各自数据分片中完成部分排序，再汇总进行归并排序。我们看一下GP中的实现，就是这个思路：
```sql
postgres=# explain (costs off) select gp_segment_id,* from t1 order by b;
                QUERY PLAN                
------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)
   Merge Key: b			-- 由maste节点进行归并
   ->  Sort             -- segment节点进行部分排序
         Sort Key: b    
         ->  Seq Scan on t1    
 Optimizer: Postgres query optimizer
(6 rows)

-- 先Sort，再Limit
postgres=# explain select gp_segment_id,* from t1 order by b limit 5;
                                      QUERY PLAN                                      
--------------------------------------------------------------------------------------
 Limit  (cost=14.44..14.52 rows=5 width=12)
   ->  Gather Motion 2:1  (slice1; segments: 2)  (cost=14.44..14.61 rows=10 width=12)
         Merge Key: b
         ->  Limit  (cost=14.44..14.46 rows=5 width=12)
               ->  Sort  (cost=14.32..15.57 rows=500 width=12)
                     Sort Key: b
                     ->  Seq Scan on t1  (cost=0.00..6.00 rows=500 width=12)
 Optimizer: Postgres query optimizer
(8 rows)
```

#### Join的执行计划
Join是非常重要的，分布式Join如何实现呢？Join有多种实现方式：
- HashJoin
- MergeJoin
- NestLoopJoin

##### HashJoin

我们先分析HashJoin，在进行Join时，会有Join的条件，比如
```sql
select * from t1,t2 where t1.a = t2.a;
```
这里判断条件为`t1.a = t2.a`，因为表`t1`,`t2`的分布键都为`a`，所以，等值的数据一定在同一个segment节点上，这种情况下就比较容易实现了，相当于在各自segment节点进行HashJoin，再在master节点上聚合就好了：
```sql
postgres=# explain (costs off) select * from t1,t2 where t1.a = t2.a;
                QUERY PLAN                
------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)    -- master节点聚合数据
   ->  Hash Join							 -- segment节点上进行HashJoin
         Hash Cond: (t1.a = t2.a)
         ->  Seq Scan on t1
         ->  Hash
               ->  Seq Scan on t2
 Optimizer: Postgres query optimizer
(7 rows)
```
但如果是下面这种情况呢？在数据分布的时候，表`t2`是按`a`进行分布的，但Join的条件却是`t1.a = t2.b`，这就存在问题了，满足Join条件的`t2`中的数据与`t1`并不一定在同一个segment节点上，这样我们在各自segment上进行HashJion再汇集到master节点的算法实现就不满足条件，我们在实现算法的时候，需要牢记“分而治之”的思想。
```sql
select * from t1,t2 where t1.a = t2.b;
```
那么怎么办呢？我们可以进行重分布，以b为分布键重分布，这样就可以借鉴前面的实现思路了。执行计划如下：
```sql
postgres=# explain (costs off) select * from t1,t2 where t1.a = t2.b;
                         QUERY PLAN                         
------------------------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)
   ->  Hash Join
         Hash Cond: (t2.b = t1.a)
         ->  Redistribute Motion 2:2  (slice2; segments: 2)   -- 重分布
               Hash Key: t2.b
               ->  Seq Scan on t2
         ->  Hash
               ->  Seq Scan on t1
 Optimizer: Postgres query optimizer
(9 rows)
```
与面前的执行计划相比，多了一个`Redistribute Motion`的步骤，即重分布，数据按照新的分布键（关联键）重新打散到各segment上。

##### NestLoopJoin
NestLoop在GP中是如何实现的呢？这里就涉及到一个概念，广播，将每个segment上的某个表的数据全部发送给所有segment（需要注意，采用这种方式的表一般较小）。在实现NestLoop时，将其中一个较小的表广播到所有节点，然后进行NestLoop，再在master节点聚合所有数据。 

> citus中有Reference Table的概念，在每个节点上保存整张表完整的数据，与这里的广播起的作用类似。

```sql
postgres=# explain (costs off) select * from t1,t2;
                       QUERY PLAN                        
---------------------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)
   ->  Nested Loop
         ->  Broadcast Motion 2:2  (slice2; segments: 2)   -- 广播
               ->  Seq Scan on t2
         ->  Materialize
               ->  Seq Scan on t1
 Optimizer: Postgres query optimizer
(7 rows)
```

##### MergeJoin
采用MergeJoin的话，需要先进行排序，再按照归并排序的方式进行Join，执行计划如下：
```sql
postgres=# set enable_hashjoin = off;
SET
-- 不涉及重分布
postgres=# explain (costs off) select * from t1,t2 where t1.a = t2.a;
                 QUERY PLAN                 
--------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)
   ->  Merge Join
         Merge Cond: (t1.a = t2.a)
         ->  Index Scan using t1_pkey on t1		-- 索引扫描，相当于排序
         ->  Index Scan using t2_pkey on t2
 Optimizer: Postgres query optimizer
(6 rows)
-- 需要进行重分布
postgres=# explain (costs off) select * from t1,t2 where t1.a = t2.b;
                            QUERY PLAN                            
------------------------------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)
   ->  Merge Join
         Merge Cond: (t1.a = t2.b)
         ->  Index Scan using t1_pkey on t1
         ->  Sort
               Sort Key: t2.b
               ->  Redistribute Motion 2:2  (slice2; segments: 2)
                     Hash Key: t2.b
                     ->  Seq Scan on t2
 Optimizer: Postgres query optimizer
(10 rows)
```

#### Group By执行计划
在AP场景中，Group by是非常重要的，实现Group by有两种方式，一种是HashAggregate，另一种是GroupAggregate。 
- HashAggregate: Hash聚合，数据库会根据Group By字段后面的值计算Hash值，并根据前面使用的聚合函数在内存中维护相应的列表，然后数据库会通过这个列表来实现聚合操作。
- GroupAggregate: Group聚合，先将表中的数据按照Group By的字段排序，这样同一个Group By的值就在一起，再对排序好的数据进行一次全表扫描就可以得到聚合结果。

在GP中，因为有分布键的问题，在执行聚合时，需要考虑Group by的字段是否为分布键两种情况，在group by的字段为分布键时，group by相同的字段一定在同一个segment中，所以，只需要在各segment节点各自进行HashAggregate，再在master节点聚合数据就可以了。为什么没有采用GroupAggregate呢，因为它需要全表排序，在分布表时，生成分布执行计划，HashAggregate效率更高。
```sql
postgres=# explain select a,count(*) from t1 group by a;
                                   QUERY PLAN                                    
---------------------------------------------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)  (cost=8.51..28.53 rows=1001 width=12)
   ->  HashAggregate  (cost=8.51..13.51 rows=500 width=12)
         Group Key: a
         ->  Seq Scan on t1  (cost=0.00..6.00 rows=500 width=4)
 Optimizer: Postgres query optimizer
(5 rows)
```
另一种情况是当group by的字段不是分布键时，这时候就需要重分布，将数据按group by 的字段作为分布键进行重分布， 再进行HashAggregate：
```sql
postgres=# explain select b,count(*) from t1 group by b;
                                           QUERY PLAN                                            
-----------------------------------------------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)  (cost=18.52..38.54 rows=1001 width=12)
   ->  HashAggregate  (cost=18.52..23.52 rows=500 width=12)
         Group Key: b
         ->  Redistribute Motion 2:2  (slice2; segments: 2)  (cost=0.00..16.02 rows=500 width=4)
               Hash Key: b
               ->  Seq Scan on t1  (cost=0.00..6.00 rows=500 width=4)
 Optimizer: Postgres query optimizer
(7 rows)
```

#### 总结
以上对Greenplum中常见的执行计划做了分析，让我们理解了分布式执行计划是如何实现的，当然，SQL语句有非常多种情况，这里也无法一一做出分析，但处理的思想是一致的，那就是分而治之，这也是MPP数据库的精髓。后续在学习Greenplum时，可以去研究ORCA优化器以及列存，这是相对于OLTP数据库较为不同的地方，AP数据库与TP数据库的用户需求不同，所以在实现的方式上，一定有所不同，后续可以进行深入学习研究。
