
SLRU(Simple LRU)是PostgreSQL中用于管理事务状态日志文件的高效缓冲系统，为CLOG(事务提交日志)、SUBTRANS(子事务)、NOTIFY(异步通知)等关键子系统提供支持。
```sql
postgres=# select * from pg_stat_slru;
      name       | blks_zeroed | blks_hit | blks_read | blks_written | blks_exists | flushes | truncates |         stats_reset         
-----------------+-------------+----------+-----------+--------------+-------------+---------+-----------+-----------------------------
 CommitTs        |           0 |        0 |         0 |            0 |           0 |       0 |         0 | 2025-04-07 15:27:10.5189+08
 MultiXactMember |           0 |        0 |         0 |            0 |           0 |       0 |         0 | 2025-04-07 15:27:10.5189+08
 MultiXactOffset |           0 |        0 |         0 |            0 |           0 |       0 |         0 | 2025-04-07 15:27:10.5189+08
 Notify          |           0 |        0 |         0 |            0 |           0 |       0 |         0 | 2025-04-07 15:27:10.5189+08
 Serial          |           0 |        0 |         0 |            0 |           0 |       0 |         0 | 2025-04-07 15:27:10.5189+08
 Subtrans        |           0 |        0 |         0 |            0 |           0 |       0 |         0 | 2025-04-07 15:27:10.5189+08
 Xact            |           0 |        0 |         0 |            0 |           0 |       0 |         0 | 2025-04-07 15:27:10.5189+08
 other           |           0 |        0 |         0 |            0 |           0 |       0 |         0 | 2025-04-07 15:27:10.5189+08
(8 rows)
```
这里我们以异步通知为例，分析SLRU缓冲池的实现。异步通知队列`pg_notify`使用SLRU缓冲池来存储通知消息。SLRU缓冲池中有8个槽位，每个槽位对应一个页面。当SLRU缓冲池没有空闲槽位时，就需要通过LRU算法将缓冲池中的一个页面淘汰出去，淘汰到哪里呢？写入到文件中，每个文件为一个段，每个段由`SLRU_PAGES_PER_SEGMEN`(32)个页面构成，每个页面大小为8K。我们可通过页面号来定位文件，通过页面号在文件中的偏移量来定位文件中的具体位置，而具体写入页面时，还会有页面偏移量，即写入页面的那个位置。
```c++
/* The number of SLRU page buffers we use for the notification queue. */
#define NUM_NOTIFY_BUFFERS	8

//	int			segno = pageno / SLRU_PAGES_PER_SEGMENT;  // 段编号
#define SlruFileName(ctl, path, seg) \
	snprintf(path, MAXPGPATH, "%s/%04X", (ctl)->Dir, seg)
```

### 数据结构

SLRU缓冲池的数据结构定义在`slru.h`文件中，主要包括以下部分：
```c++
/* SLRU共享内存状态信息 */
typedef struct SlruSharedData
{
	LWLock	   *ControlLock; // 控制锁，用于保护共享内存中的数据结构
	int			num_slots; // 缓冲池中槽位数量

    /* 页面缓冲池中每个槽位信息 */
	char	  **page_buffer;      // 缓冲池中的页面数组
	SlruPageStatus *page_status;  // 页面状态
	bool	   *page_dirty;       // 页面是否脏
	int		   *page_number;      // 每个缓存页面对应的磁盘页面编号
	int		   *page_lru_count;   // 页面LRU计数
	LWLockPadded *buffer_locks;

	/*
	 * Optional array of WAL flush LSNs associated with entries in the SLRU
	 * pages.  If not zero/NULL, we must flush WAL before writing pages 
	 */
	XLogRecPtr *group_lsn;  // 用于Clog中，multixact, pg_subtrans, pg_notify为NULL
	int			lsn_groups_per_page; // 每个缓存页面对应多少个Group LSN

	/*
	 * We mark a page "most recently used" by setting
	 *		page_lru_count[slotno] = ++cur_lru_count;
	 */
	int			cur_lru_count;   // LRU算法使用
	int			latest_page_number;  // 最新页面编号，最新页面不需要swap out

	int			slru_stats_idx;		/* SLRU's index for statistics purposes */
} SlruSharedData;

typedef SlruSharedData *SlruShared;


/* SLRU控制结构 */
typedef struct SlruCtlData
{
	SlruShared	shared; // 指向共享内存
	/* Which sync handler function to use when handing sync requests over to the checkpointer.  */
	SyncRequestHandler sync_handler;
	bool		(*PagePrecedes) (int, int); // 比较两个页面的先后顺序
	char		Dir[64];  // 物理文件在磁盘的存储目录
} SlruCtlData;

typedef SlruCtlData *SlruCtl;
```

### 初始化
初始化流程如下：
```c++
main(int argc, char *argv[])
--> PostmasterMain(argc, argv);
    --> reset_shared();
        --> CreateSharedMemoryAndSemaphores(); // 创建共享内存和信号量
            --> size = add_size(size, CLOGShmemSize());
            --> size = add_size(size, AsyncShmemSize());
            --> CLOGShmemInit();
            --> AsyncShmemInit(); 
                --> SimpleLruInit(NotifyCtl, "Notify", NUM_NOTIFY_BUFFERS, 0, NotifySLRULock, "pg_notify", LWTRANCHE_NOTIFY_BUFFER, SYNC_HANDLER_NONE);
            --> PGSharedMemoryCreate(size, &shim);
```
初始化阶段主要是初始化共享内存，首先要计算出所需的共享内存大小，可通过函数`SimpleLruShmemSize`进行计算：
```c++
/* 
 * 计算SLRU缓冲池所需的共享内存总大小
 * nslots: 缓冲池中页面数量；
 */
Size SimpleLruShmemSize(int nslots, int nlsns)
{
	Size		sz;

	/* we assume nslots isn't so large as to risk overflow */
	sz = MAXALIGN(sizeof(SlruSharedData));
	sz += MAXALIGN(nslots * sizeof(char *));	/* page_buffer[] */
	sz += MAXALIGN(nslots * sizeof(SlruPageStatus));	/* page_status[] */
	sz += MAXALIGN(nslots * sizeof(bool));	/* page_dirty[] */
	sz += MAXALIGN(nslots * sizeof(int));	/* page_number[] */
	sz += MAXALIGN(nslots * sizeof(int));	/* page_lru_count[] */
	sz += MAXALIGN(nslots * sizeof(LWLockPadded));	/* buffer_locks[] */

	if (nlsns > 0) // 通知队列pg_notify可省略，nlsns为0
		sz += MAXALIGN(nslots * nlsns * sizeof(XLogRecPtr));	/* group_lsn[] */

	return BUFFERALIGN(sz) + BLCKSZ * nslots;
}
```
计算完成所需共享内存大小后就可以初始化LRU缓冲池了，其函数为`SimpleLruInit`，创建或附加到共享内存中的LRU缓存。以异步通知为例，会初始设置`NotifyCtl`值。异步通知缓冲池通过该值来访问该缓冲池。。
```c++
/* The SLRU buffer area through which we access the notification queue */
static SlruCtlData NotifyCtlData;
#define NotifyCtl					(&NotifyCtlData)
```
具体的函数`SimpleLruInit`实现如下：
```c++
/*
 * Initialize, or attach to, a simple LRU cache in shared memory.
 *
 * ctl: address of local (unshared) control structure.
 * name: SLRU名，唯一标识. 
 * nslots: 缓冲池槽位数量
 * nlsns: number of LSN groups per page (set to zero if not relevant).
 * ctllock: LWLock to use to control access to the shared control structure.
 * subdir: 存储目录
 * tranche_id: LWLock tranche ID to use for the SLRU's per-buffer LWLocks.
 * sync_handler: which set of functions to use to handle sync requests
 */
void SimpleLruInit(SlruCtl ctl, const char *name, int nslots, int nlsns, LWLock *ctllock, const char *subdir, int tranche_id, SyncRequestHandler sync_handler)
{
	SlruShared	shared;  // 指向共享内存
	bool		found;   // 是否已经初始化

	shared = (SlruShared) ShmemInitStruct(name, SimpleLruShmemSize(nslots, nlsns), &found);
	if (!IsUnderPostmaster)
	{
		char	   *ptr;
		Size		offset;
		int			slotno;

		memset(shared, 0, sizeof(SlruSharedData));

		shared->ControlLock = ctllock; // 控制锁
		shared->num_slots = nslots;  // 缓冲池中页面数量
		shared->lsn_groups_per_page = nlsns;
		shared->cur_lru_count = 0; // LRU计数
		shared->slru_stats_idx = pgstat_slru_index(name);		/* shared->latest_page_number will be set later */

		ptr = (char *) shared;
		offset = MAXALIGN(sizeof(SlruSharedData));
		shared->page_buffer = (char **) (ptr + offset);
		offset += MAXALIGN(nslots * sizeof(char *));
		shared->page_status = (SlruPageStatus *) (ptr + offset);
		offset += MAXALIGN(nslots * sizeof(SlruPageStatus));
		shared->page_dirty = (bool *) (ptr + offset);
		offset += MAXALIGN(nslots * sizeof(bool));
		shared->page_number = (int *) (ptr + offset);
		offset += MAXALIGN(nslots * sizeof(int));
		shared->page_lru_count = (int *) (ptr + offset);
		offset += MAXALIGN(nslots * sizeof(int));

		/* Initialize LWLocks */
		shared->buffer_locks = (LWLockPadded *) (ptr + offset);
		offset += MAXALIGN(nslots * sizeof(LWLockPadded));

		if (nlsns > 0)
		{
			shared->group_lsn = (XLogRecPtr *) (ptr + offset);
			offset += MAXALIGN(nslots * nlsns * sizeof(XLogRecPtr));
		}

		ptr += BUFFERALIGN(offset);
		for (slotno = 0; slotno < nslots; slotno++)
		{
			LWLockInitialize(&shared->buffer_locks[slotno].lock, tranche_id);

			shared->page_buffer[slotno] = ptr;
			shared->page_status[slotno] = SLRU_PAGE_EMPTY;
			shared->page_dirty[slotno] = false;
			shared->page_lru_count[slotno] = 0;
			ptr += BLCKSZ;
		}
	}

	/* Initialize the unshared control struct, including directory path.  */
	ctl->shared = shared;
	ctl->sync_handler = sync_handler;
	strlcpy(ctl->Dir, subdir, sizeof(ctl->Dir));
}
```

### 写操作
我们以异步通知写为例，分析SLRU缓冲池的写操作。写操作，最重要的是写到哪里（缓冲池中的那个页面），如果没有空闲的槽位可用，则需要通过LRU算法获取一个可用的槽位，写多少字节，写什么数据。因为会有多个进程访问共享内存，对其写操作，必须要获取锁，否则会出现并发问题。写操作完成后，需要将缓冲池中的页面标记为脏页，这样在刷新到磁盘时，才能将数据写入磁盘。
```c++
static ListCell *asyncQueueAddEntries(ListCell *nextNotify)
{
	AsyncQueueEntry qe;
	QueuePosition queue_head;
	int			pageno;
	int			offset;
	int			slotno;

	LWLockAcquire(NotifySLRULock, LW_EXCLUSIVE); // 获取锁

	/* We work with a local copy of QUEUE_HEAD, which we write back to shared memory upon exiting. */
	queue_head = QUEUE_HEAD;

	pageno = QUEUE_POS_PAGE(queue_head);  
	if (QUEUE_POS_IS_ZERO(queue_head)) // 如果是第一次写，则初始化第一个页面，置为0
		slotno = SimpleLruZeroPage(NotifyCtl, pageno);
	else  // 如果不是第一次写，则获取当前页面的槽位号
		slotno = SimpleLruReadPage(NotifyCtl, pageno, true, InvalidTransactionId);

	NotifyCtl->shared->page_dirty[slotno] = true; 	/* 将页面置为脏页 */

	while (nextNotify != NULL)
	{
		Notification *n = (Notification *) lfirst(nextNotify);

		/* Construct a valid queue entry in local variable qe */
		asyncQueueNotificationToEntry(n, &qe);
		offset = QUEUE_POS_OFFSET(queue_head);  // 获取当前页面的偏移量

		/* Check whether the entry really fits on the current page */
		if (offset + qe.length <= QUEUE_PAGESIZE)
		{
			nextNotify = lnext(pendingNotifies->events, nextNotify); 			/* OK, so advance nextNotify past this item */
		}
		else
		{
			/* Write a dummy entry to fill up the page. Actually readers will
			 * only check dboid and since it won't match any reader's database
			 * OID, they will ignore this entry and move on. */
			qe.length = QUEUE_PAGESIZE - offset;
			qe.dboid = InvalidOid;
			qe.data[0] = '\0';	/* empty channel */
			qe.data[1] = '\0';	/* empty payload */
		}

		/* Now copy qe into the shared buffer page */
		memcpy(NotifyCtl->shared->page_buffer[slotno] + offset, /* 向SLRU缓冲池对应槽位的页面写入数据 */
			   &qe,
			   qe.length);

		/* Advance queue_head appropriately, and detect if page is full */
		if (asyncQueueAdvance(&(queue_head), qe.length))
		{
			/*
			 * Page is full, so we're done here, but first fill the next page
			 * with zeroes.  The reason to do this is to ensure that slru.c's
			 * idea of the head page is always the same as ours, which avoids
			 * boundary problems in SimpleLruTruncate.  The test in
			 * asyncQueueIsFull() ensured that there is room to create this
			 * page without overrunning the queue.
			 */
			slotno = SimpleLruZeroPage(NotifyCtl, QUEUE_POS_PAGE(queue_head));

			/*
			 * If the new page address is a multiple of QUEUE_CLEANUP_DELAY,
			 * set flag to remember that we should try to advance the tail
			 * pointer (we don't want to actually do that right here).
			 */
			if (QUEUE_POS_PAGE(queue_head) % QUEUE_CLEANUP_DELAY == 0)
				tryAdvanceTail = true;

			break;  			/* And exit the loop */
		}
	}

	QUEUE_HEAD = queue_head;  // 更新队列头

	LWLockRelease(NotifySLRULock);  // 释放锁

	return nextNotify;
}
```
我们看一下如何新增一个页面，以及如何将数据写入页面。
```c++
// 分配一个可用的页面槽位，并将页面置为0，返回页面槽位号
int SimpleLruZeroPage(SlruCtl ctl, int pageno)
{
	SlruShared	shared = ctl->shared;
	int	slotno = SlruSelectLRUPage(ctl, pageno); // 获取一个可用的页面槽位

	/* Mark the slot as containing this page */
	shared->page_number[slotno] = pageno; 
	shared->page_status[slotno] = SLRU_PAGE_VALID;
	shared->page_dirty[slotno] = true;
	SlruRecentlyUsed(shared, slotno);  // 标记为最近使用

	MemSet(shared->page_buffer[slotno], 0, BLCKSZ); // 将页面置为0

	SimpleLruZeroLSNs(ctl, slotno);  	/* Set the LSNs for this new page to zero */

	/* Assume this page is now the latest active page */
	shared->latest_page_number = pageno;  // 更新最新的页面号，避免对最新页面不必要的淘汰

	/* update the stats counter of zeroed pages */
	pgstat_count_slru_page_zeroed(shared->slru_stats_idx);

	return slotno;
}
```
其中函数`SlruSelectLRUPage`为SLRU缓冲池的核心页面替换算法实现函数，负责在需要空闲的槽位时选择最合适的页面槽位。如果没有空闲的槽位，则需要选择一个槽位进行页面替换。
```c++
/*
 * 选择最合适的页面槽位，返回页面槽位号，选择规则如下：
 * 1. 如果当前页已存在在槽位中，则直接返回对应的槽位号
 * 2. 如果当前页面槽位为空，则直接返回对应的槽位号
 * 3. 如果当前页面槽位不为空，则执行LRU算法进行替换，淘汰那个cur_lru_count-page_lru_count最大的页面
 */
static int SlruSelectLRUPage(SlruCtl ctl, int pageno)
{
	SlruShared	shared = ctl->shared;

	/* Outer loop handles restart after I/O */
	for (;;)
	{
		int			slotno;
		int			cur_count;
		int			bestvalidslot = 0;	/* keep compiler quiet */
		int			best_valid_delta = -1;
		int			best_valid_page_number = 0; /* keep compiler quiet */
		int			bestinvalidslot = 0;	/* keep compiler quiet */
		int			best_invalid_delta = -1;
		int			best_invalid_page_number = 0;	/* keep compiler quiet */

		/* 如果当前页已存在在槽位中，则直接返回对应的槽位号 */
		for (slotno = 0; slotno < shared->num_slots; slotno++)
		{
			if (shared->page_number[slotno] == pageno &&
				shared->page_status[slotno] != SLRU_PAGE_EMPTY)
				return slotno;
		}

		/*
		 * If we find any EMPTY slot, just select that one. Else choose a
		 * victim page to replace.  We normally take the least recently used
		 * valid page, but we will never take the slot containing
		 * latest_page_number, even if it appears least recently used.  We
		 * will select a slot that is already I/O busy only if there is no
		 * other choice: a read-busy slot will not be least recently used once
		 * the read finishes, and waiting for an I/O on a write-busy slot is
		 * inferior to just picking some other slot.  Testing shows the slot
		 * we pick instead will often be clean, allowing us to begin a read at once.
		 */
		cur_count = (shared->cur_lru_count)++;
		for (slotno = 0; slotno < shared->num_slots; slotno++) // 遍历所有页面槽位
		{
			int			this_delta;
			int			this_page_number;
            // 如果当前页面槽位为空，则直接返回对应的槽位号
			if (shared->page_status[slotno] == SLRU_PAGE_EMPTY)
				return slotno;
			this_delta = cur_count - shared->page_lru_count[slotno]; 
			if (this_delta < 0) // 发生了wrapped around, int overflow
			{
				/*
				 * Clean up in case shared updates have caused cur_count
				 * increments to get "lost".  We back off the page counts,
				 * rather than trying to increase cur_count, to avoid any
				 * question of infinite loops or failure in the presence of
				 * wrapped-around counts.
				 */
				shared->page_lru_count[slotno] = cur_count;
				this_delta = 0; 
			}
			this_page_number = shared->page_number[slotno];
			if (this_page_number == shared->latest_page_number) // 如果当前页面是最新页面，则跳过，避免将最新页面不必要的淘汰
				continue;
			if (shared->page_status[slotno] == SLRU_PAGE_VALID)
			{
				if (this_delta > best_valid_delta ||
					(this_delta == best_valid_delta &&
					 ctl->PagePrecedes(this_page_number, best_valid_page_number)))
				{
					bestvalidslot = slotno;
					best_valid_delta = this_delta;
					best_valid_page_number = this_page_number;
				}
			}
			else
			{
				if (this_delta > best_invalid_delta ||
					(this_delta == best_invalid_delta &&
					 ctl->PagePrecedes(this_page_number, best_invalid_page_number)))
				{
					bestinvalidslot = slotno;
					best_invalid_delta = this_delta;
					best_invalid_page_number = this_page_number;
				}
			}
		}

		/*
		 * If all pages (except possibly the latest one) are I/O busy, we'll
		 * have to wait for an I/O to complete and then retry.  In that
		 * unhappy case, we choose to wait for the I/O on the least recently
		 * used slot, on the assumption that it was likely initiated first of
		 * all the I/Os in progress and may therefore finish first.
		 */
		if (best_valid_delta < 0)
		{
			SimpleLruWaitIO(ctl, bestinvalidslot);
			continue;
		}

		/* If the selected page is clean, we're set. */
		if (!shared->page_dirty[bestvalidslot])
			return bestvalidslot;

		SlruInternalWritePage(ctl, bestvalidslot, NULL); // 将页面写入磁盘文件中
	}
}
```
其中函数`SlruInternalWritePage`负责将页面写入磁盘文件中，实际执行写入的是函数`SlruPhysicalWritePage`。
```c++
static void SlruInternalWritePage(SlruCtl ctl, int slotno, SlruWriteAll fdata)
{
	SlruShared	shared = ctl->shared;
	int			pageno = shared->page_number[slotno];

    // ... 
	bool ok = SlruPhysicalWritePage(ctl, pageno, slotno, fdata); // 将页面写入磁盘文件中
    // ...
}

/* 将缓冲槽中的指定页面写入磁盘文件中 */
static bool SlruPhysicalWritePage(SlruCtl ctl, int pageno, int slotno, SlruWriteAll fdata)
{
	SlruShared	shared = ctl->shared;
	int			segno = pageno / SLRU_PAGES_PER_SEGMENT; /* 计算当前页面所在的段号 */
	int			rpageno = pageno % SLRU_PAGES_PER_SEGMENT; // 计算当前页面在段中的偏移量
	off_t		offset = rpageno * BLCKSZ;  // 计算当前页面在文件中的偏移量
	char		path[MAXPGPATH];
	int			fd = -1;

	/* update the stats counter of written pages */
	pgstat_count_slru_page_written(shared->slru_stats_idx);

	/*
	 * Honor the write-WAL-before-data rule, if appropriate, so that we do not
	 * write out data before associated WAL records.  This is the same action
	 * performed during FlushBuffer() in the main buffer manager.
	 */
	if (shared->group_lsn != NULL)
	{
		/* We must determine the largest async-commit LSN for the page. This
		 * is a bit tedious, but since this entire function is a slow path
		 * anyway, it seems better to do this here than to maintain a per-page
		 * LSN variable (which'd need an extra comparison in the
		 * transaction-commit path). */
		XLogRecPtr	max_lsn;
		int			lsnindex,
					lsnoff;

		lsnindex = slotno * shared->lsn_groups_per_page;
		max_lsn = shared->group_lsn[lsnindex++];
		for (lsnoff = 1; lsnoff < shared->lsn_groups_per_page; lsnoff++)
		{
			XLogRecPtr	this_lsn = shared->group_lsn[lsnindex++];

			if (max_lsn < this_lsn)
				max_lsn = this_lsn;
		}

		if (!XLogRecPtrIsInvalid(max_lsn))
		{
			/* As noted above, elog(ERROR) is not acceptable here, so if
			 * XLogFlush were to fail, we must PANIC.  This isn't much of a
			 * restriction because XLogFlush is just about all critical
			 * section anyway, but let's make sure. */
			START_CRIT_SECTION();
			XLogFlush(max_lsn);
			END_CRIT_SECTION();
		}
	}

	/* During a WriteAll, we may already have the desired file open. */
	if (fdata)
	{
		for (int i = 0; i < fdata->num_files; i++)
		{
			if (fdata->segno[i] == segno)
			{
				fd = fdata->fd[i];
				break;
			}
		}
	}

	if (fd < 0)
	{
		SlruFileName(ctl, path, segno); // 构造文件名
		fd = OpenTransientFile(path, O_RDWR | O_CREAT | PG_BINARY); // 创建临时文件
		if (fd < 0)
		{
			slru_errcause = SLRU_OPEN_FAILED;
			slru_errno = errno;
			return false;
		}

		if (fdata)
		{
			if (fdata->num_files < MAX_WRITEALL_BUFFERS)
			{
				fdata->fd[fdata->num_files] = fd;
				fdata->segno[fdata->num_files] = segno;
				fdata->num_files++;
			}
			else
			{
				/* In the unlikely event that we exceed MAX_FLUSH_BUFFERS, fall back to treating it as a standalone write. */
				fdata = NULL;
			}
		}
	}

	errno = 0;
	pgstat_report_wait_start(WAIT_EVENT_SLRU_WRITE);
	if (pg_pwrite(fd, shared->page_buffer[slotno], BLCKSZ, offset) != BLCKSZ) // 将页面写入文件
	{
		pgstat_report_wait_end();
		/* if write didn't set errno, assume problem is no disk space */
		if (errno == 0)
			errno = ENOSPC;
		slru_errcause = SLRU_WRITE_FAILED;
		slru_errno = errno;
		if (!fdata)
			CloseTransientFile(fd);
		return false;
	}
	pgstat_report_wait_end();

	/* Queue up a sync request for the checkpointer. */
	if (ctl->sync_handler != SYNC_HANDLER_NONE)
	{
		FileTag		tag;

		INIT_SLRUFILETAG(tag, ctl->sync_handler, segno);
		if (!RegisterSyncRequest(&tag, SYNC_REQUEST, false))
		{
			/* No space to enqueue sync request.  Do it synchronously. */
			pgstat_report_wait_start(WAIT_EVENT_SLRU_SYNC);
			if (pg_fsync(fd) != 0)
			{
				pgstat_report_wait_end();
				slru_errcause = SLRU_FSYNC_FAILED;
				slru_errno = errno;
				CloseTransientFile(fd);
				return false;
			}
			pgstat_report_wait_end();
		}
	}

	if (!fdata)
	{
		if (CloseTransientFile(fd) != 0)  // 关闭文件
		{
			slru_errcause = SLRU_CLOSE_FAILED;
			slru_errno = errno;
			return false;
		}
	}

	return true;
}
```

### 读操作
读操作的逻辑是，如果读的页已存在在SLRU缓冲池槽位中，则直接返回该槽位号，否则选择一个空闲的槽位（如果此时缓冲池已满，则通过LRU算法选择一个最近最少使用的页进行替换），从文件中读取页数据到该槽位，然后返回该槽位号。
```c++
/*
 * 通过SLRU读取指定页面，如果页面不在缓冲池中，则从文件中读取
 * write_ok: 读该页面时，如果该页面为WRITE_IN_PROGRESS状态，是否进行读取
 */
int SimpleLruReadPage(SlruCtl ctl, int pageno, bool write_ok, TransactionId xid)
{
	SlruShared	shared = ctl->shared;

	/* Outer loop handles restart if we must wait for someone else's I/O */
	for (;;)
	{
		int slotno = SlruSelectLRUPage(ctl, pageno); // 选择一个缓冲池槽位，如果缓冲池已满，则选择一个最近最少使用的页进行替换

		/* Did we find the page in memory? */
		if (shared->page_number[slotno] == pageno &&
			shared->page_status[slotno] != SLRU_PAGE_EMPTY)
		{
			/*
			 * If page is still being read in, we must wait for I/O.  Likewise
			 * if the page is being written and the caller said that's not OK.
			 */
			if (shared->page_status[slotno] == SLRU_PAGE_READ_IN_PROGRESS ||
				(shared->page_status[slotno] == SLRU_PAGE_WRITE_IN_PROGRESS &&
				 !write_ok))
			{
				SimpleLruWaitIO(ctl, slotno);
				/* Now we must recheck state from the top */
				continue;
			}
			/* Otherwise, it's ready to use */
			SlruRecentlyUsed(shared, slotno);

			/* update the stats counter of pages found in the SLRU */
			pgstat_count_slru_page_hit(shared->slru_stats_idx);

			return slotno;
		}

		/* Mark the slot read-busy */
		shared->page_number[slotno] = pageno;
		shared->page_status[slotno] = SLRU_PAGE_READ_IN_PROGRESS; // 标记该槽位为读取状态
		shared->page_dirty[slotno] = false;

		/* Acquire per-buffer lock (cannot deadlock, see notes at top) */
		LWLockAcquire(&shared->buffer_locks[slotno].lock, LW_EXCLUSIVE);

		/* Release control lock while doing I/O */
		LWLockRelease(shared->ControlLock);

		bool ok = SlruPhysicalReadPage(ctl, pageno, slotno);  // 从文件中读取页面数据到缓冲池槽位

		/* Set the LSNs for this newly read-in page to zero */
		SimpleLruZeroLSNs(ctl, slotno);

		/* Re-acquire control lock and update page state */
		LWLockAcquire(shared->ControlLock, LW_EXCLUSIVE);

		Assert(shared->page_number[slotno] == pageno &&
			   shared->page_status[slotno] == SLRU_PAGE_READ_IN_PROGRESS &&
			   !shared->page_dirty[slotno]);

		shared->page_status[slotno] = ok ? SLRU_PAGE_VALID : SLRU_PAGE_EMPTY;

		LWLockRelease(&shared->buffer_locks[slotno].lock);

		/* Now it's okay to ereport if we failed */
		if (!ok)
			SlruReportIOError(ctl, pageno, xid);

		SlruRecentlyUsed(shared, slotno);

		/* update the stats counter of pages not found in SLRU */
		pgstat_count_slru_page_read(shared->slru_stats_idx);

		return slotno;
	}
}

/* 从文件中读取页面数据到缓冲池槽位 */
static bool SlruPhysicalReadPage(SlruCtl ctl, int pageno, int slotno)
{
	SlruShared	shared = ctl->shared;
	int			segno = pageno / SLRU_PAGES_PER_SEGMENT; // 根据页号计算出所在文件段号和页号
	int			rpageno = pageno % SLRU_PAGES_PER_SEGMENT;
	off_t		offset = rpageno * BLCKSZ;
	char		path[MAXPGPATH];
	int			fd;

	SlruFileName(ctl, path, segno);  // 根据文件段号生成文件路径

	fd = OpenTransientFile(path, O_RDONLY | PG_BINARY); // 打开文件
	if (fd < 0)
	{
		if (errno != ENOENT || !InRecovery)
		{
			slru_errcause = SLRU_OPEN_FAILED;
			slru_errno = errno;
			return false;
		}

		ereport(LOG, (errmsg("file \"%s\" doesn't exist, reading as zeroes", path)));
		MemSet(shared->page_buffer[slotno], 0, BLCKSZ);
		return true;
	}

	errno = 0;
	pgstat_report_wait_start(WAIT_EVENT_SLRU_READ);
	if (pg_pread(fd, shared->page_buffer[slotno], BLCKSZ, offset) != BLCKSZ) // 从文件中读取页面数据到缓冲池槽位
	{
		pgstat_report_wait_end();
		slru_errcause = SLRU_READ_FAILED;
		slru_errno = errno;
		CloseTransientFile(fd);
		return false;
	}
	pgstat_report_wait_end();

	if (CloseTransientFile(fd) != 0)  // 关闭文件
	{
		slru_errcause = SLRU_CLOSE_FAILED;
		slru_errno = errno;
		return false;
	}

	return true;
}
```