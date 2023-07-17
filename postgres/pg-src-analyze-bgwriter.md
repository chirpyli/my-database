### bgwriter进程分析

#### 为什么会有bgwriter
bgwriter进程主要负责将共享缓冲区（Buffer）中的脏页刷盘，这个进程主要是从数据库性能的考虑而加的，如果没有这个进程，数据库一样可以工作。所以，这里重点理解的就是bgwriter进程对性能的影响。

我们前面讲过，一条插入语句的执行过程，先在Buffer中找到空闲页，在页中插入元组，暂不刷盘，而且先构造WAL日志，将WAL日志刷盘，再由后台进程（bgwriter）刷盘。之所以这么设计就是出于性能的考虑，每次写后，频繁的进行刷盘会降低性能。比如，连续进行100次插入，每次插入的都是同一个页，就会造成对这个页频繁的进行刷盘，而通过bgwriter以及wal，则转换为写WAL，再对该脏页刷1次盘即可，设计WAL，bgwriter其中目的之一都是为了降低刷盘的频率。

其二，在有WAL后，那我一直不对脏页进行刷盘行不行？答案是肯定不行，即使bgwriter不进行刷盘，缓冲区也会进行页淘汰，缓冲区大小是有限的，当缓冲区满了时，又需要从磁盘中读数据页到缓冲区中，就必须将缓冲区中的部分页进行淘汰，目前的算法是时钟扫描算法，如果选择淘汰的页是脏页，则需要将脏页进行刷盘，这会导致查询或者更新需要更长的时间，自然性能降低了。周期性的进行脏页刷盘，避免了在查询过程中因为缓冲区淘汰页导致的刷盘，避免了因此导致的性能降低。

其三，bgwriter进行周期性的刷盘，对性能的平稳有益，能够一定程度的避免性能的抖动，使得IO操作尽可能的被平滑的处理了。不单单是bgwriter，其他进程也有这方面设计的思考，比如autovacuum进程。


#### 参数说明

bgwriter涉及到以下参数设置：
```shell
#bgwriter_delay = 200ms                 # 10-10000ms between rounds
#bgwriter_lru_maxpages = 100            # max buffers written/round, 0 disables
#bgwriter_lru_multiplier = 2.0          # 0-10.0 multiplier on buffers scanned/round
```
表示系统每间隔`bgwriter_delay`指定的时间启动进程bgwriter，扫描缓冲区，写出至多`bgwriter_lru_multiplier * N`个脏页，并且不超过`bgwriter_lru_maxpages`值的限制。其中N是最近一段时间在两次bgwriter运行期间系统新申请的缓冲页数。



#### 源码分析
bgwriter进程，核心流程很清晰，就是间隔一段时间进行Buffer刷盘，主流程如下：
```c++
/*
 * Main entry point for bgwriter process
 *
 * This is invoked from AuxiliaryProcessMain, which has already created the
 * basic execution environment, but not enabled signals yet.
 */
void BackgroundWriterMain(void)
{
    // 注册信号处理函数
	pqsignal(SIGTERM, SignalHandlerForShutdownRequest);
    // ...

    // 错误处理

	/* Loop forever */
	for (;;)
	{
		bool		can_hibernate;
		int			rc;

		/* Clear any already-pending wakeups */
		ResetLatch(MyLatch);

		HandleMainLoopInterrupts();

		/* Do one cycle of dirty-buffer writing. */
		can_hibernate = BgBufferSync(&wb_context);
    
        // ...

        // 等待，直到收到信号或BgWriterDelay超时
        /* Sleep until we are signaled or BgWriterDelay has elapsed. */
		rc = WaitLatch(MyLatch,
					   WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH,
					   BgWriterDelay /* ms */ , WAIT_EVENT_BGWRITER_MAIN);

        // ...
    }
}
```

其主要实现是在`BgBufferSync`函数中实现的，让我们看一下这个函数。先思考一下函数设计的关键点，第1个问题，Buffer中页面那么多，从哪里开始？ 第2个问题，每次刷盘刷多少个页面？关于从哪里开始，这个问题，首先看bgwriter存在的意义，其中要避免缓冲区淘汰脏页面进行刷盘，所以，bgwriter扫描页面，最好是领先于时钟扫描算法一轮，这样才能起到效果。而上面讲到的2个参数就是解决每次刷多少个页面相关的参数。下面的函数中，会涉及到上面2个问题，计算每次刷多少个是最优的，在哪里开始扫描。具体的算法，我们这里不进行分析，可参考《PostgreSQL技术内幕：事务处理探索》第4.12.2章节。

```c++
/* BgBufferSync -- Write out some dirty buffers in the pool. */
bool BgBufferSync(WritebackContext *wb_context)
{
	/* info obtained from freelist.c */
	int			strategy_buf_id;
	uint32		strategy_passes;
	uint32		recent_alloc;

    // ...
	/*
	 * Find out where the freelist clock sweep currently is, and how many
	 * buffer allocations have happened since our last call.
	 */
	strategy_buf_id = StrategySyncStart(&strategy_passes, &recent_alloc);

    // 执行扫描，刷盘
    /* Execute the LRU scan */
	while (num_to_scan > 0 && reusable_buffers < upcoming_alloc_est)
	{
		int			sync_state = SyncOneBuffer(next_to_clean, true,
											   wb_context);

		if (++next_to_clean >= NBuffers)
		{
			next_to_clean = 0;
			next_passes++;
		}
		num_to_scan--;

		if (sync_state & BUF_WRITTEN)
		{
			reusable_buffers++;
			if (++num_written >= bgwriter_lru_maxpages)
			{
				BgWriterStats.m_maxwritten_clean++;
				break;
			}
		}
		else if (sync_state & BUF_REUSABLE)
			reusable_buffers++;
	}

    // ...
}

```

最后是将Buffer刷盘的实现，我们看一下`SyncOneBuffer`，作用是尝试将单个缓冲区页面刷入磁盘文件。
```c++
/* SyncOneBuffer -- process a single buffer during syncing. */
static int SyncOneBuffer(int buf_id, bool skip_recently_used, WritebackContext *wb_context)
{
	BufferDesc *bufHdr = GetBufferDescriptor(buf_id);
	int			result = 0;
	uint32		buf_state;
	BufferTag	tag;

	ReservePrivateRefCountEntry();

	/*
	 * Check whether buffer needs writing.
	 *
	 * We can make this check without taking the buffer content lock so long
	 * as we mark pages dirty in access methods *before* logging changes with
	 * XLogInsert(): if someone marks the buffer dirty just after our check we
	 * don't worry because our checkpoint.redo points before log record for
	 * upcoming changes and so we are not required to write such dirty buffer.
	 */
	buf_state = LockBufHdr(bufHdr);

	if (BUF_STATE_GET_REFCOUNT(buf_state) == 0 &&
		BUF_STATE_GET_USAGECOUNT(buf_state) == 0)
	{
		result |= BUF_REUSABLE;
	}
	else if (skip_recently_used)
	{
		/* Caller told us not to write recently-used buffers */
		UnlockBufHdr(bufHdr, buf_state);
		return result;
	}

	if (!(buf_state & BM_VALID) || !(buf_state & BM_DIRTY))
	{
		/* It's clean, so nothing to do */
		UnlockBufHdr(bufHdr, buf_state);
		return result;
	}

	/*
	 * Pin it, share-lock it, write it.  (FlushBuffer will do nothing if the
	 * buffer is clean by the time we've locked it.)
	 */
	PinBuffer_Locked(bufHdr);
	LWLockAcquire(BufferDescriptorGetContentLock(bufHdr), LW_SHARED);

	FlushBuffer(bufHdr, NULL);

	LWLockRelease(BufferDescriptorGetContentLock(bufHdr));

	tag = bufHdr->tag;

	UnpinBuffer(bufHdr, true);

	ScheduleBufferTagForWriteback(wb_context, &tag);

	return result | BUF_WRITTEN;
}
```

下面就是将Buffer写入磁盘的具体过程，详细可参考函数`FlushBuffer`：
1. 依据Buffer中的描述信息，打开指定的表文件
2. 校验并copy Buffer中待写入的数据
3. 将Buffer的数据写入打开的表文件中
```c++

static void
FlushBuffer(BufferDesc *buf, SMgrRelation reln)
{
	/* Find smgr relation for buffer */
	if (reln == NULL)
		reln = smgropen(buf->tag.rnode, InvalidBackendId);

	bufBlock = BufHdrGetBlock(buf);

	/*
	 * Update page checksum if desired.  Since we have only shared lock on the
	 * buffer, other processes might be updating hint bits in it, so we must
	 * copy the page to private storage if we do checksumming.
	 */
	bufToWrite = PageSetChecksumCopy((Page) bufBlock, buf->tag.blockNum);

	/* bufToWrite is either the shared buffer or a copy, as appropriate. */
	smgrwrite(reln,
			  buf->tag.forkNum,
			  buf->tag.blockNum,
			  bufToWrite,
			  false);

}
```