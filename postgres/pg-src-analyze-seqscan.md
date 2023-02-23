### PostgreSQL源码分析——SeqScan
这里，我们分析一下顺序扫描的过程，以最简单的`select * from t1;`语句为例，分析一下查询的过程。

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
			into_clause from_clause where_clause    // from t1;
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

table_ref:	relation_expr opt_alias_clause
				{
					$1->alias = $2;
					$$ = (Node *) $1;
				}
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

/*
 * RangeVar - range variable, used in FROM clauses
 *
 * Also used to represent table names in utility statements; there, the alias
 * field is not used, and inh tells whether to apply the operation
 * recursively to child tables.  In some contexts it is also useful to carry
 * a TEMP table indication here.
 */
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
--> pg_rewrite_query
```

#### 查询优化

主要是生成执行计划，前面已经通过`EXPLAIN`命令得知其执行计划为顺序扫描，让我们具体看一下其是如何生成的。
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
                        --> make_one_rel	// Finds all possible access paths for executing a query
                            --> set_base_rel_sizes
                                --> set_rel_size
                                    --> set_plain_rel_size
                                        --> set_baserel_size_estimates	// 选择率计算，计算代价Cost要用
                            --> set_base_rel_pathlists
                                --> set_rel_pathlist // Build access paths for a base relation
                                    --> set_plain_rel_pathlist
                                        --> create_seqscan_path // 生成顺序扫描路径
                            --> make_rel_from_joinlist
                    --> apply_scanjoin_target_to_paths
            --> create_plan
                --> create_scan_plan
                    --> create_seqscan_pla
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

	/* Consider sequential scan */
	add_path(rel, create_seqscan_path(root, rel, required_outer, 0));	// 顺序扫描

	/* If appropriate, consider parallel sequential scan */
	if (rel->consider_parallel && required_outer == NULL)		// 尝试并行顺序扫描
		create_plain_partial_paths(root, rel);

	/* Consider index scans */
	create_index_paths(root, rel);			// 索引扫描

	/* Consider TID scans */
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

#### 执行器
输入执行计划，输出最终结果。执行顺序扫描。

##### 主流程

```c++
PortalStart
--> ExecutorStart
    --> InitPlan
        --> ExecInitSeqScan
PortalRun
--> ExecutorRun
    --> ExecutePlan
        --> ExecSeqScan 
            --> ExecScan
                --> ExecScanFetch
                    --> SeqNext 
						--> table_beginscan  // 表扫描函数scan_begin
							--> heap_beginscan  // 堆表beginscan
								--> initscan
						--> table_scan_getnextslot
							--> heap_getnextslot
								--> heapgettup_pagemode
									--> heapgetpage
										--> ReadBuffer_common
								--> ExecStoreBufferHeapTuple
									--> tts_buffer_heap_store_tuple
PortalDrop
```

具体实现代码如下：
```c++
/* ----------------------------------------------------------------
 *		ExecSeqScan(node)
 *
 *		Scans the relation sequentially and returns the next qualifying
 *		tuple.
 *		We call the ExecScan() routine and pass it the appropriate
 *		access method functions.
 * ---------------------------------------------------------------- */
static TupleTableSlot *
ExecSeqScan(PlanState *pstate)
{
	SeqScanState *node = castNode(SeqScanState, pstate);

	return ExecScan(&node->ss,
					(ExecScanAccessMtd) SeqNext,
					(ExecScanRecheckMtd) SeqRecheck);
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
 *		SeqNext
 *
 *		This is a workhorse for ExecSeqScan
 * ----------------------------------------------------------------*/
static TupleTableSlot *SeqNext(SeqScanState *node)
{
	TableScanDesc scandesc;
	EState	   *estate;
	ScanDirection direction;
	TupleTableSlot *slot;

	/* get information from the estate and scan state */
	scandesc = node->ss.ss_currentScanDesc;
	estate = node->ss.ps.state;
	direction = estate->es_direction;
	slot = node->ss.ss_ScanTupleSlot;

	if (scandesc == NULL)
	{
		/* We reach here if the scan is not parallel, or if we're serially
		 * executing a scan that was planned to be parallel. */
		scandesc = table_beginscan(node->ss.ss_currentRelation, estate->es_snapshot, 0, NULL);
		node->ss.ss_currentScanDesc = scandesc;
	}

	/* get the next tuple from the table */
	if (table_scan_getnextslot(scandesc, direction, slot))
		return slot;
	return NULL;
}

/*
 * Return next tuple from `scan`, store in slot.
 */
static inline bool
table_scan_getnextslot(TableScanDesc sscan, ScanDirection direction, TupleTableSlot *slot)
{
	slot->tts_tableOid = RelationGetRelid(sscan->rs_rd);
	return sscan->rs_rd->rd_tableam->scan_getnextslot(sscan, direction, slot);	// heap_getnextslot
}
```


##### 堆表扫描
我们知道在PG中，表的存储是被切成很多页（也可以叫区块）的，默认是8k的大小（引申问题：为什么是8k呢？这个页的大小涉及到性能，涉及到很多方面，比如VM，FSM，页缓存，磁盘读取等）。全表扫描就是从表的第0块开始，一直扫描到最后一块。扫描的时候，先指定要扫描的块号，查找块是否在缓冲区中，如果不在缓冲区中，则从磁盘读取该表指定的块到缓存区中。块扫描完成后，加1，继续扫描下一个块，直到扫描结束。如果表很大，缓冲区不能完全缓冲，则会依据时钟扫描算法，选择淘汰掉某些块。


```c++
/*
 * Descriptor for heap table scans.
 */
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

更详细的代码要阅读`heapam.c`中的实现。
```c++
/*
 * IDENTIFICATION
 *	  src/backend/access/heap/heapam.c
 *
 * INTERFACE ROUTINES
 *		heap_beginscan	- begin relation scan
 *		heap_rescan		- restart a relation scan
 *		heap_endscan	- end relation scan
 *		heap_getnext	- retrieve next tuple in scan
 *		heap_fetch		- retrieve tuple with given tid
 *		heap_insert		- insert tuple into a relation
 *		heap_multi_insert - insert multiple tuples into a relation
 *		heap_delete		- delete a tuple from a relation
 *		heap_update		- replace a tuple in a relation with another tuple
 */
```

```c++
bool
heap_getnextslot(TableScanDesc sscan, ScanDirection direction, TupleTableSlot *slot)
{
	HeapScanDesc scan = (HeapScanDesc) sscan;

	/* Note: no locking manipulations needed */

	if (sscan->rs_flags & SO_ALLOW_PAGEMODE)
		heapgettup_pagemode(scan, direction, sscan->rs_nkeys, sscan->rs_key);
	else
		heapgettup(scan, direction, sscan->rs_nkeys, sscan->rs_key);

	if (scan->rs_ctup.t_data == NULL)
	{
		ExecClearTuple(slot);
		return false;
	}

	/*
	 * if we get here it means we have a new current scan tuple, so point to
	 * the proper return buffer and return the tuple.
	 */

	pgstat_count_heap_getnext(scan->rs_base.rs_rd);

	ExecStoreBufferHeapTuple(&scan->rs_ctup, slot,
							 scan->rs_cbuf);
	return true;
}
```