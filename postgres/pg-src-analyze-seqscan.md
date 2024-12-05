### PostgreSQL源码分析——顺序扫描
理解顺序扫描对于理解数据库的运行是非常重要的，对于一款数据库来说，顺序扫描是必须要实现的最小功能集。这里，我们分析一下顺序扫描的过程，以最简单的`select * from t1;`语句为例，通过分析全表扫描来分析顺序扫描的过程。

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

/* RangeVar - range variable, used in FROM clauses */
typedef struct RangeVar
{
	NodeTag		type;
	char	   *catalogname;	/* the catalog (database) name, or NULL */
	char	   *schemaname;		/* the schema name, or NULL */
	char	   *relname;		/* the relation/sequence name */
	bool		inh;			/* expand rel by inheritance? recursively act on children? */
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
`SelectStmt`经过语义分析后转为`Query`表示。
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
                    --> create_seqscan_plan
```
这里有个函数要重点说明一下，在涉及多表路径规划时，单表路径是最基础的，单表路径可以是顺序扫描、索引扫描、TID扫描等，这个是最底层的。
```c++
// 表路径规划
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
typedef struct Scan
{
	Plan		plan;
	Index		scanrelid;	// 指明扫描那个表，
} Scan;

typedef Scan SeqScan;
```

#### 执行器
输入执行计划，输出最终结果。执行顺序扫描。这里先不考虑并行顺序扫描的情况，并行顺序扫描后续再进行分析。我们在继续分析顺序扫描前，先自己想一下实现顺序扫描都需要做哪些工作，需要注意哪些问题？首先，我们需要知道要扫描的那个表的OID，我们知道表是以堆表的形式存储的，我们需要根据表OID打开堆表文件，表中的元组是存在堆表的页中，每个页都有页头以及数据部分，我们需要遍历表中的页，然后读取每个页中所有的元组。还需要注意的问题是事务问题，因为我们在读取页的过程中，可能还有其他事务在插入、删除、更新元组，所以我们扫描的过程中需要注意可见性问题。

##### 主流程

```c++
PortalStart
--> ExecutorStart
    --> InitPlan
		--> ExecCheckRTPerms(rangeTable, true); // 检查权限
        --> ExecInitSeqScan
			--> ExecOpenScanRelation(estate, node->scanrelid, eflags); // 打开表
				--> ExecGetRangeTableRelation(estate, scanrelid);
					--> table_open(rte->relid, NoLock);	// 打开指定OID的表
						--> relation_open(relationId, lockmode);	// 先查relcache缓存，缓存没有就通过访问pg_class,pg_attribute系统表构造RelationData，并插入缓存
PortalRun
--> ExecutorRun
    --> ExecutePlan
        --> ExecSeqScan   // 顺序扫描算子
            --> ExecScan
                --> ExecScanFetch
                    --> SeqNext 
						// 在正式扫描前，要判断是否进行并行扫描，扫描的总页数是多少，扫描的起点页是哪里
						--> table_beginscan     // 表扫描函数scan_begin
							--> heap_beginscan  // 堆表beginscan
								--> initscan
									// 计算表的总块数，如何计算，后续有详细分析
									--> RelationGetNumberOfBlocks
										--> smgrnblocks
											--> mdnblocks // 磁盘文件获取总块数
						--> table_scan_getnextslot
							--> heap_getnextslot
								--> heapgettup_pagemode
									// 1. 判断表的总块数，如果为0，则返回NULL
									// 2. 是否进行并行处理，这里我们先不分析并行的情况
									// 3. 从buffer中读取指定的页，如果buffer中没有，则从磁盘读取，并插入buffer
									// 4. 判断页中的元组的可见性 scan->rs_vistuples
									--> heapgetpage
										--> scan->rs_cbuf = ReadBufferExtended
											--> ReadBuffer_common	// 从磁盘读取页，并插入buffer，这部分的内容在之前的文章已分析，这里不再展开细述
										--> scan->rs_cblock = page;
										--> buffer = scan->rs_cbuf;
										--> dp = BufferGetPage(buffer);
										--> // 可见性判断
										--> scan->rs_ntuples = ntup; // 页中可见的元组个数
									
								// 从缓冲区中读取元组，存储在Slot中		
								--> ExecStoreBufferHeapTuple
									--> tts_buffer_heap_store_tuple
PortalDrop
```

具体实现代码如下：
```c++
static TupleTableSlot *ExecSeqScan(PlanState *pstate)
{
	SeqScanState *node = castNode(SeqScanState, pstate);

	return ExecScan(&node->ss,
					(ExecScanAccessMtd) SeqNext,
					(ExecScanRecheckMtd) SeqRecheck);
}

/* ExecScanFetch -- check interrupts & fetch next potential tuple */
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

// 具体执行顺序扫描
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

	// 从表中获取下一个元组
	if (table_scan_getnextslot(scandesc, direction, slot))
		return slot;
	return NULL;
}

/* Return next tuple from `scan`, store in slot. */
static inline bool
table_scan_getnextslot(TableScanDesc sscan, ScanDirection direction, TupleTableSlot *slot)
{
	slot->tts_tableOid = RelationGetRelid(sscan->rs_rd);
	return sscan->rs_rd->rd_tableam->scan_getnextslot(sscan, direction, slot);	// heap_getnextslot
}
```

##### 怎么计算表的总块数
这里分析一下怎么获取堆表的页数，也就是总块数。计算的核心方法是堆表文件大小/BLOCKSZ，每个页大小默认是8k，表文件每超过1G则增加一个段文件，每个段文件大小默认是1G，所以计算堆表总页数需要考虑段文件数量。具体实现如下：
```c++
// 获取堆表总页数
BlockNumber mdnblocks(SMgrRelation reln, ForkNumber forknum)
{
	MdfdVec    *v;
	BlockNumber nblocks;
	BlockNumber segno;

	mdopenfork(reln, forknum, EXTENSION_FAIL);

	// ...

	segno = reln->md_num_open_segs[forknum] - 1;
	v = &reln->md_seg_fds[forknum][segno];

	for (;;)
	{
		nblocks = _mdnblocks(reln, forknum, v); // 获取当前段文件的总页数
		if (nblocks > ((BlockNumber) RELSEG_SIZE))
			elog(FATAL, "segment too big");
		if (nblocks < ((BlockNumber) RELSEG_SIZE)) // 如果当前段文件总页数小于段文件大小，则说明是最后一个段文件，直接返回
			return (segno * ((BlockNumber) RELSEG_SIZE)) + nblocks;

		segno++; // 如果当前段文件总页数等于段文件大小，则说明还有下一个段文件，继续计算下一个段文件的总页数

		v = _mdfd_openseg(reln, forknum, segno, 0);
		if (v == NULL)
			return segno * ((BlockNumber) RELSEG_SIZE);
	}
}

// 获取当前段文件的总页数，计算方法是文件大小/BLOCKSZ
static BlockNumber
_mdnblocks(SMgrRelation reln, ForkNumber forknum, MdfdVec *seg)
{
	off_t		len;

	len = FileSize(seg->mdfd_vfd);
	if (len < 0)
		ereport(ERROR,
				(errcode_for_file_access(),
				 errmsg("could not seek to end of file \"%s\": %m",
						FilePathName(seg->mdfd_vfd))));
	/* note that this calculation will ignore any partial block at EOF */
	return (BlockNumber) (len / BLCKSZ);
}
```

##### 堆表扫描
我们知道在PG中，表的存储是被切成很多页（也可以叫区块）的，默认是8k的大小（引申问题：为什么是8k呢？这个页的大小涉及到性能，涉及到很多方面，比如VM，FSM，页缓存，磁盘读取等）。全表扫描就是从表的第0块开始，一直扫描到最后一块。扫描的时候，先指定要扫描的块号，查找块是否在缓冲区中，如果不在缓冲区中，则从磁盘读取该表指定的块到缓存区中。块扫描完成后，加1，继续扫描下一个块，直到扫描结束。如果表很大，缓冲区不能完全缓冲，则会依据时钟扫描算法，选择淘汰掉某些块。
```c++
typedef struct HeapScanDescData
{
	TableScanDescData rs_base;	/* AM independent part of the descriptor */

	/* state set up at initscan time */
	BlockNumber rs_nblocks;		// 表中页总数
	BlockNumber rs_startblock;	// 扫描起始页
	BlockNumber rs_numblocks;	/* max number of blocks to scan */
	/* rs_numblocks is usually InvalidBlockNumber, meaning "scan whole rel" */

	/* scan current state */
	bool		rs_inited;		/* false = scan not init'd yet */
	BlockNumber rs_cblock;		/* 当前扫描的页号 */
	Buffer		rs_cbuf;		/* 当前扫描的页的Buffer */
	/* NB: if rs_cbuf is not InvalidBuffer, we hold a pin on that buffer */

	/* rs_numblocks is usually InvalidBlockNumber, meaning "scan whole rel" */
	BufferAccessStrategy rs_strategy;	/* access strategy for reads */

	HeapTupleData rs_ctup;		/* 当前扫描的元组 */

	/* these fields only used in page-at-a-time mode and for bitmap scans */
	int			rs_cindex;		/* 指明当前扫描的元祖在rs_vistuples数组中的位置 */
	int			rs_ntuples;		/* 页中有多少可见的元组 */
	OffsetNumber rs_vistuples[MaxHeapTuplesPerPage];	/* 所有可见元祖的offsets */
}			HeapScanDescData;
typedef struct HeapScanDescData *HeapScanDesc;
```
关键函数实现，`heap_getnextslot`在执行堆扫描的过程中，尝试从页面中读取可见元组，并将其存储在一个`TupleTableSlot`中。我们知道PG执行是一次一元组，每次只返回一个元组，再读取下一个元组时，其扫描过程的状态信息都在HeapScanDesc中，这样下次执行我就知道应该读取哪一个元组，所以每次扫描时，都需要将HeapScanDesc中的状态信息更新。
```c++
bool heap_getnextslot(TableScanDesc sscan, ScanDirection direction, TupleTableSlot *slot)
{
	HeapScanDesc scan = (HeapScanDesc) sscan;

	// 获取页，从页中读取元组，存在scan->re_ctup中
	if (sscan->rs_flags & SO_ALLOW_PAGEMODE)
		heapgettup_pagemode(scan, direction, sscan->rs_nkeys, sscan->rs_key);
	else
		heapgettup(scan, direction, sscan->rs_nkeys, sscan->rs_key);

	if (scan->rs_ctup.t_data == NULL)
	{
		ExecClearTuple(slot);
		return false;
	}

	// 将元组放入slot中
	ExecStoreBufferHeapTuple(&scan->rs_ctup, slot, scan->rs_cbuf);
	return true;
}

// 从页中读取元组，存在scan->rs_ctup中
// 为了提高性能，读取元组时，一次性读取页中所有可见的元组，存在scan->rs_vistuples中
static void heapgettup_pagemode(HeapScanDesc scan, ScanDirection dir, int nkeys, ScanKey key)
{
	HeapTuple	tuple = &(scan->rs_ctup);
	bool		backward = ScanDirectionIsBackward(dir);
	BlockNumber page;
	bool		finished;
	Page		dp;
	int			lines;
	int			lineindex;
	OffsetNumber lineoff;
	int			linesleft;
	ItemId		lpp;

	/* calculate next starting lineindex, given scan direction */
	if (ScanDirectionIsForward(dir))	// 正向扫描
	{
		if (!scan->rs_inited)
		{
			// 如果表是空的，则返回NULL
			if (scan->rs_nblocks == 0 || scan->rs_numblocks == 0)
			{
				tuple->t_data = NULL;
				return;
			}
			if (scan->rs_base.rs_parallel != NULL)	// 并行扫描
			{
				ParallelBlockTableScanDesc pbscan =
				(ParallelBlockTableScanDesc) scan->rs_base.rs_parallel;
				ParallelBlockTableScanWorker pbscanwork =
				scan->rs_parallelworkerdata;

				table_block_parallelscan_startblock_init(scan->rs_base.rs_rd,
														 pbscanwork, pbscan);

				page = table_block_parallelscan_nextpage(scan->rs_base.rs_rd,
														 pbscanwork, pbscan);

				/* Other processes might have already finished the scan. */
				if (page == InvalidBlockNumber)
				{
					Assert(!BufferIsValid(scan->rs_cbuf));
					tuple->t_data = NULL;
					return;
				}
			}
			else
				page = scan->rs_startblock; /* first page */
			heapgetpage((TableScanDesc) scan, page);	// 获取页
			lineindex = 0; // 从页的第一个元组开始扫描
			scan->rs_inited = true;
		}
		else
		{
			/* continue from previously returned page/tuple */
			page = scan->rs_cblock;   // 当前扫描的页
			lineindex = scan->rs_cindex + 1;   // 获取要读的下一个元组的位置
		}

		dp = BufferGetPage(scan->rs_cbuf);
		TestForOldSnapshot(scan->rs_base.rs_snapshot, scan->rs_base.rs_rd, dp);
		lines = scan->rs_ntuples;	// 页中可见的元组数量
		/* page and lineindex now reference the next visible tid */

		linesleft = lines - lineindex;		// 页中剩余的元组数量
	}
	else if (backward)		// 反向扫描
	{
		// 并行扫描不支持反向扫描
		Assert(scan->rs_base.rs_parallel == NULL);

		if (!scan->rs_inited)
		{
			if (scan->rs_nblocks == 0 || scan->rs_numblocks == 0)
			{
				Assert(!BufferIsValid(scan->rs_cbuf));
				tuple->t_data = NULL;
				return;
			}

			scan->rs_base.rs_flags &= ~SO_ALLOW_SYNC;

			/*
			 * Start from last page of the scan.  Ensure we take into account
			 * rs_numblocks if it's been adjusted by heap_setscanlimits().
			 */
			if (scan->rs_numblocks != InvalidBlockNumber)
				page = (scan->rs_startblock + scan->rs_numblocks - 1) % scan->rs_nblocks;
			else if (scan->rs_startblock > 0)
				page = scan->rs_startblock - 1;
			else
				page = scan->rs_nblocks - 1;
			heapgetpage((TableScanDesc) scan, page);
		}
		else
		{
			/* continue from previously returned page/tuple */
			page = scan->rs_cblock; /* current page */
		}

		dp = BufferGetPage(scan->rs_cbuf);
		TestForOldSnapshot(scan->rs_base.rs_snapshot, scan->rs_base.rs_rd, dp);
		lines = scan->rs_ntuples;	// 页中可见的元组数量

		if (!scan->rs_inited)
		{
			lineindex = lines - 1;	// 从页的最后一个元组开始扫描
			scan->rs_inited = true;
		}
		else
		{
			lineindex = scan->rs_cindex - 1; 	// 获取要读的下一个元组的位置，与正向扫描相反，正向是 + 1，反向是 - 1
		}
		/* page and lineindex now reference the previous visible tid */

		linesleft = lineindex + 1;
	}
	else
	{
		/*
		 * ``no movement'' scan direction: refetch prior tuple
		 */
		if (!scan->rs_inited)
		{
			Assert(!BufferIsValid(scan->rs_cbuf));
			tuple->t_data = NULL;
			return;
		}

		page = ItemPointerGetBlockNumber(&(tuple->t_self));
		if (page != scan->rs_cblock)
			heapgetpage((TableScanDesc) scan, page);

		/* Since the tuple was previously fetched, needn't lock page here */
		dp = BufferGetPage(scan->rs_cbuf);
		TestForOldSnapshot(scan->rs_base.rs_snapshot, scan->rs_base.rs_rd, dp);
		lineoff = ItemPointerGetOffsetNumber(&(tuple->t_self));
		lpp = PageGetItemId(dp, lineoff);
		Assert(ItemIdIsNormal(lpp));

		tuple->t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);
		tuple->t_len = ItemIdGetLength(lpp);

		/* check that rs_cindex is in sync */
		Assert(scan->rs_cindex < scan->rs_ntuples);
		Assert(lineoff == scan->rs_vistuples[scan->rs_cindex]);

		return;
	}

   // 开始页面扫描，循环遍历页中的元组
	for (;;)
	{
		while (linesleft > 0)	// 页中剩余的元组数量
		{
			lineoff = scan->rs_vistuples[lineindex];  // 获取元组在页中的位置
			lpp = PageGetItemId(dp, lineoff); 		  // 页面内部获取元组
			Assert(ItemIdIsNormal(lpp));

			tuple->t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);
			tuple->t_len = ItemIdGetLength(lpp);
			ItemPointerSet(&(tuple->t_self), page, lineoff);

			/*
			 * if current tuple qualifies, return it.
			 */
			if (key != NULL)
			{
				bool		valid;

				HeapKeyTest(tuple, RelationGetDescr(scan->rs_base.rs_rd),
							nkeys, key, valid);
				if (valid)
				{
					scan->rs_cindex = lineindex;
					return;
				}
			}
			else
			{
				scan->rs_cindex = lineindex;	// 记录当前元组的位置，更新扫描状态信息
				return;
			}

			/*
			 * otherwise move to the next item on the page
			 */
			--linesleft;
			if (backward)
				--lineindex;
			else
				++lineindex;
		}
		// 如果当前页读完了，则要读下一个页
		if (backward)
		{
			finished = (page == scan->rs_startblock) ||
				(scan->rs_numblocks != InvalidBlockNumber ? --scan->rs_numblocks == 0 : false);
			if (page == 0)
				page = scan->rs_nblocks;
			page--;	// 反向扫描，页号 - 1
		}
		else if (scan->rs_base.rs_parallel != NULL)
		{
			ParallelBlockTableScanDesc pbscan =
			(ParallelBlockTableScanDesc) scan->rs_base.rs_parallel;
			ParallelBlockTableScanWorker pbscanwork =
			scan->rs_parallelworkerdata;

			page = table_block_parallelscan_nextpage(scan->rs_base.rs_rd,
													 pbscanwork, pbscan);
			finished = (page == InvalidBlockNumber);
		}
		else
		{
			page++;		// 正向扫描，页号 + 1
			if (page >= scan->rs_nblocks)
				page = 0;
			finished = (page == scan->rs_startblock) ||
				(scan->rs_numblocks != InvalidBlockNumber ? --scan->rs_numblocks == 0 : false);

			/*
			 * Report our new scan position for synchronization purposes. We
			 * don't do that when moving backwards, however. That would just
			 * mess up any other forward-moving scanners.
			 *
			 * Note: we do this before checking for end of scan so that the
			 * final state of the position hint is back at the start of the
			 * rel.  That's not strictly necessary, but otherwise when you run
			 * the same query multiple times the starting position would shift
			 * a little bit backwards on every invocation, which is confusing.
			 * We don't guarantee any specific ordering in general, though.
			 */
			if (scan->rs_base.rs_flags & SO_ALLOW_SYNC)
				ss_report_location(scan->rs_base.rs_rd, page);
		}

		/*
		 * return NULL if we've exhausted all the pages
		 */
		if (finished)
		{
			if (BufferIsValid(scan->rs_cbuf))
				ReleaseBuffer(scan->rs_cbuf);
			scan->rs_cbuf = InvalidBuffer;
			scan->rs_cblock = InvalidBlockNumber;
			tuple->t_data = NULL;
			scan->rs_inited = false;
			return;
		}

		heapgetpage((TableScanDesc) scan, page);

		dp = BufferGetPage(scan->rs_cbuf);
		TestForOldSnapshot(scan->rs_base.rs_snapshot, scan->rs_base.rs_rd, dp);
		lines = scan->rs_ntuples;
		linesleft = lines;
		if (backward)
			lineindex = lines - 1;
		else
			lineindex = 0;
	}
}

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