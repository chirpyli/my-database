### PostgreSQL插件之pg_freespacemap
PostgreSQL数据库提供了丰富的插件，很多插件可以用来分析数据库的运行情况，pg_freespacemap插件就是其中一个，它可以用来分析数据库中每个表的空闲空间情况，从而帮助DBA了解数据库的运行情况，从而进行优化。


#### pg_freespacemap插件的使用
pg_freespacemap插件的使用非常简单，只需要在数据库中执行以下命令即可：
```sql
-- 安装插件
postgres=# create extension pg_freespacemap;
CREATE EXTENSION
-- 查看当前安装了哪些插件
postgres=# \dx
                                  List of installed extensions
      Name       | Version |   Schema   |                      Description                      
-----------------+---------+------------+-------------------------------------------------------
 pageinspect     | 1.9     | public     | inspect the contents of database pages at a low level
 pg_buffercache  | 1.3     | public     | examine the shared buffer cache
 pg_freespacemap | 1.2     | public     | examine the free space map (FSM)
 plpgsql         | 1.0     | pg_catalog | PL/pgSQL procedural language
(4 rows)
```
通过函数`pg_freespace`获取指定表的空闲空间情况：
```sql
-- 查询表t2的空闲空间情况
postgres=# select * from pg_freespace('t2');
 blkno | avail 
-------+-------
     0 |   288
     1 |     0
     2 |     0
     3 |     0
     4 |     0
     5 |     0
     6 |     0
     7 |     0
     8 |   928
(9 rows)

-- 查询表t2的块号为0的空闲空间大小
postgres=# select * from pg_freespace('t2', 0);
 pg_freespace 
--------------
          288
(1 row)
```
具体是如何获取的呢？只要读取指定表的FSM文件，在PostgreSQL中，为了提高获取表空闲空间的效率，数据库会为每个表创建一个FSM文件，该文件记录了每个块上的空闲空间大小，通过读取该文件，就可以获取指定表的空闲空间情况。例如，对于表t2，我们查到到其relfilenode的OID，数据库会为其创建一个名为`*_fsm`的文件，该文件记录了表t2的空闲空间情况。
```sql
postgres=# select oid,relname,relfilenode from pg_class where relname='t2';
  oid  | relname | relfilenode 
-------+---------+-------------
 16438 | t2      |       16842
(1 row)
```
```shell
postgres@slpc:~/psql/data/base/13010$ ls | grep 16842
16842       # heap堆表文件
16842_fsm   # FSM文件, 记录每个块上的空闲空间大小
16842_vm    # visibility map文件， 加快vacuum的速度
```

#### pg_freespacemap插件实现源码分析

这个插件的代码非常简单，就是一个C函数，该函数的作用是获取指定表在指定块上的空闲空间大小，代码如下：
```sql
-- Register the C function.
CREATE FUNCTION pg_freespace(regclass, bigint)
RETURNS int2
AS 'MODULE_PATHNAME', 'pg_freespace'
LANGUAGE C STRICT PARALLEL SAFE;

-- pg_freespace shows the recorded space avail at each block in a relation
CREATE FUNCTION
  pg_freespace(rel regclass, blkno OUT bigint, avail OUT int2)
RETURNS SETOF RECORD
AS $$
  SELECT blkno, pg_freespace($1, blkno) AS avail
  FROM generate_series(0, pg_relation_size($1) / current_setting('block_size')::bigint - 1) AS blkno;
$$
LANGUAGE SQL PARALLEL SAFE;
```
其中具体实现的代码如下：
```c++
/*
 * Returns the amount of free space on a given page, according to the
 * free space map.
 */
PG_FUNCTION_INFO_V1(pg_freespace);

// 入参：表名，块号
// 返回值：空闲空间大小
Datum pg_freespace(PG_FUNCTION_ARGS)
{
	Oid			relid = PG_GETARG_OID(0);   // 表OID
	int64		blkno = PG_GETARG_INT64(1); // 块号
	int16		freespace;
	Relation	rel;

	rel = relation_open(relid, AccessShareLock);

	if (blkno < 0 || blkno > MaxBlockNumber)
		ereport(ERROR,
				(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
				 errmsg("invalid block number")));

	freespace = GetRecordedFreeSpace(rel, blkno);   // 获取空闲空间大小

	relation_close(rel, AccessShareLock);
	PG_RETURN_INT16(freespace);
}
```
具体是如何获取的呢？下面函数如下：
```c++
Size GetRecordedFreeSpace(Relation rel, BlockNumber heapBlk)
{
	FSMAddress	addr;
	uint16		slot;
	Buffer		buf;
	uint8		cat;

	/* Get the location of the FSM byte representing the heap block */
	addr = fsm_get_location(heapBlk, &slot);    // 获取在FSM文件中的位置，即那个块，FSM文件也是8KB大小的块构成

	buf = fsm_readbuf(rel, addr, false);  // 读FSM文件具体的页到Buffer  
	if (!BufferIsValid(buf))
		return 0;
	cat = fsm_get_avail(BufferGetPage(buf), slot);  // 得到具体的值
	ReleaseBuffer(buf);

    // FSM的值明不是实际大小，而是一个映射中的值，需要转换一下
	return fsm_space_cat_to_avail(cat);
}
```

至于FSM是怎么设计的？后续文章会继续分析


---

参考文档：
[pg_freespacemap](http://postgres.cn/docs/15/pgfreespacemap.html)