## PostgreSQL源码分析——pg_buffercache

pg_buffercache是PostgreSQL中的一个插件，通过该插件可以关注到PostgreSQL的Buffer Cache（缓冲区缓存）的运行情况。观察缓冲区有助于我们对数据库进行优化以及性能分析，程序调试等。比如通过分析buffer cache中usage count的情况，初步判断当前缓冲区的大小设置的是否合适等。 本文将分析pg_buffercache插件的源码，以了解其实现原理。

### pg_buffercache插件的使用

pg_buffercache插件的安装：

```shell
CREATE EXTENSION pg_buffercache;
```
安装完成后，可以通过以下命令查询Buffer Cache的信息：
```sql
SELECT * FROM pg_buffercache;
```
示例：
```sql
postgres=# select * from pg_buffercache ;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
        1 |        1262 |          1664 |           0 |             0 |              0 | f       |          4 |                0
        2 |        1260 |          1664 |           0 |             0 |              0 | f       |          3 |                0
        3 |        1259 |          1663 |       13010 |             0 |              0 | t       |          5 |                0
        4 |        1259 |          1663 |       13010 |             0 |              1 | f       |          5 |                0
        5 |        1259 |          1663 |       13010 |             0 |              2 | f       |          5 |                0
        6 |        1259 |          1663 |       13010 |             0 |              3 | f       |          5 |                0
        7 |        1249 |          1663 |       13010 |             0 |              0 | f       |          5 |                0
        8 |        1249 |          1663 |       13010 |             0 |              1 | f       |          5 |                0
        9 |        1249 |          1663 |       13010 |             0 |              2 | f       |          5 |                0
       10 |        1249 |          1663 |       13010 |             0 |              3 | f       |          5 |                0
       11 |        1249 |          1663 |       13010 |             0 |              4 | f       |          5 |                0
       12 |        1249 |          1663 |       13010 |             0 |              5 | f       |          5 |                0
        -- ...
```
上面是反映buffercache的整体情况，现在我们只分析一个表，
```sql
postgres=# \d
             List of relations
 Schema |      Name      | Type  |  Owner   
--------+----------------+-------+----------
 public | pg_buffercache | view  | postgres
 public | t1             | table | postgres

 postgres=# select * from pg_class where relname='t1';
  oid  | relname | relnamespace | reltype | reloftype | relowner | relam | relfilenode | reltablespace | relpages | reltuples | relallvisible | reltoastrelid | relhasindex | relisshared | relpersistence | relkind | relnatts | relchecks 
| relhasrules | relhastriggers | relhassubclass | relrowsecurity | relforcerowsecurity | relispopulated | relreplident | relispartition | relrewrite | relfrozenxid | relminmxid |                     relacl                      | relopti
ons | relpartbound 
-------+---------+--------------+---------+-----------+----------+-------+-------------+---------------+----------+-----------+---------------+---------------+-------------+-------------+----------------+---------+----------+-----------
+-------------+----------------+----------------+----------------+---------------------+----------------+--------------+----------------+------------+--------------+------------+-------------------------------------------------+--------
----+--------------
 16385 | t1      |         2200 |   16387 |         0 |       10 |     2 |       16385 |             0 |        0 |        -1 |             0 |             0 | f           | f           | p              | r       |        1 |         0 
| f           | f              | f              | f              | f                   | t              | d            | f              |          0 |          734 |          1 | {postgres=arwdDxt/postgres,hangzhou=r/postgres} |        
    | 
(1 row)

-- 查看表t1的缓冲区情况
postgres=# select * from pg_buffercache where relfilenode = 16385;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
(0 rows)
-- 进行查询，表t1的数据被加载到缓冲区中
postgres=# select * from t1;
 a 
---
 1
(1 row)
-- 查看表t1的缓冲区情况
postgres=# select * from pg_buffercache where relfilenode = 16385;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
      302 |       16385 |          1663 |       13010 |             0 |              0 | f       |          1 |                0
(1 row)
-- 更新表t1
postgres=# update t1 set a = 10;
UPDATE 1
-- 再次查看缓冲区情况，表t1的数据被标记为脏数据，isdirty为t
postgres=# select * from pg_buffercache where relfilenode = 16385;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
      302 |       16385 |          1663 |       13010 |             0 |              0 | t       |          2 |                0
(1 row)
-- 执行checkpoint，脏页刷盘，isdirty为f
postgres=# checkpoint;
CHECKPOINT
postgres=# select * from pg_buffercache where relfilenode = 16385;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
      302 |       16385 |          1663 |       13010 |             0 |              0 | f       |          2 |                0
(1 row)

```

### pg_buffercache插件的源码分析
pg_buffercache的主要思路就是遍历buffer cache，将每个缓存的页的相关信息(relfilenode、reltablespace、reldatabase、relforknumber、relblocknumber、isdirty、usagecount、pinning_backends)提取出来，然后返回给用户。提取信息的思路也比较容易理解，就是通过buffer_id找到对应的Buffer，然后从Buffer中提取信息即可。


具体实现上就是主要代码为增加了一个函数`pg_buffercache_pages`以及一个视图`pg_buffercache`。

```sql
-- Register the function.
CREATE FUNCTION pg_buffercache_pages()
RETURNS SETOF RECORD
AS 'MODULE_PATHNAME', 'pg_buffercache_pages'
LANGUAGE C PARALLEL SAFE;

-- Create a view for convenient access.
CREATE VIEW pg_buffercache AS
	SELECT P.* FROM pg_buffercache_pages() AS P
	(bufferid integer, relfilenode oid, reltablespace oid, reldatabase oid,
	 relforknumber int2, relblocknumber int8, isdirty bool, usagecount int2,
	 pinning_backends int4);
```
其中`pg_buffercache_pages`函数会遍历Buffer Cache，提取信息。

用户使用时，会查询视图`pg_buffercache`，调用栈如下：
```c++
pg_buffercache.so!pg_buffercache_pages(FunctionCallInfo fcinfo) (contrib\pg_buffercache\pg_buffercache_pages.c:94)
ExecMakeTableFunctionResult(SetExprState * setexpr, ExprContext * econtext, MemoryContext argContext, TupleDesc expectedDesc, _Bool randomAccess) (src\backend\executor\execSRF.c:234)
FunctionNext(FunctionScanState * node) (src\backend\executor\nodeFunctionscan.c:95)
ExecScanFetch(ScanState * node, ExecScanAccessMtd accessMtd, ExecScanRecheckMtd recheckMtd) (src\backend\executor\execScan.c:132)
ExecScan(ScanState * node, ExecScanAccessMtd accessMtd, ExecScanRecheckMtd recheckMtd) (src\backend\executor\execScan.c:198)
ExecFunctionScan(PlanState * pstate) (src\backend\executor\nodeFunctionscan.c:270)
ExecProcNodeFirst(PlanState * node) (src\backend\executor\execProcnode.c:464)
ExecProcNode(PlanState * node) (src\include\executor\executor.h:260)
ExecutePlan(EState * estate, PlanState * planstate, _Bool use_parallel_mode, CmdType operation, _Bool sendTuples, uint64 numberTuples, ScanDirection direction, DestReceiver * dest, _Bool execute_once) (src\backend\executor\execMain.c:1551)
standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\execMain.c:361)
ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\execMain.c:303)
PortalRunSelect(Portal portal, _Bool forward, long count, DestReceiver * dest) (src\backend\tcop\pquery.c:921)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:765)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1217)
```

#### buffer_id获取BufferDesc
以下为相关背景代码，主要是理解通过buffer_id获取BufferDesc的流程：
```c++
typedef struct BufferDesc
{
	BufferTag	tag;			/* ID of page contained in buffer */
	int			buf_id;			/* buffer's index number (from 0) */

	/* state of the tag, containing flags, refcount and usagecount */
	pg_atomic_uint32 state;

	int			wait_backend_pid;	/* backend PID of pin-count waiter */
	int			freeNext;		/* link in freelist chain */
	LWLock		content_lock;	/* to lock access to buffer contents */
} BufferDesc;

typedef union BufferDescPadded
{
	BufferDesc	bufferdesc;
	char		pad[BUFFERDESC_PAD_TO_SIZE];
} BufferDescPadded;

BufferDescPadded *BufferDescriptors;  // buffer descriptors array

// 根据buffer_id获取BufferDesc
#define GetBufferDescriptor(id) (&BufferDescriptors[(id)].bufferdesc)

// 初始化BufferPool，分配空间
void
InitBufferPool(void)
{
	bool		foundBufs,
				foundDescs,
				foundIOCV,
				foundBufCkpt;

	/* Align descriptors to a cacheline boundary. */
	BufferDescriptors = (BufferDescPadded *)
		ShmemInitStruct("Buffer Descriptors",
						NBuffers * sizeof(BufferDescPadded),
						&foundDescs);

	BufferBlocks = (char *)
		ShmemInitStruct("Buffer Blocks",
						NBuffers * (Size) BLCKSZ, &foundBufs);

	/* Align condition variables to cacheline boundary. */
	BufferIOCVArray = (ConditionVariableMinimallyPadded *)
		ShmemInitStruct("Buffer IO Condition Variables",
						NBuffers * sizeof(ConditionVariableMinimallyPadded),
						&foundIOCV);

	/*
	 * The array used to sort to-be-checkpointed buffer ids is located in
	 * shared memory, to avoid having to allocate significant amounts of
	 * memory at runtime. As that'd be in the middle of a checkpoint, or when
	 * the checkpointer is restarted, memory allocation failures would be
	 * painful.
	 */
	CkptBufferIds = (CkptSortItem *)
		ShmemInitStruct("Checkpoint BufferIds",
						NBuffers * sizeof(CkptSortItem), &foundBufCkpt);

	if (foundDescs || foundBufs || foundIOCV || foundBufCkpt)
	{
		/* should find all of these, or none of them */
		Assert(foundDescs && foundBufs && foundIOCV && foundBufCkpt);
		/* note: this path is only taken in EXEC_BACKEND case */
	}
	else
	{
		int			i;

		/*
		 * Initialize all the buffer headers.
		 */
		for (i = 0; i < NBuffers; i++)
		{
			BufferDesc *buf = GetBufferDescriptor(i);

			CLEAR_BUFFERTAG(buf->tag);

			pg_atomic_init_u32(&buf->state, 0);
			buf->wait_backend_pid = 0;

			buf->buf_id = i;

			/*
			 * Initially link all the buffers together as unused. Subsequent
			 * management of this list is done by freelist.c.
			 */
			buf->freeNext = i + 1;

			LWLockInitialize(BufferDescriptorGetContentLock(buf),
							 LWTRANCHE_BUFFER_CONTENT);

			ConditionVariableInit(BufferDescriptorGetIOCV(buf));
		}

		/* Correct last entry of linked list */
		GetBufferDescriptor(NBuffers - 1)->freeNext = FREENEXT_END_OF_LIST;
	}

	/* Init other shared buffer-management stuff */
	StrategyInitialize(!foundDescs);

	/* Initialize per-backend file flush context */
	WritebackContextInit(&BackendWritebackContext,
						 &backend_flush_after);
}
```


#### pg_buffercache_pages函数源码如下：
```c++
Datum pg_buffercache_pages(PG_FUNCTION_ARGS)
{
	FuncCallContext *funcctx;
	Datum		result;
	MemoryContext oldcontext;
	BufferCachePagesContext *fctx;	/* User function context. */
	TupleDesc	tupledesc;
	TupleDesc	expected_tupledesc;
	HeapTuple	tuple;

	if (SRF_IS_FIRSTCALL())
	{
		int			i;

		funcctx = SRF_FIRSTCALL_INIT();

		/* Switch context when allocating stuff to be used in later calls */
		oldcontext = MemoryContextSwitchTo(funcctx->multi_call_memory_ctx);

		/* Create a user function context for cross-call persistence */
		fctx = (BufferCachePagesContext *) palloc(sizeof(BufferCachePagesContext));

		/*
		 * To smoothly support upgrades from version 1.0 of this extension
		 * transparently handle the (non-)existence of the pinning_backends
		 * column. We unfortunately have to get the result type for that... -
		 * we can't use the result type determined by the function definition
		 * without potentially crashing when somebody uses the old (or even
		 * wrong) function definition though.
		 */
		if (get_call_result_type(fcinfo, NULL, &expected_tupledesc) != TYPEFUNC_COMPOSITE)
			elog(ERROR, "return type must be a row type");

		if (expected_tupledesc->natts < NUM_BUFFERCACHE_PAGES_MIN_ELEM ||
			expected_tupledesc->natts > NUM_BUFFERCACHE_PAGES_ELEM)
			elog(ERROR, "incorrect number of output arguments");

		/* Construct a tuple descriptor for the result rows. */
		tupledesc = CreateTemplateTupleDesc(expected_tupledesc->natts);
		TupleDescInitEntry(tupledesc, (AttrNumber) 1, "bufferid",
						   INT4OID, -1, 0);
		TupleDescInitEntry(tupledesc, (AttrNumber) 2, "relfilenode",
						   OIDOID, -1, 0);
		TupleDescInitEntry(tupledesc, (AttrNumber) 3, "reltablespace",
						   OIDOID, -1, 0);
		TupleDescInitEntry(tupledesc, (AttrNumber) 4, "reldatabase",
						   OIDOID, -1, 0);
		TupleDescInitEntry(tupledesc, (AttrNumber) 5, "relforknumber",
						   INT2OID, -1, 0);
		TupleDescInitEntry(tupledesc, (AttrNumber) 6, "relblocknumber",
						   INT8OID, -1, 0);
		TupleDescInitEntry(tupledesc, (AttrNumber) 7, "isdirty",
						   BOOLOID, -1, 0);
		TupleDescInitEntry(tupledesc, (AttrNumber) 8, "usage_count",
						   INT2OID, -1, 0);

		if (expected_tupledesc->natts == NUM_BUFFERCACHE_PAGES_ELEM)
			TupleDescInitEntry(tupledesc, (AttrNumber) 9, "pinning_backends",
							   INT4OID, -1, 0);

		fctx->tupdesc = BlessTupleDesc(tupledesc);

		/* Allocate NBuffers worth of BufferCachePagesRec records. */
            // NBuffers 是缓冲区数量，BufferCachePagesRec 是缓冲区信息结构体，记录了缓冲区的各种信息
            // 这里分配了 NBuffers 个 BufferCachePagesRec 结构体，NBuffers由shared_buffers参数决定
            // 比如shared_buffers = 128MB, 则NBuffers = 128*1024/8 = 16384
		fctx->record = (BufferCachePagesRec *)
			MemoryContextAllocHuge(CurrentMemoryContext,
								   sizeof(BufferCachePagesRec) * NBuffers);

		/* Set max calls and remember the user function context. */
		funcctx->max_calls = NBuffers;
		funcctx->user_fctx = fctx;

		/* Return to original context when allocating transient memory */
		MemoryContextSwitchTo(oldcontext);

		/*
		 * Scan through all the buffers, saving the relevant fields in the
		 * fctx->record structure.
		 *
		 * We don't hold the partition locks, so we don't get a consistent
		 * snapshot across all buffers, but we do grab the buffer header
		 * locks, so the information of each buffer is self-consistent.
		 */
		for (i = 0; i < NBuffers; i++)      // 遍历所有的buffer，获取每个buffer的信息
		{
			BufferDesc *bufHdr;
			uint32		buf_state;

			bufHdr = GetBufferDescriptor(i);
			/* Lock each buffer header before inspecting. */
                 /*
                  * Buffer state is a single 32-bit variable where following data is combined.
                  *
                  * - 18 bits refcount
                  * - 4 bits usage count
                  * - 10 bits of flags
                  *
                  *  涉及到bufferstate的设计
                  */
			buf_state = LockBufHdr(bufHdr);

			fctx->record[i].bufferid = BufferDescriptorGetBuffer(bufHdr);
			fctx->record[i].relfilenode = bufHdr->tag.rnode.relNode;
			fctx->record[i].reltablespace = bufHdr->tag.rnode.spcNode;
			fctx->record[i].reldatabase = bufHdr->tag.rnode.dbNode;
			fctx->record[i].forknum = bufHdr->tag.forkNum;
			fctx->record[i].blocknum = bufHdr->tag.blockNum;
			fctx->record[i].usagecount = BUF_STATE_GET_USAGECOUNT(buf_state);
			fctx->record[i].pinning_backends = BUF_STATE_GET_REFCOUNT(buf_state);

			if (buf_state & BM_DIRTY)
				fctx->record[i].isdirty = true;
			else
				fctx->record[i].isdirty = false;

			/* Note if the buffer is valid, and has storage created */
			if ((buf_state & BM_VALID) && (buf_state & BM_TAG_VALID))
				fctx->record[i].isvalid = true;
			else
				fctx->record[i].isvalid = false;

			UnlockBufHdr(bufHdr, buf_state);
		}
	}

	funcctx = SRF_PERCALL_SETUP();

	/* Get the saved state */
	fctx = funcctx->user_fctx;

	if (funcctx->call_cntr < funcctx->max_calls)
	{
		uint32		i = funcctx->call_cntr;
		Datum		values[NUM_BUFFERCACHE_PAGES_ELEM];
		bool		nulls[NUM_BUFFERCACHE_PAGES_ELEM];

		values[0] = Int32GetDatum(fctx->record[i].bufferid);
		nulls[0] = false;

		/*
		 * Set all fields except the bufferid to null if the buffer is unused
		 * or not valid.
		 */
		if (fctx->record[i].blocknum == InvalidBlockNumber ||
			fctx->record[i].isvalid == false)
		{
			nulls[1] = true;
			nulls[2] = true;
			nulls[3] = true;
			nulls[4] = true;
			nulls[5] = true;
			nulls[6] = true;
			nulls[7] = true;
			/* unused for v1.0 callers, but the array is always long enough */
			nulls[8] = true;
		}
		else
		{
			values[1] = ObjectIdGetDatum(fctx->record[i].relfilenode);
			nulls[1] = false;
			values[2] = ObjectIdGetDatum(fctx->record[i].reltablespace);
			nulls[2] = false;
			values[3] = ObjectIdGetDatum(fctx->record[i].reldatabase);
			nulls[3] = false;
			values[4] = ObjectIdGetDatum(fctx->record[i].forknum);
			nulls[4] = false;
			values[5] = Int64GetDatum((int64) fctx->record[i].blocknum);
			nulls[5] = false;
			values[6] = BoolGetDatum(fctx->record[i].isdirty);
			nulls[6] = false;
			values[7] = Int16GetDatum(fctx->record[i].usagecount);
			nulls[7] = false;
			/* unused for v1.0 callers, but the array is always long enough */
			values[8] = Int32GetDatum(fctx->record[i].pinning_backends);
			nulls[8] = false;
		}

		/* Build and return the tuple. */
		tuple = heap_form_tuple(fctx->tupdesc, values, nulls);
		result = HeapTupleGetDatum(tuple);

		SRF_RETURN_NEXT(funcctx, result);
	}
	else
		SRF_RETURN_DONE(funcctx);
}

```

参考文档：
[pg_buffercahce](http://postgres.cn/docs/15/pgbuffercache.html)