## Redis中的持久化机制——AOF篇

这里谈谈Redis中的持久化机制，对于传统关系型数据库比如PostgreSQL，我们对其持久化机制很熟悉。如果Redis作为缓存使用的话，持久化机制某种程度上是可有可无的，毕竟缓存追求的是快，开了持久化就要可磁盘打交道，对性能肯定有影响。但从另一个角度去看，当Redis异常关闭，没有开启持久化的话，缓存的数据就全部丢失，重建缓存是需要时间的，如果有了持久化机制，那么重建缓存的过程会快很多。所以，持久化机制的使用，具体的还要看业务场景。Redis将是否开启持久化以及采用哪种持久化机制交给了用户去选择。

### 持久化机制

redis支持两种方式的持久化，一种是RDB方式，另一种是AOF方式。前者会根据指定的规则（可以是定时、程序正常退出时也会进行RDB、或者通过命令SAVE|BGSAVE）将内存中的数据存储在硬盘上，后者是在每次执行命令后将命令本身记录下来。AOF属于后者。


### AOF原理
RDB的弊端是一旦redis异常关闭，重启时，会丢失掉自上次RDB快照生成后之后的所有数据。怎么解决呢？这就需要数据库将每一次写操作对对数据库的变化记录下来，这样即使数据库异常关闭，重启时，只要将记录的写操作重新执行一遍，就可以恢复数据。在PostgreSQL中，这一机制是WAL，而在redis中，这一机制是AOF。当执行的命令（写命令）会造成数据库的修改或者状态的变化时，redis会将这个命令写入到一个文件中，这个文件就是AOF文件。与PostgreSQL不同的是，redis记录的是命令。

为什么redis记录的是命令？而PostgreSQL中记录的是对页的修改数据呢？我们得首先想清楚，这其实就是redo机制，其目的是为了通过回放日志来恢复数据库的状态，在redis中，全量数据存放在内存中，其关键数据结构是字典`dict`，其回放命令就可以将数据库恢复到之前的状态。而PostgreSQL不行，PostgreSQL中数据存放在磁盘中，具体组织上是通过数据页来存储数据的。

> 当然，这里引申出另一个问题，redis的主从同步问题。当redis节点挂掉时，重启redis节点会经历一个恢复数据的过程，如果数据量较大时，恢复的过程会耗时一段时间，怎么降低这段时间对业务的影响呢？可以做一主一从配置，当主节点挂掉时，从节点切换位主节点，从而降低了对业务的影响。那么主从节点间怎么保持同步呢？就是用redis的AOF和RDB来实现的。


### 如何开启AOF？
可通过`appendonly yes`配置项打开AOF持久化。开启AOF持久化后，每执行一条写命令，redis就会将该命令写入硬盘的AOF文件中。我们可以执行一些写命令，并查看AOF文件：
```log
postgres@slpc:~/redis/data$ tail -f appendonly.aof 
*2
$6
SELECT   这里只所以有select 0，是因为redis需要将当前命令是在哪个数据库执行的记录下来
$1
0
*3
$3
set
$7
beijing
$8
kingbase
```
可以看到AOF文件以纯文本的形式记录了redis执行的写命令，AOF文件的内容正是redis客户端向redis发送的原始通信协议的内容。

当redis异常关闭时，重启redis，在启动时redis会逐个执行AOF文件中的命令来将硬盘中的数据载入到内存中，从而恢复数据，载入的速度相较RDB会慢一些。以下是redis启动时的日志：
```log
18403:C 22 May 2025 16:33:25.588 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
18403:C 22 May 2025 16:33:25.588 # Redis version=6.2.18, bits=64, commit=ee4d13ab, modified=0, pid=18403, just started
18403:C 22 May 2025 16:33:25.588 # Configuration loaded
18403:M 22 May 2025 16:33:25.590 * monotonic clock: POSIX clock_gettime
18403:M 22 May 2025 16:33:25.591 * Running mode=standalone, port=6379.
18403:M 22 May 2025 16:33:25.591 # Server initialized
18403:M 22 May 2025 16:33:25.592 * DB loaded from append only file: 0.000 seconds  从AOF文件中加载数据
18403:M 22 May 2025 16:33:25.592 * Ready to accept connections  服务器已就绪，可以接受连接请求
```
当数据库关闭时，redis会同步AOF文件，保存RDB文件，以下是redis退出时的日志：
```log
16224:signal-handler (1747901694) Received SIGINT scheduling shutdown...   收到SIGINT信号，退出程序
16224:M 22 May 2025 16:14:54.922 # User requested shutdown...
16224:M 22 May 2025 16:14:54.922 * Calling fsync() on the AOF file.   同步AOF文件，强制刷盘
16224:M 22 May 2025 16:14:54.922 * Saving the final RDB snapshot before exiting.   保存RDB文件
16224:M 22 May 2025 16:14:54.923 * DB saved on disk
16224:M 22 May 2025 16:14:54.923 * Removing the pid file.    
16224:M 22 May 2025 16:14:54.923 # Redis is now ready to exit, bye bye...
```

### AOF文件重写
AOF文件会不断记录写命令，文件会变大，怎么解决或者缓解这个问题呢？重写AOF文件。对于回放命令恢复数据库最终状态来说，其最终状态是所有key的最终值，对于中间key值的状态其并不需要，所以可以只记录key的最终状态，而不记录中间状态。可以对AOF文件进行重写，比如`set mykey 1`,`set mykey 2`，只保留最后一个`set  mykey 2`就可以了，这样大大压缩了AOF文件的大小。


### 源码分析
这里我们从AOF文件的写入和回放进行分析，这两个过程也是AOF最核心的处理过程。

#### AOF文件写入
我们看一下AOF相关的源码实现，先看一下，redis是怎么记录AOF文件的，也就是服务端收到客户端的命令后，先执行命令，确保执行成功后，写入AOF文件，最后向客户端返回执行成功。需要保证AOF文件中记录的是写命令并且是成功执行的命令。

这里就需要判断，命令是否是写命令，以及命令是否执行成功。我们前面分析过一条命令的执行过程，这里我们再回顾一下：
```c++
readQueryFromClient(connection *conn)
--> connRead(c->conn, c->querybuf+qblen, readlen)  // 从套接字读取数据到缓冲区
    --> conn->type->read(conn, buf, buf_len);
--> processInputBuffer(c);    // 解析redis协议，将命令存入客户端的argv数组
    --> processInlineBuffer(c)   // 处理内联命令，并创建参数对象
        --> sdsfreesplitres(argv,argc);
    --> processMultibulkBuffer(c) // 将 c->querybuf 中的协议内容转换成 c->argv 中的参数对象
    --> processCommandAndResetClient(c)        // 执行命令，并返回结果
        --> processCommand(c)
            --> call(c,CMD_CALL_FULL);
                --> c->cmd->proc(c);  // 执行具体的命令
                --> 判断是否为写命令，是否需要写入AOF文件
                --> propagate(c->cmd,c->db->id,c->argv,c->argc,propagate_flags); // 将命令写入AOF文件
                    --> feedAppendOnlyFile(cmd,dbid,argv,argc);
                        --> catAppendOnlyGenericCommand(buf,argc,argv); // 将命令转为RESP协议字符串
                        --> server.aof_buf = sdscatlen(server.aof_buf,buf,sdslen(buf)); // 将命令字符串追加到AOF缓冲区
```
我们以set命令为例，看一下命令的执行过程，当完成向数据库的写入操作后，会调用`server.dirty++`，其他数据类型以及命令也类似，只要引起了数据库状态的变化，都需要调用`server.dirty++`。在判断命令是否需要记录到AOF文件中时，可以依据此进行判断。
```c++
setCommand(client *c)
--> setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL)
    --> genericSetKey(c,c->db,key, val,flags & OBJ_KEEPTTL,1);
        --> dbAdd(db,key,val);
            --> dictAdd(db->dict, copy, val); // 添加一个元素到目标哈希表中
                --> dictAddRaw(d,key,NULL);
                --> dictSetVal(d, entry, val);
    --> server.dirty++;     // 修改数据库 dirty 值    
```
判断完成是写命令后，我们就可以执行写入流程了：
```
+-------------------+      +-------------------+      +-------------------+
|   Client Command  | ---> |   Execute Command | ---> |   Append to AOF   |
|   (e.g., SET key) |      |   (Modify Data)   |      |   Buffer (Memory)  |
+-------------------+      +-------------------+      +-------------------+
                                                           |
                                                           v
+-------------------+      +-------------------+      +-------------------+
|  appendfsync      | <--> |  Flush to Disk    | <--> |   AOF File        |
|  (always/everysec/no) |  |  (fsync)         |      |   (on Disk)       |
+-------------------+      +-------------------+      +-------------------+
                                                           |
                                                           v
                                                     (AOF Rewrite)
                                                           |
                                                           v
                                                   +-------------------+
                                                   |   New AOF File    |
                                                   |   (Compressed)    |
                                                   +-------------------+
```
这里我们继续需要分析一下`call`函数的实现：
```c++
void call(client *c, int flags) 
{
    long long dirty = server.dirty;  // 保存当前数据库在执行RDB之后的修改次数，用于计算命令执行后，数据库的修改次数
    
    c->cmd->proc(c);  // 执行具体的命令

    dirty = server.dirty-dirty;  // 计算命令执行后，数据库的修改次数
    if (dirty < 0) dirty = 0;    // 修改次数不能小于0

    /* Propagate the command into the AOF and replication link */
    if (flags & CMD_CALL_PROPAGATE &&
        (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP)
    {
        int propagate_flags = PROPAGATE_NONE;

        /* Check if the command operated changes in the data set. If so
         * set for replication / AOF propagation. */
        if (dirty) propagate_flags |= (PROPAGATE_AOF|PROPAGATE_REPL); // 如果命令执行后，数据库的修改次数大于0，则设置PROPAGATE_AOF和PROPAGATE_REPL标志位，表示需要将命令写入AOF文件，并同步到从节点

        /* If the client forced AOF / replication of the command, set
         * the flags regardless of the command effects on the data set. */
        if (c->flags & CLIENT_FORCE_REPL) propagate_flags |= PROPAGATE_REPL;
        if (c->flags & CLIENT_FORCE_AOF) propagate_flags |= PROPAGATE_AOF;

        /* Call propagate() only if at least one of AOF / replication
         * propagation is needed. Note that modules commands handle replication
         * in an explicit way, so we never replicate them automatically. */
        if (propagate_flags != PROPAGATE_NONE && !(c->cmd->flags & CMD_MODULE))
            // 注意传入的参数，cmd为当前执行的命令，dbid为当前数据库的id，argv为命令的参数，argc为参数的个数，propagate_flags为传播标志位
            propagate(c->cmd,c->db->id,c->argv,c->argc,propagate_flags); // 将命令写入AOF文件，并同步到从节点
    }
}
```
`propagate`函数会调用`feedAppendOnlyFile`函数实现将命令转换为RESP协议字符串的形式并写入到`server.aof_buf`缓冲区中。因为命令最终要写入到文件中，所以这里其实做的是一个序列化或者编码的操作。

到这里，将命令写入到AOF文件的过程就差刷盘最后一步了，即将`server.aof_buf`缓冲区中的内容写入到具体的磁盘文件中。也就是要与IO打交道了，为了尽量降低写AOF文件对性能的影响，redis并没有选择直接写入磁盘，而是将这个选择权交给了用户去决定何时写入磁盘。目前支持三种方式：
- AOF_FSYNC_NO：由操作系统决定何时刷盘
- AOF_FSYNC_EVERYSEC：每秒刷盘
- AOF_FSYNC_ALWAYS：每次写操作都刷盘
由用户根据自己的业务场景需要来选择。

我们看一下具体的写过程：
```c++
main(int argc, char **argv)
--> initServer();       // 服务端初始化，创建数据库等...
--> aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);  // 事件循环初始化
--> aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL)  // 创建定时任务，处理众多后台任务，比如清理过期key,写AOF日志等。
--> aeMain(server.el);
    while (!eventLoop->stop)
        aeProcessEvents(eventLoop, AE_ALL_EVENTS | AE_CALL_BEFORE_SLEEP | AE_CALL_AFTER_SLEEP);
        --> eventLoop->beforesleep(eventLoop);  // 在进入事件循环之前调用
        --> aeApiPoll(eventLoop, tvp)
        --> eventLoop->aftersleep(eventLoop);  // 在进入事件循环之后调用
```
在每次进入事件循环之前调用函数`beforeSleep`，在函数中会调用`flushAppendOnlyFile`函数，将`server.aof_buf`缓冲区中的内容写入到具体的磁盘文件中。
```c++
void beforeSleep(struct aeEventLoop *eventLoop)
{
    // ... 
    /* Write the AOF buffer on disk */
    if (server.aof_state == AOF_ON)
        flushAppendOnlyFile(0);
    // ...
}
```
同时，在redis中，后台定时任务中，也会调用`flushAppendOnlyFile`函数，将`server.aof_buf`缓冲区中的内容写入到具体的磁盘文件中。
```c++
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData)
{
    if (server.aof_state == AOF_ON && server.aof_flush_postponed_start)
        flushAppendOnlyFile(0);     // 将AOF缓冲区中的内容写入到具体的磁盘文件中
    // ...
}
```
那么`flushAppendOnlyFile`函数做了哪些事情呢？
1. 执行`aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));`，会调用`write`写文件，但此时并不会立即写入磁盘，而是写入到内核缓冲区中。
2. 由用户设置的策略，选择何时`fsync`，将内核缓冲区中的数据写入到磁盘文件中。
```c++
/* Write the append only file buffer on disk.
 *
 * Since we are required to write the AOF before replying to the client,
 * and the only way the client socket can get a write is entering when the
 * the event loop, we accumulate all the AOF writes in a memory
 * buffer and write it on disk using this function just before entering
 * the event loop again.
 * 因为需要在回复客户端之前完成AOF的写入，而client socket只有在进入事件循环时才能写入。
 * 因此，我们将所有AOF写累积到缓冲区中，并在下一次进入事件事件循环之前，将缓冲区写入到文件中。
 */
void flushAppendOnlyFile(int force)
{
    ssize_t nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));

    /* Perform the fsync if needed. */
    if (server.aof_fsync == AOF_FSYNC_ALWAYS)   // 如果是always，则立即进行fsync
        redis_fsync(server.aof_fd)
    else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC && server.unixtime > server.aof_last_fsync)) // 如果是everysec，则每秒进行一次fsync
        aof_background_fsync(server.aof_fd); // Starts a background task that performs fsync() 
    // 如果是no，则交由操作系统进行fsync
}
```
到此，AOF日志的写入过程就结束了。

#### 回放AOF日志的过程

启动Redis时，会先检查AOF文件是否存在，如果存在则加载AOF文件，恢复数据
```c++
main(int argc, char **argv)
--> initServer();           // 初始化服务器，创建数据库等...
--> loadDataFromDisk();     // 加载（RDB、AOF文件）数据
    {
        if (server.aof_state == AOF_ON)
            loadAppendOnlyFile(server.aof_filename);  // 回放AOF日志文件，恢复数据
        else
            rdbLoad(server.rdb_filename,&rsi,RDBFLAGS_NONE);
    }
--> aeMain(server.el);      // 事件循环，处理客户端请求
```
让我们想一下怎么样才能回放AOF日志文件的命令呢？
1. 创建一个不带网络连接的伪客户端：redis的命令需要在client的上下文中执行，因此需要创建一个伪客户端。为什么说是伪客户端呢？因为并不是真实的客户端发起的连接建立的client。
2. 读取AOF文件，并根据内容还原出命令以及命令的参数、个数：文件中存储的是RESP协议的字符串，需要解析RESP协议，还原出命令以及命令的参数、个数。
3. 使用伪客户端执行该命令
4. 继续读取AOF文件，直到AOF文件中的所有命令执行完毕

让我们具体看一下`loadAppendOnlyFile`函数的实现：
```c++
int loadAppendOnlyFile(char *filename) {
    struct client *fakeClient;
    FILE *fp = fopen(filename,"r");  // 打开AOF日志文件
    // 在redis中，命令需要在client上下文中执行，因此创建一个无网络连接的伪客户端，用于模拟执行AOF中的命令
    fakeClient = createAOFClient();
    startLoadingFile(fp, filename, RDBFLAGS_AOF_PREAMBLE);

    // ...

    /* Read the actual AOF file, in REPL format, command by command. */
    while(1) {
        int argc, j;
        unsigned long len;
        robj **argv;
        char buf[128];
        sds argsds;
        struct redisCommand *cmd;

        // ...
        if (fgets(buf,sizeof(buf),fp) == NULL) { // 读取一行命令，fgets按行读取，读取到'\n'为止
            if (feof(fp))
                break;
            else
                goto readerr;
        }
        // 读取命令，正常应该是以'*'开始，比如"set shanghai aaa"的RESP字符串为"*3\r\n$3\r\nset\r\n$8\r\nshanghai\r\n$3\r\naaa\r\n"
        if (buf[0] != '*') goto fmterr;  
        if (buf[1] == '\0') goto readerr;
        argc = atoi(buf+1);  // 读取命令字符串的个数，
        if (argc < 1) goto fmterr;

        /* Load the next command in the AOF as our fake client
         * argv. */
        argv = zmalloc(sizeof(robj*)*argc);
        fakeClient->argc = argc;  // 将AOF中的下一个命令加载到伪客户端的argv中
        fakeClient->argv = argv;

        for (j = 0; j < argc; j++) {  // 读取命令以及命令参数
            /* Parse the argument len. */
            char *readres = fgets(buf,sizeof(buf),fp);  // 解析参数长度，这个长度应该是以$开头，加一个整数
            if (readres == NULL || buf[0] != '$') {
                fakeClient->argc = j; /* Free up to j-1. */
                freeFakeClientArgv(fakeClient);
                if (readres == NULL)
                    goto readerr;
                else
                    goto fmterr;
            }
            len = strtol(buf+1,NULL,10);   // 按十进制转换字符串为整数，得到命令长度

            /* Read it into a string object. */
            argsds = sdsnewlen(SDS_NOINIT,len);
            if (len && fread(argsds,len,1,fp) == 0) {  // 读取命令长度的字符串
                sdsfree(argsds);
                fakeClient->argc = j; /* Free up to j-1. */
                freeFakeClientArgv(fakeClient);
                goto readerr;
            }
            argv[j] = createObject(OBJ_STRING,argsds);  // 创建命令对象

            /* Discard CRLF. */
            if (fread(buf,2,1,fp) == 0) {  // 读取命令的CRLF
                fakeClient->argc = j+1; /* Free up to j. */
                freeFakeClientArgv(fakeClient);
                goto readerr;
            }
        }

        /* Command lookup */
        cmd = lookupCommand(argv[0]->ptr);   // 查看命令是否存在， argv[0]为命令，argv[1]为参数1，argv[2]为参数2，以此类推
        if (!cmd) {
            serverLog(LL_WARNING,
                "Unknown command '%s' reading the append only file",
                (char*)argv[0]->ptr);
            exit(1);
        }

        if (cmd == server.multiCommand) valid_before_multi = valid_up_to;

        /* Run the command in the context of a fake client */
        fakeClient->cmd = fakeClient->lastcmd = cmd;
        if (fakeClient->flags & CLIENT_MULTI &&
            fakeClient->cmd->proc != execCommand) // 如果当前是事务状态，则将命令加入到事务队列中
        {
            queueMultiCommand(fakeClient);
        } else {  // 如果不是事务状态，则执行命令
            cmd->proc(fakeClient); // 执行命令
        }

        // ...
    }

loaded_ok: /* DB loaded, cleanup and return C_OK to the caller. */
    fclose(fp);
    freeFakeClient(fakeClient);

    return C_OK;
}
```
到这里，AOF文件加载完成，Redis服务器开始接受客户端的请求，并处理客户端的请求。

### 总结
Redis的持久化机制与PostgreSQL有所不同，这是两种数据库不同的定位决定的。在考虑Redis持久化机制中的AOF方式的时候，我们需要理解Redis对持久化的需求是什么，或者说要理解用户在业务场景中对持久化的需求是什么，怎么平衡因持久化导致的性能损失。