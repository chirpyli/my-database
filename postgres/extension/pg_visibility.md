### PostgreSQL数据库插件——pg_visibility

pg_visibility是一个PostgreSQL数据库插件，提供了一种方式来检查一个表的可见性映射（VM）以及页级别的可见性信息。

#### 插件如何使用？
该插件提供了一些函数，通过调用这些函数，可以获取给定关系的可见性信息。用法示例如下：
```sql
-- 安装插件
postgres=# create extension pg_visibility;
CREATE EXTENSION

-- 为给定关系的给定块返回其在可见性映射中的“全部可见”和“全部冻结”位。
postgres=# select * from pg_visibility_map('t3',0);
 all_visible | all_frozen 
-------------+------------
 t           | f
(1 row)

-- 为给定关系的给定块返回其在可见性映射中的“全部可见”和“全部冻结”位，外加块的PD_ALL_VISIBLE位。
postgres=# select * from pg_visibility('t3',0);
 all_visible | all_frozen | pd_all_visible 
-------------+------------+----------------
 t           | f          | t
(1 row)

-- 为给定关系的每一块返回其在可见性映射中的“全部可见”和“全部冻结”位，外加每一块的PD_ALL_VISIBLE位。
postgres=# select * from pg_visibility('t2');
 blkno | all_visible | all_frozen | pd_all_visible 
-------+-------------+------------+----------------
     0 | t           | t          | t
     1 | t           | t          | t
     2 | t           | t          | t
     3 | t           | t          | t
     4 | t           | t          | t
     5 | t           | t          | t
     6 | t           | t          | t
     7 | t           | t          | t
     8 | t           | f          | t
(9 rows)
```
更多用法可参考[pg_visibility](http://www.postgres.cn/docs/current/pgvisibility.html)

其核心原理就是读取可见性文件以及页面数据，通过位运算来获取页面的可见性信息。页头部有个`pd_flags`字段，其中包含`PD_ALL_VISIBLE`位，如果该位为1，则表示该页面所有元组可见。
```c++
// 判断页面是否全部可见
#define PageIsAllVisible(page) \
	(((PageHeader) (page))->pd_flags & PD_ALL_VISIBLE)

#define PD_HAS_FREE_LINES	0x0001	/* are there any unused line pointers? */
#define PD_PAGE_FULL		0x0002	/* not enough free space for new tuple? */
#define PD_ALL_VISIBLE		0x0004	/* all tuples on page are visible to everyone */

#define PD_VALID_FLAG_BITS	0x0007	/* OR of all valid pd_flags bits */
```
函数调用栈：
```c++
pg_visibility.so!collect_visibility_data(Oid relid, _Bool include_pd) (pg_visibility\pg_visibility.c:505)
pg_visibility.so!pg_visibility_rel(FunctionCallInfo fcinfo) (pg_visibility\pg_visibility.c:225)
```
核心函数实现，获取当前表有多少个页，然后查找VM块文件，读取可见性信息，如果需要获取页可见性信息，则读取页面数据，获取`pd_flags`字段，判断是否全部可见。
```c++
static vbits *collect_visibility_data(Oid relid, bool include_pd)
{
	Relation	rel;
	BlockNumber nblocks;
	vbits	   *info;
	BlockNumber blkno;
	Buffer		vmbuffer = InvalidBuffer;
	BufferAccessStrategy bstrategy = GetAccessStrategy(BAS_BULKREAD);

	rel = relation_open(relid, AccessShareLock);

	/* Only some relkinds have a visibility map */
	check_relation_relkind(rel);

	nblocks = RelationGetNumberOfBlocks(rel);  // 获取表有多少个页
	info = palloc0(offsetof(vbits, bits) + nblocks);
	info->next = 0;
	info->count = nblocks;

	for (blkno = 0; blkno < nblocks; ++blkno)	// 遍历所有页号
	{
		int32		mapbits;

		/* Make sure we are interruptible. */
		CHECK_FOR_INTERRUPTS();

		// 获取指定表页号的位图信息
		mapbits = (int32) visibilitymap_get_status(rel, blkno, &vmbuffer);
		if ((mapbits & VISIBILITYMAP_ALL_VISIBLE) != 0)
			info->bits[blkno] |= (1 << 0);
		if ((mapbits & VISIBILITYMAP_ALL_FROZEN) != 0)
			info->bits[blkno] |= (1 << 1);

		// 是否查看页面可见性信息
		if (include_pd)
		{
			Buffer		buffer;
			Page		page;
			// 读页到buffer
			buffer = ReadBufferExtended(rel, MAIN_FORKNUM, blkno, RBM_NORMAL, bstrategy);
			LockBuffer(buffer, BUFFER_LOCK_SHARE);

			page = BufferGetPage(buffer);
			if (PageIsAllVisible(page))	// 判断页面是否全部可见
				info->bits[blkno] |= (1 << 2);

			UnlockReleaseBuffer(buffer);
		}
	}

	/* Clean up. */
	if (vmbuffer != InvalidBuffer)
		ReleaseBuffer(vmbuffer);
	relation_close(rel, AccessShareLock);

	return info;
}
```

#### 为啥需要可见性映射文件？
我们知道，PostgreSQL中因为其MVCC设计，需要通过vacuum来清理死元组，而vacuum操作会扫描整个表，如果一个页面中所有元组都是可见的，那么就不需要对其进行vacuum操作，为了提高效率，我们对每个页面进行标识，如果这个页没有无效元组，则可以跳过该页面，从而提高vacuum的效率，这就是可见性映射文件的作用。
![image](https://www.interdb.jp/pg/pgsql07/fig-7-07.png)
执行vacuum的函数调用栈：
```c++
visibilitymap_get_status(Relation rel, BlockNumber heapBlk, Buffer * buf) (src\backend\access\heap\visibilitymap.c:332)
lazy_scan_heap(LVRelState * vacrel, VacuumParams * params, _Bool aggressive) (src\backend\access\heap\vacuumlazy.c:1090)
heap_vacuum_rel(Relation rel, VacuumParams * params, BufferAccessStrategy bstrategy) (src\backend\access\heap\vacuumlazy.c:638)
table_relation_vacuum(Relation rel, struct VacuumParams * params, BufferAccessStrategy bstrategy) (src\include\access\tableam.h:1678)
vacuum_rel(Oid relid, RangeVar * relation, VacuumParams * params) (src\backend\commands\vacuum.c:2067)
vacuum(List * relations, VacuumParams * params, BufferAccessStrategy bstrategy, _Bool isTopLevel) (src\backend\commands\vacuum.c:476)
ExecVacuum(ParseState * pstate, VacuumStmt * vacstmt, _Bool isTopLevel) (src\backend\commands\vacuum.c:269)
```
在实际vacuum中，并不是可见性映射中被标记为all-visible的页面就一定会被跳过，实际上是有个一阈值`SKIP_PAGES_THRESHOLD`，默认为32，只有当连续跳过的页面数大于这个阈值时，才会跳过这些页面，否则会继续扫描这些页面，由于我们是顺序读取的，OS会进行预读，因此偶尔跳过一页并不会带来太大的性能提升，而且如何跳过一页，也意味着我们无法更新relfrozenxid，会对后续vacuum执行的策略造成影响。

Except when aggressive is set, we want to skip pages that are all-visible according to the visibility map, but only when we can skip at least SKIP_PAGES_THRESHOLD consecutive pages.  Since we're reading sequentially, the OS should be doing readahead for us, so there's no gain in skipping a page now and then; that's likely to disable readahead and so be counterproductive. Also, skipping even a single page means that we can't update relfrozenxid, so we only want to do it if we can skip a goodly number of pages.


#### 可见性映射文件的设计
如同FSM一样，每个表都有一个可见性映射文件，以_vm为后缀，如同堆表文件一样，VM文件也被划分为多个页，每个页除了页头部信息外，剩下的空间都是用来存储可见性映射信息的。
```shell
41821       # 表文件
41821_fsm   # FSM文件
41821_vm    #可见性映射文件
```
The visibility map is a bitmap with two bits **(all-visible and all-frozen)** per heap page. 
- A set all-visible bit means that all tuples on the page are known visible to all transactions, and therefore the page doesn't need to be vacuumed.
- A set all-frozen bit means that all tuples on the page are completely frozen, and therefore the page doesn't need to be vacuumed even if whole table scanning vacuum is required (e.g. anti-wraparound vacuum).

其中可见性映射文件的格式如下：
```c++
/* Number of bits for one heap page */
#define BITS_PER_HEAPBLOCK 2     // 一个页面占2个bit，分别是all_visible和all_frozen

/* Flags for bit map */
#define VISIBILITYMAP_ALL_VISIBLE	0x01
#define VISIBILITYMAP_ALL_FROZEN	0x02
#define VISIBILITYMAP_VALID_BITS	0x03	/* OR of all valid visibilitymap flags bits */
```

页面布局，这里以每个页面8KB为例，页面也可以被设置为其他值。
```
|    8KB  (VM 块)          |           ...             |            8KB              |
| PageHeaderData | bitmap  |
```
页头部如下，页中其他为位图，每个块2bit，分别表示all_visible和all_frozen。
```c++
#define MAPSIZE (BLCKSZ - MAXALIGN(SizeOfPageHeaderData))   // 位图的大小 = 页大小 - 页头部大小
#define HEAPBLOCKS_PER_BYTE (BITS_PER_BYTE / BITS_PER_HEAPBLOCK)  // 一个字节可以表示多少个块信息
#define HEAPBLOCKS_PER_PAGE (MAPSIZE * HEAPBLOCKS_PER_BYTE)    // 一个VM页面可以表示多少个块信息
/* Mapping from heap block number to the right bit in the visibility map */
#define HEAPBLK_TO_MAPBLOCK(x) ((x) / HEAPBLOCKS_PER_PAGE)  // 计算表块号在可见性映射文件中的块号
#define HEAPBLK_TO_MAPBYTE(x) (((x) % HEAPBLOCKS_PER_PAGE) / HEAPBLOCKS_PER_BYTE)  // 计算表块号在可见性映射文件中的字节偏移量
#define HEAPBLK_TO_OFFSET(x) (((x) % HEAPBLOCKS_PER_BYTE) * BITS_PER_HEAPBLOCK) // 计算表块号在可见性映射文件中的位偏移量
```
页头部结构如下：
```c++
typedef struct PageHeaderData
{
	/* XXX LSN is member of *any* block, not only page-organized ones */
	PageXLogRecPtr pd_lsn;		/* LSN: next byte after last byte of xlog
								 * record for last change to this page */
	uint16		pd_checksum;	/* checksum */
	uint16		pd_flags;		/* flag bits, see below */
	LocationIndex pd_lower;		/* offset to start of free space */
	LocationIndex pd_upper;		/* offset to end of free space */
	LocationIndex pd_special;	/* offset to start of special space */
	uint16		pd_pagesize_version;
	TransactionId pd_prune_xid; /* oldest prunable XID, or zero if none */
	ItemIdData	pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
} PageHeaderData;

typedef PageHeaderData *PageHeader;
```
查看可见性映射文件的内容，读VM文件如下：
```c++
// 读取指定表指定块的可见性映射信息
uint8 visibilitymap_get_status(Relation rel, BlockNumber heapBlk, Buffer *buf)
{
	BlockNumber mapBlock = HEAPBLK_TO_MAPBLOCK(heapBlk);   // 计算表块号在可见性映射文件中的块号
	uint32		mapByte = HEAPBLK_TO_MAPBYTE(heapBlk);  // 计算表块号在可见性映射文件中的字节偏移量
	uint8		mapOffset = HEAPBLK_TO_OFFSET(heapBlk); // 计算表块号在可见性映射文件中的位偏移量
	char	   *map;
	uint8		result;

#ifdef TRACE_VISIBILITYMAP
	elog(DEBUG1, "vm_get_status %s %d", RelationGetRelationName(rel), heapBlk);
#endif

	/* Reuse the old pinned buffer if possible */
	if (BufferIsValid(*buf))
	{
		if (BufferGetBlockNumber(*buf) != mapBlock)
		{
			ReleaseBuffer(*buf);
			*buf = InvalidBuffer;
		}
	}

	if (!BufferIsValid(*buf))
	{
		*buf = vm_readbuf(rel, mapBlock, false);	// 读取可见性映射文件中的块
		if (!BufferIsValid(*buf))
			return false;
	}

	map = PageGetContents(BufferGetPage(*buf));		// 获取可见性映射文件中的块内容(位图部分)

	// 获取具体的位信息
	result = ((map[mapByte] >> mapOffset) & VISIBILITYMAP_VALID_BITS);
	return result;
}
```