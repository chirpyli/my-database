## PostgreSQL拓展——auto_explain

auto_explain是PostgreSQL的一个插件，用于自动记录执行时间超过指定阈值的查询的执行计划。这个功能对于优化查询性能非常有用，因为它可以帮助开发人员了解哪些查询是性能瓶颈，从而进行相应的优化。

auto_explain的实现非常简单，它通过在执行查询之前和之后记录一些关键信息，然后比较这些信息（时间信息）来判断查询是否超过了指定的阈值。如果超过了阈值，auto_explain就会记录查询的执行计划，并将其输出到日志文件中。


### 插件的使用
要使用auto_explain插件，首先需要将其编译并安装到PostgreSQL中。加载插件：
```shell
share_preload_libraries = 'auto_explain' # 加载auto_explain插件
```
然后，可以在postgresql.conf文件中设置auto_explain的相关参数，例如，设置参数：
```shell
auto_explain.log_min_duration = 0 # 记录所有查询的执行计划
auto_explain.log_analyze = true # 记录查询的分析计划
```
这样，当查询的执行时间超过0毫秒时，auto_explain就会记录查询的执行计划，并将其输出到日志文件中。

常用配置参数的含义如下：
- auto_explain.log_min_duration：指定记录执行时间超过多少毫秒的查询的执行计划。
- auto_explain.log_analyze：指定是否记录查询的分析计划（explain analyze输出，默认是explain输出）。
- auto_explain.log_buffers：指定是否记录查询使用的缓冲区信息。
- auto_explain.log_nested_statements：指定是否记录嵌套查询的执行计划。
- auto_explain.log_timing：指定是否记录查询的执行时间。
- auto_explain.log_verbose：指定是否记录查询的详细信息，例如查询的输入和输出行数等。

>更多可查看[auto_explain中文文档](http://postgres.cn/docs/15/auto-explain.html)

示例：
```sql
postgres=# set auto_explain.log_min_duration to 0;
SET
postgres=# select * from t1;        -- 查询日志
 a 
---
 1
(1 row)
```
查看日志：
```sql
2024-08-20 16:33:26.790 CST [5891] LOG:  duration: 0.016 ms  plan:
	Query Text: select * from t1;
	Seq Scan on t1  (cost=0.00..35.50 rows=2550 width=4)
```

### 实现思路
具体实现上，关键是记录运行时间，需要在执行前后进行记录，即在ExecutorRun中，添加相应的代码，如果执行的时间超过GUC参数设置的阈值则根据执行器的执行计划节点输出执行计划
```c++
void standard_ExecutorRun(QueryDesc *queryDesc, ScanDirection direction, uint64 count, bool execute_once)
{

	/* Allow instrumentation of Executor overall runtime */
	if (queryDesc->totaltime)
		InstrStartNode(queryDesc->totaltime);   // 记录开始时间

	if (!ScanDirectionIsNoMovement(direction))
	{
		if (execute_once && queryDesc->already_executed)
			elog(ERROR, "can't re-execute query flagged for single execution");
		queryDesc->already_executed = true;
        // 节点的实际执行
		ExecutePlan(estate, queryDesc->planstate,
					queryDesc->plannedstmt->parallelModeNeeded,
					operation, sendTuples, count,
					direction, dest, execute_once);
	}

    // 记录运行结束的时间，计算执行的耗时，当然，不只是时间，还计算所用的buffer、wal等情况
	if (queryDesc->totaltime)
		InstrStopNode(queryDesc->totaltime, estate->es_processed);  
}
```
调用栈：
```c++
InstrStartNode(Instrumentation * instr) (src\backend\executor\instrument.c:80)
standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\execMain.c:333)
auto_explain.so!explain_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (contrib\auto_explain\auto_explain.c:334)
ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\execMain.c:303)
PortalRunSelect(Portal portal, _Bool forward, long count, DestReceiver * dest) (src\backend\tcop\pquery.c:921)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:765)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1217)
```
关键函数，在执行前后计算运行时间，buffer、wal等信息：
```c++
/* Entry to a plan node */
void InstrStartNode(Instrumentation *instr)
{
	if (instr->need_timer &&
		!INSTR_TIME_SET_CURRENT_LAZY(instr->starttime))     // 设置开始的时间
		elog(ERROR, "InstrStartNode called twice in a row");

	/* save buffer usage totals at node entry, if needed */
	if (instr->need_bufusage)
		instr->bufusage_start = pgBufferUsage;

	if (instr->need_walusage)
		instr->walusage_start = pgWalUsage;
}

/* Exit from a plan node */
void InstrStopNode(Instrumentation *instr, double nTuples)
{
	double		save_tuplecount = instr->tuplecount;
	instr_time	endtime;

	/* count the returned tuples */
	instr->tuplecount += nTuples;

	/* let's update the time only if the timer was requested */
	if (instr->need_timer)
	{
		if (INSTR_TIME_IS_ZERO(instr->starttime))
			elog(ERROR, "InstrStopNode called without start");

		INSTR_TIME_SET_CURRENT(endtime);
		INSTR_TIME_ACCUM_DIFF(instr->counter, endtime, instr->starttime);   // 计算执行时间

		INSTR_TIME_SET_ZERO(instr->starttime);
	}

	/* Add delta of buffer usage since entry to node's totals */
	if (instr->need_bufusage)
		BufferUsageAccumDiff(&instr->bufusage, &pgBufferUsage, &instr->bufusage_start);

	if (instr->need_walusage)
		WalUsageAccumDiff(&instr->walusage, &pgWalUsage, &instr->walusage_start);

	/* Is this the first tuple of this cycle? */
	if (!instr->running)
	{
		instr->running = true;
		instr->firsttuple = INSTR_TIME_GET_DOUBLE(instr->counter);
	}
	else
	{
		/* In async mode, if the plan node hadn't emitted any tuples before, this might be the first tuple */
		if (instr->async_mode && save_tuplecount < 1.0)
			instr->firsttuple = INSTR_TIME_GET_DOUBLE(instr->counter);
	}
}
```

### 源码分析
因为这个插件的源代码很少，只有几百行，所以就直接把主要代码全贴在这里了：[auto_explain.c](https://github.com/postgres/postgres/blob/master/contrib/auto_explain/auto_explain.c)
```c++
/* GUC variables */ //GUC参数变量
static int	auto_explain_log_min_duration = -1; /* msec or -1 */
static bool auto_explain_log_analyze = false;
static bool auto_explain_log_verbose = false;
static bool auto_explain_log_buffers = false;
static bool auto_explain_log_wal = false;
static bool auto_explain_log_triggers = false;
static bool auto_explain_log_timing = true;
static bool auto_explain_log_settings = false;
static int	auto_explain_log_format = EXPLAIN_FORMAT_TEXT;
static int	auto_explain_log_level = LOG;
static bool auto_explain_log_nested_statements = false;
static double auto_explain_sample_rate = 1;

static const struct config_enum_entry format_options[] = {
    // ...
};

static const struct config_enum_entry loglevel_options[] = {
    // ...
};

/* Current nesting depth of ExecutorRun calls */
static int	nesting_level = 0;

/* Is the current top-level query to be sampled? */
static bool current_query_sampled = false;

#define auto_explain_enabled() \
	(auto_explain_log_min_duration >= 0 && \
	 (nesting_level == 0 || auto_explain_log_nested_statements) && \
	 current_query_sampled)

/* Saved hook values in case of unload */
static ExecutorStart_hook_type prev_ExecutorStart = NULL;
static ExecutorRun_hook_type prev_ExecutorRun = NULL;
static ExecutorFinish_hook_type prev_ExecutorFinish = NULL;
static ExecutorEnd_hook_type prev_ExecutorEnd = NULL;

void		_PG_init(void);
void		_PG_fini(void);

// 钩子函数实现auto explain
static void explain_ExecutorStart(QueryDesc *queryDesc, int eflags);
static void explain_ExecutorRun(QueryDesc *queryDesc, ScanDirection direction, uint64 count, bool execute_once);
static void explain_ExecutorFinish(QueryDesc *queryDesc);
static void explain_ExecutorEnd(QueryDesc *queryDesc);

// 加载插件时执行该函数，定义了auto explain的GUC变量以及钩子函数
void _PG_init(void)
{
	/* Define custom GUC variables. */
	DefineCustomIntVariable("auto_explain.log_min_duration",
							"Sets the minimum execution time above which plans will be logged.",
							"Zero prints all plans. -1 turns this feature off.",
							&auto_explain_log_min_duration,
                            // ...
                            );

	DefineCustomBoolVariable("auto_explain.log_analyze",
							 "Use EXPLAIN ANALYZE for plan logging.",
							 NULL,
							 &auto_explain_log_analyze,
                            // ...
                            );

	DefineCustomBoolVariable("auto_explain.log_settings",
							 "Log modified configuration parameters affecting query planning.",
							 NULL,
							 &auto_explain_log_settings,
                            // ...
							 );

	DefineCustomBoolVariable("auto_explain.log_verbose",
							 "Use EXPLAIN VERBOSE for plan logging.",
							 NULL,
							 &auto_explain_log_verbose,
                            // ...
							 );

	DefineCustomBoolVariable("auto_explain.log_buffers",
							 "Log buffers usage.",
							 NULL,
							 &auto_explain_log_buffers,
                            // ...
							 );

	DefineCustomBoolVariable("auto_explain.log_wal",
							 "Log WAL usage.",
							 NULL,
							 &auto_explain_log_wal,
                            // ...
							 );

	DefineCustomBoolVariable("auto_explain.log_triggers",
							 "Include trigger statistics in plans.",
							 "This has no effect unless log_analyze is also set.",
							 &auto_explain_log_triggers,
                            // ...
							 );

	DefineCustomEnumVariable("auto_explain.log_format",
							 "EXPLAIN format to be used for plan logging.",
							 NULL,
							 &auto_explain_log_format,
                            // ...
							 );

	DefineCustomEnumVariable("auto_explain.log_level",
							 "Log level for the plan.",
							 NULL,
							 &auto_explain_log_level,
                             // ...
							 );

	DefineCustomBoolVariable("auto_explain.log_nested_statements",
							 "Log nested statements.",
							 NULL,
							 &auto_explain_log_nested_statements,
                            // ...
							 );

	DefineCustomBoolVariable("auto_explain.log_timing",
							 "Collect timing data, not just row counts.",
							 NULL,
							 &auto_explain_log_timing,
                            // ...
							 );

	DefineCustomRealVariable("auto_explain.sample_rate",
							 "Fraction of queries to process.",
							 NULL,
							 &auto_explain_sample_rate,
                            // ...
							 );

	EmitWarningsOnPlaceholders("auto_explain");

	/* Install hooks. */
	prev_ExecutorStart = ExecutorStart_hook;
	ExecutorStart_hook = explain_ExecutorStart;
	prev_ExecutorRun = ExecutorRun_hook;
	ExecutorRun_hook = explain_ExecutorRun;
	prev_ExecutorFinish = ExecutorFinish_hook;
	ExecutorFinish_hook = explain_ExecutorFinish;
	prev_ExecutorEnd = ExecutorEnd_hook;
	ExecutorEnd_hook = explain_ExecutorEnd;
}

/* ExecutorStart hook: start up logging if needed */
static void explain_ExecutorStart(QueryDesc *queryDesc, int eflags)
{
	/*
	 * At the beginning of each top-level statement, decide whether we'll
	 * sample this statement.  If nested-statement explaining is enabled,
	 * either all nested statements will be explained or none will.
	 *
	 * When in a parallel worker, we should do nothing, which we can implement
	 * cheaply by pretending we decided not to sample the current statement.
	 * If EXPLAIN is active in the parent session, data will be collected and
	 * reported back to the parent, and it's no business of ours to interfere.
	 */
	if (nesting_level == 0)
	{
		if (auto_explain_log_min_duration >= 0 && !IsParallelWorker())
			current_query_sampled = (random() < auto_explain_sample_rate *
									 ((double) MAX_RANDOM_VALUE + 1));
		else
			current_query_sampled = false;
	}

	if (auto_explain_enabled())
	{
		/* Enable per-node instrumentation iff log_analyze is required. */
		if (auto_explain_log_analyze && (eflags & EXEC_FLAG_EXPLAIN_ONLY) == 0)
		{
			if (auto_explain_log_timing)
				queryDesc->instrument_options |= INSTRUMENT_TIMER;
			else
				queryDesc->instrument_options |= INSTRUMENT_ROWS;
			if (auto_explain_log_buffers)
				queryDesc->instrument_options |= INSTRUMENT_BUFFERS;
			if (auto_explain_log_wal)
				queryDesc->instrument_options |= INSTRUMENT_WAL;
		}
	}

	if (prev_ExecutorStart)
		prev_ExecutorStart(queryDesc, eflags);
	else
		standard_ExecutorStart(queryDesc, eflags);

	if (auto_explain_enabled())
	{
		/*
		 * Set up to track total elapsed time in ExecutorRun.  Make sure the
		 * space is allocated in the per-query context so it will go away at ExecutorEnd.
		 */
		if (queryDesc->totaltime == NULL)
		{
			MemoryContext oldcxt;

			oldcxt = MemoryContextSwitchTo(queryDesc->estate->es_query_cxt);
			queryDesc->totaltime = InstrAlloc(1, INSTRUMENT_ALL, false);    // 指定收集哪些信息
			MemoryContextSwitchTo(oldcxt);
		}
	}
}

/* ExecutorRun hook: all we need do is track nesting depth */
static void explain_ExecutorRun(QueryDesc *queryDesc, ScanDirection direction, uint64 count, bool execute_once)
{
	nesting_level++;
	PG_TRY();
	{
		if (prev_ExecutorRun)
			prev_ExecutorRun(queryDesc, direction, count, execute_once);
		else
			standard_ExecutorRun(queryDesc, direction, count, execute_once);
	}
	PG_FINALLY();
	{
		nesting_level--;
	}
	PG_END_TRY();
}

/* ExecutorFinish hook: all we need do is track nesting depth */
static void explain_ExecutorFinish(QueryDesc *queryDesc)
{
	nesting_level++;
	PG_TRY();
	{
		if (prev_ExecutorFinish)
			prev_ExecutorFinish(queryDesc);
		else
			standard_ExecutorFinish(queryDesc);
	}
	PG_FINALLY();
	{
		nesting_level--;
	}
	PG_END_TRY();
}

/* ExecutorEnd hook: log results if needed */
static void explain_ExecutorEnd(QueryDesc *queryDesc)
{
	if (queryDesc->totaltime && auto_explain_enabled())
	{
		MemoryContext oldcxt;
		double		msec;

		/*
		 * Make sure we operate in the per-query context, so any cruft will be
		 * discarded later during ExecutorEnd.
		 */
		oldcxt = MemoryContextSwitchTo(queryDesc->estate->es_query_cxt);

		/* Make sure stats accumulation is done.   */
		InstrEndLoop(queryDesc->totaltime);

		/* Log plan if duration is exceeded. */
		msec = queryDesc->totaltime->total * 1000.0;
		if (msec >= auto_explain_log_min_duration)  // 是否大于设置的阈值
		{
			ExplainState *es = NewExplainState();

			es->analyze = (queryDesc->instrument_options && auto_explain_log_analyze);
			es->verbose = auto_explain_log_verbose;
			es->buffers = (es->analyze && auto_explain_log_buffers);
			es->wal = (es->analyze && auto_explain_log_wal);
			es->timing = (es->analyze && auto_explain_log_timing);
			es->summary = es->analyze;
			es->format = auto_explain_log_format;
			es->settings = auto_explain_log_settings;

			ExplainBeginOutput(es);
			ExplainQueryText(es, queryDesc);
			ExplainPrintPlan(es, queryDesc);    // 根据输出执行计划
			if (es->analyze && auto_explain_log_triggers)
				ExplainPrintTriggers(es, queryDesc);
			if (es->costs)
				ExplainPrintJITSummary(es, queryDesc);
			ExplainEndOutput(es);

			/* Remove last line break */
			if (es->str->len > 0 && es->str->data[es->str->len - 1] == '\n')
				es->str->data[--es->str->len] = '\0';

			/* Fix JSON to output an object */
			if (auto_explain_log_format == EXPLAIN_FORMAT_JSON)
			{
				es->str->data[0] = '{';
				es->str->data[es->str->len - 1] = '}';
			}

			/* Note: we rely on the existing logging of context or
			 * debug_query_string to identify just which statement is being
			 * reported.  This isn't ideal but trying to do it here would
			 * often result in duplication. */
			ereport(auto_explain_log_level,(errmsg("duration: %.3f ms  plan:\n%s",msec, es->str->data), errhidestmt(true)));
		}

		MemoryContextSwitchTo(oldcxt);
	}

	if (prev_ExecutorEnd)
		prev_ExecutorEnd(queryDesc);
	else
		standard_ExecutorEnd(queryDesc);
}
```

参考文档：
[auto_explain](https://www.postgresql.org/docs/current/auto-explain.html)
[auto_explain源码](https://github.com/postgres/postgres/blob/master/contrib/auto_explain/auto_explain.c)
