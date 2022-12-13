### PG数据库内核源码分析——CALL

前面我们分析了`CREATE FUNCTION`的源码，接下来我们继续分析一下`CALL`的源码。首先创建一个存储过程。测试如下语句。
```sql
create table ntc(a int);
create procedure myproc(a int) language plpgsql as $$ begin insert into ntc values(a); end; $$;
call myproc(1);
```

#### 创建存储过程
我们简单看一下创建存储过程的调用栈，与前面分析`CREATE FUNCTION`的时候差不多。执行器调用`CreateFunction`创建存储过程，调用到plpgsql中的`plpgsql_validator`。
```c++
plpgsql.so!do_compile(FunctionCallInfo fcinfo, HeapTuple procTup, PLpgSQL_function * function, PLpgSQL_func_hashkey * hashkey, _Bool forValidator) (postgres\src\pl\plpgsql\src\pl_comp.c:542)
plpgsql.so!plpgsql_compile(FunctionCallInfo fcinfo, _Bool forValidator) (postgres\src\pl\plpgsql\src\pl_comp.c:223)
plpgsql.so!plpgsql_validator(FunctionCallInfo fcinfo) (postgres\src\pl\plpgsql\src\pl_handler.c:514)
FunctionCall1Coll(FmgrInfo * flinfo, Oid collation, Datum arg1) (postgres\src\backend\utils\fmgr\fmgr.c:1142)
OidFunctionCall1Coll(Oid functionId, Oid collation, Datum arg1) (postgres\src\backend\utils\fmgr\fmgr.c:1420)
ProcedureCreate(const char * procedureName, Oid procNamespace, _Bool replace, _Bool returnsSet, Oid returnType, Oid proowner, Oid languageObjectId, Oid languageValidator, const char * prosrc, const char * probin, char prokind, _Bool security_definer, _Bool isLeakProof, _Bool isStrict, char volatility, char parallel, oidvector * parameterTypes, Datum allParameterTypes, Datum parameterModes, Datum parameterNames, List * parameterDefaults, Datum trftypes, Datum proconfig, Oid prosupport, float4 procost, float4 prorows) (postgres\src\backend\catalog\pg_proc.c:702)
CreateFunction(ParseState * pstate, CreateFunctionStmt * stmt) (postgres\src\backend\commands\functioncmds.c:1152)
ProcessUtilitySlow(ParseState * pstate, PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (postgres\src\backend\tcop\utility.c:1898)
standard_ProcessUtility(PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (postgres\src\backend\tcop\utility.c:1305)
ProcessUtility(PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (postgres\src\backend\tcop\utility.c:753)
PortalRunUtility(Portal portal, PlannedStmt * pstmt, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, QueryCompletion * qc) (postgres\src\backend\tcop\pquery.c:1161)
PortalRunMulti(Portal portal, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (postgres\src\backend\tcop\pquery.c:1307)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (postgres\src\backend\tcop\pquery.c:783)
exec_simple_query(const char * query_string) (postgres\src\backend\tcop\postgres.c:1326)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (postgres\src\backend\tcop\postgres.c:4445)
BackendRun(Port * port) (postgres\src\backend\postmaster\postmaster.c:4988)
BackendStartup(Port * port) (postgres\src\backend\postmaster\postmaster.c:4672)
ServerLoop() (postgres\src\backend\postmaster\postmaster.c:1959)
PostmasterMain(int argc, char ** argv) (postgres\src\backend\postmaster\postmaster.c:1495)
```

#### 调用存储过程CALL

在语法解析阶段，看一下gram.y中`CALL`语法定义：
```sql
CallStmt:	CALL func_application
				{
					CallStmt *n = makeNode(CallStmt);
					n->funccall = castNode(FuncCall, $2);
					$$ = (Node *)n;
				}
		;
```
由语法解析树调用`pg_analyze`->`transformStmt`->`transformCallStmt`转为查询树。
```c++
/* transform a CallStmt
 * We need to do parse analysis on the procedure call and its arguments */
static Query *transformCallStmt(ParseState *pstate, CallStmt *stmt)
{
	List	   *targs;
	ListCell   *lc;
	Node	   *node;
	Query	   *result;

	targs = NIL;
	foreach(lc, stmt->funccall->args)
	{
		targs = lappend(targs, transformExpr(pstate, (Node *) lfirst(lc), EXPR_KIND_CALL_ARGUMENT));
	}

	node = ParseFuncOrColumn(pstate, stmt->funccall->funcname, targs, pstate->p_last_srf, stmt->funccall, true, stmt->funccall->location);

	assign_expr_collations(pstate, node);
	stmt->funcexpr = castNode(FuncExpr, node);

	result = makeNode(Query);
	result->commandType = CMD_UTILITY;	// 注意，后面无需经过优化器
	result->utilityStmt = (Node *) stmt;

	return result;
}
```

CALL为UTILITY语句，在`pg_plan_queries`中直接被封装为执行计划树。我们继续分析一下执行器。执行器中调用`ExecuteCallStmt`调用存储过程。调用栈如下：
```c++
plpgsql.so!plpgsql_exec_function(PLpgSQL_function * func, FunctionCallInfo fcinfo, EState * simple_eval_estate, ResourceOwner simple_eval_resowner, _Bool atomic) (postgres\src\pl\plpgsql\src\pl_exec.c:485)
plpgsql.so!plpgsql_call_handler(FunctionCallInfo fcinfo) (postgres\src\pl\plpgsql\src\pl_handler.c:265)
ExecuteCallStmt(CallStmt * stmt, ParamListInfo params, _Bool atomic, DestReceiver * dest) (postgres\src\backend\commands\functioncmds.c:2231)
standard_ProcessUtility(PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (postgres\src\backend\tcop\utility.c:1053)
ProcessUtility(PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (postgres\src\backend\tcop\utility.c:753)
PortalRunUtility(Portal portal, PlannedStmt * pstmt, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, QueryCompletion * qc) (postgres\src\backend\tcop\pquery.c:1161)
PortalRunMulti(Portal portal, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (postgres\src\backend\tcop\pquery.c:1307)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (postgres\src\backend\tcop\pquery.c:783)
exec_simple_query(const char * query_string) (postgres\src\backend\tcop\postgres.c:1326)
```
`ExecuteCallStmt`中会调用plpgsql中的`plpgsql_call_handler`。
```c++
Datum plpgsql_call_handler(PG_FUNCTION_ARGS)
{
	bool		nonatomic;
	PLpgSQL_function *func;
	PLpgSQL_execstate *save_cur_estate;
	Datum		retval;
	int			rc;

	nonatomic = fcinfo->context && IsA(fcinfo->context, CallContext) && !castNode(CallContext, fcinfo->context)->atomic;

	/* Connect to SPI manager*/
	if ((rc = SPI_connect_ext(nonatomic ? SPI_OPT_NONATOMIC : 0)) != SPI_OK_CONNECT)
		elog(ERROR, "SPI_connect failed: %s", SPI_result_code_string(rc));

	/* Find or compile the function */
	func = plpgsql_compile(fcinfo, false);		// 在这里查找函数
	save_cur_estate = func->cur_estate;	/* Must save and restore prior value of cur_estate */
	func->use_count++;					/* Mark the function as busy, so it can't be deleted from under us */

	PG_TRY();
	{
		/* Determine if called as function or trigger and call appropriate subhandler */
		if (CALLED_AS_TRIGGER(fcinfo))
			retval = PointerGetDatum(plpgsql_exec_trigger(func, (TriggerData *) fcinfo->context));
		else if (CALLED_AS_EVENT_TRIGGER(fcinfo))
		{
			plpgsql_exec_event_trigger(func, (EventTriggerData *) fcinfo->context);
			retval = (Datum) 0;
		}
		else
			retval = plpgsql_exec_function(func, fcinfo, NULL, NULL, !nonatomic);	// 执行调用存储过程
	}
	PG_FINALLY();
	{
		func->use_count--;		/* Decrement use-count, restore cur_estate */
		func->cur_estate = save_cur_estate;
	}
	PG_END_TRY();

	if ((rc = SPI_finish()) != SPI_OK_FINISH)	/* Disconnect from SPI manager*/
		elog(ERROR, "SPI_finish failed: %s", SPI_result_code_string(rc));

	return retval;
}
```
接下来会调用的spi接口（服务器编程接口），spi能使用户从用户定义的函数内部运行SQL查询，是一组接口函数，用于简化对PG的查询解析器、规划器、优化器和执行器的访问。
```c++
		ExecutePlan(EState * estate, PlanState * planstate, _Bool use_parallel_mode, CmdType operation, _Bool sendTuples, uint64 numberTuples, ScanDirection direction, DestReceiver * dest, _Bool execute_once) (postgres\src\backend\executor\execMain.c:1615)
		standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (postgres\src\backend\executor\execMain.c:351)
		ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (postgres\src\backend\executor\execMain.c:295)
	_SPI_pquery(QueryDesc * queryDesc, _Bool fire_triggers, uint64 tcount) (postgres\src\backend\executor\spi.c:2511)
// 获取执行计划后，调用spi中的接口执行
		pg_plan_queries(List * querytrees, const char * query_string, int cursorOptions, ParamListInfo boundParams) (postgres\src\backend\tcop\postgres.c:1053)
		BuildCachedPlan(CachedPlanSource * plansource, List * qlist, ParamListInfo boundParams, QueryEnvironment * queryEnv) (postgres\src\backend\utils\cache\plancache.c:933)
	GetCachedPlan(CachedPlanSource * plansource, ParamListInfo boundParams, _Bool useResOwner, QueryEnvironment * queryEnv) (postgres\src\backend\utils\cache\plancache.c:1215)
// 调用存储过程，存储过程中含有很多sql语句，依旧需要获取执行计划，然后执行
_SPI_execute_plan(SPIPlanPtr plan, ParamListInfo paramLI, Snapshot snapshot, Snapshot crosscheck_snapshot, _Bool read_only, _Bool fire_triggers, uint64 tcount) (postgres\src\backend\executor\spi.c:2210)
SPI_execute_plan_with_paramlist(SPIPlanPtr plan, ParamListInfo params, _Bool read_only, long tcount) (postgres\src\backend\executor\spi.c:577)
plpgsql.so!exec_stmt_execsql(PLpgSQL_execstate * estate, PLpgSQL_stmt_execsql * stmt) (postgres\src\pl\plpgsql\src\pl_exec.c:4236)
plpgsql.so!exec_stmt(PLpgSQL_execstate * estate, PLpgSQL_stmt * stmt) (postgres\src\pl\plpgsql\src\pl_exec.c:2052)
plpgsql.so!exec_stmts(PLpgSQL_execstate * estate, List * stmts) (postgres\src\pl\plpgsql\src\pl_exec.c:1943)
plpgsql.so!exec_stmt_block(PLpgSQL_execstate * estate, PLpgSQL_stmt_block * block) (postgres\src\pl\plpgsql\src\pl_exec.c:1884)
plpgsql.so!exec_stmt(PLpgSQL_execstate * estate, PLpgSQL_stmt * stmt) (postgres\src\pl\plpgsql\src\pl_exec.c:1976)
plpgsql.so!plpgsql_exec_function(PLpgSQL_function * func, FunctionCallInfo fcinfo, EState * simple_eval_estate, ResourceOwner simple_eval_resowner, _Bool atomic) (postgres\src\pl\plpgsql\src\pl_exec.c:610)
plpgsql.so!plpgsql_call_handler(FunctionCallInfo fcinfo) (postgres\src\pl\plpgsql\src\pl_handler.c:265)
```
到这里以及基本清楚了其调用的过程。用户定义的存储过程，在调用的过程中，还是需要通过spi接口，调用优化器生成执行计划再执行器中执行SQL语句。当然，这个存储过程比较简单，没有涉及到控制语句、重载等面向过程的部分，这里只是用最简单的语句，快速理解其调用过程。其他实现细节，再慢慢研究分析。

最后，我们看一下spi中的关键函数的源代码：
```c++
/* Execute the given plan with the given parameter values
 *
 * snapshot: query snapshot to use, or InvalidSnapshot for the normal
 *		behavior of taking a new snapshot for each query.
 * crosscheck_snapshot: for RI use, all others pass InvalidSnapshot
 * read_only: true for read-only execution (no CommandCounterIncrement)
 * fire_triggers: true to fire AFTER triggers at end of query (normal case);
 *		false means any AFTER triggers are postponed to end of outer query
 * tcount: execution tuple-count limit, or 0 for none */
static int _SPI_execute_plan(SPIPlanPtr plan, ParamListInfo paramLI, Snapshot snapshot, Snapshot crosscheck_snapshot, bool read_only, bool fire_triggers, uint64 tcount)
{
	int			my_res = 0;
	uint64		my_processed = 0;
	SPITupleTable *my_tuptable = NULL;
	int			res = 0;
	bool		pushed_active_snap = false;
	ErrorContextCallback spierrcontext;
	CachedPlan *cplan = NULL;
	ListCell   *lc1;

	/* Setup error traceback support for ereport()*/
	spierrcontext.callback = _SPI_error_callback;
	spierrcontext.arg = NULL;	/* we'll fill this below */
	spierrcontext.previous = error_context_stack;
	error_context_stack = &spierrcontext;

	if (snapshot != InvalidSnapshot && !plan->no_snapshots)
	{
		if (read_only)
		{
			PushActiveSnapshot(snapshot);
			pushed_active_snap = true;
		}
		else
		{
			PushCopiedSnapshot(snapshot);			/* Make sure we have a private copy of the snapshot to modify */
			pushed_active_snap = true;
		}
	}

	foreach(lc1, plan->plancache_list)
	{
		CachedPlanSource *plansource = (CachedPlanSource *) lfirst(lc1);
		List	   *stmt_list;
		ListCell   *lc2;

		spierrcontext.arg = unconstify(char *, plansource->query_string);

		/* If this is a one-shot plan, we still need to do parse analysis.*/
		if (plan->oneshot)
		{
			RawStmt    *parsetree = plansource->raw_parse_tree;
			const char *src = plansource->query_string;
			List	   *stmt_list;

			/* Parameter datatypes are driven by parserSetup hook if provided,
			 * otherwise we use the fixed parameter list. */
			if (parsetree == NULL)
				stmt_list = NIL;
			else if (plan->parserSetup != NULL)
				stmt_list = pg_analyze_and_rewrite_params(parsetree, src, plan->parserSetup, plan->parserSetupArg, _SPI_current->queryEnv);
			
			else
				stmt_list = pg_analyze_and_rewrite(parsetree, src, plan->argtypes, plan->nargs, _SPI_current->queryEnv);
			

			/* Finish filling in the CachedPlanSource */
			CompleteCachedPlan(plansource, stmt_list, NULL, plan->argtypes, plan->nargs,plan->parserSetup,plan->parserSetupArg,plan->cursor_options, false);	/* not fixed result */
		}

		/* Replan if needed, and increment plan refcount.  If it's a saved
		 * plan, the refcount must be backed by the CurrentResourceOwner.*/
		cplan = GetCachedPlan(plansource, paramLI, plan->saved, _SPI_current->queryEnv);		// 获取SQL执行计划
		stmt_list = cplan->stmt_list;

		/* In the default non-read-only case, get a new snapshot, replacing
		 * any that we pushed in a previous cycle.*/
		if (snapshot == InvalidSnapshot && !read_only && !plan->no_snapshots)
		{
			if (pushed_active_snap)
				PopActiveSnapshot();
			PushActiveSnapshot(GetTransactionSnapshot());
			pushed_active_snap = true;
		}

		foreach(lc2, stmt_list)
		{
			PlannedStmt *stmt = lfirst_node(PlannedStmt, lc2);
			bool		canSetTag = stmt->canSetTag;
			DestReceiver *dest;

			_SPI_current->processed = 0;
			_SPI_current->tuptable = NULL;

			if (stmt->utilityStmt)
			{
				if (IsA(stmt->utilityStmt, CopyStmt))
				{
					CopyStmt   *cstmt = (CopyStmt *) stmt->utilityStmt;
					if (cstmt->filename == NULL)
					{
						my_res = SPI_ERROR_COPY;
						goto fail;
					}
				} else if (IsA(stmt->utilityStmt, TransactionStmt))
				{
					my_res = SPI_ERROR_TRANSACTION;
					goto fail;
				}
			}

			if (read_only && !CommandIsReadOnly(stmt))
				ereport(ERROR, (errcode(ERRCODE_FEATURE_NOT_SUPPORTED), errmsg("%s is not allowed in a non-volatile function", CreateCommandName((Node *) stmt))));

			/* If not read-only mode, advance the command counter before each command and update the snapshot.*/
			if (!read_only && !plan->no_snapshots)
			{
				CommandCounterIncrement();
				UpdateActiveSnapshotCommandId();
			}

			dest = CreateDestReceiver(canSetTag ? DestSPI : DestNone);
			if (stmt->utilityStmt == NULL)
			{
				QueryDesc  *qdesc;
				Snapshot	snap;

				if (ActiveSnapshotSet())
					snap = GetActiveSnapshot();
				else
					snap = InvalidSnapshot;

				qdesc = CreateQueryDesc(stmt,plansource->query_string,snap, crosscheck_snapshot,dest,paramLI, _SPI_current->queryEnv,0);
				res = _SPI_pquery(qdesc, fire_triggers, canSetTag ? tcount : 0);
				FreeQueryDesc(qdesc);
			} else
			{
				ProcessUtilityContext context;
				QueryCompletion qc;

				/* If the SPI context is atomic, or we are asked to manage
				 * snapshots, then we are in an atomic execution context.
				 * Conversely, to propagate a nonatomic execution context, the
				 * caller must be in a nonatomic SPI context and manage snapshots itself.*/
				if (_SPI_current->atomic || !plan->no_snapshots)
					context = PROCESS_UTILITY_QUERY;
				else
					context = PROCESS_UTILITY_QUERY_NONATOMIC;

				InitializeQueryCompletion(&qc);
				ProcessUtility(stmt, plansource->query_string, context, paramLI, _SPI_current->queryEnv, dest, &qc);

				/* Update "processed" if stmt returned tuples */
				if (_SPI_current->tuptable)
					_SPI_current->processed = _SPI_current->tuptable->numvals;

				res = SPI_OK_UTILITY;

				/* Some utility statements return a row count, even though the
				 * tuples are not returned to the caller. */
				if (IsA(stmt->utilityStmt, CreateTableAsStmt))
				{
					CreateTableAsStmt *ctastmt = (CreateTableAsStmt *) stmt->utilityStmt;
					if (qc.commandTag == CMDTAG_SELECT)
						_SPI_current->processed = qc.nprocessed;
					else
					{
						/* Must be an IF NOT EXISTS that did nothing, or a CREATE ... WITH NO DATA.*/
						_SPI_current->processed = 0;
					}

					/* For historical reasons, if CREATE TABLE AS was spelled as SELECT INTO, return a special return code.*/
					if (ctastmt->is_select_into)
						res = SPI_OK_SELINTO;
				} 
				else if (IsA(stmt->utilityStmt, CopyStmt))
					_SPI_current->processed = qc.nprocessed;
			}

			/* The last canSetTag query sets the status values returned to the
			 * caller.  Be careful to free any tuptables not returned, to
			 * avoid intra-transaction memory leak. */
			if (canSetTag)
			{
				my_processed = _SPI_current->processed;
				SPI_freetuptable(my_tuptable);
				my_tuptable = _SPI_current->tuptable;
				my_res = res;
			}
			else
			{
				SPI_freetuptable(_SPI_current->tuptable);
				_SPI_current->tuptable = NULL;
			}
			/* we know that the receiver doesn't need a destroy call */
			if (res < 0)
			{
				my_res = res;
				goto fail;
			}
		}

		ReleaseCachedPlan(cplan, plan->saved);		/* Done with this plan, so release refcount */
		cplan = NULL;

		if (!read_only)
			CommandCounterIncrement();
	}

fail:

	/* Pop the snapshot off the stack if we pushed one */
	if (pushed_active_snap)
		PopActiveSnapshot();

	/* We no longer need the cached plan refcount, if any */
	if (cplan)
		ReleaseCachedPlan(cplan, plan->saved);

	error_context_stack = spierrcontext.previous;	/* Pop the error context stack*/

	SPI_processed = my_processed;	/* Save results for caller */
	SPI_tuptable = my_tuptable;

	_SPI_current->tuptable = NULL;	/* tuptable now is caller's responsibility, not SPI's */

	/* If none of the queries had canSetTag, return SPI_OK_REWRITTEN. Prior to
	 * 8.4, we used return the last query's result code, but not its auxiliary
	 * results, but that's confusing. */
	if (my_res == 0)
		my_res = SPI_OK_REWRITTEN;

	return my_res;
}
```