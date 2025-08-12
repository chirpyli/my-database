## PolarDB存储性能优化——预读
PolarDB为云原生数据库，底层使用PolarFS文件系统。PolarFS文件系统不同于单机文件系统，其对数据量较小的I/O操作并不是特别高效。这就需要我们针对文件系统的特点对上层数据库做一些针对性的优化或者设计。


### 为什么会有堆表预读
在PostgreSQL读取堆表的过程中，会以8kB页为单位通过文件系统读取页面至缓存Buffer中。我们以对表t1进行全表扫描为例进行说明。在开始扫描前当前Buffer中没有表t1的相关缓存页。
```sql
postgres=# select * from pg_buffercache where relfilenode=16384;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinn
ing_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+-----
-------------
(0 rows)
```
读取第一个元组需要读第一个页到Buffer中，此时，Buffer中会有表t1的缓存页1个。
```sql
postgres=# select * from pg_buffercache where relfilenode=16384;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinn
ing_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+-----
-------------
      308 |       16384 |          1663 |       13010 |             0 |              0 | f       |          1 |     
           1
(1 row)
```
当读取完第一个页面的所有元组后，需要读第二个页，从文件中读取页到Buffer中，此时，Buffer中会有表t1的缓存页2个。
```sql
postgres=# select * from pg_buffercache where relfilenode=16384;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinn
ing_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+-----
-------------
      308 |       16384 |          1663 |       13010 |             0 |              0 | f       |          1 |     
           1
      309 |       16384 |          1663 |       13010 |             0 |              1 | f       |          1 |     
           1
(2 rows)
```
也就是说PostgreSQL中，每次读取一个页都需要从文件系统读取一个页，而PolarFS对这种数据量较小的I/O操作并不是特别高效，对此PolarDB为了适配PolarFS而设计了堆表批量预读。当读取的页面数量超过一定阈值时，会出发批量预读，一次I/O读取128kB数据至Buffer中，对顺序扫描以及Vacuum等场景可以带来显著的性能提升。

> 当然在PostgreSQL中对于读取堆表的性能问题，也有相关的优化设计，比如并行扫描、同步扫描、以及超过Buffer 1/4大小后开启环形缓冲区批量读取等优化。

### 堆表预读实现原理
堆表预读的实现步骤主要分为以下四步：

1. 在Buffer Pool中申请N个Buffer。
2. 通过palloc在内存中申请一段大小为`N * 页大小`的空间，简称为p。
3. 通过PolarFS批量读取堆表中`N * 页大小`的数据拷贝至p中。
4. 将p中N个页的内容逐个拷贝至从Buffer Pool申请的N个Buffer中。

后续的读取操作会直接命中Buffer。数据流图如下所示：
![image](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/8935568661/p520516.png)


### 堆表预读参数
堆表预读的参数名为polar_bulk_read_size，功能默认开启，默认大小为128 KB，也就是16个页大小。
```sql
postgres=# show polar_bulk_read_size ;
 polar_bulk_read_size 
----------------------
 128kB
(1 row)
```
如果要关闭堆表预读功能，只需要将参数设置为0即可。
```sql
ALTER SYSTEM SET polar_bulk_read_size = 0;
```
我们验证一下堆表预读，先创建一个表t1,插入6000条数据，查看一下有27页。 
```sql
postgres=# select oid,relname,relfilenode,relpages,reltuples,relallvisible from pg_class where relname='t1';
  oid  | relname | relfilenode | relpages | reltuples | relallvisible 
-------+---------+-------------+----------+-----------+---------------
 16388 | t1      |       16388 |       27 |      6000 |            27
(1 row)
```
我们通过调试源码，通过pg_buffercache看一下它是否一次性读取16个数据页到Buffer中。我们重启数据库，清空表t1的Buffer。
```sql
postgres=# select * from pg_buffercache where relfilenode=16388;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinn
ing_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+-----
-------------
(0 rows)

```
然后读取表t1的数据，通过pg_buffercache查看Buffer信息。第一个页不进行预读，所以Buffer中只有表t1的缓存页1个。
```sql
postgres=# select * from pg_buffercache where relfilenode=16388;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
      221 |       16388 |          1663 |           5 |             0 |              0 | f       |          1 |                1
(1 row)
```  
当读到第2个页时触发预读，一次性读取16个页，加上之前读取的第一个页，Buffer中会有表t1的缓存页17个。
```sql
postgres=# select * from pg_buffercache where relfilenode=16388;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
      221 |       16388 |          1663 |           5 |             0 |              0 | f       |          1 |                1
      222 |       16388 |          1663 |           5 |             0 |              1 | f       |          1 |                1
      223 |       16388 |          1663 |           5 |             0 |              2 | f       |          1 |                0
      224 |       16388 |          1663 |           5 |             0 |              3 | f       |          1 |                0
      225 |       16388 |          1663 |           5 |             0 |              4 | f       |          1 |                0
      226 |       16388 |          1663 |           5 |             0 |              5 | f       |          1 |                0
      227 |       16388 |          1663 |           5 |             0 |              6 | f       |          1 |                0
      228 |       16388 |          1663 |           5 |             0 |              7 | f       |          1 |                0
      229 |       16388 |          1663 |           5 |             0 |              8 | f       |          1 |                0
      230 |       16388 |          1663 |           5 |             0 |              9 | f       |          1 |                0
      231 |       16388 |          1663 |           5 |             0 |             10 | f       |          1 |                0
      232 |       16388 |          1663 |           5 |             0 |             11 | f       |          1 |                0
      233 |       16388 |          1663 |           5 |             0 |             12 | f       |          1 |                0
      234 |       16388 |          1663 |           5 |             0 |             13 | f       |          1 |                0
      235 |       16388 |          1663 |           5 |             0 |             14 | f       |          1 |                0
      236 |       16388 |          1663 |           5 |             0 |             15 | f       |          1 |                0
      237 |       16388 |          1663 |           5 |             0 |             16 | f       |          1 |                0
(17 rows)
```


### 源码分析

堆表预读实现的细节点，这些细节点本质上是从性能以及正确性上考虑的：
1. 只支持顺序扫描。
2. 每次不一定会预读16个数据页，如果要读取的页不足16个，则预读实际要读取的页数或者至多读到命中Buffer的页为止（不会超过16个），并且一次预读的页数不会跨文件去读，因为超过1G的表会划分为多个段文件进行存储，如果要预读的页要跨文件，那么不进行跨文件读取，近读取到当前段文件最后一个页即止。
3. 如果只需要读1个页，则无需进行预读。
4. 如果读的第一个页已在Buffer中，则无需进行预读。
5. 预读的页数不会超过参数`polar_bulk_read_size`所指定的值，默认为16个页，但是最大只能设置为64，即不超过512kB。
6. 第一个页的读取方式与普通读取方式相同，不会触发预读。主要是为了优化普通读取的场景，避免不必要的预读。

我们先看一下关键路径调用栈：
```c++
(gdb) s
// 这里的底层是通过PolarFS读文件，读去blockCount个页，然后放到buffer中
polar_mdbulkread (reln=0x646569d08860, forknum=MAIN_FORKNUM, blocknum=1, blockCount=16, buffer=0x646569d66000 "") at md.c:1566
1566		int			amount = blockCount * BLCKSZ;
(gdb) bt
#0  polar_mdbulkread (reln=0x646569d08860, forknum=MAIN_FORKNUM, blocknum=1, blockCount=16, 
    buffer=0x646569d66000 "") at md.c:1566  // 一次性读取16个页
#1  0x00006465678fa20a in polar_smgrbulkread (reln=0x646569d08860, forknum=MAIN_FORKNUM, blocknum=1, 
    blockCount=16, buffer=0x646569d66000 "") at smgr.c:617
#2  0x00006465678a2ca8 in polar_bulk_read_buffer_common (reln=0x7e6754f948c8, relpersistence=112 'p', 
    forkNum=MAIN_FORKNUM, firstBlockNum=1, mode=RBM_NORMAL, strategy=0x0, hit=0x7ffef7cbf8a3, 
    maxBlockCount=16) at polar_bufmgr.c:1432
#3  0x00006465678a2534 in polar_bulk_read_buffer_extended (reln=0x7e6754f948c8, forkNum=MAIN_FORKNUM, 
    blockNum=1, mode=RBM_NORMAL, strategy=0x0, maxBlockCount=26)
    at polar_bufmgr.c:1158 // 这里开始进行预读的逻辑
#4  0x00006465673599a1 in heapgetpage (sscan=0x646569d62a80, page=1) at heapam.c:426 //这之前的流程和PostgreSQL是一样的
#5  0x000064656735bf40 in heapgettup_pagemode (scan=0x646569d62a80, dir=ForwardScanDirection, nkeys=0, 
    key=0x0) at heapam.c:1161
#6  0x000064656735c6c9 in heap_getnextslot (sscan=0x646569d62a80, direction=ForwardScanDirection, 
    slot=0x646569d623e0) at heapam.c:1393
#7  0x00006465676b7e23 in table_scan_getnextslot (sscan=0x646569d62a80, direction=ForwardScanDirection, 
    slot=0x646569d623e0) at ../../../src/include/access/tableam.h:1046
#8  0x00006465676b7eef in SeqNext (node=0x646569d621d8) at nodeSeqscan.c:80
#9  0x00006465676734d0 in ExecScanFetch (node=0x646569d621d8, accessMtd=0x6465676b7e57 <SeqNext>, 
    recheckMtd=0x6465676b7f00 <SeqRecheck>) at execScan.c:132
#10 0x0000646567673549 in ExecScan (node=0x646569d621d8, accessMtd=0x6465676b7e57 <SeqNext>, 
    recheckMtd=0x6465676b7f00 <SeqRecheck>) at execScan.c:181
#11 0x00006465676b7f59 in ExecSeqScan (pstate=0x646569d621d8) at nodeSeqscan.c:112
#12 0x000064656766202a in ExecProcNode (node=0x646569d621d8) at ../../../src/include/executor/executor.h:262
#13 0x0000646567664fa6 in ExecutePlan (estate=0x646569d61fa0, planstate=0x646569d621d8, 
    use_parallel_mode=false, operation=CMD_SELECT, sendTuples=true, numberTuples=0, 
    direction=ForwardScanDirection, dest=0x646569c56b68, execute_once=true) at execMain.c:1645
--Type <RET> for more, q to quit, c to continue without paging--
#14 0x000064656766277e in standard_ExecutorRun (queryDesc=0x646569c87c80, direction=ForwardScanDirection, 
    count=0, execute_once=true) at execMain.c:366
#15 0x0000646567662569 in ExecutorRun (queryDesc=0x646569c87c80, direction=ForwardScanDirection, count=0, 
    execute_once=true) at execMain.c:310
#16 0x00006465679095c4 in PortalRunSelect (portal=0x646569cf3b80, forward=true, count=0, 
    dest=0x646569c56b68) at pquery.c:935
#17 0x00006465679091c7 in PortalRun (portal=0x646569cf3b80, count=9223372036854775807, isTopLevel=true, 
    run_once=true, dest=0x646569c56b68, altdest=0x646569c56b68, qc=0x7ffef7cbff20) at pquery.c:776
#18 0x0000646567900c0c in exec_simple_query (query_string=0x646569c55000 "select * from t1;")
    at postgres.c:1302
#19 0x0000646567906c4b in PostgresMain (dbname=0x646569c91e58 "postgres", 
    username=0x646569c91e30 "postgres") at postgres.c:5033
#20 0x000064656780b636 in BackendRun (port=0x646569c8bbf0) at postmaster.c:4940
#21 0x000064656780ad6f in BackendStartup (port=0x646569c8bbf0) at postmaster.c:4657
#22 0x0000646567805bdc in ServerLoop () at postmaster.c:1897
#23 0x0000646567805328 in PostmasterMain (argc=3, argv=0x646569bc0ba0) at postmaster.c:1562
#24 0x00006465676f7108 in main (argc=3, argv=0x646569bc0ba0) at main.c:202
```
关键函数`polar_bulk_read_buffer_common`实现一次读取多个页面到Buffer中
```c++
/*
 * polar_bulk_read_buffer_common -- common logic for bulk read multi buffer one time.
 *
 * Try to read as many buffers as possible, up to maxBlockCount.
 * If first block hits shared_buffers, return immediately. 如果第一个页面在Buffer中，则直接返回
 * If first block doesn't hit shared_buffer,  如果第一个页面不在Buffer中，则读取maxBlockCount个页面到Buffer中或者至多到命中Buffer为止
 * read buffers until up to maxBlockCount or
 * up to first block which hit shared_buffer.
 *
 * Returns: the buffer number for the first block blockNum.
 *		The returned buffer has been pinned.
 *		Does not return on error --- elog's instead.
 *
 * *hit is set to true if the first blockNum was satisfied from shared buffer cache.
 *
 * Deadlock avoidance：Only ascending bulk read are allowed
 * to avoid dead lock on io_in_progress_lock.
 */
static Buffer
polar_bulk_read_buffer_common(Relation reln, char relpersistence, ForkNumber forkNum,
							  BlockNumber firstBlockNum, ReadBufferMode mode,
							  BufferAccessStrategy strategy, bool *hit,
							  BlockNumber maxBlockCount)
{
	SMgrRelation smgr = reln->rd_smgr;
	BufferDesc *bufHdr;
	BufferDesc *first_buf_hdr;
	Block		bufBlock;
	bool		found;
	bool		isLocalBuf = SmgrIsTemp(smgr);
	int			actual_bulk_io_count;
	int			index;
	char	   *buf_read;

	/* POLAR: start lsn to do replay */
	XLogRecPtr	checkpoint_redo_lsn = InvalidXLogRecPtr;
	XLogRecPtr	replay_from;
	polar_redo_action redo_action;
	uint32		repeat_read_times = 0;
	bool	   *checksum_fail;

	polar_pgstat_count_bulk_read_calls(reln);

	maxBlockCount = Min(maxBlockCount, polar_bulk_read_size); // 不超过polar_bulk_read_size

	if (firstBlockNum == P_NEW ||
		maxBlockCount <= 1 ||
		mode != RBM_NORMAL)  // 如果只读1个页面就不用预读了
	{
		return ReadBuffer_common(smgr, relpersistence, forkNum, firstBlockNum,
								 mode, strategy, hit);
	}

	Assert(!polar_bulk_io_is_in_progress);
	Assert(0 == polar_bulk_io_in_progress_count);

	// 预读处理逻辑开始
	polar_bulk_io_is_in_progress = true;  

    // 给预读的页面分配内存，分配的是TopMemoryContext，也即直到进程退出才释放
	if (NULL == polar_bulk_io_in_progress_buf)
	{
		Assert(NULL == polar_bulk_io_is_for_input);
		polar_bulk_io_in_progress_buf = MemoryContextAlloc(TopMemoryContext,
														   POLAR_MAX_BULK_IO_SIZE * sizeof(polar_bulk_io_in_progress_buf[0]));
		polar_bulk_io_is_for_input = MemoryContextAlloc(TopMemoryContext,
														POLAR_MAX_BULK_IO_SIZE * sizeof(polar_bulk_io_is_for_input[0]));
	}

	*hit = false;

	/* Make sure we will have room to remember the buffer pin */
	ResourceOwnerEnlargeBuffers(CurrentResourceOwner);

	TRACE_POSTGRESQL_BUFFER_READ_START(forkNum, firstBlockNum,
									   smgr->smgr_rnode.node.spcNode,
									   smgr->smgr_rnode.node.dbNode,
									   smgr->smgr_rnode.node.relNode,
									   smgr->smgr_rnode.backend,
									   false);	/* false == isExtend */

	if (isLocalBuf) 
	{
		bufHdr = LocalBufferAlloc(smgr, forkNum, firstBlockNum, &found);
		if (found)
			pgBufferUsage.local_blks_hit++;
		else
			pgBufferUsage.local_blks_read++;
	}
	else
	{
		/*
		 * lookup the buffer.  IO_IN_PROGRESS is set if the requested block is
		 * not currently in memory. If not found,
		 * polar_bulk_io_in_progress_count will be added by 1.
		 */
		bufHdr = BufferAlloc(smgr, relpersistence, forkNum, firstBlockNum,
							 strategy, &found);
		if (found)
			pgBufferUsage.shared_blks_hit++;
		else
			pgBufferUsage.shared_blks_read++;
	}
	/* At this point we do NOT hold any locks. */

	/* 如果Buffer命中，之间返回，中断预读逻辑 */
	if (found)
	{
		/* Just need to update stats before we exit */
		*hit = true;
		VacuumPageHit++;

		if (VacuumCostActive)
			VacuumCostBalance += VacuumCostPageHit;

		TRACE_POSTGRESQL_BUFFER_READ_DONE(forkNum, firstBlockNum,
										  smgr->smgr_rnode.node.spcNode,
										  smgr->smgr_rnode.node.dbNode,
										  smgr->smgr_rnode.node.relNode,
										  smgr->smgr_rnode.backend,
										  false,
										  found);

		Assert(0 == polar_bulk_io_in_progress_count);
		/* important, mark bulk_io end */
		polar_bulk_io_is_in_progress = false;

		return BufferDescriptorGetBuffer(bufHdr);
	}

	/*
	 * if we have gotten to this point, we have allocated a buffer for the
	 * page but its contents are not yet valid.  IO_IN_PROGRESS is set for it,
	 * if it's a shared buffer.
	 *
	 * Note: if smgrextend fails, we will end up with a buffer that is
	 * allocated but not marked BM_VALID.  P_NEW will still select the same
	 * block number (because the relation didn't get any longer on disk) and
	 * so future attempts to extend the relation will find the same buffer (if
	 * it's not been recycled) but come right back here to try smgrextend
	 * again.
	 */
	Assert(!(pg_atomic_read_u32(&bufHdr->state) & BM_VALID));	/* spinlock not needed */

	Assert(1 == polar_bulk_io_in_progress_count);
	Assert(bufHdr == polar_bulk_io_in_progress_buf[polar_bulk_io_in_progress_count - 1]);

	/*
	 * Hold the first block bufHdr, after TerminateBufferIO(),
	 * polar_bulk_io_in_progress_buf is freed.
	 */
	first_buf_hdr = bufHdr;

	/* 确保不会跨段文件读取，也就是说，只会读一个段文件 */
	maxBlockCount = Min(maxBlockCount, (BlockNumber) RELSEG_SIZE - (firstBlockNum % (BlockNumber) RELSEG_SIZE));

    // 分配多个buffer
	for (index = 1; index < maxBlockCount; index++)
	{
		BlockNumber blockNum = firstBlockNum + index;

		/* Make sure we will have room to remember the buffer pin */
		ResourceOwnerEnlargeBuffers(CurrentResourceOwner);

		/*
		 * lookup the buffer.  IO_IN_PROGRESS is set if the requested block is
		 * not currently in memory.
		 *
		 * If not found, polar_bulk_io_in_progress_count will be added by 1 by StartBufferIO().
		 */
		if (isLocalBuf)
			bufHdr = LocalBufferAlloc(smgr, forkNum, blockNum, &found);
		else
			bufHdr = BufferAlloc(smgr, relpersistence, forkNum, blockNum, strategy, &found);

		/*
		 * For extra block, don't update pgBufferUsage.shared_blks_hit or
		 * pgBufferUsage.shared_blks_read, also the blocks are not used now.
		 */
		/* bufHdr == NULL, all buffers are pinned. */
		if (found || bufHdr == NULL)
		{
			/* important: this buffer is the upper boundary, it should be excluded. */
			if (bufHdr != NULL)
				ReleaseBuffer(BufferDescriptorGetBuffer(bufHdr));
			break;
		}
		Assert(!(pg_atomic_read_u32(&bufHdr->state) & BM_VALID));	/* spinlock not needed */
	}

	Assert(index == polar_bulk_io_in_progress_count);

	/*
	 * Until now, as to {blockNum + [0, polar_bulk_io_in_progress_count)}
	 * block buffers, IO_IN_PROGRESS flag is set and io_in_progress_lock is
	 * holded.
	 *
	 * Other proc(include backend sql exec、start xlog replay) which read
	 * there buffers, would be blocked on io_in_progress_lock.
	 *
	 * Deadlock avoidance：Only ascending bulk read are allowed to avoid dead
	 * lock on io_in_progress_lock.
	 */

	/*
	 * polar_bulk_io_in_progress_count will be reduced by TerminateBufferIO(),
	 * For safety, its copy actual_bulk_io_count is used.
	 */
	actual_bulk_io_count = polar_bulk_io_in_progress_count;

	/* for eliminating palloc and memcpy */
	if (1 == actual_bulk_io_count)
		buf_read = isLocalBuf ?
			(char *) LocalBufHdrGetBlock(first_buf_hdr) :
			(char *) BufHdrGetBlock(first_buf_hdr);
	else
		buf_read = (char *) palloc_io_aligned(actual_bulk_io_count * BLCKSZ, MCXT_ALLOC_ZERO);

	checksum_fail = (bool *) palloc0(actual_bulk_io_count * sizeof(bool));

repeat_read:

	/*
	 * POLAR: page-replay.
	 *
	 * Get consistent lsn which used by replay.
	 *
	 * Note: All modifications about reply-page must be applied to both
	 * polar_bulk_read_buffer_common() and ReadBuffer_common().
	 */
	redo_action = polar_require_backend_redo(isLocalBuf, mode, forkNum, &replay_from);
	if (redo_action != POLAR_REDO_NO_ACTION)
	{
		checkpoint_redo_lsn = GetRedoRecPtr();
		Assert(!XLogRecPtrIsInvalid(checkpoint_redo_lsn));
	}
	/* POLAR end */

	/*
	 * Read in the page, unless the caller intends to overwrite it and just
	 * wants us to allocate a buffer.
	 */
	{
		instr_time	io_start,
					io_time;

		if (track_io_timing)
			INSTR_TIME_SET_CURRENT(io_start);

        // 调用smgr接口，从存储中读取数据
		polar_smgrbulkread(smgr, forkNum, firstBlockNum, actual_bulk_io_count, buf_read);

		polar_pgstat_count_bulk_read_calls_IO(reln);
		polar_pgstat_count_bulk_read_blocks_IO(reln, actual_bulk_io_count);

		if (track_io_timing)
		{
			INSTR_TIME_SET_CURRENT(io_time);
			INSTR_TIME_SUBTRACT(io_time, io_start);
			pgstat_count_buffer_read_time(INSTR_TIME_GET_MICROSEC(io_time));
			INSTR_TIME_ADD(pgBufferUsage.blk_read_time, io_time);
		}

		for (index = 0; index < actual_bulk_io_count; index++)
		{
			BlockNumber blockNum = firstBlockNum + index;

			bufBlock = (Block) (buf_read + index * BLCKSZ);
			/* check for garbage data */
			if (!PageIsVerifiedExtended((Page) bufBlock, blockNum,
										PIV_LOG_WARNING | PIV_REPORT_STAT))
			{
				polar_checksum_err_action err_act;

				err_act = polar_handle_read_error_block(bufBlock, smgr, forkNum, blockNum,
														mode, redo_action, &repeat_read_times);

				if (err_act == POLAR_CHECKSUM_ERR_REPEAT_READ)
					goto repeat_read;
				else if (err_act == POLAR_CHECKSUM_ERR_CHECKPOINT_REDO)
				{
					/*
					 * We must set checksum_fail is true. During checksum_fail
					 * is true, we replay buffer from checkpoint.redo.
					 */
					checksum_fail[index] = true;

					ereport(WARNING,
							(errcode(ERRCODE_DATA_CORRUPTED),
							 errmsg("Replay from lastcheckpoint which is %X/%X",
									LSN_FORMAT_ARGS(checkpoint_redo_lsn))));
				}
			}
		}
	}

	/*
	 * notice: 1. buffers must be processed by TerminateBufferIO() from back
	 * to front. a) TerminateBufferIO() release
	 * polar_bulk_io_in_progress_buf[] in decrement order. b) For better
	 * performance, LWLockRelease() release io_in_progress_lock in decrement
	 * order. 2. polar_bulk_io_in_progress_count was reduced by
	 * TerminateBufferIO(). a) polar_bulk_io_in_progress_count must not be
	 * used here.
	 */
	for (index = actual_bulk_io_count - 1; index >= 0; index--)
	{
		BlockNumber blockNum = firstBlockNum + index;

		bufHdr = polar_bulk_io_in_progress_buf[index];
		bufBlock = isLocalBuf ? LocalBufHdrGetBlock(bufHdr) : BufHdrGetBlock(bufHdr);

		/* need copy page content from aligned_buf_read to block shared_buffer */
		if (actual_bulk_io_count != 1)
			memcpy((char *) bufBlock, buf_read + index * BLCKSZ, BLCKSZ);   // 读到共享缓冲区

		if (unlikely(polar_trace_logindex_messages <= DEBUG3))
		{
			BufferTag  *tag = &bufHdr->tag;
			Buffer		buf = BufferDescriptorGetBuffer(bufHdr);
			Page		page = BufferGetPage(buf);

			ereport(LOG, (errmsg("read buffer lsn=%lX, redo_act=%d buf=%d, " POLAR_LOG_BUFFER_TAG_FORMAT,
								 PageGetLSN(page),
								 redo_action,
								 buf,
								 POLAR_LOG_BUFFER_TAG(tag)),
						  errhidestmt(true),
						  errhidecontext(true)));
		}

		/*
		 * POLAR: page-replay.
		 *
		 * Apply logs to this old page when read from disk.
		 *
		 * Note: All modifications about replay-page must be applied to both
		 * polar_bulk_read_buffer_common() and ReadBuffer_common().
		 */
		if (redo_action == POLAR_REDO_REPLAY_XLOG)
		{
			XLogRecPtr	final_replay_from = checksum_fail[index] ? checkpoint_redo_lsn : replay_from;

			/*
			 * POLAR: we want to do record replay on page only in non-Startup
			 * process and consistency reached Startup process. This judge is
			 * *ONLY* valid in replica mode, so we set replica check above
			 * with polar_is_replica().
			 */
			polar_apply_io_locked_page(bufHdr, final_replay_from, checkpoint_redo_lsn, smgr, forkNum, blockNum);
		}
		else if (redo_action == POLAR_REDO_MARK_OUTDATE)
		{
			uint32		redo_state = polar_lock_redo_state(bufHdr);

			redo_state |= POLAR_REDO_OUTDATE;
			polar_unlock_redo_state(bufHdr, redo_state);
		}

		/* POLAR end */

		/*
		 * After TerminateBufferIO, polar_bulk_io_in_progress_buf[index] can
		 * not be used any more.
		 */
		/* Set BM_VALID, terminate IO, and wake up any waiters */
		if (isLocalBuf)
		{
			/* Only need to adjust flags */
			uint32		buf_state = pg_atomic_read_u32(&bufHdr->state);

			buf_state |= BM_VALID;
			pg_atomic_unlocked_write_u32(&bufHdr->state, buf_state);
			/* bulk io */
			polar_bulk_io_in_progress_count--;
		}
		else
			TerminateBufferIO(bufHdr, false, BM_VALID);


		/* important: buffers except firstBlockNum should release pin */
		if (index != 0)
			ReleaseBuffer(BufferDescriptorGetBuffer(bufHdr));

		VacuumPageMiss++;
		if (VacuumCostActive)
			VacuumCostBalance += VacuumCostPageMiss;
	}

	/*
	 * POLAR: page-replay.
	 *
	 * Reset consistent lsn which used by replay.
	 *
	 * Note: All modifications about replay-page must be applied to both
	 * polar_bulk_read_buffer_common() and ReadBuffer_common().
	 */
	if (redo_action != POLAR_REDO_NO_ACTION)
		POLAR_RESET_BACKEND_READ_MIN_LSN();

	/* POLAR end */

	TRACE_POSTGRESQL_BUFFER_READ_DONE(forkNum, firstBlockNum,
									  smgr->smgr_rnode.node.spcNode,
									  smgr->smgr_rnode.node.dbNode,
									  smgr->smgr_rnode.node.relNode,
									  smgr->smgr_rnode.backend,
									  false,
									  found);

	/*
	 * notice: polar_bulk_io_in_progress_count was reduced by
	 * TerminateBufferIO(). polar_bulk_io_in_progress_count must not be used
	 * here.
	 */
	if (actual_bulk_io_count != 1)
		pfree(buf_read);

	pfree(checksum_fail);

	Assert(0 == polar_bulk_io_in_progress_count);
	/* important, mark bulk_io end */
	polar_bulk_io_is_in_progress = false;

	return BufferDescriptorGetBuffer(first_buf_hdr);
}
```
具体实现一次性读取多个页在函数`polar_mdbulkread`中，该函数会调用`FileRead`函数从文件中读取数据到指定的缓冲区中。而其底层实现是通过文件系统来读取文件的。在PolarDB中为PolarFS文件系统。
```c++
/*
 *  POLAR: bulk read
 *
 *	polar_mdbulkread() -- Read the specified continuous blocks from a relation.
 *
 *  Caller must ensure that the blockcount does not exceed the length of the relation file.
 */
void polar_mdbulkread(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, int blockCount, char *buffer)
{
	off_t		seekpos;
	int			nbytes;
	MdfdVec    *v;
	int			amount = blockCount * BLCKSZ;

	AssertPointerAlignment(buffer, POLAR_BUFFER_ALIGN_LEN);

	TRACE_POSTGRESQL_SMGR_MD_READ_START(forknum, blocknum,
										reln->smgr_rnode.node.spcNode,
										reln->smgr_rnode.node.dbNode,
										reln->smgr_rnode.node.relNode,
										reln->smgr_rnode.backend);

	v = _mdfd_getseg(reln, forknum, blocknum, false,
					 EXTENSION_FAIL | EXTENSION_CREATE_RECOVERY);

	seekpos = (off_t) BLCKSZ * (blocknum % ((BlockNumber) RELSEG_SIZE));

	Assert(seekpos < (off_t) BLCKSZ * RELSEG_SIZE);
	Assert(seekpos + (off_t) amount <= (off_t) BLCKSZ * RELSEG_SIZE);

    // 一次性读取多个块
	nbytes = FileRead(v->mdfd_vfd, buffer, amount, seekpos, WAIT_EVENT_DATA_FILE_READ);

	TRACE_POSTGRESQL_SMGR_MD_READ_DONE(forknum, blocknum,
									   reln->smgr_rnode.node.spcNode,
									   reln->smgr_rnode.node.dbNode,
									   reln->smgr_rnode.node.relNode,
									   reln->smgr_rnode.backend,
									   nbytes,
									   amount);

	if (nbytes != amount)
	{
		if (nbytes < 0)
			ereport(ERROR,
					(errcode_for_file_access(),
					 errmsg("could not read block %u in file \"%s\": %m",
							blocknum, FilePathName(v->mdfd_vfd))));

		/*
		 * Short read: we are at or past EOF, or we read a partial block at
		 * EOF.  Normally this is an error; upper levels should never try to
		 * read a nonexistent block.  However, if zero_damaged_pages is ON or
		 * we are InRecovery, we should instead return zeroes without
		 * complaining.  This allows, for example, the case of trying to
		 * update a block that was later truncated away.
		 */
		if (zero_damaged_pages || InRecovery || POLAR_IN_LOGINDEX_PARALLEL_REPLAY())
		{
			/* only zero damaged_pages */
			int			damaged_pages_start_offset = nbytes - nbytes % BLCKSZ;

			MemSet((char *) buffer + damaged_pages_start_offset, 0, amount - damaged_pages_start_offset);
		}
		else
			ereport(ERROR,
					(errcode(ERRCODE_DATA_CORRUPTED),
					 errmsg("could not bulk read block %u in file \"%s\": read only %d of %d bytes",
							blocknum, FilePathName(v->mdfd_vfd),
							nbytes, amount)));
	}
}
```

---
> 这里额外说一下为什么堆表预读不建议设置的很大，其实原因和PostgreSQL中环形缓冲区大小默认为256KB道理是一样的。顺序扫描使用256KB的环形缓冲区，它足够小，因而能放入L2缓存中，从而使得操作系统缓存到共享缓冲区的页面传输变得高效。
