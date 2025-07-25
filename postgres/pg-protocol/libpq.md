libpq是应用程序员使用PostgreSQL的C接口。libpq是一个库函数的集合，它们允许客户端程序传递查询给PostgreSQL后端服务器并且接收这些查询的结果。通俗一点，就是客户端通过libpq与数据库服务器进行交互，比如发送查询，接收查询结果，负责处理PostgreSQL通信协议。

libpq分为两部分，一部分是提供给客户端的接口，另一部分是服务端处理客户端的请求。比较重要的过程是定义客户端与服务端的通信协议，即交互过程。比如建立连接阶段，如何进行认证，正常阶段如何接受客户端并解析客户端的请求，服务端如何向客户端发送结果，如何结束查询等。

### 代码分析
下面分析一些重要的接口函数。比如客户端如何连接数据库服务器，如何发送查询，如何接收查询结果。本文不对libpq协议进行完整的分析，而且通过对关键代码的分析，理解libpq的工作原理以及libpq的底层实现。

#### PGconn
`PGconn`是一个非常重要的结构体，保存了前端与服务端连接的所有状态信息。包括连接的参数，连接的状态，待发送的消息，接收的消息等等。
```c++
typedef struct pg_conn PGconn;

struct pg_conn
{
	// 连接选项
	char	   *pghost;			/* the machine on which the server is running */
	char	   *pghostaddr;		/* the numeric IP address of the machine on which the server is running. */
	char	   *pgport;			// 服务端端口号
	char	   *appname;		/* application name */
	char	   *dbName;			/* database name */
	char	   *replication;	/* connect as the replication standby? */
	char	   *pguser;			/* Postgres username and password, if any */
	char	   *pgpass;
	char	   *pgpassfile;		/* path to a file containing password(s) */

	/* Callback procedures for notice message processing */
	PGNoticeHooks noticeHooks;

	/* Event procs registered via PQregisterEventProc */
	PGEvent    *events;			/* expandable array of event data */
	int			nEvents;		/* number of active events */
	int			eventArraySize; /* allocated array size */

	PGnotify   *notifyHead;		/* oldest unreported Notify msg */
	PGnotify   *notifyTail;		/* newest unreported Notify msg */
	
	pgsocket	sock;			/* FD for socket, PGINVALID_SOCKET if unconnected */

	// 输入缓冲区，输出缓冲区
	/* Buffer for data received from backend and not yet processed */
	char	   *inBuffer;		/* currently allocated buffer */
	int			inBufSize;		/* allocated size of buffer */
	int			inStart;		/* offset to first unconsumed data in buffer */
	int			inCursor;		/* next byte to tentatively consume */
	int			inEnd;			/* offset to first position after avail data */

	/* Buffer for data not yet sent to backend */
	char	   *outBuffer;		/* currently allocated buffer */
	int			outBufSize;		/* allocated size of buffer */
	int			outCount;		/* number of chars waiting in buffer */

	/* State for constructing messages in outBuffer */
	int			outMsgStart;	/* offset to msg start (length word); if -1,
								 * msg has no length word */
	int			outMsgEnd;		/* offset to msg end (so far) */

	/* Row processor interface workspace */
	PGdataValue *rowBuf;		/* array for passing values to rowProcessor */
	int			rowBufLen;		/* number of entries allocated in rowBuf */

	PGresult   *result;			/* result being constructed */

	// ...
}
```

#### PQconnectdbParams
`PQconnectdbParams`开启一个到数据库服务器的新连接。客户端通过libpq与数据库服务器进行交互，第一步就是建立到数据库服务器的连接。连接的底层其实就是socket连接，可以看到，代码中调用了`connect`函数，这个函数就是用来建立socket连接的。

```c++
PQconnectdbParams(const char *const *keywords, const char *const *values, int expand_dbname)
--> PQconnectStartParams(keywords, values, expand_dbname);
    --> connectDBStart(conn)
        --> PQconnectPoll(conn)
            --> connect(conn->sock, addr_cur->ai_addr,addr_cur->ai_addrlen)
--> connectDBComplete(conn)
```
其主要实现如下：
```c++
// 开启一个到数据库服务器的新连接
PGconn *PQconnectdbParams(const char *const *keywords, const char *const *values, int expand_dbname)
{
	PGconn	   *conn = PQconnectStartParams(keywords, values, expand_dbname);

	if (conn && conn->status != CONNECTION_BAD)
		(void) connectDBComplete(conn);

	return conn;

}

// 连接到数据库服务器
PGconn *PQconnectStartParams(const char *const *keywords, const char *const *values, int expand_dbname)
{
	PGconn	   *conn;
	PQconninfoOption *connOptions;

	conn = makeEmptyPGconn();  // 创建一个空的PGconn结构体

	// 解析连接参数，比如host、port、dbname、user、password等
	connOptions = conninfo_array_parse(keywords, values, &conn->errorMessage, true, expand_dbname);
	if (connOptions == NULL)
	{
		conn->status = CONNECTION_BAD;
		return conn;
	}

	/* Move option values into conn structure */
	if (!fillPGconn(conn, connOptions))
	{
		PQconninfoFree(connOptions);
		return conn;
	}

	PQconninfoFree(connOptions); 	/* Free the option info - all is in conn now */

	if (!connectOptions2(conn))  	/* Compute derived options */
		return conn;

	if (!connectDBStart(conn))  	/* Connect to the database */
	{
		/* Just in case we failed to set it in connectDBStart */
		conn->status = CONNECTION_BAD;
	}

	return conn;
}
```

#### PGfinish
`PGfinish`关闭一个数据库连接，释放资源。关闭连接的底层其实就是关闭socket连接，可以看到，代码中调用了`close`函数，这个函数就是用来关闭socket连接的。
```c++
PQfinish(PGconn *conn)
--> closePGconn(conn);
	--> sendTerminateConn(conn);
		--> pqPutMsgStart('X', conn);    // 发送一个终止连接的消息给数据库服务器
		--> pqPutMsgEnd(conn);
		--> pqFlush(conn);  
	--> pqDropConnection(conn, true);
		--> pqsecure_close(conn);      // 关闭SSL安全连接
		--> closesocket(conn->sock);   // 关闭socket连接
	--> pqClearAsyncResult(conn);
	--> resetPQExpBuffer(&conn->errorMessage);
	--> release_conn_addrinfo(conn);
	--> pqDropServerData(conn);  // 清理所有连接状态数据
--> freePGconn(conn);  // 释放内存资源
```

服务器端收到`X`消息，关闭连接的代码如下：
```c++
PostgresMain()
{
		firstchar = ReadCommand(&input_message);
		switch (firstchar)
		{
			case 'X':
				if (whereToSendOutput == DestRemote) // 阻止ereport发送消息到客户端
					whereToSendOutput = DestNone;

				proc_exit(0);  // 退出进程
}
```
前面我们分析了建立的建立与关闭。下面我们分析一下怎么发送消息到数据库，怎么接收消息。
#### PQexec
同步执行函数`PQexec`发送一个查询给数据库服务器，并等待查询完成，获得执行结果。发送一个查询到服务器，首先要构造一个消息，根据消息定义`TLV`格式，`|消息类型|消息长度|消息内容|`，填写消息类型，消息的长度信息，消息内容，然后通过socket发送给数据库服务器。数据库服务器接收到消息后，解析消息，执行查询，将结果返回给客户端。客户端首先要等待服务端查询结果，注册读事件，通过poll/select等函数，等待服务端查询结果，当监测到读事件后，通过socket读数据到消息缓冲区，然后解析服务端返回的消息，构造`PGresult`结构体，返回。
```c++
// 发送一个查询给数据库服务器，并等待查询完成，获得执行结果
PGresult *PQexec(PGconn *conn, const char *query)
{
	if (!PQexecStart(conn))   		// 发送查询前的准备工作
		return NULL;
	if (!PQsendQuery(conn, query))  // 构造消息，将消息发送给数据库服务端
		return NULL;
	return PQexecFinish(conn);  // 等待查询完成，获得执行结果
}
```
主要流程如下：
```c++
PQexec(PGconn *conn, const char *query)
	// 1. 管道模式校验，同步执行函数不允许管道模式
	// 2. 清理未消费的查询结果
--> PQexecStart(conn)   // 发送查询前的准备工作
--> PQsendQuery(conn, query)  // 构造消息，将消息发送给数据库服务端
	--> PQsendQueryInternal(conn, query, true); // 发送查询，无需等待结果
		--> PQsendQueryStart(conn, newQuery);   // 发送查询前的准备工作
		--> pqAllocCmdQueueEntry(conn);   // 构造查询队列中的命令
			// 开始构造消息，TLV格式，消息类型-消息长度->消息内容
		--> pqPutMsgStart(PqMsg_Query, conn);    // 构造一个消息，将消息类型放人到conn->outBuffer消息缓冲区中
		--> pqPuts(query, conn)  // 将查询语句写入到conn->outBuffer消息缓冲区中
		--> pqPutMsgEnd(conn)  // 将消息长度信息写入到conn->outBuffer消息缓冲区中
			--> pqSendSome(conn, toSend)  // 将消息缓冲区中的消息发送给数据库服务器
				--> pqsecure_write(conn, ptr, len); // 发送消息
					--> pqsecure_raw_write(conn, ptr, len);
						--> send(conn->sock, ptr, len, flags);  // 底层TCP协议发送消息
		--> pqFlush(conn)  // 发送消息缓冲区中的消息
			--> pqSendSome(conn, conn->outCount);
--> PQexecFinish(conn)  // 等待查询完成，获得执行结果
	--> PQgetResult(conn)
		--> pqWait(true, false, conn); // 等待消息
			--> pqWaitTimed(forRead, forWrite, conn, -1);
				--> pqSocketCheck(conn, forRead, forWrite, end_time); 
					--> PQsocketPoll(sock, forRead, forWrite, end_time);
						--> poll(&input_fd, 1, timeout_ms); // 等待读事件
		--> pqReadData(conn)	// poll读事件发生后，进行读操作
			--> pqsecure_read(conn, conn->inBuffer + conn->inEnd, conn->inBufSize - conn->inEnd);
				--> pqsecure_raw_read(conn, ptr, len);
					--> recv(conn->sock, ptr, len, 0); // 底层TCP协议接收消息，读取到conn->inBuffer消息缓冲区中
		--> parseInput(conn);  // 解析输入流
			--> pqParseInput3(conn); // 按协议解析消息类型，根据消息类型执行不同的操作
		--> pqPrepareAsyncResult(conn);  // 将消息返回
```
`PQexecFinish`函数等待查询完成，获得执行结果，源码如下：
```c++
static PGresult *PQexecFinish(PGconn *conn)
{
	PGresult   *result;
	PGresult   *lastResult = NULL;
	while ((result = PQgetResult(conn)) != NULL) // 循环获取结果
	{
		PQclear(lastResult); // 清理上一次查询结果
		lastResult = result;
		if (result->resultStatus == PGRES_COPY_IN ||
			result->resultStatus == PGRES_COPY_OUT ||
			result->resultStatus == PGRES_COPY_BOTH ||
			conn->status == CONNECTION_BAD)
			break;
	}

	return lastResult;  // 返回最后一个查询结果
}
```

#### PGresult
`PGresult`结构封装了由服务器返回的结果。
```c++
typedef struct pg_result PGresult;   // PGresult encapsulates the result of a query

struct pg_result
{
	int			ntups;
	int			numAttributes;
	PGresAttDesc *attDescs;     // 属性（列）信息
	PGresAttValue **tuples;	    // 二维数组，返回结果集的行和列，共计ntups行，numAttributes列
	int			tupArrSize;		/* allocated size of tuples array */
	int			numParameters;
	PGresParamDesc *paramDescs;
	ExecStatusType resultStatus;
	char		cmdStatus[CMDSTATUS_LEN];	/* cmd status from the query */
	int			binary;			/* binary tuple values if binary == 1,
								 * otherwise text */

	/*
	 * These fields are copied from the originating PGconn, so that operations
	 * on the PGresult don't have to reference the PGconn.
	 */
	PGNoticeHooks noticeHooks;
	PGEvent    *events;
	int			nEvents;
	int			client_encoding;	/* encoding id */

	/*
	 * Error information (all NULL if not an error result).  errMsg is the
	 * "overall" error message returned by PQresultErrorMessage.  If we have
	 * per-field info then it is stored in a linked list.
	 */
	char	   *errMsg;			/* error message, or NULL if no error */
	PGMessageField *errFields;	/* message broken into fields */
	char	   *errQuery;		/* text of triggering query, if available */

	/* All NULL attributes in the query result point to this null string */
	char		null_field[1];

	/*
	 * Space management information.  Note that attDescs and error stuff, if
	 * not null, point into allocated blocks.  But tuples points to a
	 * separately malloc'd block, so that we can realloc it.
	 */
	PGresult_data *curBlock;	/* most recently allocated block */
	int			curOffset;		/* start offset of free space in block */
	int			spaceLeft;		/* number of free bytes remaining in block */

	size_t		memorySize;		/* total space allocated for this PGresult */
};
```


### 前后端交互协议
前面的代码忽略了消息解析处理，根据不同的消息，进行不同的处理，收到消息后，如何进行处理呢？这就需要定义前后端交互的协议了。前后端交互是通过TCP协议进行通信的，这就需要定义前端与后端的通信协议，定义数据包是什么类型的消息，才能正常工作。

#### 消息定义
每个消息采用`T-L-V`结构，其中，T表示消息类型，L表示消息长度，V表示消息内容。消息格式为：`| 消息类型(1 byte) | 长度(4 bytes) | 消息内容(可变) |`。其中，有前端发送给服务端的消息类型和数据库服务端发送给客户端前端的消息类型。

#### 消息类型

前端发送给服务端的消息类型：
```c++
#define PqMsg_Bind					'B'   // 绑定参数
#define PqMsg_Close					'C'
#define PqMsg_Describe				'D'
#define PqMsg_Execute				'E'   // 执行查询
#define PqMsg_FunctionCall			'F'
#define PqMsg_Flush					'H'
#define PqMsg_Parse					'P'   
#define PqMsg_Query					'Q'     // 简单查询语句
#define PqMsg_Sync					'S'
#define PqMsg_Terminate				'X'     // 关闭连接
#define PqMsg_CopyFail				'f'
#define PqMsg_GSSResponse			'p'
#define PqMsg_PasswordMessage		'p'
#define PqMsg_SASLInitialResponse	'p'
#define PqMsg_SASLResponse			'p'
```
数据库服务端发送给客户端前端的消息类型：
```c++
#define PqMsg_ParseComplete			'1'
#define PqMsg_BindComplete			'2'
#define PqMsg_CloseComplete			'3'
#define PqMsg_NotificationResponse	'A'
#define PqMsg_CommandComplete		'C'
#define PqMsg_DataRow				'D'   // 返回结果集中的一行数据
#define PqMsg_ErrorResponse			'E'
#define PqMsg_CopyInResponse		'G'
#define PqMsg_CopyOutResponse		'H'
#define PqMsg_EmptyQueryResponse	'I'
#define PqMsg_BackendKeyData		'K'
#define PqMsg_NoticeResponse		'N'
#define PqMsg_AuthenticationRequest 'R'
#define PqMsg_ParameterStatus		'S'
#define PqMsg_RowDescription		'T'   // 描述了返回行每个列元素的类型格式
#define PqMsg_FunctionCallResponse	'V'
#define PqMsg_CopyBothResponse		'W'
#define PqMsg_ReadyForQuery			'Z'    // Ready for query
#define PqMsg_NoData				'n'
#define PqMsg_PortalSuspended		's'
#define PqMsg_ParameterDescription	't'
#define PqMsg_NegotiateProtocolVersion 'v'
```
其他较为特殊的消息类型：
```c++
/* These are the codes sent by both the frontend and backend. */
#define PqMsg_CopyDone				'c'
#define PqMsg_CopyData				'd'

/* These are the codes sent by parallel workers to leader processes. */
#define PqMsg_Progress              'P'
```

#### 消息交互流程

定义了消息类型和消息定义后，就可以定义消息交互流程了。就类似于TCP协议中的三次握手四次分手一样，也需要定义消息交互流程。这里可以参考文档[解析PostgreSQL协议](https://www.modb.pro/doc/58173)。详细的过程这里不再赘述。

PostgreSQL协议交互流程主要包括：
- 认证过程
- 数据请求应答过程
- 错误处理过程

##### 认证过程
认证过程中比较重要的是第一个消息，`Startup`消息，这个消息中包含有客户端与服务端的认证信息，比如用户名、密码、数据库名等。


##### 数据请求应答过程
在认证过程结束后，服务端会向前端发送`ReadyForQuery`消息。`ReadyForQuery`消息是后端已经准备好接收新的前端请求的通知消息。当有新消息时，服务端首先读取消息类型`T`，先读取一个字符，根据不同的消息类型，设置最长的消息长度，对于`Q`类型来说，上限为1G，也就是说PostgreSQL数据库允许用户输入的最大SQL长度为1GB，然后读取4字节的长度信息，获取此条消息的长度，然后根据此长度读取消息内容，最后根据消息类型，执行不同的处理逻辑。遵循`T-L-V`的处理原则。

`CommandComplete`消息已经表示该条SQL请求执行结束了，但是实际上一次查询可能包裹多个SQL，比如`sql1;sql2;`，执行完`sql1`后会通过`EndCommand`函数发送`CommandComplete`消息，只有所有的SQL执行完毕后才会发送`ReadyForQuery`消息，所以PostgreSQL协议要求前端应该侦听到`ReadyForQuery`消息之后才能发出新的查询请求，而不是侦听到`CommandComplete`消息。

然后客户端就可以向服务端发送查询请求，比如`Q`消息，服务端会返回`RowDescription`消息和`DataRow`消息，表示查询结果。


```c++
PostgresMain(int argc, char *argv[], const char *dbname, const char *username)
{
	/* 
	 * 发送K消息，cancel key
	 * 在服务端认证通过后，服务端会返回一个cancel key，用于取消查询
	 * 客户端可以发送这个cancel key到服务端，取消查询
	 */
 	if (whereToSendOutput == DestRemote)
	{
		StringInfoData buf;

		pq_beginmessage(&buf, 'K');
		pq_sendint32(&buf, (int32) MyProcPid);   // 进程ID
		pq_sendint32(&buf, (int32) MyCancelKey); // 随机生成
		pq_endmessage(&buf);
		/* Need not flush since ReadyForQuery will do it. */
	}

	// 不断接收客户端请求
	for (;;)
	{
		// 发送ReadyForQuery消息，表示服务端准备好接收客户端请求了
		if (send_ready_for_query)
		{
			ReadyForQuery(whereToSendOutput); 
			send_ready_for_query = false;
		}

		// 读取一个字符，判断消息类型T
		firstchar = ReadCommand(&input_message);

		switch (firstchar)
		{
			case 'Q':			/* simple query */
				{
					const char *query_string;
					// 获取查询字符串，先读取4个字节，读取消息长度，然后再读取消息内容
					query_string = pq_getmsgstring(&input_message);
					pq_getmsgend(&input_message);

					if (am_walsender)
					{
						if (!exec_replication_command(query_string))
							exec_simple_query(query_string);
					}
					else
						exec_simple_query(query_string);
						{
							// 遍历抽象语法树链表（可能有多个SQL）
							foreach(parsetree_item, parsetree_list)
							{
								// 生成查询树
								pg_analyze_and_rewrite(parsetree, query_string, NULL, 0, NULL);
								// 生成执行计划
								pg_plan_queries(querytree_list, query_string, CURSOR_OPT_PARALLEL_OK, NULL);
								// 执行
								PortalRun(portal, FETCH_ALL, true, true, receiver, receiver, &qc);

								// 结束命令，发送`C`消息
								EndCommand(&qc, dest, false);
							}
						}

					send_ready_for_query = true;
				}
				break;
			// ...
		}

	}

}
 
```


##### 异步消息
异步操作`NoticeResponse`消息，类型为`N`，其内容为一组固定的key-value对，异步操作通常是后端主动推送的，例如当后端的数据库服务器需要重启时，后端可能会通知前端来关闭连接。


### 其他
这里补充一下PostgreSQ中的postmaster进程负责监听端口，当客户端发起连接时，创建一个postgres后台进程负责该客户端的连接，创建进程postgres进程后，由postgres进程来与客户端进行握手，也就是第一个阶段启动阶段，负责与客户端交互通信协议，发送授权信息等，如果一切正常，服务端会反馈状态信息，连接成功创建，通过函数`ReadyForQuery`发送`Z`消息告知客户端一切就绪，进入到常规状态。也就是说是postgres进程负责与客户端的socket发送与接收，并不是postmaster进程。


参考文档：
[libpq](http://www.postgres.cn/docs/current/libpq.html)
[PostgreSQL 通信协议剖析](https://www.modb.pro/db/63988)
[Postgres on the wire A look at the PostgreSQL wire protocol](https://www.pgcon.org/2014/schedule/attachments/330_postgres-for-the-wire.pdf)
