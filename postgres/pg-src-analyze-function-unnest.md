### Postgres源码分析 —— unnest函数
本文分析一下unnest函数的执行过程，以`select unnest(array[(1,1),(2,2),(3,3)]) as a;`为例。

`unnest`：将输入的数组转换成一个表。

#### 查询优化器
##### 语法解析
解析层面流程如下：
```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
```

gram.y中语法表示如下：
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
	
opt_target_list: target_list						{ $$ = $1; }
			| /* EMPTY */							{ $$ = NIL; }
		;

target_list:
			target_el								{ $$ = list_make1($1); }
			| target_list ',' target_el				{ $$ = lappend($1, $3); }
		;

target_el:	a_expr AS ColLabel
				{
					$$ = makeNode(ResTarget);
					$$->name = $3;
					$$->indirection = NIL;
					$$->val = (Node *)$1;
					$$->location = @1;
				}
func_expr: func_application within_group_clause filter_clause over_clause
				{
					FuncCall *n = (FuncCall *) $1;
					// ......
				}
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
func_arg_list:  func_arg_expr
				{
					$$ = list_make1($1);
				}
func_arg_expr:  a_expr
				{
					$$ = $1;
				}
a_expr:		c_expr				{ $$ = $1; }
c_expr:		columnref			{ $$ = $1; }
			| ARRAY array_expr
				{
					A_ArrayExpr *n = castNode(A_ArrayExpr, $2);
					/* point outermost A_ArrayExpr to the ARRAY keyword */
					n->location = @1;
					$$ = (Node *)n;
				}
array_expr: '[' expr_list ']'
				{
					$$ = makeAArrayExpr($2, @1);
				}
			| '[' array_expr_list ']'
				{
					$$ = makeAArrayExpr($2, @1);
				}
			| '[' ']'
				{
					$$ = makeAArrayExpr(NIL, @1);
				}
		;

```
##### 语义分析
语义分析流程：
```c++
pg_analyze_and_rewrite
--> pg_analyze
    --> transformStmt
        --> transformSelectStmt
            --> transformTargetList   // 函数FuncCall在ResTarget中
			    --> transformTargetEntry
				    --> transformExpr
					    --> transformFuncCall
						    --> ParseFuncOrColumn
					--> makeTargetEntry
--> pg_rewrite_query
```

##### 优化器

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
    --> preprocess_expression
	    --> eval_const_expressions
		    --> simplify_function
			    --> FunctionCall1Coll
			    	--> array_unnest_support
				--> inline_function
    --> grouping_planner
        --> query_planner
            --> setup_simple_rel_arrays
                --> build_simple_rel
				--> create_group_result_path
        --> create_pathtarget
		    --> set_pathtarget_cost_width
			    --> cost_qual_eval_node
				    --> add_function_cost
					    --> FunctionCallInvoke
						    --> array_unnest_support
		--> apply_scanjoin_target_to_paths
		    --> create_projection_path
			--> adjust_paths_for_srfs
			    --> create_set_projection_path
--> create_plan
    --> create_project_set_plan
        --> create_projection_plan
            --> create_group_result_plan
```
关键函数源码：
```c++
/* Planner support function for array_unnest(anyarray) */
Datum array_unnest_support(PG_FUNCTION_ARGS)
{
	Node	   *rawreq = (Node *) PG_GETARG_POINTER(0);
	Node	   *ret = NULL;

	if (IsA(rawreq, SupportRequestRows))
	{
		/* Try to estimate the number of rows returned */
		SupportRequestRows *req = (SupportRequestRows *) rawreq;

		if (is_funcclause(req->node))	/* be paranoid */
		{
			List	   *args = ((FuncExpr *) req->node)->args;
			Node	   *arg1;

			/* We can use estimated argument values here */
			arg1 = estimate_expression_value(req->root, linitial(args));

			req->rows = estimate_array_length(arg1);
			ret = (Node *) req;
		}
	}

	PG_RETURN_POINTER(ret);
}
```

#### 执行器
```c++
PortalStart
--> ExecutorStart
    --> InitPlan
        --> ExecInitRangeTable
		--> ExecInitProjectSet
		    --> ExecInitResult
			    --> ExecInitResultTupleSlotTL
				--> ExecAssignProjectionInfo
			--> ExecInitFunctionResultSet
PortalRun
--> ExecutorRun
    --> ExecutePlan
        --> ExecProjectSet
		    --> ExecResult
			    --> ExecProject
		    --> ExecProjectSRF
			    --> ExecMakeFunctionResultSet
				    --> FunctionCallInvoke
					    --> array_unnest
				--> ExecStoreVirtualTuple

PortalDrop
```


##### InitPlan关键函数源码
```c++
/* ----------------------------------------------------------------
 *		ExecInitProjectSet
 *
 *		Creates the run-time state information for the ProjectSet node
 *		produced by the planner and initializes outer relations
 *		(child nodes).
 * ---------------------------------------------------------------- */
ProjectSetState *
ExecInitProjectSet(ProjectSet *node, EState *estate, int eflags)
{
	ProjectSetState *state;
	ListCell   *lc;
	int			off;

	/*
	 * create state structure
	 */
	state = makeNode(ProjectSetState);
	state->ps.plan = (Plan *) node;
	state->ps.state = estate;
	state->ps.ExecProcNode = ExecProjectSet;

	state->pending_srf_tuples = false;

	/*
	 * Miscellaneous initialization
	 *
	 * create expression context for node */
	ExecAssignExprContext(estate, &state->ps);

	/* initialize child nodes */
	outerPlanState(state) = ExecInitNode(outerPlan(node), estate, eflags);

	/* tuple table and result type initialization */
	ExecInitResultTupleSlotTL(&state->ps, &TTSOpsVirtual);

	/* Create workspace for per-tlist-entry expr state & SRF-is-done state */
	state->nelems = list_length(node->plan.targetlist);
	state->elems = (Node **)palloc(sizeof(Node *) * state->nelems);
	state->elemdone = (ExprDoneCond *)palloc(sizeof(ExprDoneCond) * state->nelems);

	/*
	 * Build expressions to evaluate targetlist.  We can't use
	 * ExecBuildProjectionInfo here, since that doesn't deal with SRFs.
	 * Instead compile each expression separately, using
	 * ExecInitFunctionResultSet where applicable.
	 */
	off = 0;
	foreach(lc, node->plan.targetlist)
	{
		TargetEntry *te = (TargetEntry *) lfirst(lc);
		Expr	   *expr = te->expr;  // 这里是FuncExpr节点

		if ((IsA(expr, FuncExpr) && ((FuncExpr *) expr)->funcretset) || (IsA(expr, OpExpr) && ((OpExpr *) expr)->opretset))
		{
			state->elems[off] = (Node *)
				ExecInitFunctionResultSet(expr, state->ps.ps_ExprContext, &state->ps);
		}
		else
		{
			Assert(!expression_returns_set((Node *) expr));
			state->elems[off] = (Node *) ExecInitExpr(expr, &state->ps);
		}

		off++;
	}

	/*
	 * Create a memory context that ExecMakeFunctionResultSet can use to
	 * evaluate function arguments in.  We can't use the per-tuple context for
	 * this because it gets reset too often; but we don't want to leak
	 * evaluation results into the query-lifespan context either.  We use one
	 * context for the arguments of all tSRFs, as they have roughly equivalent
	 * lifetimes.
	 */
	state->argcontext = AllocSetContextCreate(CurrentMemoryContext,
											  "tSRF function arguments",
											  ALLOCSET_DEFAULT_SIZES);

	return state;
}
```


##### 函数关键源码
```c++
/* UNNEST */
Datum array_unnest(PG_FUNCTION_ARGS)
{
	typedef struct
	{
		array_iter	iter;
		int			nextelem;
		int			numelems;
		int16		elmlen;
		bool		elmbyval;
		char		elmalign;
	} array_unnest_fctx;

	FuncCallContext *funcctx;
	array_unnest_fctx *fctx;
	MemoryContext oldcontext;

	/* stuff done only on the first call of the function */
	if (SRF_IS_FIRSTCALL())
	{
		AnyArrayType *arr;

		/* create a function context for cross-call persistence */
		funcctx = SRF_FIRSTCALL_INIT();

		/* switch to memory context appropriate for multiple function calls */
		oldcontext = MemoryContextSwitchTo(funcctx->multi_call_memory_ctx);

		/* Get the array value and detoast if needed.  We can't do this
		 * earlier because if we have to detoast, we want the detoasted copy
		 * to be in multi_call_memory_ctx, so it will go away when we're done
		 * and not before.  (If no detoast happens, we assume the originally
		 * passed array will stick around till then.) */
		arr = PG_GETARG_ANY_ARRAY_P(0);

		/* allocate memory for user context */
		fctx = (array_unnest_fctx *) palloc(sizeof(array_unnest_fctx));

		/* initialize state */
		array_iter_setup(&fctx->iter, arr);
		fctx->nextelem = 0;
		fctx->numelems = ArrayGetNItems(AARR_NDIM(arr), AARR_DIMS(arr));

		if (VARATT_IS_EXPANDED_HEADER(arr))
		{
			/* we can just grab the type data from expanded array */
			fctx->elmlen = arr->xpn.typlen;
			fctx->elmbyval = arr->xpn.typbyval;
			fctx->elmalign = arr->xpn.typalign;
		}
		else
			get_typlenbyvalalign(AARR_ELEMTYPE(arr), &fctx->elmlen, &fctx->elmbyval, &fctx->elmalign);

		funcctx->user_fctx = fctx;
		MemoryContextSwitchTo(oldcontext);
	}

	/* stuff done on every call of the function */
	funcctx = SRF_PERCALL_SETUP();
	fctx = funcctx->user_fctx;

	if (fctx->nextelem < fctx->numelems)
	{
		int			offset = fctx->nextelem++;
		Datum		elem;

		elem = array_iter_next(&fctx->iter, &fcinfo->isnull, offset, fctx->elmlen, fctx->elmbyval, fctx->elmalign);

		SRF_RETURN_NEXT(funcctx, elem);
	}
	else
	{
		SRF_RETURN_DONE(funcctx); 		/* do when there is no more left */
	}
}

static inline Datum
array_iter_next(array_iter *it, bool *isnull, int i, int elmlen, bool elmbyval, char elmalign)
{
	Datum		ret;
	if (it->datumptr)
	{
		ret = it->datumptr[i];
		*isnull = it->isnullptr ? it->isnullptr[i] : false;
	} else
	{
		if (it->bitmapptr && (*(it->bitmapptr) & it->bitmask) == 0)
		{
			*isnull = true;
			ret = (Datum) 0;
		} else
		{
			*isnull = false;
			ret = fetch_att(it->dataptr, elmbyval, elmlen);
			it->dataptr = att_addlength_pointer(it->dataptr, elmlen, it->dataptr);
			it->dataptr = (char *) att_align_nominal(it->dataptr, elmalign);
		}
		it->bitmask <<= 1;
		if (it->bitmask == 0x100)
		{
			if (it->bitmapptr)
				it->bitmapptr++;
			it->bitmask = 1;
		}
	}

	return ret;
}
```