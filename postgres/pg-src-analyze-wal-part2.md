### PostgreSQL源码分析——WAL日志（二）

#### 日志的组成
日志的注册主要是将WAL日志所需的信息保存在内存中，这些信息需要由`XLogRecordAssemble`函数组装成为最终的日志记录，主要是处理日志记录中与页面（Block）相关的部分，即对在registered_buffers数组中的数据进行二次加工，例如判断是否需要做Full Page Write、是否需要压缩页面等。

这个函数的具体实现源码如下：
```c++
/*
 * Assemble a WAL record from the registered data and buffers into an
 * XLogRecData chain, ready for insertion with XLogInsertRecord().
 *
 * The record header fields are filled in, except for the xl_prev field. The
 * calculated CRC does not include the record header yet.
 *
 * If there are any registered buffers, and a full-page image was not taken
 * of all of them, *fpw_lsn is set to the lowest LSN among such pages. This
 * signals that the assembled record is only good for insertion on the
 * assumption that the RedoRecPtr and doPageWrites values were up-to-date.
 */
static XLogRecData *XLogRecordAssemble(RmgrId rmid, uint8 info,
				   XLogRecPtr RedoRecPtr, bool doPageWrites,
				   XLogRecPtr *fpw_lsn, int *num_fpi)
{
	XLogRecData *rdt;
	uint32		total_len = 0;
	int			block_id;
	pg_crc32c	rdata_crc;
	registered_buffer *prev_regbuf = NULL;
	XLogRecData *rdt_datas_last;		// 尾指针
	XLogRecord *rechdr;
	char	   *scratch = hdr_scratch;

	/*
	 * Note: this function can be called multiple times for the same record.
	 * All the modifications we do to the rdata chains below must handle that.
	 */

	/* The record begins with the fixed-size header */
	rechdr = (XLogRecord *) scratch;
	scratch += SizeOfXLogRecord;

	hdr_rdt.next = NULL;
	rdt_datas_last = &hdr_rdt;
	hdr_rdt.data = hdr_scratch;

	// 用户可以通过wal_consistency_checking参数来控制事务日志是否记录一致性检查的信息
	// 在事务日志回放完毕后会检查数据页面的一致性
	if (wal_consistency_checking[rmid])
		info |= XLR_CHECK_CONSISTENCY;

	// 逐个处理XLogRegisterBuffer函数注册的各个Block
	*fpw_lsn = InvalidXLogRecPtr;
	for (block_id = 0; block_id < max_registered_block_id; block_id++)
	{
		registered_buffer *regbuf = &registered_buffers[block_id];
		bool		needs_backup;	
		bool		needs_data;
		XLogRecordBlockHeader bkpb;			// 通用的Block的Header信息
		XLogRecordBlockImageHeader bimg;	// 如果做FPW，则需要这个Header信息
		XLogRecordBlockCompressHeader cbimg = {0};	// 如果做FPW，且页面需要压缩
		bool		samerel;	// 日志记录的前一个页面是不是和本日志记录是同一个关系的
		bool		is_compressed = false; // 页面是否已经压缩
		bool		include_image;

		if (!regbuf->in_use)
			continue;

		// 是否需要做FPW，优先根据flag信息判断，否则根据GUC参数和是否处于backup判断，最后是依据LSN判断
		/* Determine if this block needs to be backed up */
		if (regbuf->flags & REGBUF_FORCE_IMAGE)
			needs_backup = true;
		else if (regbuf->flags & REGBUF_NO_IMAGE)
			needs_backup = false;
		else if (!doPageWrites)
			needs_backup = false;
		else {
			/* We assume page LSN is first data on *every* page that can be
			 * passed to XLogInsert, whether it has the standard page layout or not. */
			XLogRecPtr	page_lsn = PageGetLSN(regbuf->page);

			needs_backup = (page_lsn <= RedoRecPtr);
			if (!needs_backup) {
				if (*fpw_lsn == InvalidXLogRecPtr || page_lsn < *fpw_lsn)
					*fpw_lsn = page_lsn;
			}
		}

		// 是否保存页面数据
		/* Determine if the buffer data needs to included */
		if (regbuf->rdata_len == 0)   // 如果页面没有数据
			needs_data = false;
		else if ((regbuf->flags & REGBUF_KEEP_DATA) != 0)  // 页面明确指出了需要保存数据
			needs_data = true;
		else   // 如果没有指定，则根据是否做FPW来决定是否保存数据
			needs_data = !needs_backup;

		// 组装XLogRecordBlockHeader
		bkpb.id = block_id;
		bkpb.fork_flags = regbuf->forkno;
		bkpb.data_length = 0;

		if ((regbuf->flags & REGBUF_WILL_INIT) == REGBUF_WILL_INIT)
			bkpb.fork_flags |= BKPBLOCK_WILL_INIT;

		// 如果要做FPW，则需要保存页面的备份
		// 如果在回放时要检查日志的一致性，则需要做页面的备份
		/* If needs_backup is true or WAL checking is enabled for current
		 * resource manager, log a full-page write for the current block. */
		include_image = needs_backup || (info & XLR_CHECK_CONSISTENCY) != 0;

		if (include_image) {
			Page		page = regbuf->page;
			uint16		compressed_len = 0;

			// 标准页面中在pg_lower和pd_upper之间会有一个空洞，这部分没有数据，可以考虑裁剪掉，节约存储空间
			/* The page needs to be backed up, so calculate its hole length and offset.*/
			if (regbuf->flags & REGBUF_STANDARD) {
				/* Assume we can omit data between pd_lower and pd_upper */
				uint16		lower = ((PageHeader) page)->pd_lower;
				uint16		upper = ((PageHeader) page)->pd_upper;

				if (lower >= SizeOfPageHeaderData && upper > lower && upper <= BLCKSZ) {
					bimg.hole_offset = lower;
					cbimg.hole_length = upper - lower;
				} else {
					/* No "hole" to remove */
					bimg.hole_offset = 0;
					cbimg.hole_length = 0;
				}
			} else{
				/* Not a standard page header, don't try to eliminate "hole" */
				bimg.hole_offset = 0;
				cbimg.hole_length = 0;
			}

			// 如果开启了wal_compression参数，则会对保存进日志记录的数据页面进行压缩
			if (wal_compression) {
				is_compressed = XLogCompressBackupBlock(page, bimg.hole_offset,cbimg.hole_length,regbuf->compressed_page,&compressed_len);
			}

			/* Fill in the remaining fields in the XLogRecordBlockHeader struct */
			bkpb.fork_flags |= BKPBLOCK_HAS_IMAGE;

			/* Report a full page image constructed for the WAL record */
			*num_fpi += 1;

			/* Construct XLogRecData entries for the page content. */
			rdt_datas_last->next = &regbuf->bkp_rdatas[0];
			rdt_datas_last = rdt_datas_last->next;

			bimg.bimg_info = (cbimg.hole_length == 0) ? 0 : BKPIMAGE_HAS_HOLE;

			/*
			 * If WAL consistency checking is enabled for the resource manager
			 * of this WAL record, a full-page image is included in the record
			 * for the block modified. During redo, the full-page is replayed
			 * only if BKPIMAGE_APPLY is set. */
			if (needs_backup)
				bimg.bimg_info |= BKPIMAGE_APPLY;

			if (is_compressed)	// 如果是压缩页面，则空洞信息已经包含在其中
			{
				bimg.length = compressed_len;
				bimg.bimg_info |= BKPIMAGE_IS_COMPRESSED;

				rdt_datas_last->data = regbuf->compressed_page;
				rdt_datas_last->len = compressed_len;
			} else {
				bimg.length = BLCKSZ - cbimg.hole_length;

				if (cbimg.hole_length == 0) {  // 如果空洞长度是0，则直接记录整个页面
					rdt_datas_last->data = page;
					rdt_datas_last->len = BLCKSZ;
				} else {	// 如果未压缩且有空洞
					/* must skip the hole */
					rdt_datas_last->data = page;
					rdt_datas_last->len = bimg.hole_offset;

					rdt_datas_last->next = &regbuf->bkp_rdatas[1];
					rdt_datas_last = rdt_datas_last->next;

					rdt_datas_last->data =
						page + (bimg.hole_offset + cbimg.hole_length);
					rdt_datas_last->len =
						BLCKSZ - (bimg.hole_offset + cbimg.hole_length);
				}
			}

			total_len += bimg.length;
		}

		if (needs_data) {
			/* Link the caller-supplied rdata chain for this buffer to the overall list. */
			bkpb.fork_flags |= BKPBLOCK_HAS_DATA;
			bkpb.data_length = regbuf->rdata_len;
			total_len += regbuf->rdata_len;

			rdt_datas_last->next = regbuf->rdata_head;
			rdt_datas_last = regbuf->rdata_tail;
		}

		// 如果连续的两个日志都是同一个表中的日志记录，则可以省略一个filenode的空间
		if (prev_regbuf && RelFileNodeEquals(regbuf->rnode, prev_regbuf->rnode)) {
			samerel = true;
			bkpb.fork_flags |= BKPBLOCK_SAME_REL;
		} else
			samerel = false;
		prev_regbuf = regbuf;

		// 正式组装XLogRecordBlockHeader，复制多个Block相关的Header到hdr_scratch
		/* Ok, copy the header to the scratch buffer */
		// 1. 复制XLogRecordBlockHeader信息
		memcpy(scratch, &bkpb, SizeOfXLogRecordBlockHeader);
		scratch += SizeOfXLogRecordBlockHeader;
		if (include_image) {
			// 2. 复制LogRecordBlockImageHeader信息
			memcpy(scratch, &bimg, SizeOfXLogRecordBlockImageHeader);
			scratch += SizeOfXLogRecordBlockImageHeader;
			if (cbimg.hole_length != 0 && is_compressed) {
					// 3. 复制XLogRecordBlockImageHeader信息
				memcpy(scratch, &cbimg, SizeOfXLogRecordBlockCompressHeader);
				scratch += SizeOfXLogRecordBlockCompressHeader;
			}
		}
		// 是否可以节省一个filenode的空间
		if (!samerel) {
			memcpy(scratch, &regbuf->rnode, sizeof(RelFileNode));
			scratch += sizeof(RelFileNode);
		}
		memcpy(scratch, &regbuf->block, sizeof(BlockNumber));
		scratch += sizeof(BlockNumber);
	}

	/* followed by the record's origin, if any */
	if ((curinsert_flags & XLOG_INCLUDE_ORIGIN) && replorigin_session_origin != InvalidRepOriginId) {
		*(scratch++) = (char) XLR_BLOCK_ID_ORIGIN;
		memcpy(scratch, &replorigin_session_origin, sizeof(replorigin_session_origin));
		scratch += sizeof(replorigin_session_origin);
	}

	/* followed by toplevel XID, if not already included in previous record */
	if (IsSubTransactionAssignmentPending()) {
		TransactionId xid = GetTopTransactionIdIfAny();

		/* update the flag (later used by XLogResetInsertion) */
		XLogSetRecordFlags(XLOG_INCLUDE_XID);

		*(scratch++) = (char) XLR_BLOCK_ID_TOPLEVEL_XID;
		memcpy(scratch, &xid, sizeof(TransactionId));
		scratch += sizeof(TransactionId);
	}

	// 记录maindata的长度
	/* followed by main data, if any */
	if (mainrdata_len > 0) {
		if (mainrdata_len > 255) {
			*(scratch++) = (char) XLR_BLOCK_ID_DATA_LONG;
			memcpy(scratch, &mainrdata_len, sizeof(uint32));
			scratch += sizeof(uint32);
		} else {
			*(scratch++) = (char) XLR_BLOCK_ID_DATA_SHORT;
			*(scratch++) = (uint8) mainrdata_len;
		}
		rdt_datas_last->next = mainrdata_head;
		rdt_datas_last = mainrdata_last;
		total_len += mainrdata_len;
	}
	rdt_datas_last->next = NULL;

	hdr_rdt.len = (scratch - hdr_scratch);
	total_len += hdr_rdt.len;

	/* Calculate CRC of the data */
	INIT_CRC32C(rdata_crc);
	COMP_CRC32C(rdata_crc, hdr_scratch + SizeOfXLogRecord, hdr_rdt.len - SizeOfXLogRecord);
	for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
		COMP_CRC32C(rdata_crc, rdt->data, rdt->len);

	/* Fill in the fields in the record header. Prev-link is filled in later,
	 * once we know where in the WAL the record will be inserted. The CRC does
	 * not include the record header yet. */
	rechdr->xl_xid = GetCurrentTransactionIdIfAny();
	rechdr->xl_tot_len = total_len;
	rechdr->xl_info = info;
	rechdr->xl_rmid = rmid;
	rechdr->xl_prev = InvalidXLogRecPtr;
	rechdr->xl_crc = rdata_crc;

	return &hdr_rdt;
}

```
#### 日志的写入
`XLogRecordAssemble`函数能够将所注册的日志转换为二进制的形式，下一步就是写入WAL Buffer。PG将这个过程分为预留空间和数据复制两个步骤：
- 预留空间：前面日志已经完成组装，日志的长度已经确定，因此可以先计算WAL日志记录的长度，然后按照该长度从WAL Buffer中预留空间，空间预留的过程通过`XLogCtl->Insert->insertpos_lck`锁保护。也就是说，每个需要写入WAL日志记录的进程在预留空间时都是互斥的。
- 数据复制：将组装之后的数据复制到预留空间中，这个操作可以并发完成，由于每个进程都将WAL日志复制到自己预留的空间中，因此向WAL Buffer中复制的过程不会产生冲突。

```c++
// 预留空间函数实现源码
static void ReserveXLogInsertLocation(int size, XLogRecPtr *StartPos, XLogRecPtr *EndPos, XLogRecPtr *PrevPtr)
{
	XLogCtlInsert *Insert = &XLogCtl->Insert;
	uint64		startbytepos;
	uint64		endbytepos;
	uint64		prevbytepos;

	size = MAXALIGN(size);		// 预留空间大小

	SpinLockAcquire(&Insert->insertpos_lck);	// 加锁

	startbytepos = Insert->CurrBytePos;		// 预留的起点
	endbytepos = startbytepos + size;		// 预留的终点
	prevbytepos = Insert->PrevBytePos;
	Insert->CurrBytePos = endbytepos;
	Insert->PrevBytePos = startbytepos;

	SpinLockRelease(&Insert->insertpos_lck);	// 释放锁

	*StartPos = XLogBytePosToRecPtr(startbytepos);
	*EndPos = XLogBytePosToEndRecPtr(endbytepos);
	*PrevPtr = XLogBytePosToRecPtr(prevbytepos);
}
```

由于多个进程需要向同一个WAL Buffer中并发写入日志，所以这是数据库的一个性能瓶颈点，PG将预留空间和数据复制做分离就是一个非常重要的优化。在`XLogCtl->Insert->insertpos_lck`锁的保护下预留空间，保证了各个事务占用的空间不会重叠。一旦事务所占用的空间被预留，则数据复制的过程是可以并发的，通过`WALInsertLocks`锁来控制并发复制过程。
```c++
typedef struct
{
	LWLock		lock;	// 轻量锁，当锁释放时，代表WAL日志记录已经写入WAL buffer
	XLogRecPtr	insertingAt;	// 记录当前WAL日志记录刷入WAL buffer的进展
	XLogRecPtr	lastImportantAt;
} WALInsertLock;

/* Number of WAL insertion locks to use.*/
#define NUM_XLOGINSERT_LOCKS  8
```
PG声明了`NUM_XLOGINSERT_LOCKS`个`WALInsertLocks`，每个`WALInsertLocks`由“轻量锁+日志写入位置”组成。不同Backends（不同事务）在刷入日志时会随机获取一个`WALInsertLocks`。每个写入WAL日志的进程在复制数据时都需要申请一个这样的锁，虽然并发度受到`NUM_XLOGINSERT_LOCKS`的限制，但它仍能并发操作，可以提高并发写入的性能。

一个事务是否能将WAL Buffer中的数据刷入磁盘，取决于两个因素：
- 如果当前没有其他进程持有`WALInsertLocks`锁，就代表之前的WAL日志记录已经完成了数据复制
- 如果有其他进程获得了`WALInsertLocks`，但是它的`insertingAt`大于当前事务要刷入的LSN，则WAL Buffer刷入磁盘也没有问题。

```c++
XLogRecPtr XLogInsertRecord(XLogRecData *rdata, XLogRecPtr fpw_lsn, uint8 flags, int num_fpi)
{
	XLogCtlInsert *Insert = &XLogCtl->Insert;
	pg_crc32c	rdata_crc;
	bool		inserted;
	XLogRecord *rechdr = (XLogRecord *) rdata->data;
	uint8		info = rechdr->xl_info & ~XLR_INFO_MASK;
	bool		isLogSwitch = (rechdr->xl_rmid == RM_XLOG_ID && info == XLOG_SWITCH);
	XLogRecPtr	StartPos;
	XLogRecPtr	EndPos;
	bool		prevDoPageWrites = doPageWrites;

	/*----------
	 * We have now done all the preparatory work we can without holding a
	 * lock or modifying shared state. From here on, inserting the new WAL
	 * record to the shared WAL buffer cache is a two-step process:
	 *
	 * 1. Reserve the right amount of space from the WAL. The current head of
	 *	  reserved space is kept in Insert->CurrBytePos, and is protected by
	 *	  insertpos_lck.
	 *
	 * 2. Copy the record to the reserved WAL space. This involves finding the
	 *	  correct WAL buffer containing the reserved space, and copying the
	 *	  record in place. This can be done concurrently in multiple processes.
	 *---------- */
	START_CRIT_SECTION();
	// 如果进行WAL日志段的切换，则此时其他进程就不能预定空间
	if (isLogSwitch)
		WALInsertLockAcquireExclusive();
	else
		WALInsertLockAcquire();		// 获取WALInsertLocks锁

	if (RedoRecPtr != Insert->RedoRecPtr)
	{
		Assert(RedoRecPtr < Insert->RedoRecPtr);
		RedoRecPtr = Insert->RedoRecPtr;
	}
	doPageWrites = (Insert->fullPageWrites || Insert->forcePageWrites);

	if (doPageWrites &&
		(!prevDoPageWrites ||
		 (fpw_lsn != InvalidXLogRecPtr && fpw_lsn <= RedoRecPtr)))
	{
		/*
		 * Oops, some buffer now needs to be backed up that the caller didn't
		 * back up.  Start over.
		 */
		WALInsertLockRelease();
		END_CRIT_SECTION();
		return InvalidXLogRecPtr;
	}

	/*
	 * Reserve space for the record in the WAL. This also sets the xl_prev
	 * pointer.
	 */
	if (isLogSwitch)
		inserted = ReserveXLogSwitch(&StartPos, &EndPos, &rechdr->xl_prev);
	else {	// 预留空间
		ReserveXLogInsertLocation(rechdr->xl_tot_len, &StartPos, &EndPos, &rechdr->xl_prev);
		inserted = true;
	}

	if (inserted) {
		/* Now that xl_prev has been filled in, calculate CRC of the record header. */
		rdata_crc = rechdr->xl_crc;
		COMP_CRC32C(rdata_crc, rechdr, offsetof(XLogRecord, xl_crc));
		FIN_CRC32C(rdata_crc);
		rechdr->xl_crc = rdata_crc;

		// 将WAL日志记录写入复制到WAL Buffer
		CopyXLogRecordToWAL(rechdr->xl_tot_len, isLogSwitch, rdata, StartPos, EndPos);

		/* Unless record is flagged as not important, update LSN of last
		 * important record in the current slot. When holding all locks, just update the first one*/
		if ((flags & XLOG_MARK_UNIMPORTANT) == 0) {
			int			lockno = holdingAllLocks ? 0 : MyLockNo;
			WALInsertLocks[lockno].l.lastImportantAt = StartPos;
		}
	} else {
		/* This was an xlog-switch record, but the current insert location was
		 * already exactly at the beginning of a segment, so there was no need
		 * to do anything. */
	}

	/* Done! Let others know that we're finished. */
	WALInsertLockRelease();

	MarkCurrentTransactionIdLoggedIfAny();

	END_CRIT_SECTION();

	// ...

	/* Update our global variables */
	ProcLastRecPtr = StartPos;
	XactLastRecEnd = EndPos;

	/* Report WAL traffic to the instrumentation. */
	if (inserted) {
		pgWalUsage.wal_bytes += rechdr->xl_tot_len;
		pgWalUsage.wal_records++;
		pgWalUsage.wal_fpi += num_fpi;
	}

	return EndPos;
}
```

到这里，其实并没有真正完成日志的写入，因为只是写入了WAL Buffer中，并没有刷入磁盘中，当事务提交时，需要先调用`XLogFlush`完成刷盘，在这里会进行刷盘。调用栈如下：
```c++
XLogFlush(XLogRecPtr record) (src\backend\access\transam\xlog.c:2910)
RecordTransactionCommit() (src\backend\access\transam\xact.c:1403)
CommitTransaction() (src\backend\access\transam\xact.c:2190)
CommitTransactionCommand() (src\backend\access\transam\xact.c:2986)
finish_xact_command() (src\backend\tcop\postgres.c:2734)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1238)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (src\backend\tcop\postgres.c:4508)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4537)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4259)
ServerLoop() (src\backend\postmaster\postmaster.c:1745)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1417)
main(int argc, char ** argv) (src\backend\main\main.c:209)
```
函数实现如下：
```c++
void XLogFlush(XLogRecPtr record)
{
	XLogRecPtr	WriteRqstPtr;
	XLogwrtRqst WriteRqst;

	/* During REDO, we are reading not writing WAL.  Therefore, instead of
	 * trying to flush the WAL, we should update minRecoveryPoint instead. We
	 * test XLogInsertAllowed(), not InRecovery, because we need checkpointer
	 * to act this way too, and because when it tries to write the
	 * end-of-recovery checkpoint, it should indeed flush. */
	if (!XLogInsertAllowed()) {
		UpdateMinRecoveryPoint(record, false);
		return;
	}

	/* Quick exit if already known flushed */
	if (record <= LogwrtResult.Flush)
		return;

#ifdef WAL_DEBUG
	if (XLOG_DEBUG)
		elog(LOG, "xlog flush request %X/%X; write %X/%X; flush %X/%X", LSN_FORMAT_ARGS(record),LSN_FORMAT_ARGS(LogwrtResult.Write), LSN_FORMAT_ARGS(LogwrtResult.Flush));
#endif

	START_CRIT_SECTION();

	/* initialize to given target; may increase below */
	WriteRqstPtr = record;

	/* Now wait until we get the write lock, or someone else does the flush for us. */
	for (;;)
	{
		XLogRecPtr	insertpos;

		/* read LogwrtResult and update local state */
		SpinLockAcquire(&XLogCtl->info_lck);
		if (WriteRqstPtr < XLogCtl->LogwrtRqst.Write)
			WriteRqstPtr = XLogCtl->LogwrtRqst.Write;
		LogwrtResult = XLogCtl->LogwrtResult;
		SpinLockRelease(&XLogCtl->info_lck);

		/* done already? */
		if (record <= LogwrtResult.Flush)
			break;

		/* Before actually performing the write, wait for all in-flight insertions to the pages we're about to write to finish. */
		insertpos = WaitXLogInsertionsToFinish(WriteRqstPtr);

		/* Try to get the write lock. If we can't get it immediately, wait
		 * until it's released, and recheck if we still need to do the flush
		 * or if the backend that held the lock did it for us already. This
		 * helps to maintain a good rate of group committing when the system
		 * is bottlenecked by the speed of fsyncing. */
		if (!LWLockAcquireOrWait(WALWriteLock, LW_EXCLUSIVE))
		{
			/* The lock is now free, but we didn't acquire it yet. Before we do, loop back to check if someone else flushed the record for us already. */
			continue;
		}

		/* Got the lock; recheck whether request is satisfied */
		LogwrtResult = XLogCtl->LogwrtResult;
		if (record <= LogwrtResult.Flush) {
			LWLockRelease(WALWriteLock);
			break;
		}

		/* Sleep before flush! By adding a delay here, we may give further
		 * backends the opportunity to join the backlog of group commit
		 * followers; this can significantly improve transaction throughput,
		 * at the risk of increasing transaction latency. */
		if (CommitDelay > 0 && enableFsync && MinimumActiveBackends(CommitSiblings))
		{
			pg_usleep(CommitDelay);
			/* Re-check how far we can now flush the WAL. */
			insertpos = WaitXLogInsertionsToFinish(insertpos);
		}

		/* try to write/flush later additions to XLOG as well */
		WriteRqst.Write = insertpos;
		WriteRqst.Flush = insertpos;

		XLogWrite(WriteRqst, false);		// 具体向磁盘中的日志段文件写入

		LWLockRelease(WALWriteLock);
		/* done */
		break;
	}

	END_CRIT_SECTION();

	/* wake up walsenders now that we've released heavily contended locks */
	WalSndWakeupProcessRequests();

	if (LogwrtResult.Flush < record)
		elog(ERROR, "xlog flush request %X/%X is not satisfied --- flushed only to %X/%X", LSN_FORMAT_ARGS(record), LSN_FORMAT_ARGS(LogwrtResult.Flush));
}

```



#### WAL日志总结
关于是否需要记录整页的问题，WAL日志记录最终是要用来回放的，最容易想到的方法就是，记录每次修改后的页面，如果没有WAL日志的话，实际上就是需要将每次的页面刷盘的。在回放的时候，将页面还原。但这样处理有很大的问题，就是整页存储的代价很大，且有非常多重复的数据，为了提高性能，降低存储的代价，可以考虑仅在不得不存储页面的时候进行整页存储，其他情况下不用整页存储，比如，第一次向页面中插入数据，此时就必须存储整个页面，每次checkpoint后，也必须保存整个页面，否则在故障恢复进行WAL日志重放的时候，则没有足够的信息还原整个页面。记录页面的时候，因为页的布局特点，可将空洞进行压缩，不存储空洞，从而节约空间。