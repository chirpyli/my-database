### 逻辑优化——子查询优化
子查询使用的频率非常高，对其进行优化十分重要。这里我们分析一下PostgreSQL是如何进行子查询优化的。

#### 关于子查询在SQL中的位置
除了子查询，还有子连接的说法，它们的主要区别是出现在查询语句中的位置不同。这里我们不将他们进行区分，重点分析子查询出现在目标列、FROM子句、WHERE子句的情况。

当出现在目标列时，则只能是标量子查询，否则会返回错误。
```sql
-- fail
postgres@postgres=# select aid, (select * from pgbench_branches ) from pgbench_accounts ;
ERROR:  subquery must return only one column
LINE 1: select aid, (select * from pgbench_branches ) from pgbench_a...

-- success
postgres@postgres=# explain select aid, (select bid from pgbench_branches ) from pgbench_accounts ;
                                                QUERY PLAN                           
                      
-------------------------------------------------------------------------------------
----------------------
 Index Only Scan using pgbench_accounts_pkey on pgbench_accounts  (cost=1.30..1777.30
 rows=100000 width=8)
   InitPlan 1 (returns $0)
     ->  Seq Scan on pgbench_branches  (cost=0.00..1.01 rows=1 width=4)
(3 rows)
```

出现在FROM子句的位置时，相关子查询不能出现在FROM子句中，只能是非相关子查询，可上拉子查询到父层，通过Semi-Join进行优化。

出现在WHERE子句位置时，其实就是子连接，此时子查询是条件表达式的一部分。


#### 相关子查询与非相关子查询
这里单独说明一下相关子查询和非相关子查询，因为后续优化的时候，会区分这两种情况。
- 相关子查询： 子查询的执行依赖于外层父查询的一些属性值，当父查询的参数改变时，子查询需要根据新的参数值重新执行。
```sql
postgres@postgres=# explain select * from pgbench_accounts ta where aid in (select bid from pgbench_branches tb where ta.aid=tb.bid);
                                 QUERY PLAN                                 
----------------------------------------------------------------------------
 Seq Scan on pgbench_accounts ta  (cost=0.00..53640.00 rows=50000 width=97)
   Filter: (SubPlan 1)
   SubPlan 1
     ->  Seq Scan on pgbench_branches tb  (cost=0.00..1.01 rows=1 width=4)
           Filter: (ta.aid = bid)
(5 rows)
```
- 非相关子查询： 不依赖于外层父查询的任何属性值，可独立求解。
```sql
postgres@postgres=# explain select * from pgbench_accounts ta where aid in (select bid from pgbench_branches tb where tb.bid = 10);
                                               QUERY PLAN                                               
--------------------------------------------------------------------------------------------------------
 Nested Loop Semi Join  (cost=0.29..3.33 rows=1 width=97)
   ->  Index Scan using pgbench_accounts_pkey on pgbench_accounts ta  (cost=0.29..2.31 rows=1 width=97)
         Index Cond: (aid = 10)
   ->  Seq Scan on pgbench_branches tb  (cost=0.00..1.01 rows=1 width=4)
         Filter: (bid = 10)
(5 rows)
```

#### 子查询优化
如果子查询不做优化，其执行逻辑基本就是每执行一次父查询，就执行一次子查询（子查询生成子查询计划），也就是嵌套执行的逻辑，很明显，这样效率会比较低。其实很多情况下，可将嵌套执行转为Semi-Join进行优化。

我们先通过一个例子分析一下是怎么进行优化的：
```sql
create table t1(id int, data int);
insert into t1 values(1,1);
insert into t1 values(2,2);
insert into t1 values(3,3);
insert into t1 values(4,4);

create table t2(a int, b int);
insert into t2 values(1,1);
insert into t2 values(2,2);

-- 测试例句
postgres@postgres=# select * from t1 where id in(select a from t2);
 id | data 
----+------
  1 |    1
  2 |    2
(2 rows)
```
我们看一下优化器给出的执行计划：
```sql
postgres=# explain select * from t1 where id in(select a from t2);
                               QUERY PLAN                               
------------------------------------------------------------------------
 Hash Join  (cost=42.75..93.85 rows=1130 width=8)
   Hash Cond: (t1.id = t2.a)
   ->  Seq Scan on t1  (cost=0.00..32.60 rows=2260 width=8)
   ->  Hash  (cost=40.25..40.25 rows=200 width=4)
         ->  HashAggregate  (cost=38.25..40.25 rows=200 width=4)
               Group Key: t2.a
               ->  Seq Scan on t2  (cost=0.00..32.60 rows=2260 width=4)
(7 rows)
```
对于子查询`select * from t1 where id in (select a from t2)`;等同于`select t1.id,t1.data from t1,t2 where t1.id = t2.a;` 。在绝大多数情况下，后面的等价语句效率更高。所以，经过优化器的处理后，不再是通过子计划的方式，而是通过上拉子查询，转换为`Hash Semi Join`进行优化。

优化器怎么转换为后面的等价语句的过程大致为：
1. 将子查询中的表t2,上拉到上一层查询的from语句中，`select t1.id,t1.data from t1,t2 where id in (select a from t2)`
2. 重写where子句，`select t1.id,t1.data from t1,t2 where t1.id = t2.a`

当然也并不是所有情况下，转为为Semi-Join的方式更高效，我们知道优化器多表进行Join的物理优化算法是动态规划，当表的数量超过一定值时，其穷举的数量越高，计划生成的代价越高，但在表数量较少时，往往转换为Semi-Join的方式更高效。

当然，也并不是所有情况下都可以转换为半连接（Semi-Join），可以转换的前提是等价变换，语义不能发生变化。那么什么情况下不能进行上拉，什么情况下可以进行上拉呢？
- 如果子查询中出现了聚集、GROUPBY、DISTINCT子句，则子查询只能单独求解，不可以上拉。
```sql
postgres@postgres=# explain select * from t1 where id in(select sum(a) from t2 group by a);
                                    QUERY PLAN                                     
-----------------------------------------------------------------------------------
 Hash Semi Join  (cost=50.40..51.59 rows=4 width=8)
   Hash Cond: (t1.id = "ANY_subquery".sum)
   ->  Seq Scan on t1  (cost=0.00..1.04 rows=4 width=8)
   ->  Hash  (cost=47.90..47.90 rows=200 width=8)
         ->  Subquery Scan on "ANY_subquery"  (cost=43.90..47.90 rows=200 width=8)
               ->  HashAggregate  (cost=43.90..45.90 rows=200 width=12)
                     Group Key: t2.a
                     ->  Seq Scan on t2  (cost=0.00..32.60 rows=2260 width=4)
(8 rows)
-- 可以认为是子连接上拉为子查询，原语句转换为：select * from t1 semi join (select sum(a) from t2 group by a) as ANY_subquery where t1.id = ANY_subquery.sum
```
- 如果子查询是简单的SPJ（选择投影连接）的语句，则可以上拉。
```sql
postgres@postgres=# explain select * from t1 where id = any (select a from t2);
                            QUERY PLAN                            
------------------------------------------------------------------
 Hash Join  (cost=39.34..42.13 rows=2 width=8)
   Hash Cond: (t2.a = t1.id)
   ->  HashAggregate  (cost=38.25..40.25 rows=200 width=4)
         Group Key: t2.a
         ->  Seq Scan on t2  (cost=0.00..32.60 rows=2260 width=4)
   ->  Hash  (cost=1.04..1.04 rows=4 width=8)
         ->  Seq Scan on t1  (cost=0.00..1.04 rows=4 width=8)
(7 rows)
```
上拉的步骤就如同前面例子中所讲的那样：
1. 将子查询和上层查询的FROM子句连接为同一个FROM子句
2. 将子查询的谓词符合进行修改（如IN修改为ANY）
3. 将子查询的WHERE条件作为一个整体与上层查询的WHERE条件合并，并用AND条件连接词进行连接，从而保证新生成的谓词与原谓词的上下文意思相同，且成为一个整体。

补充一点，对于`IN`，等价于`=ANY`， 比如下面的语句是等价的：
```sql
select * from t1 where id in (select a from t2);
-- 等价于
select * from t1 where id = ANY (select a from t2);
```

对于什么样的子查询可以进行优化，能不能优化，规则可能比较难记，但是一定要记住一个核心的原则，就是等价代换，不过等价，那么一定无法进行上拉优化，下面看这3个IN子查询的例子：
- 语句一：简单的IN子查询
```sql
select * from t1 where id = any (select a from t2);
-- 可进行上拉优化：
postgres@postgres=# explain select * from t1 where id = any (select a from t2);
                            QUERY PLAN                            
------------------------------------------------------------------
 Hash Join  (cost=39.34..42.13 rows=2 width=8)
   Hash Cond: (t2.a = t1.id)
   ->  HashAggregate  (cost=38.25..40.25 rows=200 width=4)
         Group Key: t2.a
         ->  Seq Scan on t2  (cost=0.00..32.60 rows=2260 width=4)
   ->  Hash  (cost=1.04..1.04 rows=4 width=8)
         ->  Seq Scan on t1  (cost=0.00..1.04 rows=4 width=8)
(7 rows)
```
- 语句二： 简单的IN子查询，但子查询结果不影响父查询
```sql
select * from t1 where 10 = any (select a from t2);
-- 不可以进行优化， 可以分析一下，这个语句是无法进行上拉的，只能对子查询求值
postgres@postgres=# explain select * from t1 where 10 = any (select a from t2);
                          QUERY PLAN                          
--------------------------------------------------------------
 Result  (cost=38.25..39.29 rows=4 width=8)
   One-Time Filter: (hashed SubPlan 1)
   ->  Seq Scan on t1  (cost=38.25..39.29 rows=4 width=8)
   SubPlan 1
     ->  Seq Scan on t2  (cost=0.00..32.60 rows=2260 width=4)
(5 rows)
```
- 语句三：带有易失函数的IN子查询
```sql
select * from t1 where id + random()*10 in (select a from t2);
-- 很明显，带有易失函数的IN子查询是无法进行上拉的。
postgres@postgres=# explain select * from t1 where id + random()*10 in (select a from t2);
                          QUERY PLAN                          
--------------------------------------------------------------
 Seq Scan on t1  (cost=38.25..39.35 rows=2 width=8)
   Filter: (hashed SubPlan 1)
   SubPlan 1
     ->  Seq Scan on t2  (cost=0.00..32.60 rows=2260 width=4)
(4 rows)
```

对于ANY子连接，则可以进行某种转换，比如`val > ANY`等价于`val > MIN (SELECT ...)`。还有SOME/ALL，都可以进行某种转换，我们不进行一一列举。以如下语句为例： 
```sql
select * from t1 where id > any (select a from t2);
--等价于
select * from t1 where id > (select min(a) from t2);
```