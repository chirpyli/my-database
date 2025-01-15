### PostgreSQL源码分析——索引扫描
这里，我们分析一下索引扫描的过程，以最简单的`select * from t1 where a = 100;`语句为例，分析一下索引扫描的过程。

```sql
postgres@postgres=# \d t1;
                 Table "public.t1"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 a      | integer |           | not null | 
 b      | integer |           |          | 
Indexes:
    "t1_pkey" PRIMARY KEY, btree (a)

-- 顺序扫描
postgres@postgres=# explain select * from t1;
                      QUERY PLAN                      
------------------------------------------------------
 Seq Scan on t1  (cost=0.00..15.00 rows=1000 width=8)
(1 row)

-- 索引扫描
postgres@postgres=# explain select * from t1 where a = 100;
                            QUERY PLAN                            
------------------------------------------------------------------
 Index Scan using t1_pkey on t1  (cost=0.28..2.29 rows=1 width=8)
   Index Cond: (a = 100)
(2 rows)
```

#### 语法解析以及语义分析
这部分前面已有分析过，这里简要描述：

```c++
解析层面流程如下：
```c++
exec_simple_query
--> pg_parse_query              // 生成抽象语法树
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite      // 语义分析，转为查询树Query
```
语法表示可查看gram.y中的定义。
```c++
simple_select:
			SELECT opt_all_clause opt_target_list   // select *
			into_clause from_clause where_clause    // from t1 where a = 100;
			group_clause having_clause window_clause
				{
					SelectStmt *n = makeNode(SelectStmt);
					// ......
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
// 表示t1
table_ref:	relation_expr opt_alias_clause
				{
					$1->alias = $2;
					$$ = (Node *) $1;
				}
where_clause:
			WHERE a_expr							{ $$ = $2; }	// 表示 where a = 100;
			| /*EMPTY*/								{ $$ = NULL; }
		;
```

关键数据结构：
```c++
// select语句的抽象语法树表示
typedef struct SelectStmt
{
	NodeTag		type;
    // 对应select *
	List	   *targetList;		/* the target list (of ResTarget) */ 
    // 对应         from t1
	List	   *fromClause;		/* the FROM clause */  // 保存RangeVar节点

    // ...... 

} SelectStmt;

/* RangeVar - range variable, used in FROM clauses */
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
} RangeVar;    // 存储表的所在库名，模式名，表名信息

语义分析流程：
```c++
pg_analyze_and_rewrite   // 语义分析，生成查询树Query
--> pg_analyze
    --> transformStmt
        --> transformSelectStmt
            --> transformFromClause  // 处理表， 将RangeVar转为RangeTblEntry
                --> transformFromClauseItem
                    --> transformTableEntry     
            --> transformTargetList     // 处理 * ，展开为a, b
			--> transformWhereClause    // 处理 where a = 100
--> pg_rewrite_query
```

#### 查询优化

主要是生成执行计划，前面已经通过`EXPLAIN`命令得知其执行计划为索引扫描，让我们具体看一下其是如何生成的。这里选择率以及代价估算先不进行分析，后续单独进行分析。
```c++
pg_plan_queries
--> pg_plan_query
    --> planner
        --> standard_planner
            --> subquery_planner
                --> grouping_planner
                    --> query_planner 		// Generate a path (that is, a simplified plan) for a basic query
                        --> setup_simple_rel_arrays
                        --> add_base_rels_to_query
                            --> build_simple_rel
						--> generate_base_implied_equalities(root);
							--> generate_base_implied_equalities_const
								--> distribute_restrictinfo_to_rels	// 选择下推， where a = 100 , 绑定到表t1中
									--> find_base_rel
									--> rel->baserestrictinfo = lappend(rel->baserestrictinfo, restrictinfo);
						--> extract_restriction_or_clauses
                        --> make_one_rel	// Finds all possible access paths for executing a query
                            --> set_base_rel_sizes
                                --> set_rel_size
                                    --> set_plain_rel_size
                                        --> set_baserel_size_estimates	// 选择率计算，计算代价Cost要用
											--> clauselist_selectivity // 选择率计算， 会将表以及where a = 100限制条件传进去进行计算
												--> clauselist_selectivity_simple
													--> clause_selectivity
														--> restriction_selectivity /* Estimate selectivity for a restriction clause. */
															--> eqsel
																--> eqsel_internal
																	--> var_eq_const
																		--> get_variable_numdistinct
                            --> set_base_rel_pathlists
                                --> set_rel_pathlist // Build access paths for a base relation
                                        --> create_seqscan_path // 生成顺序扫描路径
										--> create_index_paths	 // 索引扫描路径
											--> get_index_paths
												--> build_index_paths
													--> check_index_only  // 检查是否可以IndexOnlyScan，仅索引扫描
                            --> make_rel_from_joinlist
                    --> apply_scanjoin_target_to_paths
            --> create_plan
                --> create_scan_plan
                    --> create_indexscan_plan
						--> if indexonly
							--> make_indexonlyscan
							else
							--> make_indexscan
```
这里有个函数要重点说明一下，在涉及多表路径规划时，单表路径是最基础的，单表路径可以是顺序扫描、索引扫描、TID扫描等，这个是最底层的。
```c++
/* Build access paths for a plain relation (no subquery, no inheritance) */
void set_plain_rel_pathlist(PlannerInfo *root, RelOptInfo *rel, RangeTblEntry *rte)
{
	Relids		required_outer;

	/* We don't support pushing join clauses into the quals of a seqscan, but
	 * it could still have required parameterization due to LATERAL refs in its tlist. */
	required_outer = rel->lateral_relids;

	add_path(rel, create_seqscan_path(root, rel, required_outer, 0));	// 顺序扫描

	if (rel->consider_parallel && required_outer == NULL)		// 尝试并行顺序扫描
		create_plain_partial_paths(root, rel);

	create_index_paths(root, rel);			// 索引扫描
	create_tidscan_paths(root, rel);      // TID扫描
}
```
最后由最佳路径`Path`生成顺序扫描执行计划`SeqScan`：
```c++
/* ==========
 * Scan nodes
 * ========== */
typedef struct Scan
{
	Plan		plan;
	// 指明扫描那个表，
	Index		scanrelid;		/* relid is index into the range table */
} Scan;

/* ----------------
 *		sequential scan node
 * ---------------- */
typedef Scan SeqScan;
```

其中有个非常重要的函数， 这个函数有个细节，就是IndexOnlyScan，在这里会进行一个判断，是否可以进行IndexOnly扫描，判断条件时target列是否仅索引列，如果满足的话，则可以用IndexOnlyScan。当然前提是`enable_indexonlyscan = true`。比如下面的语句：
```sql
postgres@postgres=# show enable_indexonlyscan ;
 enable_indexonlyscan 
----------------------
 on
(1 row)

-- Index Only Scan
postgres@postgres=# explain select a from t1 where a = 100;
                              QUERY PLAN                               
-----------------------------------------------------------------------
 Index Only Scan using t1_pkey on t1  (cost=0.28..2.29 rows=1 width=4)
   Index Cond: (a = 100)
(2 rows)
-- 关闭 enable_indexonlyscan后，不能进行 IndexOnlyScan
postgres@postgres=# set enable_indexonlyscan = false;
SET
postgres@postgres=# explain select a from t1 where a = 100;
                            QUERY PLAN                            
------------------------------------------------------------------
 Index Scan using t1_pkey on t1  (cost=0.28..2.29 rows=1 width=4)
   Index Cond: (a = 100)
(2 rows)
```
我们继续看一下如何构造IndexPath
```c++
/*
 * build_index_paths
 *	  Given an index and a set of index clauses for it, construct zero
 *	  or more IndexPaths. It also constructs zero or more partial IndexPaths.
 *
 * We return a list of paths because (1) this routine checks some cases
 * that should cause us to not generate any IndexPath, and (2) in some
 * cases we want to consider both a forward and a backward scan, so as
 * to obtain both sort orders.  Note that the paths are just returned
 * to the caller and not immediately fed to add_path().
 *
 * At top level, useful_predicate should be exactly the index's predOK flag
 * (ie, true if it has a predicate that was proven from the restriction
 * clauses).  When working on an arm of an OR clause, useful_predicate
 * should be true if the predicate required the current OR list to be proven.
 * Note that this routine should never be called at all if the index has an
 * unprovable predicate.
 *
 * scantype indicates whether we want to create plain indexscans, bitmap
 * indexscans, or both.  When it's ST_BITMAPSCAN, we will not consider
 * index ordering while deciding if a Path is worth generating.
 *
 * If skip_nonnative_saop is non-NULL, we ignore ScalarArrayOpExpr clauses
 * unless the index AM supports them directly, and we set *skip_nonnative_saop
 * to true if we found any such clauses (caller must initialize the variable
 * to false).  If it's NULL, we do not ignore ScalarArrayOpExpr clauses.
 *
 * If skip_lower_saop is non-NULL, we ignore ScalarArrayOpExpr clauses for
 * non-first index columns, and we set *skip_lower_saop to true if we found
 * any such clauses (caller must initialize the variable to false).  If it's
 * NULL, we do not ignore non-first ScalarArrayOpExpr clauses, but they will
 * result in considering the scan's output to be unordered.
 *
 * 'rel' is the index's heap relation
 * 'index' is the index for which we want to generate paths
 * 'clauses' is the collection of indexable clauses (IndexClause nodes)
 * 'useful_predicate' indicates whether the index has a useful predicate
 * 'scantype' indicates whether we need plain or bitmap scan support
 * 'skip_nonnative_saop' indicates whether to accept SAOP if index AM doesn't
 * 'skip_lower_saop' indicates whether to accept non-first-column SAOP
 */
static List *
build_index_paths(PlannerInfo *root, RelOptInfo *rel,
				  IndexOptInfo *index, IndexClauseSet *clauses,
				  bool useful_predicate,
				  ScanTypeControl scantype,
				  bool *skip_nonnative_saop,
				  bool *skip_lower_saop)
{
	List	   *result = NIL;
	IndexPath  *ipath;
	List	   *index_clauses;
	Relids		outer_relids;
	double		loop_count;
	List	   *orderbyclauses;
	List	   *orderbyclausecols;
	List	   *index_pathkeys;
	List	   *useful_pathkeys;
	bool		found_lower_saop_clause;
	bool		pathkeys_possibly_useful;
	bool		index_is_ordered;
	bool		index_only_scan;
	int			indexcol;

	/* Check that index supports the desired scan type(s) */
	switch (scantype)
	{
		case ST_INDEXSCAN:
			if (!index->amhasgettuple)
				return NIL;
			break;
		case ST_BITMAPSCAN:
			if (!index->amhasgetbitmap)
				return NIL;
			break;
		case ST_ANYSCAN:
			/* either or both are OK */
			break;
	}

	/*
	 * 1. Combine the per-column IndexClause lists into an overall list.
	 *
	 * In the resulting list, clauses are ordered by index key, so that the
	 * column numbers form a nondecreasing sequence.  (This order is depended
	 * on by btree and possibly other places.)  The list can be empty, if the
	 * index AM allows that.
	 *
	 * found_lower_saop_clause is set true if we accept a ScalarArrayOpExpr
	 * index clause for a non-first index column.  This prevents us from
	 * assuming that the scan result is ordered.  (Actually, the result is
	 * still ordered if there are equality constraints for all earlier
	 * columns, but it seems too expensive and non-modular for this code to be
	 * aware of that refinement.)
	 *
	 * We also build a Relids set showing which outer rels are required by the
	 * selected clauses.  Any lateral_relids are included in that, but not
	 * otherwise accounted for.
	 */
	index_clauses = NIL;
	found_lower_saop_clause = false;
	outer_relids = bms_copy(rel->lateral_relids);
	for (indexcol = 0; indexcol < index->nkeycolumns; indexcol++)
	{
		ListCell   *lc;

		foreach(lc, clauses->indexclauses[indexcol])
		{
			IndexClause *iclause = (IndexClause *) lfirst(lc);
			RestrictInfo *rinfo = iclause->rinfo;

			/* We might need to omit ScalarArrayOpExpr clauses */
			if (IsA(rinfo->clause, ScalarArrayOpExpr))
			{
				if (!index->amsearcharray)
				{
					if (skip_nonnative_saop)
					{
						/* Ignore because not supported by index */
						*skip_nonnative_saop = true;
						continue;
					}
					/* Caller had better intend this only for bitmap scan */
					Assert(scantype == ST_BITMAPSCAN);
				}
				if (indexcol > 0)
				{
					if (skip_lower_saop)
					{
						/* Caller doesn't want to lose index ordering */
						*skip_lower_saop = true;
						continue;
					}
					found_lower_saop_clause = true;
				}
			}

			/* OK to include this clause */
			index_clauses = lappend(index_clauses, iclause);
			outer_relids = bms_add_members(outer_relids,
										   rinfo->clause_relids);
		}

		/*
		 * If no clauses match the first index column, check for amoptionalkey
		 * restriction.  We can't generate a scan over an index with
		 * amoptionalkey = false unless there's at least one index clause.
		 * (When working on columns after the first, this test cannot fail. It
		 * is always okay for columns after the first to not have any
		 * clauses.)
		 */
		if (index_clauses == NIL && !index->amoptionalkey)
			return NIL;
	}

	/* We do not want the index's rel itself listed in outer_relids */
	outer_relids = bms_del_member(outer_relids, rel->relid);
	/* Enforce convention that outer_relids is exactly NULL if empty */
	if (bms_is_empty(outer_relids))
		outer_relids = NULL;

	/* Compute loop_count for cost estimation purposes */
	loop_count = get_loop_count(root, rel->relid, outer_relids);

	/*
	 * 2. Compute pathkeys describing index's ordering, if any, then see how
	 * many of them are actually useful for this query.  This is not relevant
	 * if we are only trying to build bitmap indexscans, nor if we have to
	 * assume the scan is unordered.
	 */
	pathkeys_possibly_useful = (scantype != ST_BITMAPSCAN &&
								!found_lower_saop_clause &&
								has_useful_pathkeys(root, rel));
	index_is_ordered = (index->sortopfamily != NULL);
	if (index_is_ordered && pathkeys_possibly_useful)
	{
		index_pathkeys = build_index_pathkeys(root, index,
											  ForwardScanDirection);
		useful_pathkeys = truncate_useless_pathkeys(root, rel,
													index_pathkeys);
		orderbyclauses = NIL;
		orderbyclausecols = NIL;
	}
	else if (index->amcanorderbyop && pathkeys_possibly_useful)
	{
		/* see if we can generate ordering operators for query_pathkeys */
		match_pathkeys_to_index(index, root->query_pathkeys,
								&orderbyclauses,
								&orderbyclausecols);
		if (orderbyclauses)
			useful_pathkeys = root->query_pathkeys;
		else
			useful_pathkeys = NIL;
	}
	else
	{
		useful_pathkeys = NIL;
		orderbyclauses = NIL;
		orderbyclausecols = NIL;
	}

	// 这里非常重要，检查是否可以进行index-only scan
	index_only_scan = (scantype != ST_BITMAPSCAN && check_index_only(rel, index));

	/*
	 * 4. Generate an indexscan path if there are relevant restriction clauses
	 * in the current clauses, OR the index ordering is potentially useful for
	 * later merging or final output ordering, OR the index has a useful
	 * predicate, OR an index-only scan is possible.
	 */
	if (index_clauses != NIL || useful_pathkeys != NIL || useful_predicate ||
		index_only_scan)
	{
		ipath = create_index_path(root, index,
								  index_clauses,
								  orderbyclauses,
								  orderbyclausecols,
								  useful_pathkeys,
								  index_is_ordered ?
								  ForwardScanDirection :
								  NoMovementScanDirection,
								  index_only_scan,
								  outer_relids,
								  loop_count,
								  false);
		result = lappend(result, ipath);

		/*
		 * If appropriate, consider parallel index scan.  We don't allow
		 * parallel index scan for bitmap index scans.
		 */
		if (index->amcanparallel &&
			rel->consider_parallel && outer_relids == NULL &&
			scantype != ST_BITMAPSCAN)
		{
			ipath = create_index_path(root, index,
									  index_clauses,
									  orderbyclauses,
									  orderbyclausecols,
									  useful_pathkeys,
									  index_is_ordered ?
									  ForwardScanDirection :
									  NoMovementScanDirection,
									  index_only_scan,
									  outer_relids,
									  loop_count,
									  true);

			/*
			 * if, after costing the path, we find that it's not worth using
			 * parallel workers, just free it.
			 */
			if (ipath->path.parallel_workers > 0)
				add_partial_path(rel, (Path *) ipath);
			else
				pfree(ipath);
		}
	}

	/*
	 * 5. If the index is ordered, a backwards scan might be interesting.
	 */
	if (index_is_ordered && pathkeys_possibly_useful)
	{
		index_pathkeys = build_index_pathkeys(root, index,
											  BackwardScanDirection);
		useful_pathkeys = truncate_useless_pathkeys(root, rel,
													index_pathkeys);
		if (useful_pathkeys != NIL)
		{
			ipath = create_index_path(root, index,
									  index_clauses,
									  NIL,
									  NIL,
									  useful_pathkeys,
									  BackwardScanDirection,
									  index_only_scan,
									  outer_relids,
									  loop_count,
									  false);
			result = lappend(result, ipath);

			/* If appropriate, consider parallel index scan */
			if (index->amcanparallel &&
				rel->consider_parallel && outer_relids == NULL &&
				scantype != ST_BITMAPSCAN)
			{
				ipath = create_index_path(root, index,
										  index_clauses,
										  NIL,
										  NIL,
										  useful_pathkeys,
										  BackwardScanDirection,
										  index_only_scan,
										  outer_relids,
										  loop_count,
										  true);

				/*
				 * if, after costing the path, we find that it's not worth
				 * using parallel workers, just free it.
				 */
				if (ipath->path.parallel_workers > 0)
					add_partial_path(rel, (Path *) ipath);
				else
					pfree(ipath);
			}
		}
	}

	return result;
}
```


#### 执行器
输入执行计划，输出最终结果。执行索引扫描。

##### 主流程

```c++
PortalStart
--> ExecutorStart
    --> InitPlan
        --> ExecInitIndexScan
PortalRun
--> ExecutorRun
    --> ExecutePlan
        --> ExecIndexScan
            --> ExecScan
                --> ExecScanFetch
                    --> IndexNext
						--> index_beginscan  
							--> index_beginscan_internal  
								--> btbeginscan	//start a scan on a btree index
							--> table_index_fetch_begin
								--> heapam_index_fetch_begin
						--> index_getnext_slot
								// 获取TID
							--> index_getnext_tid(scan, direction) 
								--> btgettuple(scan, direction)
								// 获取TID对应的HeapTuple
							--> index_fetch_heap(scan, slot)
								--> table_index_fetch_tuple
									--> heapam_index_fetch_tuple
										--> ReleaseAndReadBuffer(hscan->xs_cbuf, hscan->xs_base.rel, ItemPointerGetBlockNumber(tid));
										--> heap_hot_search_buffer
PortalDrop
```

具体实现代码如下：
```c++
/* ----------------------------------------------------------------
 *		ExecIndexScan(node)
 * ----------------------------------------------------------------
 */
static TupleTableSlot *
ExecIndexScan(PlanState *pstate)
{
	IndexScanState *node = castNode(IndexScanState, pstate);

	/*
	 * If we have runtime keys and they've not already been set up, do it now.
	 */
	if (node->iss_NumRuntimeKeys != 0 && !node->iss_RuntimeKeysReady)
		ExecReScan((PlanState *) node);

	if (node->iss_NumOrderByKeys > 0)
		return ExecScan(&node->ss,
						(ExecScanAccessMtd) IndexNextWithReorder,
						(ExecScanRecheckMtd) IndexRecheck);
	else
		return ExecScan(&node->ss,
						(ExecScanAccessMtd) IndexNext,
						(ExecScanRecheckMtd) IndexRecheck);
}

/*
 * ExecScanFetch -- check interrupts & fetch next potential tuple
 *
 * This routine is concerned with substituting a test tuple if we are
 * inside an EvalPlanQual recheck.  If we aren't, just execute
 * the access method's next-tuple routine.
 */
static inline TupleTableSlot *ExecScanFetch(ScanState *node, ExecScanAccessMtd accessMtd, ExecScanRecheckMtd recheckMtd)
{
	EState	   *estate = node->ps.state;

	if (estate->es_epq_active != NULL)
	{
		// ...
	}

	/* Run the node-type-specific access method function to get the next tuple */
	return (*accessMtd) (node);
}

/* ----------------------------------------------------------------
 *		IndexNext
 *
 *		Retrieve a tuple from the IndexScan node's currentRelation
 *		using the index specified in the IndexScanState information.
 * ---------------------------------------------------------------- */
static TupleTableSlot *IndexNext(IndexScanState *node)
{
	EState	   *estate;
	ExprContext *econtext;
	ScanDirection direction;
	IndexScanDesc scandesc;
	TupleTableSlot *slot;

	/* extract necessary information from index scan node */
	estate = node->ss.ps.state;
	direction = estate->es_direction;
	/* flip direction if this is an overall backward scan */
	if (ScanDirectionIsBackward(((IndexScan *) node->ss.ps.plan)->indexorderdir))
	{
		if (ScanDirectionIsForward(direction))
			direction = BackwardScanDirection;
		else if (ScanDirectionIsBackward(direction))
			direction = ForwardScanDirection;
	}
	scandesc = node->iss_ScanDesc;
	econtext = node->ss.ps.ps_ExprContext;
	slot = node->ss.ss_ScanTupleSlot;

	if (scandesc == NULL)
	{
		/*
		 * We reach here if the index scan is not parallel, or if we're
		 * serially executing an index scan that was planned to be parallel.
		 */
		scandesc = index_beginscan(node->ss.ss_currentRelation,
								   node->iss_RelationDesc,
								   estate->es_snapshot,
								   node->iss_NumScanKeys,
								   node->iss_NumOrderByKeys);

		node->iss_ScanDesc = scandesc;

		/*
		 * If no run-time keys to calculate or they are ready, go ahead and
		 * pass the scankeys to the index AM.
		 */
		if (node->iss_NumRuntimeKeys == 0 || node->iss_RuntimeKeysReady)
			index_rescan(scandesc,
						 node->iss_ScanKeys, node->iss_NumScanKeys,
						 node->iss_OrderByKeys, node->iss_NumOrderByKeys);
	}

	/* ok, now that we have what we need, fetch the next tuple. */
	while (index_getnext_slot(scandesc, direction, slot))
	{
		CHECK_FOR_INTERRUPTS();

		/* If the index was lossy, we have to recheck the index quals using the fetched tuple. */
		if (scandesc->xs_recheck)
		{
			econtext->ecxt_scantuple = slot;
			if (!ExecQualAndReset(node->indexqualorig, econtext))
			{
				/* Fails recheck, so drop it and loop back for another */
				InstrCountFiltered2(node, 1);
				continue;
			}
		}

		return slot;
	}

	/* if we get here it means the index scan failed so we are at the end of the scan.. */
	node->iss_ReachedEnd = true;
	return ExecClearTuple(slot);
}
```


##### 索引扫描
索引与表一样，单独存储，同样是被切分成很多页，索引扫描，先访问索引（访问索引所在页，在索引页中继续找，找到匹配的键，获取被查找的表的元组的TID，再根据该TID去访问表的某页，根据页偏移找到指定元组返回）

```c++
/* Descriptor for heap table scans.*/
typedef struct HeapScanDescData
{
	TableScanDescData rs_base;	/* AM independent part of the descriptor */

	/* state set up at initscan time */
	BlockNumber rs_nblocks;		/* total number of blocks in rel */
	BlockNumber rs_startblock;	/* block # to start at */
	BlockNumber rs_numblocks;	/* max number of blocks to scan */
	/* rs_numblocks is usually InvalidBlockNumber, meaning "scan whole rel" */

	/* scan current state */
	bool		rs_inited;		/* false = scan not init'd yet */
	BlockNumber rs_cblock;		/* current block # in scan, if any */
	Buffer		rs_cbuf;		/* current buffer in scan, if any */
	/* NB: if rs_cbuf is not InvalidBuffer, we hold a pin on that buffer */

	/* rs_numblocks is usually InvalidBlockNumber, meaning "scan whole rel" */
	BufferAccessStrategy rs_strategy;	/* access strategy for reads */

	HeapTupleData rs_ctup;		/* current tuple in scan, if any */

	/* these fields only used in page-at-a-time mode and for bitmap scans */
	int			rs_cindex;		/* current tuple's index in vistuples */
	int			rs_ntuples;		/* number of visible tuples on page */
	OffsetNumber rs_vistuples[MaxHeapTuplesPerPage];	/* their offsets */
}			HeapScanDescData;
typedef struct HeapScanDescData *HeapScanDesc;
```

更详细的代码要阅读`nbtree.c`中的实现。
```c++
/* btgettuple() -- Get the next tuple in the scan. */
bool btgettuple(IndexScanDesc scan, ScanDirection dir)
{
	BTScanOpaque so = (BTScanOpaque) scan->opaque;
	bool		res;

	/* btree indexes are never lossy */
	scan->xs_recheck = false;

	/*
	 * If we have any array keys, initialize them during first call for a
	 * scan.  We can't do this in btrescan because we don't know the scan
	 * direction at that time.
	 */
	if (so->numArrayKeys && !BTScanPosIsValid(so->currPos))
	{
		/* punt if we have any unsatisfiable array keys */
		if (so->numArrayKeys < 0)
			return false;

		_bt_start_array_keys(scan, dir);
	}

	/* This loop handles advancing to the next array elements, if any */
	do
	{
		/*
		 * If we've already initialized this scan, we can just advance it in
		 * the appropriate direction.  If we haven't done so yet, we call
		 * _bt_first() to get the first item in the scan.
		 */
		if (!BTScanPosIsValid(so->currPos))
			res = _bt_first(scan, dir);
		else
		{
			/*
			 * Check to see if we should kill the previously-fetched tuple.
			 */
			if (scan->kill_prior_tuple)
			{
				/*
				 * Yes, remember it for later. (We'll deal with all such
				 * tuples at once right before leaving the index page.)  The
				 * test for numKilled overrun is not just paranoia: if the
				 * caller reverses direction in the indexscan then the same
				 * item might get entered multiple times. It's not worth
				 * trying to optimize that, so we don't detect it, but instead
				 * just forget any excess entries.
				 */
				if (so->killedItems == NULL)
					so->killedItems = (int *)
						palloc(MaxTIDsPerBTreePage * sizeof(int));
				if (so->numKilled < MaxTIDsPerBTreePage)
					so->killedItems[so->numKilled++] = so->currPos.itemIndex;
			}

			/*
			 * Now continue the scan.
			 */
			res = _bt_next(scan, dir);
		}

		/* If we have a tuple, return it ... */
		if (res)
			break;
		/* ... otherwise see if we have more array keys to deal with */
	} while (so->numArrayKeys && _bt_advance_array_keys(scan, dir));

	return res;
}
```