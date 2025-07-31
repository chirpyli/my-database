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

### 示例

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
我们创建一个btree索引，看一下执行结果：
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
在处理这类查询时，bloom索引优于B-Tree索引。可以看到bloom索引的空间占用以及查询性能都优于B-Tree索引（在这种场景下）。一定要注意，不同的索引使用的场景不同，没有绝对好坏之分，只有更适合哪个场景，B-Tree索引是一种更加通用的索引，可以处理范围查询等，而bloom索引是一种针对某种场景特殊设计的索引，只在某些特定的场景下性能更好。
```sql
postgres=# CREATE INDEX bloomidx ON tbloom USING bloom (i1, i2, i3, i4, i5, i6);
CREATE INDEX
Time: 10585.735 ms (00:10.586)
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