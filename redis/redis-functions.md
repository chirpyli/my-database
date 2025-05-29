## redis学习笔记
以下是在学习redis的过程中记录的一些笔记，侧重介绍redis的功能，以及部分功能的实现原理。redis是开源的，可以分析其代码实现，也有很多书，比如[Redis 设计与实现](https://redisbook.readthedocs.io/en/latest/index.html)可以学习参考。

![redisdatabase.svg](https://oss-emcsprod-public.modb.pro/image/editor/20250513-1922195116979138560_433778.svg)


### redis协议

要实现redis，首先需要实现redis协议，是redis客户端与redis服务端的通信协议，也是实现各种语言的redis客户端的基础。

可以参考[Redis 序列化协议规范](https://redis.ac.cn/docs/latest/develop/reference/protocol-spec/)

### redis数据结构
redis支持多种数据结构，比如字符串、哈希、列表、集合、有序集合、HyperLogLog、流、地理空间等等。其中最常用的是5种基本数据类型：
#### 字符串类型
支持字符串类型，Redis 字符串存储字节序列，包括文本、序列化对象和二进制数组。值可以是任何类型的字符串（包括二进制数据），一个值不能超过 512 MB。

字符串类型相关命令如下：
- SET key value    存储一个字符串值
- GET key           检索一个字符串值
- GETRANGE key start end  返回存储在键处的字符串的子字符串
- GETSET key value     删除键后返回键的字符串值
- GETEX                 设置键的过期时间后返回键的字符串值
- MGET key1 [key2..]    在单个操作中检索多个字符串值。
- SETEX key seconds value  设置键的字符串值和过期时间，如果键不存在则创建它
- SETNX key value     仅在键不存在时存储一个字符串值。
- SETRANGE key offset value      通过偏移量用另一字符串值覆盖字符串值的一部分。如果键不存在，则创建它。
- STRLEN key                       返回字符串值的长度
- MSET key value [key value ...]    原子地创建或修改一个或多个键的字符串值
- MSETNX key value [key value ...]  仅当所有键都不存在时，才原子地修改一个或多个键的字符串值
- PSETEX key milliseconds value    设置键的字符串值和以毫秒为单位的过期时间，如果键不存在则创建它
- INCR key      INCR 命令将字符串值解析为整数，将其加一，然后将获得的值设置为新值。具有原子性，即使多个客户端对同一个键执行 INCR 命令，也永远不会发生竞态条件。
- INCRBY key increment   原子性地递增（传递负数时为递减）给定键存储的计数器
- INCRBYFLOAT key increment   浮点计数器
- DECR key     将键的整数值减一，如果键不存在，则使用0作为初始值
- DECRBY key decrement    将键的整数值中减去一个数字，如果键不存在，则使用0作为初始值
- APPEND key value     将字符串附加到键的值，如果键不存在则创建它

其中比较特殊的位图类型，它不是一个独立的数据类型，而是一组针对字符串类型定义的位操作，涉及如下命令：
- GETBIT key offset         按偏移量返回位值
- SETBIT key offset value   设置或清除字符串值位偏移量的位，如果键不存在则创建它
- BITCOUNT        计算字符串中设置位的数量
- BITOP           对多个字符串执行按位操作，并存储结果
- BITFIELD        对字符串执行任意位字段整数操作
- BITFIELD_RO     对字符串执行任意只读位字段整数操作
- BITPOS          查找字符串中第一个设置或清除位



[字符串命令参考](https://redis.ac.cn/docs/latest/commands/?group=string)

#### 哈希类型
Redis 散列是记录类型，建模为字段-值对的集合，hset命令不区分插入和更新操作，如果不存在，则执行插入操作，如果存在，则执行更新操作。哈希类型，一个键可以包含至多2^32-1个字段。

哈希类型相关命令如下：
- HDEL key field1 [field2]    从哈希中删除一个或多个字段及其值，如果字段为空，则删除哈希
- HEXISTS key field           判断哈希中是否存在某个字段
- HGET key field               获取哈希中某个字段的值
- HGETALL key                  获取哈希中所有字段和值
- HINCRBY key field increment  将哈希中字段的整数值按数字增加，如果字段不存在，则使用0作为初始值
- HINCRBYFLOAT key field increment   将哈希中字段的浮点值按数字增加，如果字段不存在，则使用0作为初始值
- HKEYS key                     返回哈希中的所有字段
- HLEN key                       返回哈希中的字段数量
- HMGET key field1 [field2]      返回哈希中多个字段的值
- HMSET key field1 value1 [field2 value2 ]   设置多个字段的值    （不建议使用，v4.0.0已被废弃）
- HSET key field value            设置多个字段的值
- HSETNX key field value        仅当字段不存在时，才设置字段的值
- HVALS key                      返回哈希中的所有值
- HSCAN key cursor [MATCH pattern] [COUNT count]   迭代哈希的字段和值
- HRANGEFIELD                    从哈希中返回一个或多个随机字段
- HSTRLEN                       返回存储在 key 中的哈希表里与 field 关联的值的字符串长度。如果 key 或 field 不存在，则返回 0。
- HEXPIRE                      使用相对过期时间设置哈希字段的过期时间              v7.4.0
- HEXPIREAT                    使用绝对过期时间设置哈希字段的过期时间              v7.4.0
- HEXPIRETIME                  以秒为单位，将哈希字段的过期时间作为Unix时间戳返回   v7.4.0
- HGETDEL                      返回字段的值并从哈希中删除它                         v8.0.0
- HGETEX                      获取给定哈希键的一个或多个字段的值，并可选地设置其过期时间  v8.0.0
- HPERSIST                    移除每个指定字段的过期时间                           v7.4.0
- HPEXPIRE                    使用相对过期时间设置哈希字段的过期时间               v7.4.0
- HPEXPIREAT                  使用绝对过期时间设置哈希字段的过期时间                v7.4.0
- HPEXPIRETIME                以毫秒为单位，将哈希字段的过期时间作为Unix时间戳返回   v7.4.0
- HPTTL                       返回哈希字段的剩余过期时间                           v7.4.0


[命令参考](https://redis.ac.cn/docs/latest/commands/?group=hash)


#### 列表类型
Redis 列表是按插入顺序排序的字符串列表，列表类型内部是使用双向链表实现的，向列表两端添加元素的时间复杂度为O(1)，

列表类型相关命令如下：
- BLPOP key1 [key2 ] timeout      移除并返回列表中的第一个元素
- BRPOP key1 [key2 ] timeout      移除并返回列表中的最后一个元素
- BRPOPLPUSH source destination timeout    从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它 v6.2.0废弃，不建议使用
- LINDEX key index          按索引返回列表中的元素
- LINSERT key BEFORE|AFTER pivot value 在列表中另一个元素之前或之后插入元素
- LLEN key                    返回列表的长度
- LPOP key                    移除并返回列表的第一个元素
- LPUSH key value1 [value2]      将一个或多个元素添加到列表头部
- LPUSHX key value           仅当列表存在时，才将一个或多个元素前置到列表中
- LRANGE key start stop      返回列表中指定范围内的元素
- LREM key count value       从列表中移除元素
- LSET key index value       将列表中指定索引的元素设置为值
- LTRIM key start stop       修剪列表，使其包含指定范围内的元素
- RPOP key                   移除并返回列表的最后一个元素
- RPOPLPUSH source destination  从列表中移除最后一个元素并推入另一个列表后返回该元素 v6.2.0废弃，不建议使用
- RPUSH key value1 [value2]      将一个或多个元素追加到列表中
- RPUSHX key value           仅当列表存在时，才将一个或多个元素追加到列表中
- BLMOVE            从一个列表弹出元素，将其推入另一个列表并返回，如果无元素则阻塞直到可用，如果移动了最后一个元素，则删除列表
- BLPOP            移除并返回列表中的第一个元素，如果无元素则阻塞直到可用，如果弹出了最后一个元素，则删除列表
- BRPOP            移除并返回列表中的最后一个元素，如果无元素则阻塞直到可用，如果弹出了最后一个元素，则删除列表
- LMOVE            从一个列表弹出元素，将其推入另一个列表并返回，如果移动了最后一个元素，则删除列表
- LMPOP            从列表中移除多个元素后返回它们，如果弹出了最后一个元素，则删除列表。   v7.0.0
- BLMPOP           从多个列表中的一个弹出第一个元素，如果无元素则阻塞直到可用，如果弹出了最后一个元素，则删除列表。 v7.0.0

[列表类型命令参考](https://redis.ac.cn/docs/latest/commands/?group=list)

redis列表最大长度为2^32-1个元素。

#### 集合类型
Redis集合是唯一的、无序的字符串集合，集合类型在redis内部是使用值为空的散列表实现的，集合类型相关命令如下：
- SADD key member1 [member2]     向集合中添加一个或多个成员
- SCARD key                                   获取集合的成员数
- SDIFF key1 [key2]               获取多个集合的差集
- SDIFFSTORE destination key1 [key2]         获取多个集合的差集并保存到目标集合
- SINTER key1 [key2]              获取多个集合的交集
- SINTERSTORE destination key1 [key2]        获取多个集合的交集并保存到目标集合
- SISMEMBER key member             判断成员是否在集合中
- SMEMBERS key                    获取集合中的所有成员
- SMOVE source destination member  将成员从源集合移动到目标集合
- SPOP key        从集合中随机返回一个或多个成员并将其删除
- SRANDMEMBER key [count]        从集合中随机返回一个或多个成员
- SREM key member1 [member2]     从集合中移除一个或多个成员
- SUNION key1 [key2]             获取多个集合的并集
- SUNIONSTORE destination key1 [key2]      获取多个集合的并集并保存到目标集合
- SSCAN key cursor [MATCH pattern] [COUNT count]       迭代集合中的元素


#### 有序集合类型
Redis 有序集合是唯一的字符串集合，通过每个字符串关联的分数来维护顺序。有序集合类型在redis内部是使用散列表和跳跃表实现的，相关命令如下：
- ZADD key score1 member1 [score2 member2]    向有序集合中添加元素
- ZCARD key                                   获取有序集合的成员数
- ZCOUNT key min max                          获取有序集合中指定分数范围内的成员数
- ZINCRBY key increment member                将有序集合中成员的分数按指定增量增加
- ZINTERSTORE destination numkeys key [key ...]   获取多个有序集合的交集并保存到目标有序集合
- ZLEXCOUNT key min max                       获取有序集合中指定字典范围内的成员数
- ZRANGE key start stop [WITHSCORES]          获取有序集合中指定区间的成员
- ZRANGEBYLEX key min max [LIMIT offset count]  获取有序集合中指定字典范围内的成员
- ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT]  获取有序集合中指定分数范围内的成员
- ZRANK key member                            获取有序集合中按升序排列的成员的索引
- ZREM key member [member ...]           从有序集合中移除一个或多个成员，如果移除了所有成员，则删除有序集合
- ZREMRANGEBYLEX key min max                  删除有序集合中给定的字典区间的所有成员
- ZREMRANGEBYRANK key start stop              删除有序集合中给定的排名区间的所有成员
- ZREMRANGEBYSCORE key min max            删除有序集合中给定的分数区间的所有成员
- ZREVRANGE key start stop [WITHSCORES]       获取有序集合中指定区间的成员，按反向顺序返回
- ZREVRANGEBYSCORE key max min [WITHSCORES]    获取有序集合中指定分数范围内的成员，按反向顺序返回
- ZREVRANK key member                        获取有序集合中按降序排列的成员的索引
- ZSCORE key member                          获取有序集合中成员的分数
- ZUNIONSTORE destination numkeys key [key ...]  获取多个有序集合的并集并保存到目标有序集合
- ZSCAN key cursor [MATCH pattern] [COUNT count]      迭代有序集合的元素
- BZPOPMAX 从一个或多个有序集合中移除并返回分数最高的成员。
- BZPOPMIN 从一个或多个有序集合中移除并返回分数最低的成员。
- BZMPOP 从一个或多个有序集合中按分数移除并返回成员，如果无成员则阻塞直到可用。 v7.0.0

#### 其他数据类型
除了基本的5种数据类型，redis还有以下几种数据类型：
- [矢量集合](https://redis.ac.cn/docs/latest/develop/data-types/vector-sets/)：矢量集合是一种专门的数据类型，用于管理高维矢量数据，可在Redis中实现快速高效的矢量相似性搜索。  v8.0.0
- 流： 是一种数据结构，其行为类似于仅附加日志。流有助于按事件发生的顺序记录事件，然后将它们联合起来进行处理
- 位图：并不是一个独立的数据类型，位图允许您对字符串执行位操作
- 位域：位域有效地在字符串值中编码多个计数器。位域提供原子性的`get`、`set`和`increment`操作，并支持不同的溢出策略
- 地理空间： 地理空间索引允许您存储坐标并对其进行搜索
- JSON：提供了JavaScript对象表示法 (JSON) 支持。它允许你在Redis数据库中存储、更新和检索JSON值，类似于任何其他Redis数据类型。
- 概率型数据类型：这些数据类型允许您以一种近似但高效的方式收集和计算统计数据。
    - HyperLogLog：提供了对大型集合基数（即元素数量）的概率估计
    - 布隆过滤器：允许您检查元素是否存在于集合中
    - 布谷鸟过滤器： 允许您检查元素是否存在于集合中
    - t-digest： 估计数据值流中的百分位数
    - Top-K： 估计值流中数据点的排名
    - Count-min sketch： 估计值流中数据点的频率
- 时间序列

#### 相关命令

- LCS 查找最长公共子串     redis-7.0.0


#### 键
在redis中，键是字符串，键可以由字母、数字、特殊字符组成，键的长度不能超过512MB。键可以是任意二进制数据。

键全局唯一，这里指的是在单个库中的键是唯一的，但相同的键可以存在于多个数据库中。

##### 键相关命令
- DEL key    删除键及其关联的值
- DUMP key     序列化给定key，并返回被序列化的值
- EXISTS key     指示数据库中是否存在给定的键
- EXPIRE key seconds    设置过期时间
- EXPIREAT key timestamp   为key设置过期时间
- PEXPIREAT key milliseconds-timestamp   以毫秒为单位设置过期时间
- KEYS pattern   阻塞Redis服务器直到返回所有匹配的键
- MOVE key db   将当前数据库的key移动到给定的数据库db当中
- PERSIST key   移除 key 的过期时间，key 将持久保持。
- PTTL key   以毫秒为单位返回 key 的剩余的过期时间。
- TTL key   以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)。
- RANDOMKEY   从当前数据库中随机返回一个 key
- RENAME key newkey   修改 key 的名称
- RENAMENX key newkey  仅当 newkey 不存在时，将 key 改名为 newkey
- SCAN cursor [MATCH pattern] [COUNT count]   用于增量遍历元素集合
- TYPE key   返回 key 所储存的值的类型

##### 过期键处理
键过期允许为键设置超时时间，也称为TTL，当生存时间到期时，键会自动销毁。redis通过过期字典（expires字典）存储键的过期时间，key指向实际键的指针，value，该键的过期时间。怎么删除过期键呢？redis提供了两种方式来处理过期键：

###### 惰性删除（被动删除）
触发时机：客户端尝试访问一个键时。
执行流程：
1. 检测键是否在expires字典中存在。
2. 如果存在且已过期，直接删除该键。
3. 返回nil或执行后续操作。

###### 定期删除（主动删除）
触发时机：redis周期性执行，可通过hz配置来设置。
执行流程：
1. 随机抽取N个键，检查是否过期
2. 删除所有已过期的键
3. 若过期键比例超过25%，重复上述步骤，直到比例低于25%或达到最大时间限制。

异步删除优化：
使用UNLINK命令，主线程标记键为删除状态，由后台线程实际释放内存，执行实际删除动作，避免删除大键时可能阻塞主线程。


##### 键的自动创建和移除
1. 当我们向聚合数据类型添加元素时，如果目标键不存在，将在添加元素之前创建一个空的聚合数据类型。
2. 当我们从聚合数据类型中移除元素时，如果值变为空，则键会自动销毁。流数据类型是此规则的唯一例外。
3. 对一个空键调用只读命令（例如返回列表长度的 LLEN）或移除元素的写命令，其结果总是与该键持有一个命令期望的空聚合类型相同。

##### 键逐出
redis常用作缓存，以加快对较慢的服务器或数据库的读取访问速度，优化缓存的是持久存储数据的副本（比如client--redis--postgres，数据存储在postgres中，redis作为缓存），因此当缓存内存不足时，逐出它们是安全的（因为数据还可以从postgres中获取）。redis允许指定逐出策略，以便在缓存大小超出设定内存限制时自动逐出键。每当客户端向缓存添加新命令时，redis都会检查内存使用情况，如果超出限制，redis将根据所选的逐出策略逐出键，直到使用的总内存回到限制以下。

配置maxmemory选项来设置最大内存限制，当内存使用超过限制时，redis将根据逐出策略逐出键，逐出策略由maxmemory-policy选项配置。当前提供以下策略：
- noeviction：键不会被逐出，但当您尝试执行缓存新数据的命令时，服务器将返回错误。如果您的数据库使用复制，则此条件仅适用于主数据库。请注意，仅读取现有数据的命令仍正常工作。
- allkeys-lru：逐出最近最少使用（LRU）的键。
- allkeys-lfu：逐出最不常用（LFU）的键。
- allkeys-random：随机逐出键。
- volatile-lru：逐出设置了 expire 字段为 true 的最近最少使用的键。
- volatile-lfu：逐出设置了 expire 字段为 true 的最不常用的键。
- volatile-random：仅当键设置了 expire 字段为 true 时，随机逐出键。
- volatile-ttl：逐出设置了 expire 字段为 true 且剩余生存时间（TTL）值最短的键

[键逐出](https://redis.ac.cn/docs/latest/develop/reference/eviction/)

对于LRU算法，redis采用的是近似LRU算法，使用最近最少使用的近似值而不是精确计算它们，它随机抽取少量键样本，然后逐出自上次访问以来时间最长的键。之所以采用近似LRU算法而不使用真正的LRU实现的原因是它会消耗更多内存。

### redis多数据库支持

redis默认支持16个数据库，客户端与redis建立连接后会自动选择0号数据库，可通过select命令更换数据库，redis不支持自定义数据库的名字，每个数据库都以编号命名。多个数据库之间并不是完全隔离的，比如flushall命令可以清空一个redis实例中所有数据库中的数据，数据库更像是一种命名空间。

flushdb和flushall因为耗时过长，redis中可以进行异步操作，flushall async， 交由后台执行。

### redis持久化
redis支持两种方式的持久化，一种是RDB方式，另一种是AOF方式。前者会根据指定的规则定时将内存中的数据存储在硬盘上，后者是在每次执行命令后将命令本身记录下来。

#### RDB方式

RDB方式的持久化是通过快照完成的，当符合一定条件时，Redis会自动将内存中的所有数据生成一份副本并存储在硬盘上。会在以下几种情况下对数据进行快照：
- 执行save命令或bgsave命令 （save命令为同步执行快照操作，执行过程中会阻塞所有客户端请求，bgsave命令在后台异步地进行快照操作，可以继续响应客户端的请求）
- 根据配置规则进行自动快照
- 执行flushall命令
- 执行复制时


快照过程：
1. redis使用fork函数复制一份当前进程的副本。
2. 父进程继续接收并处理客户端发来的命令，而子进程开始将内存中的数据写入硬盘中的临时文件。
3. 当紫禁城写入完所有数据后会用该临时文件替换旧的RDB文件，至此一次快照操作完成。

快照原理：
在执行fork的时候，操作系统会使用写时复制策略（copy-on-write），即fork函数发生的一刻父子进程共享同一内存数据，当父进程要更改其中某片数据时（如执行一个写命令），操作系统会将该片数据复制一份以保证子进程的数据不受影响，所以新的RDB文件存储的是执行fork一刻的内存数据。

Redis启动后会读取RDB快照文件，将数据从硬盘载入到内存。通过RDB方式实现持久化，一旦Redis异常退出，就会丢失最后一次快照以后更改的所有数据。

#### AOF方式
当使用Redis存储非临时数据时，一般需要打开AOF持久化来降低进程中止导致的数据丢失。AOF可以将redis执行的每一条写命令追加到硬盘文件中，这一过程显然会降低redis性能，根据业务需求来决定是否开启AOF。默认 情况下，AOF是关闭的。

可通过appendonly yes 配置项打开AOF持久化。开启AOF持久化后，每执行一条写命令，redis就会将该命令写入硬盘的AOF文件中。

AOF文件以纯文本的形式记录了redis执行的写命令，AOF文件的内容正是redis客户端向redis发送的原始通信协议的内容。

在启动时redis会逐个执行AOF文件中的命令来将硬盘中的数据载入到内存中，载入的速度相较RDB会慢一些。

### redis事务

客户端通常会执行一系列命令来对数据对象进行一组相关的更改。然而，另一个客户端也可能在此期间使用类似的命令修改相同的数据对象。这种情况可能导致数据损坏或不一致。使用事务将来自客户端的多个命令作为一个单元分组在一起。事务中的命令保证按顺序执行，不受其他客户端命令的干扰。

这里怎么理解呢？为什么redis中事务的实现要比PostgreSQL中要简单很多？因为redis中关键处理是单线程的，保证只有一个客户端的命令在执行，执行完当前客户端的命令后，才能执行下一个客户端的命令。而PostgreSQL是多进程架构，多个进程并发执行，其事务实现要复杂很多，事务的保证要强很多。

另外redis中的事务，不支持回滚，若事务中的某个命令执行失败，后续命令仍会继续执行，无法撤销已执行的操作，对此，需要客户端对事务失败去做相应的处理。

### redis查询引擎
redis查询引擎允许您按内容而非按键检索数据，之前介绍的各种数据类型，都是通过键进行查找。

### 发布/订阅
redis具有发布/订阅功能，允许客户端订阅一个或多个频道，并接收其他客户端发布的消息。

### redis pipelining
Redis Pipelining是一种通过一次性发出多个命令而无需等待每个单独命令的响应来提高性能的技术。
redis是一个tcp服务器，使用C/S模型和请求/响应协议。通常一个请求的过程为：
- 客户端向redis发送请求，并从套接字读取redis的响应。
- redis处理命令并将响应发送回客户端。

而pipelining可以实现即使客户端尚未读取旧响应，也能处理新请求，这样就可以向服务器发送多个命令，而无需等待任何回复，最后一次性读取所有回复。

需要注意的是pipelining不保证原子性，如果需要事务保证，则需启动事务。

管道的本质是客户端通过改变了读写顺序带来的性能提升。比如将`write-read-write-read`改为`write-write-read-read`，减少了客户端和服务器之间的往返次数。

### 客户端缓存
客户端缓存减少了redis客户端和服务器之间的网络流量，通常可以提高性能。启用客户端缓存后，Redis 服务器会记住或跟踪每个客户端连接之前读取过的键集。这包括客户端直接读取数据的情况（例如使用 GET 命令），也包括服务器从存储的数据计算值的情况（例如使用 STRLEN）。当任何客户端向被跟踪的键写入新数据时，服务器会向所有之前访问过该键的客户端发送一个失效消息。此消息会警告客户端其缓存的数据副本不再有效，并且客户端会响应性地驱逐过时数据。下次客户端读取同一键时，它将直接访问数据库并使用更新后的数据刷新其缓存。

使用客户端缓存时，客户端库在从数据库检索数据项时会维护这些数据项的本地缓存。当再次需要相同的项时，客户端可以直接从缓存满足读取请求，而不是访问数据库。

![image](https://redis.ac.cn/docs/latest/images/csc/CSCWithCache.drawio.svg)

建议使用场景：只有一小部分数据被访问的频率远高于其余数据

缓存面临一个问题？数据一致性问题，数据更改时如何更新缓存？

所有缓存系统都必须实现一种方案，以便在主数据库中相应数据发生更改时更新缓存中的数据。Redis 使用一种称为跟踪的方法。


客户端缓存实现：
redis客户端缓存支持称为tracking跟踪，它有两种模式
- 在默认模式下，服务器会记住给定客户端访问了哪些键，并在这些键被修改时发送失效消息。这会占用服务器端的内存，但只会针对客户端可能在内存中缓存的键集合发送失效消息。
- 在广播模式下，服务器不会尝试记住给定客户端访问了哪些键，因此次模式安全不占用服务器端的内存。相反，客户端订阅键前缀，例如object:或user:，并在每次触及与订阅前缀匹配的键时收到通知信息。

默认模式：
1. 客户端如果需要，可以启动跟踪，连接开始时默认未启动跟踪。
2. 启用跟踪后，服务器会记住每个客户端在连接生命周期内请求了哪些键。
3. 当某个客户端修改了键，或者由于关联的过期时间被逐出，或者由于maxmemory策略被逐出时，所有启用了跟踪且可能缓存了该键的客户端都会收到一条失效消息通知。
4. 客户端收到失效消息后，需要移除相应的键，以避免提供过时的数据。

表面上看这很棒，但如果你想象有 1 万个连接的客户端通过长时间连接请求数百万个键，服务器最终会存储太多信息。因此，Redis 采用了两个关键思想来限制服务器端使用的内存量以及处理实现该功能的数据结构的 CPU 开销

- 服务器在一个全局表中记录了可能缓存给定键的客户端列表。此表称为失效表。失效表可以包含最大数量的条目。如果插入一个新键，服务器可能会通过假装该键已修改（即使实际上没有修改），并向客户端发送失效消息来逐出旧条目。这样做可以回收用于该键的内存，即使这会强制持有该键本地副本的客户端将其逐出。
- 在失效表中，我们实际上不需要存储指向客户端结构的指针，这会在客户端断开连接时强制进行垃圾回收过程：相反，我们只存储客户端 ID（每个 Redis 客户端都有一个唯一的数字 ID）。如果客户端断开连接，信息会随着缓存槽位的失效而逐步进行垃圾回收。
- 键名空间是单一的，不按数据库编号划分。因此，如果一个客户端在数据库 2 中缓存了键 foo，而其他客户端更改了数据库 3 中键 foo 的值，失效消息仍然会被发送。这样我们可以忽略数据库编号，从而减少内存使用和实现复杂性。


客户端缓存默认是关闭的，如何开启客户端缓存？在resp3协议中，redis客户端缓存分为两种模式：
- 默认模式：执行命令`CLIENT TRACKING ON` 开启， 关闭：执行命令`CLIENT TRACKING OFF`
- 广播模式：执行命令`CLIENT TRACKING ON BCAST` 开启，可通过命令`CLIENT TRACKING ON BCAST PREFIX prefix1 prefix2`开启指定前缀的广播模式，这样就不会收到所有键值的失效消息了， 而是只收到指定前缀的键值的失效消息。

在resp2协议中，提供了转发模式（redirect），将消息通过pub/sub通知给另外一个客户端。

```sh
# telent B
client id
:368
subscribe _redis_:invalidate  # 订阅 __redis__:invalidate 信道

# telnet A，开启 track 并指定转发给 B
client tracking on bcast redirect 368

# telent B 此时有键值被修改，收到 __redis__:invalidate 信道的消息
message
$20
__redis__:invalidate
*1
$1
a
```

#### 选择加入 (Opt-in) 和选择退出 (Opt-out) 缓存

##### 选择加入 (Opt-in) 
客户端实现可能只想缓存选定的键，并明确告知服务器它们将缓存什么以及不缓存什么。这在缓存新对象时会需要更多带宽，但同时减少了服务器需要记住的数据量以及客户端收到的失效消息量。

为此，必须使用 OPTIN 选项启用跟踪
```sh
CLIENT TRACKING ON REDIRECT 1234 OPTIN
```
在此模式下，默认情况下，读查询中提及的键不应被缓存，相反，当客户端想要缓存某些内容时，必须在实际检索数据的命令之前立即发送一个特殊命令
```sh
CLIENT CACHING YES
+OK
GET foo
"bar"
```

##### 选择退出 (Opt-out)
选择退出缓存允许客户端自动在本地缓存键，而无需显式地为每个键选择加入。这种方法确保所有键默认都被缓存，除非另有指定。选择退出缓存可以通过减少为单个键启用缓存的显式命令的需求来简化客户端缓存的实现。

必须使用 OPTOUT 选项启用跟踪以启用选择退出缓存
```sh
CLIENT TRACKING ON OPTOUT
```
如果要从跟踪和缓存中排除特定键，请使用 CLIENT UNTRACKING 命令
```sh
CLIENT UNTRACKING key
```

#### 广播模式
到目前为止，我们描述了 Redis 实现的第一种客户端缓存模型。还有另一种模式，称为广播模式，它从不同的权衡角度看待问题，不占用服务器端的任何内存，但会向客户端发送更多失效消息。在这种模式下，我们有以下主要行为
- 客户端使用 BCAST 选项启用客户端缓存，并使用 PREFIX 选项指定一个或多个前缀。例如：CLIENT TRACKING on REDIRECT 10 BCAST PREFIX object: PREFIX user:。如果完全没有指定前缀，则前缀被视为空字符串，因此客户端将收到所有被修改的键的失效消息。相反，如果使用一个或多个前缀，则失效消息中只会发送与指定前缀之一匹配的键。
- 服务器不在失效表中存储任何内容。相反，它使用一个不同的前缀表，其中每个前缀都关联到客户端列表。
- 任意两个前缀不能跟踪键空间的重叠部分。例如，前缀 "foo" 和 "foob" 是不允许的，因为它们都会触发键 "foobar" 的失效。然而，仅使用前缀 "foo" 就足够了。
- 每次修改与任何前缀匹配的键时，所有订阅该前缀的客户端都将收到失效消息。
- 服务器消耗的 CPU 与注册的前缀数量成比例。如果你只有少量前缀，很难看到任何差异。前缀数量很多时，CPU 开销会相当大。
- 在这种模式下，服务器可以执行优化，为所有订阅给定前缀的客户端创建单一回复，并将相同的回复发送给所有客户端。这有助于降低 CPU 使用率。

[客户端缓存介绍](https://redis.ac.cn/docs/latest/develop/clients/client-side-caching/)
[客户端缓存参考](https://redis.ac.cn/docs/latest/develop/reference/client-side-caching/)


### 复制
redis提供了复制功能，可以实现当一台数据库中的数据更新后，自动将更新的数据同步到其他数据库上。只需要在从节点配置`--slaveof 主节点IP 主节点端口`即可

当一个从数据库启动后，会向主数据库发送sync命令，同时，主数据库接收到sync命令后，会开始在后台保存快照文件，并将保存快照期间接收到的命令缓存起来，当快照完成后，redis会将快照文件和所有缓存的命令发送给从数据库。从数据库收到后，会载入快照并执行收到的缓存的命令。以上过程为复制初始化，复制初始化结束后，主数据库每当收到写命令时就会将命令同步给从数据库，从而保证主从数据库数据一致。


参考文档：
[Redis 命令参考](http://doc.redisfans.com/)
[读写 Redis RESP3 协议以及Redis 6.0客户端缓存](https://colobu.com/2019/08/09/read-and-write-redis-RESP3-protocol/)