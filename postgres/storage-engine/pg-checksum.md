### PostgreSQL源码分析——数据页校验
我们知道数据库是持久化存储到外存（磁盘）的，而存储介质可能损坏，数据通过IO传输时也有可能传输错误，虽然概率非常低，那么，数据库怎么应对这种情况呢？能否拥有一定的校验能力，检测出这些故障。对此有了数据库数据页校验的能力。
#### README
在PG的源码中关于校验的设计，可阅读[src/backend/storage/page/README](https://github.com/postgres/postgres/blob/master/src/backend/storage/page/README)。
这里摘录比较重要的描述： 校验数据页被设计为检测I/O系统故障。我们不会检测内存Buffer错误，因为它时一个非常低概率事件，可参考论文[http://www.cs.toronto.edu/~bianca/papers/sigmetrics09.pdf](https://www.cs.toronto.edu/~bianca/papers/sigmetrics09.pdf)中的描述。 对一个数据页的校验并不是所有的时候都有效。校验仅在数据页离开缓存share buffer以及从I/O中读取数据页时有效。我们在数据页从缓冲区刷入磁盘前设置checksum值，在读入缓冲区前进行校验。

这就意味着，当一个页的WAL日志改变时，页没有更新页checksum值，所以full page image并没有一个有效的checksum。当WAL回放时不进行checksum校验。

#### 开启数据校验
默认情况下PG是没有开启数据校验的，开启后会有性能损失，所以，这个功能并不是没有代价的，需要用户根据实际情况选择是否开启。可以查看只读配置变量`data_checksums`的值来验证集群中校验和的当前状态，通过执行`SHOW data_checksums`命令。
```sql
-- 默认是关闭的
postgres=# show data_checksums;
 data_checksums 
----------------
 off
(1 row)
```
如果要开启数据校验，则需要在initdb中指定`-k`或者`--data-checksums`。在数据页面上使用校验码来帮助检测I/O系统造成的损坏。启用校验码将会引起显著的性能惩罚。 如果设置，则为所有对象计算校验和，在整个数据库中。 
```shell
postgres@slpc:~/pg15.5$ ./bin/initdb -D mn -k
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with this locale configuration:
  provider:    libc
  LC_COLLATE:  en_US.UTF-8
  LC_CTYPE:    en_US.UTF-8
  LC_MESSAGES: en_US.UTF-8
  LC_MONETARY: zh_CN.UTF-8
  LC_NUMERIC:  zh_CN.UTF-8
  LC_TIME:     zh_CN.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.   # 页数据校验已开启

creating directory mn ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Asia/Shanghai
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
initdb: hint: You can change this by editing pg_hba.conf or using the option -A, or --auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    bin/pg_ctl -D mn -l logfile start

```
所有校验和失败都将报告在pg_stat_database视图。我们看一下`pg_stat_database`视图，如果校验失败，其中的`checksum_failures`、`checksum_last_failure`会有信息。
```sql
postgres=# select * from pg_stat_database where datname='postgres';
-[ RECORD 1 ]------------+-----------
datid                    | 5
datname                  | postgres
numbackends              | 1
xact_commit              | 10
xact_rollback            | 0
blks_read                | 258
blks_hit                 | 3920
tup_returned             | 1663
tup_fetched              | 1547
tup_inserted             | 129
tup_updated              | 18
tup_deleted              | 36
conflicts                | 0
temp_files               | 0
temp_bytes               | 0
deadlocks                | 0
checksum_failures        | 0        -- 在此数据库(或共享对象)中检测到的数据页校验码失败数，如果没有启用数据校验码则为NULL。
checksum_last_failure    |          -- 在此数据库(或共享对象)中检测到最后一个数据页校验码失败的时间，如果没有启用数据校验码则为NULL。
blk_read_time            | 0
blk_write_time           | 0
session_time             | 100412.449
active_time              | 49.166
idle_in_transaction_time | 0
sessions                 | 1
sessions_abandoned       | 0
sessions_fatal           | 0
sessions_killed          | 0
stats_reset              | 
```
默认情况下，数据页不被校验和保护，当开启后，每个数据页都包含一个校验和，该校验和在写页到磁盘时更新，在每次从磁盘读页时进行验证。如果验证失败，则会发生错误并终止当前正在执行的事务，该功能使得 PostgreSQL 自身拥有了检测 I/O 或硬件错误的能力。

```sql
postgres=# show data_checksums ;
 data_checksums 
----------------
 on
```
还有一个GUC参数`ignore_checksum_failure`，该参数若设置为 true，校验失败后不会产生错误，而是给客户端发送一个警告。当然，checksum 失败意味着磁盘上的数据已经损坏，忽略此类错误可能导致数据损坏扩散甚至导致系统奔溃，此时宜尽早修复，因此，若开启 checksum，该参数建议设置为 false。当试图从页面损坏中恢复时，可能需要旁路校验和保护，可临时设置配置参数`ignore_checksum_failure`。
```sql
postgres=# show ignore_checksum_failure ;
 ignore_checksum_failure 
-------------------------
 off
(1 row)
```

查看校验值：
```sql
postgres=# select * from page_header(get_raw_page('t1',0));
    lsn    | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
-----------+----------+-------+-------+-------+---------+----------+---------+-----------
 0/175E7E8 |   -28273 |     0 |    28 |  8160 |    8192 |     8192 |       4 |         0
(1 row)
```


还有一个工具，pg_checksums，在PostgreSQL数据库集簇中启用、禁用或检查数据校验和。后续在分析。


#### 源码分析

在页进行初始化的时候，无需进行设置checksum，直到写的时候。
```c++
/*
 * PageInit
 *		Initializes the contents of a page.
 *		Note that we don't calculate an initial checksum here; that's not done
 *		until it's time to write.
 */
void PageInit(Page page, Size pageSize, Size specialSize)
{
	PageHeader	p = (PageHeader) page;

	specialSize = MAXALIGN(specialSize);

	Assert(pageSize == BLCKSZ);
	Assert(pageSize > specialSize + SizeOfPageHeaderData);

	/* Make sure all fields of page are zero, as well as unused space */
	MemSet(p, 0, pageSize);

	p->pd_flags = 0;
	p->pd_lower = SizeOfPageHeaderData;
	p->pd_upper = pageSize - specialSize;
	p->pd_special = pageSize - specialSize;
	PageSetPageSizeAndVersion(page, pageSize, PG_PAGE_LAYOUT_VERSION);
	/* p->pd_prune_xid = InvalidTransactionId;		done by above MemSet */
}
```

##### 设置checksum值

页checksum在从Buffer刷盘到磁盘时进行设置，我们可看一下bgwriter进程刷盘的代码，在`FlushBuffer`函数中，`PageSetChecksumCopy((Page) bufBlock, buf->tag.blockNum);`即为设置checksun。数据页没有刷盘保存在Buffer Pool时是没必要做checksum的，因为其主要目的是检测I/O错误或者磁盘等硬件错误。设置校验的目的也是为了校验可靠性相对较差的外部存储等。

```c++
BackgroundWriterMain(void)  // bgwriter进程主流程
{
    //...
    if (sigsetjmp(local_sigjmp_buf, 1) != 0) // 错误处理
    {
        /* Close all open files after any error. */
		smgrcloseall();
    }

    for (;;)
    {
        BgBufferSync(&wb_context);  // 刷脏页落盘
        --> SyncOneBuffer(next_to_clean, true, wb_context);
            --> FlushBuffer(bufHdr, NULL);  // 具体脏页落盘的实现
                {
                    Block		bufBlock;
                    /* Find smgr relation for buffer */
                    if (reln == NULL)
                        reln = smgropen(buf->tag.rnode, InvalidBackendId);

	                bufBlock = BufHdrGetBlock(buf);

                    // 设置页checksum
                   /* Update page checksum if desired.  Since we have only shared lock on the
                    * buffer, other processes might be updating hint bits in it, so we must
                    * copy the page to private storage if we do checksumming. */
                    bufToWrite = PageSetChecksumCopy((Page) bufBlock, buf->tag.blockNum);

                    // ...
                    /* bufToWrite is either the shared buffer or a copy, as appropriate.*/
                    smgrwrite(reln, buf->tag.forkNum, buf->tag.blockNum, bufToWrite, false);
                    // ...
                }
    }
    // ...
}
```
我们看一下`PageSetChecksumCopy`函数实现：
```c++
/*
 * Set checksum for a page in shared buffers.
 *
 * If checksums are disabled, or if the page is not initialized, just return
 * the input.  Otherwise, we must make a copy of the page before calculating
 * the checksum, to prevent concurrent modifications (e.g. setting hint bits)
 * from making the final checksum invalid.  It doesn't matter if we include or
 * exclude hints during the copy, as long as we write a valid page and
 * associated checksum.
 *
 * Returns a pointer to the block-sized data that needs to be written. Uses
 * statically-allocated memory, so the caller must immediately write the
 * returned page and not refer to it again.
 */
char *
PageSetChecksumCopy(Page page, BlockNumber blkno)
{
	static char *pageCopy = NULL;

	/* If we don't need a checksum, just return the passed-in data */
	if (PageIsNew(page) || !DataChecksumsEnabled())
		return (char *) page;

	/*
	 * We allocate the copy space once and use it over on each subsequent
	 * call.  The point of palloc'ing here, rather than having a static char
	 * array, is first to ensure adequate alignment for the checksumming code
	 * and second to avoid wasting space in processes that never call this.
	 */
	if (pageCopy == NULL)
		pageCopy = MemoryContextAlloc(TopMemoryContext, BLCKSZ);
    // 在计算校验和前先copy数据页以防止并发修改对计算的影响
	memcpy(pageCopy, (char *) page, BLCKSZ);
    // 页pd_checksum值的计算
	((PageHeader) pageCopy)->pd_checksum = pg_checksum_page(pageCopy, blkno);
	return pageCopy;
}

/*
 * Compute the checksum for a Postgres page.
 *
 * The page must be adequately aligned (at least on a 4-byte boundary).
 * Beware also that the checksum field of the page is transiently zeroed.
 *
 * The checksum includes the block number (to detect the case where a page is
 * somehow moved to a different location), the page header (excluding the
 * checksum itself), and the page data.
 */
uint16 pg_checksum_page(char *page, BlockNumber blkno)
{
	PGChecksummablePage *cpage = (PGChecksummablePage *) page;
	uint16		save_checksum;
	uint32		checksum;

	/* We only calculate the checksum for properly-initialized pages */
	Assert(!PageIsNew(&cpage->phdr));

	/*
	 * Save pd_checksum and temporarily set it to zero, so that the checksum
	 * calculation isn't affected by the old checksum stored on the page.
	 * Restore it after, because actually updating the checksum is NOT part of
	 * the API of this function. */
	save_checksum = cpage->phdr.pd_checksum;
	cpage->phdr.pd_checksum = 0;
	checksum = pg_checksum_block(cpage);
	cpage->phdr.pd_checksum = save_checksum;

	/* Mix in the block number to detect transposed pages */
	checksum ^= blkno;

	/*
	 * Reduce to a uint16 (to fit in the pd_checksum field) with an offset of
	 * one. That avoids checksums of zero, which seems like a good idea. */
	return (uint16) ((checksum % 65535) + 1);  //pd_checksum为uint16，截取16位。
}


/*
 * Set checksum for a page in private memory.
 *
 * This must only be used when we know that no other process can be modifying
 * the page buffer.
 */
void
PageSetChecksumInplace(Page page, BlockNumber blkno)
{
	/* If we don't need a checksum, just return */
	if (PageIsNew(page) || !DataChecksumsEnabled())
		return;

	((PageHeader) page)->pd_checksum = pg_checksum_page((char *) page, blkno);
}
```

##### 校验checksum值
校验是在从磁盘读到Buffer时进行校验，我们先看一下从磁盘读一个数据页到Buffer的调用栈，通过调试`select * from t1`，我们跟踪源码：
```c++
mdread(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, char * buffer) (src\backend\storage\smgr\md.c:687)
smgrread(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, char * buffer) (src\backend\storage\smgr\smgr.c:536)
ReadBuffer_common(SMgrRelation smgr, char relpersistence, ForkNumber forkNum, BlockNumber blockNum, ReadBufferMode mode, BufferAccessStrategy strategy, _Bool * hit) (src\backend\storage\buffer\bufmgr.c:1021)
ReadBufferExtended(Relation reln, ForkNumber forkNum, BlockNumber blockNum, ReadBufferMode mode, BufferAccessStrategy strategy) (src\backend\storage\buffer\bufmgr.c:780)
heapgetpage(TableScanDesc sscan, BlockNumber page) (src\backend\access\heap\heapam.c:402)
heapgettup_pagemode(HeapScanDesc scan, ScanDirection dir, int nkeys, ScanKey key) (src\backend\access\heap\heapam.c:901)
heap_getnextslot(TableScanDesc sscan, ScanDirection direction, TupleTableSlot * slot) (src\backend\access\heap\heapam.c:1352)
table_scan_getnextslot(TableScanDesc sscan, ScanDirection direction, TupleTableSlot * slot) (src\include\access\tableam.h:1046)
SeqNext(SeqScanState * node) (src\backend\executor\nodeSeqscan.c:80)
ExecScanFetch(ScanState * node, ExecScanAccessMtd accessMtd, ExecScanRecheckMtd recheckMtd) (src\backend\executor\execScan.c:132)
ExecScan(ScanState * node, ExecScanAccessMtd accessMtd, ExecScanRecheckMtd recheckMtd) (src\backend\executor\execScan.c:181)
ExecSeqScan(PlanState * pstate) (src\backend\executor\nodeSeqscan.c:112)
ExecProcNodeFirst(PlanState * node) (src\backend\executor\execProcnode.c:464)
ExecProcNode(PlanState * node) (src\include\executor\executor.h:262)
ExecutePlan(EState * estate, PlanState * planstate, _Bool use_parallel_mode, CmdType operation, _Bool sendTuples, uint64 numberTuples, ScanDirection direction, DestReceiver * dest, _Bool execute_once) (src\backend\executor\execMain.c:1636)
standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\execMain.c:363)
ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\execMain.c:307)
PortalRunSelect(Portal portal, _Bool forward, long count, DestReceiver * dest) (src\backend\tcop\pquery.c:924)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:768)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1250)
PostgresMain(const char * dbname, const char * username) (src\backend\tcop\postgres.c:4598)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4514)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4242)
ServerLoop() (src\backend\postmaster\postmaster.c:1809)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1481)
main(int argc, char ** argv) (src\backend\main\main.c:202)
```
我们看一下校验相关的代码：
```c++
/*
 * ReadBuffer_common -- common logic for all ReadBuffer variants
 *
 * *hit is set to true if the request was satisfied from shared buffer cache.
 */
static Buffer ReadBuffer_common(SMgrRelation smgr, char relpersistence, ForkNumber forkNum,
				  BlockNumber blockNum, ReadBufferMode mode,
				  BufferAccessStrategy strategy, bool *hit)
{
	BufferDesc *bufHdr;
	Block		bufBlock;

    // ...
	if (isExtend) {
		/* new buffers are zero-filled */
		MemSet((char *) bufBlock, 0, BLCKSZ);
		/* don't set checksum for all-zero page */
		smgrextend(smgr, forkNum, blockNum, (char *) bufBlock, false);
	} else {
		/* Read in the page, unless the caller intends to overwrite it and
		 * just wants us to allocate a buffer. */
		if (mode == RBM_ZERO_AND_LOCK || mode == RBM_ZERO_AND_CLEANUP_LOCK)
			MemSet((char *) bufBlock, 0, BLCKSZ);
		else
		{
            // 从磁盘读数据页
			smgrread(smgr, forkNum, blockNum, (char *) bufBlock);

			/* check for garbage data */
            // 校验数据页
			if (!PageIsVerifiedExtended((Page) bufBlock, blockNum,PIV_LOG_WARNING | PIV_REPORT_STAT))
			{
				if (mode == RBM_ZERO_ON_ERROR || zero_damaged_pages)
				{
					ereport(WARNING,
							(errcode(ERRCODE_DATA_CORRUPTED),
							 errmsg("invalid page in block %u of relation %s; zeroing out page",
									blockNum,
									relpath(smgr->smgr_rnode, forkNum))));
					MemSet((char *) bufBlock, 0, BLCKSZ);
				}
				else   // 如果校验失败，直接报错。
					ereport(ERROR,
							(errcode(ERRCODE_DATA_CORRUPTED),
							 errmsg("invalid page in block %u of relation %s",
									blockNum,
									relpath(smgr->smgr_rnode, forkNum))));
			}
		}
	}
}
```
校验数据页
```c++
bool PageIsVerifiedExtended(Page page, BlockNumber blkno, int flags)
{
	PageHeader	p = (PageHeader) page;
	size_t	   *pagebytes;
	int			i;
	bool		checksum_failure = false;
	bool		header_sane = false;
	bool		all_zeroes = false;
	uint16		checksum = 0;

	/*
	 * Don't verify page data unless the page passes basic non-zero test
	 */
	if (!PageIsNew(page))
	{
		if (DataChecksumsEnabled())
		{
            // 计算校验和
			checksum = pg_checksum_page((char *) page, blkno);
            // 与页Header中存储的pd_checksum进行比对，如果不同，则校验失败
			if (checksum != p->pd_checksum)
				checksum_failure = true;
		}

		/*
		 * The following checks don't prove the header is correct, only that
		 * it looks sane enough to allow into the buffer pool. Later usage of
		 * the block can still reveal problems, which is why we offer the
		 * checksum option.
		 */
		if ((p->pd_flags & ~PD_VALID_FLAG_BITS) == 0 &&
			p->pd_lower <= p->pd_upper &&
			p->pd_upper <= p->pd_special &&
			p->pd_special <= BLCKSZ &&
			p->pd_special == MAXALIGN(p->pd_special))
			header_sane = true;

		if (header_sane && !checksum_failure)
			return true;
	}

	/* Check all-zeroes case */
	all_zeroes = true;
	pagebytes = (size_t *) page;
	for (i = 0; i < (BLCKSZ / sizeof(size_t)); i++)
	{
		if (pagebytes[i] != 0)
		{
			all_zeroes = false;
			break;
		}
	}

	if (all_zeroes)
		return true;

	/* Throw a WARNING if the checksum fails, but only after we've checked for
	 * the all-zeroes case. */
	if (checksum_failure)
	{
		if ((flags & PIV_LOG_WARNING) != 0)
			ereport(WARNING,
					(errcode(ERRCODE_DATA_CORRUPTED),
					 errmsg("page verification failed, calculated checksum %u but expected %u",
							checksum, p->pd_checksum)));

		if ((flags & PIV_REPORT_STAT) != 0)
			pgstat_report_checksum_failure();

		if (header_sane && ignore_checksum_failure)
			return true;
	}

	return false;
}
```




#### 计算校验和的算法
关于具体的算法实现，可参考[src/include/storage/checksum_impl.h](https://github.com/postgres/postgres/blob/master/src/include/storage/checksum_impl.h)。基于[FNV-1a hash](http://www.isthe.com/chongo/tech/comp/fnv/)算法改造而来。具体算法我们不去研究。但是对于数据页的校验和计算，其实可以认为是对页数据计算哈希，找到一种能快速计算，为了提升快速计算的能力，要求算法能够支持SIMD计算。
```c++
/*
 * The algorithm used to checksum pages is chosen for very fast calculation.
 * Workloads where the database working set fits into OS file cache but not
 * into shared buffers can read in pages at a very fast pace and the checksum
 * algorithm itself can become the largest bottleneck.
 *
 * The checksum algorithm itself is based on the FNV-1a hash (FNV is shorthand
 * for Fowler/Noll/Vo).  The primitive of a plain FNV-1a hash folds in data 1
 * byte at a time according to the formula:
 *
 *	   hash = (hash ^ value) * FNV_PRIME
 *
 * FNV-1a algorithm is described at http://www.isthe.com/chongo/tech/comp/fnv/
 *
 * PostgreSQL doesn't use FNV-1a hash directly because it has bad mixing of
 * high bits - high order bits in input data only affect high order bits in
 * output data. To resolve this we xor in the value prior to multiplication
 * shifted right by 17 bits. The number 17 was chosen because it doesn't
 * have common denominator with set bit positions in FNV_PRIME and empirically
 * provides the fastest mixing for high order bits of final iterations quickly
 * avalanche into lower positions. For performance reasons we choose to combine
 * 4 bytes at a time. The actual hash formula used as the basis is:
 *
 *	   hash = (hash ^ value) * FNV_PRIME ^ ((hash ^ value) >> 17)
 *
 * The main bottleneck in this calculation is the multiplication latency. To
 * hide the latency and to make use of SIMD parallelism multiple hash values
 * are calculated in parallel. The page is treated as a 32 column two
 * dimensional array of 32 bit values. Each column is aggregated separately
 * into a partial checksum. Each partial checksum uses a different initial
 * value (offset basis in FNV terminology). The initial values actually used
 * were chosen randomly, as the values themselves don't matter as much as that
 * they are different and don't match anything in real data. After initializing
 * partial checksums each value in the column is aggregated according to the
 * above formula. Finally two more iterations of the formula are performed with
 * value 0 to mix the bits of the last value added.
 *
 * The partial checksums are then folded together using xor to form a single
 * 32-bit checksum. The caller can safely reduce the value to 16 bits
 * using modulo 2^16-1. That will cause a very slight bias towards lower
 * values but this is not significant for the performance of the
 * checksum.
 *
 * The algorithm choice was based on what instructions are available in SIMD
 * instruction sets. This meant that a fast and good algorithm needed to use
 * multiplication as the main mixing operator. The simplest multiplication
 * based checksum primitive is the one used by FNV. The prime used is chosen
 * for good dispersion of values. It has no known simple patterns that result
 * in collisions. Test of 5-bit differentials of the primitive over 64bit keys
 * reveals no differentials with 3 or more values out of 100000 random keys
 * colliding. Avalanche test shows that only high order bits of the last word
 * have a bias. Tests of 1-4 uncorrelated bit errors, stray 0 and 0xFF bytes,
 * overwriting page from random position to end with 0 bytes, and overwriting
 * random segments of page with 0x00, 0xFF and random data all show optimal
 * 2e-16 false positive rate within margin of error.
 *
 * Vectorization of the algorithm requires 32bit x 32bit -> 32bit integer
 * multiplication instruction. As of 2013 the corresponding instruction is
 * available on x86 SSE4.1 extensions (pmulld) and ARM NEON (vmul.i32).
 * Vectorization requires a compiler to do the vectorization for us. For recent
 * GCC versions the flags -msse4.1 -funroll-loops -ftree-vectorize are enough
 * to achieve vectorization.
 *
 * The optimal amount of parallelism to use depends on CPU specific instruction
 * latency, SIMD instruction width, throughput and the amount of registers
 * available to hold intermediate state. Generally, more parallelism is better
 * up to the point that state doesn't fit in registers and extra load-store
 * instructions are needed to swap values in/out. The number chosen is a fixed
 * part of the algorithm because changing the parallelism changes the checksum
 * result.
 *
 * The parallelism number 32 was chosen based on the fact that it is the
 * largest state that fits into architecturally visible x86 SSE registers while
 * leaving some free registers for intermediate values. For future processors
 * with 256bit vector registers this will leave some performance on the table.
 * When vectorization is not available it might be beneficial to restructure
 * the computation to calculate a subset of the columns at a time and perform
 * multiple passes to avoid register spilling. This optimization opportunity
 * is not used. Current coding also assumes that the compiler has the ability
 * to unroll the inner loop to avoid loop overhead and minimize register
 * spilling. For less sophisticated compilers it might be beneficial to
 * manually unroll the inner loop.
 */
```