### Postgres源码分析之CTE
Postgres支持使用WITH子句的子查询，那么什么是WITH子句呢？如果在一个SQL语句中多次使用同一个子查询，可以通过WITH子句给子查询指定一个名字，从而可以实现通过名字引用该子查询，而不必每次都完整写出该子查询。这一特性常称为CTE（Common Table Expressions）公共表表达式，常用于复杂查询或递归查询应用场景。

我们举个实际的例子进行说明：
```sql
with t as (
	select generate_series(1,3)
	)
select * from t;
/*
运行结果：
 generate_series 
-----------------
               1
               2
               3
(3 rows)
*/

--可以理解为t为子查询的别名，等同于下面的子查询。有别名的好处是在复杂语句中可以方便的多次使用
postgres@postgres=# select * from (select generate_series(1,3));
 generate_series 
-----------------
               1
               2
               3
(3 rows)

-- 通过别名方便多次使用
postgres@postgres=# with t as (select generate_series(1,3)) select * from t union all select * from t;
 generate_series 
-----------------
               1
               2
               3
               1
               2
               3
(6 rows)

-- with递归查询例子
with recursive t(x) as (
	select 1 union select x + 1 from t where x < 5
	)
	select sum(x) from t;
/*
运行结果
 sum 
-----
  15
(1 row)
*/
```

### 源码分析
我们以下面这条SQL为例，看一下Postgres是如何实现CTE的。当然下面这条SQL不能代表CTE的全部，所以后续还会继续分析其实现。
```sql
create table t1(a int , b int);
insert into t1 values(1,1),(2,2),(3,3);
-- 分析这条SQL的实现
with t as (select * from t1) select * from t;
```

主流程如下：
```c++
exec_simple_query
--> pg_parse_query    // 生成语法解析树
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformSelectStmt  // 语义分析,转换为查询树
                --> transformWithClause  // 处理with 语句
    --> pg_rewrite_query
--> pg_plan_queries
```

#### Parser语法解析部分
```c++
exec_simple_query
--> pg_parse_query    // 生成语法解析树
    --> raw_parser
        --> base_yyparse
```

语法解析部分，主要分析一下gram.y中的CTE表示。
```sql
/*
 * SQL standard WITH clause looks like:
 *
 * WITH [ RECURSIVE ] <query name> [ (<column>,...) ]
 *		AS (query) [ SEARCH or CYCLE clause ]
 *
 * We don't currently support the SEARCH or CYCLE clause.
 *
 * Recognizing WITH_LA here allows a CTE to be named TIME or ORDINALITY.
 */
with_clause:
		WITH cte_list
			{
				$$ = makeNode(WithClause);
				$$->ctes = $2;
				$$->recursive = false;
				$$->location = @1;
			}
		| WITH_LA cte_list
			{
				$$ = makeNode(WithClause);
				$$->ctes = $2;
				$$->recursive = false;
				$$->location = @1;
			}
		| WITH RECURSIVE cte_list
			{
				$$ = makeNode(WithClause);
				$$->ctes = $3;
				$$->recursive = true;
				$$->location = @1;
			}
		;

cte_list:
		common_table_expr						{ $$ = list_make1($1); }
		| cte_list ',' common_table_expr		{ $$ = lappend($1, $3); }
		;

common_table_expr:  name opt_name_list AS opt_materialized '(' PreparableStmt ')'
			{
				CommonTableExpr *n = makeNode(CommonTableExpr);
				n->ctename = $1;
				n->aliascolnames = $2;
				n->ctematerialized = $4;
				n->ctequery = $6;
				n->location = @1;
				$$ = (Node *) n;
			}
		;
```
具体的对于例句来说，`with_clause`表示的是`with t as (select * from t1)`这一部分。后半部分是通过`select_clause`表示的`select * from t`，
```sql
select_no_parens:
			simple_select						{ $$ = $1; }
			| with_clause select_clause    -- 匹配
				{
                    -- 将WithClause插入到SelectStmt对应的withClause字段中
					insertSelectOptions((SelectStmt *) $2, NULL, NIL, NULL, $1, yyscanner);
					$$ = $2;  -- 返回SelectStmt
				}
		;
```

```c++
/*
 * WithClause -
 *	   representation of WITH clause
 */
typedef struct WithClause
{
	NodeTag		type;
	List	   *ctes;			/* list of CommonTableExprs */
	bool		recursive;		/* true = WITH RECURSIVE */
	int			location;		/* token location, or -1 if unknown */
} WithClause;

typedef struct CommonTableExpr
{
	NodeTag		type;
	char	   *ctename;		/* query name (never qualified) */
	List	   *aliascolnames;	/* optional list of column names */
	CTEMaterialize ctematerialized; /* is this an optimization fence? */
	/* SelectStmt/InsertStmt/etc before parse analysis, Query afterwards: */
	Node	   *ctequery;		/* the CTE's subquery */
	int			location;		/* token location, or -1 if unknown */
	/* These fields are set during parse analysis: */
	bool		cterecursive;	/* is this CTE actually recursive? */
	int			cterefcount;	/* number of RTEs referencing this CTE
								 * (excluding internal self-references) */
	List	   *ctecolnames;	/* list of output column names */
	List	   *ctecoltypes;	/* OID list of output column type OIDs */
	List	   *ctecoltypmods;	/* integer list of output column typmods */
	List	   *ctecolcollations;	/* OID list of column collation OIDs */
} CommonTableExpr;

// withClause字段保存with_clause信息
typedef struct SelectStmt
{
	NodeTag		type;

    // ......
	WithClause *withClause;		/* WITH clause */  
    // ......

} SelectStmt;
```

#### 语义分析部分

```c++
exec_simple_query
--> pg_parse_query    // 生成语法解析树
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformSelectStmt  // 语义分析,转换为查询树
                --> transformWithClause  // 处理with 语句
                    --> analyzeCTE
                        --> parse_sub_analyze  // 处理ctequery (select * from t1)
                            --> transformStmt
                        --> analyzeCTETargetList
    --> pg_rewrite_query
--> pg_plan_queries
```

#### 优化器部分
```c++
exec_simple_query
--> pg_parse_query    // 生成语法解析树
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformSelectStmt  // 语义分析,转换为查询树
                --> transformWithClause  // 处理with 语句
                    --> analyzeCTE
                        --> parse_sub_analyze  // 处理ctequery (select * from t1)
                            --> transformStmt
                        --> analyzeCTETargetList
    --> pg_rewrite_query
--> pg_plan_queries
    --> pg_plan_query
        --> planner
            --> standard_planner   // 由查询树生成执行计划
                --> subquery_planner
                    /*
                    * If there is a WITH list, process each WITH query and either convert it
                    * to RTE_SUBQUERY RTE(s) or build an initplan SubPlan structure for it.
                    */
                    --> SS_process_ctes
                        --1> inline_cte
                        --2> subquery_planner
                        --2> create_plan
                        --2> cost_subplan
                    --> grouping_planner
                        --> query_planner
                    --> SS_charge_for_initplans
                --> create_plan
                    --> create_plan_recurse
                --> set_plan_references
--> PortalStart
    --> ExecutorStart
        --> InitPlan
--> PortalRun
    --> PortalRunSelect
        --> ExecutorRun
            --> ExecutePlan
--> PortalDrop
```
我们重点看一下SS_process_ctes是如何处理的：
```c++
/*--------------------
 * subquery_planner
 *	  Invokes the planner on a subquery.  We recurse to here for each
 *	  sub-SELECT found in the query tree.
 */
PlannerInfo *
subquery_planner(PlannerGlobal *glob, Query *parse, PlannerInfo *parent_root, bool hasRecursion, double tuple_fraction)
{
	PlannerInfo *root;

    // ...... 
	/*
	 * If there is a WITH list, process each WITH query and either convert it
	 * to RTE_SUBQUERY RTE(s) or build an initplan SubPlan structure for it.
	 */
	if (parse->cteList)
		SS_process_ctes(root);   // 处理WITH clause

    // ...... 
	preprocess_function_rtes(root);

    // ......
    grouping_planner(root, false, tuple_fraction);

    // ......
}

/*
 * SS_process_ctes: process a query's WITH list
 *
 * Consider each CTE in the WITH list and either ignore it (if it's an
 * unreferenced SELECT), "inline" it to create a regular sub-SELECT-in-FROM,
 * or convert it to an initplan.
 *
 * A side effect is to fill in root->cte_plan_ids with a list that
 * parallels root->parse->cteList and provides the subplan ID for
 * each CTE's initplan, or a dummy ID (-1) if we didn't make an initplan.
 */
void
SS_process_ctes(PlannerInfo *root)
{
    ListCell   *lc;
	foreach(lc, root->parse->cteList)
	{
		CommonTableExpr *cte = (CommonTableExpr *) lfirst(lc);
		CmdType		cmdType = ((Query *) cte->ctequery)->commandType;
		Query	   *subquery;
		PlannerInfo *subroot;
		RelOptInfo *final_rel;
		Path	   *best_path;
		Plan	   *plan;
		SubPlan    *splan;
		int			paramid;

		/* Ignore SELECT CTEs that are not actually referenced anywhere. */
		if (cte->cterefcount == 0 && cmdType == CMD_SELECT)
		{
			/* Make a dummy entry in cte_plan_ids */
			root->cte_plan_ids = lappend_int(root->cte_plan_ids, -1);
			continue;
		}

		/* Consider inlining the CTE (creating RTE_SUBQUERY RTE(s)) instead of
		 * implementing it as a separately-planned CTE.
		 *
		 * We cannot inline if any of these conditions hold:
		 *
		 * 1. The user said not to (the CTEMaterializeAlways option).
		 *
		 * 2. The CTE is recursive.
		 *
		 * 3. The CTE has side-effects; this includes either not being a plain
		 * SELECT, or containing volatile functions.  Inlining might change
		 * the side-effects, which would be bad.
		 *
		 * 4. The CTE is multiply-referenced and contains a self-reference to
		 * a recursive CTE outside itself.  Inlining would result in multiple
		 * recursive self-references, which we don't support.
		 *
		 * Otherwise, we have an option whether to inline or not.  That should
		 * always be a win if there's just a single reference, but if the CTE
		 * is multiply-referenced then it's unclear: inlining adds duplicate
		 * computations, but the ability to absorb restrictions from the outer
		 * query level could outweigh that.  We do not have nearly enough
		 * information at this point to tell whether that's true, so we let
		 * the user express a preference.  Our default behavior is to inline
		 * only singly-referenced CTEs, but a CTE marked CTEMaterializeNever
		 * will be inlined even if multiply referenced.
		 *
		 * Note: we check for volatile functions last, because that's more
		 * expensive than the other tests needed. */
		if ((cte->ctematerialized == CTEMaterializeNever ||
			 (cte->ctematerialized == CTEMaterializeDefault && cte->cterefcount == 1)) &&
			!cte->cterecursive && cmdType == CMD_SELECT && !contain_dml(cte->ctequery) &&
			(cte->cterefcount <= 1 || !contain_outer_selfref(cte->ctequery)) &&
			!contain_volatile_functions(cte->ctequery))
		{
			inline_cte(root, cte);
			/* Make a dummy entry in cte_plan_ids */
			root->cte_plan_ids = lappend_int(root->cte_plan_ids, -1);
			continue;
		}

		/* Copy the source Query node.  Probably not necessary, but let's keep
		 * this similar to make_subplan.*/
		subquery = (Query *) copyObject(cte->ctequery);

		/* Generate Paths for the CTE query.  Always plan for full retrieval
		 * --- we don't have enough info to predict otherwise.*/
		subroot = subquery_planner(root->glob, subquery, root, cte->cterecursive, 0.0);

		/* Since the current query level doesn't yet contain any RTEs, it
		 * should not be possible for the CTE to have requested parameters of
		 * this level. */
		if (root->plan_params)
			elog(ERROR, "unexpected outer reference in CTE query");

		/* Select best Path and turn it into a Plan.  At least for now, there
		 * seems no reason to postpone doing that. */
		final_rel = fetch_upper_rel(subroot, UPPERREL_FINAL, NULL);
		best_path = final_rel->cheapest_total_path;

		plan = create_plan(subroot, best_path);

		/* Make a SubPlan node for it.  This is just enough unlike
		 * build_subplan that we can't share code.
		 *
		 * Note plan_id, plan_name, and cost fields are set further down.*/
		splan = makeNode(SubPlan);
		splan->subLinkType = CTE_SUBLINK;
		splan->testexpr = NULL;
		splan->paramIds = NIL;
		get_first_col_type(plan, &splan->firstColType, &splan->firstColTypmod,
						   &splan->firstColCollation);
		splan->useHashTable = false;
		splan->unknownEqFalse = false;

		/* CTE scans are not considered for parallelism (cf
		 * set_rel_consider_parallel), and even if they were, initPlans aren't
		 * parallel-safe. */
		splan->parallel_safe = false;
		splan->setParam = NIL;
		splan->parParam = NIL;
		splan->args = NIL;

		/* The node can't have any inputs (since it's an initplan), so the
		 * parParam and args lists remain empty.  (It could contain references
		 * to earlier CTEs' output param IDs, but CTE outputs are not
		 * propagated via the args list.) */

		/* Assign a param ID to represent the CTE's output.  No ordinary
		 * "evaluation" of this param slot ever happens, but we use the param
		 * ID for setParam/chgParam signaling just as if the CTE plan were
		 * returning a simple scalar output.  (Also, the executor abuses the
		 * ParamExecData slot for this param ID for communication among
		 * multiple CteScan nodes that might be scanning this CTE.) */
		paramid = assign_special_exec_param(root);
		splan->setParam = list_make1_int(paramid);

		/* Add the subplan and its PlannerInfo to the global lists.*/
		root->glob->subplans = lappend(root->glob->subplans, plan);
		root->glob->subroots = lappend(root->glob->subroots, subroot);
		splan->plan_id = list_length(root->glob->subplans);

		root->init_plans = lappend(root->init_plans, splan);
		root->cte_plan_ids = lappend_int(root->cte_plan_ids, splan->plan_id);

		/* Label the subplan for EXPLAIN purposes */
		splan->plan_name = psprintf("CTE %s", cte->ctename);

		/* Lastly, fill in the cost estimates for use later */
		cost_subplan(root, splan, plan);
	}
}
```

到这里，我们只是梳理了一下Postgres中CTE的处理流程，后续会继续深入分析。



---
> 参考文档：
- [MySQL · 特性分析 · common table expression](https://www.bookstack.cn/read/aliyun-rds-core/6c9219c65f63bd9c.md)
- [MySQL · 新特性分析 · CTE执行过程与实现原理](https://www.bookstack.cn/read/aliyun-rds-core/9513cff30a6d9640.md)
- [SQL之with子句](https://blog.csdn.net/qq_42374697/article/details/115293553)