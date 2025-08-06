## PostgreSQL增量物化视图pg_ivm


增量物化视图（Incremental View Maintenance，简称 IVM）是一种物化视图数据增量更新的技术，其通过仅计算并应用视图中的增量变更来实现更新，而不是像 `REFRESH MATERIALIZED VIEW` 命令那样从头开始重新计算整个视图。当视图中只有少量部分发生变更时，IVM更新物化视图的效率要高于完全重新计算。

关于视图维护的时机，有两种方法：立即更新和延迟更新。在立即更新中，视图会在其基表被修改的同一事务中进行更新。而在延迟更新中，视图的更新会在事务提交之后进行，例如在访问视图时、响应如 `REFRESH MATERIALIZED VIEW` 这样的用户命令时，或在后台定期执行等。`pg_ivm` 提供的是一种立即更新方式，即在基表被修改时，通过AFTER触发器立即更新物化视图。


### 增量物化视图的使用

安装扩展：
```sql
postgres=# create extension pg_ivm ;
CREATE EXTENSION
```

可通过函数`create_immv`创建增量物化视图。参数为增量物化视图名称和视图定义。其创建过程等同于使用`CREATE MATERIALIZED VIEW`创建物化视图。
```sql
postgres=# select pgivm.create_immv('mvt1','select * from t1');
NOTICE:  could not create an index on immv "mvt1" automatically
DETAIL:  This target list does not have all the primary key columns, or this view does not contain GROUP BY or DISTINCT clause.
HINT:  Create an index on the immv for efficient incremental maintenance.
 create_immv 
-------------
           6
(1 row)
```
当增量物化视图被创建成功后，物化视图的更新将自动进行。
```sql
-- 查看基表数据
postgres=# select * from t1;
 id |  name   | course  | score 
----+---------+---------+-------
  1 | qupeng  | math    |    90
  2 | qupeng  | english |    60
  3 | yangwei | math    |    69
  4 | yangwei | english |    80
  5 | lijing  | math    |    88
  6 | lijing  | english |    80
(6 rows)
-- 查看物化视图数据
postgres=# select * from mvt1;
 id |  name   | course  | score 
----+---------+---------+-------
  1 | qupeng  | math    |    90
  2 | qupeng  | english |    60
  3 | yangwei | math    |    69
  4 | yangwei | english |    80
  5 | lijing  | math    |    88
  6 | lijing  | english |    80
(6 rows)
-- 插入数据
postgres=# insert into t1 values(7,'xiexuedi','computer',100);
INSERT 0 1
-- 查看物化视图，有更新
postgres=# select * from mvt1;
 id |   name   |  course  | score 
----+----------+----------+-------
  1 | qupeng   | math     |    90
  2 | qupeng   | english  |    60
  3 | yangwei  | math     |    69
  4 | yangwei  | english  |    80
  5 | lijing   | math     |    88
  6 | lijing   | english  |    80
  7 | xiexuedi | computer |   100
(7 rows)
```

#### 创建增量物化视图

使用`create_immv`函数创建增量物化视图。
```sql
pgivm.create_immv(immv_name text, view_definition text) RETURNS bigint
```
参数说明：
- immv_name：物化视图名称
- view_definition：物化视图定义
创建增量物化视图后，将增量物化视图的元数据信息存储在表`pgivm.pg_ivm_immv`中。
```sql
postgres=# \d pgivm.pg_ivm_immv
                 Table "pgivm.pg_ivm_immv"
    Column     |   Type   | Collation | Nullable | Default 
---------------+----------+-----------+----------+---------
 immvrelid     | regclass |           | not null | 
 viewdef       | text     |           | not null | 
 ispopulated   | boolean  |           | not null | 
 lastivmupdate | xid8     |           |          | 
Indexes:
    "pg_ivm_immv_pkey" PRIMARY KEY, btree (immvrelid)
```

在创建增量物化视图时，会自动创建一些触发器用于当监控到基表发生更新时立即更新物化视图。如果可能的话，会自动创建一个唯一索引。如果视图定义中有GROUP BY子句，则会在GROUP BY子句中指定的列上创建索引。如果视图定义中有DISTINCT子句，则会在所有目标列上创建唯一索引。如果IMMV在其目标列表中包含了其基表的所有主键属性，则会在这些属性上创建一个唯一索引。在其他情况下，不会创建索引。
> 为什么呢？因为视图定义中的目标列表其实就是增量物化视图要创建的堆表的列，需要在增量物化视图对应的表中的列创建索引，这样在应用增量数据（其实就是INSERT增量数据，DELETE增量数据）的时候效率才高。如果没有创建索引的话，应用增量数据执行INSERT或DELETE操作时就会走顺序扫描，性能就会比较差。

`create_immv`函数会在视图上获取一个`AccessExclusiveLock`锁。然而，即使我们能够成功获取该锁，在我们获取锁之前，一个并发事务可能已经对视图进行了增量更新并提交。在 REPEATABLE READ 或 SERIALIZABLE 隔离级别下，这可能导致视图处于不一致的状态。遗憾的是，这种情况在视图创建过程中无法被检测到。因此，`create_immv`在这些隔离级别下会发出一个警告，建议在 READ COMMITTED 隔离级别下执行该命令，或者在之后执行`refresh_immv`以确保视图内容的一致性。
```sql
postgres=# select pgivm.create_immv('mvt2','select b,sum(a) from t4 group by b');
NOTICE:  created index "mvt2_index" on immv "mvt2"
WARNING:  inconsistent view can be created in isolation level SERIALIZABLE or REPEATABLE READ
DETAIL:  The view may not include effects of a concurrent transaction.
HINT:  create_immv should be used in isolation level READ COMMITTED, or execute refresh_immv to make sure the view is consistent.
 create_immv 
-------------
           2
(1 row)

postgres=# \d mvt2
                     Table "public.mvt2"
      Column       |  Type   | Collation | Nullable | Default 
-------------------+---------+-----------+----------+---------
 b                 | integer |           |          | 
 sum               | bigint  |           |          | 
 __ivm_count_sum__ | bigint  |           |          | 
 __ivm_count__     | bigint  |           |          | 
Indexes:
    "mvt2_index" UNIQUE, btree (b)
```

为什么呢？在隔离级别为`REPEATABLE READ`的情况下，`create_immv`创建增量物化视图的事务中，其他并发事务对基表的修改（比如INSERT）在隔离级别为`REPEATABLE READ`的情况下是不可见的，也就是说在读基表并插入到增量物化视图的表中没有读到这些“不可见”的修改，造成了视图内容的不一致。

#### 刷新增量物化视图
`refresh_immv` 刷新增量物化视图。
```sql
pgivm.refresh_immv(immv_name text, with_data bool) RETURNS bigint
```
`refresh_immv`会完全刷新增量物化视图，原有的数据会被丢弃，其效果等同与`REFRESH MATERIALIZED VIEW`命令。
`with_data`标志对应于`REFRESH MATERIALIZED VIEW`命令中的`WITH [NO] DATA`选项。如果`with_data`为`true`，则会执行底层查询以提供新数据，如果IMMV尚未填充数据，则会创建用于更新视图的触发器。同时，如果满足条件且视图尚未有索引，则会为IMMV创建一个唯一索引。如果`with_data`为`false`，则不会生成新数据，IMMV将变为未填充状态，并且会删除IMMV上用于更新视图的触发器。请注意，即使未填充数据的IMMV仍然是可扫描的，尽管其查询结果为空。未来可能会更改此行为，在扫描未填充的IMMV时改为抛出错误。

`refresh_immv`函数如同`create_immv`函数一样，建议在隔离级别为 READ COMMITTED下执行。

#### 获取增量物化视图的定义
`get_immv_def`函数可获取增量物化视图的定义，也就是`create_immv`函数中引用视图SQL语句。
```sql
postgres=# SELECT immvrelid, pgivm.get_immv_def(immvrelid) FROM pgivm.pg_ivm_immv ORDER BY 1;
 immvrelid |   get_immv_def    
-----------+-------------------
 mvt1      |  SELECT id,      +
           |     name,        +
           |     course,      +
           |     score        +
           |    FROM t1
 mvt2      |  SELECT b,       +
           |     sum(a) AS sum+
           |    FROM t4       +
           |   GROUP BY b
(3 rows)
```

#### 如何禁止或启用立即维护
当我们想要使IMMV保持最新并且不经常修改基表或者仅修改基表的一小部分时，IVM是有效的。由于即时维护的开销，当频繁修改基表时，IVM不起作用。此外，当修改基表的很大一部分或将大量数据插入基表时，IVM是无效的，维护成本可能比从头开始刷新更大。
当遇到此类情形时，我们可以在修改基表前使用 refesh_immv 函数并带上 with_data = false 禁止立即维护。等基表修改完成后，调用 refresh_immv 并带上with_data = true 刷新物化视图并启用立即维护。

#### 行级安全
如果某些基表具有行级安全策略，则对物化视图所有者不可见的行将从结果中排除。此外，当以增量方式维护视图时，也会排除此类行。但是，如果在创建物化视图后定义了新策略或更改了策略，则新策略将不会应用于视图内容。要应用新策略，您需要重新创建IMMV。

### 支持的视图定义和限制
目前，IMMV 的视图定义可以包含内连接（inner joins）、DISTINCT 子句、一些内置的聚合函数、FROM 子句中的简单子查询、EXISTS 子查询以及简单的 CTE（WITH 查询）。支持包含自连接的内连接，但不支持外连接（outer joins）。支持的聚合函数包括 count、sum、avg、min 和 max。视图定义中不能使用其他聚合函数，也不能使用包含聚合函数或 DISTINCT 子句的子查询、FROM 子句以外的子查询、窗口函数、HAVING、ORDER BY、LIMIT/OFFSET、UNION/INTERSECT/EXCEPT、DISTINCT ON、TABLESAMPLE、VALUES 以及 FOR UPDATE/SHARE。

基表必须是普通表，不能使用视图、物化视图、继承父表、分区表、分区、外部表。

视图定义查询中不能包含任何系统列。目标列表中也不能包含名称以 __ivm_ 开头的列。

视图目标列表中使用的数据类型必须具有 btree 访问方法的默认操作符类。例如，json、xml 或 point 类型不能出现在目标列表中。

不支持逻辑复制，也就是说，即使修改发布者节点上的基表，这些基表上定义的订阅者节点上的IMMV也不会更新
#### 聚合
支持的聚合函数包括 count、sum、avg、min 和 max。目前仅支持内置的聚合函数，用户自定义的聚合函数无法使用。

当创建包含聚合函数的 IMMV 时，系统会自动在目标列表中添加一些以 `__ivm` 开头的额外列。`__ivm_count__` 列包含每组中被聚合的元组数量。此外，为了维护每个聚合值，还会为每个被聚合的列添加一个或多个额外列。例如，为了维护平均值，会添加名为 `__ivm_count_avg__` 和 `__ivm_sum_avg__` 的列。当基础表被修改时，新的聚合值将通过 IMMV 中存储的旧聚合值和相关额外列的值进行增量计算。

需要注意的是，对于 min 或 max 函数，如果从基础表中删除了包含当前最小或最大值的元组，则可能需要重新从基础表中针对受影响的组重新计算新值。因此，更新包含这些函数的 IMMV 可能需要较长时间。

另外，注意在 IMMV 中对 real（float4）类型或 double precision（float8）类型使用 sum 或 avg 是不安全的，因为由于这些类型的精度有限，IMMV 中的聚合值可能会与从基础表计算出的结果不同。为了避免此问题，建议使用 numeric 类型。

##### 聚合限制
如果使用了 `GROUP BY` 子句，则 `GROUP BY` 中指定的表达式必须出现在目标列表中。这是识别 IMMV 中需要更新的元组的方式。这些属性将用作在 IMMV 中搜索元组的扫描键，因此为了高效地进行增量视图维护（IVM），必须在这些属性上创建索引。

目标列表不能包含包含聚合函数的表达式。


#### 子查询
支持 FROM 子句中的简单子查询和 WHERE 子句中的 EXISTS 子查询。

##### 子查询限制
支持使用 EXISTS 的子查询以及 FROM 子句中的简单子查询。不支持包含除 'AND' 以外条件的 EXISTS 子查询，也不支持目标列表（targetlist）中的子查询。EXISTS 子查询仅支持在 WHERE 子句中使用，不支持在目标列表中使用。

如果 EXISTS 子查询中包含引用外部查询表中列的列，则这些列必须包含在目标列表中。包含聚合函数或 DISTINCT 的子查询不被支持。


#### CTE
支持简单CTE（WITH查询）
##### CTE限制
包含聚合函数或 DISTINCT 的 WITH 查询不被支持。

不允许使用递归查询（WITH RECURSIVE）。未被引用的 CTE 也是不允许的，也就是说，一个 CTE 必须在视图定义查询中至少被引用一次。

#### DISTINCT
在IMMV的定义查询中允许使用 DISTINCT。假设在一个包含重复元组的基础表上定义了一个使用 DISTINCT 的 IMMV。当从基础表中删除元组时，只有当该元组的重数（multiplicity）变为零时，视图中的对应元组才会被删除。此外，当向基础表中插入元组时，只有当视图中尚不存在相同元组时，才会将该元组插入到视图中。

物理上，使用 DISTINCT 定义的 IMMV 包含去重后的元组，每个元组的重数存储在一个名为 `__ivm_count__` 的额外列中，该列在创建此类 IMMV 时被添加。

#### TRUNCATE
当基表被truncate时，如果视图定义查询不包含不带GROUP BY子句的聚合，则IMMV也被截断，并且内容变为空。没有GROUP BY子句的聚合视图始终有一行。因此，在这种情况下，如果基表被truncate，IMMV只会被刷新，而不是被截断。

#### 并发事务
假设在两个基表上定义了IMMV，并且每个表在不同的并发事务中被同时修改。在先被提交的事务中，IMMV可被更新，因为这里只需要考虑发生在本事务中的改变。另一方面，为了正确更新在稍后提交事务里的IMMV，我们需要知道发生在每个事务里的变化。因此，在读已提交模式下修改基表后，IMMV立即持有ExclusiveLock，以确保在前一个事务提交后在后一个事务中更新IMMV。 在 REPEATABLE READ 或者 SERIALIZABLE 模式，如果获取锁失败，则会立即引发错误，因为其他事务中发生的任何更改在这些模式下都不可见，并且IMMV在这种情况下无法正确更新。当然也有例外：如果IMMV只有一个基表并且不使用DISTINCT或者GROUP BY，并且表是被 INSERT 修改的，IMMV持有的锁是 RowExclusiveLock。




### 性能对比
我们对比一下物化视图和增量物化视图的性能差异。

我们首先创建普通物化视图，对基表进行更新以及对普通物化视图进行刷新。
```sql
postgres=# CREATE MATERIALIZED VIEW mv_normal AS
        SELECT a.aid, b.bid, a.abalance, b.bbalance
        FROM pgbench_accounts a JOIN pgbench_branches b USING(bid);
SELECT 1000000
Time: 3372.753 ms (00:03.373)
postgres=# UPDATE pgbench_accounts SET abalance = 1000 WHERE aid = 1;
UPDATE 1
Time: 1.652 ms
postgres=# REFRESH MATERIALIZED VIEW mv_normal ;
REFRESH MATERIALIZED VIEW
Time: 2710.322 ms (00:02.710)
```
可以看到对普通物化视图进行刷新与新创建物化视图的代价相近，代价非常大。而增量物化视图是实时更新，非常快。正常情况下，增量物化视图相比普通物化视图，其优势在于其更新物化视图快，可以立即更新。而带来的代价就是对基表的更新慢一些。这是因为在增量物化视图中，对基表进行更新时，还需要额外做物化视图的增量更新，所以对基表的更新操作会慢一些。
```sql
postgres=# SELECT pgivm.create_immv('immv',
        'SELECT a.aid, b.bid, a.abalance, b.bbalance
         FROM pgbench_accounts a JOIN pgbench_branches b USING(bid)');
NOTICE:  created index "immv_index" on immv "immv"
 create_immv 
-------------
     1000000
(1 row)

Time: 3581.543 ms (00:03.582)
postgres=# UPDATE pgbench_accounts SET abalance = 1234 WHERE aid = 1;
UPDATE 1
Time: 5.112 ms
postgres=# SELECT * FROM immv WHERE aid = 1;
 aid | bid | abalance | bbalance 
-----+-----+----------+----------
   1 |   1 |     1234 |        0
(1 row)

Time: 0.959 ms
```
可以看到，增量物化视图的更新代价非常小。


### 基本设计原理

设计原理可参考[Incremental View Maintenance](https://wiki.postgresql.org/wiki/Incremental_View_Maintenance)

里面说的非常明白。

IVM computes and applies only the incremental changes to the materialized views. Suppose that view V is defined by query Q over a state of base relations D. When D changes D' = D + dD, we can get the new view state V' by calculating from D' and Q, and this is re-computation performed by REFRESH MATERIALIZED VIEW command. On the other hand, IVM calculates the delta for view (dV) from the base tables delta (dD) and view definition (Q), and applies this to get the new view state, V' = V + dV.

In theory, the view definition is described in a relational algebra (or bag algebra) form. For example, a (inner) join view of table R and S is defined as V = R ⨝ S.

When table R is changed in a transaction, this can be described as R ← R - ∇R + ΔR, where ∇R and ΔR denote tuples deleted from and inserted into R, respectively. (To be accurate, instead of - and +, ∸ and ⨄ are used by tradition in bag algebra.) In this condition, the deltas of the view are calculated as ∇V = ∇R ⨝ S and ΔV = ΔR ⨝ S, then the view can be updated as V ← V - ∇V + ΔV.

基表的变化分为新增和删除。 
V ← V - ∇V + ΔV. 

术语：
- 基表（Base table）:指物化视图定义中使用到的普通表
- 增量（Delta）：指基表的数据发生变化时，与物化视图中的数据相比，增加和删除的数据集合。
  

增量物化视图是一种使物化视图保持最新的方法。其中只计算增量更改并将其应用于视图，而不是重新计算整个视图（`REFRESH MATERIALIZED VIEW`）。

物化视图的更新有两种方法：
- 立即更新：在修改物化视图的同一事务中更新物化视图
- 延迟更新：在事务提交后更新物化视图，例如当要访问物化视图时使用
  
当前PostgreSQL支持物化视图，但仅支持`REFRESH MATERIALIZED VIEW`命令刷新物化视图，此种方式为重新构建物化视图。pg_ivm扩展提供了一种即时维护物化视图的方法，当修改基表时，物化视图在AFTER触发器中立即更新。所以，这样的设计的代价就是更新基表会更慢，但好处是比`REFRESH MATERIALIZED VIEW`方式更新物化视图要快。

每次修改基表时，会通过特定的算法计算增量数据，这个算法是通过关系代数水下的。基表与增量物化视图的变化不是一一对应的，基表插入十条数据，增量物化视图不一定插入十条数据，所以要通过计算得到增量数据。

如何监测对基表的改动呢？核心是触发器（after trigger）和转换表机制， 是基于触发器去实现的，在基表上创建`INSERT | UPDAE | DELETE | TRUNCATE ` AFTER STATEMENT触发器，当对基表修改时，通过触发器执行触发器函数，触发器中有个特性，支持REFERENCING选项。

REFERENCING选项启用对传递关系的收集，传递关系是包括被当前SQL语句插入、删除或者修改的行的行集合。这个特性让触发器能看到该语句做的事情的全局视图，而不是一次只看到一行。仅对非约束触发器的AFTER触发器允许这个选项。此外，如果触发器是一个UPDATE触发器，则它不能指定column_name列表。OLD TABLE仅可以被指定一次，并且只能为在UPDATE或DELETE事件上引发的触发器指定，它创建的传递关系包含有该语句更新或删除的所有行的前映像。类似地，NEW TABLE仅可以被指定一次，并且只能为在UPDATE或INSERT事件上引发的触发器指定，它创建的传递关系包含有该语句更新或插入的所有行的后映像。
```sql
CREATE [ OR REPLACE ] [ CONSTRAINT ] TRIGGER name { BEFORE | AFTER | INSTEAD OF } { event [ OR ... ] }
    ON table_name
    [ FROM referenced_table_name ]
    [ NOT DEFERRABLE | [ DEFERRABLE ] [ INITIALLY IMMEDIATE | INITIALLY DEFERRED ] ]
    [ REFERENCING { { OLD | NEW } TABLE [ AS ] transition_relation_name } [ ... ] ]
    [ FOR [ EACH ] { ROW | STATEMENT } ]
    [ WHEN ( condition ) ]
    EXECUTE { FUNCTION | PROCEDURE } function_name ( arguments )

这里的event可以是下列之一：

    INSERT
    UPDATE [ OF column_name [, ... ] ]
    DELETE
    TRUNCATE
```
> [CREATE TRIGGER](http://postgres.cn/docs/14/sql-createtrigger.html)


对基表创建触发器，以便监测基表的更新，获取基表的更新。
在满足一定条件的时候，对增量物化视图创建唯一索引，加速增量物化视图的增量刷新。

如何计算增量？
其实基表的变化就2点：
- 新增的数据
- 删除的数据
UPDATE可视为删除再新增。物化视图的更新同样，新增的数据，删除的数据。

怎么把增量数据应用到物化视图中呢？
```sql
-- view definition
select ... fom A,B where ...
-- rewrite query
-- 这里new_table_A，就是存储的基表A新增的数据
-- 经过计算就是物化视图新增的数据
select ... from new_table_A,B where ...
-- 这里old_table_A，就是存储的基表A删除的数据
-- 经过计算就是物化视图删除的数据
select ... from old_table_A,B where ...
```


如果视图定义中含有DISTINCT怎么处理？
1. 添加一个特殊列__ivm_count__，维护count值
2. 当视图中已含有相同的元组，count数值相应的增加计数，如果相同的元组不存在，则视图中插入一个新元组。
3. 当将要在视图中删除元组时，count计数减少，当count数值为0时，则可以从视图中删除该元组。


需要禁止用户直接操作增量物化视图表。

为什么不支持外部表、视图，因为触发器不支持TRUNCATE外部表、视图，我们的设计是基于触发器实现的。

在计算增量视图前，必须要锁定所有基表，设置for update锁。


ivm_immediate_maintenance
--> calc_delta
--> apply_delta

 R ← R - ∇R + ΔR


核心实现函数：当基表发生更新时，会触发After触发器，调用该函数。
```c++
/*
 * IVM_immediate_maintenance
 *
 * IVM trigger function invoked after base table is modified.
 * For each table, tuplestores of transition tables are collected.
 * and after the last modification
 */
Datum IVM_immediate_maintenance(PG_FUNCTION_ARGS)
{
	MV_TriggerTable		*table;

	/* Save the transition tables and make a request to not free immediately */
    // 获取增量数据，delete数据
	if (trigdata->tg_oldtable)
	{
		oldcxt = MemoryContextSwitchTo(TopTransactionContext);
		table->old_tuplestores = lappend(table->old_tuplestores, tuplestore_copy(trigdata->tg_oldtable, rel));
		entry->has_old = true;
		MemoryContextSwitchTo(oldcxt);
	}
    // 获取增量数据，insert数据
	if (trigdata->tg_newtable)
	{
		oldcxt = MemoryContextSwitchTo(TopTransactionContext);
		table->new_tuplestores = lappend(table->new_tuplestores, tuplestore_copy(trigdata->tg_newtable, rel));
		entry->has_new = true;
		MemoryContextSwitchTo(oldcxt);
	}

	/* calculate delta tables */
    // 计算增量数据，根据基表的oldtable,newtable 应用到视图定义中得到视图的增量数据 
	calc_delta(table, rte_path, rewritten, dest_old, dest_new, &tupdesc_old, &tupdesc_new, queryEnv);

	/* apply the delta tables to the materialized view */
    // 将视图增量数据应用到物化视图表中，也就是将上一步的计算结果，通过insert/delete更新物化视图
	apply_delta(matviewOid, old_tuplestore, new_tuplestore,tupdesc_old, tupdesc_new, query, use_count,count_colname);

}
```

参考[deepwiki](https://deepwiki.com/search/_4d453e22-b0e2-4540-8603-2e9157d3b2cf)

```c++
/*
 * apply_new_delta
 *
 * Execute a query for applying a delta table given by deltaname_new
 * which contains tuples to be inserted into a materialized view given by
 * matviewname.  This is used when counting is not required.
 */
static void
apply_new_delta(const char *matviewname, const char *deltaname_new,
				StringInfo target_list)
{
	StringInfoData	querybuf;

	/* Search for matching tuples from the view and update or delete if found. */
	initStringInfo(&querybuf);
	appendStringInfo(&querybuf,
					"INSERT INTO %s (%s) SELECT %s FROM ("
						"SELECT diff.*, pg_catalog.generate_series(1, diff.\"__ivm_count__\") AS __ivm_generate_series__ "
						"FROM %s AS diff) AS v",
					matviewname, target_list->data, target_list->data,
					deltaname_new);

	if (SPI_exec(querybuf.data, 0) != SPI_OK_INSERT)
		elog(ERROR, "SPI_exec failed: %s", querybuf.data);
}

```

```c++
/*
 * apply_old_delta
 *
 * Execute a query for applying a delta table given by deltaname_old
 * which contains tuples to be deleted from a materialized view given by
 * matviewname.  This is used when counting is not required.
 */
static void
apply_old_delta(const char *matviewname, const char *deltaname_old,
				List *keys)
{
	StringInfoData	querybuf;
	StringInfoData	keysbuf;
	char   *match_cond;
	ListCell *lc;

	/* build WHERE condition for searching tuples to be deleted */
	match_cond = get_matching_condition_string(keys);

	/* build string of keys list */
	initStringInfo(&keysbuf);
	foreach (lc, keys)
	{
		Form_pg_attribute attr = (Form_pg_attribute) lfirst(lc);
		char   *resname = NameStr(attr->attname);
		appendStringInfo(&keysbuf, "%s", quote_qualified_identifier("mv", resname));
		if (lnext(keys, lc))
			appendStringInfo(&keysbuf, ", ");
	}

	/* Search for matching tuples from the view and update or delete if found. */
	initStringInfo(&querybuf);
	appendStringInfo(&querybuf,
	"DELETE FROM %s WHERE ctid IN ("
		"SELECT tid FROM (SELECT pg_catalog.row_number() over (partition by %s) AS \"__ivm_row_number__\","
								  "mv.ctid AS tid,"
								  "diff.\"__ivm_count__\""
						 "FROM %s AS mv, %s AS diff "
						 "WHERE %s) v "
					"WHERE v.\"__ivm_row_number__\" OPERATOR(pg_catalog.<=) v.\"__ivm_count__\")",
					matviewname,
					keysbuf.data,
					matviewname, deltaname_old,
					match_cond);

	if (SPI_exec(querybuf.data, 0) != SPI_OK_DELETE)
		elog(ERROR, "SPI_exec failed: %s", querybuf.data);
}
```





参考文档：
[CloudberryDB内核分享：增量物化视图的原理与实现讲解](https://segmentfault.com/a/1190000045387918)
[PostgreSQL物化视图增量更新扩展 -- pg_ivm](https://developer.aliyun.com/article/1362757)


