### PostgreSQL备机回放流程
前面我们分析了故障恢复的情况，这里，我们分析一下PostgreSQL一主一备，备机回放的流程。主备通过流复制协议进行WAL日志同步，主节点生成WAL日志，通过walwriter写入WAL日志段文件进行持久化保存，然后通知walsender进行发送。walsender将WAL日志通过流复制协议发送到备机，备节点通过walreceiver接收WAL日志，写入备机的pg_wal目录下的WAL日志段文件中，通知startup有新的WAL日志到达，然后startup进程读取WAL日志进行回放。大致的流程就是这样，下面我们分析一下其中的一些细节。

#### 备机回放主流程
备机回放的过程，就是通过backup_label获得重放的起点，或者通过读取pg_control获取重做点。然后从这个起点开始，读WAL日志回放，备机的话，是一直重复这个过程，如果没有WAL日志了，就等待WAL日志有效，然后继续读。

备机回放主流程如下：
```c++
StartupProcessMain(void)
--> StartupXLOG();
--> InitWalRecovery(ControlFile, &wasShutdown, &haveBackupLabel, &haveTblspcMap);
	--> readRecoverySignalFile();
	--> validateRecoveryParameters();
	--> XLogReaderAllocate(wal_segment_size, NULL, XL_ROUTINE(.page_read = &XLogPageRead, .segment_open = NULL, .segment_close = wal_segment_close), private);
	--> XLogPrefetcherAllocate(xlogreader);
	// 读backup_label的重做点，回放的起点
	--> read_backup_label(&CheckPointLoc, &CheckPointTLI, &backupEndRequired, &backupFromStandby)
	--> if (InRecovery)
		{
			// 回放WAL日志
			PerformWalRecovery();
		}
```

我们看一下`PerformWalRecovery`函数的实现，就是不断读WAL日志，然后进行回放。
```c++
void PerformWalRecovery(void)
{
	XLogRecord *record;

	// ...

	/* Find the first record that logically follows the checkpoint --- it might physically precede it, though. */
	if (RedoStartLSN < CheckPointLoc)
	{
		/* back up to find the record */
		replayTLI = RedoStartTLI;
		XLogPrefetcherBeginRead(xlogprefetcher, RedoStartLSN);
		record = ReadRecord(xlogprefetcher, PANIC, false, replayTLI);
	} else {
		/* just have to read next record after CheckPoint */
		Assert(xlogreader->ReadRecPtr == CheckPointLoc);
		replayTLI = CheckPointTLI;
		record = ReadRecord(xlogprefetcher, LOG, false, replayTLI);
	}

	if (record != NULL)
	{
		RmgrStartup();

		/* main redo apply loop */
		do
		{
			/* Apply the record */
			ApplyWalRecord(xlogreader, record, &replayTLI);

			/* Else, try to fetch the next WAL record */
			record = ReadRecord(xlogprefetcher, LOG, false, replayTLI);
		} while (record != NULL);

		// ...
	}

	// ...
}

// 执行回放
static void ApplyWalRecord(XLogReaderState *xlogreader, XLogRecord *record, TimeLineID *replayTLI)
{
	// ...
	/* Now apply the WAL record itself */
	GetRmgr(record->xl_rmid).rm_redo(xlogreader);
}
```

#### 读WAL日志
备机是不断读WAL日志，然后恢复WAL日志的过程，这里我们看一下读WAL日志的过程。对于备机，会一直尝试读取WAL日志，而不是返回NULL。
```c++
static XLogRecord *ReadRecord(XLogPrefetcher *xlogprefetcher, int emode, bool fetching_ckpt, TimeLineID replayTLI)
{
	XLogRecord *record;
	XLogReaderState *xlogreader = XLogPrefetcherGetReader(xlogprefetcher);
	XLogPageReadPrivate *private = (XLogPageReadPrivate *) xlogreader->private_data;
	// ...

	/* This is the first attempt to read this page. */
	lastSourceFailed = false;

	for (;;)
	{
		char	   *errormsg;

		record = XLogPrefetcherReadRecord(xlogprefetcher, &errormsg);
		// ...

		if (record)
		{
			/* Great, got a record */
			return record;
		}
		else 
		{
			// 如果没有读到WAL日志， 
			// 1. 如果处在崩溃恢复，不是归档状态，则尝试切换到归档状态，
			// 2. 如果是备机模式，则继续retry，不返回NULL
			/* No valid record available from this source */
			lastSourceFailed = true;

			/*
			 * If archive recovery was requested, but we were still doing
			 * crash recovery, switch to archive recovery and retry using the
			 * offline archive. We have now replayed all the valid WAL in
			 * pg_wal, so we are presumably now consistent.
			 *
			 * We require that there's at least some valid WAL present in
			 * pg_wal, however (!fetching_ckpt).  We could recover using the
			 * WAL from the archive, even if pg_wal is completely empty, but
			 * we'd have no idea how far we'd have to replay to reach
			 * consistency.  So err on the safe side and give up.
			 */
			if (!InArchiveRecovery && ArchiveRecoveryRequested &&
				!fetching_ckpt)
			{
				ereport(DEBUG1,
						(errmsg_internal("reached end of WAL in pg_wal, entering archive recovery")));
				InArchiveRecovery = true;
				if (StandbyModeRequested)
					EnableStandbyMode();

				SwitchIntoArchiveRecovery(xlogreader->EndRecPtr, replayTLI);
				minRecoveryPoint = xlogreader->EndRecPtr;
				minRecoveryPointTLI = replayTLI;

				CheckRecoveryConsistency();

				/* Before we retry, reset lastSourceFailed and currentSource so that we will check the archive next. */
				lastSourceFailed = false;
				currentSource = XLOG_FROM_ANY;

				continue;
			}

			/* In standby mode, loop back to retry. Otherwise, give up. */
			if (StandbyMode && !CheckForStandbyTrigger())
				continue;  // 备机模式，继续重试
			else
				return NULL;
		}
	}
}
```
`XLogPrefetcherReadRecord`会调用`XLogPageRead`读WAL日志。
```c++
// Read the XLOG page containing RecPtr into readBuf
int XLogPageRead(XLogReaderState *xlogreader, XLogRecPtr targetPagePtr, int reqLen, XLogRecPtr targetRecPtr, char *readBuf)
{
	XLogPageReadPrivate *private =
	(XLogPageReadPrivate *) xlogreader->private_data;
	int			emode = private->emode;
	uint32		targetPageOff;
	XLogSegNo	targetSegNo PG_USED_FOR_ASSERTS_ONLY;
	int			r;

	XLByteToSeg(targetPagePtr, targetSegNo, wal_segment_size);
	targetPageOff = XLogSegmentOffset(targetPagePtr, wal_segment_size);

	/* See if we need to switch to a new segment because the requested record is not in the currently open one. */
	if (readFile >= 0 && !XLByteInSeg(targetPagePtr, readSegNo, wal_segment_size))
	{
		/* Request a restartpoint if we've replayed too much xlog since the last one. */
		if (ArchiveRecoveryRequested && IsUnderPostmaster)
		{
			if (XLogCheckpointNeeded(readSegNo))
			{
				(void) GetRedoRecPtr();
				if (XLogCheckpointNeeded(readSegNo))
					RequestCheckpoint(CHECKPOINT_CAUSE_XLOG);
			}
		}

		close(readFile);
		readFile = -1;
		readSource = XLOG_FROM_ANY;
	}

	XLByteToSeg(targetPagePtr, readSegNo, wal_segment_size);

retry:
	/* See if we need to retrieve more data */
	if (readFile < 0 ||	(readSource == XLOG_FROM_STREAM && flushedUpto < targetPagePtr + reqLen))
	{
		if (readFile >= 0 && xlogreader->nonblocking && readSource == XLOG_FROM_STREAM && flushedUpto < targetPagePtr + reqLen)
			return XLREAD_WOULDBLOCK;
		// 等待WAL日志可用
		switch (WaitForWALToBecomeAvailable(targetPagePtr + reqLen,private->randAccess,private->fetching_ckpt,targetRecPtr,private->replayTLI,xlogreader->EndRecPtr,xlogreader->nonblocking))
		{
			case XLREAD_WOULDBLOCK:
				return XLREAD_WOULDBLOCK;
			case XLREAD_FAIL:
				if (readFile >= 0)
					close(readFile);
				readFile = -1;
				readLen = 0;
				readSource = XLOG_FROM_ANY;
				return XLREAD_FAIL;
			case XLREAD_SUCCESS:
				break;
		}
	}

	/* If the current segment is being streamed from the primary, calculate
	 * how much of the current page we have received already. We know the
	 * requested record has been received, but this is for the benefit of
	 * future calls, to allow quick exit at the top of this function. */
	if (readSource == XLOG_FROM_STREAM)
	{
		if (((targetPagePtr) / XLOG_BLCKSZ) != (flushedUpto / XLOG_BLCKSZ))
			readLen = XLOG_BLCKSZ;
		else
			readLen = XLogSegmentOffset(flushedUpto, wal_segment_size) - targetPageOff;
	}
	else
		readLen = XLOG_BLCKSZ;

	/* Read the requested page */
	readOff = targetPageOff;

	r = pg_pread(readFile, readBuf, XLOG_BLCKSZ, (off_t) readOff);  // 读日志段文件
	if (r != XLOG_BLCKSZ)
	{
		// ...
		goto next_record_is_invalid;
	}

	xlogreader->seg.ws_tli = curFileTLI;
	if (StandbyMode && !XLogReaderValidatePageHeader(xlogreader, targetPagePtr, readBuf))
	{
		// ...
		goto next_record_is_invalid;
	}

	return readLen;

next_record_is_invalid:
	if (xlogreader->nonblocking)
		return XLREAD_WOULDBLOCK;

	lastSourceFailed = true;

	if (readFile >= 0)
		close(readFile);
	readFile = -1;
	readLen = 0;
	readSource = XLOG_FROM_ANY;

	/* In standby-mode, keep trying */
	if (StandbyMode)
		goto retry;	// 备机模式，重试
	else
		return XLREAD_FAIL;
}
```

#### 日志来源
WAL日志的来源有三处，PG会根据当前的状态确定初始的日志源，当某个日志源的读取发生错误或有确切的信息需要切换日志源时，则根据状态机切换日志源。状态机如下，备机模式下，优先从归档日志读，否则从pg_wal目录读WAL日志，如果失败了，再从流复制去读，如果读STREAM还失败，就睡眠等待一段时间再重新尝试从归档和pg_wal日志读。
```c++
/*
 * Standby mode is implemented by a state machine:
 *
 * 1. Read from either archive or pg_wal (XLOG_FROM_ARCHIVE), or just pg_wal (XLOG_FROM_PG_WAL)
 * 2. Check trigger file
 * 3. Read from primary server via walreceiver (XLOG_FROM_STREAM)
 * 4. Rescan timelines
 * 5. Sleep wal_retrieve_retry_interval milliseconds, and loop back to 1.
 *
 * Failure to read from the current source advances the state machine to
 * the next state.
 */
typedef enum
{
	XLOG_FROM_ANY = 0,			/* request to read WAL from any source */
	XLOG_FROM_ARCHIVE,			/* restored using restore_command */  // 归档日志
	XLOG_FROM_PG_WAL,			/* existing file in pg_wal */         // pg_wal目录 
	XLOG_FROM_STREAM			/* streamed from primary */           // 流复制
} XLogSource;
```
读日志的调用栈如下：
```c++
WaitForWALToBecomeAvailable(XLogRecPtr RecPtr, _Bool randAccess, _Bool fetching_ckpt, XLogRecPtr tliRecPtr, TimeLineID replayTLI, XLogRecPtr replayLSN, _Bool nonblocking) (\src\backend\access\transam\xlogrecovery.c:3661)
XLogPageRead(XLogReaderState * xlogreader, XLogRecPtr targetPagePtr, int reqLen, XLogRecPtr targetRecPtr, char * readBuf) (\src\backend\access\transam\xlogrecovery.c:3244)
ReadPageInternal(XLogReaderState * state, XLogRecPtr pageptr, int reqLen) (\src\backend\access\transam\xlogreader.c:1032)
XLogDecodeNextRecord(XLogReaderState * state, _Bool nonblocking) (\src\backend\access\transam\xlogreader.c:570)
XLogReadAhead(XLogReaderState * state, _Bool nonblocking) (\src\backend\access\transam\xlogreader.c:945)
XLogPrefetcherNextBlock(uintptr_t pgsr_private, XLogRecPtr * lsn) (\src\backend\access\transam\xlogprefetcher.c:496)
lrq_prefetch(LsnReadQueue * lrq) (\src\backend\access\transam\xlogprefetcher.c:256)
lrq_complete_lsn(LsnReadQueue * lrq, XLogRecPtr lsn) (\src\backend\access\transam\xlogprefetcher.c:294)
XLogPrefetcherReadRecord(XLogPrefetcher * prefetcher, char ** errmsg) (\src\backend\access\transam\xlogprefetcher.c:1039)
ReadRecord(XLogPrefetcher * xlogprefetcher, int emode, _Bool fetching_ckpt, TimeLineID replayTLI) (\src\backend\access\transam\xlogrecovery.c:3047)
ReadCheckpointRecord(XLogPrefetcher * xlogprefetcher, XLogRecPtr RecPtr, int whichChkpt, _Bool report, TimeLineID replayTLI) (\src\backend\access\transam\xlogrecovery.c:3972)
InitWalRecovery(ControlFileData * ControlFile, _Bool * wasShutdown_ptr, _Bool * haveBackupLabel_ptr, _Bool * haveTblspcMap_ptr) (\src\backend\access\transam\xlogrecovery.c:628)
StartupXLOG() (\src\backend\access\transam\xlog.c:5082)
StartupProcessMain() (\src\backend\postmaster\startup.c:282)
AuxiliaryProcessMain(AuxProcType auxtype) (\src\backend\postmaster\auxprocess.c:141)
StartChildProcess(AuxProcType type) (\src\backend\postmaster\postmaster.c:5432)
PostmasterMain(int argc, char ** argv) (\src\backend\postmaster\postmaster.c:1473)
main(int argc, char ** argv) (\src\backend\main\main.c:202)
```
这里我们分析一下`WaitForWALToBecomeAvailable`：
```c++
static XLogPageReadResult WaitForWALToBecomeAvailable(XLogRecPtr RecPtr, bool randAccess,bool fetching_ckpt, XLogRecPtr tliRecPtr,TimeLineID replayTLI, XLogRecPtr replayLSN,bool nonblocking)
{
	// 如果是主备模式，则日志源优先选择归档日志，否则考虑从pg_wal目录获取日志
	if (!InArchiveRecovery)
		currentSource = XLOG_FROM_PG_WAL;
	else if (currentSource == XLOG_FROM_ANY || (!StandbyMode && currentSource == XLOG_FROM_STREAM))
	{
		lastSourceFailed = false;
		currentSource = XLOG_FROM_ARCHIVE;
	}

	for (;;)
	{
		XLogSource	oldSource = currentSource;
		bool		startWalReceiver = false;

		// 如果日志在读取的过程中发生了错误，则考虑开始切换日志源
		if (lastSourceFailed)
		{
			/* Don't allow any retry loops to occur during nonblocking readahead.  
			   Let the caller process everything that has been decoded already first. */
			if (nonblocking)
				return XLREAD_WOULDBLOCK;

			switch (currentSource)
			{
				case XLOG_FROM_ARCHIVE:
				case XLOG_FROM_PG_WAL:
					// 备机遇到trigger，考虑关闭walreceiver进程，升主机
					if (StandbyMode && CheckForStandbyTrigger()) {
						XLogShutdownWalRcv();
						return XLREAD_FAIL;
					}

					/*Not in standby mode, and we've now tried the archive and pg_wal. */
					if (!StandbyMode)
						return XLREAD_FAIL;

					/* Move to XLOG_FROM_STREAM state, and set to start a walreceiver if necessary. */
					currentSource = XLOG_FROM_STREAM; // 日志源切换成STREAM
					startWalReceiver = true;
					break;

				case XLOG_FROM_STREAM:
					// 如果流复制读取WAL失败，则考虑关闭walreceiver进程，重新从归档日志/pg_wal目录读，
					/* Failure while streaming.  So, we treat that the
					  same as disconnection, and retry from archive/pg_wal again.  */

					/* We should be able to move to XLOG_FROM_STREAM only in standby mode. */
					Assert(StandbyMode);

					/*Before we leave XLOG_FROM_STREAM state, make sure that
					 * walreceiver is not active, so that it won't overwrite
					 * WAL that we restore from archive. */
					XLogShutdownWalRcv();

					/* Before we sleep, re-scan for possible new timelines if
					 * we were requested to recover to the latest timeline. */
					if (recoveryTargetTimeLineGoal == RECOVERY_TARGET_TIMELINE_LATEST){
						if (rescanLatestTimeLine(replayTLI, replayLSN)){
							currentSource = XLOG_FROM_ARCHIVE;
							break;
						}
					}

					/* XLOG_FROM_STREAM is the last state in our state
					 * machine, so we've exhausted all the options for
					 * obtaining the requested WAL. We're going to loop back
					 * and retry from the archive, but if it hasn't been long
					 * since last attempt, sleep wal_retrieve_retry_interval
					 * milliseconds to avoid busy-waiting. */
					now = GetCurrentTimestamp();
					if (!TimestampDifferenceExceeds(last_fail_time, now,wal_retrieve_retry_interval))
					{
						long		wait_time;

						wait_time = wal_retrieve_retry_interval - TimestampDifferenceMilliseconds(last_fail_time, now);
						elog(LOG, "waiting for WAL to become available at %X/%X", LSN_FORMAT_ARGS(RecPtr));

						/* Do background tasks that might benefit us later. */
						KnownAssignedTransactionIdsIdleMaintenance();

						(void) WaitLatch(&XLogRecoveryCtl->recoveryWakeupLatch, WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH, wait_time, WAIT_EVENT_RECOVERY_RETRIEVE_RETRY_INTERVAL);
						ResetLatch(&XLogRecoveryCtl->recoveryWakeupLatch);
						now = GetCurrentTimestamp();

						/* Handle interrupt signals of startup process */
						HandleStartupProcInterrupts();
					}
					last_fail_time = now;
					currentSource = XLOG_FROM_ARCHIVE;
					break;

				default:
					elog(ERROR, "unexpected WAL source %d", currentSource);
			}
		} else if (currentSource == XLOG_FROM_PG_WAL)
		{
			/* We just successfully read a file in pg_wal. We prefer files in
			 * the archive over ones in pg_wal, so try the next file again from the archive first. */
			if (InArchiveRecovery)
				currentSource = XLOG_FROM_ARCHIVE;
		}

		if (currentSource != oldSource)
			elog(DEBUG2, "switched WAL source from %s to %s after %s",
				 xlogSourceNames[oldSource], xlogSourceNames[currentSource],
				 lastSourceFailed ? "failure" : "success");

		/* We've now handled possible failure. Try to read from the chosen source. */
		lastSourceFailed = false;

		switch (currentSource)
		{
			case XLOG_FROM_ARCHIVE:
			case XLOG_FROM_PG_WAL:
				/* Close any old file we might have open. */
				if (readFile >= 0) {
					close(readFile);
					readFile = -1;
				}
				/* Reset curFileTLI if random fetch. */
				if (randAccess)
					curFileTLI = 0;

				/* Try to restore the file from archive, or read an existing file from pg_wal. */
				readFile = XLogFileReadAnyTLI(readSegNo, DEBUG2, currentSource == XLOG_FROM_ARCHIVE ? XLOG_FROM_ANY : currentSource);
				if (readFile >= 0)
					return XLREAD_SUCCESS;	/* success! */

				/* Nope, not found in archive or pg_wal. */
				lastSourceFailed = true;
				break;

			case XLOG_FROM_STREAM:
				{
					bool		havedata;

					/* We should be able to move to XLOG_FROM_STREAM only in standby mode. */
					Assert(StandbyMode);

					/*
					 * First, shutdown walreceiver if its restart has been
					 * requested -- but no point if we're already slated for
					 * starting it.
					 */
					if (pendingWalRcvRestart && !startWalReceiver)
					{
						XLogShutdownWalRcv();

						/*
						 * Re-scan for possible new timelines if we were
						 * requested to recover to the latest timeline.
						 */
						if (recoveryTargetTimeLineGoal ==
							RECOVERY_TARGET_TIMELINE_LATEST)
							rescanLatestTimeLine(replayTLI, replayLSN);

						startWalReceiver = true;
					}
					pendingWalRcvRestart = false;

					/*
					 * Launch walreceiver if needed.
					 *
					 * If fetching_ckpt is true, RecPtr points to the initial
					 * checkpoint location. In that case, we use RedoStartLSN
					 * as the streaming start position instead of RecPtr, so
					 * that when we later jump backwards to start redo at
					 * RedoStartLSN, we will have the logs streamed already.
					 */
					if (startWalReceiver &&
						PrimaryConnInfo && strcmp(PrimaryConnInfo, "") != 0)
					{
						XLogRecPtr	ptr;
						TimeLineID	tli;

						if (fetching_ckpt)
						{
							ptr = RedoStartLSN;
							tli = RedoStartTLI;
						}
						else
						{
							ptr = RecPtr;

							/*
							 * Use the record begin position to determine the
							 * TLI, rather than the position we're reading.
							 */
							tli = tliOfPointInHistory(tliRecPtr, expectedTLEs);

							if (curFileTLI > 0 && tli < curFileTLI)
								elog(ERROR, "according to history file, WAL location %X/%X belongs to timeline %u, but previous recovered WAL file came from timeline %u",
									 LSN_FORMAT_ARGS(tliRecPtr),
									 tli, curFileTLI);
						}
						curFileTLI = tli;
						SetInstallXLogFileSegmentActive();
						RequestXLogStreaming(tli, ptr, PrimaryConnInfo,
											 PrimarySlotName,
											 wal_receiver_create_temp_slot);
						flushedUpto = 0;
					}

					/*
					 * Check if WAL receiver is active or wait to start up.
					 */
					if (!WalRcvStreaming())
					{
						lastSourceFailed = true;
						break;
					}

					/*
					 * Walreceiver is active, so see if new data has arrived.
					 *
					 * We only advance XLogReceiptTime when we obtain fresh
					 * WAL from walreceiver and observe that we had already
					 * processed everything before the most recent "chunk"
					 * that it flushed to disk.  In steady state where we are
					 * keeping up with the incoming data, XLogReceiptTime will
					 * be updated on each cycle. When we are behind,
					 * XLogReceiptTime will not advance, so the grace time
					 * allotted to conflicting queries will decrease.
					 */
					if (RecPtr < flushedUpto)
						havedata = true;
					else
					{
						XLogRecPtr	latestChunkStart;

						flushedUpto = GetWalRcvFlushRecPtr(&latestChunkStart, &receiveTLI);
						if (RecPtr < flushedUpto && receiveTLI == curFileTLI)
						{
							havedata = true;
							if (latestChunkStart <= RecPtr)
							{
								XLogReceiptTime = GetCurrentTimestamp();
								SetCurrentChunkStartTime(XLogReceiptTime);
							}
						}
						else
							havedata = false;
					}
					if (havedata)
					{
						/*
						 * Great, streamed far enough.  Open the file if it's
						 * not open already.  Also read the timeline history
						 * file if we haven't initialized timeline history
						 * yet; it should be streamed over and present in
						 * pg_wal by now.  Use XLOG_FROM_STREAM so that source
						 * info is set correctly and XLogReceiptTime isn't
						 * changed.
						 *
						 * NB: We must set readTimeLineHistory based on
						 * recoveryTargetTLI, not receiveTLI. Normally they'll
						 * be the same, but if recovery_target_timeline is
						 * 'latest' and archiving is configured, then it's
						 * possible that we managed to retrieve one or more
						 * new timeline history files from the archive,
						 * updating recoveryTargetTLI.
						 */
						if (readFile < 0)
						{
							if (!expectedTLEs)
								expectedTLEs = readTimeLineHistory(recoveryTargetTLI);
							readFile = XLogFileRead(readSegNo, PANIC,receiveTLI,XLOG_FROM_STREAM, false);
							Assert(readFile >= 0);
						}
						else
						{
							/* just make sure source info is correct... */
							readSource = XLOG_FROM_STREAM;
							XLogReceiptSource = XLOG_FROM_STREAM;
							return XLREAD_SUCCESS;
						}
						break;
					}

					/* In nonblocking mode, return rather than sleeping. */
					if (nonblocking)
						return XLREAD_WOULDBLOCK;

					/* Data not here yet. Check for trigger, then wait for
					 * walreceiver to wake us up when new WAL arrives. */
					if (CheckForStandbyTrigger())
					{
						/*
						 * Note that we don't return XLREAD_FAIL immediately
						 * here. After being triggered, we still want to
						 * replay all the WAL that was already streamed. It's
						 * in pg_wal now, so we just treat this as a failure,
						 * and the state machine will move on to replay the
						 * streamed WAL from pg_wal, and then recheck the
						 * trigger and exit replay. */
						lastSourceFailed = true;
						break;
					}

					/* Since we have replayed everything we have received so
					 * far and are about to start waiting for more WAL, let's
					 * tell the upstream server our replay location now so
					 * that pg_stat_replication doesn't show stale information. */
					if (!streaming_reply_sent)
					{
						WalRcvForceReply();
						streaming_reply_sent = true;
					}

					/* Do any background tasks that might benefit us later. */
					KnownAssignedTransactionIdsIdleMaintenance();

					/* Update pg_stat_recovery_prefetch before sleeping. */
					XLogPrefetcherComputeStats(xlogprefetcher);

					/* Wait for more WAL to arrive. Time out after 5 seconds
					 * to react to a trigger file promptly and to check if the
					 * WAL receiver is still active. */
					(void) WaitLatch(&XLogRecoveryCtl->recoveryWakeupLatch,
									 WL_LATCH_SET | WL_TIMEOUT |
									 WL_EXIT_ON_PM_DEATH,
									 5000L, WAIT_EVENT_RECOVERY_WAL_STREAM);
					ResetLatch(&XLogRecoveryCtl->recoveryWakeupLatch);
					break;
				}

			default:
				elog(ERROR, "unexpected WAL source %d", currentSource);
		}

		/* Check for recovery pause here so that we can confirm more quickly
		 * that a requested pause has actually taken effect. */
		if (((volatile XLogRecoveryCtlData *) XLogRecoveryCtl)->recoveryPauseState != RECOVERY_NOT_PAUSED)
			recoveryPausesHere(false);

		/* This possibly-long loop needs to handle interrupts of startup process. */
		HandleStartupProcInterrupts();
	}

	return XLREAD_FAIL;			/* not reached */
}
```




#### 什么时候启动walreceiver进程？
什么时候启动walreceiver进程呢？自然是需要WAL日志的时候，回放的过程需要读WAL日志，这个时候会调用`WaitForWALToBecomeAvailable`等待可用的WAL日志。

如果日志来源是流复制，则发送信号启动walreceiver进程，然后由walreceiver进程去连接主节点，主节点生成walsender进程握手发送数据。

```c++
WaitForWALToBecomeAvailable(targetPagePtr + reqLen,private->randAccess,private->fetching_ckpt,targetRecPtr,private->replayTLI,xlogreader->EndRecPtr,xlogreader->nonblocking)
--> RequestXLogStreaming(tli, ptr, PrimaryConnInfo, PrimarySlotName,wal_receiver_create_temp_slot)
	--> SendPostmasterSignal(PMSIGNAL_START_WALRECEIVER)	// 发送信号，启动walreceiver进程
```


```c++
void PostmasterMain(int argc, char *argv[])
{
	// 注册信号处理函数
	pqsignal_pm(SIGUSR1, sigusr1_handler);	/* message from child process */
}

/* sigusr1_handler - handle signal conditions from child processes */
static void sigusr1_handler(SIGNAL_ARGS)
{
	// ...

	if (CheckPostmasterSignal(PMSIGNAL_START_WALRECEIVER))
	{
		/* Startup Process wants us to start the walreceiver process. */
		/* Start immediately if possible, else remember request for later. */
		WalReceiverRequested = true;
		MaybeStartWalReceiver();	// 启动walreceiver进程
	}

	// ...
}
```
