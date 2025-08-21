
## PostgreSQL同步扫描

### 为什么需要同步扫描？
当多个后端并发访问顺序扫描一张大表时，可能会出现该表的数据页被反复读取到缓冲区中，极端情况下，每个后端都重复的将表从第0页读取到最后一页，所产生的IO代价是`表大小*后端数量`。IO的代价是非常大的，我们需要减少多个并发顺序扫描对同一张大表时的重复IO。

怎么优化这种情况呢？理想的情况应该是尽量复用进度较快的后端读取到缓冲区的页，不再重复从磁盘冲读取数据页到缓冲区，这就需要后端顺序扫描进行同步，使它们读取页块尽量对齐，这个解决方案就是同步扫描。

> 对于大表，同步扫描的效率会更高，因此只有超过一定阈值的表才会启动同步扫描。

### 同步扫描设计原理
当多个后端backend对同一个表进行顺序扫描时，我们尝试让它们保持同步，以减少整体所需I/O。目标是:每个数据页只需被读取到共享缓冲区一次，并让所有参与该同步扫描的后端在该页面被移出缓冲区前处理完。

在一组执行顺序顺序扫描的backend进程中，leader进程需要等待I/O完成，而其他followers进程则无需等待。需要做的是让一个新启动的顺序扫描的backend进程从其他backend进程当前读取的位置附近开始扫描。
我们以循环方式扫描表，leader进程从第0页开始扫描，直到结束，其他followers进程从第X个页开始扫描，扫描到表的结束页，然后再从第0页开始扫描到第X-1页。这样可以确保访问所有行，同时能参与到同步扫描过程。

为了实现这一点，我们会记录每张表的扫描位置，并让新的扫描从之前扫描所处的位置附近开始。我们并不要求在初始对齐后继续强制同步，因为某些查询可能处理的比其他backend进程要慢（例如，结果需要通过慢速网络传输给客户端），我们不希望这类慢查询拖慢其他的查询。实际上，同时进行大规模顺序扫描的往往只涉及少数几张表，我们只需要要将这些扫描位置维护在一个小型的LRU中，每次需要查找或更新扫描位置时，遍历LRU即可。该机制仅对超过一定大小的表启用。


接口函数：
- ss_get_location    返回某关系当前的扫描位置
- ss_report_location  更新某关系当前的扫描位置，间隔`SYNC_SCAN_REPORT_INTERVAL`(如果页大小为8kB，则间隔16)个页更新一次。

上面提到的开启同步扫描的表大小的阈值是多少呢？与开启`BAS_BULKREAD`的阈值相同，即超过`shared_buffers/4`。
```c++
    // 必须不是临时表，并且表的大小必须大于1/4的shared_buffers大小大小
	if (!RelationUsesLocalBuffers(scan->rs_base.rs_rd) &&
		scan->rs_nblocks > NBuffers / 4)
	{
		allow_strat = (scan->rs_base.rs_flags & SO_ALLOW_STRAT) != 0;
		allow_sync = (scan->rs_base.rs_flags & SO_ALLOW_SYNC) != 0;   // 允许同步扫描
	}
```


#### 在扫描开始前设置扫描初始页号

表扫描初始化：
```c++
table_beginscan
--> heap_beginscan
    --> initscan
```
initscan具体实现如下：
```c++
// 扫描初始化
static void initscan(HeapScanDesc scan, ScanKey key, bool keep_startblock)
{
	ParallelBlockTableScanDesc bpscan = NULL;
	bool		allow_strat;
	bool		allow_sync;

    // 获取扫描的页数
	if (scan->rs_base.rs_parallel != NULL)
	{
		bpscan = (ParallelBlockTableScanDesc) scan->rs_base.rs_parallel;  // 并行扫描
		scan->rs_nblocks = bpscan->phs_nblocks;
	}
	else
		scan->rs_nblocks = RelationGetNumberOfBlocks(scan->rs_base.rs_rd); // 计算表的页数

    // 如果表超过shared_buffers/4，则允许bulk-read访问策略以及同步扫描
	if (!RelationUsesLocalBuffers(scan->rs_base.rs_rd) &&
		scan->rs_nblocks > NBuffers / 4)
	{
		allow_strat = (scan->rs_base.rs_flags & SO_ALLOW_STRAT) != 0;
		allow_sync = (scan->rs_base.rs_flags & SO_ALLOW_SYNC) != 0;
	}
	else
		allow_strat = allow_sync = false;

	if (allow_strat)
	{
		/* During a rescan, keep the previous strategy object. */
		if (scan->rs_strategy == NULL)
			scan->rs_strategy = GetAccessStrategy(BAS_BULKREAD);
	}
	else
	{
		if (scan->rs_strategy != NULL)
			FreeAccessStrategy(scan->rs_strategy);
		scan->rs_strategy = NULL;
	}

	if (scan->rs_base.rs_parallel != NULL)
	{
		/* For parallel scan, believe whatever ParallelTableScanDesc says. */
		if (scan->rs_base.rs_parallel->phs_syncscan)
			scan->rs_base.rs_flags |= SO_ALLOW_SYNC;
		else
			scan->rs_base.rs_flags &= ~SO_ALLOW_SYNC;
	}
	else if (keep_startblock)
	{
		/*
		 * When rescanning, we want to keep the previous startblock setting,
		 * so that rewinding a cursor doesn't generate surprising results.
		 * Reset the active syncscan setting, though.
		 */
		if (allow_sync && synchronize_seqscans)
			scan->rs_base.rs_flags |= SO_ALLOW_SYNC;
		else
			scan->rs_base.rs_flags &= ~SO_ALLOW_SYNC;
	}
	else if (allow_sync && synchronize_seqscans) // 允许同步扫描
	{
		scan->rs_base.rs_flags |= SO_ALLOW_SYNC;  // 设置同步扫描标识位
        // 设置rs_startblock起始扫描的页号
		scan->rs_startblock = ss_get_location(scan->rs_base.rs_rd, scan->rs_nblocks); 
	}
	else
	{
		scan->rs_base.rs_flags &= ~SO_ALLOW_SYNC; // 取消同步扫描标识位
		scan->rs_startblock = 0;    // 起始扫描页号设为0
	}

	scan->rs_numblocks = InvalidBlockNumber;  // 扫描全部页

    // ...
}
```

#### 从报告位置开始扫描后，扫描到尾部再从0页扫描

对于同步扫描，首先从`scan->rs_startblock`开始扫描，扫描到`scan->rs_nblocks`（也就是表的最后一页）后，再从第0页开始扫描，扫描到`scan->rs_startblock`位置结束扫描。
```c++
static void heapgettup_pagemode(HeapScanDesc scan, ScanDirection dir, int nkeys, ScanKey key)
{
	if (ScanDirectionIsForward(dir))
	{
		if (!scan->rs_inited)  // 如果没有初始化，进行初始设置
		{
            // 空表，直接返回
			if (scan->rs_nblocks == 0 || scan->rs_numblocks == 0)
			{
				tuple->t_data = NULL;
				return;
			}
			if (scan->rs_base.rs_parallel != NULL)
			{
                // 并行扫描
            }
            else
                page = scan->rs_startblock; /* 起始扫描页 */
            // 获取页
            heapgetpage((TableScanDesc) scan, page);
			lineindex = 0;
			scan->rs_inited = true;
        }
        else
        {
            /* continue from previously returned page/tuple */
			page = scan->rs_cblock; /* current page */
			lineindex = scan->rs_cindex + 1;
        }
		dp = BufferGetPage(scan->rs_cbuf);
		TestForOldSnapshot(scan->rs_base.rs_snapshot, scan->rs_base.rs_rd, dp);
		lines = scan->rs_ntuples;
		/* page and lineindex now reference the next visible tid */

		linesleft = lines - lineindex;
    }

    for (;;)
    {
		while (linesleft > 0)
		{
			lineoff = scan->rs_vistuples[lineindex];
			lpp = PageGetItemId(dp, lineoff);
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
				scan->rs_cindex = lineindex;
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

        if (backward)
            // 反向扫描...
        else if (scan->rs_base.rs_parallel != NULL)
            // 并行扫描...
        else
        {
			page++;
			if (page >= scan->rs_nblocks)  // 如果已经扫描到表最后一页
				page = 0;   // 将page设置为0
			finished = (page == scan->rs_startblock) ||
				(scan->rs_numblocks != InvalidBlockNumber ? --scan->rs_numblocks == 0 : false);  // 判断是否全部扫描完毕，page已经等于起始页

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
            // 如果是同步扫描，则报告当前扫描的位置
            // 在反向扫描时不会进行报告，因为那样会扰乱其他正在前向扫描的过程
            // 同步扫描不保证所有扫描完全同步，尽力而为
			if (scan->rs_base.rs_flags & SO_ALLOW_SYNC)
				ss_report_location(scan->rs_base.rs_rd, page);            
        }

    }

}
```

#### 报告扫描位置间隔
配置参数`synchronize_seqscans`，是否开启同步扫描，默认开启。
```sql
postgres=# show synchronize_seqscans;
 synchronize_seqscans 
----------------------
 on
(1 row)
```


> 调试方法：configure中增加编译选项`CFLAGS="-DTRACE_SYNCSCAN"`。


报告当前扫描位置的间隔大小，16个页，这个值必须小于环形缓冲区的大小，不然的话，待同步的扫描的页已经不在环形缓冲区了，在大表扫描时，如果表大小超过SharedBuffer的四分之一，那么就会开启环形缓冲区。
```c++
#define SYNC_SCAN_REPORT_INTERVAL (128 * 1024 / BLCKSZ)    // 16
```
同步扫描的位置信息：
```c++
typedef struct ss_scan_location_t
{
	RelFileNode relfilenode;	/*  那张表 */
	BlockNumber location;		/* 最新报告的扫描的位置 */
} ss_scan_location_t;
```

报告间隔：
```c++
void ss_report_location(Relation rel, BlockNumber location)
{
	/*
	 * To reduce lock contention, only report scan progress every N pages. For
	 * the same reason, don't block if the lock isn't immediately available.
	 * Missing a few updates isn't critical, it just means that a new scan
	 * that wants to join the pack will start a little bit behind the head of
	 * the scan.  Hopefully the pages are still in OS cache and the scan
	 * catches up quickly.
	 */
	if ((location % SYNC_SCAN_REPORT_INTERVAL) == 0)
	{
		if (LWLockConditionalAcquire(SyncScanLock, LW_EXCLUSIVE))
		{
			(void) ss_search(rel->rd_node, location, true);
			LWLockRelease(SyncScanLock);
		}
	}
}
```