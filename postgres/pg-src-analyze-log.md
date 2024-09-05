## PostgreSQL源码分析——日志模块

日志模块是PostgreSQL中负责记录系统运行状态和错误信息的模块。它将系统运行过程中的关键信息记录到日志文件中，以便于系统管理员进行故障排查和性能优化。这里我们就分析一下其实现。

### 实现思路
在PG中，采用的是多进程架构，每个进程都会记录自己的日志信息，因此日志模块需要考虑如何将不同进程的日志信息进行统一管理，并且能够方便的查询和查看。因此，我们需要将不用进程的日志统一记录到一个日志文件中，而不是每个进程一个日志文件。这样，我们就可以通过查看一个日志文件来了解整个系统的运行状态。对此，PG中从postmaster进程中fork了一个日志进程logger，其他进程通过管道与logger进程通信，将日志信息发送给logger进程，logger进程再将日志信息写入日志文件中。

除了需要增加一个日志进程，还需要定义日志级别，解决日志发送端的问题，需要定义函数，各进程通过调用这个函数，将日志写入管道中。

### logger进程源码分析
1. 日志进程的启动，postmaster进程fork一个日志子进程
```c++
SysLoggerMain(int argc, char ** argv) (src\backend\postmaster\syslogger.c:303)
SysLogger_Start() (src\backend\postmaster\syslogger.c:652)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1113)
main(int argc, char ** argv) (src\backend\main\main.c:209)
```
主要代码在`src/backend/postmaster/syslogger.c`中，其他进程通过管道与日志进程通信，日志进程只读管道，其他进程将日志写入管道中。

```c++
int			syslogPipe[2] = {-1, -1};       // 管道，文件描述符

// 启动日志进程
int SysLogger_Start(void)
{
	if (syslogPipe[0] < 0)  // 如果管道未创建，则创建管道
	{
		if (pipe(syslogPipe) < 0)
			ereport(FATAL,
					(errcode_for_socket_access(),
					 errmsg("could not create pipe for syslog: %m")));
	}
    // 创建日志目录，如果不存在的话
    (void) MakePGDirectory(Log_directory);

	first_syslogger_file_time = time(NULL);

	filename = logfile_getname(first_syslogger_file_time, NULL);

	syslogFile = logfile_open(filename, "a", false);

	pfree(filename);

	switch ((sysloggerPid = fork_process()))    // fork一个日志子进程
	{
		case -1:
			ereport(LOG,
					(errmsg("could not fork system logger: %m")));
			return 0;

		case 0:     // logger子进程
			/* in postmaster child ... */
			InitPostmasterChild();

			/* Close the postmaster's sockets */
			ClosePostmasterPorts(true);

			/* Drop our connection to postmaster's shared memory, as well */
			dsm_detach_all();
			PGSharedMemoryDetach();

			/* do the work */
			SysLoggerMain(0, NULL);
			break;

		default:    // postmaster进程
			/* success, in postmaster */
            // 重定向，stdout,stderr输出到管道中
			/* now we redirect stderr, if not done already */
			if (!redirection_done)
			{
				ereport(LOG,
						(errmsg("redirecting log output to logging collector process"),
						 errhint("Future log output will appear in directory \"%s\".",
								 Log_directory)));

				fflush(stdout);
				if (dup2(syslogPipe[1], fileno(stdout)) < 0)
					ereport(FATAL,
							(errcode_for_file_access(),
							 errmsg("could not redirect stdout: %m")));
				fflush(stderr);
				if (dup2(syslogPipe[1], fileno(stderr)) < 0)
					ereport(FATAL,
							(errcode_for_file_access(),
							 errmsg("could not redirect stderr: %m")));
				/* Now we are done with the write end of the pipe. */
				close(syslogPipe[1]);
				syslogPipe[1] = -1;

				redirection_done = true;
			}

			/* postmaster will never write the file(s); close 'em */
			fclose(syslogFile);
			syslogFile = NULL;
			if (csvlogFile != NULL)
			{
				fclose(csvlogFile);
				csvlogFile = NULL;
			}
			return (int) sysloggerPid;
	}

	/* we should never reach here */
	return 0;
}

// logger子进程读管道，将读到的日志写入日志文件中
void SysLoggerMain(int argc, char *argv[])
{
	if (syslogPipe[1] >= 0)
		close(syslogPipe[1]);
	syslogPipe[1] = -1;      // logger进程只需要读管道，关闭写
	
    last_file_name = logfile_getname(first_syslogger_file_time, NULL);

	/* remember active logfile parameters */
	currentLogDir = pstrdup(Log_directory);
	currentLogFilename = pstrdup(Log_filename);
	currentLogRotationAge = Log_RotationAge;
	/* set next planned rotation time */
	set_next_rotation_time();
	update_metainfo_datafile();

    wes = CreateWaitEventSet(CurrentMemoryContext, 2);
	AddWaitEventToSet(wes, WL_LATCH_SET, PGINVALID_SOCKET, MyLatch, NULL);
	AddWaitEventToSet(wes, WL_SOCKET_READABLE, syslogPipe[0], NULL, NULL);

    	/* main worker loop */
	for (;;)
	{
        /* Clear any already-pending wakeups */
		ResetLatch(MyLatch);
        // 处理配置文件重载
        if (ConfigReloadPending)
		{
			ConfigReloadPending = false;
			ProcessConfigFile(PGC_SIGHUP);
            // 处理与日志相关的一些GUC参数变化
            // ...
        }

		rc = WaitEventSetWait(wes, cur_timeout, &event, 1, WAIT_EVENT_SYSLOGGER_MAIN);    
		if (rc == 1 && event.events == WL_SOCKET_READABLE)
		{
			int			bytesRead;
            // 读管道
			bytesRead = read(syslogPipe[0],
							 logbuffer + bytes_in_logbuffer,
							 sizeof(logbuffer) - bytes_in_logbuffer);
			if (bytesRead < 0)
			{
				if (errno != EINTR)
					ereport(LOG,
							(errcode_for_socket_access(),
							 errmsg("could not read from logger pipe: %m")));
			}
			else if (bytesRead > 0)
			{
                // 读取管道中的日志，并写入日志文件
				bytes_in_logbuffer += bytesRead;
				process_pipe_input(logbuffer, &bytes_in_logbuffer);
				continue;
			}
			else
			{
				/*
				 * Zero bytes read when select() is saying read-ready means
				 * EOF on the pipe: that is, there are no longer any processes
				 * with the pipe write end open.  Therefore, the postmaster
				 * and all backends are shut down, and we are done.
				 */
				pipe_eof_seen = true;

				/* if there's any data left then force it out now */
				flush_pipe_input(logbuffer, &bytes_in_logbuffer);
			}
		}            
    }
}
```

具体的，还需要一个协议，用于解析日志文件中的日志条目。也就是说，日志文件中的日志条目需要有一个协议，用于解析这些日志条目。其中从管道读数据后写入文件的调用栈如下：
```
libc.so.6!__GI__IO_fwrite(const void * buf, size_t size, size_t count, FILE * fp) (iofwrite.c:31)
write_syslogger_file(const char * buffer, int count, int destination) (src\backend\postmaster\syslogger.c:1099)
process_pipe_input(char * logbuffer, int * bytes_in_logbuffer) (src\backend\postmaster\syslogger.c:981)
SysLoggerMain(int argc, char ** argv) (src\backend\postmaster\syslogger.c:479)
SysLogger_Start() (src\backend\postmaster\syslogger.c:652)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1113)
main(int argc, char ** argv) (src\backend\main\main.c:209)
```

定义如下：
```c++

/*
 * Primitive protocol structure for writing to syslogger pipe(s).  The idea
 * here is to divide long messages into chunks that are not more than
 * PIPE_BUF bytes long, which according to POSIX spec must be written into
 * the pipe atomically.  The pipe reader then uses the protocol headers to
 * reassemble the parts of a message into a single string.  The reader can
 * also cope with non-protocol data coming down the pipe, though we cannot
 * guarantee long strings won't get split apart.
 *
 * We use non-nul bytes in is_last to make the protocol a tiny bit
 * more robust against finding a false double nul byte prologue. But
 * we still might find it in the len and/or pid bytes unless we're careful.
 */

#ifdef PIPE_BUF
/* Are there any systems with PIPE_BUF > 64K?  Unlikely, but ... */
#if PIPE_BUF > 65536
#define PIPE_CHUNK_SIZE  65536
#else
#define PIPE_CHUNK_SIZE  ((int) PIPE_BUF)
#endif
#else							/* not defined */
/* POSIX says the value of PIPE_BUF must be at least 512, so use that */
#define PIPE_CHUNK_SIZE  512
#endif

typedef struct
{
	char		nuls[2];		/* always \0\0 */
	uint16		len;			/* size of this chunk (counts data only) */
	int32		pid;			/* writer's pid */
	char		is_last;		/* last chunk of message? 't' or 'f' ('T' or
								 * 'F' for CSV case) */
	char		data[FLEXIBLE_ARRAY_MEMBER];	/* data payload starts here */
} PipeProtoHeader;

typedef union
{
	PipeProtoHeader proto;
	char		filler[PIPE_CHUNK_SIZE];
} PipeProtoChunk;

#define PIPE_HEADER_SIZE  offsetof(PipeProtoHeader, data)
#define PIPE_MAX_PAYLOAD  ((int) (PIPE_CHUNK_SIZE - PIPE_HEADER_SIZE))
```
因为是依赖于管道进行通信的，`PIPE_BUF`规定了内核的管道缓冲区大小，正常情况下是4096字节，但是有些系统可能会更大。如果日志条目超过这个字节数，那么就会分成多个chunk进行传输。

```c++
/* --------------------------------
 *		pipe protocol handling
 * --------------------------------
 */

/*
 * Process data received through the syslogger pipe.
 *
 * This routine interprets the log pipe protocol which sends log messages as
 * (hopefully atomic) chunks - such chunks are detected and reassembled here.
 *
 * The protocol has a header that starts with two nul bytes, then has a 16 bit
 * length, the pid of the sending process, and a flag to indicate if it is
 * the last chunk in a message. Incomplete chunks are saved until we read some
 * more, and non-final chunks are accumulated until we get the final chunk.
 *
 * All of this is to avoid 2 problems:
 * . partial messages being written to logfiles (messes rotation), and
 * . messages from different backends being interleaved (messages garbled).
 *
 * Any non-protocol messages are written out directly. These should only come
 * from non-PostgreSQL sources, however (e.g. third party libraries writing to
 * stderr).
 *
 * logbuffer is the data input buffer, and *bytes_in_logbuffer is the number
 * of bytes present.  On exit, any not-yet-eaten data is left-justified in
 * logbuffer, and *bytes_in_logbuffer is updated.
 */
static void
process_pipe_input(char *logbuffer, int *bytes_in_logbuffer)
{
	char	   *cursor = logbuffer;
	int			count = *bytes_in_logbuffer;
	int			dest = LOG_DESTINATION_STDERR;

	/* While we have enough for a header, process data... */
	while (count >= (int) (offsetof(PipeProtoHeader, data) + 1))
	{
		PipeProtoHeader p;
		int			chunklen;

		/* Do we have a valid header? */
		memcpy(&p, cursor, offsetof(PipeProtoHeader, data));
		if (p.nuls[0] == '\0' && p.nuls[1] == '\0' &&
			p.len > 0 && p.len <= PIPE_MAX_PAYLOAD &&
			p.pid != 0 &&
			(p.is_last == 't' || p.is_last == 'f' ||
			 p.is_last == 'T' || p.is_last == 'F'))
		{
			List	   *buffer_list;
			ListCell   *cell;
			save_buffer *existing_slot = NULL,
					   *free_slot = NULL;
			StringInfo	str;

			chunklen = PIPE_HEADER_SIZE + p.len;

			/* Fall out of loop if we don't have the whole chunk yet */
			if (count < chunklen)
				break;

			dest = (p.is_last == 'T' || p.is_last == 'F') ?
				LOG_DESTINATION_CSVLOG : LOG_DESTINATION_STDERR;

			/* Locate any existing buffer for this source pid */
			buffer_list = buffer_lists[p.pid % NBUFFER_LISTS];
			foreach(cell, buffer_list)
			{
				save_buffer *buf = (save_buffer *) lfirst(cell);

				if (buf->pid == p.pid)
				{
					existing_slot = buf;
					break;
				}
				if (buf->pid == 0 && free_slot == NULL)
					free_slot = buf;
			}

			if (p.is_last == 'f' || p.is_last == 'F')
			{
				/*
				 * Save a complete non-final chunk in a per-pid buffer
				 */
				if (existing_slot != NULL)
				{
					/* Add chunk to data from preceding chunks */
					str = &(existing_slot->data);
					appendBinaryStringInfo(str,
										   cursor + PIPE_HEADER_SIZE,
										   p.len);
				}
				else
				{
					/* First chunk of message, save in a new buffer */
					if (free_slot == NULL)
					{
						/*
						 * Need a free slot, but there isn't one in the list,
						 * so create a new one and extend the list with it.
						 */
						free_slot = palloc(sizeof(save_buffer));
						buffer_list = lappend(buffer_list, free_slot);
						buffer_lists[p.pid % NBUFFER_LISTS] = buffer_list;
					}
					free_slot->pid = p.pid;
					str = &(free_slot->data);
					initStringInfo(str);
					appendBinaryStringInfo(str,
										   cursor + PIPE_HEADER_SIZE,
										   p.len);
				}
			}
			else
			{
				/*
				 * Final chunk --- add it to anything saved for that pid, and
				 * either way write the whole thing out.
				 */
				if (existing_slot != NULL)
				{
					str = &(existing_slot->data);
					appendBinaryStringInfo(str,
										   cursor + PIPE_HEADER_SIZE,
										   p.len);
					write_syslogger_file(str->data, str->len, dest);
					/* Mark the buffer unused, and reclaim string storage */
					existing_slot->pid = 0;
					pfree(str->data);
				}
				else
				{
					/* The whole message was one chunk, evidently. */
					write_syslogger_file(cursor + PIPE_HEADER_SIZE, p.len,
										 dest);
				}
			}

			/* Finished processing this chunk */
			cursor += chunklen;
			count -= chunklen;
		}
		else
		{
			/* Process non-protocol data */

			/*
			 * Look for the start of a protocol header.  If found, dump data
			 * up to there and repeat the loop.  Otherwise, dump it all and
			 * fall out of the loop.  (Note: we want to dump it all if at all
			 * possible, so as to avoid dividing non-protocol messages across
			 * logfiles.  We expect that in many scenarios, a non-protocol
			 * message will arrive all in one read(), and we want to respect
			 * the read() boundary if possible.)
			 */
			for (chunklen = 1; chunklen < count; chunklen++)
			{
				if (cursor[chunklen] == '\0')
					break;
			}
			/* fall back on the stderr log as the destination */
			write_syslogger_file(cursor, chunklen, LOG_DESTINATION_STDERR);
			cursor += chunklen;
			count -= chunklen;
		}
	}

	/* We don't have a full chunk, so left-align what remains in the buffer */
	if (count > 0 && cursor != logbuffer)
		memmove(logbuffer, cursor, count);
	*bytes_in_logbuffer = count;
}

void
write_syslogger_file(const char *buffer, int count, int destination)
{
	int			rc;
	FILE	   *logfile;

	/*
	 * If we're told to write to csvlogFile, but it's not open, dump the data
	 * to syslogFile (which is always open) instead.  This can happen if CSV
	 * output is enabled after postmaster start and we've been unable to open
	 * csvlogFile.  There are also race conditions during a parameter change
	 * whereby backends might send us CSV output before we open csvlogFile or
	 * after we close it.  Writing CSV-formatted output to the regular log
	 * file isn't great, but it beats dropping log output on the floor.
	 *
	 * Think not to improve this by trying to open csvlogFile on-the-fly.  Any
	 * failure in that would lead to recursion.
	 */
	logfile = (destination == LOG_DESTINATION_CSVLOG &&
			   csvlogFile != NULL) ? csvlogFile : syslogFile;

	rc = fwrite(buffer, 1, count, logfile);

	/*
	 * Try to report any failure.  We mustn't use ereport because it would
	 * just recurse right back here, but write_stderr is OK: it will write
	 * either to the postmaster's original stderr, or to /dev/null, but never
	 * to our input pipe which would result in a different sort of looping.
	 */
	if (rc != count)
		write_stderr("could not write to log file: %s\n", strerror(errno));
}
```

### elog源码分析
这部分的代码主要再elog.h、elog.c中。定义了日志级别，日志打印的函数定义，日志项的定义。 这里是日志的写端，logger进程是日志的读端，读后写入日志文件中。

在各进程需要打印日志时，调用`ereport`，定义如下：
```c++
#define ereport_domain(elevel, domain, ...)	\
	do { \
		const int elevel_ = (elevel); \
		pg_prevent_errno_in_scope(); \
		if (errstart(elevel_, domain)) \
			__VA_ARGS__, errfinish(__FILE__, __LINE__, PG_FUNCNAME_MACRO); \
		if (elevel_ >= ERROR) \
			pg_unreachable(); \
	} while(0)

#define ereport(elevel, ...)	\
	ereport_domain(elevel, TEXTDOMAIN, __VA_ARGS__)
```
我们任意调用一个日志，具体的调用栈如下：
```c++
write_pipe_chunks(char * data, int len, int dest) (src\backend\utils\error\elog.c:3290)
send_message_to_server_log(ErrorData * edata) (src\backend\utils\error\elog.c:3191)
EmitErrorReport() (src\backend\utils\error\elog.c:1546)
errfinish(const char * filename, int lineno, const char * funcname) (src\backend\utils\error\elog.c:597)
ShowUsage(const char * title) (src\backend\tcop\postgres.c:4977)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1321)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (src\backend\tcop\postgres.c:4577)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4540)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4262)
ServerLoop() (src\backend\postmaster\postmaster.c:1748)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1420)
main(int argc, char ** argv) (src\backend\main\main.c:209)
```
将日志发送到logger进程中。
```c++
// Send data to the syslogger using the chunked protocol
static void
write_pipe_chunks(char *data, int len, int dest)
{
	PipeProtoChunk p;
	int			fd = fileno(stderr);
	int			rc;

	Assert(len > 0);

	p.proto.nuls[0] = p.proto.nuls[1] = '\0';
	p.proto.pid = MyProcPid;

	/* write all but the last chunk */
	while (len > PIPE_MAX_PAYLOAD)
	{
		p.proto.is_last = (dest == LOG_DESTINATION_CSVLOG ? 'F' : 'f');
		p.proto.len = PIPE_MAX_PAYLOAD;
		memcpy(p.proto.data, data, PIPE_MAX_PAYLOAD);
		rc = write(fd, &p, PIPE_HEADER_SIZE + PIPE_MAX_PAYLOAD);
		(void) rc;
		data += PIPE_MAX_PAYLOAD;
		len -= PIPE_MAX_PAYLOAD;
	}

	/* write the last chunk */
	p.proto.is_last = (dest == LOG_DESTINATION_CSVLOG ? 'T' : 't');
	p.proto.len = len;
	memcpy(p.proto.data, data, len);
	rc = write(fd, &p, PIPE_HEADER_SIZE + len);
	(void) rc;
}
```