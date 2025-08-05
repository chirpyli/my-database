## PostgreSQL数据库bloom索引
PostgreSQL数据库中bloom索引是通过扩展bloom来实现的，是PostgreSQL自带的一个扩展。bloom索引是一种基于布隆过滤器的索引，可加速多列等值查询，它通过概率性数据结构布隆过滤器快速判断数据是否存在，以牺牲少量精度换取高效的查询性能。具有如下特性：
- 多列联合索引：支持同时为多个列建立索引。
- 高效过滤：适用于高基数的列，快速排除不匹配的数据。
- 空间优化：比传统B-Tree索引更节省存储空间。
- 概率性结果：可能存在“假阳性”（误判匹配），但是不会漏判。
- 仅限等值查询：不支持范围查询。

> 在数据库中，高基数列（High-Cardinality Column） 指包含大量唯一值或接近唯一值的列，其特点是该列的取值种类多、重复率低

bloom索引的核心是快速判断一个键是否可能存在。可适用的场景如下：
1. 多列条件筛选，适用于需要同时满足多列等值条件的查询。
2. 宽表数据分析，在宽表（数百列）场景中，当查询条件不确定时，bloom索引可快速缩小扫描范围。


### 使用方法

1. 安装
```sql
CREATE EXTENSION bloom;
```

2. 创建索引
创建索引的语法如下：
```sql
CREATE INDEX idx_name ON table_name USING bloom (col1, col2, ...)
WITH (length=..., col1=..., col2=...);
```
bloom索引其WITH子句参数如下：
- length：每个签名（索引项）的长度位数，它会被取整成为最近的16的倍数（默认80，最长为4096，越大误判率越低，占用空间越大）。
- col1, col2, ...：每一个索引列产生的位数。每个参数的名字表示它所控制的索引列（默认2，最大是4095，越多误判率越低，消耗计算资源越多）。

### 使用示例

这里通过一个示例来说明bloom索引的使用以及效果。首先创建表，构造数据。
```sql
postgres=# CREATE TABLE tbloom AS
   SELECT
     (random() * 1000000)::int as i1,
     (random() * 1000000)::int as i2,
     (random() * 1000000)::int as i3,
     (random() * 1000000)::int as i4,
     (random() * 1000000)::int as i5,
     (random() * 1000000)::int as i6
   FROM
  generate_series(1,10000000);
SELECT 10000000
```
现在没有任何索引，在这个大表上的顺序扫描需要很长时间：
```sql
postgres=# EXPLAIN ANALYZE SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;
                     QUERY PLAN                                                 
--------------------------------------------------------------------------
 Seq Scan on tbloom  (cost=0.00..213693.07 rows=1 width=24) (actual time=1648.289..1648.290 rows=0 loops=1)
   Filter: ((i2 = 898732) AND (i5 = 123451))
   Rows Removed by Filter: 10000000
 Planning Time: 0.471 ms
 Execution Time: 1648.901 ms
(5 rows)

Time: 1652.018 ms (00:01.652)
```
我们创建一个btree索引，加上索引后看一下执行结果：
```sql
postgres=# CREATE INDEX btreeidx ON tbloom (i1, i2, i3, i4, i5, i6);
CREATE INDEX
Time: 19911.860 ms (00:19.912)
-- 查看索引大小
postgres=# SELECT pg_size_pretty(pg_relation_size('btreeidx'));
 pg_size_pretty 
----------------
 386 MB
(1 row)
postgres=# EXPLAIN ANALYZE SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;       
                           QUERY PLAN                                               
-------------------------------------------------------------------------
 Index Only Scan using btreeidx on tbloom  (cost=0.43..154382.24 rows=1 width=24) (actual time=876.070..876.071 rows
=0 loops=1)
   Index Cond: ((i2 = 898732) AND (i5 = 123451))
   Heap Fetches: 0
 Planning Time: 0.482 ms
 Execution Time: 876.106 ms
(5 rows)

Time: 878.031 ms
```
可以看到，索引扫描的耗时确实比顺序扫描耗时要少，但是也仅仅是快了一倍，付出的是额外的空间开销以及索引维护的开销。能不能再快一些呢？可以尝试bloom索引。
```sql
postgres=# CREATE INDEX bloomidx ON tbloom USING bloom (i1, i2, i3, i4, i5, i6);
CREATE INDEX
Time: 10585.735 ms (00:10.586)
```
在处理这类查询时，bloom索引优于B-Tree索引。可以看到bloom索引的空间占用以及查询性能都优于B-Tree索引（在这种场景下）。
```sql
-- 获取索引大小，相比B-Tree索引，bloom索引的空间占用少。
postgres=# SELECT pg_size_pretty(pg_relation_size('bloomidx'));
 pg_size_pretty 
----------------
 153 MB
(1 row)

postgres=# EXPLAIN ANALYZE SELECT * FROM tbloom WHERE i2 = 898732 AND i5 = 123451;
                    QUERY PLAN                                                  
---------------------------------------------------------------
 Bitmap Heap Scan on tbloom  (cost=121569.90..121571.02 rows=1 width=24) (actual time=115.898..115.899 rows=0 loops=
1)
   Recheck Cond: ((i2 = 898732) AND (i5 = 123451))
   Rows Removed by Index Recheck: 2407
   Heap Blocks: exact=2369
   ->  Bitmap Index Scan on bloomidx  (cost=0.00..121569.90 rows=1 width=0) (actual time=81.975..81.975 rows=2407 lo
ops=1)
         Index Cond: ((i2 = 898732) AND (i5 = 123451))
 Planning Time: 1.286 ms
 Execution Time: 115.951 ms
(8 rows)

Time: 118.161 ms   -- 性能优于B-Tree索引
```
需要注意的是，不同的索引使用的场景不同，没有绝对好坏之分，只有更适合哪个场景，B-Tree索引是一种更加通用的索引，可以处理范围查询等，而bloom索引是一种针对某种场景特殊设计的索引，只在某些特定的场景下性能更好。

### 使用限制
使用时需要注意，bloom索引是不支持更新的，不太适用于高并发写入场景，更适合分析型场景，可根据实际场景需要重建索引。

1. 不支持更新：高并发写入时需谨慎使用，必要时需重建索引。
2. 误判率可控但存在：需要通过参数调优（length和哈希数量）平衡性能与精度。
3. <font color=blue>仅支持等值查询：不支持范围查询。</font>
4. 无排序功能：不同于B-tree，无法用于ORDER BY或DISTINCT操作。
5. bloom访问方法不支持对NULL值的搜索。
6. bloom访问方法不支持UNIQUE索引。
7. bloom中只包括了用于int4以及text的操作符类。
```sql
postgres=# \d t4
                                Table "public.t4"
 Column |            Type             | Collation | Nullable |      Default      
--------+-----------------------------+-----------+----------+-------------------
 a      | integer                     |           |          | 
 b      | integer                     |           |          | 
 c      | timestamp without time zone |           |          | CURRENT_TIMESTAMP

-- 仅支持int4以及text类型，其他类型不支持
postgres=# create index bloom_idx_t4 on t4 using bloom (a,b,c);
ERROR:  data type timestamp without time zone has no default operator class for access method "bloom"
HINT:  You must specify an operator class for the index or define a default operator class for the data type.
```

### 实现原理
bloom索引的核心原理是Bloom Filter，它是一种概率性数据结构，具体原理可参考文档[Bloom过滤器](https://mp.weixin.qq.com/s/D13E5T5i0nNqyNHcvYGbeA)，这里不再赘述。这里主要讲述bloom索引在数据库中的实现。

PostgreSQL数据库中的bloom索引是二级索引，索引在物理上与它描述的表文件分离。你可以理解为索引也是独立的表文件，只不过它所存储的数据信息是索引信息。可通过索引访问方法访问索引数据。PostgreSQL通过`IndexAmRoutine`抽象了不同索引索引访问方法，这样只有实现了索引访问方法，就可以增加新的索引。可通过`pg_am`系统表查看当前支持的索引。
```sql
postgres=# select * from pg_am;
  oid  | amname |      amhandler       | amtype 
-------+--------+----------------------+--------
     2 | heap   | heap_tableam_handler | t
   403 | btree  | bthandler            | i
   405 | hash   | hashhandler          | i
   783 | gist   | gisthandler          | i
  2742 | gin    | ginhandler           | i
  4000 | spgist | spghandler           | i
  3580 | brin   | brinhandler          | i
 20167 | bloom  | blhandler            | i
(8 rows)
```
>  索引实际上是数据键值到TID的映射

所有索引访问方法都把索引划分成标准大小的页面，这样它们就可以使用常规的存储管理器和缓冲区来访问索引内容。


bloom索引其第0个页面是元页面，后面所有的页面都是包含索引行的常规页面，每个索引行包含一个签名和一个指向表中行的TID。

```c++
// bloom索引的元数据页
typedef struct BloomMetaPageData
{
	uint32		magickNumber;
	uint16		nStart;
	uint16		nEnd;
	BloomOptions opts;
	FreeBlockNumberArray notFullPage;
} BloomMetaPageData;
```

元数据页存储索引的元数据，包括签名长度，每个列的哈希次数等。数据页存储`BloomTuple`。
| 元数据页meta page | 数据页data page | data page | ... |




#### 创建索引

索引的创建，索引可以认为也是一张表，只不过表中存储的是索引数据，索引的物理存储同普通表一样，也是通过堆表存储的。

创建索引的逻辑如下：
1. 更新系统表pg_class，向pg_class系统表中插入索引表的元数据信息
2. 向系统表pg_index中插入索引元数据信息
3. 调用blbuild创建bloom索引
```c++
bloom.so!blbuild(Relation heap, Relation index, IndexInfo * indexInfo) (\home\postgres\works\my-github\postgres\contrib\bloom\blinsert.c:122)
index_build(Relation heapRelation, Relation indexRelation, IndexInfo * indexInfo, _Bool isreindex, _Bool parallel) (\catalog\index.c:3021)
index_create(Relation heapRelation, const char * indexRelationName, Oid indexRelationId, Oid parentIndexRelid, Oid parentConstraintId, Oid relFileNode, IndexInfo * indexInfo, List * indexColNames, Oid accessMethodObjectId, Oid tableSpaceId, Oid * collationObjectId, Oid * classObjectId, int16 * coloptions, Datum reloptions, bits16 flags, bits16 constr_flags, _Bool allow_system_table_mods, _Bool is_internal, Oid * constraintId) (\catalog\index.c:1234)
DefineIndex(Oid relationId, IndexStmt * stmt, Oid indexRelationId, Oid parentIndexId, Oid parentConstraintId, _Bool is_alter_table, _Bool check_rights, _Bool check_not_in_use, _Bool skip_build, _Bool quiet) (\commands\indexcmds.c:1174)
ProcessUtilitySlow(ParseState * pstate, PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (\tcop\utility.c:1535)
standard_ProcessUtility(PlannedStmt * pstmt, const char * queryString, _Bool readOnlyTree, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (\tcop\utility.c:1066)
```
构建bloom索引
```c++
IndexBuildResult *blbuild(Relation heap, Relation index, IndexInfo *indexInfo)
{
	IndexBuildResult *result;
	double		reltuples;
	BloomBuildState buildstate;

  // 检查索引是否已经包含数据
	if (RelationGetNumberOfBlocks(index) != 0)
		elog(ERROR, "index \"%s\" already contains data",
			 RelationGetRelationName(index));

	/* 构造元页面 */
	BloomInitMetapage(index);

	/* 初始化BloomBuildState结构体 */
	memset(&buildstate, 0, sizeof(buildstate));
	initBloomState(&buildstate.blstate, index);
	buildstate.tmpCtx = AllocSetContextCreate(CurrentMemoryContext,
											  "Bloom build temporary context",
											  ALLOCSET_DEFAULT_SIZES);
	initCachedPage(&buildstate);

	/* 扫描堆表（需要被构建索引的表） */
	reltuples = table_index_build_scan(heap, index, indexInfo, true, true,
									   bloomBuildCallback, (void *) &buildstate, NULL);

	/* Flush last page if needed (it will be, unless heap was empty) */
	if (buildstate.count > 0)
		flushCachedPage(index, &buildstate);

	MemoryContextDelete(buildstate.tmpCtx);

	result = (IndexBuildResult *) palloc(sizeof(IndexBuildResult));
	result->heap_tuples = reltuples;
	result->index_tuples = buildstate.indtuples;

	return result;
}
```



#### 索引访问



参考文档：
[PostgreSQL索引(10) – Bloom](https://www.mengqingzhong.com/2020/10/01/postgresql-index-bloom-10/)
[索引访问方法接口定义](http://postgres.cn/docs/current/indexam.html)