## Redis中的持久化机制——RDB篇

我们知道Redis数据库它的全量数据是存储在内存中的，如果关闭Redis，那么内存中的数据就会丢失，为了解决这个问题，Redis提供了两种持久化的方式：RDB和AOF。对于AOF方式，前面我们分析过一篇[Redis中的持久化机制——AOF篇](./redis-aof.md)，本篇我们继续分析另一种方式RDB。

### Redis持久化与PostgreSQL的不同

我们在考虑对Redis进行持久化的时候，必须要考虑的一点就是，持久化应对Redis性能的影响最低，因为Redis核心应用场景之一是缓存，缓存就要求性能高，不然直接访问关系型数据库就好了，没必要加一层缓存了。所以，Redis的持久化机制设计上，一大出发点就是对性能影响小，持久化过程最好不要阻塞客户端命令的执行。相比PostgreSQL等关系型数据库，其持久化设计有所不同，PostgreSQL其是通过WAL日志以及通过一个bgwriter后台进程来周期性的将缓冲池中的数据页刷盘进行持久化。PostgreSQL数据库全量数据存储在磁盘中，其数据查询过程中，如果内存缓冲区ShareBuffer中没有命中，需要将磁盘中的数据页读取到缓冲区ShareBuffer中，同时，当内存缓冲区没有足够的空间时，需要根据淘汰算法将ShareBuffer中的数据页刷盘到磁盘中。也就是说PostgreSQL数据库在运行过程中会面临大量的磁盘IO操作，这几乎无法避免，因为其数据量可能非常大，比内存要大很多。而Redis，其全量数据在内存中，在执行命令的过程中，不需要从磁盘中读取数据。其持久化的目的，在于启动时可以快速恢复数据，从而避免数据丢失（RDB不能完全避免数据丢，这里的避免数据丢失是只RDB快照之前的数据可以避免丢失）。所以，Redis的持久化，就是将其全量内存数据根据数据类型的不同，以不同的编码方式存储到磁盘中，在启动时，将磁盘中的数据解码读取到Redis的不同数据结构中。

### RDB实现原理
我们前面分析过，Redis的持久化最好不要阻塞命令的执行，所以，比较容易想到可以fork一个子进程去做持久化。RDB方式的持久化是通过快照完成的，当符合一定条件时，Redis会自动将内存中的所有数据生成一份副本并存储在硬盘上。会在以下几种情况下对数据进行快照：
- 执行save命令或bgsave命令 （save命令为同步执行快照操作，执行过程中会阻塞所有客户端请求，bgsave命令在后台异步地进行快照操作，可以继续响应客户端的请求）
- 根据配置规则进行自动快照
- 执行flushall命令
- 执行复制时

快照过程：
1. redis使用fork函数复制一份当前进程的副本。
2. 父进程继续接收并处理客户端发来的命令，而子进程开始将内存中的数据写入硬盘中的临时文件。
3. 当子进程写入完所有数据后会用该临时文件替换旧的RDB文件，至此一次快照操作完成。

快照原理：
在执行fork的时候，操作系统会使用写时复制策略（copy-on-write），即fork函数发生的一刻父子进程共享同一内存数据，当父进程要更改其中某片数据时（如执行一个写命令），操作系统会将该片数据复制一份以保证子进程的数据不受影响，所以新的RDB文件存储的是执行fork一刻的内存数据。

Redis启动后会读取RDB快照文件，将数据从硬盘载入到内存。通过RDB方式实现持久化，一旦Redis异常退出，就会丢失最后一次快照以后更改的所有数据。


### 源码分析
我们看一下RDB相关的源码，看一下具体是如何实现写RDB文件以及怎么通过读RDB文件恢复数据。

#### RDB写文件的过程

可通过`BGSAVE`命令或者配置规则触发RDB文件的生成，`BGSAVE`命令和配置规则都会调用`rdbSaveBackground`函数去执行生成RDB文件。
```sh
127.0.0.1:6379> bgsave    # 命令方式
Background saving started
```
调用过程：
```c++
bgsaveCommand(client *c)  // 执行BGSAVE命令
--> rdbSaveBackground(server.rdb_filename,rsiptr)
    --> if ((childpid = redisFork(CHILD_TYPE_RDB)) == 0) // fork子进程
        {
            rdbSave(filename,rsi);
        }
```
也可通过配置规则触发RDB文件的生成，配置规则在`redis.conf`文件中配置，格式如下：
```conf
# save <seconds> <changes>
save 3600 1     # 3600秒内至少有1个key被修改，则进行快照
save 300 100    # 300秒内至少有100个key被修改，则进行快照
save 60 10000    # 60秒内至少有10000个key被修改，则进行快照
```
会通过定时任务去检查是否满足配置规则，满足则调用`rdbSaveBackground`函数去执行生成RDB文件。
调用过程：
```c++
serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData)
--> rdbSaveBackground(server.rdb_filename,rsiptr);  // fork子进程生成RDB文件
    --> rdbSave(filename,rsi);     // 执行写RDB文件
        --> rdbSaveRio(&rdb,&error,RDBFLAGS_NONE,rsi)
            --> rdbSaveKeyValuePair(rdb,&key,o,expire) // 将键值对写入RDB文件
                --> rdbSaveObjectType(rdb,val)  // 写入 type，key, value
                --> rdbSaveStringObject(rdb,key)
                --> rdbSaveObject(rdb,val,key)  // 根据不通的数据类型，处理数据
```
具体的我们看一下`rdbSave`函数的实现。写RDB文件，需要先创建一个临时文件`temp-pid.rdb`，不能直接覆盖写原有的RDB文件，万一此次执行失败，那么就会将原来正常的RDB文件破坏掉，这肯定是不行的，所以需要先创建一个临时文件，然后写入数据，写入完成后，将临时文件替换成原来的RDB文件。
```c++
int rdbSave(char *filename, rdbSaveInfo *rsi)
{
    char tmpfile[256];
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid()); // 创建临时文件
    FILE *fp = fopen(tmpfile,"w");  // 以写入模式打开文件

    rioInitWithFile(&rdb,fp);   // 初始化rio结构，rio是Redis中用于文件操作的封装，可以方便的进行读写操作
    startSaving(RDBFLAGS_NONE);

    // 将内存数据序列化写入RDB文件
    if (rdbSaveRio(&rdb,&error,RDBFLAGS_NONE,rsi) == C_ERR) {
        errno = error;
        goto werr;
    }

    // 强制刷盘
    if (fflush(fp)) goto werr;
    if (fsync(fileno(fp))) goto werr;
    if (fclose(fp)) { fp = NULL; goto werr; }

    rename(tmpfile,filename) == -1)；  // 将临时文件替换成原来的RDB文件
}

int rdbSaveRio(rio *rdb, int *error, int rdbflags, rdbSaveInfo *rsi)
{
    char magic[10];

    snprintf(magic,sizeof(magic),"REDIS%04d",RDB_VERSION);
    if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;     // 写REDIS以及RDB版本号RDB_VERSION，标识次为RDB文件以及RDB版本号

    // 遍历所有数据库，将数据库中的数据写入RDB文件
    for (j = 0; j < server.dbnum; j++) {
        redisDb *db = server.db+j;
        dict *d = db->dict;
        if (dictSize(d) == 0) continue;  // 跳过空数据库
        
        // 获取数据库中键值对的迭代器，实际上是遍历哈希表
        // 如何遍历哈希表，实际上遍历哈希表数组以及为解决冲突而建立的链表
        dictIterator *di = dictGetSafeIterator(d);   

        // 写入SELECT DB操作码及数据库编号， 相当于select db
        if (rdbSaveType(rdb,RDB_OPCODE_SELECTDB) == -1) goto werr;
        if (rdbSaveLen(rdb,j) == -1) goto werr;

        // 写入RESIZE DB操作码及数据库大小， 调整字典中哈希表的大小，这样就不用再进行扩容了
        uint64_t db_size, expires_size;
        db_size = dictSize(db->dict);
        expires_size = dictSize(db->expires);
        if (rdbSaveType(rdb,RDB_OPCODE_RESIZEDB) == -1) goto werr;
        if (rdbSaveLen(rdb,db_size) == -1) goto werr;
        if (rdbSaveLen(rdb,expires_size) == -1) goto werr;        

        // 遍历数据库中的所有键值对，将键值对写入RDB文件
        while((de = dictNext(di)) != NULL) {
            sds keystr = dictGetKey(de);
            robj key, *o = dictGetVal(de);
            long long expire;

            initStaticStringObject(key,keystr);
            expire = getExpire(db,&key); // 获取键的过期时间

            // 写入键值对以及元数据信息（过期时间、LRU/LFU信息）
            // 会根据不同的数据类型处理数据，
            if (rdbSaveKeyValuePair(rdb,&key,o,expire) == -1) goto werr;

            // 混合持久化
            /* When this RDB is produced as part of an AOF rewrite, move
             * accumulated diff from parent to child while rewriting in
             * order to have a smaller final write. */
            if (rdbflags & RDBFLAGS_AOF_PREAMBLE &&
                rdb->processed_bytes > processed+AOF_READ_DIFF_INTERVAL_BYTES)
            {
                processed = rdb->processed_bytes;
                aofReadDiffFromParent();
            }

            /* Update child info every 1 second (approximately).
             * in order to avoid calling mstime() on each iteration, we will
             * check the diff every 1024 keys */
            if ((key_count++ & 1023) == 0) {
                long long now = mstime();
                if (now - info_updated_time >= 1000) {
                    sendChildInfo(CHILD_INFO_TYPE_CURRENT_INFO, key_count, pname);
                    info_updated_time = now;
                }
            }
        }

        dictReleaseIterator(di);
        di = NULL;
    }
        
    // 写入EOF结束标记
    if (rdbSaveType(rdb,RDB_OPCODE_EOF) == -1) goto werr;

    // 计算最终CRC64校验和，并写入
    cksum = rdb->cksum;
    memrev64ifbe(&cksum);
    if (rioWrite(rdb,&cksum,8) == 0) goto werr;
    return C_OK;
}
```

可以看出，RDB文件遵从一定的编码规则，不然读取的时候怎么解析文件内容呢？我们看一下RDB的文件结构：
```sh
+-------+-------------+-----------+-----------------+-----+-----------+
| REDIS | RDB-VERSION | SELECT-DB | KEY-VALUE-PAIRS | EOF | CHECK-SUM |
+-------+-------------+-----------+-----------------+-----+-----------+

                      |<-------- DB-DATA ---------->|
```
编码说明：
- `REDIS`，标识一个RDB文件，以`REDIS`开头。
- `RDB_VERSION`，RDB的版本号，不同版本的RDB文件不兼容，在读RDB文件时需要根据版本来选择不同的读入方式。
- `SELECT-DB`，表示后续`KEY-VALUE_PAIRS`键值对所属的数据库，在读RDB还原数据时，会根据这个值来切换数据库。
- `KEY-VALUE-PAIRS`，表示键值对信息，包含键值对的过期时间、LRU/LFU信息，以及键值数据信息（数据类型、键、值），其中值这部分根据不同的数据类型进行不同的编码处理
- `EOF`标识文件结束
- `CHECK-SUM`表示校验和，用于校验文件是否损坏。

键值对部分编码为`| type | key | value |`的形式。

对不同的数据类型，编码不同，以哈希和集合为例：
对set类型，`OBJ_ENCODING_HT`编码结构为：
```sh
+----------+-----------+-----------+-----+-----------+
| SET-SIZE | ELEMENT-1 | ELEMENT-2 | ... | ELEMENT-N |
+----------+-----------+-----------+-----+-----------+
```
对hash类型，`OBJ_ENCODING_HT`编码结构为：
```sh
+-----------+-------+---------+-------+---------+-----+-------+---------+
| HASH-SIZE | KEY-1 | VALUE-1 | KEY-2 | VALUE-2 | ... | KEY-N | VALUE-N |
+-----------+-------+---------+-------+---------+-----+-------+---------+
```
具体代码如下，因代码较多，这里只列出部分关键代码：
```c++
ssize_t rdbSaveObject(rio *rdb, robj *o, robj *key) {
    ssize_t n = 0, nwritten = 0;

    if (o->type == OBJ_STRING) {    // 字符串类型
        /* Save a string value */
        if ((n = rdbSaveStringObject(rdb,o)) == -1) return -1;
        nwritten += n;
    } else if (o->type == OBJ_LIST) {    // 列表类型
        /* Save a list value */
        if (o->encoding == OBJ_ENCODING_QUICKLIST) {
            // ...
        } 
    } else if (o->type == OBJ_SET) {  // 集合类型
        /* Save a set value */
        if (o->encoding == OBJ_ENCODING_HT) {
            dict *set = o->ptr;
            dictIterator *di = dictGetIterator(set);
            dictEntry *de;

            if ((n = rdbSaveLen(rdb,dictSize(set))) == -1) {
                dictReleaseIterator(di);
                return -1;
            }
            nwritten += n;

            while((de = dictNext(di)) != NULL) {
                sds ele = dictGetKey(de);
                if ((n = rdbSaveRawString(rdb,(unsigned char*)ele,sdslen(ele)))
                    == -1)
                {
                    dictReleaseIterator(di);
                    return -1;
                }
                nwritten += n;
            }
            dictReleaseIterator(di);
        } else if (o->encoding == OBJ_ENCODING_INTSET) {
            size_t l = intsetBlobLen((intset*)o->ptr);

            if ((n = rdbSaveRawString(rdb,o->ptr,l)) == -1) return -1;
            nwritten += n;
        } 
    } else if (o->type == OBJ_ZSET) {  // 有序集合类型
        /* Save a sorted set value */
        if (o->encoding == OBJ_ENCODING_ZIPLIST) {
            size_t l = ziplistBlobLen((unsigned char*)o->ptr);

            if ((n = rdbSaveRawString(rdb,o->ptr,l)) == -1) return -1;
            nwritten += n;
        } else if (o->encoding == OBJ_ENCODING_SKIPLIST) { // 跳表
            // ...
        }
    } else if (o->type == OBJ_HASH) {  // 哈希类型
        /* Save a hash value */
        if (o->encoding == OBJ_ENCODING_ZIPLIST) {  
            size_t l = ziplistBlobLen((unsigned char*)o->ptr);

            if ((n = rdbSaveRawString(rdb,o->ptr,l)) == -1) return -1;
            nwritten += n;

        } else if (o->encoding == OBJ_ENCODING_HT) {  // 哈希表
            dictIterator *di = dictGetIterator(o->ptr);
            dictEntry *de;

            if ((n = rdbSaveLen(rdb,dictSize((dict*)o->ptr))) == -1) {
                dictReleaseIterator(di);
                return -1;
            }
            nwritten += n;

            while((de = dictNext(di)) != NULL) {
                sds field = dictGetKey(de);
                sds value = dictGetVal(de);

                if ((n = rdbSaveRawString(rdb,(unsigned char*)field,
                        sdslen(field))) == -1)
                {
                    dictReleaseIterator(di);
                    return -1;
                }
                nwritten += n;
                if ((n = rdbSaveRawString(rdb,(unsigned char*)value,
                        sdslen(value))) == -1)
                {
                    dictReleaseIterator(di);
                    return -1;
                }
                nwritten += n;
            }
            dictReleaseIterator(di);
        } 
    } else if (o->type == OBJ_STREAM) {
        // ...
    } else if (o->type == OBJ_MODULE) {
        // ...
    } 
    return nwritten;
}
```

#### 通过RDB文件恢复数据的过程

恢复数据的过程，就是写入的逆过程，恢复出原有的数据结构和上下文信息。
```c++
main(int argc, char **argv)
--> initServer();           // 初始化服务器，创建数据库等...
--> loadDataFromDisk();     // 加载（RDB、AOF文件）数据
    {
        if (server.aof_state == AOF_ON)
            loadAppendOnlyFile(server.aof_filename);  // 回放AOF日志文件，恢复数据
        else
            rdbLoad(server.rdb_filename,&rsi,RDBFLAGS_NONE); // 加载RDB文件，恢复数据
    }
--> aeMain(server.el);      // 事件循环，处理客户端请求
```
我们具体看一下`rdbLoad`函数的实现：
```c++
int rdbLoad(char *filename, rdbSaveInfo *rsi, int rdbflags) {
    FILE *fp;
    rio rdb;
    int retval;

    // 打开RDB文件
    if ((fp = fopen(filename,"r")) == NULL) return C_ERR;
    startLoadingFile(fp, filename,rdbflags);
    rioInitWithFile(&rdb,fp);
    retval = rdbLoadRio(&rdb,rdbflags,rsi); // 恢复数据
    fclose(fp);
    stopLoading(retval==C_OK);
    return retval;
}

int rdbLoadRio(rio *rdb, int rdbflags, rdbSaveInfo *rsi) {
    uint64_t dbid;
    int type, rdbver;
    redisDb *db = server.db+0;
    char buf[1024];
    int error;
    long long empty_keys_skipped = 0, expired_keys_skipped = 0, keys_loaded = 0;

    rdb->update_cksum = rdbLoadProgressCallback;
    rdb->max_processing_chunk = server.loading_process_events_interval_bytes;
    // 读取文件头，检查RDB版本号以及REDIS标识符
    if (rioRead(rdb,buf,9) == 0) goto eoferr;
    buf[9] = '\0';
    if (memcmp(buf,"REDIS",5) != 0) {
        serverLog(LL_WARNING,"Wrong signature trying to load DB from file");
        errno = EINVAL;
        return C_ERR;
    }
    rdbver = atoi(buf+5);
    if (rdbver < 1 || rdbver > RDB_VERSION) {
        serverLog(LL_WARNING,"Can't handle RDB format version %d",rdbver);
        errno = EINVAL;
        return C_ERR;
    }

    /* Key-specific attributes, set by opcodes before the key type. */
    long long lru_idle = -1, lfu_freq = -1, expiretime = -1, now = mstime();
    long long lru_clock = LRU_CLOCK();

    while(1) {  // 处理操作码，恢复数据
        sds key;
        robj *val;

        /* Read type. */ // 读取操作码或者对象类型
        if ((type = rdbLoadType(rdb)) == -1) goto eoferr;

        /* Handle special types. */ // 特殊操作码的处理
        if (type == RDB_OPCODE_EXPIRETIME) { // 读过期时间
            /* EXPIRETIME: load an expire associated with the next key
             * to load. Note that after loading an expire we need to
             * load the actual type, and continue. */
            expiretime = rdbLoadTime(rdb);
            expiretime *= 1000;
            if (rioGetReadError(rdb)) goto eoferr;
            continue; /* Read next opcode. */
        } else if (type == RDB_OPCODE_EXPIRETIME_MS) { // 读毫秒精度的过期时间
            /* EXPIRETIME_MS: milliseconds precision expire times introduced
             * with RDB v3. Like EXPIRETIME but no with more precision. */
            expiretime = rdbLoadMillisecondTime(rdb,rdbver);
            if (rioGetReadError(rdb)) goto eoferr;
            continue; /* Read next opcode. */
        } else if (type == RDB_OPCODE_FREQ) { // 读LFU频率
            /* FREQ: LFU frequency. */
            uint8_t byte;
            if (rioRead(rdb,&byte,1) == 0) goto eoferr;
            lfu_freq = byte;
            continue; /* Read next opcode. */
        } else if (type == RDB_OPCODE_IDLE) { // 读LRU空闲时间
            /* IDLE: LRU idle time. */
            uint64_t qword;
            if ((qword = rdbLoadLen(rdb,NULL)) == RDB_LENERR) goto eoferr;
            lru_idle = qword;
            continue; /* Read next opcode. */
        } else if (type == RDB_OPCODE_EOF) { // 文件结束
            /* EOF: End of file, exit the main loop. */
            break;
        } else if (type == RDB_OPCODE_SELECTDB) { // 选择数据库
            /* SELECTDB: Select the specified database. */
            if ((dbid = rdbLoadLen(rdb,NULL)) == RDB_LENERR) goto eoferr;
            if (dbid >= (unsigned)server.dbnum) {
                serverLog(LL_WARNING,
                    "FATAL: Data file was created with a Redis "
                    "server configured to handle more than %d "
                    "databases. Exiting\n", server.dbnum);
                exit(1);
            }
            db = server.db+dbid;  // 选择数据库 相当于执行select db
            continue; /* Read next opcode. */
        } else if (type == RDB_OPCODE_RESIZEDB) { // 调整dict中哈希表的大小
            /* RESIZEDB: Hint about the size of the keys in the currently
             * selected data base, in order to avoid useless rehashing. */
            uint64_t db_size, expires_size;
            if ((db_size = rdbLoadLen(rdb,NULL)) == RDB_LENERR)
                goto eoferr;
            if ((expires_size = rdbLoadLen(rdb,NULL)) == RDB_LENERR)
                goto eoferr;
            dictExpand(db->dict,db_size);
            dictExpand(db->expires,expires_size);
            continue; /* Read next opcode. */
        } else if (type == RDB_OPCODE_AUX) {
            // 辅助信息处理，解析内存用量等AUX信息
            continue; /* Read type again. */
        } else if (type == RDB_OPCODE_MODULE_AUX) {
            // 加载自定义模块的特殊数据
        }

        /* Read key */  // 读取key
        if ((key = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL)) == NULL)
            goto eoferr;
        /* Read value */  // 读取value，恢复不同数据类型的数据结构
        val = rdbLoadObject(type,rdb,key,&error);

        /* Set the expire time if needed */
        if (expiretime != -1) { // 设置过期时间
            setExpire(NULL,db,&keyobj,expiretime);
        }

        /* Set usage information (for eviction). */ // 设置使用信息（用于驱逐）
        objectSetLRUOrLFU(val,lfu_freq,lru_idle,lru_clock,1000);

        // ...
    }

    return C_OK;
}
```
其中恢复不同数据类型的数据结构是通过`rdbLoadObject`函数来实现的。这个函数非常长，这里只截取其中一部分
```c++
robj *rdbLoadObject(int rdbtype, rio *rdb, sds key, int *error) {

    if (rdbtype == RDB_TYPE_STRING) {
        /* Read string value */
        if ((o = rdbLoadEncodedStringObject(rdb)) == NULL) return NULL;
        o = tryObjectEncoding(o);
    } else if (rdbtype == RDB_TYPE_LIST) {
        /* Read list value */
        // 创建一个list对象
        // 插入数据
        // ...
    } else if (rdbtype == RDB_TYPE_SET) {
        /* Read Set value */
        // ...
    } else if (rdbtype == RDB_TYPE_HASH) {

        o = createHashObject(); // 创建一个hash对象

        // 根据元素数量选择ziplist或哈希表        
        /* Load every field and value into the ziplist */
        while (o->encoding == OBJ_ENCODING_ZIPLIST && len > 0) {
            // ...
        }

        /* Load remaining fields and values into the hash table */
        while (o->encoding == OBJ_ENCODING_HT && len > 0) {
            len--;
            /* Load encoded strings */
            if ((field = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL)) == NULL) {
                decrRefCount(o);
                return NULL;
            }
            if ((value = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL)) == NULL) {
                sdsfree(field);
                decrRefCount(o);
                return NULL;
            }

            /* Add pair to hash table */ // 将<field,value>添加到hash表中
            ret = dictAdd((dict*)o->ptr, field, value);
            if (ret == DICT_ERR) {
                rdbReportCorruptRDB("Duplicate hash fields detected");
                sdsfree(value);
                sdsfree(field);
                decrRefCount(o);
                return NULL;
            }
        }
    } else if // 其他类型 ...
}
```
到这里，通过rdb文件恢复内存数据结构已经恢复完毕，接下来就可以响应客户端请求了。

### 总结
Redis提供了两种持久化方式，RDB以及AOF，这两者往往需要结合使用，同时在集群复制场景也会使用RDB以及AOF进行节点数据同步。用户可根据具体场景选择适合的持久化方式。除了以上两种方式，还可以选择混合持久化，也就是综合RDB和AOF的优点，在AOF重写时，先fork子进程将当前redis内存快照写RDB格式数据作为AOF文件上半部分，同时主进程记录重写AOF文件后的所有AOF命令，当RDB格式数据写入AOF文件的上半部分完成时，再将主进程记录的AOF命令写入AOF文件的下半部分，最后替换旧的AOF文件，完成AOF重写。
