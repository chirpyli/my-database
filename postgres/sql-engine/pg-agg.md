### PostgreSQL聚合算子实现原理
聚合功能是SQL中非常常用的功能，PostgreSQL提供了丰富的聚合函数，如`COUNT`、`SUM`、`AVG`、`MAX`、`MIN`等等。这里我们分析一下其实现原理。

#### 聚合实现原理
我们先通过一个例子分析一下聚集的执行计划。准备数据。
```sql
-- 建表
postgres=# create table t1(id int, name varchar(20), course varchar(20), score int);
CREATE TABLE
-- 造数据
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
-- 聚合查询，无分组情况
postgres=# explain select sum(score) from t1;
                       QUERY PLAN                       
--------------------------------------------------------
 Aggregate  (cost=1.07..1.08 rows=1 width=8)
   ->  Seq Scan on t1  (cost=0.00..1.06 rows=6 width=4)
(2 rows)
-- FILTER条件过滤，允许在传递给聚合函数前过滤数据
postgres=# explain select sum(score) FILTER (WHERE course = 'math') from t1;
                       QUERY PLAN                        
---------------------------------------------------------
 Aggregate  (cost=1.09..1.10 rows=1 width=8)
   ->  Seq Scan on t1  (cost=0.00..1.06 rows=6 width=62)
(2 rows)
```
执行`SUM`聚集，查看执行计划，可以看到是通过哈希实现聚集。哈希实现非常容易理解，因为每个`GROUP BY`键都可以映射为一个哈希值，然后通过哈希值进行分组，最后进行求和。相同分组的哈希值相同，就会进行分组。
```sql
-- 执行sum聚合，统计名字对应的分数和
postgres=# select name,sum(score) from t1 group by name;
  name   | sum 
---------+-----
 yangwei | 149
 qupeng  | 150
 lijing  | 168
(3 rows)
-- 查看执行计划，为通过哈希HashAggregate实现
postgres=# explain select name,sum(score) from t1 group by name;
                         QUERY PLAN                         
------------------------------------------------------------
 HashAggregate  (cost=17.95..19.95 rows=200 width=66)
   Group Key: name
   ->  Seq Scan on t1  (cost=0.00..15.30 rows=530 width=62)
(3 rows)
```
除了哈希实现分组聚合，还可以通过排序实现分组聚合，当我们对name列创建了一个索引后，再次查看执行计划，发现为`GroupAggregate`实现。其原理为先对`GROUP BY`列进行排序，然后进行分组聚合，分组集合时，当发现与前行相同时，则将当前行加入当前分组，否则创建一个新的分组。
```sql
postgres=# create index idx_t1_name on t1 (name);
CREATE INDEX
postgres=# explain select name,sum(score) from t1 group by name order by name;
                          QUERY PLAN                           
---------------------------------------------------------------
 GroupAggregate  (cost=1.14..1.24 rows=6 width=66)
   Group Key: name
   ->  Sort  (cost=1.14..1.15 rows=6 width=62)
         Sort Key: name
         ->  Seq Scan on t1  (cost=0.00..1.06 rows=6 width=62)
(5 rows)
```

当通过哈希或者排序的方式实现分组后，通过执行不同的聚集函数，可以得到不同的结果。比如，`sum()`函数，可以得到所有分组的求和结果，`avg()`函数，可以得到所有分组的求平均结果。优化器选择那种方式是根据执行代价来决定的，那种代价低选择那种方式，通常而言，如果`GROUP BY`的列是索引列或者已排序，这种情况`GroupAgregate`的执行代价往往比`HashAggregate`方式要低，会选择采用`GroupAgregate`的方式。分组后怎么对不同的聚集函数进行计算呢？PostgreSQL中对这部分计算分为三步：
1. 设置初始条件
2. 对所有输入元组调用转换函数transfunc
3. 对所有分组调用最终函数finalfunc		

当前并不是所有的聚合操作都需要调用最终函数finalfunc进行处理，比如像count，sum等这些转换函数的最终状态就是结果值，就无需再进行最终处理了。之所以分成以上步骤，其实是对所有聚合函数计算过程的抽象，也就是说，所有的聚合计算都可以分解为下述步骤进行计算。
```c++
    // 初始条件，对于不同的聚集函数，初始条件不同，比如sum为0，avg为(0,0)等
    transvalue = initcond   
    foreach input_tupe do  // 对每个输入元组
        // 执行转换函数，如果有FILTER子句，则在调用transfunc之前先根据条件进行过滤
        // 例如：对SUM，state为当前总和， transfunc(state, new_value), 返回state+new_value
        // 例如：对AVG，state为(sum,count)， transfunc(state, new_value), 返回sum+new_value，count+1
        transvalue = transfunc(transvalue, input_value(s))
    // 执行最终处理函数 
    // 例如：对SUM，不需要finalfunc，返回transvalue
    // 例如：对AVG，finalfunc接收(sun,count)，计算平均值sum/count，返回结果
    result = finalfunc(transvalue, direct_argument(s))

```
如果没有提供最终处理函数finalfunc，则结果为transvalue的最终值。

整体的处理逻辑如下：
1. 查询优化器阶段：
- 选择代价最低的聚合策略，`HashAggregate`或`GroupAggregate`。
1. 执行器阶段：
- 底层节点获取输入行，比如`Seq Scan`
- 计算分组键：对每一行，计算`GROUP BY`子句中的表达式的值，得到分组键
- 查找或者创建分组：
  - `GroupAggregate`：因为已按分组键排序，只需比较当前行的分组键与上一行的分组键是否相同，相同则为当前分组，不同则完成当前分组的聚合（调用`finalfunc`，如果存在），输出结果行，然后创建新的分组。
  - `HashAggregate`：使用哈希表存储分组，哈希表键为分组键，使用计算出的分组键查询哈希表，如果命中，则找到对应的分组，否则创建新的分组。
- 更新聚合状态：对于当前行所属分组，调用transfunc函数更新状态信息
- 最后调用finalfunc最终函数（如果定义了），对于`GroupAggregate`，触发时机为进入下一个分组或者输入结束时触发，对于`HashAggregate`，触发时机为输入结束时触发。
  
那么每个聚合函数的转换函数和最终处理函数都是什么呢？可通过pg_aggregate系统表进行查看：
```sql
-- 以avg聚合函数为例
postgres=# select * from pg_aggregate where aggfnoid = 2101;
-[ RECORD 1 ]----+-------------------
aggfnoid         | pg_catalog.avg
aggkind          | n
aggnumdirectargs | 0
aggtransfn       | int4_avg_accum    -- 转换函数
aggfinalfn       | int8_avg          -- 最终处理函数
aggcombinefn     | int4_avg_combine  -- 合并函数（组合函数）
aggserialfn      | -
aggdeserialfn    | -
aggmtransfn      | int4_avg_accum    
aggminvtransfn   | int4_avg_accum_inv
aggmfinalfn      | int8_avg
aggfinalextra    | f
aggmfinalextra   | f
aggfinalmodify   | r
aggmfinalmodify  | r
aggsortop        | 0
aggtranstype     | 1016
aggtransspace    | 0
aggmtranstype    | 1016
aggmtransspace   | 0
agginitval       | {0,0}      -- 初始值{sum, count}
aggminitval      | {0,0}
```
在avg聚合实现过程中，需要中间值`{sum, count}`。
```c++
typedef struct Int8TransTypeData
{
	int64		count;
	int64		sum;
} Int8TransTypeData;

// avg转换函数, sum+=newval,count++
Datum int4_avg_accum(PG_FUNCTION_ARGS)
{
	ArrayType  *transarray;
	int32		newval = PG_GETARG_INT32(1);
	Int8TransTypeData *transdata;
  // ...

	transdata = (Int8TransTypeData *) ARR_DATA_PTR(transarray);
	transdata->count++;
	transdata->sum += newval;

	PG_RETURN_ARRAYTYPE_P(transarray);
}

// 组合函数, 	
// state1->count += state2->count;
// state1->sum += state2->sum;
Datum int4_avg_combine(PG_FUNCTION_ARGS)
{
	ArrayType  *transarray1;
	ArrayType  *transarray2;
	Int8TransTypeData *state1;
	Int8TransTypeData *state2;

	if (!AggCheckCallContext(fcinfo, NULL))
		elog(ERROR, "aggregate function called in non-aggregate context");

	transarray1 = PG_GETARG_ARRAYTYPE_P(0);
	transarray2 = PG_GETARG_ARRAYTYPE_P(1);

  // ...

	state1 = (Int8TransTypeData *) ARR_DATA_PTR(transarray1);
	state2 = (Int8TransTypeData *) ARR_DATA_PTR(transarray2);

	state1->count += state2->count;
	state1->sum += state2->sum;

	PG_RETURN_ARRAYTYPE_P(transarray1);
}

// 最终处理函数，avg = sum/count
Datum int8_avg(PG_FUNCTION_ARGS)
{
	ArrayType  *transarray = PG_GETARG_ARRAYTYPE_P(0);
	Int8TransTypeData *transdata;
	Datum		countd,sumd;

	transdata = (Int8TransTypeData *) ARR_DATA_PTR(transarray);

	/* SQL defines AVG of no values to be NULL */
	if (transdata->count == 0)
		PG_RETURN_NULL();

	countd = NumericGetDatum(int64_to_numeric(transdata->count));
	sumd = NumericGetDatum(int64_to_numeric(transdata->sum));
  // 除法计算
	PG_RETURN_DATUM(DirectFunctionCall2(numeric_div, sumd, countd));
}
```

#### 并行聚合

PostgreSQL支持并行聚合，那么并行聚合是如何实现的呢？整体思路就是每个工作进程执行一部分聚合任务，产生部分聚合状态，然后由主进程通过组合函数combinefunc进行最终聚合。combinefunc负责将两个部分聚合状态internal_state合并，主进程最终对合并后的状态调用finalfunc得到最终结果。对于组合函数，我们举个例子：比如对sum()：combinefunc(state1, state2), 返回state1+state2。 并行聚合的实现要点如下：
- 并行扫描：多个工作进程并行扫描表数据
- 本地聚合：每个工作进程对自己扫描到的数据做本地聚合
- 全局聚合：主进程将个工作进程的聚合结果做最终聚集

我们查看一个并行聚合的执行计划：
```sql
-- 为了能触发并行聚合，设置min_parallel_table_scan_size为0
postgres=# set min_parallel_table_scan_size = 0;
SET
postgres=# create table bigtable(id serial primary key, amount int);
CREATE TABLE
-- 插入数据
postgres=# INSERT INTO bigtable(amount) SELECT (random() * 1000)::int FROM generate_series(1,1000000);
INSERT 0 1000000
-- 并行聚合
postgres=# explain select sum(amount) from bigtable where amount > 100;
                                        QUERY PLAN                                         
-------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=10981.05..10981.06 rows=1 width=8)   -- 主进程进行最终聚合
   ->  Gather  (cost=10980.83..10981.04 rows=2 width=8)   -- 主进程收集子进程的结果
         Workers Planned: 2
         ->  Partial Aggregate  (cost=9980.83..9980.84 rows=1 width=8)   -- 工作进程本地聚合
               ->  Parallel Seq Scan on bigtable  (cost=0.00..9633.59 rows=138896 width=4) 
                     Filter: (amount > 100)
(6 rows)
```

#### 源码分析

`ExecAgg`是执行聚合的核心函数，我们看一下它的实现。
```c++
static TupleTableSlot *ExecAgg(PlanState *pstate)
{
	AggState   *node = castNode(AggState, pstate);
	TupleTableSlot *result = NULL;

	if (!node->agg_done)
	{
    	// 根据策略的不同，选择不同的方式进行计算
		switch (node->phase->aggstrategy)
		{
			case AGG_HASHED:   // 哈希聚合
				if (!node->table_filled)   // 如果哈希表没有填充完成，
					agg_fill_hash_table(node); // 从底层算子获取所有元组，进行哈希分组，执行转换函数，填充哈希表
				/* FALLTHROUGH */
			case AGG_MIXED:  
				result = agg_retrieve_hash_table(node);  // 如果哈希表填充完成，则从哈希表中获取聚集结果
				break;
			case AGG_PLAIN:  // 普通聚合
			case AGG_SORTED:  // 分组聚合
				result = agg_retrieve_direct(node);
				break;
		}

		if (!TupIsNull(result))
			return result;
	}

	return NULL;
}
```

对哈希聚合，通过`agg_fill_hash_table`函数从底层算子（例如扫描算子）获取数据，进行哈希分组，执行转换函数，构建哈希表。
```c++
/* 读取所有输入元组，构建哈希表 */
static void agg_fill_hash_table(AggState *aggstate)
{
	TupleTableSlot *outerslot;
	ExprContext *tmpcontext = aggstate->tmpcontext;

  // 获取输入元组
	for (;;)
	{
		outerslot = fetch_input_tuple(aggstate);
		if (TupIsNull(outerslot))
			break;

		/* set up for lookup_hash_entries and advance_aggregates */
		tmpcontext->ecxt_outertuple = outerslot;

    // 查找或者构建当前输入元组对应的哈希表项，即分组对应的哈希桶
		lookup_hash_entries(aggstate); 

		/* 执行转换函数或组合函数，更新中间结果 */
		advance_aggregates(aggstate);

		/* Reset per-input-tuple context after each tuple, but note that the hash lookups do this too */
		ResetExprContext(aggstate->tmpcontext);
	}

  // 如果构建哈希表过程中有数据溢出写到磁盘，则进行数据溢出处理
	hashagg_finish_initial_spills(aggstate);

	aggstate->table_filled = true;    // 哈希表已填充完成
	/* Initialize to walk the first hash table */
	select_current_set(aggstate, 0, true);
	ResetTupleHashIterator(aggstate->perhash[0].hashtable, &aggstate->perhash[0].hashiter);
}
```
当哈希表填充完成后，会调用`agg_retrieve_hash_table`函数从填充完成的哈希表中获取聚集结果，考虑到哈希表溢出到磁盘的情况，在读取聚集结果时，先通过`agg_retrieve_hash_table_in_memory`函数从内存中的哈希表中获取结果，该函数内部会调用`finalize_aggregates`完成最终计算。当内存中的哈希表已获取完成时，调用`agg_refill_hash_table`函数将磁盘中的哈希表数据填充到内存哈希表中，再次进行读取，直到内存和磁盘中的所有分组都处理完成。
```c++
static TupleTableSlot *agg_retrieve_hash_table(AggState *aggstate)
{
	TupleTableSlot *result = NULL;

	while (result == NULL)
	{
		result = agg_retrieve_hash_table_in_memory(aggstate);
		if (result == NULL)
		{
			if (!agg_refill_hash_table(aggstate))
			{
				aggstate->agg_done = true;
				break;
			}
		}
	}

	return result;
}
```
对于通过排序方式实现分组聚合，通过`agg_retrieve_direct`函数负责处理，要求下层算子为排序好的数据，当发现与前行相同时，则将当前行加入当前分组，否则创建一个新的分组。
```c++
static TupleTableSlot *agg_retrieve_direct(AggState *aggstate)
{
    // 如果还没有完成聚合
	  while (!aggstate->agg_done)
	  {   
        // 初始化聚合状态
        initialize_aggregates(aggstate, pergroups, numReset);

        for (;;)
        {
            advance_aggregates(aggstate); // 执行转换函数或组合函数
            
            outerslot = fetch_input_tuple(aggstate);  // 获取下一个输入元组
         		if (TupIsNull(outerslot))
					  {
                /* no more outer-plan tuples available */

                /* if we built hash tables, finalize any spills */
                if (aggstate->aggstrategy == AGG_MIXED &&
                  aggstate->current_phase == 1)
                  hashagg_finish_initial_spills(aggstate);

                if (hasGroupingSets)
                {
                  aggstate->input_done = true;
                  break;
                }
                else
                {
                  aggstate->agg_done = true;  // 聚合结束
                  break;
                }
					  } 
         }
         	// 完成最终聚合计算，会调用finalize_aggregate函数
          finalize_aggregates(aggstate, peragg, pergroups[currentSet]);
    }
}
```


---

参考文档：
[PostgreSQL 技术内幕(三)聚集算子](https://mp.weixin.qq.com/s/PatFKIuIr2dA2G5kebvdLA)