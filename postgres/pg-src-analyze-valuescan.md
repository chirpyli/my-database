### Postgres源码分析—— ValueScan
本文以`select * from (values (1,1),(2,2)) as foo;`为例，分析一下ValueScan其源码执行过程。

#### 语法解析层
Select SQL语句字符串输入到数据库后，会首先在语法解析层表示成抽象语法树`SelectStmt`，进而经过语义分析，转换为查询树`Query`。 将values的信息保存在了valuesLists字段中。

```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
```
核心结构体
```c++
typedef struct SelectStmt
{
	NodeTag		type;

	/* These fields are used only in "leaf" SelectStmts.*/
	List	   *distinctClause; /* NULL, list of DISTINCT ON exprs, or
								 * lcons(NIL,NIL) for all (SELECT DISTINCT) */
	IntoClause *intoClause;		/* target for SELECT INTO */
	List	   *targetList;		/* the target list (of ResTarget) */
	List	   *fromClause;		/* the FROM clause */
	Node	   *whereClause;	/* WHERE qualification */
	List	   *groupClause;	/* GROUP BY clauses */
	Node	   *havingClause;	/* HAVING conditional-expression */
	List	   *windowClause;	/* WINDOW window_name AS (...), ... */

	/*
	 * In a "leaf" node representing a VALUES list, the above fields are all
	 * null, and instead this field is set.  Note that the elements of the
	 * sublists are just expressions, without ResTarget decoration. Also note
	 * that a list element can be DEFAULT (represented as a SetToDefault
	 * node), regardless of the context of the VALUES list. It's up to parse
	 * analysis to reject that where not valid.
	 */
	List	   *valuesLists;	/* untransformed list of expression lists */

	/* These fields are used in both "leaf" SelectStmts and upper-level SelectStmts.*/
	List	   *sortClause;		/* sort clause (a list of SortBy's) */
	Node	   *limitOffset;	/* # of result tuples to skip */
	Node	   *limitCount;		/* # of result tuples to return */
	LimitOption limitOption;	/* limit type */
	List	   *lockingClause;	/* FOR UPDATE (list of LockingClause's) */
	WithClause *withClause;		/* WITH clause */

	/* These fields are used only in upper-level SelectStmts.*/
	SetOperation op;			/* type of set op */
	bool		all;			/* ALL specified? */
	struct SelectStmt *larg;	/* left child */
	struct SelectStmt *rarg;	/* right child */
} SelectStmt;

/* RangeSubselect - subquery appearing in a FROM clause */
typedef struct RangeSubselect
{
	NodeTag		type;
	bool		lateral;		/* does it have LATERAL prefix? */
	Node	   *subquery;		/* the untransformed sub-select clause */
	Alias	   *alias;			/* table alias & optional column aliases */
} RangeSubselect;
```


gram.y中语法定义分析：
```sql
-- 表示的 from () as foo
from_clause:
			FROM from_list							{ $$ = $2; }
			| /*EMPTY*/								{ $$ = NIL; }
		;

from_list:
			table_ref								{ $$ = list_make1($1); }
			| from_list ',' table_ref				{ $$ = lappend($1, $3); }
		;

/*
 * table_ref is where an alias clause can be attached.
 */
table_ref:	relation_expr opt_alias_clause
				{
					$1->alias = $2;
					$$ = (Node *) $1;
				}
			| select_with_parens opt_alias_clause
				{   -- 将values 表示为了子查询 
					RangeSubselect *n = makeNode(RangeSubselect);
					n->lateral = false;
					n->subquery = $1;
					n->alias = $2;

					if ($2 == NULL)
					{
						if (IsA($1, SelectStmt) &&
							((SelectStmt *) $1)->valuesLists)
							ereport(ERROR,
									(errcode(ERRCODE_SYNTAX_ERROR),
									 errmsg("VALUES in FROM must have an alias"),
									 errhint("For example, FROM (VALUES ...) [AS] foo."),
									 parser_errposition(@1)));
					}
					$$ = (Node *) n;
				}

select_with_parens:
			'(' select_no_parens ')'				{ $$ = $2; }

select_no_parens:
			simple_select						{ $$ = $1; }
        
simple_select:
			SELECT opt_all_clause opt_target_list
			into_clause from_clause where_clause
			group_clause having_clause window_clause
				{
					SelectStmt *n = makeNode(SelectStmt);
					n->targetList = $3;
					n->intoClause = $4;
					n->fromClause = $5;
					n->whereClause = $6;
					n->groupClause = ($7)->list;
					n->havingClause = $8;
					n->windowClause = $9;
					$$ = (Node *)n;
				}
    		| values_clause							{ $$ = $1; }

-- 表示 values ( (1,1), (2,2))
values_clause:
			VALUES '(' expr_list ')'
				{
					SelectStmt *n = makeNode(SelectStmt);
					n->valuesLists = list_make1($3);
					$$ = (Node *) n;
				}
			| values_clause ',' '(' expr_list ')'
				{
					SelectStmt *n = (SelectStmt *) $1;
					n->valuesLists = lappend(n->valuesLists, $4);
					$$ = (Node *) n;
				}
		;
```

最终表示为：
```c++
// select * from Subquery,  Subquery is (Values (1,1),(2,2)) as foo;
SelectStmt  (top-sql)
{
    targetList :  '*'
    fromClause :  RangeSubselect
}
// 子查询范围表
RangeSubselect
{
    subquery:  SelectStmt  (Subquery is (Values (1,1),(2,2)))
    alias:    as foo
}
// 子查询
SelectStmt
{
    valuesLists:   `(Values (1,1),(2,2))`
}
```

#### 语义分析
将上面的抽象语法树`SelectStmt`转换为查询树`Query`。检查语义是否合法。流程如下：
```c++
pg_analyze_and_rewrite
--> pg_analyze
    --> transformStmt
        --> transformSelectStmt
            --> transformFromClause  // 处理子查询范围表
                --> transformFromClauseItem
                    --> transformRangeSubselect
                        --> parse_sub_analyze // 处理子查询，生成子查询的查询树Query
                            --> transformStmt
                                --> transformValuesClause  // 处理子查询中的values表达式
                                    --> transformExpressionList
                                        --> transformExpr
                                    --> addRangeTableEntryForValues
                        --> addRangeTableEntryForSubquery //将values expr list 放到rte->values_lists字段中
            --> transformTargetList
--> pg_rewrite_query
```

关键数据结构：
```c++
// 范围表类型
typedef enum RTEKind
{
	RTE_RELATION,				/* ordinary relation reference */
	RTE_SUBQUERY,				/* subquery in FROM */
	RTE_JOIN,					/* join */
	RTE_FUNCTION,				/* function in FROM */
	RTE_TABLEFUNC,				/* TableFunc(.., column list) */
	RTE_VALUES,					/* VALUES (<exprlist>), (<exprlist>), ... */
	RTE_CTE,					/* common table expr (WITH list element) */
	RTE_NAMEDTUPLESTORE,		/* tuplestore, e.g. for AFTER triggers */
	RTE_RESULT					/* RTE represents an empty FROM clause; such
								 * RTEs are added by the planner, they're not
								 * present during parsing or rewriting */
} RTEKind;
// 范围表
typedef struct RangeTblEntry
{
	NodeTag		type;
	RTEKind		rtekind;		/* see above */
	/*
	 * Fields valid for a plain relation RTE (else zero):
	 */
	Oid			relid;			/* OID of the relation */
	char		relkind;		/* relation kind (see pg_class.relkind) */
	int			rellockmode;	/* lock level that query requires on the rel */
	struct TableSampleClause *tablesample;	/* sampling info, or NULL */

	/*
	 * Fields valid for a subquery RTE (else NULL):
	 */
	Query	   *subquery;		/* the sub-query */
	bool		security_barrier;	/* is from security_barrier view? */

	/*
	 * Fields valid for a join RTE (else NULL/zero):
	 */
	JoinType	jointype;		/* type of join */
	int			joinmergedcols; /* number of merged (JOIN USING) columns */
	List	   *joinaliasvars;	/* list of alias-var expansions */
	List	   *joinleftcols;	/* left-side input column numbers */
	List	   *joinrightcols;	/* right-side input column numbers */

	/*
	 * Fields valid for a function RTE (else NIL/zero):
	 */
	List	   *functions;		/* list of RangeTblFunction nodes */
	bool		funcordinality; /* is this called WITH ORDINALITY? */

	/*
	 * Fields valid for a TableFunc RTE (else NULL):
	 */
	TableFunc  *tablefunc;

	/*
	 * Fields valid for a values RTE (else NIL):
	 */
	List	   *values_lists;	/* list of expression lists */  // Values表达式列表

	/*
	 * Fields valid for a CTE RTE (else NULL/zero):
	 */
	char	   *ctename;		/* name of the WITH list item */
	Index		ctelevelsup;	/* number of query levels up */
	bool		self_reference; /* is this a recursive self-reference? */

	/*
	 * Fields valid for CTE, VALUES, ENR, and TableFunc RTEs (else NIL):
	 */
	List	   *coltypes;		/* OID list of column type OIDs */
	List	   *coltypmods;		/* integer list of column typmods */
	List	   *colcollations;	/* OID list of column collation OIDs */

	/*
	 * Fields valid for ENR RTEs (else NULL/zero):
	 */
	char	   *enrname;		/* name of ephemeral named relation */
	double		enrtuples;		/* estimated or actual from caller */

	/*
	 * Fields valid in all RTEs:
	 */
	Alias	   *alias;			/* user-written alias clause, if any */
	Alias	   *eref;			/* expanded reference names */
	bool		lateral;		/* subquery, function, or values is LATERAL? */
	bool		inh;			/* inheritance requested? */
	bool		inFromCl;		/* present in FROM clause? */
	AclMode		requiredPerms;	/* bitmask of required access permissions */
	Oid			checkAsUser;	/* if valid, check access as this role */
	Bitmapset  *selectedCols;	/* columns needing SELECT permission */
	Bitmapset  *insertedCols;	/* columns needing INSERT permission */
	Bitmapset  *updatedCols;	/* columns needing UPDATE permission */
	Bitmapset  *extraUpdatedCols;	/* generated columns being updated */
	List	   *securityQuals;	/* security barrier quals to apply, if any */
	const char *qb_name;
} RangeTblEntry;
```

关键处理逻辑：
// 子查询
SelectStmt
{
    valuesLists:   `(Values (1,1),(2,2))`
}

会将valuesLists转为`RangeTblEntry`，相关信息保存到values_list字段中。子查询`SelectStmt`转为子查询树`Query`。

// 子查询范围表
RangeSubselect
{
    subquery:  SelectStmt  (Subquery is (Values (1,1),(2,2)))
    alias:    as foo
}
`RangeSubselect`转为父查询的`RangeTblEntry`，

// select * from Subquery,  Subquery is (Values (1,1),(2,2)) as foo;
SelectStmt  (top-sql)
{
    targetList :  '*'
    fromClause :  RangeSubselect
}
`SelectStmt`转为top查询树`Query`

#### 优化器
通过查询树生成最优路径，再生成最优执行计划。

优化器主流程如下：
```c++
pg_plan_queries
--> pg_plan_query
    --> planner
        --> standard_planner
            --> subquery_planner
                --> pull_up_subqueries  // 上拉子查询
                --> grouping_planner
                    --> query_planner
                        --> setup_simple_rel_arrays
                        --> add_base_rels_to_query
                            --> build_simple_rel
                        --> make_one_rel
                            --> set_base_rel_sizes
                                --> set_rel_size
                                    --> set_values_size_estimates
                                        --> set_baserel_size_estimates
                            --> set_base_rel_pathlists
                                --> set_rel_pathlist
                                    --> set_values_pathlist
                                        --> create_valuesscan_path  // 生成ValueScanPath
                            --> make_rel_from_joinlist
                    --> apply_scanjoin_target_to_paths
            --> create_plan
                --> create_scan_plan
                    --> create_valuesscan_plan
```

重点函数分析：最重要的是看一下VauleScan执行计划的生成
```c++
/*
 * create_valuesscan_plan
 *	 Returns a valuesscan plan for the base relation scanned by 'best_path'
 *	 with restriction clauses 'scan_clauses' and targetlist 'tlist'.
 */
ValuesScan *
create_valuesscan_plan(PlannerInfo *root, Path *best_path, List *tlist, List *scan_clauses)
{
	ValuesScan *scan_plan;
	Index		scan_relid = best_path->parent->relid;
	RangeTblEntry *rte;
	List	   *values_lists;

	rte = planner_rt_fetch(scan_relid, root);
	values_lists = rte->values_lists;       //存放在RangeTblEntry中的values_lists

	/* Sort clauses into best execution order */
	scan_clauses = order_qual_clauses(root, scan_clauses);

	/* Reduce RestrictInfo list to bare expressions; ignore pseudoconstants */
	scan_clauses = extract_actual_clauses(scan_clauses, false);

	/* Replace any outer-relation variables with nestloop params */
	if (best_path->param_info)
	{
		scan_clauses = (List *)replace_nestloop_params(root, (Node *) scan_clauses);
		/* The values lists could contain nestloop params, too */
		values_lists = (List *)replace_nestloop_params(root, (Node *) values_lists);
	}

	scan_plan = make_valuesscan(tlist, scan_clauses, scan_relid, values_lists);
	copy_generic_path_info(&scan_plan->scan.plan, best_path);

	return scan_plan;
}

ValuesScan *
make_valuesscan(List *qptlist,List *qpqual,Index scanrelid,List *values_lists)
{
	ValuesScan *node = makeNode(ValuesScan);
	Plan	   *plan = &node->scan.plan;

	plan->targetlist = qptlist;
	plan->qual = qpqual;
	plan->lefttree = NULL;
	plan->righttree = NULL;
	node->scan.scanrelid = scanrelid;
	node->values_lists = values_lists;  // values expr list

	return node;
}
```

最终的执行计划：
```sql
postgres@postgres=# explain select * from (values (1,1), (2,2)) as foo;
                         QUERY PLAN                          
-------------------------------------------------------------
 Values Scan on "*VALUES*"  (cost=0.00..0.03 rows=2 width=8)
(1 row)
```

#### 执行器
输入执行计划，输出最终结果。

```c++
PortalStart
--> ExecutorStart
    --> InitPlan
        --> ExecInitValuesScan
PortalRun
--> ExecutorRun
    --> ExecutePlan
        --> ExecValuesScan   // 从这里开始，ExecValuesScan、ExecScan、ExecScanFetch、ValuesNext都要读懂
            --> ExecScan
                --> ExecScanFetch
                    --> ValuesNext   // 从 values expr list中获取值 转为tuple
PortalDrop
```

执行器算子实现源码如下：因为这几个算子都是比较重要且高频的算子，所以我们把他的源码全部列了出来，要把整个都读懂才行。
```c++
/* ----------------------------------------------------------------
 *		ExecValuesScan(node)
 *
 *		Scans the values lists sequentially and returns the next qualifying tuple.
 *		We call the ExecScan() routine and pass it the appropriate access method functions.
 * ----------------------------------------------------------------*/
TupleTableSlot *ExecValuesScan(PlanState *pstate)
{
	ValuesScanState *node = castNode(ValuesScanState, pstate);

	return ExecScan(&node->ss,
					(ExecScanAccessMtd) ValuesNext,   // 这个参数最重要，access method实现
					(ExecScanRecheckMtd) ValuesRecheck);
}

/* ----------------------------------------------------------------
 *		ExecScan
 *
 *		Scans the relation using the 'access method' indicated and
 *		returns the next qualifying tuple.
 *		The access method returns the next tuple and ExecScan() is
 *		responsible for checking the tuple returned against the qual-clause.
 *
 *		A 'recheck method' must also be provided that can check an
 *		arbitrary tuple of the relation against any qual conditions
 *		that are implemented internal to the access method.
 *
 *		Conditions:
 *		  -- the "cursor" maintained by the AMI is positioned at the tuple
 *			 returned previously.
 *
 *		Initial States:
 *		  -- the relation indicated is opened for scanning so that the
 *			 "cursor" is positioned before the first qualifying tuple.
 * ----------------------------------------------------------------
 */
TupleTableSlot *
ExecScan(ScanState *node, ExecScanAccessMtd accessMtd, ExecScanRecheckMtd recheckMtd)
{
	ExprContext *econtext;
	ExprState  *qual;
	ProjectionInfo *projInfo;

	/* Fetch data from node */
	qual = node->ps.qual;
	projInfo = node->ps.ps_ProjInfo;
	econtext = node->ps.ps_ExprContext;

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
	 * passes the qualification. */
	for (;;)
	{
		TupleTableSlot *slot;
		slot = ExecScanFetch(node, accessMtd, recheckMtd);

		/* if the slot returned by the accessMtd contains NULL, then it means
		 * there is nothing more to scan so we just return an empty slot,
		 * being careful to use the projection result slot so it has correct tupleDesc. */
		if (TupIsNull(slot))
		{
			if (projInfo)
				return ExecClearTuple(projInfo->pi_state.resultslot);
			else
				return slot;
		}

		/* place the current tuple into the expr context */
		econtext->ecxt_scantuple = slot;

		/* check that the current tuple satisfies the qual-clause
		 *
		 * check for non-null qual here to avoid a function call to ExecQual()
		 * when the qual is null ... saves only a few cycles, but they add up ... */
		if (qual == NULL || ExecQual(qual, econtext))
		{
			/* Found a satisfactory scan tuple. */
			if (projInfo)
			{
				/* Form a projection tuple, store it in the result tuple slot and return it. */
				return ExecProject(projInfo);
			} else
			{
				return slot;  /* Here, we aren't projecting, so just return scan tuple. */
			}
		}
		else
			InstrCountFiltered1(node, 1);

		/* Tuple fails qual, so free per-tuple memory and try again. */
		ResetExprContext(econtext);
	}
}


/* ----------------------------------------------------------------
 *		ValuesNext
 *
 *		This is a workhorse for ExecValuesScan
 * ----------------------------------------------------------------*/
TupleTableSlot *ValuesNext(ValuesScanState *node)
{
	TupleTableSlot *slot;
	EState	   *estate;
	ExprContext *econtext;
	ScanDirection direction;
	int			curr_idx;

	/* get information from the estate and scan state */
	estate = node->ss.ps.state;
	direction = estate->es_direction;
	slot = node->ss.ss_ScanTupleSlot;
	econtext = node->rowcontext;

	/* Get the next tuple. Return NULL if no more tuples. */
	if (ScanDirectionIsForward(direction))
	{
		if (node->curr_idx < node->array_len)
			node->curr_idx++;
	} else
	{
		if (node->curr_idx >= 0)
			node->curr_idx--;
	}

	/* Always clear the result slot; this is appropriate if we are at the end
	 * of the data, and if we're not, we still need it as the first step of
	 * the store-virtual-tuple protocol.  It seems wise to clear the slot
	 * before we reset the context it might have pointers into. */
	ExecClearTuple(slot);

	curr_idx = node->curr_idx;
	if (curr_idx >= 0 && curr_idx < node->array_len)
	{
		List	   *exprlist = node->exprlists[curr_idx];
		List	   *exprstatelist = node->exprstatelists[curr_idx];
		MemoryContext oldContext;
		Datum	   *values;
		bool	   *isnull;
		ListCell   *lc;
		int			resind;

		/* Get rid of any prior cycle's leftovers.  We use ReScanExprContext
		 * not just ResetExprContext because we want any registered shutdown callbacks to be called. */
		ReScanExprContext(econtext);

		/* Do per-VALUES-row work in the per-tuple context.*/
		oldContext = MemoryContextSwitchTo(econtext->ecxt_per_tuple_memory);

		/*
		 * Unless we already made the expression eval state for this row,
		 * build it in the econtext's per-tuple memory.  This is a tad
		 * unusual, but we want to delete the eval state again when we move to
		 * the next row, to avoid growth of memory requirements over a long
		 * values list.  For rows in which that won't work, we already built
		 * the eval state at plan startup.
		 */
		if (exprstatelist == NIL)
		{
			/*
			 * Pass parent as NULL, not my plan node, because we don't want
			 * anything in this transient state linking into permanent state.
			 * The only expression type that might wish to do so is a SubPlan,
			 * and we already checked that there aren't any.
			 *
			 * Note that passing parent = NULL also disables JIT compilation
			 * of the expressions, which is a win, because they're only going
			 * to be used once under normal circumstances.
			 */
			exprstatelist = ExecInitExprList(exprlist, NULL);
		}

		/*
		 * Compute the expressions and build a virtual result tuple. We
		 * already did ExecClearTuple(slot).
		 */
		values = slot->tts_values;
		isnull = slot->tts_isnull;

		resind = 0;
		foreach(lc, exprstatelist)
		{
			ExprState  *estate = (ExprState *) lfirst(lc);
			Form_pg_attribute attr = TupleDescAttr(slot->tts_tupleDescriptor, resind);

			values[resind] = ExecEvalExpr(estate, econtext, &isnull[resind]);

			/*
			 * We must force any R/W expanded datums to read-only state, in
			 * case they are multiply referenced in the plan node's output
			 * expressions, or in case we skip the output projection and the
			 * output column is multiply referenced in higher plan nodes.
			 */
			values[resind] = MakeExpandedObjectReadOnly(values[resind],isnull[resind],attr->attlen);

			resind++;
		}

		MemoryContextSwitchTo(oldContext);
		ExecStoreVirtualTuple(slot);  		/* And return the virtual tuple. */
	}

	return slot;
}
```