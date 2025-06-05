## Redis中Hash类型的实现原理

Hash类型是Redis中基本数据类型之一，使用非常广泛，这里我们分析一下Hash类型的实现原理。

### 从宏观看Redis数据结构

在分析哈希数据类型时，我们需要了解一下redis整体的组织。Redis启动会，会调用`initServer`函数初始化全局变量`struct redisServer server;`，其中保存了所有的数据库信息，然后会调用`aeMain(server.el);`进入事件循环，不断接收客户端命令并执行。数据也是保存在`redisServer`中的`redisDB`中。`redisDB`中通过字典`dict`保存所有键值对数据。而字典`dict`是通过哈希表`dictht`实现的。
```c++
redisServer   // 全局变量server，保存了所有的数据库信息
--> redisDB   // 数据库信息，默认16个数据库
    --> dict   // 字典，保存了所有的键值对
        --> dictht   // 哈希表，底层数据结构
```
重要的数据结构定义如下：
```c++
// redisServer保存了所有的数据库信息
struct redisServer {
    redisDb *db;  // 数据库数组，默认16个数据库
    // ...
}
// redis数据库，默认16个数据库
typedef struct redisDb {
    dict *dict;  // 字典，保存了所有的键值对
    // ...
    int id;    // 数据库id
} redisDb;
// 字典，保存了所有的键值对
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
} dict;

typedef struct dictEntry {
    void *key;    // key
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;    // value
    struct dictEntry *next;   // 指向下一个节点
} dictEntry;
```
在哈希表`dictht`中会通过`dictEntry`存储key-value对，key和value都是一个`redisObject`对象，key总是一个字符串类型的对象，value则根据具体数据类型底层数据结构的不同而不同，可以是字符串、哈希表等。

![image](https://geekdaxue.co/uploads/projects/yinhuidong@redis/37f54c1499b71f3c7d920a532798f173.jpeg)

> 需要注意的是，这里分析的是redis-6.2.8版本的源代码，实现细节可能与最新代码有所差异。

可参考[Redis[四]从宏观上看Redis数据结构](https://geekdaxue.co/read/yinhuidong@redis/nzn3h4)

### 字典
字典是redis中用于存储键值对数据的数据结构，字典的实现基于哈希表。可以看到字典结构中定义了`dictht ht[2]`，是通过两个哈希表实现的字典。为什么是两个哈希表呢？这是为了解决字典扩缩容问题，实现渐进式rehash，避免一次性rehash导致性能问题。
```c++
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```
我们看一下`dictRehash`函数，这里实现了迁移n个元素到新的表，迁移的过程就是找到下一个待迁移的元素，迁移到新表中，旧表中的桶置为NULL，更新rehashidx。
```c++
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    unsigned long s0 = d->ht[0].size;
    unsigned long s1 = d->ht[1].size;
    // 检查是否允许rehash
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;
    if (dict_can_resize == DICT_RESIZE_AVOID && 
        ((s1 > s0 && s1 / s0 < dict_force_resize_ratio) ||
         (s1 < s0 && s0 / s1 < dict_force_resize_ratio)))
    {
        return 0;
    }

    while(n-- && d->ht[0].used != 0) { // 执行n步，每步将一个节点从ht[0]迁移到ht[1]
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1; // 跳过空桶
        }
        de = d->ht[0].table[d->rehashidx]; // 获取当前桶的元素
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;   // 将元素插入到新的哈希桶中
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;  // 将旧哈希桶置为空
        d->rehashidx++;   // 下一个要处理的哈希桶索引
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {  // 如果ht[0]已经为空，则rehash完成
        zfree(d->ht[0].table);  // 释放ht[0]
        d->ht[0] = d->ht[1];  // 重置ht[0]
        _dictReset(&d->ht[1]);  // 重置ht[1]
        d->rehashidx = -1;   // rehashidx置为-1，表示rehash完成
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```
迁移过程是渐进式迁移，每次只迁移n个桶，通常会有两个过程会进行迁移，一个是正常的命令请求过程中会迁移1个桶，另一个是定时任务`serverCron`，会定时调用`incrementallyRehash`进而调用`dictRehashMilliseconds`函数进而调用`dictRehash`进行迁移，每次迁移100个桶。

参考文档：[Redis设计与实现](https://redisbook.readthedocs.io/en/latest/internal-datastruct/dict.html)


### Hash类型的实现
在介绍了从宏观看Redis数据结构以及字典后，可以进入正题了，说明一下Hash类型的实现。

Hash类型底层是通过压缩列表ziplist以及字典dict实现的，因为压缩列表比字典更节省内存，所以程序在创建新Hash键时，默认使用压缩列表作为底层实现，当有需要时，程序才会将底层实现从压缩列表转换到字典。
```c++
robj *createHashObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;   // 底层实现为压缩列表
    return o;
}
```
那么什么时候进行转换呢？
- 元素个数超过一定阈值`server.hash_max_ziplist_entries`时，会进行转换。
- 某个键或者某个值的长度超过一定阈值`server.hash_max_ziplist_value`时，也会进行转换。

```c++
void hashTypeTryConversion(robj *o, robj **argv, int start, int end) {
    int i;
    size_t sum = 0;

    if (o->encoding != OBJ_ENCODING_ZIPLIST) return;

    for (i = start; i <= end; i++) {
        if (!sdsEncodedObject(argv[i]))
            continue;
        size_t len = sdslen(argv[i]->ptr);
        if (len > server.hash_max_ziplist_value) { // 当值长度大于这个值时，则进行转换
            // 将压缩列表转为哈希表
            // 转换的具体过程就是创建哈希表，遍历压缩列表，获取key,value，然后插入哈希表
            hashTypeConvert(o, OBJ_ENCODING_HT); 
            return;
        }
        sum += len;
    }
    if (!ziplistSafeToAdd(o->ptr, sum))
        hashTypeConvert(o, OBJ_ENCODING_HT);
}
```

我们看一下`hset`命令的实现，也就是向哈希类型的数据结构中添加数据：
```c++
void hsetCommand(client *c) {
    int i, created = 0;
    robj *o;

    if ((c->argc % 2) == 1) {  // 参数个数必须是偶数，放在field和value不成对
        addReplyErrorFormat(c,"wrong number of arguments for '%s' command",c->cmd->name);
        return;
    }

    // 检查键空间中是否存在该键，如果不存在，则创建一个新的hash对象
    if ((o = hashTypeLookupWriteOrCreate(c,c->argv[1])) == NULL) return;
    // 尝试转换类型，检查key,value的长度，如果超过一定大小，需要将ziplist转换为哈希表
    hashTypeTryConversion(o,c->argv,2,c->argc-1); 

    // 遍历所有field和value，插入到底层数据结构中
    // 具体插入时需要判断是ziplist还是hash表
    for (i = 2; i < c->argc; i += 2)
        created += !hashTypeSet(o,c->argv[i]->ptr,c->argv[i+1]->ptr,HASH_SET_COPY);

    /* HMSET (deprecated) and HSET return value is different. */
    char *cmdname = c->argv[0]->ptr;
    if (cmdname[1] == 's' || cmdname[1] == 'S') {
        /* HSET */
        addReplyLongLong(c, created);
    } else {
        /* HMSET */
        addReply(c, shared.ok);
    }
    signalModifiedKey(c,c->db,c->argv[1]);  // 通知数据库该键被修改
    // 发布键空间通知
    notifyKeyspaceEvent(NOTIFY_HASH,"hset",c->argv[1],c->db->id);
    server.dirty += (c->argc - 2)/2;   // 增加数据库的脏数据计数器
}
```
我们具体看一下压缩列表数据结构，压缩列表的布局：`<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>`
|字段	|类型|	描述|
|:----|:----|:----|
|zlbytes	|uint32_t	|总字节数（包含自身4字节）|
|zltail|	uint32_t|	最后一个entry的偏移量（支持快速弹出操作）|
|zllen|	uint16_t	|entry数量（超过2^16-2时设为2^16-1，需遍历统计）|
|zlend|	uint8_t	|结束标记(0xFF)|

其中每个`entry`的布局为：`<prevlen> <encoding> <entry-data>`
- `prevlen`：前一个`entry`的长度（动态编码）
- `encoding`：数据类型和编码方式
- `entry-data`： 实际数据

这里采用动态编码是为了节省存储空间，与protobuf编码中的varint原理类似。

具体的对应哈希类型的field和value在`entry-data`中存储，相邻节点交替存储`field`和`value`，比如`[field1,value1,field2,value2,field3,value3]`，每个`field`或者`value`都是一个`entry`，依次排列。

核心操作：
- `hashTypeSet`:通过`ziplistPush`追加新元素
- `hashTypeGetFromZiplist`：使用`ziplistFind`函数遍历查找
- 自动转换：当插入新元素导致长度/值超过阈值时，会调用`hashTypeConvert`进行转换

压缩列表比较适合数量量小的场景，每当插入一个新元素时，如果ziplist的容量不够，则会重新分配内存空间，将原有数据拷贝到新空间中，并插入新元素。当数据量较大时，每次分配内存空间和拷贝数据都会产生较大的额外开销，因此，当数据量较大时，会考虑将压缩列表转换为哈希表存储。实际上就是前面介绍的字典`dict`，这里不再进行介绍。

