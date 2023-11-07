### PostgreSQL源码分析——WAL日志（一）
WAL日志在数据库中非常重要，这里我们分析一下WAL日志的相关内容。WAL日志对数据库故障恢复，物理备份，事务处理等至关重要。


如果没有WAL日志文件，那么数据库需要在每次更新页面后，刷盘持久化才能提交，频繁的进行刷盘，并且是随机写，而且是整页的写，代价很大。对此，设计了WAL，每次只将对数据页的修改抽象成WAL日志记录，顺序写入WAL日志文件中，这就将随机写变为了顺序写入，并且文件的大小也有所减少，因为绝大部分情况不用整页写入了，减少了刷盘的代价。不用每次更新都需要刷盘了，变为启动一个后台bgwriter进程，间隔一段时间进行刷盘即可，出现故障，对WAL日志进行重放从而进行故障恢复，因为故障是个低概率事件，我们通过WAL的方式，避免了每次都进行数据页刷盘，提高了性能，并且通过后台bgwriter进程进行刷盘，对运行过程中的稳定性也是有非常大的好处。

所以一定要理解WAL机制。PG的WAL机制是怎么设计的呢？我们先看一下日志文件的命名规则，从这里开始看起。WAL日志是顺序写入日志文件中，并且通过LSN标识每一条日志记录。所以说日志文件的名称并不是随便起一个名字就好了，我们要通过文件名称能够划分一些WAL日志范围，这样在查找WAL日志的时候，就不用遍历所有文件了。

#### 日志文件命名规则
WAL日志，在数据库实例的pg_wal文件目录中，默认文件大小为16MB，文件名有其命名规则，文件名长度24个字符（TimeLine  + 逻辑ID + 段号），每个字符以十六进制表示，前8个字符是事务日志的时间线（TimeLine），中间8位是逻辑ID，最后8位是段号。 具体的WAL日志存在WAL日志文件中，WAL的日志以LSN进行标识，由日志的逻辑ID、段号以及段内偏移量构成LSN。LSN可以用形如`0/1696FD0`的字符串表示，其中斜线前面的`0`代表日志的逻辑ID，斜线后的`1`表示段ID，这里比较特殊，因为斜线后面是7位字符，后6位数是代表这个LSN在段内的偏移量，前面2位，这里0被省略了，前面的`1`表示段ID。当前时间线为1，所以，上面就表示`0/1696FD0`对于的日志文件是`000000010000000000000001`，在文件中的偏移量是`6909904`（将696FD0由16进制转为10进制）

```shell
postgres@slpc:~/pgsql/pgdata/pg_wal$ ls -lh
total 17M
-rw------- 1 postgres postgres  16M  9月 26 14:48 000000010000000000000001
drwx------ 2 postgres postgres 4.0K  9月 23 10:41 archive_status
```
我们看一下LSN对于的日志文件, ：
```sql
postgres=# show wal_segment_size;
 wal_segment_size 
------------------
 16MB
(1 row)

postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/1696FD0
(1 row)

postgres=# select pg_walfile_name(pg_current_wal_insert_lsn());
     pg_walfile_name      
--------------------------
 000000010000000000000001
(1 row)
```

#### 查看日志内容
PG提供了查看事务日志的方法，方便开发者进行调试，编译时可打开`WAL_DEBUG`编译开关，例如：`./configure --prefix=/home/postgres/pgsql --enable-debug --enable-cassert  --enable-depend CFLAGS="-O0" CPPFLAGS='-DWAL_DEBUG'`进行编译，设置`set wal_debug = on`即可。
```sql
postgres=# begin;
BEGIN
postgres=*# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/1696FD0
(1 row)

postgres=*# insert into t1 values(3),(4),(5);
LOG:  INSERT @ 0/1697090:  - Heap/INSERT: off 3 flags 0x00
LOG:  INSERT @ 0/16970D0:  - Heap/INSERT: off 4 flags 0x00
LOG:  INSERT @ 0/1697110:  - Heap/INSERT: off 5 flags 0x00
INSERT 0 3
postgres=*# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/1697148
(1 row)

postgres=*# select txid_current();
 txid_current 
--------------
          737
(1 row)

postgres=*# commit;
LOG:  INSERT @ 0/1697170:  - Transaction/COMMIT: 2023-09-26 16:53:37.499218+08
LOG:  xlog flush request 0/1697170; write 0/1696FD0; flush 0/1696FD0
COMMIT
```
更详细的，除了查看日志信息，还可以通过pg_waldump工具解析日志文件。例如，在插入前，事务日志的插入点是`0/1696FD0`，我们查看这个位置之后的WAL日志，我们看到有3条insert日志，事务ID是737, 开启了full page write。
```shell
postgres@slpc:~/pgsql/pgdata$ pg_waldump -s 0/1696FD0 pg_wal/000000010000000000000001 
rmgr: Heap        len (rec/tot):     54/   186, tx:        737, lsn: 0/01696FD0, prev 0/01696F98, desc: INSERT off 3 flags 0x00, blkref #0: rel 1663/13010/16384 blk 0 FPW
rmgr: Heap        len (rec/tot):     59/    59, tx:        737, lsn: 0/01697090, prev 0/01696FD0, desc: INSERT off 4 flags 0x00, blkref #0: rel 1663/13010/16384 blk 0
rmgr: Heap        len (rec/tot):     59/    59, tx:        737, lsn: 0/016970D0, prev 0/01697090, desc: INSERT off 5 flags 0x00, blkref #0: rel 1663/13010/16384 blk 0
rmgr: Standby     len (rec/tot):     54/    54, tx:          0, lsn: 0/01697110, prev 0/016970D0, desc: RUNNING_XACTS nextXid 738 latestCompletedXid 736 oldestRunningXid 737; 1 xacts: 737
pg_waldump: fatal: error in WAL record at 0/1697110: invalid record length at 0/1697148: wanted 24, got 0
```

#### 日志格式
我们看一下PG中WAL日志的设计，很重要的一点事日志格式的设计。日志由固定大小的多个段组成，每个段是一个独立的日志文件。每个文件都在内部划分为多个日志页面，默认情况下，日志页面大小为8kB。日志页面由页面头和连续的日志记录（Record）组成。日志记录是事务日志的最小单元。
```shell
日志文件（段）--> 页 --> (页头 + 日志记录...) 
``` 
可以认为，WAL日志，是一个单调递增的日志，会不断地想日志文件追加日志，同时因为checkpoint，也会进行WAL日志回收。这个特点，是我们设计WAL日志机制的出发考量点之一。

WAL日志的存储，有点类似于堆表，存在日志文件中，每个段划分为若干页面，在页面中追加日志记录。具体的，在存储时，日志页面的Header分为两种类型，日志段（日志文件）中的第一个页面的Header中包含段长度段页面的大小等信息，使用`XLogLongPageHeaderData`标识，日志段的其他页面使用`XLogPageHeaderData`标识。

```c++
// 非日志段第一个页面的页Header
typedef struct XLogPageHeaderData
{
	uint16		xlp_magic;		/* magic value for correctness checks */
	uint16		xlp_info;		/* flag bits, see below */
	TimeLineID	xlp_tli;		/* TimeLineID of first record on page */
	// 当前日志页面敌营的日志文件中的开始位置
	XLogRecPtr	xlp_pageaddr;	/* XLOG address of this page */
	// 日志记录跨页面保存时，当前页面续接上一个页面的部分日志的长度  
	uint32		xlp_rem_len;	/* total len of remaining data for record */
} XLogPageHeaderData;

// 日志段第一个页面的页Header
typedef struct XLogLongPageHeaderData
{
	XLogPageHeaderData std;		/* standard header fields */
	uint64		xlp_sysid;		/* system identifier from pg_control */
	uint32		xlp_seg_size;	/* just as a cross-check */
	uint32		xlp_xlog_blcksz;	/* just as a cross-check */
} XLogLongPageHeaderData;
```
每个页面中都有多条日志记录，为了节省页面空间，如果页面的剩余空间大于日志记录的头部，同时又不足以放下一条完整的日志记录，则这条日志记录可以跨页面保存。通过`XlogPageHeaderData.xlp_rem_len`来进行判断接续上一个页面的部分日志记录长度。
```c++
// 日志段1
/*
       第1个页面                第2个页面                页面n ...
| XLogLongPageHeaderData |  XLogPageHeaderData  | 
| XLogRecord1            |  XLogRecord3的后半部分|
| XLogRecord2            |  XLogRecord4         |
| XLogRecord3 前半部分    |  XLogRecord5         |

// 日志段2

// ...

// 日志段n
*/
```
上面讲述了日志段与页面设计，在看一下具体存放的日志记录的设计，日志记录采用`XLogRecord`结构体与数据的方式表示，每条日志记录都表示修改数据库的一个动作。
```c++
typedef struct XLogRecord
{
	uint32		xl_tot_len;		/* total len of entire record */ // 日志记录的长度
	TransactionId xl_xid;		/* xact id */ // 事务ID
	XLogRecPtr	xl_prev;		/* ptr to previous record in log */ // 上一条日志记录的LSN
	uint8		xl_info;		/* flag bits, see below */	// 产生这个记录的动作
	RmgrId		xl_rmid;		/* resource manager for this record */
	/* 2 bytes of padding here, initialize to zero */
	pg_crc32c	xl_crc;			/* CRC for this record */

	/* XLogRecordBlockHeaders and XLogRecordDataHeader follow, no padding */
	// 日志记录的数据信息
} XLogRecord;
```
日志记录之间隐含着链接关系。通过`xl_prev`能够定位到上一条日志记录的起始位置。由于日志记录是顺序存储的，所以通过`xl_tot_len`可以定位到下一条日志记录的起始位置。也就是说，日志记录之间形成了一个双向链表。其布局结构可参考`XLogRecord`的注释说明：
```c++
/*
 * The overall layout of an XLOG record is:
 *		Fixed-size header (XLogRecord struct)     // 每个日志记录固定的头部信息
 *		XLogRecordBlockHeader struct			  // 每个日志记录可以有0~N个Block头部信息，与下面的BlockData对应
 *		XLogRecordBlockHeader struct
 *		...
 *		XLogRecordDataHeader[Short|Long] struct		// 日志记录的实际数据信息
 *		block data
 *		block data
 *		...
 *		main data
 *
 * There can be zero or more XLogRecordBlockHeaders, and 0 or more bytes of
 * rmgr-specific data not associated with a block.  XLogRecord structs
 * always start on MAXALIGN boundaries in the WAL files, but the rest of
 * the fields are not aligned.
 *
 * The XLogRecordBlockHeader, XLogRecordDataHeaderShort and
 * XLogRecordDataHeaderLong structs all begin with a single 'id' byte. It's
 * used to distinguish between block references, and the main data structs.
 */
```
日志记录的数据主要分为两部分：一部分是与页面有关的信息，这部分不保存实际数据，只保存与页面相关的Header信息：实际数据在第二部分，在`XLogRecordDataHeader`存储。如果数据的长度小于256字节，使用`XLogRecordDataHeaderShort`结构体，用1字节来保存数据的长度；否则使用`XLogRecordDataHeaderLong`结构体，用4字节保存数据长度。为啥要这样呢？节约存储空间，性能就是这样一点一点抠出来的。

```c++
// Header info for block data appended to an XLOG record.
typedef struct XLogRecordBlockHeader
{
	uint8		id;				/* block reference ID */
	uint8		fork_flags;		/* fork within the relation, and flags */
	uint16		data_length;	/* number of payload bytes (not including page image) */
} XLogRecordBlockHeader;

typedef struct XLogRecordDataHeaderShort
{
	uint8		id;				/* XLR_BLOCK_ID_DATA_SHORT */
	uint8		data_length;	/* number of payload bytes */ // 数据长度
}			XLogRecordDataHeaderShort;

typedef struct XLogRecordDataHeaderLong
{
	uint8		id;				/* XLR_BLOCK_ID_DATA_LONG */
	/* followed by uint32 data_length, unaligned */
}			XLogRecordDataHeaderLong;
```

#### XLog初始化
在事务提交时，需要将事务日志落盘。也就是说在事务运行过程中，事务日志并非实时刷入磁盘。PG在共享内存中分配了XLog Buffer来缓存WAL日志页面，当需要写入新的日志记录时，这些日志记录会被写入XLog Buffer的页面中，当有事务提交时，会将XLog Buffer中的缓存页面落盘，保证数据不丢失。

在WAL日志的运行过程中，必然涉及到一些状态信息，其中涉及到2个重要的结构体，`XLogCtlData`中记录了当前WAL的写入状态，刷盘状态及Bufer页面的状态信息等。
```c++
/* Total shared-memory state for XLOG.*/
typedef struct XLogCtlData
{
	XLogCtlInsert Insert;	// 控制日志插入的结构体
	
	XLogwrtRqst LogwrtRqst;		// 日志需要写入和刷入的LSN
	XLogRecPtr	RedoRecPtr;		/* a recent copy of Insert->RedoRecPtr */
	FullTransactionId ckptFullXid;	// 最近一次Checkpoint时对应的下一个事务ID
	XLogRecPtr	asyncXactLSN;	/* 最新更新的异步提交LSNt */
	// 日志复制时每个备机会占用一个槽位slot，这里记录所有备机中最小的刷入LSN
	XLogRecPtr	replicationSlotMinLSN;	/* oldest LSN needed by any slot */
	// 最近一次删除的日志段ID
	XLogSegNo	lastRemovedSegNo;	/* latest removed/recycled XLOG segment */

	/* Fake LSN counter, for unlogged relations. Protected by ulsn_lck. */
	XLogRecPtr	unloggedLSN;
	slock_t		ulsn_lck;

	/* Time and LSN of last xlog segment switch. Protected by WALWriteLock. */
	pg_time_t	lastSegSwitchTime; 	// WAL切换到不同的段时，记录当前的时间和刷入的日志的LSN
	XLogRecPtr	lastSegSwitchLSN;

	XLogwrtResult LogwrtResult;		// 日志已经写入和刷入的LSN
	XLogRecPtr	InitializedUpTo;	// 当前XLog Buffer分配的页面中的最后一个页面的LSN

	// XLog Buffer中的页面以及页面的编号
	char	   *pages;			/* buffers for unwritten XLOG pages */
	XLogRecPtr *xlblocks;		/* 1st byte ptr-s + XLOG_BLCKSZ */
	int			XLogCacheBlck;	/* highest allocated xlog buffer index */
	// 当前的时间线信息
	TimeLineID	ThisTimeLineID;
	TimeLineID	PrevTimeLineID;

	/* SharedRecoveryState indicates if we're still in crash or archive recovery.  Protected by info_lck.*/
	RecoveryState SharedRecoveryState;
	/* SharedHotStandbyActive indicates if we allow hot standby queries to be run.  Protected by info_lck.*/
	bool		SharedHotStandbyActive;
	/*SharedPromoteIsTriggered indicates if a standby promotion has beentriggered.*/
	bool		SharedPromoteIsTriggered;
	/* WalWriterSleeping indicates whether the WAL writer is currently in low-power mode */
	bool		WalWriterSleeping;

	/* recoveryWakeupLatch is used to wake up the startup process to continue
	 * WAL replay, if it is waiting for WAL to arrive or failover trigger file to appear. */
	Latch		recoveryWakeupLatch;

	/* During recovery, we keep a copy of the latest checkpoint record here.
	 * lastCheckPointRecPtr points to start of checkpoint record and
	 * lastCheckPointEndPtr points to end+1 of checkpoint record.  Used by the
	 * checkpointer when it wants to create a restartpoint.
	 *
	 * Protected by info_lck. */
	XLogRecPtr	lastCheckPointRecPtr;
	XLogRecPtr	lastCheckPointEndPtr;
	CheckPoint	lastCheckPoint;

	/* lastReplayedEndRecPtr points to end+1 of the last record successfully
	 * replayed. When we're currently replaying a record, ie. in a redo
	 * function, replayEndRecPtr points to the end+1 of the record being
	 * replayed, otherwise it's equal to lastReplayedEndRecPtr.*/
	XLogRecPtr	lastReplayedEndRecPtr;
	TimeLineID	lastReplayedTLI;
	XLogRecPtr	replayEndRecPtr;
	TimeLineID	replayEndTLI;
	/* timestamp of last COMMIT/ABORT record replayed (or being replayed) */
	TimestampTz recoveryLastXTime;

	/* timestamp of when we started replaying the current chunk of WAL data, only relevant for replication or archive recovery */
	TimestampTz currentChunkStartTime;
	/* Recovery pause state */
	RecoveryPauseState recoveryPauseState;
	ConditionVariable recoveryNotPausedCV;

	/* lastFpwDisableRecPtr points to the start of the last replayed
	 * XLOG_FPW_CHANGE record that instructs full_page_writes is disabled.*/
	XLogRecPtr	lastFpwDisableRecPtr;

	slock_t		info_lck;		/* locks shared variables shown above */
} XLogCtlData;

static XLogCtlData *XLogCtl = NULL;  // 定义了静态全局变量XLogCtl
```
`XLogCtlInsert`记录了向WAL日志中写入新的日志记录所需的各种变量。
```c++
/* Shared state data for WAL insertion. */
typedef struct XLogCtlInsert
{
	slock_t		insertpos_lck;	/* protects CurrBytePos and PrevBytePos */

	/*
	 * CurrBytePos is the end of reserved WAL. The next record will be
	 * inserted at that position. PrevBytePos is the start position of the
	 * previously inserted (or rather, reserved) record - it is copied to the
	 * prev-link of the next record. These are stored as "usable byte
	 * positions" rather than XLogRecPtrs (see XLogBytePosToRecPtr()).
	 */
	uint64		CurrBytePos;	// 新日志写入的位置
	uint64		PrevBytePos;	// 新日志需要记录前一条日志的LSN

	/*
	 * Make sure the above heavily-contended spinlock and byte positions are
	 * on their own cache line. In particular, the RedoRecPtr and full page
	 * write variables below should be on a different cache line. They are
	 * read on every WAL insertion, but updated rarely, and we don't want
	 * those reads to steal the cache line containing Curr/PrevBytePos.
	 */
	char		pad[PG_CACHE_LINE_SIZE];

	// FPW相关变量
	XLogRecPtr	RedoRecPtr;		/* current redo point for insertions */
	bool		forcePageWrites;	/* forcing full-page writes for PITR? */
	bool		fullPageWrites;	 // pg_start_backup/pg_stop_backup设置

	// 在线备份
	ExclusiveBackupState exclusiveBackupState;
	int			nonExclusiveBackups;
	XLogRecPtr	lastBackupStart;	// lastBackupStart is the latest checkpoint redo location used as a starting point for an online backup.

	WALInsertLockPadded *WALInsertLocks;  	// WAL日志插入时的锁
} XLogCtlInsert;

```

#### 日志的注册

到目前为止，我们都没有真正的讲到一条WAL日志是如何写入的。下面我就看一下，具体是怎么写入的。在数据库启动之初，会在`InitXLogInsert`函数中初始化几个变量，用来临时保存日志的中间状态：
```c++
/* An array of XLogRecData structs, to hold registered data. */
static XLogRecData *rdatas;   // 长度为XLR_NORMAL_RDATAS的数组，用来保存日志数据信息
static int	num_rdatas;			/* entries currently used */
static int	max_rdatas;			/* allocated size */

/* A chain of XLogRecDatas to hold the "main data" of a WAL record, registered with XLogRegisterData(...).*/
static XLogRecData *mainrdata_head;	// 指针变量，日志数据信息，XLogRegisterData函数注册数据时，从rdatas数组中获取槽位，并通过mainrdata系列变量串联起来
static XLogRecData *mainrdata_last = (XLogRecData *) &mainrdata_head;
static uint32 mainrdata_len;	/* total # of bytes in chain */

#define XLR_NORMAL_MAX_BLOCK_ID		4
#define XLR_NORMAL_RDATAS			20

typedef struct
{
	bool		in_use;			/* 槽位是否在使用中*/
	uint8		flags;			/* REGBUF_* flags */
	/* 页面对于的物理信息 */
	RelFileNode rnode;			
	ForkNumber	forkno;
	BlockNumber block;
	Page		page;			/* 页面内容指针 */
	// XLogRegisterBufData会注册进来的长度
	uint32		rdata_len;		/* total length of data in rdata chain */
	// XLogRegisterBufData会注册数据到这个链表
	XLogRecData *rdata_head;	/* head of the chain of data registered with this block */
	XLogRecData *rdata_tail;	/* last entry in the chain, or &rdata_head if empty */

	XLogRecData bkp_rdatas[2];	/* temporary rdatas used to hold references to backup block data in XLogRecordAssemble() */

	/* buffer to store a compressed version of backup block image */
	char		compressed_page[PGLZ_MAX_BLCKSZ];	// 压缩页面所使用的临时空间
} registered_buffer;
// 页面占用registered_buffers中的槽位序号由手工指定。也就是说当需要注册两个页面时，就为两个页面设置编号0和1，分别会占用数组中的0号和1号槽位
static registered_buffer *registered_buffers;	// 长度为XLR_NORMAL_MAX_BLOCK_ID+1的数组，用于注册被修改的页面信息，由XLogRegisterBuffer注册页面时，会在其中占用一个槽位
static int	max_registered_buffers; /* allocated size */
static int	max_registered_block_id = 0;	/* highest block_id + 1 currently registered */

#define HEADER_SCRATCH_SIZE \
	(SizeOfXLogRecord + \
	 MaxSizeOfXLogRecordBlockHeader * (XLR_MAX_BLOCK_ID + 1) + \
	 SizeOfXLogRecordDataHeaderLong + SizeOfXlogOrigin + \
	 SizeOfXLogTransactionId)
static char *hdr_scratch = NULL;   // 在组装日志时使用的临时内存，长度为HEADER_SCRATCH_SIZ，保存日志记录的Header部分
```
`InitXLogInsert`函数源码如下：
```c++
/* Allocate working buffers needed for WAL record construction. */
void InitXLogInsert(void)
{
	/* Initialize the working areas */
	if (xloginsert_cxt == NULL) {
		xloginsert_cxt = AllocSetContextCreate(TopMemoryContext, "WAL record construction", ALLOCSET_DEFAULT_SIZES);
	}

	if (registered_buffers == NULL) {
		registered_buffers = (registered_buffer *) MemoryContextAllocZero(xloginsert_cxt, sizeof(registered_buffer) * (XLR_NORMAL_MAX_BLOCK_ID + 1));
		max_registered_buffers = XLR_NORMAL_MAX_BLOCK_ID + 1;
	}
	if (rdatas == NULL) {
		rdatas = MemoryContextAlloc(xloginsert_cxt, sizeof(XLogRecData) * XLR_NORMAL_RDATAS);
		max_rdatas = XLR_NORMAL_RDATAS;
	}

	/* Allocate a buffer to hold the header information for a WAL record. */
	if (hdr_scratch == NULL)
		hdr_scratch = MemoryContextAllocZero(xloginsert_cxt, HEADER_SCRATCH_SIZE);
}
```


PG中提供了一套写入日志的“标准接口”，包括注册数据、组装数据及写入数据等。事务日志不直接写入WAL Buffer，而是先组成`XLogRecData`链表，然后将这个链表转化为一条事务日志。
```c++
/* The functions in xloginsert.c construct a chain of XLogRecData structs to represent the final WAL record.*/
typedef struct XLogRecData
{
	struct XLogRecData *next;	/* next struct in chain, or NULL  */
	char	   *data;			/* start of rmgr data to include  */
	uint32		len;			/* length of rmgr data to include */
} XLogRecData;
```
组装以及写入日志的流程大致如下：
```c++
/*
0. 初始化用来临时保存日志的中间状态的几个变量
InitXLogInsert

1. 设置begininsert_called标识防止递归调用日志生成函数，以及代码安全检查工作
XLogBeginInsert  

2. 注册一切所需信息，为组成日志记录做准备
XLogRegisterData   注册生成日志记录的数据，每调用一次就会在rdatas数组中占用一个槽位
XLogRegisterBlock	// Like XLogRegisterBuffer, but for registering a block that's not in the shared buffer pool
XLogRegisterBuffer  //  Register a reference to a buffer with the WAL record being constructed.
XLogRegisterBufData  // Add buffer-specific data to the WAL record that's being constructed.

3. 根据注册的信息，组装日志记录、写入日志记录
XLogInsert
	XLogRecordAssemble    // 组装日志记录
	XLogInsertRecord      // 将日志记录写入日志缓存
	XLogResetInsertion    // 重置写入日志需要的各种变量
*/
```

上面说了这些可能没有一个直观的概念，可以调试一条insert语句的代码，看一下`heap_insert`中是如何插入一条日志记录的，其实要理解WAL日志的插入，光这些是不够的，还要理解日志是怎么进行回放的。日志回放需要哪些信息，我们在构造日志记录阶段就需要构造什么信息。比如插入一条数据，回放的时候，就需要知道插入的是那个页？具体的元组数据是什么？插入的位置是哪里。这些都需要我们在构造日志记录的时候写入日志文件中。这样回放的时候才知道怎么进行回放。
```c++
// 堆表中插入一条元组，   插入过程中会执行向WAL日志中插入一条日志记录
void heap_insert(Relation relation, HeapTuple tup, CommandId cid, int options, BulkInsertState bistate)
{
	TransactionId xid = GetCurrentTransactionId();  // 获取事务XID
	HeapTuple	heaptup;
	Buffer		buffer;
	Buffer		vmbuffer = InvalidBuffer;
	bool		all_visible_cleared = false;
	
	// ...
	// 构造HeapTuple， 添加事务等信息
	heaptup = heap_prepare_insert(relation, tup, xid, cid, options);
	
	// Buffer中获取足够插入tuple大小的页， 要大于heaptup->t_len
	buffer = RelationGetBufferForTuple(relation, heaptup->t_len, InvalidBuffer, options, bistate, &vmbuffer, NULL);
	// ...

	// 向页中插入tuple
	RelationPutHeapTuple(relation, buffer, heaptup, (options & HEAP_INSERT_SPECULATIVE) != 0);
	
	// ...

	// 页插入tuple后，标识为脏页
	MarkBufferDirty(buffer);

	// 向WAL日志中插入日志记录
	/* XLOG stuff */
	if (RelationNeedsWAL(relation))
	{
		xl_heap_insert xlrec;
		xl_heap_header xlhdr;
		XLogRecPtr	recptr;
		Page		page = BufferGetPage(buffer);
		uint8		info = XLOG_HEAP_INSERT;
		int			bufflags = 0;

		/* If this is a catalog, we need to transmit combo CIDs to properly decode, so log that as well. */
		if (RelationIsAccessibleInLogicalDecoding(relation))
			log_heap_new_cid(relation, heaptup);

		/* If this is the single and first tuple on page, we can reinit the page instead of restoring the whole thing.  Set flag, and hide buffer references from XLogInsert. */
		if (ItemPointerGetOffsetNumber(&(heaptup->t_self)) == FirstOffsetNumber && PageGetMaxOffsetNumber(page) == FirstOffsetNumber)
		{
			info |= XLOG_HEAP_INIT_PAGE;
			bufflags |= REGBUF_WILL_INIT;
		}

		xlrec.offnum = ItemPointerGetOffsetNumber(&heaptup->t_self);
		xlrec.flags = 0;
		if (all_visible_cleared)
			xlrec.flags |= XLH_INSERT_ALL_VISIBLE_CLEARED;
		if (options & HEAP_INSERT_SPECULATIVE)
			xlrec.flags |= XLH_INSERT_IS_SPECULATIVE;
		Assert(ItemPointerGetBlockNumber(&heaptup->t_self) == BufferGetBlockNumber(buffer));

		/* For logical decoding, we need the tuple even if we're doing a full page write, so make sure it's included even if we take a full-page image. */
		if (RelationIsLogicallyLogged(relation) && !(options & HEAP_INSERT_NO_LOGICAL))
		{
			xlrec.flags |= XLH_INSERT_CONTAINS_NEW_TUPLE;
			bufflags |= REGBUF_KEEP_DATA;

			if (IsToastRelation(relation))
				xlrec.flags |= XLH_INSERT_ON_TOAST_RELATION;
		}

		XLogBeginInsert();
		XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);

		xlhdr.t_infomask2 = heaptup->t_data->t_infomask2;
		xlhdr.t_infomask = heaptup->t_data->t_infomask;
		xlhdr.t_hoff = heaptup->t_data->t_hoff;

		/* note we mark xlhdr as belonging to buffer; if XLogInsert decides to write the whole page to the xlog, we don't need to store xl_heap_header in the xlog. */
		// INSERT操作只注册了一个页面，它占用的是0号槽位 
		XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);	// 数据页Buffer
		// 0号槽位已经注册了Buffer数据，向0号页面注册元组相关的数据
		XLogRegisterBufData(0, (char *) &xlhdr, SizeOfHeapHeader);
		/* PG73FORMAT: write bitmap [+ padding] [+ oid] + data */
		XLogRegisterBufData(0,
							(char *) heaptup->t_data + SizeofHeapTupleHeader,
							heaptup->t_len - SizeofHeapTupleHeader);

		/* filtering by origin on a row level is much more efficient */
		XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);

		recptr = XLogInsert(RM_HEAP_ID, info);	// 把数据写入WAL Buffer

		PageSetLSN(page, recptr);
	}

	// ...
}
```
用`XLogRegisterBuffer`函数注册页面后，才可以用`XLogRegisterBufData`函数注册数据，因为`XLogRegisterBufData`函数的第一个参数就是`XLogRegisterBuffer`函数注册页面时占用的槽位。
