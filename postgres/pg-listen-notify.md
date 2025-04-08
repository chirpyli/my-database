## PostgreSQL中的异步通知机制
PostgreSQL提供了客户端和其他客户端通过数据库服务器进行消息通信的机制，即异步通知，PostgreSQL通过LISTEN和NOTIFY命令提供了异步通知。。

### 使用方法

客户端通过LISTEN命令监听一个通道，当通道有NOTIFY消息时，客户端会收到通知。
```sql
-- 监听mychannel通道
postgres=# listen mychannel;
LISTEN
```
其他客户端通过NOTIFY命令向一个通道发送消息，所有监听该通道的客户端都会收到通知。如果在事务中发生NOTIFY，那么只有事务提交后，才会发送消息，如果事务回滚，则不会发送消息。另外需要注意的是，如果发送通知时，没有监听该通道的客户端，那么消息会丢失，不会像消息队列一样存储在队列中，等待消费者消费，即使在之后有客户端监听该通道，也不会重新收到该消息。
```sql
-- 向mychannel通道发送消息
postgres=# notify mychannel,'hello world';
NOTIFY
```
也可以通过函数`pg_notify`发送消息，效果与`notify`相同。`pg_notify`函数的参数是通道名称和消息内容。消息内容的长度是有限制的，最大长度为8000字节（```#define NOTIFY_PAYLOAD_MAX_LENGTH	(BLCKSZ - NAMEDATALEN - 128)```）。
```sql
postgres=# select pg_notify('mychannel','use pg_notify notify message');
 pg_notify 
-----------
 
(1 row)
```

此时，监听的客户端还没有反应，需要执行任意一条SQL语句，才会收到通知。
```sql
postgres=# select now();
              now              
-------------------------------
 2025-04-03 10:25:20.027341+08
(1 row)
-- 监听的客户端收到通知
Asynchronous notification "mychannel" with payload "hello world" received from server process with PID 6425.
```
如何某个客户端要取消监听，可以使用UNLISTEN命令。
```sql
-- 取消监听mychannel通道
postgres=# unlisten mychannel;
UNLISTEN
```
可通过函数`pg_listening_channels`查看当前客户端监听的通道。
```sql
postgres=# select pg_listening_channels();
 pg_listening_channels 
-----------------------
 mychannel
(1 row)
```

关于事务，需要注意的是，NOTIFY能够保证来自同一个事务的消息按照发送时的顺序通知，也能保证来自不同事务的消息按照事务提交的顺序通知。

在客户端监听通道时，一个客户端应用检测通知事件的必用方法取决于它使用的PostgreSQL应用编程接口。如果使用libpq库，应用会将LISTEN作为一个普通 SQL 命令发出，并且接着必须周期性地调用函数PQnotifies来查看是否接收到通知事件。其他诸如libpgtcl的接口提供了更高层次上的处理通知事件的方法。

### LISTEN NOTIFY机制的使用场景
LISTEN NOTIFY机制可以用于实现数据库的异步通知，但是其通知不是持久化的，没有监听者时消息会丢失，也就是与消息队列相比，其不适合持久化存储消息。建议可考虑PGQMQ（PostgreSQL Queue Manager）等持久化消息队列。


### LISTEN NOTIFY机制的工作原理

要想PostgreSQL实现不同客户端进程之间的异步通信，数据库本身需要有一个各进程都可以访问的数据结构，来保存通知消息或者监听者列表。对此，PostgreSQL选择使用共享内存来保存这些信息，并且使用SLRU页面缓冲池来管理这些共享内存。同时，各客户端怎么知道什么时候有新的通知消息呢？PostgreSQL通过信号来实现，当有新的通知消息时，PostgreSQL进程会向监听该通道的客户端进程发送信号，客户端进程收到信号后，会去共享内存中查看是否有新的通知消息。
```c
PostmasterMain(int argc, char *argv[])
--> reset_shared();  // 初始化共享内存和信号量
    --> CreateSharedMemoryAndSemaphores();
        --> AsyncShmemInit();   // 初始化异步通知相关共享内存和信号量
```
我们看一下以下的`AsyncShmemInit`函数，该函数用于初始化异步通知相关共享内存。
```c
void AsyncShmemInit(void)
{
	bool		found;
	Size		size;

	/*
	 * Create or attach to the AsyncQueueControl structure.
	 *
	 * The used entries in the backend[] array run from 1 to MaxBackends; the
	 * zero'th entry is unused but must be allocated.
	 */
	size = mul_size(MaxBackends + 1, sizeof(QueueBackendStatus));
	size = add_size(size, offsetof(AsyncQueueControl, backend));

	asyncQueueControl = (AsyncQueueControl *)
		ShmemInitStruct("Async Queue Control", size, &found);
    // 为所有Backend初始化
	if (!found)
	{
		/* First time through, so initialize it */
		SET_QUEUE_POS(QUEUE_HEAD, 0, 0);
		SET_QUEUE_POS(QUEUE_TAIL, 0, 0);
		QUEUE_STOP_PAGE = 0;
		QUEUE_FIRST_LISTENER = InvalidBackendId;
		asyncQueueControl->lastQueueFillWarn = 0;
		/* zero'th entry won't be used, but let's initialize it anyway */
		for (int i = 0; i <= MaxBackends; i++)
		{
			QUEUE_BACKEND_PID(i) = InvalidPid;
			QUEUE_BACKEND_DBOID(i) = InvalidOid;
			QUEUE_NEXT_LISTENER(i) = InvalidBackendId;
			SET_QUEUE_POS(QUEUE_BACKEND_POS(i), 0, 0);
		}
	}

	/* Set up SLRU management of the pg_notify data. */
	NotifyCtl->PagePrecedes = asyncQueuePagePrecedes;
	SimpleLruInit(NotifyCtl, "Notify", NUM_NOTIFY_BUFFERS, 0,
				  NotifySLRULock, "pg_notify", LWTRANCHE_NOTIFY_BUFFER,
				  SYNC_HANDLER_NONE);

	if (!found)
	{
		/* During start or reboot, clean out the pg_notify directory. */
		(void) SlruScanDirectory(NotifyCtl, SlruScanDirCbDeleteAll, NULL);
	}
}

static AsyncQueueControl *asyncQueueControl;

// 共享内存数据结构，用于保存异步通知相关数据
typedef struct AsyncQueueControl
{
	QueuePosition head;			/* head points to the next free location */ // 指向下一个空闲位置，表示下一个通知将被写入的位置
	QueuePosition tail;			/* tail must be <= the queue position of every
								 * listening backend */ // 表示所有监听者后端都已读取的位置
	int			stopPage;		/* oldest unrecycled page; must be <=
								 * tail.page */  // SLRU回收机制使用，标识可回收的最旧页面
	BackendId	firstListener;	/* id of first listener, or InvalidBackendId */ // 第一个监听者的ID
	TimestampTz lastQueueFillWarn;	/* time of last queue-full msg */
	QueueBackendStatus backend[FLEXIBLE_ARRAY_MEMBER];
	/* backend[0] is not used; used entries are from [1] to [MaxBackends] */
} AsyncQueueControl;
```

#### LISTEN命令
我们首先分析一下LISTEN命令的实现，在`src/backend/commands/async.c`中，`Async_Listen`函数实现了LISTEN命令。主流程如下：
```c++
exec_simple_query
--> PortalRun
    --> standard_ProcessUtility
        --> Async_Listen(stmt->conditionname);  // 插入LISTEN动作到pendingActions链表
            --> queue_listen(LISTEN_LISTEN, channel);
--> finish_xact_command()  // 事务提交时，处理pendingActions链表中的动作
    --> PreCommit_Notify();
            // 1. 注册监听者
            // 2. 初始化监听位置指针
            // 3. 确保进程退出时正确清理监听状态
        --> Exec_ListenPreCommit(); 
			--> before_shmem_exit(Async_UnlistenOnExit, 0); // 确保进程退出时正确清理监听状态
    --> AtCommit_Notify();  // 发送通知信号给所有监听者
        --> Exec_ListenCommit(actrec->channel); // 如果当前channel中不存在，则将新的channel加入到listenChannels中
			--> ClearPendingActionsAndNotifies(); // 清理pendingActions和pendingNotifies链表
        	
```
`LISTEN`命令的实现主要通过`queue_listen`函数实现，该函数将`LISTEN`动作插入到`pendingActions`链表中，该链表在事务提交时才被处理。
```c++
// 事务级临时存储的链表，用于缓存当前事务中所有待处理的动作，直到事务提交时才批量处理。
static ActionList *pendingActions = NULL;
// 待处理的通知列表
static NotificationList *pendingNotifies = NULL;

// 插入LISTEN动作到pendingActions链表
void Async_Listen(const char *channel)
{
	if (Trace_notify)
		elog(DEBUG1, "Async_Listen(%s,%d)", channel, MyProcPid);

	queue_listen(LISTEN_LISTEN, channel);
}

/*
 * queue_listen
 *		Common code for listen, unlisten, unlisten all commands.
 *
 *		Adds the request to the list of pending actions.
 *		Actual update of the listenChannels list happens during transaction
 *		commit.
 */
static void queue_listen(ListenActionKind action, const char *channel)；
// 通道列表，当事务提交时，会更新该列表
/*
 * listenChannels identifies the channels we are actually listening to
 * (ie, have committed a LISTEN on).  It is a simple list of channel names,
 * allocated in TopMemoryContext.
 */
static List *listenChannels = NIL;	/* list of C strings */
```

#### NOTIFY命令
我们再分析一下`NOTIFY`命令的实现，在`src/backend/commands/async.c`中，`Async_Notify`函数实现了`NOTIFY`命令。主流程如下：
```c++
exec_simple_query
--> PortalRun
    --> standard_ProcessUtility
        --> Async_Notify(stmt->conditionname, stmt->payload); // 插入NOTIFY动作到pendingNotifies链表，事务提交时才被处理
--> finish_xact_command()
    --> CommitTransactionCommand();
        --> CommitTransaction();
            --> PreCommit_Notify(); // 处理待处理的通知列表
            --> AtCommit_Notify();
                --> SignalBackends(); // 发送通知信号给所有监听者
                    --> SendProcSignal(pid, PROCSIG_NOTIFY_INTERRUPT, ids[i])
                --> ClearPendingActionsAndNotifies();
```
我们继续看一下函数`PreCommit_Notify`是如何将消息写入到队列中的。
```c++
// 待处理的通知列表，当事务提交时，会处理该列表中的通知
static NotificationList *pendingNotifies = NULL;
// 处理待处理的通知列表，pendingNotifies以及pendingActions链表
void PreCommit_Notify(void)
{
	ListCell   *p;

	if (!pendingActions && !pendingNotifies)
		return;					/* no relevant statements in this xact */

	/* Preflight for any pending listen/unlisten actions */
	if (pendingActions != NULL)
	{
		foreach(p, pendingActions->actions)
		{
			ListenAction *actrec = (ListenAction *) lfirst(p);

			switch (actrec->action)
			{
				case LISTEN_LISTEN:
					Exec_ListenPreCommit();
					break;
				case LISTEN_UNLISTEN:
					break;
				case LISTEN_UNLISTEN_ALL:
					break;
			}
		}
	}

	/* Queue any pending notifies (must happen after the above) */
	if (pendingNotifies)
	{
		ListCell   *nextNotify;

		(void) GetCurrentTransactionId();

		/*
		 * Serialize writers by acquiring a special lock that we hold till
		 * after commit.  This ensures that queue entries appear in commit
		 * order, and in particular that there are never uncommitted queue
		 * entries ahead of committed ones, so an uncommitted transaction
		 * can't block delivery of deliverable notifications.
		 */
		LockSharedObject(DatabaseRelationId, InvalidOid, 0, AccessExclusiveLock);

		/* Now push the notifications into the queue */
		nextNotify = list_head(pendingNotifies->events);
		while (nextNotify != NULL)
		{
			/*
			 * Add the pending notifications to the queue.  We acquire and
			 * release NotifyQueueLock once per page, which might be overkill
			 * but it does allow readers to get in while we're doing this.
			 *
			 * A full queue is very uncommon and should really not happen,
			 * given that we have so much space available in the SLRU pages.
			 * Nevertheless we need to deal with this possibility. Note that
			 * when we get here we are in the process of committing our
			 * transaction, but we have not yet committed to clog, so at this
			 * point in time we can still roll the transaction back.
			 */
			LWLockAcquire(NotifyQueueLock, LW_EXCLUSIVE);
			asyncQueueFillWarning();
			if (asyncQueueIsFull())
				ereport(ERROR,
						(errcode(ERRCODE_PROGRAM_LIMIT_EXCEEDED),
						 errmsg("too many notifications in the NOTIFY queue")));
			nextNotify = asyncQueueAddEntries(nextNotify);  // 将通知添加到队列中，写入page中
			LWLockRelease(NotifyQueueLock);
		}
	}
}
```

#### 异步通知
当`NOTIFY`消息后，监听者如何获取到通知呢？客户端进程收到了`PROCSIG_NOTIFY_INTERRUPT`信号后，会调用`ProcessNotifyInterrupt`函数，该函数会调用`asyncQueueReadAllNotifications`函数，从队列中读取通知，并通过`NotifyMyFrontEnd`函数将通知发送给客户端。
调用栈如下：
```c++
NotifyMyFrontEnd(const char * channel, const char * payload, int32 srcPid) (src\backend\commands\async.c:2311)
asyncQueueProcessPageEntries(volatile QueuePosition * current, QueuePosition stop, char * page_buffer, Snapshot snapshot) (src\backend\commands\async.c:2142)
asyncQueueReadAllNotifications() (src\backend\commands\async.c:2044)
ProcessIncomingNotify(_Bool flush) (src\backend\commands\async.c:2276)
ProcessNotifyInterrupt(_Bool flush) (src\backend\commands\async.c:1904)
ProcessClientReadInterrupt(_Bool blocked) (src\backend\tcop\postgres.c:510)
secure_read(Port * port, void * ptr, size_t len) (src\backend\libpq\be-secure.c:215)
pq_recvbuf() (src\backend\libpq\pqcomm.c:955)
pq_getbyte() (src\backend\libpq\pqcomm.c:1001)
SocketBackend(StringInfo inBuf) (src\backend\tcop\postgres.c:356)
ReadCommand(StringInfo inBuf) (src\backend\tcop\postgres.c:479)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (src\backend\tcop\postgres.c:4538)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4539)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4261)
ServerLoop() (src\backend\postmaster\postmaster.c:1748)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1420)
main(int argc, char ** argv) (src\backend\main\main.c:211)
```
函数源码如下：
```c++
void PostgresMain(int argc, char *argv[], const char *dbname, const char *username)
{
	volatile bool send_ready_for_query = true;
    // 注册信号处理函数
    pqsignal(SIGUSR1, procsignal_sigusr1_handler); 
    
    for (;;)
	{
		if (send_ready_for_query)
		{
			if (IsAbortedTransactionBlockState())
			{
                // 检查当前事务块状态是否处于已终止状态
            }
            else if (IsTransactionOrTransactionBlock())
            {
                // 检查当前是否处于活动事务或事务块中
            }
            else
            {
				if (notifyInterruptPending)
					ProcessNotifyInterrupt(false); // 处理通知
            }
            
        }
    }
}

/* 信号处理函数 - handle SIGUSR1 signal. */
void procsignal_sigusr1_handler(SIGNAL_ARGS)
{
	int			save_errno = errno;

	if (CheckProcSignal(PROCSIG_NOTIFY_INTERRUPT))
		HandleNotifyInterrupt();      // 处理PROCSIG_NOTIFY_INTERRUPT信号
    	
    SetLatch(MyLatch);
    errno = save_errno;
}

/* HandleNotifyInterrupt */
void HandleNotifyInterrupt(void)
{
	/* signal that work needs to be done */
	notifyInterruptPending = true;

	/* make sure the event is processed in due course */
	SetLatch(MyLatch);
}

void ProcessNotifyInterrupt(bool flush)
{
	if (IsTransactionOrTransactionBlock())
		return;					/* not really idle */

	/* Loop in case another signal arrives while sending messages */
	while (notifyInterruptPending)
		ProcessIncomingNotify(flush);
}

// 扫描队列，将通知发送给前端
static void ProcessIncomingNotify(bool flush)
{
	MemoryContext oldcontext;

	/* We *must* reset the flag */
	notifyInterruptPending = false;

	/* Do nothing else if we aren't actively listening */
	if (listenChannels == NIL)
		return;

	/*
	 * We must run asyncQueueReadAllNotifications inside a transaction, else
	 * bad things happen if it gets an error.  However, we need to preserve
	 * the caller's memory context (typically MessageContext).
	 */
	oldcontext = CurrentMemoryContext;

	StartTransactionCommand();

	asyncQueueReadAllNotifications();

	CommitTransactionCommand();

	/* Caller's context had better not have been transaction-local */
	Assert(MemoryContextIsValid(oldcontext));
	MemoryContextSwitchTo(oldcontext);

	/*
	 * If this isn't an end-of-command case, we must flush the notify messages
	 * to ensure frontend gets them promptly.
	 */
	if (flush)
		pq_flush();
}

/* 读取所有未处理的通知，并把通知发送给前端 */
static void asyncQueueReadAllNotifications(void)
{
	volatile QueuePosition pos;
	QueuePosition head;
	Snapshot	snapshot;

	/* page_buffer must be adequately aligned, so use a union */
	union
	{
		char		buf[QUEUE_PAGESIZE];
		AsyncQueueEntry align;
	}			page_buffer;

	/* 获取当前队列状态 */
	LWLockAcquire(NotifyQueueLock, LW_SHARED);
	/* Assert checks that we have a valid state entry */
	Assert(MyProcPid == QUEUE_BACKEND_PID(MyBackendId));
	pos = QUEUE_BACKEND_POS(MyBackendId);
	head = QUEUE_HEAD;
	LWLockRelease(NotifyQueueLock);

	if (QUEUE_POS_EQUAL(pos, head))
	{
		/* 如果已处理完所有通知则立即返回 */
		return;
	}

	/*----------
	 * Get snapshot we'll use to decide which xacts are still in progress.
	 * This is trickier than it might seem, because of race conditions.
	 * Consider the following example:
	 *
	 * Backend 1:					 Backend 2:
	 *
	 * transaction starts
	 * UPDATE foo SET ...;
	 * NOTIFY foo;
	 * commit starts
	 * queue the notify message
	 *								 transaction starts
	 *								 LISTEN foo;  -- first LISTEN in session
	 *								 SELECT * FROM foo WHERE ...;
	 * commit to clog
	 *								 commit starts
	 *								 add backend 2 to array of listeners
	 *								 advance to queue head (this code)
	 *								 commit to clog
	 *
	 * Transaction 2's SELECT has not seen the UPDATE's effects, since that
	 * wasn't committed yet.  Ideally we'd ensure that client 2 would
	 * eventually get transaction 1's notify message, but there's no way
	 * to do that; until we're in the listener array, there's no guarantee
	 * that the notify message doesn't get removed from the queue.
	 *
	 * Therefore the coding technique transaction 2 is using is unsafe:
	 * applications must commit a LISTEN before inspecting database state,
	 * if they want to ensure they will see notifications about subsequent
	 * changes to that state.
	 *
	 * What we do guarantee is that we'll see all notifications from
	 * transactions committing after the snapshot we take here.
	 * Exec_ListenPreCommit has already added us to the listener array,
	 * so no not-yet-committed messages can be removed from the queue
	 * before we see them.
	 *----------
	 */
	snapshot = RegisterSnapshot(GetLatestSnapshot());

	/*
	 * It is possible that we fail while trying to send a message to our
	 * frontend (for example, because of encoding conversion failure).  If
	 * that happens it is critical that we not try to send the same message
	 * over and over again.  Therefore, we place a PG_TRY block here that will
	 * forcibly advance our queue position before we lose control to an error.
	 * (We could alternatively retake NotifyQueueLock and move the position
	 * before handling each individual message, but that seems like too much
	 * lock traffic.)
	 */
	PG_TRY();
	{
		bool		reachedStop;

		do
		{
			int			curpage = QUEUE_POS_PAGE(pos);	// 当前页
			int			curoffset = QUEUE_POS_OFFSET(pos); // 当前页偏移
			int			slotno;
			int			copysize;

			/*
			 * We copy the data from SLRU into a local buffer, so as to avoid
			 * holding the NotifySLRULock while we are examining the entries
			 * and possibly transmitting them to our frontend.  Copy only the
			 * part of the page we will actually inspect.
			 */
			slotno = SimpleLruReadPage_ReadOnly(NotifyCtl, curpage,
												InvalidTransactionId);
			if (curpage == QUEUE_POS_PAGE(head))
			{
				/* we only want to read as far as head */
				copysize = QUEUE_POS_OFFSET(head) - curoffset;
				if (copysize < 0)
					copysize = 0;	/* just for safety */
			}
			else
			{
				/* fetch all the rest of the page */
				copysize = QUEUE_PAGESIZE - curoffset;
			}
			memcpy(page_buffer.buf + curoffset,
				   NotifyCtl->shared->page_buffer[slotno] + curoffset,
				   copysize);
			/* Release lock that we got from SimpleLruReadPage_ReadOnly() */
			LWLockRelease(NotifySLRULock);

			/*
			 * Process messages up to the stop position, end of page, or an
			 * uncommitted message.
			 *
			 * Our stop position is what we found to be the head's position
			 * when we entered this function. It might have changed already.
			 * But if it has, we will receive (or have already received and
			 * queued) another signal and come here again.
			 *
			 * We are not holding NotifyQueueLock here! The queue can only
			 * extend beyond the head pointer (see above) and we leave our
			 * backend's pointer where it is so nobody will truncate or
			 * rewrite pages under us. Especially we don't want to hold a lock
			 * while sending the notifications to the frontend.
			 */
			reachedStop = asyncQueueProcessPageEntries(&pos, head,
													   page_buffer.buf,
													   snapshot);
		} while (!reachedStop);
	}
	PG_FINALLY();
	{
		/* Update shared state */
		LWLockAcquire(NotifyQueueLock, LW_SHARED);
		QUEUE_BACKEND_POS(MyBackendId) = pos;
		LWLockRelease(NotifyQueueLock);
	}
	PG_END_TRY();

	/* Done with snapshot */
	UnregisterSnapshot(snapshot);
}

/*
 * Fetch notifications from the shared queue, beginning at position current,
 * and deliver relevant ones to my frontend.
 *
 * The current page must have been fetched into page_buffer from shared
 * memory.  (We could access the page right in shared memory, but that
 * would imply holding the NotifySLRULock throughout this routine.)
 *
 * We stop if we reach the "stop" position, or reach a notification from an
 * uncommitted transaction, or reach the end of the page.
 *
 * The function returns true once we have reached the stop position or an
 * uncommitted notification, and false if we have finished with the page.
 * In other words: once it returns true there is no need to look further.
 * The QueuePosition *current is advanced past all processed messages.
 */
static bool
asyncQueueProcessPageEntries(volatile QueuePosition *current,
							 QueuePosition stop,
							 char *page_buffer,
							 Snapshot snapshot)
{
	bool		reachedStop = false;
	bool		reachedEndOfPage;
	AsyncQueueEntry *qe;

	do
	{
		QueuePosition thisentry = *current;

		if (QUEUE_POS_EQUAL(thisentry, stop))
			break;

		qe = (AsyncQueueEntry *) (page_buffer + QUEUE_POS_OFFSET(thisentry));

		/*
		 * Advance *current over this message, possibly to the next page. As
		 * noted in the comments for asyncQueueReadAllNotifications, we must
		 * do this before possibly failing while processing the message.
		 */
		reachedEndOfPage = asyncQueueAdvance(current, qe->length);

		/* Ignore messages destined for other databases */
		if (qe->dboid == MyDatabaseId)
		{
			if (XidInMVCCSnapshot(qe->xid, snapshot))
			{
				/*
				 * The source transaction is still in progress, so we can't
				 * process this message yet.  Break out of the loop, but first
				 * back up *current so we will reprocess the message next
				 * time.  (Note: it is unlikely but not impossible for
				 * TransactionIdDidCommit to fail, so we can't really avoid
				 * this advance-then-back-up behavior when dealing with an
				 * uncommitted message.)
				 *
				 * Note that we must test XidInMVCCSnapshot before we test
				 * TransactionIdDidCommit, else we might return a message from
				 * a transaction that is not yet visible to snapshots; compare
				 * the comments at the head of heapam_visibility.c.
				 *
				 * Also, while our own xact won't be listed in the snapshot,
				 * we need not check for TransactionIdIsCurrentTransactionId
				 * because our transaction cannot (yet) have queued any
				 * messages.
				 */
				*current = thisentry;
				reachedStop = true;
				break;
			}
			else if (TransactionIdDidCommit(qe->xid))
			{
				/* qe->data is the null-terminated channel name */
				char	   *channel = qe->data;

				if (IsListeningOn(channel))
				{
					/* payload follows channel name */
					char	   *payload = qe->data + strlen(channel) + 1;

					NotifyMyFrontEnd(channel, payload, qe->srcPid);
				}
			}
			else
			{
				/*
				 * The source transaction aborted or crashed, so we just
				 * ignore its notifications.
				 */
			}
		}

		/* Loop back if we're not at end of page */
	} while (!reachedEndOfPage);

	if (QUEUE_POS_EQUAL(*current, stop))
		reachedStop = true;

	return reachedStop;
}
```

```c
/*-------------------------------------------------------------------------
9.0版本的异步通知模型：

1. 同一台机器上的多个后端。多个后端监听多个通道。（代码的其他部分中，通道也称为“条件”。）

2. 有一个基于磁盘的中央队列（目录pg_notify/），由slru.c模块将活跃使用的页面映射到共享内存中。
	  所有通知消息都放置在队列中，稍后由监听的后端读取。

	  没有中央的渠道监听信息；每个后端都有自己的感兴趣渠道列表。

	  尽管只有一个队列，但通知被视为数据库本地的；这是通过在每个通知消息中包含发送者的数据库OID来实现的。
	  监听的后端会忽略与自己的数据库OID不匹配的消息。这是重要的，因为它确保发送者和接收者具有相同的数据库编码，
	  不会错误解释通道名称或有效负载字符串中的非ASCII文本。

	  由于不期望通知在数据库崩溃后仍然存在，我们可以在任何重启时清空pg_notify数据，而不需要WAL支持或fsync。

3. 每个至少监听一个通道的后端通过将PID写入AsyncQueueControl中的数组来注册。然后它会扫描中央队列中的所有
	  进入的通知，首先比较通知的数据库OID与自己的数据库OID，然后比较通知的通道与自己监听的通道列表。
	  如果匹配，它将通知事件传递给前端。不匹配的事件将被忽略。

4. NOTIFY语句（Async_Notify）将通知存储在后端本地的列表中，直到事务结束时才会处理。

	  同一事务中的重复通知仅发送一次。这是为了节省工作，例如在一个包含200万行的表上触发的通知，每行更改都会触发
	  一个通知。如果应用程序需要接收每个发送的通知，可以在额外的有效负载参数中添加一些唯一字符串。

	  当事务准备提交时，PreCommit_Notify()将待处理的通知添加到队列的头部。队列的头部指针始终指向下一个空闲位置，
	  位置只是一个页码和该页中的偏移量。这在标记事务在clog中提交之前完成。如果在写入通知时遇到问题，
	  我们仍然可以调用elog(ERROR, ...)，事务将回滚。

	  一旦将所有通知放入队列，我们返回CommitTransaction()，它将执行实际的事务提交。

	  提交后，我们会被再次调用（AtCommit_Notify()）。在这里，我们对有效的监听状态（listenChannels）进行任何实际更新。
	  然后我们向可能对我们的消息感兴趣的任何后端发送信号（包括我们自己的后端，如果它正在监听）。这是通过
	  SignalBackends()完成的，它扫描监听后端的列表并向每个监听后端发送PROCSIG_NOTIFY_INTERRUPT信号（我们不知道
	  哪个后端在监听哪个通道，所以必须向所有后端发送信号）。我们可以排除已经更新的后端，也可以排除其他数据库中的
	  后端（除非它们远远落后，需要被踢动以推进它们的指针）。

	  最后，当事务完全结束后即将进入空闲状态时，我们扫描队列以查找需要发送到前端的消息（这些消息可能是来自其他
	  后端的通知，也可能是来自我们自己的自通知）。这一步不是CommitTransaction序列的一部分，有两个重要原因。
	  首先，我们在向前端发送数据时可能会遇到错误，而在提交后的清理过程中发生错误是真的很糟糕。
	  其次，在一个前端命令内多次提交的场景中，我们不希望在命令完成前向前端发送通知；但每次提交后，
	  应立即向其他后端发送通知。

5. 当接收到PROCSIG_NOTIFY_INTERRUPT信号时，信号处理程序会设置进程的latch，如果此后端处于空闲状态
	  （即，它正在等待前端命令且不在事务块内，参见ProcessClientReadInterrupt()），这将触发事件立即被处理。
	  否则，处理程序只能设置一个标志，这将导致在我们下一次进入空闲状态之前进行处理。

	  入站通知处理包括读取自上次扫描以来到达的所有通知。我们读取每个通知，直到遇到一个未提交事务的通知或
	  队列头部指针的位置。

6. 为了避免SLRU环绕并限制磁盘空间消耗，需要推进尾指针，以便截断旧页面。
	  这相对昂贵（特别是，它需要一个排他锁），所以我们不想频繁这样做。我们让发送后端在将队列头部推进到新页面时
	  完成这项工作，但每QUEUE_CLEANUP_DELAY页才做一次。

监听同一通道的应用程序将接收到自己的NOTIFY消息。如果这些消息没有用处，可以通过比较NOTIFY消息中的be_pid与
应用程序的后端PID来忽略它们。（从FE/BE协议2.0起，后端的PID在启动时提供给前端。）上述设计保证了通过忽略自通知，
不会错过来自其他后端的通知。

用于通知管理的共享内存数量（NUM_NOTIFY_BUFFERS）可以在不影响其他任何内容的情况下调整，仅影响性能。
一次可以排队的最大通知数据量由slru.c的环绕限制决定；请参见QUEUE_MAX_PAGE。
-------------------------------------------------------------------------
/
```


参考文档：
[LISTEN](http://postgres.cn/docs/current/sql-listen.html)
[NOTIFY](http://postgres.cn/docs/current/sql-notify.html)
[34.9. 异步通知](http://www.postgres.cn/docs/current/libpq-notify.html)