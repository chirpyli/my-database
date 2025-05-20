
## redis
redis是一个常用的开源的内存数据库，常被用作缓存，读写性能极快。支持数据持久化，可以将内存中的数据保存在磁盘中。支持如下数据类型：字符串，哈希，列表，集合，有序集合。

![image](https://oss-emcsprod-public.modb.pro/image/editor/20250512-1921771945792647168_433778.svg)

### 编译安装

获取源码，编译，安装到指定目录：
```sh
git clone https://github.com/redis/redis.git
cd redis
make CFLAGS="-O0 -g" MALLOC=jemalloc
make instal PREFIX=/home/postgres/redis   # 安装到指定目录
```
配置redis.conf，还有非常多的配置项，这里只列出最常用的几个：
```ini
port 6379      #监听端口
dir /home/postgres/redis/data   #数据目录
logfile /home/postgres/redis/redis.log  #日志文件
pidfile /home/postgres/redis/redis.pid  #pid文件
maxmemory 2gb    #最大内存
timeout 300      #客户端空闲断开连接超时时间，单位秒
loglevel debug   #日志级别
databases 16     #数据库数量
```
启动redis，使用默认配置：
```sh
postgres@slpc:~/redis$ ./bin/redis-server 
60088:C 06 May 2025 17:09:50.146 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
60088:C 06 May 2025 17:09:50.146 * Redis version=8.0.0, bits=64, commit=e91a340e, modified=0, pid=60088, just started
60088:C 06 May 2025 17:09:50.146 # Warning: no config file specified, using the default config. In order to specify a config file use ./bin/redis-server /path/to/redis.conf
60088:M 06 May 2025 17:09:50.146 * monotonic clock: POSIX clock_gettime
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis Open Source            
  .-`` .-```.  ```\/    _.,_ ''-._      8.0.0 (e91a340e/0) 64 bit
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 60088
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           https://redis.io       
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

60088:M 06 May 2025 17:09:50.148 * Server initialized
60088:M 06 May 2025 17:09:50.148 * Ready to accept connections tcp
```
启动redis，使用自定义配置：
```sh
postgres@slpc:~/redis$ redis-server redis.conf   # 启动redis，使用自定义配置
postgres@slpc:~/redis$ ls
bin  data  redis.conf  redis.log  redis.pid    # redis启动后，生成了redis.log以及redis.pid文件，数据文件在data目录下
postgres@slpc:~/redis$ tail -f redis.log 
60303:C 06 May 2025 17:13:50.450 * Configuration loaded
60303:M 06 May 2025 17:13:50.451 * monotonic clock: POSIX clock_gettime
60303:M 06 May 2025 17:13:50.452 * Running mode=standalone, port=6379.
60303:M 06 May 2025 17:13:50.452 * Server initialized
60303:M 06 May 2025 17:13:50.453 . The AOF directory appendonlydir doesn't exist
60303:M 06 May 2025 17:13:50.453 * Ready to accept connections tcp

```
通过客户端连接redis，验证
```sh
postgres@slpc:~/redis/bin$ redis-cli 
127.0.0.1:6379> ping    # 测试连接
PONG
127.0.0.1:6379> set mykey hangzhou   
OK
127.0.0.1:6379> get mykey
"hangzhou"

```

### redis的基本使用

字符串类型是最基本的类型，一个key对应一个value。
```sh
# 字符串类型，一个键最大能存储512MB
127.0.0.1:6379> set tianjin "shentongdb"
OK
127.0.0.1:6379> get tianjin
"shentongdb"
```
哈希是一个键值对集合
```sh
127.0.0.1:6379> hmset db beijing kingbasedb tianjing shentongdb wuhan damengdb
OK
127.0.0.1:6379> hgetall db  #获取在哈希表中指定 key 的所有字段和值
1) "beijing"
2) "kingbasedb"
3) "tianjing"
4) "shentongdb"
5) "wuhan"
6) "damengdb"

127.0.0.1:6379> hmget db beijing  # 获取所有给定字段的值
1) "kingbasedb"
```
列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。
```sh
127.0.0.1:6379> lpush database dameng   #将一个或多个值插入到列表头部
(integer) 1
127.0.0.1:6379> lpush database kingbase
(integer) 2
127.0.0.1:6379> lrange database 0 10
1) "kingbase"
2) "dameng"
```
集合是String类型的无序集合，不允许重复成员。集合是通过哈希表实现的，所以添加、删除、查找的复杂度都是O(1)。
```sh
127.0.0.1:6379> sadd city tianjing  #将一个元素添加到集合
(integer) 1
127.0.0.1:6379> sadd city beijing
(integer) 1
127.0.0.1:6379> sadd city hangzhou
(integer) 1
127.0.0.1:6379> smembers city  #返回集合中的所有成员
1) "tianjing"
2) "beijing"
3) "hangzhou"
```
Redis 有序集合和集合一样也是 string 类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个 double 类型的分数。redis 正是通过分数来为集合中的成员进行从小到大的排序。有序集合的成员是唯一的,但分数(score)却可以重复。集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。
```sh
127.0.0.1:6379> zadd mycity 1 nanjing  # 向有序集合添加一个或多个成员，或者更新已存在成员的分数
(integer) 1
127.0.0.1:6379> zadd mycity 2 guilin
(integer) 1
127.0.0.1:6379> zadd mycity 5 hangzhou
(integer) 1
127.0.0.1:6379> zadd mycity 3 qingdao
(integer) 1
127.0.0.1:6379> zadd mycity 4 tianjin
(integer) 1
127.0.0.1:6379> zrange mycity 0 10 withscores  #在有序集合中计算指定字典区间内成员数量
 1) "nanjing"
 2) "1"
 3) "guilin"
 4) "2"
 5) "qingdao"
 6) "3"
 7) "tianjin"
 8) "4"
 9) "hangzhou"
10) "5"
```
Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。
```sh
127.0.0.1:6379> pfadd base "beijing" "tianjin" "qingdao" "xian" "nanjing"
(integer) 1
127.0.0.1:6379> pfcount base
(integer) 5
```
Redis 发布订阅 (pub/sub) 是一种消息通信模式：发送者 (pub) 发送消息，订阅者 (sub) 接收消息。
```sh
127.0.0.1:6379> subscribe mychannel   #  订阅消息
1) "subscribe"
2) "mychannel"
3) (integer) 1
1) "message"     # 接收到消息
2) "mychannel"
3) "oscar"

postgres@slpc:~/works/opensource/redis$ redis-cli  # 发布消息
127.0.0.1:6379> publish mychannel "oscar"
(integer) 1
```
Redis 事务可以一次执行多个命令， 并且带有以下三个重要的保证：
- 批量操作在发送 EXEC 命令前被放入队列缓存。
- 收到 EXEC 命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行。
- 在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中。
```sh
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> set db postgres
QUEUED
127.0.0.1:6379(TX)> get db
QUEUED
127.0.0.1:6379(TX)> exec
1) OK
2) "postgres"
```

### 主流程
redis主流程如下，启动后会做一些初始化工作，然后进入事件循环，等待命令请求。
```c++
main(int argc, char **argv)
--> spt_init(argc, argv);       // 初始化进程标题，通过ps命令看到的进程名称
--> zmalloc_set_oom_handler(redisOutOfMemoryHandler); // 设置内存分配失败时的自定义处理函数
--> initServerConfig();  // 设置 Redis 的全局默认配置参数
--> ACLInit();   // ACL（访问控制列表）子系统的初始化
--> moduleInitModulesSystem();  // 初始化模块系统
--> connTypeInitialize();   // 初始化连接类型
--> initServer();       // 服务端初始化
    --> ThreadsManager_init();
    --> createSharedObjects();
    --> adjustOpenFilesLimit();
    --> aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);  // 事件循环初始化
        --> aeApiCreate(eventLoop)
            --> epoll_create(1024);   // 创建 epoll 实例
    --> evictionPoolAlloc();
    --> slowlogInit();
    --> applyWatchdogPeriod();
--> ACLLoadUsersAtStartup();
--> initListeners();  // 初始化网络监听，注册监听事件
    --> connListen(listener)   // 创建监听套接字
    --> createSocketAcceptHandler(listener, connAcceptHandler(listener->ct))
        --> aeCreateFileEvent(server.el, sfd->fd[j], AE_READABLE, accept_handler,sfd)
            --> aeApiAddEvent(eventLoop, fd, mask)
                --> epoll_ctl(state->epfd,op,fd,&ee)
--> InitServerLast();
--> redisSetCpuAffinity(server.server_cpulist);
--> setOOMScoreAdj(-1);
--> aeMain(server.el);
    while (!eventLoop->stop)
        aeProcessEvents(eventLoop, AE_ALL_EVENTS | AE_CALL_BEFORE_SLEEP | AE_CALL_AFTER_SLEEP);
--> aeDeleteEventLoop(server.el);
```
redis服务端会调用`aeApiPoll`函数等待事件，当事件发生时，会调用`aeApiPoll`函数，该函数会调用`epoll_wait`函数，该函数会阻塞当前线程，直到有事件发生。调用栈如下：
```c++
aeApiPoll(aeEventLoop * eventLoop, struct timeval * tvp) (redis\src\ae_epoll.c:93)
aeProcessEvents(int flags, aeEventLoop * eventLoop) (redis\src\ae.c:398)
aeMain(aeEventLoop * eventLoop) (redis\src\ae.c:495)
main(int argc, char ** argv) (redis\src\server.c:7553)
```

## redis与postgres的对比

redis与postgres相比，两者的应用场景和技术实现都有非常大的差异。

在应用场景上，redis核心场景为高速缓存（比如缓存数据库查询结果），而postgres核心场景为事务型应用，OLTP场景，存储结构化数据。虽然redis也可以持久化存储数据，但是其优势场景依然是作为缓存使用。为什么redis非常快呢？因为它简单，没有复杂的查询和事务处理，没有WAL日志，没有MVCC，无需解析器，不用生成执行计划，最重要的一点，他的全量数据是存放在内存中的，不用像postgres，buffer中没有还需要访问磁盘，这当然快了。为什么redis可以把全量数据放在内存中呢？因为它的设计初衷就是作为缓存使用，既然是缓存，丢就丢了，不用考虑持久化的问题，当然redis还是支持持久化的，毕竟宕机重启后，有持久化其重建缓存也快呀，当前考虑的点不单单是这一点。相比postgres，其精简了大多数的处理过程，使得redis的读写速度非常快。

在功能上，redis与postgres支持的数据类型有较大区别，redis支持string、hash、list、set、zset、bitmap、hyperloglog、geo等数据类型，而postgres支持的数据类型更多，如数组、JSON、JSONB、XML、UUID、HSTORE、CLOB、BLOB等。

在持久化上，redis以内存为主，可进行持久化配置（AOF会削弱redis缓存性能），而postgres以磁盘为主，支持ACID事务。

在事务支持上，postgres支持完整的ACID事务，支持隔离级别，而redis支持简单事务，无法回滚，无隔离级别。

在资源消耗上，redis对内存要求较高，全量数据在内存中，内存必须足够，而postgres对内存要求较低，内存作为缓存使用，磁盘作为持久化存储，内存不足时会将数据淘汰到磁盘，从而降低内存压力。

典型应用场景：
用户请求 -> Redis 查询缓存 -> 未命中 -> 查询 PostgreSQL -> 结果写入 Redis

为什么需要在PostgreSQL数据库前面加一层redis缓存呢？如果请求量不大，PostgreSQL数据库性能足够支撑，那么就不需要redis缓存了，当请求量变大，PostgreSQL数据库性能不足以支撑，那么就需要redis缓存了，加了一层缓存，为PostgreSQL削减了请求压力。

redis作为缓存，其QPS必须要比PostgreSQL高，否则redis缓存没有意义。

## redis应用场景

redis有着非常多的应用场景，比如：
- 缓存
- 计数器
- 消息队列
- 延时队列
- 分布式锁
