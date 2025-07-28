## PostgreSQL执行计划过滤扩展——plan_filter
PostgreSQL提供了一些对执行计划管理控制类的扩展，比如pg_hint_plan可通过注释直接控制执行计划，auto_explain用于自动记录执行时间超过指定阈值的查询的执行计划。这里介绍的plan_filter扩展，用于对代价超过一定阈值的执行计划进行拦截，比如过滤掉某些执行代价比较大的执行计划，这样就能阻止某些查询在数据库上执行低效的执行计划。

### plan_filter介绍

该插件的主要目的，是将执行计划中代价超过一定阈值的计划进行拦截，避免某些低效的执行计划被执行。也可以起到某种程度的SQL防火墙的作用，避免执行一个过大代价的SQL。下面以一个示例为例：

修改配置文件postgresql.conf，启动数据库。
```conf
shared_preload_libraries = 'plan_filter'        # (change requires restart)
```
通过pgbench生成数据，我们测试一下全表扫描执行计划的代价，因为全表扫描的代价普遍非常高，所以我们以此为例。
```sql
postgres=# explain analyze select * from pgbench_accounts;
                                                         QUERY PLAN                                                          
------------------------------------------------------------------------
 Seq Scan on pgbench_accounts  (cost=0.00..26394.00 rows=1000000 width=97) (actual time=1.647..440.851 rows=1000000 loops=1)
 Planning Time: 0.152 ms
 Execution Time: 484.895 ms
(3 rows)
```
配置参数`plan_filter.statement_cost_limit`，设置执行语句代价的上限。默认值为0，表示不进行过滤，没有设置限制。
```sql
postgres=# set plan_filter.statement_cost_limit = 10000;
SET
```
再次执行该查询，发现报错`plan cost limit exceeded`，表示执行计划的代价超出上限，建议重写查询语句以优化查询，降低执行计划代价或者提升`plan_filter.statement_cost_limit`参数上限值。当然这个扩展的本意肯定是让你去优化查询，避免某些低效的执行计划被执行。
```sql
postgres=# explain analyze select * from pgbench_accounts ;
ERROR:  plan cost limit exceeded
HINT:  The plan for your query shows that it would probably have an excessive run time. This may be due to a logic error in the SQL, or it maybe just a very costly query. Rewrite your query or increase the configuration parameter "plan_filter.statement_cost_limit".
```
还可以通过参数`plan_filter.filter_select_only`设置仅将过滤限制在SELECT语句上，默认为false。
```sql
-- update语句
postgres=# explain analyze update pgbench_accounts set abalance = 0; 
ERROR:  plan cost limit exceeded
HINT:  The plan for your query shows that it would probably have an excessive run time. This may be due to a logic error in the SQL, or it maybe just a very costly query. Rewrite your query or increase the configuration parameter "plan_filter.statement_cost_limit".
-- 设置仅对SELECT语句生效
postgres=# set plan_filter.filter_select_only to true;
SET
-- 未拦截update语句
postgres=# explain analyze update pgbench_accounts set abalance = 0; 
                                                              QUERY PLAN                                                              
-------------------------------------------------------------
 Update on pgbench_accounts  (cost=0.00..52786.39 rows=0 width=0) (actual time=20628.109..20628.110 rows=0 loops=1)
   ->  Seq Scan on pgbench_accounts  (cost=0.00..52786.39 rows=1999939 width=10) (actual time=416.778..1278.151 rows=1000000 loops=1)
 Planning Time: 0.180 ms
 Execution Time: 20628.150 ms
(4 rows)
-- select语句被拦截
postgres=# explain analyze select * from pgbench_accounts ;
ERROR:  plan cost limit exceeded
HINT:  The plan for your query shows that it would probably have an excessive run time. This may be due to a logic error in the SQL, or it maybe just a very costly query. Rewrite your query or increase the configuration parameter "plan_filter.statement_cost_limit".
```

注意这里的SELECT并非表示只读语句，因为有的SELECT语句也会修改数据，比如`SELECT FUNCTION`类的语句。

具体使用时，也可以限制某个用户的语句，即配置用户级参数来限制对某个用户有效。提供了更大的灵活性。
```sql
alter role postgres set plan_filter.statement_cost_limit = 10000;
```

### 实现原理

其实现原理非常简单，就是在生成执行计划后，去读取`PlannedStmt->planTree->total_cost`，即执行计划总的代价，然后与预设值的GUC参数值`statement_cost_limit`进行比较，如果大于预设值，则报错进行拦截。
```c++
void _PG_init(void)
{
	/* 自定义GUC参数 */
	DefineCustomRealVariable("plan_filter.statement_cost_limit", ...);
	DefineCustomBoolVariable("plan_filter.module_loaded", ...);
	DefineCustomBoolVariable("plan_filter.filter_select_only", ...);

	/* 注册钩子函数，在planner_hook中执行过滤逻辑 */
	prev_planner_hook = planner_hook;
	planner_hook = limit_func;
}

// 执行过滤逻辑
static PlannedStmt *limit_func(PLANNER_HOOK_PARAMS)
{
	PlannedStmt *result;

	/* this way we can daisy chain planner hooks if necessary */
	if (prev_planner_hook != NULL)
		result = (*prev_planner_hook) (PLANNER_HOOK_ARGS);
	else
		result = standard_planner(PLANNER_HOOK_ARGS);

    // 判断是否只过滤SELECT语句
    if(filter_select_only && parse->commandType != CMD_SELECT)
		return result;

    // 判断是否过滤，为0时不进行过滤，超过0时，判断执行计划代价是否超过限制
	if (statement_cost_limit > 0.0 &&
		result->planTree->total_cost > statement_cost_limit)
		ereport(ERROR,
				(errcode(ERRCODE_STATEMENT_TOO_COMPLEX),
				 errmsg("plan cost limit exceeded"),
			  errhint("The plan for your query shows that it would probably "
					  "have an excessive run time. This may be due to a "
					  "logic error in the SQL, or it maybe just a very "
					  "costly query. Rewrite your query or increase the "
					  "configuration parameter "
					  "\"plan_filter.statement_cost_limit\".")));

	return result;
}
```

### 开源情况
该扩展遵循postgres开源协议，可以放心使用。我们看一下该插件当前的情况，可以看到这个插件基本已经停止维护了。
> github: https://github.com/pgexperts/pg_plan_filter.git
> star: 91
> 最近更新：2021年9月23日

但是因为这个扩展的实现非常简单，所以如果有实际场景需要使用的话，不用担心社区停止维护的事情，掌握原理后自己就可以进行维护。目前仅有一个issue，提出了对事务内的SQL设置一个阈值，即一个事务内的所有SQL总的执行代价如果超出阈值就进行拦截，这个需求，个人认为使用场景有限，如果是以优化SQL为目的，完全没有必要对事务内的所有SQL进行判断，如果是以拦截执行代价过大的SQL为目的，或者拦截不安全的SQL为目的，事务内的某个语句被拦截执行失败，事务同样会回滚，等同于被拦截。所以意义十分有限，当前没有采纳该issue。

### 总结

在实际工程使用时，建议在开发测试阶段使用该插件，而实际生产环境不太建议使用该插件。生成环境更建议使用auto_explain扩展或者结合pg_stat_statements来使用，不要暴力性的直接拦截某个指定代价比较高的执行计划。这样直接拦截掉，缺乏一定的灵活性，以及存在误报的可能性，因为有的时候有的语句执行计划代价就是很高的，没有办法生成更高质量的执行计划，并且代价估算本来就是有误差的，可能因为统计信息的不准确或未及时更新而导致成本计算的不准确，从而可能被误判拦截。在SQL防火墙中，也存在误报的情况，根据某个规则去拦截SQL，只要这个规则比较宽泛，就存在误报的情况，除非规则定义的非常严谨以及用户的使用场景以及SQL相对固定，则可能减缓误报的情况。也可以考虑将此功能放到SQL防火墙中。

而auto_explain是将直接代价比较高的执行计划打印到日志中，并且其阈值的判断`auto_explain.log_min_duration`是依据真实的执行时间去判断（当然，这样与plan_filter相比有利有弊，缺点就是无法拦截SQL，优点也是不拦截SQL，看具体的使用场景），管理员或者开发人员分析日志，查看该执行代价高的执行计划是否合理，如果不合理，则优化相关语句。更具有灵活性，不会产生误报的情况，当然也不会去拦截某个SQL。


