## checkpoint源码分析

因为检查点checkpoint相关的代码不是一篇文章就能分析完的，所以，相关的代码与逻辑可能会不连续，需要后续结合上下文理解。这里只列出了其中一部分。
其核心代码在：`src/backend/postmaster/checkpointer.c`以及`src/backend/access/transam/xlog.c` 中。

### 为什么会有checkpoint
数据库最核心的功能就是存数据，查询数据。而其中存数据主要涉及建表以及INSERT语句，在向表插入数据时，正常的流程是用户输入一条SQL，数据库经过语法解析，语义分析，最后执行插入操作，而插入则是要构造一条元组tuple，插入到空闲页page中，插入过程需要先在Buffer中查找有没有指定的空闲页，没有的话要从磁盘中读入页到Buffer中，然后将插入新元组后的页标记为脏页，进行刷盘（持久化），然后事务才返回插入成功。为了提升性能，有了WAL，即，每次不再直接刷盘，而是先将对于的修改信息写入WAL日志，然后由后台进程异步的批量的进行刷盘操作，以提高性能。因此磁盘的特点，顺序访问相比随机访问性能要高，所以，通过WAL的方式将大量的刷盘（随机访问）转为了WAL顺序写入。如果中间数据库崩溃了，重启数据库后，可通过回放WAL日志进行恢复，因为WAL记录了每次的修改信息。对此，就有了新问题了，从哪里进行回放？如何降低崩溃恢复的时间？这就是checkpoint要解决的问题。

### checkpoint触发的时机
PG中有单独的检查点进程执行checkpoint，那么什么时候会触发checkpint呢？
```
postgres@slpc:~$ ps -ef | grep postgres
postgres  942757       1  0 11:00 ?        00:00:00 /home/postgres/pgsql/bin/postgres -D data
postgres  942759  942757  0 11:00 ?        00:00:00 postgres: logger 
postgres  942764  942757  0 11:00 ?        00:00:00 postgres: checkpointer   检查点进程
postgres  942765  942757  0 11:00 ?        00:00:00 postgres: background writer 
postgres  942766  942757  0 11:00 ?        00:00:00 postgres: walwriter 
postgres  942767  942757  0 11:00 ?        00:00:00 postgres: stats collector 
postgres  942769  942757  0 11:00 ?        00:00:00 postgres: logical replication launcher 
```
在配置文件postgresql.conf中，涉及到checkpoint的参数如下：
```shell
# - Checkpoints -
checkpoint_timeout = 600min             # range 30s-1d
#checkpoint_completion_target = 0.9     # checkpoint target duration, 0.0 - 1.0
#checkpoint_flush_after = 256kB         # measured in pages, 0 disables
#checkpoint_warning = 30s               # 0 disables
max_wal_size = 1GB
min_wal_size = 80MB
```
所以，可以通过参数来控制何时触发：
- 超过了`checkpoint_timeout`参数设置的时间即触发checkpoint
- 超过了`max_wal_size`设置的最大的WAL日志size， 触发checkpoint。
其逻辑也较为容易理解，间隔一段时间就要执行一次checkpoint，超过一定大小，就要执行checkpoint，不然WAL日志会不断增长，必须将大小设置在一定范围内。

另外， checkpoint还可通过执行`checkpoint`语句触发。


还有一种情况，以smart或fast模式关闭PostgreSQL时触发。PG源码注释中有如下的解释：
` The checkpointer is started by the postmaster as soon as the startup subprocess finishes, or as soon as recovery begins if we are doing archive recovery.  It remains alive until the postmaster commands it to terminate. Normal termination is by SIGUSR2, which instructs the checkpointer to execute a shutdown checkpoint and then exit(0)`
当然异常退出的时候，下面的代码注释中也给予了解释：
```c++
/*
 * The checkpointer is new as of Postgres 9.2.  It handles all checkpoints.
 * Checkpoints are automatically dispatched after a certain amount of time has
 * elapsed since the last one, and it can be signaled to perform requested
 * checkpoints as well. 
 *
 * The checkpointer is started by the postmaster as soon as the startup
 * subprocess finishes, or as soon as recovery begins if we are doing archive
 * recovery.  It remains alive until the postmaster commands it to terminate.
 * Normal termination is by SIGUSR2, which instructs the checkpointer to
 * execute a shutdown checkpoint and then exit(0).  (All backends must be
 * stopped before SIGUSR2 is issued!)  Emergency termination is by SIGQUIT;
 * like any backend, the checkpointer will simply abort and exit on SIGQUIT.
 *
 * If the checkpointer exits unexpectedly, the postmaster treats that the same
 * as a backend crash: shared memory may be corrupted, so remaining backends
 * should be killed by SIGQUIT and then a recovery cycle started.  (Even if
 * shared memory isn't corrupted, we have lost information about which
 * files need to be fsync'd for the next checkpoint, and so a system
 * restart needs to be forced.)
 */
```

### checkpointer进程
在节点启动时，会创建检查点进程，该进程启动后会进入一个无限循环，一直执行，直到数据库被关闭或遇到错误崩溃。在循环中，进程会不断的检查`((volatile CheckpointerShmemStruct *) CheckpointerShmem)->ckpt_flags`，根据`ckpt_flags`去做对应的动作。其他进程会修改`ckpt_flags`，让checkpointer去做对应的操作。其设计的逻辑是通过共享内存实现checkpointer进程与其他进程进行交互。
```c++
typedef struct
{
	pid_t		checkpointer_pid;	/* PID (0 if not started) */

	slock_t		ckpt_lck;		/* protects all the ckpt_* fields */  // 锁，多进程同步处理用

	int			ckpt_started;	/* advances when checkpoint starts */
	int			ckpt_done;		/* advances when checkpoint done */
	int			ckpt_failed;	/* advances when checkpoint fails */

	int			ckpt_flags;		/* checkpoint flags, defined in xlog.h */ // 检查点flag，这个要重点看

	ConditionVariable start_cv; /* signaled when ckpt_started advances */
	ConditionVariable done_cv;	/* signaled when ckpt_done advances */

	uint32		num_backend_writes; /* counts user backend buffer writes */
	uint32		num_backend_fsync;	/* counts user backend fsync calls */

	int			num_requests;	/* current # of requests */
	int			max_requests;	/* allocated array size */
	CheckpointerRequest requests[FLEXIBLE_ARRAY_MEMBER];
} CheckpointerShmemStruct;

static CheckpointerShmemStruct *CheckpointerShmem;
```
检查点flag定义再`xlog.h`中，具体代码如下：
```c++
/*
 * OR-able request flag bits for checkpoints.  The "cause" bits are used only
 * for logging purposes.  Note: the flags must be defined so that it's
 * sensible to OR together request flags arising from different requestors.
 */

/* These directly affect the behavior of CreateCheckPoint and subsidiaries */
#define CHECKPOINT_IS_SHUTDOWN	0x0001	/* Checkpoint is for shutdown */
#define CHECKPOINT_END_OF_RECOVERY	0x0002	/* Like shutdown checkpoint, but issued at end of WAL recovery */
#define CHECKPOINT_IMMEDIATE	0x0004	/* Do it without delays */
#define CHECKPOINT_FORCE		0x0008	/* Force even if no activity */
#define CHECKPOINT_FLUSH_ALL	0x0010	/* Flush all pages, including those belonging to unlogged tables */
/* These are important to RequestCheckpoint */
#define CHECKPOINT_WAIT			0x0020	/* Wait for completion */
#define CHECKPOINT_REQUESTED	0x0040	/* Checkpoint request has been made */
/* These indicate the cause of a checkpoint request */
#define CHECKPOINT_CAUSE_XLOG	0x0080	/* XLOG consumption */
#define CHECKPOINT_CAUSE_TIME	0x0100	/* Elapsed time */
```

checkpointer进程代码如下：
```c++
/*
 * Main entry point for checkpointer process
 *
 * This is invoked from AuxiliaryProcessMain, which has already created the
 * basic execution environment, but not enabled signals yet.
 */
void CheckpointerMain(void)
{
    // 注册信号处理函数 
    pqsignal(SIGINT, ReqCheckpointHandler); 
    // ...

	if (sigsetjmp(local_sigjmp_buf, 1) != 0)
	{
        // 错误处理 ...
    }

    /* Loop forever */
	for (;;)
	{
		bool		do_checkpoint = false;
		int			flags = 0;
		pg_time_t	now;
		int			elapsed_secs;
		int			cur_timeout;

		/* Clear any already-pending wakeups */
		ResetLatch(MyLatch);

		/* Process any requests or signals received recently. */
		AbsorbSyncRequests();
		HandleCheckpointerInterrupts();

		/* Detect a pending checkpoint request by checking whether the flags
		 * word in shared memory is nonzero.  We shouldn't need to acquire the
		 * ckpt_lck for this. */
		if (((volatile CheckpointerShmemStruct *) CheckpointerShmem)->ckpt_flags) 
		{
            // 判断ckpt_flags，这个标识在手动执行checkpoint时等情况下会在其他进程中被修改
			do_checkpoint = true;
			BgWriterStats.m_requested_checkpoints++;
		}

		/* Force a checkpoint if too much time has elapsed since the last one.
		 * Note that we count a timed checkpoint in stats only when this
		 * occurs without an external request, but we set the CAUSE_TIME flag
		 * bit even if there is also an external request. */
		now = (pg_time_t) time(NULL);
		elapsed_secs = now - last_checkpoint_time;
		if (elapsed_secs >= CheckPointTimeout)   // checkpoint_timeout是否超时
		{
			if (!do_checkpoint)
				BgWriterStats.m_timed_checkpoints++;
			do_checkpoint = true;
			flags |= CHECKPOINT_CAUSE_TIME;
		}

		/* Do a checkpoint if requested. */
		if (do_checkpoint)
		{
			bool		ckpt_performed = false;
			bool		do_restartpoint;

			/* Check if we should perform a checkpoint or a restartpoint. As a
			 * side-effect, RecoveryInProgress() initializes TimeLineID if it's not set yet.*/
			do_restartpoint = RecoveryInProgress();

			/*
			 * Atomically fetch the request flags to figure out what kind of a
			 * checkpoint we should perform, and increase the started-counter
			 * to acknowledge that we've started a new checkpoint.
			 */
			SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
			flags |= CheckpointerShmem->ckpt_flags;
			CheckpointerShmem->ckpt_flags = 0;  // 已将do_checkpoint置为true，重设置ckpt_flags为0
			CheckpointerShmem->ckpt_started++;
			SpinLockRelease(&CheckpointerShmem->ckpt_lck);

			ConditionVariableBroadcast(&CheckpointerShmem->start_cv);

			/* The end-of-recovery checkpoint is a real checkpoint that's performed while we're still in recovery. */
			if (flags & CHECKPOINT_END_OF_RECOVERY)
				do_restartpoint = false;

			/* We will warn if (a) too soon since last checkpoint (whatever
			 * caused it) and (b) somebody set the CHECKPOINT_CAUSE_XLOG flag
			 * since the last checkpoint start.  Note in particular that this
			 * implementation will not generate warnings caused by CheckPointTimeout < CheckPointWarning. */
			if (!do_restartpoint && (flags & CHECKPOINT_CAUSE_XLOG) && elapsed_secs < CheckPointWarning)
				ereport(LOG,
						(errmsg_plural("checkpoints are occurring too frequently (%d second apart)",
									   "checkpoints are occurring too frequently (%d seconds apart)",
									   elapsed_secs,
									   elapsed_secs),
						 errhint("Consider increasing the configuration parameter \"max_wal_size\".")));

			/* Initialize checkpointer-private variables used during checkpoint.*/
			ckpt_active = true;
			if (do_restartpoint)
				ckpt_start_recptr = GetXLogReplayRecPtr(NULL);
			else
				ckpt_start_recptr = GetInsertRecPtr();
			ckpt_start_time = now;
			ckpt_cached_elapsed = 0;

			/* Do the checkpoint. */
			if (!do_restartpoint)
			{
				CreateCheckPoint(flags);    // 执行checkpoint
				ckpt_performed = true;
			}
			else
				ckpt_performed = CreateRestartPoint(flags); 

			/* After any checkpoint, close all smgr files.  This is so we
			 * won't hang onto smgr references to deleted files indefinitely.*/
			smgrcloseall();

			/* Indicate checkpoint completion to any waiting backends. */
            // 通知等待的进程，checkpoint完成
			SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
			CheckpointerShmem->ckpt_done = CheckpointerShmem->ckpt_started;
			SpinLockRelease(&CheckpointerShmem->ckpt_lck);

			ConditionVariableBroadcast(&CheckpointerShmem->done_cv);

			if (ckpt_performed)
			{
				/*
				 * Note we record the checkpoint start time not end time as
				 * last_checkpoint_time.  This is so that time-driven
				 * checkpoints happen at a predictable spacing.
				 */
				last_checkpoint_time = now;
			}
			else
			{
				/*
				 * We were not able to perform the restartpoint (checkpoints
				 * throw an ERROR in case of error).  Most likely because we
				 * have not received any new checkpoint WAL records since the
				 * last restartpoint. Try again in 15 s. */
				last_checkpoint_time = now - CheckPointTimeout + 15;
			}

			ckpt_active = false;

			/* We may have received an interrupt during the checkpoint. */
			HandleCheckpointerInterrupts();
		}

		/* Check for archive_timeout and switch xlog files if necessary. */
		CheckArchiveTimeout();

		/*
		 * Send off activity statistics to the stats collector.  (The reason
		 * why we re-use bgwriter-related code for this is that the bgwriter
		 * and checkpointer used to be just one process.  It's probably not
		 * worth the trouble to split the stats support into two independent
		 * stats message types.)
		 */
		pgstat_send_bgwriter();

		/* Send WAL statistics to the stats collector. */
		pgstat_send_wal(true);

		/* If any checkpoint flags have been set, redo the loop to handle the checkpoint without sleeping. */
		if (((volatile CheckpointerShmemStruct *) CheckpointerShmem)->ckpt_flags)
			continue;

		/* Sleep until we are signaled or it's time for another checkpoint or xlog file switch. */
		now = (pg_time_t) time(NULL);
		elapsed_secs = now - last_checkpoint_time;
		if (elapsed_secs >= CheckPointTimeout)
			continue;			/* no sleep for us ... */
		cur_timeout = CheckPointTimeout - elapsed_secs;
		if (XLogArchiveTimeout > 0 && !RecoveryInProgress())
		{
			elapsed_secs = now - last_xlog_switch_time;
			if (elapsed_secs >= XLogArchiveTimeout)
				continue;		/* no sleep for us ... */
			cur_timeout = Min(cur_timeout, XLogArchiveTimeout - elapsed_secs);
		}

		(void) WaitLatch(MyLatch, WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH, cur_timeout * 1000L /* convert to ms */ , WAIT_EVENT_CHECKPOINTER_MAIN);
    }
}
```



### checkpoint语句


我们先建表并插入一条数据，手动执行checkpoint语句触发checkpoint进行调试。

```sql
-- 建表，
-- create table t1(a int, b int);
-- 插入数据，并打印WAL_DEBUG日志信息，如何开启可参考参考文档
postgres=# insert into t1 values(7,7);
LOG:  INSERT @ 0/16FBDE8:  - Heap/INSERT: off 7 flags 0x00
LOG:  INSERT @ 0/16FBE10:  - Transaction/COMMIT: 2023-07-10 13:33:42.089411+08
LOG:  xlog flush request 0/16FBE10; write 0/16FBB78; flush 0/16FBB78
INSERT 0 1
-- 手动触发checkpoint
postgres=# checkpoint;
CHECKPOINT
```


主流程分析：
```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
--> pg_plan_queries
--> PortalStart
--> PortalRun
    --> standard_ProcessUtility
        --> RequestCheckpoint   // Called in backend processes to request a checkpoint
--> PortalDrop
```

我们看一下`RequestCheckpoint`函数的实现，顾名思义，这个函数是请求checkpoint的意思，向谁请求呢？向checkpointer进程请求执行一次checkpointer操作。修改与checkpointer进程共享的`CheckpointerShmem->ckpt_flags`，将flag置为`CHECKPOINT_REQUESTED`，checkpointer进程在无限循环中检查这个flag，发现有`CHECKPOINT_REQUESTED`请求后，调用`CreateCheckPoint`执行checkpoint操作。
```c++
// Called in backend processes to request a checkpoint
void RequestCheckpoint(int flags)
{
    // ...

	/*
	 * Atomically set the request flags, and take a snapshot of the counters.
	 * When we see ckpt_started > old_started, we know the flags we set here
	 * have been seen by checkpointer.
	 *
	 * Note that we OR the flags with any existing flags, to avoid overriding
	 * a "stronger" request by another backend.  The flag senses must be
	 * chosen to make this work!
	 */
	SpinLockAcquire(&CheckpointerShmem->ckpt_lck);

	old_failed = CheckpointerShmem->ckpt_failed;
	old_started = CheckpointerShmem->ckpt_started;
    // 修改ckpt_flags，这个会被checkpointer进程的无限循环中被检测到标识被改变了，然后再checkpointer进程中完成checkpoint操作
	CheckpointerShmem->ckpt_flags |= (flags | CHECKPOINT_REQUESTED);

	SpinLockRelease(&CheckpointerShmem->ckpt_lck);
}
```

### checkpoint的执行
具体的CHECKPOINT是如何被执行的？这里分析一下`CreateCheckPoint`函数的实现。CheckPoint实现的要点，记录本次检查点信息，删除历史WAL日志记录，刷脏页到磁盘。主流程如下：
```c++
CheckpointerMain()
--> CreateCheckPoint(flags)
	--> curInsert = XLogBytePosToRecPtr(Insert->CurrBytePos);
	--> checkPoint.redo = curInsert;
	--> CheckPointGuts(checkPoint.redo, flags);
		--> CheckPointBuffers(flags); 
			--> BufferSync(flags);
				--> SyncOneBuffer(buf_id, false, &wb_context)
					--> FlushBuffer(bufHdr, NULL);
						--> smgrwrite
							--> mdwrite
	--> UpdateControlFile();
	--> RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr);
```
其中一个很重要的点是记录本次检查点信息，检查点会把本次检查点开始时的日志插入点`Insert->CurrBytePos`记录到`RedoRecPtr`变量中，这样在数据库故障恢复时，就可以以此为起点进行恢复。为啥是这个位置呢？在此之前的WAL日志，对应的Buffer中的脏页需要全部刷盘，也就是说，在此位置之前的WAL日志是不需要进行回放的，因为页已经落盘了。崩溃恢复的起点，也就是Redo位置，是从这个位置开始，Buffer中没有落盘的开始的位置，因为在执行Checkpoint的过程中，数据库依旧在运行，业务仍然在不断产生WAL日志，以及Buffer中会缓存脏页，这部分并没有进行落盘，所以当数据库发生崩溃恢复时，如果从Redo之前的位置开始回放，则因为页已经持久化了，无须进行回放，所以，回放的起点就是Redo的位置。这篇博文有图片解释，可能会生动一些，可参考[Postgresql Checkpoint 原理](https://zhmin.github.io/posts/postgresql-checkpoint/)

>关于重做点，重做点其实就是当前WAL日志的插入点，或者说当前WAL日志记录的终点位置，从这个位置开始，之前的所有WAL日志对应的脏页刷盘，然后清理WAL日志，在进行checkpoint的过程中，其实业务是不停的，还会继续源源不断的产生WAL日志，后续产生的WAL日志就是从重做点开始的，而这部分WAL日志对于的脏页并没有被刷盘，所以崩溃恢复时，需要从这个重做点开始进行回放。

```c++
void CreateCheckPoint(int flags)
{
	CheckPoint	checkPoint;
	XLogRecPtr	recptr;
	XLogSegNo	_logSegNo;
	XLogCtlInsert *Insert = &XLogCtl->Insert;
	uint32		freespace;
	XLogRecPtr	PriorRedoPtr;
	XLogRecPtr	curInsert;
	XLogRecPtr	last_important_lsn;

	// ...

	/* Begin filling in the checkpoint WAL record */
	MemSet(&checkPoint, 0, sizeof(checkPoint));
	checkPoint.time = (pg_time_t) time(NULL);

	/* Get location of last important record . */
	last_important_lsn = GetLastImportantRecPtr();

	/* We must block concurrent insertions while examining insert state to determine the checkpoint REDO pointer. */
	WALInsertLockAcquireExclusive();
	curInsert = XLogBytePosToRecPtr(Insert->CurrBytePos);

	/* If this isn't a shutdown or forced checkpoint, and if there has been no WAL activity requiring a checkpoint, skip it.  */
	if ((flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY | CHECKPOINT_FORCE)) == 0)
	{
		// 如果“重要”事务日志还处于上次CheckPoint的位置，则本次CheckPoint可以不用执行
		if (last_important_lsn == ControlFile->checkPoint)
		{
			WALInsertLockRelease();
			END_CRIT_SECTION();
			ereport(DEBUG1, (errmsg_internal("checkpoint skipped because system is idle")));
			return;
		}
	}

	/* Compute new REDO record ptr = location of next XLOG record. */
	// 如果当前页面没有空闲空间，则推进到下一个页面
	freespace = INSERT_FREESPACE(curInsert);
	if (freespace == 0)
	{
		if (XLogSegmentOffset(curInsert, wal_segment_size) == 0)
			curInsert += SizeOfXLogLongPHD;
		else
			curInsert += SizeOfXLogShortPHD;
	}
	// 记录本次检查点开始的LSN
	checkPoint.redo = curInsert;

	RedoRecPtr = XLogCtl->Insert.RedoRecPtr = checkPoint.redo;
	WALInsertLockRelease();

	// 更新RedoRecPtr
	SpinLockAcquire(&XLogCtl->info_lck);
	XLogCtl->RedoRecPtr = checkPoint.redo;
	SpinLockRelease(&XLogCtl->info_lck);

	// ...

	// 将脏页刷盘，以及其他需要落盘的数据
	CheckPointGuts(checkPoint.redo, flags);

	// ...

	// 更新pg_control文件
	UpdateControlFile();

	// ...

	// 获得清理位置，删除无用的日志文件
	/*
	 * Delete old log files, those no longer needed for last checkpoint to
	 * prevent the disk holding the xlog from growing full.
	 */
	XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size);
	KeepLogSeg(recptr, &_logSegNo);
	if (InvalidateObsoleteReplicationSlots(_logSegNo))
	{
		XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size);
		KeepLogSeg(recptr, &_logSegNo);
	}
	_logSegNo--;
	RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr);

	// ...
}

/* Flush all data in shared memory to disk, and fsync */
static void CheckPointGuts(XLogRecPtr checkPointRedo, int flags)
{
	CheckPointRelationMap();		// 保障在检查点开始之前所有Relation Map都已经被刷盘
	CheckPointReplicationSlots();	// 把日志复制使用的Slot信息刷入磁盘
	CheckPointSnapBuild();			// 移除无用的快照信息
	CheckPointLogicalRewriteHeap();
	CheckPointReplicationOrigin();

	/* Write out all dirty data in SLRUs and the main buffer pool */
	TRACE_POSTGRESQL_BUFFER_CHECKPOINT_START(flags);
	CheckpointStats.ckpt_write_t = GetCurrentTimestamp();
	CheckPointCLOG();			// 刷新事务提交日志
	CheckPointCommitTs();
	CheckPointSUBTRANS();		// 刷新子事务日志
	CheckPointMultiXact();		// 刷新元组的事务状态日志信息
	CheckPointPredicate();
	CheckPointBuffers(flags);	// 刷入主缓冲区中的脏页

	/* Perform all queued up fsyncs */
	TRACE_POSTGRESQL_BUFFER_CHECKPOINT_SYNC_START();
	CheckpointStats.ckpt_sync_t = GetCurrentTimestamp();
	ProcessSyncRequests();
	CheckpointStats.ckpt_sync_end_t = GetCurrentTimestamp();
	TRACE_POSTGRESQL_BUFFER_CHECKPOINT_DONE();

	/* We deliberately delay 2PC checkpointing as long as possible */
	CheckPointTwoPhase(checkPointRedo);
}
```
我们看一下Checkpoint最重要的一个任务，将脏页刷盘：
```c++
/* Flush all dirty blocks in buffer pool to disk at checkpoint time. */
void CheckPointBuffers(int flags)
{
	BufferSync(flags);
}
```







---

参考文档：
[POSTGRESQL: WHAT IS A CHECKPOINT?](https://www.cybertec-postgresql.com/en/postgresql-what-is-a-checkpoint/)
[探索 PostgreSQL 中的 checkpooint 机制](https://zhuanlan.zhihu.com/p/360506744)
[开发人员的postgresql选项-wal_debug](https://mlog.club/article/2500221)
[CHECKPOINT](https://www.postgresql.org/docs/14/sql-checkpoint.html)