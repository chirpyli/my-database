### Postgres源码分析 —— FunctionScan
本文分析一下FunctionScan的源码，加深一下理解。以`SELECT * FROM generate_series(2,4);`为例进行分析。
```sql
postgres@postgres=# SELECT * FROM generate_series(2,4);
 generate_series 
-----------------
               2
               3
               4
(3 rows)

postgres@postgres=# explain SELECT * FROM generate_series(2,4);
                             QUERY PLAN                             
--------------------------------------------------------------------
 Function Scan on generate_series  (cost=0.00..0.03 rows=3 width=4)
(1 row)
```

#### 查询优化器
解析层面流程如下：
```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
```
语法表示可查看gram.y中的定义。
```c++
simple_select:
			SELECT hint_string opt_all_clause opt_target_list
			into_clause from_clause where_clause
			group_clause having_clause window_clause
				{
					SelectStmt *n = makeNode(SelectStmt);
					n->targetList = $4;
					n->intoClause = $5;
					n->fromClause = $6;
					n->whereClause = $7;
					n->groupClause = ($8)->list;
					n->havingClause = $9;
					n->windowClause = $10;
					n->comment = $2;
					$$ = (Node *)n;
				}
from_clause:
			FROM from_list							{ $$ = $2; }
			| /*EMPTY*/								{ $$ = NIL; }
		;

from_list:
			table_ref								{ $$ = list_make1($1); }
			| from_list ',' table_ref				{ $$ = lappend($1, $3); }
		;

table_ref:	relation_expr opt_alias_clause
				{
					$1->alias = $2;
					$$ = (Node *) $1;
				}
			| func_table func_alias_clause
				{
					RangeFunction *n = (RangeFunction *) $1;
					n->alias = linitial($2);
					n->coldeflist = lsecond($2);
					$$ = (Node *) n;
				}
/*
 * func_table represents a function invocation in a FROM list. It can be
 * a plain function call, like "foo(...)", or a ROWS FROM expression with
 * one or more function calls, "ROWS FROM (foo(...), bar(...))",
 * optionally with WITH ORDINALITY attached.
 * In the ROWS FROM syntax, a column definition list can be given for each
 * function, for example:
 *     ROWS FROM (foo() AS (foo_res_a text, foo_res_b text),
 *                bar() AS (bar_res_a text, bar_res_b text))
 * It's also possible to attach a column definition list to the RangeFunction
 * as a whole, but that's handled by the table_ref production.
 */
func_table: func_expr_windowless opt_ordinality
				{
					RangeFunction *n = makeNode(RangeFunction);
					n->lateral = false;
					n->ordinality = $2;
					n->is_rowsfrom = false;
					n->functions = list_make1(list_make2($1, NIL));
					/* alias and coldeflist are set by table_ref production */
					$$ = (Node *) n;
				}
			| ROWS FROM '(' rowsfrom_list ')' opt_ordinality
				{
					RangeFunction *n = makeNode(RangeFunction);
					n->lateral = false;
					n->ordinality = $6;
					n->is_rowsfrom = true;
					n->functions = $4;
					/* alias and coldeflist are set by table_ref production */
					$$ = (Node *) n;
				}
		;

func_expr_windowless:
			func_application						{ $$ = $1; }
			| func_expr_common_subexpr				{ $$ = $1; }
		;

func_application: func_name '(' ')'
				{
					$$ = (Node *) makeFuncCall($1, NIL, @1);
				}
			| func_name '(' func_arg_list opt_sort_clause ')'
				{
					FuncCall *n = makeFuncCall($1, $3, @1);
					n->agg_order = $4;
					$$ = (Node *)n;
				}
```

关键数据结构：
```c++
// select语句的抽象语法树表示
typedef struct SelectStmt
{
	NodeTag		type;
    // 对应select *
	List	   *targetList;		/* the target list (of ResTarget) */  // *
    // 对应         from generate_series(2,4)
	List	   *fromClause;		/* the FROM clause */  // 保存RangeFunction节点

    // ...... 

} SelectStmt;

// 范围表：函数类型范围表
/*
 * RangeFunction - function call appearing in a FROM clause
 */
typedef struct RangeFunction
{
	NodeTag		type;
	bool		lateral;		/* does it have LATERAL prefix? */
	bool		ordinality;		/* does it have WITH ORDINALITY suffix? */
	bool		is_rowsfrom;	/* is result of ROWS FROM() syntax? */
	List	   *functions;		/* per-function information, see above */
	Alias	   *alias;			/* table alias & optional column aliases */
	List	   *coldeflist;		/* list of ColumnDef nodes to describe result of function returning RECORD */
} RangeFunction;

// 函数调用 ，函数名，参数列表
typedef struct FuncCall
{
	NodeTag		type;
	List	   *funcname;		/* qualified name of function */
	List	   *args;			/* the arguments (list of exprs) */
	List	   *agg_order;		/* ORDER BY (list of SortBy) */
	Node	   *agg_filter;		/* FILTER clause, if any */
	bool		agg_within_group;	/* ORDER BY appeared in WITHIN GROUP */
	bool		agg_star;		/* argument was really '*' */
	bool		agg_distinct;	/* arguments were labeled DISTINCT */
	bool		func_variadic;	/* last argument was labeled VARIADIC */
	struct WindowDef *over;		/* OVER clause, if any */
	int			location;		/* token location, or -1 if unknown */
} FuncCall;
```


语义分析流程：
```c++
pg_analyze_and_rewrite
--> pg_analyze
    --> transformStmt
        --> transformSelectStmt
            --> transformFromClause  // 处理Function范围表
                --> transformFromClauseItem
                    --> transformRangeFunction
                        --> transformExpr
                            --> transformFuncCall
                                --> ParseFuncOrColumn // 构造FuncExpr
                        --> addRangeTableEntryForFunction
                            --> get_expr_result_type
                                --> internal_get_result_type
                                    --> build_function_result_tupdesc_t
                                    --> get_type_func_class
                            --> buildRelationAliases
                            --> buildNSItemFromTupleDesc
            --> transformTargetList
--> pg_rewrite_query
```

关键数据结构：
```c++
/* FuncExpr - expression node for a function call*/
typedef struct FuncExpr
{
	Expr		xpr;
	Oid			funcid;			/* PG_PROC OID of the function */
	Oid			funcresulttype; /* PG_TYPE OID of result value */
	bool		funcretset;		/* true if function returns set */
	bool		funcvariadic;	/* true if variadic arguments have been
								 * combined into an array last argument */
	CoercionForm funcformat;	/* how to display this function call */
	Oid			funccollid;		/* OID of collation of result */
	Oid			inputcollid;	/* OID of collation that function should use */
	List	   *args;			/* arguments to the function */
	int			location;		/* token location, or -1 if unknown */
} FuncExpr;

/*
 * RangeTblFunction -
 *	  RangeTblEntry subsidiary data for one function in a FUNCTION RTE.
 */
typedef struct RangeTblFunction
{
	NodeTag		type;

	Node	   *funcexpr;		/* expression tree for func call */
	int			funccolcount;	/* number of columns it contributes to RTE */
	/* These fields record the contents of a column definition list, if any: */
	List	   *funccolnames;	/* column names (list of String) */
	List	   *funccoltypes;	/* OID list of column type OIDs */
	List	   *funccoltypmods; /* integer list of column typmods */
	List	   *funccolcollations;	/* OID list of column collation OIDs */
	/* This is set during planning for use by the executor: */
	Bitmapset  *funcparams;		/* PARAM_EXEC Param IDs affecting this func */
} RangeTblFunction;
```
最终生成查询树`Query`，下一步是生成执行计划。

优化器主流程：
```c++
pg_plan_queries
--> pg_plan_query
    --> planner
        --> standard_planner   调用标准优化器
```
```c++
standard_planner
--> subquery_planner
    --> preprocess_function_rtes
    --> pull_up_subqueries  // 上拉子查询
    --> grouping_planner
        --> query_planner
            --> setup_simple_rel_arrays
            --> add_base_rels_to_query  // 构造RelOptInfo
                --> build_simple_rel
            --> make_one_rel
                --> set_base_rel_sizes
                    --> set_rel_size
                        --> set_function_size_estimates
                            --> set_baserel_size_estimates
                --> set_base_rel_pathlists
                    --> set_rel_pathlist
                        --> set_function_pathlist
                            --> create_functionscan_path  // 生成FunctionScanPath
                --> make_rel_from_joinlist
        --> apply_scanjoin_target_to_paths
--> create_plan
    --> create_scan_plan
        --> create_functionscan_plan
            --> make_functionscan
```
核心函数：
```c++
FunctionScan *create_functionscan_plan(PlannerInfo *root, Path *best_path, List *tlist, List *scan_clauses)
{
	FunctionScan *scan_plan;
	Index		scan_relid = best_path->parent->relid;
	RangeTblEntry *rte;
	List	   *functions;

	rte = planner_rt_fetch(scan_relid, root);
	functions = rte->functions;   // 函数表达式 

	/* Sort clauses into best execution order */
	scan_clauses = order_qual_clauses(root, scan_clauses);

	/* Reduce RestrictInfo list to bare expressions; ignore pseudoconstants */
	scan_clauses = extract_actual_clauses(scan_clauses, false);

	/* Replace any outer-relation variables with nestloop params */
	if (best_path->param_info)
	{
		scan_clauses = (List *)replace_nestloop_params(root, (Node *) scan_clauses);
		/* The function expressions could contain nestloop params, too */
		functions = (List *) replace_nestloop_params(root, (Node *) functions);
	}

	scan_plan = make_functionscan(tlist, scan_clauses, scan_relid, functions, rte->funcordinality);
	copy_generic_path_info(&scan_plan->scan.plan, best_path);

	return scan_plan;
}
```


#### 执行器
生成执行计划后，进入执行器。
主流程如下：
```c++
PortalStart
--> ExecutorStart
    --> InitPlan
        --> ExecInitFunctionScan
            --> ExecInitTableFunctionResult
            --> get_expr_result_type
            --> ExecInitScanTupleSlot
            --> ExecInitResultTypeTL
                --> ExecTypeFromTL
            --> ExecAssignScanProjectionInfo
PortalRun
--> ExecutorRun
    --> ExecutePlan
        --> ExecFunctionScan
            --> ExecScan
                --> ExecScanFetch
                    --> FunctionNext   // 首先获取函数的结果集保存起来
                        --> ExecMakeTableFunctionResult
                            --> FunctionCallInvoke
                                --> generate_series_int4
                                    --> generate_series_step_int4
                        --> tuplestore_gettupleslot  // 第一次执行获得结果集后，每次从tuplestore中取出
PortalDrop
```

核心代码：
```c++
/* ----------------------------------------------------------------
 *		ExecFunctionScan(node)
 *
 *		Scans the function sequentially and returns the next qualifying
 *		tuple.
 *		We call the ExecScan() routine and pass it the appropriate
 *		access method functions.
 * ----------------------------------------------------------------*/
TupleTableSlot *ExecFunctionScan(PlanState *pstate)
{
	FunctionScanState *node = castNode(FunctionScanState, pstate);

	return ExecScan(&node->ss,
					(ExecScanAccessMtd) FunctionNext,
					(ExecScanRecheckMtd) FunctionRecheck);
}

TupleTableSlot *
ExecScan(ScanState *node,
		 ExecScanAccessMtd accessMtd,	/* function returning a tuple */
		 ExecScanRecheckMtd recheckMtd)
{
	ExprContext *econtext;
	ExprState  *qual;
	ProjectionInfo *projInfo;

	/* Fetch data from node */
	qual = node->ps.qual;
	projInfo = node->ps.ps_ProjInfo;
	econtext = node->ps.ps_ExprContext;

	/* interrupt checks are in ExecScanFetch */

	/* If we have neither a qual to check nor a projection to do, just skip
	 * all the overhead and return the raw scan tuple.*/
	if (!qual && !projInfo)
	{
		ResetExprContext(econtext);
		return ExecScanFetch(node, accessMtd, recheckMtd);
	}

	/* Reset per-tuple memory context to free any expression evaluation
	 * storage allocated in the previous tuple cycle.*/
	ResetExprContext(econtext);

	/* get a tuple from the access method.  Loop until we obtain a tuple that
	 * passes the qualification.*/
	for (;;)
	{
		TupleTableSlot *slot;

		slot = ExecScanFetch(node, accessMtd, recheckMtd);

        // ......
    }
}

/*
 * ExecScanFetch -- check interrupts & fetch next potential tuple
 *
 * This routine is concerned with substituting a test tuple if we are
 * inside an EvalPlanQual recheck.  If we aren't, just execute
 * the access method's next-tuple routine. */
TupleTableSlot *
ExecScanFetch(ScanState *node,
			  ExecScanAccessMtd accessMtd,
			  ExecScanRecheckMtd recheckMtd)
{
	EState	   *estate = node->ps.state;

	CHECK_FOR_INTERRUPTS();

	if (estate->es_epq_active != NULL)
	{
		// ......
	}

	/* Run the node-type-specific access method function to get the next tuple */
	return (*accessMtd) (node);
}

/* ----------------------------------------------------------------
 *		FunctionNext
 *
 *		This is a workhorse for ExecFunctionScan
 * ---------------------------------------------------------------- */
TupleTableSlot *FunctionNext(FunctionScanState *node)
{
	EState	   *estate;
	ScanDirection direction;
	TupleTableSlot *scanslot;
	bool		alldone;
	int64		oldpos;
	int			funcno;
	int			att;

	/* get information from the estate and scan state */
	estate = node->ss.ps.state;
	direction = estate->es_direction;
	scanslot = node->ss.ss_ScanTupleSlot;

	if (node->simple)
	{
		/* Fast path for the trivial case: the function return type and scan
		 * result type are the same, so we fetch the function result straight
		 * into the scan result slot. No need to update ordinality or rowcounts either. */
		Tuplestorestate *tstore = node->funcstates[0].tstore;

		/*
		 * If first time through, read all tuples from function and put them
		 * in a tuplestore. Subsequent calls just fetch tuples from
		 * tuplestore.
		 */
		if (tstore == NULL)
		{
			node->funcstates[0].tstore = tstore =
				ExecMakeTableFunctionResult(node->funcstates[0].setexpr,
											node->ss.ps.ps_ExprContext,
											node->argcontext,
											node->funcstates[0].tupdesc,
											node->eflags & EXEC_FLAG_BACKWARD);

			/*
			 * paranoia - cope if the function, which may have constructed the
			 * tuplestore itself, didn't leave it pointing at the start. This
			 * call is fast, so the overhead shouldn't be an issue.
			 */
			tuplestore_rescan(tstore);
		}

		/* Get the next tuple from tuplestore. */
		(void) tuplestore_gettupleslot(tstore,
									   ScanDirectionIsForward(direction),
									   false,
									   scanslot);
		return scanslot;
	}

	/*
	 * Increment or decrement ordinal counter before checking for end-of-data,
	 * so that we can move off either end of the result by 1 (and no more than
	 * 1) without losing correct count.  See PortalRunSelect for why we can
	 * assume that we won't be called repeatedly in the end-of-data state.
	 */
	oldpos = node->ordinal;
	if (ScanDirectionIsForward(direction))
		node->ordinal++;
	else
		node->ordinal--;

	/*
	 * Main loop over functions.
	 *
	 * We fetch the function results into func_slots (which match the function
	 * return types), and then copy the values to scanslot (which matches the
	 * scan result type), setting the ordinal column (if any) as well.
	 */
	ExecClearTuple(scanslot);
	att = 0;
	alldone = true;
	for (funcno = 0; funcno < node->nfuncs; funcno++)
	{
		FunctionScanPerFuncState *fs = &node->funcstates[funcno];
		int			i;

		/*
		 * If first time through, read all tuples from function and put them
		 * in a tuplestore. Subsequent calls just fetch tuples from
		 * tuplestore.
		 */
		if (fs->tstore == NULL)
		{
			fs->tstore =
				ExecMakeTableFunctionResult(fs->setexpr,
											node->ss.ps.ps_ExprContext,
											node->argcontext,
											fs->tupdesc,
											node->eflags & EXEC_FLAG_BACKWARD);

			/*
			 * paranoia - cope if the function, which may have constructed the
			 * tuplestore itself, didn't leave it pointing at the start. This
			 * call is fast, so the overhead shouldn't be an issue.
			 */
			tuplestore_rescan(fs->tstore);
		}

		/*
		 * Get the next tuple from tuplestore.
		 *
		 * If we have a rowcount for the function, and we know the previous
		 * read position was out of bounds, don't try the read. This allows
		 * backward scan to work when there are mixed row counts present.
		 */
		if (fs->rowcount != -1 && fs->rowcount < oldpos)
			ExecClearTuple(fs->func_slot);
		else
			(void) tuplestore_gettupleslot(fs->tstore,
										   ScanDirectionIsForward(direction),
										   false,
										   fs->func_slot);

		if (TupIsNull(fs->func_slot))
		{
			/*
			 * If we ran out of data for this function in the forward
			 * direction then we now know how many rows it returned. We need
			 * to know this in order to handle backwards scans. The row count
			 * we store is actually 1+ the actual number, because we have to
			 * position the tuplestore 1 off its end sometimes.
			 */
			if (ScanDirectionIsForward(direction) && fs->rowcount == -1)
				fs->rowcount = node->ordinal;

			/*
			 * populate the result cols with nulls
			 */
			for (i = 0; i < fs->colcount; i++)
			{
				scanslot->tts_values[att] = (Datum) 0;
				scanslot->tts_isnull[att] = true;
				att++;
			}
		}
		else
		{
			/*
			 * we have a result, so just copy it to the result cols.
			 */
			slot_getallattrs(fs->func_slot);

			for (i = 0; i < fs->colcount; i++)
			{
				scanslot->tts_values[att] = fs->func_slot->tts_values[i];
				scanslot->tts_isnull[att] = fs->func_slot->tts_isnull[i];
				att++;
			}

			/*
			 * We're not done until every function result is exhausted; we pad
			 * the shorter results with nulls until then.
			 */
			alldone = false;
		}
	}

	/*
	 * ordinal col is always last, per spec.
	 */
	if (node->ordinality)
	{
		scanslot->tts_values[att] = Int64GetDatumFast(node->ordinal);
		scanslot->tts_isnull[att] = false;
	}

	/*
	 * If alldone, we just return the previously-cleared scanslot.  Otherwise,
	 * finish creating the virtual tuple.
	 */
	if (!alldone)
		ExecStoreVirtualTuple(scanslot);

	return scanslot;
}
```