 ### UPDATE源码分析
本文主要描述SQL中UPDATE语句的源码分析，代码为PG13.3版本。

### 整体流程分析
以`update dtea set id = 1;`这条最简单的Update语句进行源码分析(dtea不是分区表，不考虑并行等，没有建立任何索引)，帮助我们理解update的大致流程。

SQL流程如下：
- parser(语法解析，生成语法解析树UpdateStmt，检查是否有语法层面的错误)
- analyze(语义分析， UpdateStmt转为查询树Query， 会查系统表检查有无语义方面的错误)
- rewrite(规则重写, 根据规则rules重写查询树Query， 根据事先存储在系统表中的规则进行重写，没有的话不进行重写，另外加一句，视图的实现是根据规则系统实现的，也是在这里需要进行处理) 
- optimizer(优化器：逻辑优化、物理优化、生成执行计划， 由Query生成对应的执行计划PlannedStmt， 基于代价的优化器，由最佳路径Path生成最佳执行计划Plan)
- executor(执行器，会有各种算子，依据执行计划进行处理，火山模型，一次一元组)
- storage(存储引擎)。中间还有事务处理。事务处理部分的代码这里不再进行分析，免得将问题复杂化。存储引擎那部分也不进行分析，重点关注解析、优化、执行这三部分。

对应的代码：
```c++
exec_simple_query(const char *query_string)
// ------- 解析器部分--------------
--> pg_parse_query(query_string);    //生成语法解析树
--> pg_analyze_and_rewrite(parsetree, query_string,NULL, 0, NULL);   // 生成查询树Query
    --> parse_analyze(parsetree, query_string, paramTypes, numParams,queryEnv); // 语义分析
    --> pg_rewrite_query(query);    // 规则重写

// --------优化器----------
--> pg_plan_queries()

//-------- 执行器----------
--> PortalStart(portal, NULL, 0, InvalidSnapshot);
--> PortalRun(portal,FETCH_ALL,true,true,receiver,receiver,&qc);    // 执行器执行
--> PortalDrop(portal, false);
```


#### 解析部分——生成语法解析树UpdateStmt
关键数据结构：`UpdateStmt`、`RangeVar`、`ResTarget`:
```c++
/* Update Statement  */
typedef struct UpdateStmt
{
	NodeTag		type;
	RangeVar   *relation;		/* relation to update */
	List	   *targetList;		/* the target list (of ResTarget) */	// 对应语句中的set id = 0;信息在这里
	Node	   *whereClause;	/* qualifications */
	List	   *fromClause;		/* optional from clause for more tables */
	List	   *returningList;	/* list of expressions to return */
	WithClause *withClause;		/* WITH clause */
} UpdateStmt;

// dtea 表
typedef struct RangeVar
{
	NodeTag		type;
	char	   *catalogname;	/* the catalog (database) name, or NULL */
	char	   *schemaname;		/* the schema name, or NULL */
	char	   *relname;		/* the relation/sequence name */
	bool		inh;			/* expand rel by inheritance? recursively act
								 * on children? */
	char		relpersistence; /* see RELPERSISTENCE_* in pg_class.h */
	Alias	   *alias;			/* table alias & optional column aliases */
	int			location;		/* token location, or -1 if unknown */
} RangeVar;

// set id = 0;   经transformTargetList() -> transformTargetEntry，会转为TargetEntry
typedef struct ResTarget
{
	NodeTag		type;
	char	   *name;			/* column name or NULL */     // id column
	List	   *indirection;	/* subscripts, field names, and '*', or NIL */
	Node	   *val;			/* the value expression to compute or assign */  // = 1表达式节点存在这里
	int			location;		/* token location, or -1 if unknown */
} ResTarget;
```
用户输入的update语句`update dtea set id = 1`由字符串会转为可由数据库理解的内部数据结构语法解析树`UpdateStmt`。执行逻辑在`pg_parse_query(query_string);`中，需要理解flex与bison。

gram.y中Update语法的定义：
```c++
/*****************************************************************************
 *		QUERY:
 *				UpdateStmt (UPDATE)
 *****************************************************************************/
//结合这条语句分析 update dtea set id = 0;
UpdateStmt: opt_with_clause UPDATE relation_expr_opt_alias
			SET set_clause_list from_clause where_or_current_clause returning_clause
				{
					UpdateStmt *n = makeNode(UpdateStmt);
					n->relation = $3;
					n->targetList = $5;
					n->fromClause = $6;
					n->whereClause = $7;
					n->returningList = $8;
					n->withClause = $1;
					$$ = (Node *)n;
				}
		;

set_clause_list:
			set_clause							{ $$ = $1; }
			| set_clause_list ',' set_clause	{ $$ = list_concat($1,$3); }
		;
// 对应的是 set id = 0
set_clause:	  // id     =   0
			set_target '=' a_expr
				{
					$1->val = (Node *) $3;
					$$ = list_make1($1);
				}
			| '(' set_target_list ')' '=' a_expr
				{
					int ncolumns = list_length($2);
					int i = 1;
					ListCell *col_cell;

					foreach(col_cell, $2)	/* Create a MultiAssignRef source for each target */
					{
						ResTarget *res_col = (ResTarget *) lfirst(col_cell);
						MultiAssignRef *r = makeNode(MultiAssignRef);

						r->source = (Node *) $5;
						r->colno = i;
						r->ncolumns = ncolumns;
						res_col->val = (Node *) r;
						i++;
					}

					$$ = $2;
				}
		;

set_target:
			ColId opt_indirection
				{
					$$ = makeNode(ResTarget);
					$$->name = $1;
					$$->indirection = check_indirection($2, yyscanner);
					$$->val = NULL;	/* upper production sets this */
					$$->location = @1;
				}
		;

set_target_list:
			set_target								{ $$ = list_make1($1); }
			| set_target_list ',' set_target		{ $$ = lappend($1,$3); }
		;
```

#### 解析部分——生成查询树Query
生成了`UpdateStmt`后， 会经由`parse_analyze`语义分析，生成查询树`Query`，以供后续优化器生成执行计划。主要代码在`src/backent/parser/analyze.c`中

> analyze.c : transform the raw parse tree into a query tree

```c++
parse_analyze()
--> transformTopLevelStmt(pstate, parseTree);
    --> transformOptionalSelectInto(pstate, parseTree->stmt);
        --> transformStmt(pstate, parseTree);
            // transforms an update statement
            --> transformUpdateStmt(pstate, (UpdateStmt *) parseTree);  // 实际由UpdateStmt转为Query的处理函数
```

具体的我们看一下`transformUpdateStmt`函数实现：
```c++
/* transformUpdateStmt -  transforms an update statement  */
static Query *transformUpdateStmt(ParseState *pstate, UpdateStmt *stmt) {
	Query	   *qry = makeNode(Query);
	ParseNamespaceItem *nsitem;
	Node	   *qual;

	qry->commandType = CMD_UPDATE;
	pstate->p_is_insert = false;

	/* process the WITH clause independently of all else */
	if (stmt->withClause) {
		qry->hasRecursive = stmt->withClause->recursive;
		qry->cteList = transformWithClause(pstate, stmt->withClause);
		qry->hasModifyingCTE = pstate->p_hasModifyingCTE;
	}

	qry->resultRelation = setTargetTable(pstate, stmt->relation, stmt->relation->inh, true, ACL_UPDATE);
	nsitem = pstate->p_target_nsitem;

	/* subqueries in FROM cannot access the result relation */
	nsitem->p_lateral_only = true;
	nsitem->p_lateral_ok = false;

	/* the FROM clause is non-standard SQL syntax. We used to be able to do this with REPLACE in POSTQUEL so we keep the feature.*/
	transformFromClause(pstate, stmt->fromClause);

	/* remaining clauses can reference the result relation normally */
	nsitem->p_lateral_only = false;
	nsitem->p_lateral_ok = true;

	qual = transformWhereClause(pstate, stmt->whereClause,EXPR_KIND_WHERE, "WHERE");
	qry->returningList = transformReturningList(pstate, stmt->returningList);

	/* Now we are done with SELECT-like processing, and can get on with
	 * transforming the target list to match the UPDATE target columns.*/
	qry->targetList = transformUpdateTargetList(pstate, stmt->targetList);  // 处理SQL语句中的 set id =1 

	qry->rtable = pstate->p_rtable;
	qry->jointree = makeFromExpr(pstate->p_joinlist, qual);
	qry->hasTargetSRFs = pstate->p_hasTargetSRFs;
	qry->hasSubLinks = pstate->p_hasSubLinks;

	assign_query_collations(pstate, qry);

	return qry;
}
```
这里面要重点关注一下`transformTargetList`，会将抽象语法树中的`ResTarget`转为查询器的`TargetEntry`。
```c++
typedef struct TargetEntry
{
	Expr		xpr;
	Expr	   *expr;			/* expression to evaluate */
	AttrNumber	resno;			/* attribute number (see notes above) */
	char	   *resname;		/* name of the column (could be NULL) */
	Index		ressortgroupref;	/* nonzero if referenced by a sort/group clause */
	Oid			resorigtbl;		/* OID of column's source table */
	AttrNumber	resorigcol;		/* column's number in source table */
	bool		resjunk;		/* set to true to eliminate the attribute from final target list */
} TargetEntry;
```

> 对于其内部处理可参考源码`src/backend/parser`中的相关处理，这里不再细述。需要重点阅读一下[README](https://github.com/postgres/postgres/blob/master/src/backend/parser/README)，PG源码中所有的README都是非常好的资料，一定要认真读。

#### 优化器——生成执行计划
这块的内容很多，主要的逻辑是先进行逻辑优化，比如子查询、子链接、常量表达式、选择下推等等的处理，因为我们要分析的这条语句十分简单，所以逻辑优化的这部分都没有涉及到。物理优化，涉及到选择率，代价估计，索引扫描还是顺序扫描，选择那种连接方式，应用动态规划呢还是基因算法，选择nestloop-join、merge-join还是hash-join等。因为我们这个表没有建索引，更新单表也不涉及到多表连接，所以物理优化这块涉及的也不多。路径生成，生成最佳路径，再由最佳路径生成执行计划。

在路径生成这块，最基础的是对表的扫描方式，比如顺序扫描、索引扫描，再往上是连接方式，采用那种连接方式，再往上是比如排序、Limit等路径......，由底向上生成路径。我们要分析的语句很简单，没有其他处理，就顺序扫描再更新就可以了。

这里先不考虑并行执行计划。我们先看一下其执行计划结果：
```sql
postgres@postgres=# explain update dtea set id = 0;
                          QUERY PLAN                          
--------------------------------------------------------------
 Update on dtea  (cost=0.00..19.00 rows=900 width=68)
   ->  Seq Scan on dtea  (cost=0.00..19.00 rows=900 width=68)
(2 rows)
```
下面我们分析一下其执行计划的生成流程：
```c++
// 由查询树Query--> Path --> Plan (PlannedStmt)
pg_plan_queries()
--> pg_plan_query()
    --> planner()
        --> standard_planner(Query *parse, const char *query_string, int cursorOptions,ParamListInfo boundParams)
            // 由Query---> PlannerInfo
            --> subquery_planner(glob, parse, NULL,false, tuple_fraction);  // 涉及到很多逻辑优化的内容，很多不列出
                --> pull_up_sublinks(root);
                --> pull_up_subqueries(root);   // 这里只列出几个重要的逻辑优化内容，其他的不再列出......
                // 如果是update/delete分区表继承表则走inheritance_planner(),其他情况走grouping_planner()
                --> inheritance_planner()   // update/delete分区表继承表的情况
                    --> grouping_planner()
                --> grouping_planner() // 非分区表、继承表的情况
                    --> preprocess_targetlist(root); // update虽然只更新一列，但是插入一条新元组的时候，需要知道其他列信息.
                        --> rewriteTargetListUD(parse, target_rte, target_relation);
                        --> expand_targetlist()
                    --> query_planner(root, standard_qp_callback, &qp_extra);   // 重要
                        --> add_base_rels_to_query()
                        --> deconstruct_jointree(root);
                        --> add_other_rels_to_query(root); // 展开分区表到PlannerInfo中的相关字段中 
                            --> expand_inherited_rtentry()  
								--> expand_planner_arrays(root, num_live_parts);
                        --> make_one_rel(root, joinlist);   // 动态规划，基因算法在这个里面
                            --> set_base_rel_sizes(root); 
                                --> set_rel_size();
									--> set_append_rel_size(root, rel, rti, rte);	// 如果是分区表或者继承走这里，否则走下面
										--> set_rel_size(root, childrel, childRTindex, childRTE);	// 处理子分区表
											--> set_plain_rel_size(root, rel, rte);
                                    --> set_plain_rel_size()  	// 如果不是分区表或者继承
                                        --> set_baserel_size_estimates()
                            --> set_base_rel_pathlists(root);
								--> set_rel_pathlist(root, rel, rti, root->simple_rte_array[rti]);
									--> set_append_rel_pathlist(root, rel, rti, rte);	// 生成各分区表的访问路径
                            --> make_rel_from_joinlist(root, joinlist);
                    --> apply_scanjoin_target_to_paths()
                    --> create_modifytable_path()
            // 由PlannerInfo---> RelOptInfo 
            --> fetch_upper_rel(root, UPPERREL_FINAL, NULL);
            // 由RelOptInfo---> Path
            --> get_cheapest_fractional_path(final_rel, tuple_fraction);
            // 由 PlannerInfo+Path  ---> Plan
            --> create_plan(root, best_path);
            // 后续处理，由Plan ---> PlannedStmt
```

核心数据结构： PlannedStmt、PlannerInfo、RelOptInfo（存储访问路径及其代价）、Path

`Path`：所有的路径都继承自`Path`，所以这个比较重要。
```c++
typedef struct Path
{
	NodeTag		type;
	NodeTag		pathtype;		/* tag identifying scan/join method */

	RelOptInfo *parent;			/* the relation this path can build */
	PathTarget *pathtarget;		/* list of Vars/Exprs, cost, width */

	ParamPathInfo *param_info;	/* parameterization info, or NULL if none */

	bool		parallel_aware; /* engage parallel-aware logic? */
	bool		parallel_safe;	/* OK to use as part of parallel plan? */
	int			parallel_workers;	/* desired # of workers; 0 = not parallel */

	/* estimated size/costs for path (see costsize.c for more info) */
	double		rows;			/* estimated number of result tuples */
	Cost		startup_cost;	/* cost expended before fetching any tuples */
	Cost		total_cost;		/* total cost (assuming all tuples fetched) */

	List	   *pathkeys;		/* sort ordering of path's output */
	/* pathkeys is a List of PathKey nodes; see above */
} Path;

/* ModifyTablePath represents performing INSERT/UPDATE/DELETE modifications
 * We represent most things that will be in the ModifyTable plan node
 * literally, except we have child Path(s) not Plan(s).  But analysis of the
 * OnConflictExpr is deferred to createplan.c, as is collection of FDW data. */
typedef struct ModifyTablePath
{
	Path		path;			// 可以看到ModifyTablePath继承自Path
	CmdType		operation;		/* INSERT, UPDATE, or DELETE */
	bool		canSetTag;		/* do we set the command tag/es_processed? */
	Index		nominalRelation;	/* Parent RT index for use of EXPLAIN */
	Index		rootRelation;	/* Root RT index, if target is partitioned */
	bool		partColsUpdated;	/* some part key in hierarchy updated */
	List	   *resultRelations;	/* integer list of RT indexes */
	List	   *subpaths;		/* Path(s) producing source data */
	List	   *subroots;		/* per-target-table PlannerInfos */
	List	   *withCheckOptionLists;	/* per-target-table WCO lists */
	List	   *returningLists; /* per-target-table RETURNING tlists */
	List	   *rowMarks;		/* PlanRowMarks (non-locking only) */
	OnConflictExpr *onconflict; /* ON CONFLICT clause, or NULL */
	int			epqParam;		/* ID of Param for EvalPlanQual re-eval */
} ModifyTablePath;
```
生成update执行路径，最终都是要生成ModifyTablePath，本例中路径生成过程： Path-->ProjectionPath-->ModifyTablePath，也就是先顺序扫描表，再修改表。后面由路径生成执行计划。
```c++
/* create_modifytable_path
 *	  Creates a pathnode that represents performing INSERT/UPDATE/DELETE mods
 *
 * 'rel' is the parent relation associated with the result
 * 'resultRelations' is an integer list of actual RT indexes of target rel(s)
 * 'subpaths' is a list of Path(s) producing source data (one per rel)
 * 'subroots' is a list of PlannerInfo structs (one per rel)*/
ModifyTablePath *create_modifytable_path(PlannerInfo *root, RelOptInfo *rel,
						CmdType operation, bool canSetTag,
						Index nominalRelation, Index rootRelation,
						bool partColsUpdated,
						List *resultRelations, List *subpaths,
						List *subroots,
						List *withCheckOptionLists, List *returningLists,
						List *rowMarks, OnConflictExpr *onconflict,
						int epqParam)
{
	ModifyTablePath *pathnode = makeNode(ModifyTablePath);
	double		total_size;
	ListCell   *lc;

	Assert(list_length(resultRelations) == list_length(subpaths));
	Assert(list_length(resultRelations) == list_length(subroots));
	Assert(withCheckOptionLists == NIL || list_length(resultRelations) == list_length(withCheckOptionLists));
	Assert(returningLists == NIL || list_length(resultRelations) == list_length(returningLists));

	pathnode->path.pathtype = T_ModifyTable;
	pathnode->path.parent = rel;

	pathnode->path.pathtarget = rel->reltarget;	/* pathtarget is not interesting, just make it minimally valid */
	/* For now, assume we are above any joins, so no parameterization */
	pathnode->path.param_info = NULL;
	pathnode->path.parallel_aware = false;
	pathnode->path.parallel_safe = false;
	pathnode->path.parallel_workers = 0;
	pathnode->path.pathkeys = NIL;

	/** Compute cost & rowcount as sum of subpath costs & rowcounts.
	 *
	 * Currently, we don't charge anything extra for the actual table
	 * modification work, nor for the WITH CHECK OPTIONS or RETURNING
	 * expressions if any.  It would only be window dressing, since
	 * ModifyTable is always a top-level node and there is no way for the
	 * costs to change any higher-level planning choices.  But we might want
	 * to make it look better sometime.*/
	pathnode->path.startup_cost = 0;
	pathnode->path.total_cost = 0;
	pathnode->path.rows = 0;
	total_size = 0;
	foreach(lc, subpaths)
	{
		Path	   *subpath = (Path *) lfirst(lc);

		if (lc == list_head(subpaths))	/* first node? */
			pathnode->path.startup_cost = subpath->startup_cost;
		pathnode->path.total_cost += subpath->total_cost;
		pathnode->path.rows += subpath->rows;
		total_size += subpath->pathtarget->width * subpath->rows;
	}

	/* Set width to the average width of the subpath outputs.  XXX this is
	 * totally wrong: we should report zero if no RETURNING, else an average
	 * of the RETURNING tlist widths.  But it's what happened historically,
	 * and improving it is a task for another day.*/
	if (pathnode->path.rows > 0)
		total_size /= pathnode->path.rows;
	pathnode->path.pathtarget->width = rint(total_size);

	pathnode->operation = operation;
	pathnode->canSetTag = canSetTag;
	pathnode->nominalRelation = nominalRelation;
	pathnode->rootRelation = rootRelation;
	pathnode->partColsUpdated = partColsUpdated;
	pathnode->resultRelations = resultRelations;
	pathnode->subpaths = subpaths;
	pathnode->subroots = subroots;
	pathnode->withCheckOptionLists = withCheckOptionLists;
	pathnode->returningLists = returningLists;
	pathnode->rowMarks = rowMarks;
	pathnode->onconflict = onconflict;
	pathnode->epqParam = epqParam;

	return pathnode;
}
```
现在我们生成了最优的update路径，需要由路径生成执行计划：
```c++
Plan *create_plan(PlannerInfo *root, Path *best_path)
{
	Plan	   *plan;
	Assert(root->plan_params == NIL);	/* plan_params should not be in use in current query level */

	/* Initialize this module's workspace in PlannerInfo */
	root->curOuterRels = NULL;
	root->curOuterParams = NIL;

	/* Recursively process the path tree, demanding the correct tlist result */
	plan = create_plan_recurse(root, best_path, CP_EXACT_TLIST);	// 实际实现是在这里

	/** Make sure the topmost plan node's targetlist exposes the original
	 * column names and other decorative info.  Targetlists generated within
	 * the planner don't bother with that stuff, but we must have it on the
	 * top-level tlist seen at execution time.  However, ModifyTable plan
	 * nodes don't have a tlist matching the querytree targetlist.*/
	if (!IsA(plan, ModifyTable))
		apply_tlist_labeling(plan->targetlist, root->processed_tlist);

	/** Attach any initPlans created in this query level to the topmost plan
	 * node.  (In principle the initplans could go in any plan node at or
	 * above where they're referenced, but there seems no reason to put them
	 * any lower than the topmost node for the query level.  Also, see
	 * comments for SS_finalize_plan before you try to change this.)*/
	SS_attach_initplans(root, plan);

	/* Check we successfully assigned all NestLoopParams to plan nodes */
	if (root->curOuterParams != NIL)
		elog(ERROR, "failed to assign all NestLoopParams to plan nodes");

	/** Reset plan_params to ensure param IDs used for nestloop params are not re-used later*/
	root->plan_params = NIL;

	return plan;
}

// 由最佳路径生成最佳执行计划
static ModifyTable *create_modifytable_plan(PlannerInfo *root, ModifyTablePath *best_path)
{
	ModifyTable *plan;
	List	   *subplans = NIL;
	ListCell   *subpaths,
			   *subroots;

	/* Build the plan for each input path */
	forboth(subpaths, best_path->subpaths, subroots, best_path->subroots)
	{
		Path	   *subpath = (Path *) lfirst(subpaths);
		PlannerInfo *subroot = (PlannerInfo *) lfirst(subroots);
		Plan	   *subplan;

		/* In an inherited UPDATE/DELETE, reference the per-child modified
		 * subroot while creating Plans from Paths for the child rel.  This is
		 * a kluge, but otherwise it's too hard to ensure that Plan creation
		 * functions (particularly in FDWs) don't depend on the contents of
		 * "root" matching what they saw at Path creation time.  The main
		 * downside is that creation functions for Plans that might appear
		 * below a ModifyTable cannot expect to modify the contents of "root"
		 * and have it "stick" for subsequent processing such as setrefs.c.
		 * That's not great, but it seems better than the alternative.*/
		subplan = create_plan_recurse(subroot, subpath, CP_EXACT_TLIST);

		/* Transfer resname/resjunk labeling, too, to keep executor happy */
		apply_tlist_labeling(subplan->targetlist, subroot->processed_tlist);

		subplans = lappend(subplans, subplan);
	}

	plan = make_modifytable(root,
							best_path->operation,
							best_path->canSetTag,
							best_path->nominalRelation,
							best_path->rootRelation,
							best_path->partColsUpdated,
							best_path->resultRelations,
							subplans,
							best_path->subroots,
							best_path->withCheckOptionLists,
							best_path->returningLists,
							best_path->rowMarks,
							best_path->onconflict,
							best_path->epqParam);

	copy_generic_path_info(&plan->plan, &best_path->path);

	return plan;
}
```
最终的执行计划是`ModifyTable`：
```c++
/* ----------------
 *	 ModifyTable node -
 *		Apply rows produced by subplan(s) to result table(s),
 *		by inserting, updating, or deleting.
 *
 * If the originally named target table is a partitioned table, both
 * nominalRelation and rootRelation contain the RT index of the partition
 * root, which is not otherwise mentioned in the plan.  Otherwise rootRelation
 * is zero.  However, nominalRelation will always be set, as it's the rel that
 * EXPLAIN should claim is the INSERT/UPDATE/DELETE target.
 *
 * Note that rowMarks and epqParam are presumed to be valid for all the
 * subplan(s); they can't contain any info that varies across subplans.
 * ----------------*/
typedef struct ModifyTable
{
	Plan		plan;
	CmdType		operation;		/* INSERT, UPDATE, or DELETE */
	bool		canSetTag;		/* do we set the command tag/es_processed? */
	Index		nominalRelation;	/* Parent RT index for use of EXPLAIN */
	Index		rootRelation;	/* Root RT index, if target is partitioned */
	bool		partColsUpdated;	/* some part key in hierarchy updated */
	List	   *resultRelations;	/* integer list of RT indexes */
	int			resultRelIndex; /* index of first resultRel in plan's list */
	int			rootResultRelIndex; /* index of the partitioned table root */
	List	   *plans;			/* plan(s) producing source data */
	List	   *withCheckOptionLists;	/* per-target-table WCO lists */
	List	   *returningLists; /* per-target-table RETURNING tlists */
	List	   *fdwPrivLists;	/* per-target-table FDW private data lists */
	Bitmapset  *fdwDirectModifyPlans;	/* indices of FDW DM plans */
	List	   *rowMarks;		/* PlanRowMarks (non-locking only) */
	int			epqParam;		/* ID of Param for EvalPlanQual re-eval */
	OnConflictAction onConflictAction;	/* ON CONFLICT action */
	List	   *arbiterIndexes; /* List of ON CONFLICT arbiter index OIDs  */
	List	   *onConflictSet;	/* SET for INSERT ON CONFLICT DO UPDATE */
	Node	   *onConflictWhere;	/* WHERE for ON CONFLICT UPDATE */
	Index		exclRelRTI;		/* RTI of the EXCLUDED pseudo relation */
	List	   *exclRelTlist;	/* tlist of the EXCLUDED pseudo relation */
} ModifyTable;
```

#### 执行器
根据上面的执行计划，去执行。主要是各种算子的实现，其中要理解执行器的运行原理，主要是火山模型，一次一元组。我们看一下其调用过程。
```c++
CreatePortal("", true, true);
PortalDefineQuery(portal,NULL,query_string,commandTag,plantree_list,NULL);
PortalStart(portal, NULL, 0, InvalidSnapshot);
PortalRun(portal,FETCH_ALL,true,true,receiver,receiver,&qc);
--> PortalRunMulti()
	--> ProcessQuery()
		--> ExecutorStart(queryDesc, 0);
			--> standard_ExecutorStart()
				--> InitPlan(queryDesc, eflags);
					--> ExecInitNode(plan, estate, eflags);		
						--> ExecInitModifyTable()
		--> ExecutorRun(queryDesc, ForwardScanDirection, 0L, true);
			--> standard_ExecutorRun()
				--> ExecutePlan()
					--> ExecProcNode(planstate);	// 一次一元组 火山模型
						--> node->ExecProcNode(node);
							--> ExecProcNodeFirst(PlanState *node)
								--> node->ExecProcNode(node);
									--> ExecModifyTable(PlanState *pstate)
										--> ExecUpdate()
											--> table_tuple_update(Relation rel, ......)
												--> rel->rd_tableam->tuple_update()
													--> heapam_tuple_update(Relation relation, ......)
														--> heap_update(relation, otid, tuple, cid, ......)

		--> ExecutorFinish(queryDesc);
		--> ExecutorEnd(queryDesc);
PortalDrop(portal, false);
```

关键数据结构：
```c++
// ModifyTableState information
typedef struct ModifyTableState
{
	PlanState	ps;				/* its first field is NodeTag */
	CmdType		operation;		/* INSERT, UPDATE, or DELETE */
	bool		canSetTag;		/* do we set the command tag/es_processed? */
	bool		mt_done;		/* are we done? */
	PlanState **mt_plans;		/* subplans (one per target rel) */
	int			mt_nplans;		/* number of plans in the array */
	int			mt_whichplan;	/* which one is being executed (0..n-1) */
	TupleTableSlot **mt_scans;	/* input tuple corresponding to underlying
								 * plans */
	ResultRelInfo *resultRelInfo;	/* per-subplan target relations */
	ResultRelInfo *rootResultRelInfo;	/* root target relation (partitioned
										 * table root) */
	List	  **mt_arowmarks;	/* per-subplan ExecAuxRowMark lists */
	EPQState	mt_epqstate;	/* for evaluating EvalPlanQual rechecks */
	bool		fireBSTriggers; /* do we need to fire stmt triggers? */

	/* Slot for storing tuples in the root partitioned table's rowtype during
	 * an UPDATE of a partitioned table. */
	TupleTableSlot *mt_root_tuple_slot;

	struct PartitionTupleRouting *mt_partition_tuple_routing;	/* Tuple-routing support info */

	struct TransitionCaptureState *mt_transition_capture;	/* controls transition table population for specified operation */

	/* controls transition table population for INSERT...ON CONFLICT UPDATE */
	struct TransitionCaptureState *mt_oc_transition_capture;

	/* Per plan map for tuple conversion from child to root */
	TupleConversionMap **mt_per_subplan_tupconv_maps;
} ModifyTableState;
```

核心执行算子实现：
```c++
/* ----------------------------------------------------------------
 *	   ExecModifyTable
 *
 *		Perform table modifications as required, and return RETURNING results
 *		if needed.
 * ---------------------------------------------------------------- */
static TupleTableSlot *ExecModifyTable(PlanState *pstate)
{
	ModifyTableState *node = castNode(ModifyTableState, pstate);
	PartitionTupleRouting *proute = node->mt_partition_tuple_routing;
	EState	   *estate = node->ps.state;
	CmdType		operation = node->operation;
	ResultRelInfo *saved_resultRelInfo;
	ResultRelInfo *resultRelInfo;
	PlanState  *subplanstate;
	JunkFilter *junkfilter;
	TupleTableSlot *slot;
	TupleTableSlot *planSlot;
	ItemPointer tupleid;
	ItemPointerData tuple_ctid;
	HeapTupleData oldtupdata;
	HeapTuple	oldtuple;

	CHECK_FOR_INTERRUPTS();

	/* This should NOT get called during EvalPlanQual; we should have passed a
	 * subplan tree to EvalPlanQual, instead.  Use a runtime test not just
	 * Assert because this condition is easy to miss in testing. */
	if (estate->es_epq_active != NULL)
		elog(ERROR, "ModifyTable should not be called during EvalPlanQual");

	/* If we've already completed processing, don't try to do more.  We need
	 * this test because ExecPostprocessPlan might call us an extra time, and
	 * our subplan's nodes aren't necessarily robust against being called
	 * extra times.*/
	if (node->mt_done)
		return NULL;

	/* On first call, fire BEFORE STATEMENT triggers before proceeding.*/
	if (node->fireBSTriggers)
	{
		fireBSTriggers(node);
		node->fireBSTriggers = false;
	}

	/* Preload local variables */
	resultRelInfo = node->resultRelInfo + node->mt_whichplan;
	subplanstate = node->mt_plans[node->mt_whichplan];
	junkfilter = resultRelInfo->ri_junkFilter;

	/* es_result_relation_info must point to the currently active result relation while we are within this ModifyTable node.  
	 * Even though ModifyTable nodes can't be nested statically, they can be nested
	 * dynamically (since our subplan could include a reference to a modifying
	 * CTE).  So we have to save and restore the caller's value.*/
	saved_resultRelInfo = estate->es_result_relation_info;
	estate->es_result_relation_info = resultRelInfo;

	/* Fetch rows from subplan(s), and execute the required table modification for each row.*/
	for (;;)
	{
		/* Reset the per-output-tuple exprcontext.  This is needed because
		 * triggers expect to use that context as workspace.  It's a bit ugly
		 * to do this below the top level of the plan, however.  We might need to rethink this later.*/
		ResetPerTupleExprContext(estate);

		/* Reset per-tuple memory context used for processing on conflict and
		 * returning clauses, to free any expression evaluation storage allocated in the previous cycle. */
		if (pstate->ps_ExprContext)
			ResetExprContext(pstate->ps_ExprContext);

		planSlot = ExecProcNode(subplanstate);
		if (TupIsNull(planSlot))
		{
			/* advance to next subplan if any */
			node->mt_whichplan++;	// 分区表的update，每个分区分布对应一个subplan，当执行完一个分区再执行下一个分区
			if (node->mt_whichplan < node->mt_nplans)
			{
				resultRelInfo++;
				subplanstate = node->mt_plans[node->mt_whichplan];
				junkfilter = resultRelInfo->ri_junkFilter;
				estate->es_result_relation_info = resultRelInfo;
				EvalPlanQualSetPlan(&node->mt_epqstate, subplanstate->plan, node->mt_arowmarks[node->mt_whichplan]);
				/* Prepare to convert transition tuples from this child. */
				if (node->mt_transition_capture != NULL) {
					node->mt_transition_capture->tcs_map = tupconv_map_for_subplan(node, node->mt_whichplan);
				}
				if (node->mt_oc_transition_capture != NULL) {
					node->mt_oc_transition_capture->tcs_map = tupconv_map_for_subplan(node, node->mt_whichplan);
				}
				continue;
			}
			else
				break;
		}

		/* Ensure input tuple is the right format for the target relation.*/
		if (node->mt_scans[node->mt_whichplan]->tts_ops != planSlot->tts_ops) {
			ExecCopySlot(node->mt_scans[node->mt_whichplan], planSlot);
			planSlot = node->mt_scans[node->mt_whichplan];
		}

		/* If resultRelInfo->ri_usesFdwDirectModify is true, all we need to do here is compute the RETURNING expressions.*/
		if (resultRelInfo->ri_usesFdwDirectModify)
		{
			Assert(resultRelInfo->ri_projectReturning);
			slot = ExecProcessReturning(resultRelInfo->ri_projectReturning, RelationGetRelid(resultRelInfo->ri_RelationDesc), NULL, planSlot);

			estate->es_result_relation_info = saved_resultRelInfo;
			return slot;
		}

		EvalPlanQualSetSlot(&node->mt_epqstate, planSlot);
		slot = planSlot;

		tupleid = NULL;
		oldtuple = NULL;
		if (junkfilter != NULL)
		{
			/* extract the 'ctid' or 'wholerow' junk attribute.*/
			if (operation == CMD_UPDATE || operation == CMD_DELETE)
			{
				char		relkind;
				Datum		datum;
				bool		isNull;

				relkind = resultRelInfo->ri_RelationDesc->rd_rel->relkind;
				if (relkind == RELKIND_RELATION || relkind == RELKIND_MATVIEW)
				{
					datum = ExecGetJunkAttribute(slot,junkfilter->jf_junkAttNo,&isNull);
					/* shouldn't ever get a null result... */
					if (isNull)
						elog(ERROR, "ctid is NULL");

					tupleid = (ItemPointer) DatumGetPointer(datum);
					tuple_ctid = *tupleid;	/* be sure we don't free ctid!! */
					tupleid = &tuple_ctid;
				}
				/* Use the wholerow attribute, when available, to reconstruct the old relation tuple.*/
				else if (AttributeNumberIsValid(junkfilter->jf_junkAttNo))
				{
					datum = ExecGetJunkAttribute(slot,junkfilter->jf_junkAttNo,&isNull);
					/* shouldn't ever get a null result... */
					if (isNull)
						elog(ERROR, "wholerow is NULL");

					oldtupdata.t_data = DatumGetHeapTupleHeader(datum);
					oldtupdata.t_len = HeapTupleHeaderGetDatumLength(oldtupdata.t_data);
					ItemPointerSetInvalid(&(oldtupdata.t_self));
					/* Historically, view triggers see invalid t_tableOid. */
					oldtupdata.t_tableOid = (relkind == RELKIND_VIEW) ? InvalidOid : RelationGetRelid(resultRelInfo->ri_RelationDesc);
					oldtuple = &oldtupdata;
				}
				else
					Assert(relkind == RELKIND_FOREIGN_TABLE);
			}

			/* apply the junkfilter if needed. */
			if (operation != CMD_DELETE)
				slot = ExecFilterJunk(junkfilter, slot);
		}

		switch (operation)
		{
			case CMD_INSERT:
				if (proute)				/* Prepare for tuple routing if needed. */
					slot = ExecPrepareTupleRouting(node, estate, proute, resultRelInfo, slot);
				slot = ExecInsert(node, slot, planSlot, NULL, estate->es_result_relation_info, estate, node->canSetTag);
				if (proute)				/* Revert ExecPrepareTupleRouting's state change. */
					estate->es_result_relation_info = resultRelInfo;
				break;
			case CMD_UPDATE:
				slot = ExecUpdate(node, tupleid, oldtuple, slot, planSlot,
								  &node->mt_epqstate, estate, node->canSetTag);
				break;
			case CMD_DELETE:
				slot = ExecDelete(node, tupleid, oldtuple, planSlot,
								  &node->mt_epqstate, estate,
								  true, node->canSetTag, false /* changingPart */ , NULL, NULL);
				break;
			default:
				elog(ERROR, "unknown operation");
				break;
		}

		/* If we got a RETURNING result, return it to caller.  We'll continue the work on next call.*/
		if (slot) {
			estate->es_result_relation_info = saved_resultRelInfo;
			return slot;
		}
	}

	estate->es_result_relation_info = saved_resultRelInfo;	/* Restore es_result_relation_info before exiting */
	fireASTriggers(node);	/* We're done, but fire AFTER STATEMENT triggers before exiting.*/

	node->mt_done = true;

	return NULL;
}
```

#### 事务
这一块主要是要理解PG中update语句并不是原地更新元组，而是插入一条新元组。因为PG实现MVCC与Mysql，Oracle的实现方式有所不同，并不是通过undo日志实现的，相当于把undo日志记录到了原有的表中，并不是单独存放在一个地方。具体的不再细述，内容太多了，以后再分析事务部分。



---
#### 参考文档
[PgSQL · 源码分析 · PG优化器浅析](http://mysql.taobao.org/monthly/2016/09/07/)