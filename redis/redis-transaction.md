## Redis事务

客户端通常会执行一系列命令来对数据对象进行一组相关的更改。然而，另一个客户端也可能在此期间使用类似的命令修改相同的数据对象。这种情况可能导致数据损坏或不一致。为了解决这个问题，redis引入了事务功能。使用事务将来自客户端的多个命令作为一个单元组在一起。事务中的命令保证按顺序执行，不受其他客户端命令的干扰。

这里怎么理解呢？为什么redis中事务的实现要比PostgreSQL中要简单很多？因为redis中关键处理是单线程的，保证只有一个客户端的命令在执行，执行完当前客户端的命令后，才能执行下一个客户端的命令。而PostgreSQL是多进程架构，多个进程并发执行，其事务实现要复杂很多。Redis只实现了有限的事务功能，只保证事务内命令按顺序执行，不被其他客户端命令干扰，事务内命令执行失败则无法进行回滚（PostgreSQL中事务失败则回滚），继续执行下一个命令。对此，需要客户端对事务失败去做相应的处理。


### Redis事务实现

Redis事务相关的五个命令：`MULTI、EXEC、DISCARD、WATCH、UNWATCH`。
|命令|描述|返回值|
|---|---|---|
|MULTI|开启一个事务|OK|
|EXEC|执行事务|事务中所有命令的返回值|
|DISCARD|放弃事务,如果正在使用WATCH命令监视某个key,那么取消所有监视|OK|
|WATCH|监视一个或多个键，如果事务在执行之前key被其他命令改动，那么事务将被打断|OK|
|UNWATCH|取消对所有键的监视|OK|

MULTI命令用于开启一个事务，EXEC命令用于执行事务，DISCARD命令用于放弃事务，WATCH命令用于监视一个或多个键，如果在事务执行之前这些键被其他客户端修改，那么事务将不会执行。

对此我们可以使用以下命令来测试事务功能：
```sh
127.0.0.1:6379> set foo 1
OK
127.0.0.1:6379> set bar 1
OK
127.0.0.1:6379> multi      # 使用MULTI命令进入Redis事务。该命令始终回复 OK
OK
127.0.0.1:6379(TX)> incr foo    # 用户可以发出多个命令。Redis不会立即执行这些命令，而是将它们在服务端排队
127.0.0.1:6379(TX)> incr bar
QUEUED
127.0.0.1:6379(TX)> exec   # 一旦调用 EXEC，所有命令就会执行事务
1) (integer) 2
2) (integer) 2
```
调用 DISCARD 命令将清空事务队列并退出事务
```sh
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> incr foo
QUEUED
127.0.0.1:6379(TX)> incr bar
QUEUED
127.0.0.1:6379(TX)> discard   # 调用DISCARD命令将清空事务队列并退出事务
OK
```
如果在事务执行过程中发生错误，redis将继续执行事务队列中的命令。
```sh
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> set k1 v1
QUEUED
127.0.0.1:6379(TX)> incr k1   # incr命令只能对数字进行操作，如果对字符串进行操作会报错
QUEUED
127.0.0.1:6379(TX)> set k2 1  # 事务中命令执行失败，不影响后续命令执行
QUEUED
127.0.0.1:6379(TX)> get k2
QUEUED
127.0.0.1:6379(TX)> exec
1) OK
2) (error) ERR value is not an integer or out of range
3) OK
4) "1"
```

Redis事务的执行过程如下：

1. 客户端发送`MULTI`命令给Redis服务器，Redis服务端将该客户端client标识为`CLIENT_MULTI`状态，表示客户端处于事务模式，并返回一个`OK`响应给客户端。
2. 客户端发送命令给Redis服务器，Redis服务器将命令通过`queueMultiCommand(c)`放入队列`c->mstate`中，并返回一个`QUEUED`响应给客户端。
3. 客户端发送`EXEC`命令给Redis服务器，Redis服务器执行队列中的命令，并返回命令的执行结果给客户端。


### 源码分析
与客户端建立连接完成后，当客户端向redis发送命令时，即当有数据可读时，会调用`readQueryFromClient`函数，首先是读取客户端发送的命令（字符串），然后根据RESP协议对命令进行解析，最后根据解析后的命令调用相应的命令处理函数。
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
```
对于事务，因事务中包含多个命令，其执行流程与普通命令执行流程有些区别。事务命令，一个事务中往往包含多个命令，所以需要一个数据结构来保存事务中的命令。
```c++
typedef struct multiCmd {
    robj **argv;       // 参数
    int argc;          // 参数数量
    struct redisCommand *cmd;      // 命令指针
} multiCmd;  // 该结构用来保存单个排队的命令
```
事务状态，一般对每个客户端来说，都需要保存所有事务的状态，需要一个事务队列去保存，`commands`数组存储所有排队的命令，`count`记录命令的数量。
```c++
typedef struct multiState {
    // 事务队列，FIFO 顺序
    multiCmd *commands;     /* Array of MULTI commands */
    // 已入队命令计数
    int count;              /* Total number of MULTI commands */
    int minreplicas;        /* MINREPLICAS for synchronous replication */
    time_t minreplicas_timeout; /* MINREPLICAS timeout as unixtime. */
} multiState;
```
`client`是一个非常重要的结构体，它包含了客户端的所有信息，包括当前正在使用的数据库、输入输出缓冲区、命令队列、事务状态等等。
```c++
typedef struct client {
    redisDb *db;            // 当前正在使用的数据库

    struct redisCommand *cmd, *lastcmd;      // 记录被客户端执行的命令

    int reqtype;      // 请求的类型：内联命令还是多条命令
    // 客户端状态标识
    uint64_t flags;         /* Client flags: CLIENT_* macros. */
    // 事务状态
    multiState mstate;      /* MULTI/EXEC state */

    // 被监视的键
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */    
    // ...
} client;
```
介绍完相关的数据结构后，我们看一下函数`processCommand`的实现，接收客户端的命令后，对于事务命令，需要将事务中的命令加入到事务队列中，并返回`QUEUED`。对于事务相关的命令，比如`EXEC、DISCARD、MULTI、WATCH`，不放入事务队列中。
```c++
int processCommand(client *c) 
{
    // ...

    /* Exec the command */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand &&
        c->cmd->proc != resetCommand) 
    {
        // 命令不是EXEC、DISCARD、MULTI、WATCH、RESET，则将命令加入到事务队列中
        queueMultiCommand(c);  
        addReply(c,shared.queued);  // 返回QUEUED
    } else {
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }
}
```
`queueMultiCommand`函数为将一个命令添加到到事务队列中，也就是`client->mstate.commands`中。
```c++
void queueMultiCommand(client *c) {
    multiCmd *mc;
    int j;

    /* No sense to waste memory if the transaction is already aborted.
     * this is useful in case client sends these in a pipeline, or doesn't
     * bother to read previous responses and didn't notice the multi was already
     * aborted. */
    if (c->flags & CLIENT_DIRTY_EXEC)
        return;

    c->mstate.commands = zrealloc(c->mstate.commands,
            sizeof(multiCmd)*(c->mstate.count+1)); // 新分配内存，增加一个multiCmd
    mc = c->mstate.commands+c->mstate.count;  
    mc->cmd = c->cmd;
    mc->argc = c->argc;
    mc->argv = zmalloc(sizeof(robj*)*c->argc);
    memcpy(mc->argv,c->argv,sizeof(robj*)*c->argc);
    for (j = 0; j < c->argc; j++)
        incrRefCount(mc->argv[j]);
    c->mstate.count++;   // 增加命令数量
    c->mstate.cmd_flags |= c->cmd->flags;
    c->mstate.cmd_inv_flags |= ~c->cmd->flags;
}
```
我们知道redis中`multi`命令开启事务，我们看一下`multi`命令的实现，设置该客户端的状态为`CLIENT_MULTI`，表示进入事务模式，返回客户端`OK`。
```c++
void multiCommand(client *c) {
    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }
    c->flags |= CLIENT_MULTI;  // 设置事务flag

    addReply(c,shared.ok);   // 返回客户端OK
}
```
执行一个事务是`exec`命令，我们看一下`exec`命令的实现，首先判断客户端是否处于事务状态，如果处于事务状态，则依次执行事务队列中的命令，并返回结果。如果客户端不处于事务状态，则返回错误。需要注意，事务中禁止执行阻塞命令，
```c++
void execCommand(client *c) {
    int j;
    robj **orig_argv;
    int orig_argc;
    struct redisCommand *orig_cmd;
    int was_master = server.masterhost == NULL;

    // 检查，保证客户端是在事务中
    if (!(c->flags & CLIENT_MULTI)) {
        addReplyError(c,"EXEC without MULTI");
        return;
    }

    /* EXEC with expired watched key is disallowed*/
    if (isWatchedKeyExpired(c)) { // 检查被监视的键是否过期
        c->flags |= (CLIENT_DIRTY_CAS);
    }

    /* Check if we need to abort the EXEC because:
     * 检查是否需要终止事务：
     * 1) Some WATCHed key was touched.
     *    某些被监视的键被修改了
     * 2) There was a previous error while queueing commands.
     *     命令在入队列时发生错误
     * A failed EXEC in the first case returns a multi bulk nil object
     * (technically it is not an error but a special behavior), while
     * in the second an EXECABORT error is returned. 
     *  第一种情况 返回空数组，表示事务因监视的键被修改而失败
     *  第二种情况 返回错误 EXECABORT
     * */
    if (c->flags & (CLIENT_DIRTY_CAS | CLIENT_DIRTY_EXEC)) {
        if (c->flags & CLIENT_DIRTY_EXEC) {
            addReplyErrorObject(c, shared.execaborterr); // 返回错误信息
        } else {
            addReply(c, shared.nullarray[c->resp]); // 返回空数组
        }

        discardTransaction(c);  // 终止事务
        return;
    }

    uint64_t old_flags = c->flags;

    /* we do not want to allow blocking commands inside multi */
    c->flags |= CLIENT_DENY_BLOCKING; // 设置CLIENT_DENY_BLOCKING标志，防止事务中的命令阻塞服务端（如 BLPOP）

    /* Exec all the queued commands */
    unwatchAllKeys(c); /* 取消所有键的监视 */

    server.in_exec = 1;

    orig_argv = c->argv;
    orig_argc = c->argc;
    orig_cmd = c->cmd;
    addReplyArrayLen(c,c->mstate.count);
    for (j = 0; j < c->mstate.count; j++) {   // 遍历事务队列
        c->argc = c->mstate.commands[j].argc;
        c->argv = c->mstate.commands[j].argv;
        c->cmd = c->mstate.commands[j].cmd; 

        /* ACL permissions are also checked at the time of execution in case
         * they were changed after the commands were queued. */
        int acl_errpos;
        int acl_retval = ACLCheckAllPerm(c,&acl_errpos); // 检查ACL权限
        if (acl_retval != ACL_OK) {
            char *reason;
            switch (acl_retval) {
            case ACL_DENIED_CMD:
                reason = "no permission to execute the command or subcommand";
                break;
            case ACL_DENIED_KEY:
                reason = "no permission to touch the specified keys";
                break;
            case ACL_DENIED_CHANNEL:
                reason = "no permission to access one of the channels used "
                         "as arguments";
                break;
            default:
                reason = "no permission";
                break;
            }
            addACLLogEntry(c,acl_retval,acl_errpos,NULL);
            addReplyErrorFormat(c,
                "-NOPERM ACLs rules changed between the moment the "
                "transaction was accumulated and the EXEC call. "
                "This command is no longer allowed for the "
                "following reason: %s", reason);
        } else {
            call(c,server.loading ? CMD_CALL_NONE : CMD_CALL_FULL); // 执行事务队列中的命令
            serverAssert((c->flags & CLIENT_BLOCKED) == 0);
        }

        /* Commands may alter argc/argv, restore mstate. */
        c->mstate.commands[j].argc = c->argc;
        c->mstate.commands[j].argv = c->argv;
        c->mstate.commands[j].cmd = c->cmd;
    }

    // restore old DENY_BLOCKING value
    if (!(old_flags & CLIENT_DENY_BLOCKING))
        c->flags &= ~CLIENT_DENY_BLOCKING; // 恢复允许阻塞的标志

    c->argv = orig_argv;
    c->argc = orig_argc;
    c->cmd = orig_cmd;
    discardTransaction(c);  

    /* Make sure the EXEC command will be propagated as well if MULTI
     * was already propagated. */
    if (server.propagate_in_transaction) {
        int is_master = server.masterhost == NULL;
        server.dirty++;
        /* If inside the MULTI/EXEC block this instance was suddenly
         * switched from master to slave (using the SLAVEOF command), the
         * initial MULTI was propagated into the replication backlog, but the
         * rest was not. We need to make sure to at least terminate the
         * backlog with the final EXEC. */
        if (server.repl_backlog && was_master && !is_master) {
            char *execcmd = "*1\r\n$4\r\nEXEC\r\n";
            feedReplicationBacklog(execcmd,strlen(execcmd));
        }
        afterPropagateExec();
    }

    server.in_exec = 0;
}
```

如果是`DISCARD`命令，则调用调用`discardTransaction`函数，清理事务状态，并返回`OK`。
```c++
void discardCommand(client *c) {
    if (!(c->flags & CLIENT_MULTI)) {
        addReplyError(c,"DISCARD without MULTI");
        return;
    }
    discardTransaction(c);
    addReply(c,shared.ok);
}

void discardTransaction(client *c) {
    freeClientMultiState(c);
    initClientMultiState(c);
    c->flags &= ~(CLIENT_MULTI|CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC);
    unwatchAllKeys(c);  // 取消所有的watch键
}
```

### WATCH机制实现
Redis事务支持乐观锁，通过`WATCH`命令实现，每个客户端维护一个`watched_keys`列表来记录哪些键被watch了。也就是当前客户端关注的哪些键被`WATCH`了。
```c++
typedef struct client {
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
    // ...
} client;
```
另一个是需要知道当一个被`WATCH`的键被修改时，需要通知哪些客户端，Redis通过`watched_keys`字典来实现，键是被`WATCH`的键，值是所有监视该键的客户端列表。
```c++
typedef struct redisDb {
    dict *watched_keys;    /* Keys WATCHED for MULTI/EXEC CAS */
    // ...
} redisDb;
```

当客户端执行`WATCH`命令时，将键通过`watchCommand->watchForKey`加入到`watched_keys`列表中。
```c++
void watchCommand(client *c) {
    int j;

    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"WATCH inside MULTI is not allowed");
        return;
    }
    for (j = 1; j < c->argc; j++)
        watchForKey(c,c->argv[j]);
    addReply(c,shared.ok);
}

/* Watch for the specified key */
void watchForKey(client *c, robj *key) {
    list *clients = NULL;
    listIter li;
    listNode *ln;
    watchedKey *wk;

    // 该键已经watch，则返回
    listRewind(c->watched_keys,&li);
    while((ln = listNext(&li))) {
        wk = listNodeValue(ln);
        if (wk->db == c->db && equalStringObjects(key,wk->key))
            return; /* Key already watched */
    }
    
    // 该键没有被watch，
    // 1. 在c->db数据库的watched_keys字典中建立键到客户端的映射关系
    // 2. 加入到watched_keys列表中
    clients = dictFetchValue(c->db->watched_keys,key);
    if (!clients) {
        clients = listCreate();
        dictAdd(c->db->watched_keys,key,clients);// 键：被监视的键，值：链表，保存所有监视该键的客户端
        incrRefCount(key);
    }
    listAddNodeTail(clients,c);
    /* Add the new key to the list of keys watched by this client */
    wk = zmalloc(sizeof(*wk));
    wk->key = key;
    wk->db = c->db;
    incrRefCount(key);
    listAddNodeTail(c->watched_keys,wk);
}
```

当键被修改时，会调用`signalModifiedKey`函数，进而`touchWatchedKey`函数会被调用，它会标记所有监视该键的客户端为`CLIENT_DIRTY_CAS`状态，这样在`EXEC`时就会检测到冲突并中止事务。
```c++
void touchWatchedKey(redisDb *db, robj *key) {
    list *clients;
    listIter li;
    listNode *ln;

    if (dictSize(db->watched_keys) == 0) return;
    clients = dictFetchValue(db->watched_keys, key);
    if (!clients) return;

    /* Mark all the clients watching this key as CLIENT_DIRTY_CAS */
    /* Check if we are already watching for this key */
    listRewind(clients,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);

        c->flags |= CLIENT_DIRTY_CAS;
    }
}
```
我们回看`execCommand`函数,当检测到`CLIENT_DIRTY_CAS`标识时，会终止事务。
```c++
    /* Check if we need to abort the EXEC because:
     * 检查是否需要终止事务：
     * 1) Some WATCHed key was touched.
     *    某些被监视的键被修改了
     * 2) There was a previous error while queueing commands.
     *     命令在入队列时发生错误
     * A failed EXEC in the first case returns a multi bulk nil object
     * (technically it is not an error but a special behavior), while
     * in the second an EXECABORT error is returned. 
     *  第一种情况 返回空数组，表示事务因监视的键被修改而失败
     *  第二种情况 返回错误 EXECABORT
     * */
    if (c->flags & (CLIENT_DIRTY_CAS | CLIENT_DIRTY_EXEC)) {
        if (c->flags & CLIENT_DIRTY_EXEC) {
            addReplyErrorObject(c, shared.execaborterr); // 返回错误信息
        } else {
            addReply(c, shared.nullarray[c->resp]); // 返回空数组
        }

        discardTransaction(c);  // 终止事务
        return;
    }
```

至此，Redis事务的基本原理就分析完了。

### 总结
Redis的事务实现相对简单而有效，不支持PostgreSQL数据库的回滚机制，在使用时需要注意事务失败时该如何处理。如果Redis需要支持事务回滚，那么会变得很复杂，需要增加undo日志，而日志都是需要持久化存储的，会影响Redis的性能，同时redis的实现会变得很复杂，还有就是redis需要这个功能吗？

参考文档：
[Redis 事务的工作原理](https://redis.ac.cn/docs/latest/develop/interact/transactions/)
[redis源码分析](https://geekdaxue.co/read/yinhuidong@redis/nzn3h4)
