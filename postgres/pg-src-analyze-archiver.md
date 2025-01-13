

### PostgreSQL源码分析——日志归档

PG中有日志归档功能，主要目的就是备份恢复，PITR，为啥要做日志归档呢？因为在做检查点时会清理WAL日志，清理了之后，就没法实现恢复到任意时刻数据库状态了，而有了日志归档，我们可以保存从数据库初始状态到当前时刻的所有日志，相当于给数据库做了一个备份。当发生故障或者误操作时，可以恢复到指定时刻数据库的状态。

#### 打开日志归档
在配置文件中配置`archive_mode=on`打开日志归档，启动时会创建归档进程archiver，通过`archive_command`中配置的命令进行归档。
```shell
archive_mode = on               # enables archiving; off, on, or always (change requires restart)
archive_command = 'cp %p /home/postgres/pgsql/archive/%f'               # command to use to archive a logfile segment
                                # placeholders: %p = path of file to archive
                                #               %f = file name only
                                # e.g. 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f'
archive_timeout = 1800          # force a logfile segment switch after this
                                # number of seconds; 0 disables
```

#### 归档进程源码
我们看一下归档进程的源码，在`src/backend/postmaster/pgarch.c`中：
```c++
PgArchiverMain(void)
--> pgarch_MainLoop();	// 进入归档主循环
	--> pgarch_ArchiverCopyLoop(); 
		--> pgarch_readyXlog
```
日志归档的逻辑，主要是什么时候进行归档？**核心要点是发生日志段切换时会触发**，那我们看一下那些情况会触发日志切换
- 当WAL日志中的一个日志段（日志文件）已满，需要切换到下一个日志段时，就可以通知archiver进程将这个日志归档。产生日志切换的进程会在通知Postmaster之前先在`pg_wal/archive_status`下生成一个`.ready`文件，这个文件和待归档日志同名。
- 如果长时间没有归档，触发`archive_timeout`超时，则强制进行日志切换，强制归档
- 调用`pg_switch_wal()`函数手动触发

我们看一下归档进程主循环的实现逻辑，就是等待归档通知信号，拷贝日志：
```c++
static void pgarch_MainLoop(void)
{
	pg_time_t	last_copy_time = 0;
	bool		time_to_stop;
	// 进入主循环, 等待收到日志归档通知
	do {
		ResetLatch(MyLatch);

		/* When we get SIGUSR2, we do one more archive cycle, then exit */
		time_to_stop = ready_to_stop;

		/* Check for barrier events and config update */
		HandlePgArchInterrupts();

		// ...

		/* Do what we're here for */
		pgarch_ArchiverCopyLoop();		// 进行日志归档，拷贝WAL日志
		last_copy_time = time(NULL);

		/* Sleep until a signal is received, or until a poll is forced by
		 * PGARCH_AUTOWAKE_INTERVAL having passed since last_copy_time, or until postmaster dies. */
		if (!time_to_stop)		/* Don't wait during last iteration */
		{
			pg_time_t	curtime = (pg_time_t) time(NULL);
			int			timeout;

			timeout = PGARCH_AUTOWAKE_INTERVAL - (curtime - last_copy_time);
			if (timeout > 0) {
				int			rc;
				rc = WaitLatch(MyLatch, WL_LATCH_SET | WL_TIMEOUT | WL_POSTMASTER_DEATH,timeout * 1000L, WAIT_EVENT_ARCHIVER_MAIN);
				if (rc & WL_POSTMASTER_DEATH)
					time_to_stop = true;
			}
		}
	} while (!time_to_stop);
}
```

##### 触发归档的时机1
其中最重要的就是什么时候发信号，通知可以归档，是在切换日志段的时候，什么时候会切换日志段呢？用户可以通过调用`pg_switch_wal`函数强制切换日志段，正常情况下是不断插入日志的过程中，如果超出了日志段的大小，会触发切换日志段。我们看一下这块的处理逻辑。

> 具体的`XLogWrite`调用过程可参考文章[PostgreSQL源码分析——WAL日志（二）](https://www.modb.pro/db/1707333250107142144)
```c++
static void XLogWrite(XLogwrtRqst WriteRqst, bool flexible)
{
	// ...
			/* If we just wrote the whole last page of a logfile segment,
			 * fsync the segment immediately.  This avoids having to go back
			 * and re-open prior segments when an fsync request comes along
			 * later. Doing it here ensures that one and only one backend will
			 * perform this fsync.
			 *
			 * This is also the right place to notify the Archiver that the
			 * segment is ready to copy to archival storage, and to update the
			 * timer for archive_timeout, and to signal for a checkpoint if
			 * too many logfile segments have been used since the last checkpoint. */
			if (finishing_seg)	// 一个段已满
			{
				// 将该段刷入磁盘，保证归档日志的数据完整性
				issue_xlog_fsync(openLogFile, openLogSegNo);

				// 通知walsender进程发送日志给standby
				WalSndWakeupRequest();

				LogwrtResult.Flush = LogwrtResult.Write;	/* end of page */

				if (XLogArchivingActive())
					XLogArchiveNotifySeg(openLogSegNo);	// 发送日志归档通知信息

				// 更新日志切换时间，计算archive_timeout用
				XLogCtl->lastSegSwitchTime = (pg_time_t) time(NULL);
				XLogCtl->lastSegSwitchLSN = LogwrtResult.Flush;

				/* Request a checkpoint if we've consumed too much xlog since
				 * the last one.  For speed, we first check using the local
				 * copy of RedoRecPtr, which might be out of date; if it looks
				 * like a checkpoint is needed, forcibly update RedoRecPtr and recheck. */
				if (IsUnderPostmaster && XLogCheckpointNeeded(openLogSegNo))
				{
					(void) GetRedoRecPtr();
					if (XLogCheckpointNeeded(openLogSegNo))
						RequestCheckpoint(CHECKPOINT_CAUSE_XLOG);
				}
			}
}


```
我们看一下这个`XLogArchiveNotify`函数实现，日志归档通知，创建一个`.ready`文件，表示可以进行归档。当归档完成时，将对应的`.ready`文件重命名为`.done`文件
```c++
void XLogArchiveNotifySeg(XLogSegNo segno)
{
	char		xlog[MAXFNAMELEN];
	// 根据timeline, LSN生成日志文件名
	XLogFileName(xlog, ThisTimeLineID, segno, wal_segment_size);
	XLogArchiveNotify(xlog);
}

/* XLogArchiveNotify
 *
 * Create an archive notification file
 *
 * The name of the notification file is the message that will be picked up
 * by the archiver, e.g. we write 0000000100000001000000C6.ready
 * and the archiver then knows to archive XLOGDIR/0000000100000001000000C6,
 * then when complete, rename it to 0000000100000001000000C6.done */
void XLogArchiveNotify(const char *xlog)
{
	char		archiveStatusPath[MAXPGPATH];
	FILE	   *fd;

	/* insert an otherwise empty file called <XLOG>.ready */
	StatusFilePath(archiveStatusPath, xlog, ".ready");
	fd = AllocateFile(archiveStatusPath, "w");
	if (fd == NULL) {
		ereport(LOG,(errcode_for_file_access(), errmsg("could not create archive status file \"%s\": %m",archiveStatusPath)));
		return;
	}
	if (FreeFile(fd)) {
		ereport(LOG,(errcode_for_file_access(),errmsg("could not write archive status file \"%s\": %m",archiveStatusPath)));
		return;
	}

	if (IsUnderPostmaster)
		PgArchWakeup();		// 唤醒归档进程，进入拷贝日志逻辑
}
```

当唤醒归档进程后，归档进程检测`pg_wal/archive_status`目录下的`.ready`文件，发现有新的`.ready`文件出现，就对`pg_wal`下的同名日志段依照`archive_command`参数中的归档命令进行归档。

##### 触发时机2
另一种情况是`archive_timeout`超时触发。如果超过这个设定的时间而没有进行日志归档，则做一次日志切换，强制归档。之所以增加了这个参数，是为了避免数据库如果长时间没有写请求造成长时间不会发生日志段切换，而造成日志段长时间不进行归档的问题。
```c++
CheckpointerMain(void)
--> CheckArchiveTimeout();
	--> RequestXLogSwitch(true)
```
源码如下：
```c++
static void CheckArchiveTimeout(void)
{
	pg_time_t	now;
	pg_time_t	last_time;
	XLogRecPtr	last_switch_lsn;

	if (XLogArchiveTimeout <= 0 || RecoveryInProgress())
		return;

	now = (pg_time_t) time(NULL);

	/* First we do a quick check using possibly-stale local state. */
	if ((int) (now - last_xlog_switch_time) < XLogArchiveTimeout)
		return;

	/* Update local state ... note that last_xlog_switch_time is the last time a switch was performed *or requested*. */
	last_time = GetLastSegSwitchData(&last_switch_lsn);

	last_xlog_switch_time = Max(last_xlog_switch_time, last_time);

	// 如果已经超时，切换日志段
	/* Now we can do the real checks */
	if ((int) (now - last_xlog_switch_time) >= XLogArchiveTimeout)
	{
		/* Switch segment only when "important" WAL has been logged since the
		 * last segment switch (last_switch_lsn points to end of segment switch occurred in). */
		if (GetLastImportantRecPtr() > last_switch_lsn)
		{
			XLogRecPtr	switchpoint;

			/* mark switch as unimportant, avoids triggering checkpoints */
			switchpoint = RequestXLogSwitch(true);	// 切换日志段

			/* If the returned pointer points exactly to a segment boundary, assume nothing happened. */
			if (XLogSegmentOffset(switchpoint, wal_segment_size) != 0)
				elog(DEBUG1, "write-ahead log switch forced (archive_timeout=%d)",
					 XLogArchiveTimeout);
		}

		/* Update state in any case, so we don't retry constantly when the system is idle. */
		last_xlog_switch_time = now;
	}
}
```

##### 拷贝归档日志
实现拷贝归档日志源码，其核心逻辑是，检测`pg_wal/archive_status`目录下的.ready文件，最先选择段号最小的文件，也就是最老的日志段文件进行归档，解析%f,%p，然后具体执行调用`system`执行归档命令，完成日志归档。
```c++
static void pgarch_ArchiverCopyLoop(void)
{
	char		xlog[MAX_XFN_CHARS + 1];

	// 检测`pg_wal/archive_status`目录下的.ready文件，选择段号最小的文件，也就是最老的日志段文件
	while (pgarch_readyXlog(xlog)) {
		int			failures = 0;
		int			failures_orphan = 0;

		for (;;) {
			// ...

			// 检查归档命令有无设置
			if (!XLogArchiveCommandSet()) {
				ereport(WARNING, (errmsg("archive_mode enabled, yet archive_command is not set")));
				return;
			}

			/* Since archive status files are not removed in a durable manner,
			 * a system crash could leave behind .ready files for WAL segments
			 * that have already been recycled or removed.  In this case,
			 * simply remove the orphan status file and move on.  unlink() is
			 * used here as even on subsequent crashes the same orphan files
			 * would get removed, so there is no need to worry about durability. */
			snprintf(pathname, MAXPGPATH, XLOGDIR "/%s", xlog);
			if (stat(pathname, &stat_buf) != 0 && errno == ENOENT){
				char		xlogready[MAXPGPATH];

				StatusFilePath(xlogready, xlog, ".ready");
				if (unlink(xlogready) == 0){
					ereport(WARNING,(errmsg("removed orphan archive status file \"%s\"",xlogready)));

					/* leave loop and move to the next status file */
					break;
				}

				if (++failures_orphan >= NUM_ORPHAN_CLEANUP_RETRIES){
					ereport(WARNING,(errmsg("removal of orphan archive status file \"%s\" failed too many times, will try again later",xlogready)));

					/* give up cleanup of orphan status files */
					return;
				}

				/* wait a bit before retrying */
				pg_usleep(1000000L);
				continue;
			}
			// 进行日志归档
			if (pgarch_archiveXlog(xlog)) {
				/* successful */
				pgarch_archiveDone(xlog);	// 将对应日志文件.ready重命名为.done

				/* Tell the collector about the WAL file that we successfully archived */
				pgstat_send_archiver(xlog, false);

				break;			/* out of inner retry loop */
			} else {
				// 如果归档失败，进行3重试，依旧失败则报错返回
				/* Tell the collector about the WAL file that we failed to archive */
				pgstat_send_archiver(xlog, true);

				if (++failures >= NUM_ARCHIVE_RETRIES) {
					ereport(WARNING,(errmsg("archiving write-ahead log file \"%s\" failed too many times, will try again later",xlog)));
					return;		/* give up archiving for now */
				}
				pg_usleep(1000000L);	/* wait a bit before retrying */
			}
		}
	}
}

// 调用system执行归档命令， 中间需要解析%f, %p
static bool pgarch_archiveXlog(char *xlog)
{
	char		xlogarchcmd[MAXPGPATH];
	char		pathname[MAXPGPATH];
	char		activitymsg[MAXFNAMELEN + 16];
	char	   *dp;
	char	   *endp;
	const char *sp;
	int			rc;

	snprintf(pathname, MAXPGPATH, XLOGDIR "/%s", xlog);

	/* construct the command to be executed */
	dp = xlogarchcmd;
	endp = xlogarchcmd + MAXPGPATH - 1;
	*endp = '\0';

	for (sp = XLogArchiveCommand; *sp; sp++)
	{
		if (*sp == '%')
		{
			switch (sp[1])
			{
				case 'p':
					/* %p: relative path of source file */
					sp++;
					strlcpy(dp, pathname, endp - dp);
					make_native_path(dp);
					dp += strlen(dp);
					break;
				case 'f':
					/* %f: filename of source file */
					sp++;
					strlcpy(dp, xlog, endp - dp);
					dp += strlen(dp);
					break;
				case '%':
					/* convert %% to a single % */
					sp++;
					if (dp < endp)
						*dp++ = *sp;
					break;
				default:
					/* otherwise treat the % as not special */
					if (dp < endp)
						*dp++ = *sp;
					break;
			}
		}
		else
		{
			if (dp < endp)
				*dp++ = *sp;
		}
	}
	*dp = '\0';

	ereport(DEBUG3, (errmsg_internal("executing archive command \"%s\"", xlogarchcmd)));

	/* Report archive activity in PS display */
	snprintf(activitymsg, sizeof(activitymsg), "archiving %s", xlog);
	set_ps_display(activitymsg);

	rc = system(xlogarchcmd);		// 执行归档命令
	// ...

	elog(DEBUG1, "archived write-ahead log file \"%s\"", xlog);

	snprintf(activitymsg, sizeof(activitymsg), "last was %s", xlog);
	set_ps_display(activitymsg);

	return true;
}
```

##### 什么时候删除.done文件呢？
`.done`文件用于标识日志段文件已归档完成，可以删除了，所以其删除的逻辑就是删除日志文件的时候，同步删除对应的`.done`文件。 
```c++
RemoveOldXlogFiles
--> RemoveXlogFile
	--> XLogArchiveCleanup(segname);
```

```c++
static void RemoveXlogFile(const char *segname, XLogSegNo recycleSegNo, XLogSegNo *endlogSegNo)
{
	// 回收段文件或者删除
	// ...
	/* Before deleting the file, see if it can be recycled as a future log
	 * segment. Only recycle normal files, because we don't want to recycle
	 * symbolic links pointing to a separate archive directory */
	if (wal_recycle &&*endlogSegNo <= recycleSegNo &&
		lstat(path, &statbuf) == 0 && S_ISREG(statbuf.st_mode) &&
		InstallXLogFileSegment(endlogSegNo, path, true, recycleSegNo, true))
	{
		ereport(DEBUG2,
				(errmsg_internal("recycled write-ahead log file \"%s\"",
								 segname)));
		CheckpointStats.ckpt_segs_recycled++;
		/* Needn't recheck that slot on future iterations */
		(*endlogSegNo)++;
	}else{
		/* No need for any more future segments... */
		int			rc;

		ereport(DEBUG2,(errmsg_internal("removing write-ahead log file \"%s\"",segname)));
		rc = durable_unlink(path, LOG);

		if (rc != 0){
			/* Message already logged by durable_unlink() */
			return;
		}
		CheckpointStats.ckpt_segs_removed++;
	}

	// 删除.done文件
	XLogArchiveCleanup(segname);
}
// 删除.done文件
void XLogArchiveCleanup(const char *xlog)
{
	char		archiveStatusPath[MAXPGPATH];

	/* Remove the .done file */
	StatusFilePath(archiveStatusPath, xlog, ".done");
	unlink(archiveStatusPath);
	/* should we complain about failure? */

	/* Remove the .ready file if present --- normally it shouldn't be */
	StatusFilePath(archiveStatusPath, xlog, ".ready");
	unlink(archiveStatusPath);
	/* should we complain about failure? */
}
```

#### 归档需要注意的地方
1. 日志级别必须大于`minimal`，否则不能通过日志进行恢复。同时`archive_mode`不能是关闭状态
2. `archive_command`归档命令必须有效，如果无效的话，会因为没有进行归档而造成日志清理时无法正常进行清理，导致日志堆积。
```c++
static void RemoveOldXlogFiles(XLogSegNo segno, XLogRecPtr lastredoptr, XLogRecPtr endptr)
{
	DIR		   *xldir;
	struct dirent *xlde;
	char		lastoff[MAXFNAMELEN];
	XLogSegNo	endlogSegNo;
	XLogSegNo	recycleSegNo;

	/* Initialize info about where to try to recycle to */
	XLByteToSeg(endptr, endlogSegNo, wal_segment_size);
	recycleSegNo = XLOGfileslop(lastredoptr);

	/* Construct a filename of the last segment to be kept.  */
	XLogFileName(lastoff, 0, segno, wal_segment_size);

	elog(DEBUG2, "attempting to remove WAL segments older than log file %s", lastoff);

	xldir = AllocateDir(XLOGDIR);

	while ((xlde = ReadDir(xldir, XLOGDIR)) != NULL)
	{
		/* Ignore files that are not XLOG segments */
		if (!IsXLogFileName(xlde->d_name) &&
			!IsPartialXLogFileName(xlde->d_name))
			continue;

		if (strcmp(xlde->d_name + 8, lastoff + 8) <= 0) {
			// 如果日志没有归档完成，则不删除日志
			if (XLogArchiveCheckDone(xlde->d_name)) {
				/* Update the last removed location in shared memory first */
				UpdateLastRemovedPtr(xlde->d_name);

				RemoveXlogFile(xlde->d_name, recycleSegNo, &endlogSegNo);
			}
		}
	}

	FreeDir(xldir);
}
```
检查归档是否完成，归档没有完成的话，检查点进程触发的清理WAL日志时是不能清理这部分日志的
```c++
/*
 * XLogArchiveCheckDone
 *
 * This is called when we are ready to delete or recycle an old XLOG segment
 * file or backup history file.  If it is okay to delete it then return true.
 * If it is not time to delete it, make sure a .ready file exists, and return
 * false.
 *
 * If <XLOG>.done exists, then return true; else if <XLOG>.ready exists,
 * then return false; else create <XLOG>.ready and return false.
 *
 * The reason we do things this way is so that if the original attempt to
 * create <XLOG>.ready fails, we'll retry during subsequent checkpoints.
 */
// 检查指定日志段文件是否完成归档，完成归档，返回true，表示可删除， 未完成归档，返回false
bool XLogArchiveCheckDone(const char *xlog)
{
	char		archiveStatusPath[MAXPGPATH];
	struct stat stat_buf;

	/* The file is always deletable if archive_mode is "off". */
	if (!XLogArchivingActive())
		return true;

	/* During archive recovery, the file is deletable if archive_mode is not "always". */
	if (!XLogArchivingAlways() && GetRecoveryState() == RECOVERY_STATE_ARCHIVE)
		return true;

	/* At this point of the logic, note that we are either a primary with
	 * archive_mode set to "on" or "always", or a standby with archive_mode set to "always". */

	/* First check for .done --- this means archiver is done with it */
	// 检查是否有同名.done文件，有则说明归档完成
	StatusFilePath(archiveStatusPath, xlog, ".done");
	if (stat(archiveStatusPath, &stat_buf) == 0)
		return true;

	/* check for .ready --- this means archiver is still busy with it */
	StatusFilePath(archiveStatusPath, xlog, ".ready");
	if (stat(archiveStatusPath, &stat_buf) == 0)
		return false;

	/* Race condition --- maybe archiver just finished, so recheck */
	StatusFilePath(archiveStatusPath, xlog, ".done");
	if (stat(archiveStatusPath, &stat_buf) == 0)
		return true;

	/* Retry creation of the .ready file */
	XLogArchiveNotify(xlog);
	return false;
}
```
如果归档完成，会在`pg_wal/archive_status`目录下生成一个`.done`文件。如果归档没有完成，尝试检测是否存在`.ready`文件，如果存在，说明归档尚未完成，返回false，如果`.ready`文件不存在，则尝试生成`.ready`文件，表示该文件已就绪可以进行归档了。
```sh
postgres@slpc:~/pgsql/pgdata/pg_wal/archive_status$ ls
000000010000000000000001.done  000000010000000000000002.ready	# .done表示已完成归档， .ready表示可以进行归档
```