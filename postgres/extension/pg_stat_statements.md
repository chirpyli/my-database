## PostgreSQL源码分析——pg_stat_statements
pg_stat_statements插件用于跟踪所有SQL语句的执行情况，包括执行时间、执行次数、返回行数、共享内存使用情况等。通过该插件我们可以一定程度上监控数据库中SQL的执行情况，从而可以进行优化，例如找出哪些SQL语句执行效率低，针对性的进行分析优化等。


### pg_stat_statements的使用

#### 安装插件
```sql
postgres=# create extension pg_stat_statements ;
CREATE EXTENSION
postgres=# \dx
                                            List of installed extensions
        Name        | Version |   Schema   |                              De
scription                               
--------------------+---------+------------+--------------------------------
----------------------------------------
 pg_buffercache     | 1.3     | public     | examine the shared buffer cache
 pg_stat_statements | 1.9     | public     | track planning and execution st
atistics of all SQL statements executed
 plpgsql            | 1.0     | pg_catalog | PL/pgSQL procedural language
(3 rows)
```

#### 配置参数

- pg_stat_statements.max  跟踪的语句的最大数目。 如果超过此限制，则删除执行次数最少的语句。 
- pg_stat_statements.track  控制跟踪哪些语句。指定top可以跟踪顶层语句（那些直接由客户端发出的语句），指定all还可以跟踪嵌套的语句（例如在函数中调用的语句），指定none可以禁用语句统计信息收集。默认值是top
- pg_stat_statements.track_utility  是否跟踪utility命令，默认为on。
- pg_stat_statements.save  是否将pg_stat_statements信息保存到磁盘，这样数据库重启后信息依旧可见
- pg_stat_statements.track_planning 控制模块是否跟踪计划操作和持续时间


```sql
postgres=# alter system set pg_stat_statements.max = 1000;
ALTER SYSTEM
postgres=# show pg_stat_statements.track;
 pg_stat_statements.track 
--------------------------
 top
(1 row)

postgres=# show pg_stat_statements.track_utility ;
 pg_stat_statements.track_utility 
----------------------------------
 on
(1 row)

postgres=# show pg_stat_statements.save ;
 pg_stat_statements.save 
-------------------------
 on
(1 row)

postgres=# alter system set pg_stat_statements.save to off;
ALTER SYSTEM
```
更多使用方法请参考文档[pg_stat_statements](http://postgres.cn/docs/15/pgstatstatements.html)


#### pg_stat_statements视图

插件的使用非常简单，通过函数pg_stat_statements_reset可以删除目前pg_stat_statements中已收集的信息。如果指定了userid,dbid,queryid参数，则只删除对应的信息。

查询视图pg_stat_statements，返回目前统计的SQL的信息。我们看一下视图的定义：注意这里是pg14版本的，不同版本可能有些许不同，但大体一致。
```sql
postgres=# \d pg_stat_statements
                    View "public.pg_stat_statements"
       Column        |       Type       | Collation | Nullable | Default 
---------------------+------------------+-----------+----------+---------
 userid              | oid              |           |          | 
 dbid                | oid              |           |          | 
 toplevel            | boolean          |           |          | 
 queryid             | bigint           |           |          | 
 query               | text             |           |          | 
 plans               | bigint           |           |          | 
 total_plan_time     | double precision |           |          | 
 min_plan_time       | double precision |           |          | 
 max_plan_time       | double precision |           |          | 
 mean_plan_time      | double precision |           |          | 
 stddev_plan_time    | double precision |           |          | 
 calls               | bigint           |           |          | 
 total_exec_time     | double precision |           |          | 
 min_exec_time       | double precision |           |          | 
 max_exec_time       | double precision |           |          | 
 mean_exec_time      | double precision |           |          | 
 stddev_exec_time    | double precision |           |          | 
 rows                | bigint           |           |          | 
 shared_blks_hit     | bigint           |           |          | 
 shared_blks_read    | bigint           |           |          | 
 shared_blks_dirtied | bigint           |           |          | 
 shared_blks_written | bigint           |           |          | 
 local_blks_hit      | bigint           |           |          | 
 local_blks_read     | bigint           |           |          | 
 local_blks_dirtied  | bigint           |           |          | 
 local_blks_written  | bigint           |           |          | 
 temp_blks_read      | bigint           |           |          | 
 temp_blks_written   | bigint           |           |          | 
 blk_read_time       | double precision |           |          | 
 blk_write_time      | double precision |           |          | 
 wal_records         | bigint           |           |          | 
 wal_fpi             | bigint           |           |          | 
 wal_bytes           | numeric          |           |          | 
```

含义如下：
| 列 | 描述| 
| -- | -- |
| userid | 执行该语句的用户的OID |
dbid  | 在其中执行该语句的数据库的OID
toplevel | 如果查询作为顶级 SQL 语句执行则为True（pg_stat_statements.track如果设置为top则始终为真）
queryid  | 用于识别相同规范化查询的哈希码
query  | 语句的文本形式
plans  | 计划语句的次数（如果启用了pg_stat_statements.track_planning，否则为零）
total_plan_time  |  计划语句所花费的总时间，以毫秒为单位（如果启用了pg_stat_statements.track_planning，否则为零）
min_plan_time | 计划语句所花费的最短时间，以毫秒为单位（如果启用了pg_stat_statements.track_planning，否则为零）
max_plan_time  | 计划语句所花费的最长时间，以毫秒为单位（如果启用了pg_stat_statements.track_planning，否则为零）
mean_plan_time  | 计划语句所花费的平均时间，以毫秒为单位（如果启用了pg_stat_statements.track_planning，否则为零）
stddev_plan_time | 计划语句花费的时间的总体标准偏差，以毫秒为单位（如果启用了pg_stat_statements.track_planning，否则为零）
calls  | 语句被执行的次数
total_exec_time  | 执行语句所花费的总时间，以毫秒为单位
min_exec_time  | 执行语句所花费的最短时间，以毫秒为单位
max_exec_time  | 执行语句所花费的最长时间，以毫秒为单位
mean_exec_time  | 执行语句的平均时间，以毫秒为单位
stddev_exec_time  | 执行语句花费的时间的总体标准偏差，以毫秒为单位
rows  | 语句检索或影响的总行数
shared_blks_hit  | 语句的共享块缓存命中总数
shared_blks_read  | 语句读取的共享块总数
shared_blks_dirtied | 被语句弄脏的共享块总数
shared_blks_written | 语句写入的共享块总数
local_blks_hit  | 语句的本地块缓存命中总数
local_blks_read  | 语句读取的本地块总数
local_blks_dirtied  | 被语句弄脏的本地块总数
local_blks_written  | 语句写入的本地块总数
temp_blks_read  | 语句读取的临时块总数
temp_blks_written  | 语句写入的临时块总数
blk_read_time | 语句读取块所花费的总时间，以毫秒为单位（如果启用了track_io_timing，否则为零）
blk_write_time  | 语句写入块所花费的总时间，以毫秒为单位（如果启用了track_io_timing，否则为零）
wal_records  | 语句生成的 WAL 记录总数
wal_fpi  | 语句生成的 WAL 整页图像总数
wal_bytes  | SQL 语句生成的 WAL 总量（以字节为单位）


使用示例：
```sql
postgres=# select * from pg_stat_statements;
-[ RECORD 1 ]-------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------
userid              | 10
dbid                | 13010
toplevel            | t
queryid             | -7254762239935552721
query               | select * from t1
plans               | 0
total_plan_time     | 0
min_plan_time       | 0
max_plan_time       | 0
mean_plan_time      | 0
stddev_plan_time    | 0
calls               | 4
total_exec_time     | 1.9633049999999999
min_exec_time       | 0.011065
max_exec_time       | 1.920548
mean_exec_time      | 0.4908262499999999
stddev_exec_time    | 0.8254525675889484
rows                | 4
shared_blks_hit     | 3
shared_blks_read    | 1
shared_blks_dirtied | 1
shared_blks_written | 0
local_blks_hit      | 0
local_blks_read     | 0
local_blks_dirtied  | 0
local_blks_written  | 0
temp_blks_read      | 0
temp_blks_written   | 0
blk_read_time       | 0
blk_write_time      | 0
wal_records         | 0
wal_fpi             | 0
wal_bytes           | 0

```