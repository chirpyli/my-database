## 在PostgreSQL中实现堆表预读
目前polardb for postgresql以及openGauss中已经实现了堆表预读，我们在PostgreSQL中也参考实现堆表预读并进行测试。


堆表预读，仅限heap表顺序扫描，索引扫描不会走堆表扫描。在PostgreSQL中，有Buffer，有环形缓冲区，例如vacuum，copy from等，也同样适用堆表预读，读大表时也会将表读到环形缓冲区中。


堆表扫描描述符，继承自表扫描描述符，堆表扫描描述符包含堆表扫描相关的信息，例如当前扫描的块号，当前扫描的块缓冲区，当前扫描的元组等。
```c++
/* Descriptor for heap table scans. */
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

	/*
	 * For parallel scans to store page allocation data.  NULL when not
	 * performing a parallel scan.
	 */
	ParallelBlockTableScanWorkerData *rs_parallelworkerdata;

	/* these fields only used in page-at-a-time mode and for bitmap scans */
	int			rs_cindex;		/* current tuple's index in vistuples */
	int			rs_ntuples;		/* number of visible tuples on page */
	OffsetNumber rs_vistuples[MaxHeapTuplesPerPage];	/* their offsets */

	ScanDirection scan_direction;
}			HeapScanDescData;
typedef struct HeapScanDescData *HeapScanDesc;
```


### 关于堆表预读性能提升
1. 堆表预读性能提升受多种因素影响，堆表预读是在操作系统文件系统之上。Linux文件系统有预读以及文件系统缓存，在测试性能时，数据库与文件系统参数的调优都有影响，比如设置数据库Buffer大小，文件缓存的大小。


Linux文件系统对顺序读的优化：预读，读流程，首先从页缓存查找数据，没有命中则从磁盘读取数据并填充页缓存。Linux如何判断是顺序读还是随机读呢？当有多个地址连续地读请求时会触发预读操作。



顺序扫描性能相关设计：

1. 并行顺序扫描
2. 需要注意的是，当顺序扫描大表时，采用的是环形缓冲区而不是Buffer，判断条件为表读取数据的大小超多缓冲池的1/4时，将采用环形缓冲区，大小为256kB。
3. 如果表相对于 NBuffers 而言较大，使用批量读取访问策略并启用同步扫描

/*
 * syncscan.c
 *      扫描同步支持
 *
 * 当多个后台进程在同一个表上进行顺序扫描时，我们尝试使它们同步，以减少总体I/O需求。目标是仅将每个页面读取到共享缓冲区缓存一次，并让参与共享扫描的所有后台进程在页面从缓存中移出之前处理该页面。
 *
 * 由于进行顺序扫描的后台进程中的“领导者”必须等待I/O，而“跟随者”则不需要，因此一旦我们能使后台进程同时检查表的大致相同部分，就会有很强的自同步效果。因此，真正需要的只是让一个新启动顺序扫描的后台进程开始接近其他后台进程正在读取的位置。我们可以从块X开始循环扫描表，一直到末尾，然后从块0到X-1，以确保我们在参与公共扫描的同时访问所有行。
 *
 * 为此，我们跟踪每个表的扫描位置，并在先前扫描的位置附近开始新扫描。我们不尝试进行任何额外的同步来让扫描保持在一起；例如，如果结果需要通过慢速网络传输到客户端，有些扫描可能会比其他扫描进行得慢得多，我们不希望此类查询拖慢其他查询。
 *
 * 实际上，同时进行的不同表上的大型顺序扫描只能有几个。因此，我们只是将扫描位置保存在一个小的LRU列表中，每次需要查找或更新扫描位置时进行扫描。整个机制仅适用于超出阈值大小的表（但这不是本模块关注的问题）。
 *
 * 接口例程
 *      ss_get_location    - 返回关系的当前扫描位置
 *      ss_report_location - 更新当前扫描位置
 *
 *
 * 部分版权 (c) 1996-2021, PostgreSQL全球开发组
 * 部分版权 (c) 1994, 加州大学董事会
 *
 * 标识
 *      src/backend/access/common/syncscan.c
 *
 */


---
参考文档：
[MogDB顺序扫描预读](https://docs.mogdb.io/zh/mogdb/v5.0/seqscan-prefetch)