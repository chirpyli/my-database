### PostgreSQL源码分析——WAL日志清理
当数据库发生写操作时，会生成WAL日志，比如INSERT操作，那么什么时候会删除日志呢？这里我们分析其删除日志的逻辑。

#### 什么时候需要WAL日志？
在分析什么时候删日志之前，我们需要分析一下，什么时候需要用到WAL日志，反过来思考，就是在用WAL日志的时候，一定是不能被删除的，删除日志的判断逻辑中，必须要有相应的判断。
- 崩溃恢复，页未刷盘的WAL日志，必须保留，崩溃恢复时，读取WAL日志进行回放恢复。具体的就是checkpoint，pg_control文件中Redo重做点之后的日志都不能清理。
- 备机需要WAL日志进行回放。PG的逻辑是必须发送到备机才能删除，每个日志复制的槽位中记录的当前复制的最小LSN，大于最小LSN的日志需要保留，因为还没有被发送到备机。
- 归档进程需要WAL日志，将尚未归档的日志归档，还没有完成归档的日志，不能被删除。

#### 什么时候触发删除日志的操作？
我们想一下，之前分析过日志归档的逻辑，其中就日志归档完成才可以被删除，触发归档操作是生成WAL日志的时候，如果发生segment switch，那么就触发日志归档，而触发删除日志，则是checkpoint，因为checkpoint会将脏页刷盘持久化，就无需WAL日志了。所以，每次checkpoint操作的最后，会触发清理日志的操作。
```c++
void CreateCheckPoint(int flags)
{
	bool		shutdown;
	CheckPoint	checkPoint;
	XLogRecPtr	recptr;
	XLogSegNo	_logSegNo;
	XLogCtlInsert *Insert = &XLogCtl->Insert;
	uint32		freespace;
	XLogRecPtr	PriorRedoPtr;
	XLogRecPtr	curInsert;
	XLogRecPtr	last_important_lsn;
	VirtualTransactionId *vxids;
	int			nvxids;
	int			oldXLogAllowed = 0;

	// ...

	/* Delete old log files, those no longer needed for last checkpoint to
	 * prevent the disk holding the xlog from growing full. */
	// 从checkpoint的角度看，可以清理Redo重做点之前的日志，因为在这之前的脏页都已经刷盘了。
	XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size);	
	// 但是因为我们上面分析的原因，很多情况下，除了考虑主机，还要考虑备机的情况
	KeepLogSeg(recptr, &_logSegNo);  // recptr是当前最新的日志记录
	if (InvalidateObsoleteReplicationSlots(_logSegNo))
	{
		/* Some slots have been invalidated; recalculate the old-segment
		 * horizon, starting again from RedoRecPtr. */
		XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size);
		KeepLogSeg(recptr, &_logSegNo);
	}
	_logSegNo--;
	RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr, checkPoint.ThisTimeLineID);

	// ...
}
```

#### 源码分析
我们继续进行分析。checkpoint清理日志时，从重做点Redo之前的开始清理WAL日志，但是需要考虑到备机的情况，所以，需要进行判断。如果没有设置`max_slot_wal_keep_size`,`wal_keep_size`这两个参数，那么可以清理的位置就是需要确保满足可复制到备机的最小LSN位置之前的位置，再与checkpoint传进来的重做点相比较，那个更旧，则清理更旧之前的日志。

```c++
/*
 * Retreat *logSegNo to the last segment that we need to retain because of
 * either wal_keep_size or replication slots.
 *
 * This is calculated by subtracting wal_keep_size from the given xlog
 * location, recptr and by making sure that that result is below the
 * requirement of replication slots.  For the latter criterion we do consider
 * the effects of max_slot_wal_keep_size: reserve at most that much space back
 * from recptr.
 */
static void KeepLogSeg(XLogRecPtr recptr, XLogSegNo *logSegNo)
{
	XLogSegNo	currSegNo;
	XLogSegNo	segno;
	XLogRecPtr	keep;
	// 当前事务日志的“终点”
	XLByteToSeg(recptr, currSegNo, wal_segment_size);
	segno = currSegNo;

	/* Calculate how many segments are kept by slots first, adjusting for
	 * max_slot_wal_keep_size. */
	// 每个日志复制的槽位中都记录了当前复制的最小LSN，多个最小LSN会汇总出一个总的最小LSN
	// 大于这个LSN的日志还没有复制完成
	keep = XLogGetReplicationSlotMinimumLSN();
	if (keep != InvalidXLogRecPtr && keep < recptr)	 // 如果比最新日志旧
	{
		XLByteToSeg(keep, segno, wal_segment_size);

		/* Cap by max_slot_wal_keep_size ... */
		if (max_slot_wal_keep_size_mb >= 0)
		{
			uint64		slot_keep_segs;
			// 根据max_slot_wal_keep_size计算出的需要保留的日志段数量
			slot_keep_segs =
				ConvertToXSegs(max_slot_wal_keep_size_mb, wal_segment_size);
			// currSegNo - segno 表示当前日志段ID和可清理的日志段ID之间的日志段数量
			// 如果这个数量超过了max_slot_wal_keep_size的要求，则调整为按照max_slot_wal_keep_size要求保留的数量
			if (currSegNo - segno > slot_keep_segs)
				segno = currSegNo - slot_keep_segs;
		}
	}

	/* but, keep at least wal_keep_size if that's set */
	// 按照wal_keep_size参数再调整一次
	if (wal_keep_size_mb > 0)
	{
		uint64		keep_segs;

		keep_segs = ConvertToXSegs(wal_keep_size_mb, wal_segment_size);
		if (currSegNo - segno < keep_segs)
		{
			/* avoid underflow, don't go below 1 */
			if (currSegNo <= keep_segs)
				segno = 1;
			else
				segno = currSegNo - keep_segs;
		}
	}

	/* don't delete WAL segments newer than the calculated segment */
	// 根据日志复制槽最小LSN，以及max_slot_wal_keep_size,wal_keep_size两个参数的调整，
	// 如果比checkpoint输入的重做点记录还旧，那么更新为更旧的WAL日志段
	if (segno < *logSegNo)
		*logSegNo = segno;
}

```

具体删除WAL的源码，前面是确定了在那个位置之前的WAL日志可以被清理，下面的函数就是具体清理的过程。这其中，涉及日志归档，如果归档完成，则可以进行删除，如果没有完成归档，则不能删除日志。
```c++
/*
 * Recycle or remove all log files older or equal to passed segno.
 *
 * endptr is current (or recent) end of xlog, and lastredoptr is the
 * redo pointer of the last checkpoint. These are used to determine
 * whether we want to recycle rather than delete no-longer-wanted log files.
 *
 * insertTLI is the current timeline for XLOG insertion. Any recycled
 * segments should be reused for this timeline.
 */
static void
RemoveOldXlogFiles(XLogSegNo segno, XLogRecPtr lastredoptr, XLogRecPtr endptr,
				   TimeLineID insertTLI)
{
	DIR		   *xldir;
	struct dirent *xlde;
	char		lastoff[MAXFNAMELEN];
	XLogSegNo	endlogSegNo;
	XLogSegNo	recycleSegNo;

	/* Initialize info about where to try to recycle to */
	XLByteToSeg(endptr, endlogSegNo, wal_segment_size);
	recycleSegNo = XLOGfileslop(lastredoptr);

	/*
	 * Construct a filename of the last segment to be kept. The timeline ID
	 * doesn't matter, we ignore that in the comparison. (During recovery,
	 * InsertTimeLineID isn't set, so we can't use that.)
	 */
	XLogFileName(lastoff, 0, segno, wal_segment_size);

	elog(DEBUG2, "attempting to remove WAL segments older than log file %s",
		 lastoff);

	xldir = AllocateDir(XLOGDIR);

	while ((xlde = ReadDir(xldir, XLOGDIR)) != NULL)
	{
		/* Ignore files that are not XLOG segments */
		if (!IsXLogFileName(xlde->d_name) &&
			!IsPartialXLogFileName(xlde->d_name))
			continue;

		/*
		 * We ignore the timeline part of the XLOG segment identifiers in
		 * deciding whether a segment is still needed.  This ensures that we
		 * won't prematurely remove a segment from a parent timeline. We could
		 * probably be a little more proactive about removing segments of
		 * non-parent timelines, but that would be a whole lot more
		 * complicated.
		 *
		 * We use the alphanumeric sorting property of the filenames to decide
		 * which ones are earlier than the lastoff segment.
		 */
		if (strcmp(xlde->d_name + 8, lastoff + 8) <= 0)	// 比较文件，日志段文件名
		{
			if (XLogArchiveCheckDone(xlde->d_name))	// 如果归档完成
			{
				/* Update the last removed location in shared memory first */
				UpdateLastRemovedPtr(xlde->d_name);
				// 删日志文件
				RemoveXlogFile(xlde->d_name, recycleSegNo, &endlogSegNo,
							   insertTLI);
			}
		}
	}

	FreeDir(xldir);
}

```
其中用`XLogCtl->lastRemovedSegNo`记录删除的最新的日志段文件号
```c++
/*  Update the last removed segno pointer in shared memory, to reflect that the
 * given XLOG file has been removed. */
static void UpdateLastRemovedPtr(char *filename)
{
	uint32		tli;
	XLogSegNo	segno;

	XLogFromFileName(filename, &tli, &segno, wal_segment_size);

	SpinLockAcquire(&XLogCtl->info_lck);
	if (segno > XLogCtl->lastRemovedSegNo)
		XLogCtl->lastRemovedSegNo = segno;
	SpinLockRelease(&XLogCtl->info_lck);
}
```
删除日志后，需清理对于的归档相关的标识
```c++
/*
 * Recycle or remove a log file that's no longer needed.
 *
 * segname is the name of the segment to recycle or remove.  recycleSegNo
 * is the segment number to recycle up to.  endlogSegNo is the segment
 * number of the current (or recent) end of WAL.
 *
 * endlogSegNo gets incremented if the segment is recycled so as it is not
 * checked again with future callers of this function.
 *
 * insertTLI is the current timeline for XLOG insertion. Any recycled segments
 * should be used for this timeline.
 */
static void
RemoveXlogFile(const char *segname, XLogSegNo recycleSegNo, XLogSegNo *endlogSegNo, TimeLineID insertTLI)
{
	char		path[MAXPGPATH];
	struct stat statbuf;

	snprintf(path, MAXPGPATH, XLOGDIR "/%s", segname);

	/* Before deleting the file, see if it can be recycled as a future log
	 * segment. Only recycle normal files, because we don't want to recycle
	 * symbolic links pointing to a separate archive directory. */
	if (wal_recycle &&
		*endlogSegNo <= recycleSegNo &&
		XLogCtl->InstallXLogFileSegmentActive &&	/* callee rechecks this */
		lstat(path, &statbuf) == 0 && S_ISREG(statbuf.st_mode) &&
		InstallXLogFileSegment(endlogSegNo, path,
							   true, recycleSegNo, insertTLI))
	{
		ereport(DEBUG2, (errmsg_internal("recycled write-ahead log file \"%s\"", segname)));
		CheckpointStats.ckpt_segs_recycled++;
		/* Needn't recheck that slot on future iterations */
		(*endlogSegNo)++;
	} else {
		/* No need for any more future segments, or recycling failed ... */
		int			rc;
		ereport(DEBUG2,(errmsg_internal("removing write-ahead log file \"%s\"",segname)));
		rc = durable_unlink(path, LOG);
		if (rc != 0) {
			/* Message already logged by durable_unlink() */
			return;
		}
		CheckpointStats.ckpt_segs_removed++;
	}

	XLogArchiveCleanup(segname);	//移除.done文件
}

```