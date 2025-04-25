## PostgreSQL中Latch实现分析

`Latch`是PostgreSQL中的一种多进程同步机制，提供休眠唤醒功能，比如，当前进程休眠指定的时间后唤醒，或者当前进程等待指定的信号后唤醒。

> 因`Latch`的实现依赖底层操作系统，这里仅分析Linux操作系统的实现。

### Latch类型
有两种类型的`Latch`：`本地Latch`和`共享Latch`。
- 其中`本地Latch`由`InitLatch`初始化，只能被同一进程设置，可用于等待信号到达，通过在信号处理程序中调用`SetLatch`。
- `共享Latch`位于共享内存，由postmaster进程在启动时通过`InitSharedLatch`初始化，在`共享Latch`可以被等待之前，必须使用`OwnLatch`将其与进程关联，只有拥有`Latch`的进程才能等待它，其可以被多个进程设置。

在使用上，`共享Latch`是其他进程在等待，如果你想唤醒其他进程，则需要获取其他进程的Latch对象，然后调用`SetLatch(其他进程的Latch对象)`，而`本地Latch`是自己进程使用`SetLatch(本地Latch对象)`唤醒自己。


`Latch`的定义：
```c++
typedef struct Latch
{
	sig_atomic_t is_set;       	  // 原子操作，是否被设置
	sig_atomic_t maybe_sleeping;  // 可能有进程正在休眠
	bool		 is_shared;  	  // 是否是共享的
	int			 owner_pid;  	  // 当前进程pid
} Latch;
```
`本地Latch`和`共享Latch`的初始化函数如下：
```c++
void InitLatch(Latch *latch)
{
	latch->is_set = false;
	latch->maybe_sleeping = false;
	latch->owner_pid = MyProcPid;   // 当前进程pid
	latch->is_shared = false;       // 本地Latch，设置为false
}

void InitSharedLatch(Latch *latch)
{
	latch->is_set = false;
	latch->maybe_sleeping = false;
	latch->owner_pid = 0;           // 共享Latch，owner_pid为0
	latch->is_shared = true;        // 共享Latch，设置为true
}
```
将`共享Latch`与进程关联，`OwnLatch`实现如下：
```c++
void OwnLatch(Latch *latch)
{
	Assert(latch->is_shared);  // 必须是共享Latch

	if (latch->owner_pid != 0) // 已经被关联
		elog(ERROR, "latch already owned");

	latch->owner_pid = MyProcPid;  // 关联当前进程
}
```

### Latch操作

有三种基本的`Latch`操作：
- `SetLatch`：设置`Latch`。
- `ResetLatch`：重置`Latch`，使其可再次设置。
- `WaitLatch`：等待`Latch`，直到被设置或者超时。

其中`WaitLatch`包括了超时机制，以及提供了postmaster子进程立即唤醒当postmaster进程终止时机制。

等待事件的正确模式：
```c
  for (;;)
  {
 	   ResetLatch();
 	   if (work to do) // 有工作要处理
 		   Do Stuff();
 	   WaitLatch();
  }
```
另一种方式是：
```c
 for (;;)
 {
	   if (work to do)
		   Do Stuff(); // in particular, exit loop if some condition satisfied
	   WaitLatch();
	   ResetLatch();
 }
```

#### SetLatch
设置`Latch`，即将`latch->is_set`置为`true`，唤醒等待该`Latch`的进程，分两种情况处理，一种是当前进程，唤醒方式可以是发送信号的方式或者通过自管道的方式；另一种是其他进程，可通过发送信号的方式进行唤醒（其他进程在创建之初会注册SIGURG信号处理函数`pqsignal(SIGURG, latch_sigurg_handler)`， `latch_sigurg_handler`函数通过向自管道写端写入一个字节，触发本地epoll事件）。具体实现上，是通过自管道的形式，通过向自管道写入一个字节，然后`WaitLatch`通过`epoll`监听自管道读端，当自管道有数据可读时，触发`epoll`事件，从而唤醒处在`WaitLatch`等待状态的进程。其实现如下：
```c++
void SetLatch(Latch *latch)
{
	pg_memory_barrier();      // 内存屏障，确保之前的所有内存操作已完成
	if (latch->is_set)  // Latch已经设置，直接返回
		return;

	latch->is_set = true;    // 设置is_set为true

	pg_memory_barrier();  // 内存屏障，确保该修改立即可见
	if (!latch->maybe_sleeping)
		return;

	pid_t owner_pid = latch->owner_pid;
	if (owner_pid == 0)
		return;
	else if (owner_pid == MyProcPid) // 如果是当前进程
	{
#if defined(WAIT_USE_SELF_PIPE)
		if (waiting)
			sendSelfPipeByte(); // 如果当前进程在waiting状态则向当前进程发送一个字节用于唤醒WaitLatch
#else
		if (waiting)        // 如果当前进程在waiting状态则向当前进程发送SIGURG信号
			kill(MyProcPid, SIGURG);
#endif
	}
	else
		kill(owner_pid, SIGURG);    // 向owner_pid进程发送SIGURG信号
}
```

#### WaitLatch
`WaitLatch`作用是阻塞当前进程，直到`Latch`被设置（`is_set = true`）或者超时，或者postmaster进程终止。其参数中，`latch`是要等待的`Latch`对象，`wakeEvents`是等待的事件类型掩码，`timeout`是超时时间，`wait_event_info`是等待事件信息。具体实现时，就是调用`epoll`来等待之前注册过的事件，其实现如下：
```c++
int WaitLatch(Latch *latch, int wakeEvents, long timeout, uint32 wait_event_info)
{
	WaitEvent	event;

	if (!(wakeEvents & WL_LATCH_SET)) // 如果等待事件中没有指定WL_LATCH_SET，则latch置为NULL，无需监听Latch
		latch = NULL;
	ModifyWaitEvent(LatchWaitSet, LatchWaitSetLatchPos, WL_LATCH_SET, latch);
	LatchWaitSet->exit_on_postmaster_death = ((wakeEvents & WL_EXIT_ON_PM_DEATH) != 0);

	if (WaitEventSetWait(LatchWaitSet,
						 (wakeEvents & WL_TIMEOUT) ? timeout : -1,
						 &event, 1,
						 wait_event_info) == 0)
		return WL_TIMEOUT;    // 超时发生
	else
		return event.events;  // 事件触发
}

// 等待事件发生，或者超时。至多nevents个被触发的事件返回。如果timeout为-1，则一直阻塞直到事件发生
// 如果timeout为0，则检查socket是否就绪，不阻塞；如果timeout大于0，则阻塞timeout毫秒。
int WaitEventSetWait(WaitEventSet *set, long timeout, WaitEvent *occurred_events, int nevents, uint32 wait_event_info)
{
	int			returned_events = 0;
	instr_time	start_time;
	instr_time	cur_time;
	long		cur_timeout = -1;

	if (timeout >= 0) // 如果timeout大于0，则记录当前时间以计算剩余时间
	{
		INSTR_TIME_SET_CURRENT(start_time);
		cur_timeout = timeout;
	}

	pgstat_report_wait_start(wait_event_info);

	waiting = true;   // 标记当前进程正在等待

	while (returned_events == 0)
	{
		if (set->latch && !set->latch->is_set)
		{
			set->latch->maybe_sleeping = true;  /* about to sleep on a latch */
			pg_memory_barrier();
			/* and recheck */
		}

		if (set->latch && set->latch->is_set) // 如果latch已经设置，则返回WL_LATCH_SET事件
		{
			occurred_events->fd = PGINVALID_SOCKET;
			occurred_events->pos = set->latch_pos;
			occurred_events->user_data = set->events[set->latch_pos].user_data;
			occurred_events->events = WL_LATCH_SET;
			occurred_events++;
			returned_events++;

			set->latch->maybe_sleeping = false;  /* could have been set above */

			break;   // 跳出等待，直接返回
		}

		// 等待事件发生，或者超时。
		int rc = WaitEventSetWaitBlock(set, cur_timeout, occurred_events, nevents);

		if (set->latch)
		{
			Assert(set->latch->maybe_sleeping);
			set->latch->maybe_sleeping = false;
		}

		if (rc == -1)
			break;			// 超时发生，跳出等待
		else
			returned_events = rc;

		if (returned_events == 0 && timeout >= 0) // 更新剩余时间，超时用
		{
			INSTR_TIME_SET_CURRENT(cur_time);
			INSTR_TIME_SUBTRACT(cur_time, start_time);
			cur_timeout = timeout - (long) INSTR_TIME_GET_MILLISEC(cur_time);
			if (cur_timeout <= 0)
				break;
		}
	}

	waiting = false;  // 标记当前进程不再处于等待状态

	pgstat_report_wait_end();

	return returned_events;  // 返回触发的事件数量
}

// 使用epoll实现等待
static inline int WaitEventSetWaitBlock(WaitEventSet *set, int cur_timeout, WaitEvent *occurred_events, int nevents)
{
	int			returned_events = 0;
	WaitEvent  *cur_event;
	struct epoll_event *cur_epoll_event;

	// 等待事件发生，或者超时。
	int rc = epoll_wait(set->epoll_fd, set->epoll_ret_events, Min(nevents, set->nevents_space), cur_timeout);

	if (rc < 0)  // 如果返回值小于0，则检查错误码，如果是EINTR则继续等待，否则报错
	{
		if (errno != EINTR)
		{
			waiting = false;
			ereport(ERROR, (errcode_for_socket_access(), errmsg("%s() failed: %m","epoll_wait")));
		}
		return 0;
	}
	else if (rc == 0)
		return -1;  // 超时发生

	// 至少一个事件发生，遍历epoll返回的事件，处理事件
	for (cur_epoll_event = set->epoll_ret_events; cur_epoll_event < (set->epoll_ret_events + rc) && returned_events < nevents; cur_epoll_event++)
	{
		/* epoll's data pointer is set to the associated WaitEvent */
		cur_event = (WaitEvent *) cur_epoll_event->data.ptr;

		occurred_events->pos = cur_event->pos;
		occurred_events->user_data = cur_event->user_data;
		occurred_events->events = 0;


		if (cur_event->events == WL_LATCH_SET &&
			cur_epoll_event->events & (EPOLLIN | EPOLLERR | EPOLLHUP))
		{
			// 处理Latch事件

			drain();  // 清空signalfd或者清空自管道中的数据，避免重复触发事件

			if (set->latch && set->latch->is_set) // 如果latch已经设置，则返回WL_LATCH_SET事件
			{
				occurred_events->fd = PGINVALID_SOCKET;
				occurred_events->events = WL_LATCH_SET;
				occurred_events++;
				returned_events++;
			}
		}
		else if (cur_event->events == WL_POSTMASTER_DEATH &&
				 cur_epoll_event->events & (EPOLLIN | EPOLLERR | EPOLLHUP))
		{
			// 处理postmaster进程终止事件，二次确认postmaster进程是否存活，如果存活则继续等待，否则返回WL_POSTMASTER_DEATH事件
			if (!PostmasterIsAliveInternal())
			{
				if (set->exit_on_postmaster_death) // 如果需要退出（WL_EXIT_ON_PM_DEATH），则退出进程，否则返回WL_POSTMASTER_DEATH事件
					proc_exit(1);
				occurred_events->fd = PGINVALID_SOCKET;
				occurred_events->events = WL_POSTMASTER_DEATH;
				occurred_events++;
				returned_events++;
			}
		}
		else if (cur_event->events & (WL_SOCKET_READABLE | WL_SOCKET_WRITEABLE))
		{
			// 处理socket事件
			if ((cur_event->events & WL_SOCKET_READABLE) &&
				(cur_epoll_event->events & (EPOLLIN | EPOLLERR | EPOLLHUP)))
			{
				/* data available in socket, or EOF */
				occurred_events->events |= WL_SOCKET_READABLE;
			}

			if ((cur_event->events & WL_SOCKET_WRITEABLE) &&
				(cur_epoll_event->events & (EPOLLOUT | EPOLLERR | EPOLLHUP)))
			{
				/* writable, or EOF */
				occurred_events->events |= WL_SOCKET_WRITEABLE;
			}

			if (occurred_events->events != 0)
			{
				occurred_events->fd = cur_event->fd;
				occurred_events++;
				returned_events++;
			}
		}
	}

	return returned_events;
}
```

其中需要等待事件的类型掩码如下，需要注意`WL_POSTMASTER_DEATH`以及`WL_EXIT_ON_PM_DEATH`的区别
```c++
/* Bitmasks for events that may wake-up WaitLatch(), WaitLatchOrSocket(), or WaitEventSetWait(). */
#define WL_LATCH_SET		 (1 << 0)   // latch设置
#define WL_SOCKET_READABLE	 (1 << 1)   // socket读
#define WL_SOCKET_WRITEABLE  (1 << 2)   // socket写
#define WL_TIMEOUT			 (1 << 3)	// 超时
#define WL_POSTMASTER_DEATH  (1 << 4)   // postmaster进程终止时唤醒子进程，子进程需要进行退出前操作
#define WL_EXIT_ON_PM_DEATH	 (1 << 5)   // postmaster终止时子进程直接退出
#define WL_SOCKET_CONNECTED  WL_SOCKET_WRITEABLE  // socket连接成功
#define WL_SOCKET_MASK		(WL_SOCKET_READABLE | WL_SOCKET_WRITEABLE | WL_SOCKET_CONNECTED)
```

#### ResetLatch
将`Latch`中的标识位重置为`false`。
```c++
void ResetLatch(Latch *latch)
{
	latch->is_set = false;  // 重置latch为false
	pg_memory_barrier();  // 内存屏障，确保latch->is_set的写操作在后续的读操作之前完成
}
```

### WaitEventSet
`WaitEventSet`是一个事件集合，用于等待多个事件，其定义如下：
```c++
struct WaitEventSet
{
	int			nevents;		/* 注册的事件数量 */
	int			nevents_space;	/* 最多支持的事件数量 */

	WaitEvent  *events;  // 存储等待事件的数组

	Latch	   *latch; // 如果wait event中指定了WL_LATCH_SET，则latch指向该latch，latch_pos是events数组中的偏移量
	int			latch_pos;

	bool		exit_on_postmaster_death; // 如果postmaster终止则立即退出，WL_EXIT_ON_PM_DEATH会转为为WL_POSTMASTER_DEATH事件，如果该标志位为true，则检测到postmaster终止时立即退出，否则返回

#if defined(WAIT_USE_EPOLL)
	int			epoll_fd;  // epoll文件描述符
	/* epoll_wait returns events in a user provided arrays, allocate once */
	struct epoll_event *epoll_ret_events; 
};
```

#### CreateWaitEventSet
创建一个等待事件集合，其中最重要的是创建`epoll`，其实现如下
```c++
WaitEventSet *CreateWaitEventSet(MemoryContext context, int nevents)
{
	Size		sz = 0;

	/*
	 * Use MAXALIGN size/alignment to guarantee that later uses of memory are
	 * aligned correctly. E.g. epoll_event might need 8 byte alignment on some
	 * platforms, but earlier allocations like WaitEventSet and WaitEvent
	 * might not be sized to guarantee that when purely using sizeof().
	 */
	sz += MAXALIGN(sizeof(WaitEventSet));
	sz += MAXALIGN(sizeof(WaitEvent) * nevents);
	sz += MAXALIGN(sizeof(struct epoll_event) * nevents);

	char *data = (char *) MemoryContextAllocZero(context, sz);

	WaitEventSet *set = (WaitEventSet *) data;
	data += MAXALIGN(sizeof(WaitEventSet));

	set->events = (WaitEvent *) data;
	data += MAXALIGN(sizeof(WaitEvent) * nevents);

	set->epoll_ret_events = (struct epoll_event *) data;
	data += MAXALIGN(sizeof(struct epoll_event) * nevents);

	set->latch = NULL;
	set->nevents_space = nevents;
	set->exit_on_postmaster_death = false;

	if (!AcquireExternalFD())
	{
		/* treat this as though epoll_create1 itself returned EMFILE */
		elog(ERROR, "epoll_create1 failed: %m");
	}
	set->epoll_fd = epoll_create1(EPOLL_CLOEXEC); // 创建epoll
	if (set->epoll_fd < 0)
	{
		ReleaseExternalFD();
		elog(ERROR, "epoll_create1 failed: %m");
	}

	return set;
}
```

#### AddWaitEventToSet
向等待事件集合中添加一个等待事件，具体实现就是注册epoll事件，其实现如下：
```c++
int AddWaitEventToSet(WaitEventSet *set, uint32 events, pgsocket fd, Latch *latch, void *user_data)
{
	WaitEvent  *event;
	if (events == WL_EXIT_ON_PM_DEATH) // 如果是WL_EXIT_ON_PM_DEATH，则转为WL_POSTMASTER_DEATH事件，并设置exit_on_postmaster_death为true
	{
		events = WL_POSTMASTER_DEATH;
		set->exit_on_postmaster_death = true;
	}

	if (latch)
	{
		if (latch->owner_pid != MyProcPid)
			elog(ERROR, "cannot wait on a latch owned by another process");
		if (set->latch)
			elog(ERROR, "cannot wait on more than one latch");
		if ((events & WL_LATCH_SET) != WL_LATCH_SET)
			elog(ERROR, "latch events only support being set");
	}
	else
	{
		if (events & WL_LATCH_SET)
			elog(ERROR, "cannot wait on latch without a specified latch");
	}

	/* waiting for socket readiness without a socket indicates a bug */
	if (fd == PGINVALID_SOCKET && (events & WL_SOCKET_MASK))
		elog(ERROR, "cannot wait on socket event without a socket");

	event = &set->events[set->nevents]; // 当前事件在事件数组中的地址
	event->pos = set->nevents++;   // 记录当前事件在事件集合中的位置
	event->fd = fd;    // 会被注册到epoll中
	event->events = events;
	event->user_data = user_data;

	if (events == WL_LATCH_SET)  // 如果添加的是latch事件
	{
		set->latch = latch;
		set->latch_pos = event->pos;
#if defined(WAIT_USE_SELF_PIPE)        
		event->fd = selfpipe_readfd;   // latch事件fd使用自管道的读端，这里会被注册到epoll事件中监听，当latch被设置时，会向管道写端写入一个字节，从而epoll可以监听到该事件，从而触发事件
#elif defined(WAIT_USE_SIGNALFD)
		event->fd = signal_fd;     // latch事件fd使用信号文件描述符
#else
		event->fd = PGINVALID_SOCKET;
#ifdef WAIT_USE_EPOLL
		return event->pos;
#endif
#endif
	}
	else if (events == WL_POSTMASTER_DEATH) // 如果添加的是postmaster终止事件
	{
		event->fd = postmaster_alive_fds[POSTMASTER_FD_WATCH];
	}

	WaitEventAdjustEpoll(set, event, EPOLL_CTL_ADD); // 注册事件到epoll中
	return event->pos;
}
```

### Latch实现

Latch的实现依赖操作系统底层机制，我们只分析Linux操作系统的清理，在Linux操作系统中，其实现方式之一是`self-pipe`，即通过一个自管道的方式，实现进程之间的通信，实现原理是：每个进程都创建一个管道`pipe`，这个管道写端用于触发事件，读端通过`select/poll/epoll`监听。如果需要触发事件，则向管道写端写入一个字节，通过`select/poll/epoll`监听到该事件，则触发事件。
```c++
static int	selfpipe_readfd = -1;  // 自管道读端
static int	selfpipe_writefd = -1; // 自管道写端

/* Process owning the self-pipe --- needed for checking purposes */
static int	selfpipe_owner_pid = 0;

/* Private function prototypes */
static void latch_sigurg_handler(SIGNAL_ARGS);  // SIGURG信号处理函数
static void sendSelfPipeByte(void);    // 向自管道写端写入一个字节
```
初始化Latch支持，创建一个自管道，并设置读端为非阻塞，写端为非阻塞，并设置读端为`FD_CLOEXEC`，表示在fork时，不会继承该文件描述符，同时设置写端为`FD_CLOEXEC`，表示在fork时，不会继承该文件描述符。
```c++
void InitializeLatchSupport(void)
{
	int			pipefd[2]; // 创建一个自管道
	if (IsUnderPostmaster)
	{
		// 继承自postmaster的自管道，关闭自管道
		if (selfpipe_owner_pid != 0)
		{
			/* Release postmaster's pipe FDs; ignore any error */
			(void) close(selfpipe_readfd);
			(void) close(selfpipe_writefd);
			/* Clean up, just for safety's sake; we'll set these below */
			selfpipe_readfd = selfpipe_writefd = -1;
			selfpipe_owner_pid = 0;
			/* Keep fd.c's accounting straight */
			ReleaseExternalFD();
			ReleaseExternalFD();
		}
	}

	if (pipe(pipefd) < 0) // 创建一个自管道
		elog(FATAL, "pipe() failed: %m");
	if (fcntl(pipefd[0], F_SETFL, O_NONBLOCK) == -1)  // 设置读端为非阻塞
		elog(FATAL, "fcntl(F_SETFL) failed on read-end of self-pipe: %m");
	if (fcntl(pipefd[1], F_SETFL, O_NONBLOCK) == -1) // 设置写端为非阻塞
		elog(FATAL, "fcntl(F_SETFL) failed on write-end of self-pipe: %m");
	if (fcntl(pipefd[0], F_SETFD, FD_CLOEXEC) == -1)
		elog(FATAL, "fcntl(F_SETFD) failed on read-end of self-pipe: %m");
	if (fcntl(pipefd[1], F_SETFD, FD_CLOEXEC) == -1)
		elog(FATAL, "fcntl(F_SETFD) failed on write-end of self-pipe: %m");

	selfpipe_readfd = pipefd[0];     // 自管道读端
	selfpipe_writefd = pipefd[1];    // 自管道写端
	selfpipe_owner_pid = MyProcPid;  // 自管道拥有者，当前进程

	/* Tell fd.c about these two long-lived FDs */
	ReserveExternalFD();
	ReserveExternalFD();

	pqsignal(SIGURG, latch_sigurg_handler); // 注册SIGURG信号，处理信号，如果收到SIGURG信号，并且当前进程正在等待，则调用sendSelfPipeByte()向当前进程发送一个字节，用于唤醒WaitLatch()
}

// SIGURG信号处理函数
static void latch_sigurg_handler(SIGNAL_ARGS)
{
	int	save_errno = errno;

	if (waiting)
		sendSelfPipeByte();   // 向自管道写端写入一个字节，用于唤醒WaitLatch()

	errno = save_errno;
}
```
自管道的读端通过`poll/epoll`监听，下面的代码可以看到自管道的读端被注册到`epoll`中。
```c++
int AddWaitEventToSet(WaitEventSet *set, uint32 events, pgsocket fd, Latch *latch, void *user_data)
{
	WaitEvent  *event;
	// ...
	event = &set->events[set->nevents];
	event->pos = set->nevents++;
	event->fd = fd;
	event->events = events;
	event->user_data = user_data;

	if (events == WL_LATCH_SET)
	{
		set->latch = latch;
		set->latch_pos = event->pos;
#if defined(WAIT_USE_SELF_PIPE)
		event->fd = selfpipe_readfd;  // 自管道读端
#elif defined(WAIT_USE_SIGNALFD)
		event->fd = signal_fd;
#else
		event->fd = PGINVALID_SOCKET;
#ifdef WAIT_USE_EPOLL
		return event->pos;
#endif
#endif
	}
	else if (events == WL_POSTMASTER_DEATH)
	{
#ifndef WIN32
		event->fd = postmaster_alive_fds[POSTMASTER_FD_WATCH];
#endif
	}
}

static void WaitEventAdjustEpoll(WaitEventSet *set, WaitEvent *event, int action)
{
	// ...
	rc = epoll_ctl(set->epoll_fd, action, event->fd, &epoll_ev);
	// ...
}
```

### PostgreSQL Latch初始化流程
我们看一下PostgreSQL Latch的初始化流程，主要分析本地Latch以及共享Latch的初始化流程。
```c++
main(int argc, char *argv[])
--> MyProcPid = getpid();       // 获取当前进程pid
--> PostmasterMain(argc, argv); // postmaster主进程启动
    --> reset_shared();         // 设置共享内存和信号量
        --> CreateSharedMemoryAndSemaphores();
            --> InitProcGlobal();  // Initialize the global process table during postmaster startup
                --> for (i < MaxBackends + NUM_AUXILIARY_PROCS)
                        InitSharedLatch(&(procs[i].procLatch)); // 对每个后端进程初始化Latch
    --> ServerLoop();
        --> BackendStartup(port);
            --> InitPostmasterChild();
                --> InitializeLatchSupport();  /* Initialize process-local latch support */
				    --> pipe(pipefd)  // 创建一个自管道
					--> pqsignal(SIGURG, latch_sigurg_handler);    // 注册信号处理函数，通过自管道触发事件，唤醒WaitLatch()
	            --> MyLatch = &LocalLatchData;
	            --> InitLatch(MyLatch);     // 初始化当前postmaster子进程的本地Latch
	            --> InitializeLatchWaitSet();
                    --> LatchWaitSet = CreateWaitEventSet(TopMemoryContext, 2); // 创建一个WaitEventSet，用于等待事件
						--> set->epoll_fd = epoll_create1(EPOLL_CLOEXEC); // 创建一个epoll实例
                    --> AddWaitEventToSet(LatchWaitSet, WL_LATCH_SET, PGINVALID_SOCKET, MyLatch, NULL); // 添加WL_LATCH_SET事件到WaitEventSet中
                    --> AddWaitEventToSet(LatchWaitSet, WL_EXIT_ON_PM_DEATH, PGINVALID_SOCKET, NULL, NULL); // 如果是postmaster子进程则添加WL_EXIT_ON_PM_DEATH事件到WaitEventSet中
                        --> WaitEventAdjustEpoll(set, event, EPOLL_CTL_ADD);
                            --> epoll_ctl(set->epoll_fd, action, event->fd, &epoll_ev);
            --> BackendInitialize(port);
				--> pq_init();   // 初始化libpq
					--> FeBeWaitSet = CreateWaitEventSet(TopMemoryContext, 3); // 创建一个WaitEventSet，用于等待事件
					--> AddWaitEventToSet(FeBeWaitSet, WL_SOCKET_WRITEABLE, MyProcPort->sock, NULL, NULL);  // 添加WL_SOCKET_WRITEABLE事件到WaitEventSet中，套接字可写事件
					--> AddWaitEventToSet(FeBeWaitSet, WL_LATCH_SET, PGINVALID_SOCKET, MyLatch, NULL); // 添加WL_LATCH_SET事件到WaitEventSet中，用于等待本地Latch
					--> AddWaitEventToSet(FeBeWaitSet, WL_POSTMASTER_DEATH, PGINVALID_SOCKET, NULL, NULL);  // 添加postmaster进程终止事件
            --> BackendRun(port);
                --> PostgresMain(ac, av, port->database_name, port->user_name);
                    --> InitProcess();
                        --> OwnLatch(&MyProc->procLatch);   // Acquire ownership of the PGPROC's latch
                        --> SwitchToSharedLatch(); // 从本地Latch切换到共享Latch
                            --> MyLatch = &MyProc->procLatch;
                            --> ModifyWaitEvent(FeBeWaitSet, FeBeWaitSetLatchPos, WL_LATCH_SET, MyLatch);
                            --> SetLatch(MyLatch);
                    --> InitPostgres(dbname, InvalidOid, username, InvalidOid, NULL, false);
```

其中初始化共享Latch流程的关键代码如下：
```c++
/* Pointers to shared-memory structures */
PROC_HDR   *ProcGlobal = NULL;

void InitProcGlobal(void)
{
	/* Create the ProcGlobal shared structure */
	ProcGlobal = (PROC_HDR *) ShmemInitStruct("Proc Header", sizeof(PROC_HDR), &found);

	PGPROC	   *procs = (PGPROC *) ShmemAlloc(TotalProcs * sizeof(PGPROC));
	MemSet(procs, 0, TotalProcs * sizeof(PGPROC));
	ProcGlobal->allProcs = procs;

	for (i = 0; i < TotalProcs; i++)
	{
		/* Common initialization for all PGPROCs, regardless of type. */

		/*
		 * Set up per-PGPROC semaphore, latch, and fpInfoLock.  Prepared xact
		 * dummy PGPROCs don't need these though - they're never associated
		 * with a real process
		 */
		if (i < MaxBackends + NUM_AUXILIARY_PROCS)
		{
			procs[i].sem = PGSemaphoreCreate();
			InitSharedLatch(&(procs[i].procLatch));
			LWLockInitialize(&(procs[i].fpInfoLock), LWTRANCHE_LOCK_FASTPATH);
		}

        // ...
    }    

    // ...
}
```

每个后端进程都有自己的PGPROC结构体，其中包含一个Latch结构。
```c++
// Each backend has a PGPROC struct in shared memory
struct PGPROC
{
	/* proc->links MUST BE FIRST IN STRUCT (see ProcSleep,ProcWakeup,etc) */
	SHM_QUEUE	links;			/* list link if process is in a list */
	PGPROC	  **procgloballist; /* procglobal list that owns this PGPROC */

	PGSemaphore sem;			/* ONE semaphore to sleep on */
	ProcWaitStatus waitStatus;

	Latch		procLatch;		/* generic latch for process */

    // ... 
}
```

参考文档：
[Postgresql中latch实现中self-pipe trick解决什么问题](https://blog.csdn.net/jackgo73/article/details/123580226)
[Postgresql源码（40）Latch的原理分析和应用场景](https://blog.csdn.net/jackgo73/article/details/123600955)