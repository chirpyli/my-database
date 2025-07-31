# my-database
数据库学习笔记，分享，交流！


## SQL篇

- [SQL必知必会笔记](./sql/SQL必知必会笔记.md)
- [SQL查询](./sql/sql-select.md)
- [PostgreSQL外部数据包装器](./postgres/pg-foreign-table.md)

## 数据库系统原理篇


## 数据库篇

### PostgreSQL
开源关系型集中式数据库。

#### PostgreSQL协议
- [libpq原理分析](./postgres/pg-protocol/libpq.md)
- [Postgres源码分析——口令认证](./postgres/pg-protocol/pg-src-analyze-auth.md)
- [Postgres源码分析——psql](./postgres/pg-protocol/pg-src-analyze-psql.md)

#### SQL引擎

##### 解析器
- [词法分析与flex](./postgres/sql-engine/词法分析与flex.md)
- [SQL解析器](./postgres/sql-engine/sql-parser.md)
- [PostgreSQL中列名表名长度限制](./postgres/PostgreSQL中表名列名长度限制.md)
- [Postgres源码分析——scan.l](./pg-src-analyze-scaan.l.md)
- [Postgres源码分析——INSERT](./postgres/sql-engine/pg-src-analyze-insert.md)
- [Postgres视图实现原理](./postgres/sql-engine/pg-analyze-view.md)
- [PostgreSQL如何添加内核函数](./postgres/PostgreSQL如何添加内核函数.md)

- [Postgres源码分析——执行计划EXPLAIN](./postgres/pg-src-analyze-explain.md)
- [Postgres源码分析——UPDATE](./postgres/pg-src-analyze-update.md)

- [Postgres源码分析——UPDATE分区表](./postgres/update-partiton-source-analyze.md)

- [Postgres源码分析——SeqScan](./postgres/sql-engine/pg-src-analyze-seqscan.md)
- [Postgres源码分析——IndexScan](./postgres/pg-src-analyze-indexscan.md)
- [Postgres源码分析—— ValueScan](./postgres/pg-src-analyze-valuescan.md)
- [Postgres源码分析 —— FunctionScan](./postgres/pg-src-analyze-functionscan.md)

- [Postgres源码分析——存储过程调用CALL](./postgres/pg-src-analyze-call.md)
- [Postgres源码分析——CREATE FUNCTION](./postgres/pg-src-analyze-create-function.md)

- [Postgres源码分析——CTE](./postgres/pg-src-analyze-cte.md)
- [Postgres源码分析——绑定变量](./postgres/pg-src-analyze-cacheplan-1.md)
- [Postgres源码分析——COPY](./postgres/pg-src-analyze-copy.md)

- [Postgres源码分析——绑定变量](./postgres/pg-src-analyze-cacheplan-1.md)

- [PostgreSQL源码分析——CREATE SERVER](./postgres/pg-src-analyze-create-server.md)
- [Postgres源码分析之CTE](./postgres/pg-src-analyze-cte.md)
- [Postgres源码分析 —— unnest函数](./postgres/pg-src-analyze-function-unnest.md)

##### 规则系统
- [Postgres源码分析——视图查询重写](./postgres/pg-src-analyze-rewrite-view.md)

##### 优化器

- [逻辑优化——常量表达式化简](./postgres/sql-engine/simplify_const_expression.md)
- [查询优化之子连接优化](./postgres/sql-engine/pg-subquery-optimizer.md)
- [逻辑优化——子查询优化](./postgres/sql-engine/subquery-optimizer.md)

##### 执行器
- [PostgreSQL聚合算子实现原理](./postgres/sql-engine/pg-agg.md)

#### 存储引擎
- [PostgreSQL源码分析——缓冲区](./postgres/storage-engine/pg-src-analyze-shared-buffer.md)
- [PostgreSQL源码分析——数据页校验](./postgres/storage-engine/pg-src-analyze-checksum.md)
- [PostgreSQL源码分析——外存管理](./postgres/storage-engine/pg-src-analyze-smgr.md)
- [PostgreSQL源码分析——空闲空间映射表](./postgres/storage-engine/pg-src-analyze-freespacemap.md)
- [PostgreSQL中TOAST机制](./postgres/storage-engine/pg-toast.md)
- [PostgreSQL中大对象存储](./postgres/storage-engine/pg-largeobject.md)

##### 建表相关

- [Postgres源码分析——CREATE DATABASE](./postgres/pg-src-analyze-create-database.md)
- [Postgres源码分析——CREATE SCHEMA](./postgres/pg-src-analyze-create-schema.md)

- [Postgres源码分析——创建分区表](./postgres/pg-src-analyze-create-partition-table.md)
- [Postgres源码分析——建表含有序列](./postgres/pg-src-analyze-create-table-series.md)
- [Postgres源码分析 —— CTE](./postgres/pg-src-analyze-cte.md)
- [Postgres源码分析——范围表](./postgres/pg-src-analyze-rte.md)

##### 类型系统
- [Postgres源码分析——CREATE TYPE](./postgres/sql-engine/pg-src-analyze-create-type.md)
- [Postgres源码分析——CREATE CAST](./postgres/sql-engine/pg-src-analyze-create-cast.md)
- [Postgres源码分析——CAST类型转换](./postgres/sql-engine/pg-src-analyze-cast.md)
- [Postgres源码分析——除法运算符](./postgres/pg-src-analyze-operator-div.md)



##### WAL日志相关
- [Postgres源码分析——CHECKPOINT](./postgres/pg-src-analyze-checkpoint.md)
- [Postgres源码分析——基础备份](./postgres/pg-src-analyze-basebackup.md)
- [Postgres源码分析——日志归档](./postgres/pg-src-analyze-archiver.md)
- [Postgres源码分析——WAL日志（一）](./postgres/pg-src-analyze-wal-part1.md)
- [Postgres源码分析——WAL日志（二）](./postgres/pg-src-analyze-wal-part2.md)
- [Postgres源码分析——WAL日志清理](./postgres/pg-src-analyze-wal-clean.md)
- [PostgreSQL源码分析 —— pg_control](./postgres/pg-src-analyze-pg_control.md)
- [PostgreSQL备机回放流程](./postgres/pg-src-analyze-standby.md)
- [PostgreSQL源码分析——SLRU缓冲池](./postgres/pg-src-analyze-slru.md)

#### 后台进程
- [Postgres源码分析——bgwriter](./postgres/pg-src-analyze-bgwriter.md)
- [Postgres源码分析——启动流程](./postgres/pg-src-analyze-start.md)


#### 安全类
- [Postgres源码分析——创建用户](./postgres/pg-src-analyze-create-user.md)

- [PostgreSQL源码分析——对象访问控制](./postgres/pg-src-analyze-rbac.md)


#### 工具类
- [Postgres源码分析——pg_basebackup](./postgres/pg-src-analyze-pg_basebackup.md)
- [Postgres源码分析——initdb](./postgres/pg-src-analyze-initdb.md)

- [Postgres源码分析——日志进程](./postgres/pg-src-analyze-log.md)

#### 异步通知
- [Postgres源码分析——异步通知](./postgres/pg-listen-notify.md)

#### 扩展
- [PostgreSQL源码分析——pg_stat_statements](./postgres/extension/pg_stat_statements.md)
- [PostgreSQL拓展之auth_delay](./postgres/extension/auth_delay.md)
- [PostgreSQL源码分析——auto_explain](./postgres/extension/auto_explain.md)
- [PostgreSQL执行计划过滤扩展——plan_filter](./postgres/extension/pg_plan_filter.md)
- [PostgreSQL优化器提示扩展——pg_hint_plan](./postgres/extension/pg_hint_plan.md)
- [PostgreSQL bloom索引扩展——bloom](./postgres/extension/bloom.md)
- [PostgreSQL源码分析——pg_buffercache](./postgres/extension/pg-src-analyze-buffercache.md)
- [PostgreSQL审计插件pgaudit](./postgres/extension/pg-src-analyze-pgaudit.md)
- [PostgreSQL插件之pg_freespacemap](./postgres/extension/pg_freespacemap.md)
- [PostgreSQL数据库插件——pg_visibility](./postgres/extension/pg_visibility.md)
- [PostgreSQL定时任务插件——pg_cron](./postgres/extension/pg_cron.md)
- [PostgreSQL消息队列拓展——PGMQ](./postgres/extension/pgmq.md)
- [PostgreSQL消息队列拓展——PGQ](./postgres/extension/pgq.md)
- [PostgreSQL分区管理拓展——pg_partman](./postgres/extension/pg_partman.md)
- [PostgreSQL消息队列拓展PGMQ实现原理分析](./postgres/extension/pgmq-analyze.md)
- [PostgreSQL消息队列拓展PGQ实现原理分析](./postgres/extension/pgq-analyze.md)
- [向量拓展pgvector](./postgres/extension/pgvector.md)
- [citus建表分析](./postgres/extension/citus-analyze-create-distributed-table.md)

#### 其他
- [PostgreSQL数据结构List解析](./postgres/pg-list.md)
- [PostgreSQL位图集合](./postgres/pg-bitmapset.md)

### Greenplum
基于PostgreSQL的MPP分布式数据库。目前已经闭源，但是代码还可以看到。
- [Greenplum执行计划分析](./greenplum/gp-plan-analyze.md)
- [Greenplum源码分析——简单查询SELECT](./greenplum/gp-src-analyze-select.md)

### Citus
PostgreSQL的插件，开源分布式数据库。


### openGauss
华为开源的数据库，基于PostgreSQL。
- [openGauss支持堆表预读](./opengauss/opengauss-heap-table-pre-read.md)

### openHalo
- [openHalo初识](./openhalo/first-use-openHalo.md)
- [openHalo的启动流程](./openhalo/openhalo-start.md)

### MySQL

- [MySQL源码分析——除法运算符](./mysql/mysql-src-analyze-operator-div.md)
- [MySQL源码分析——str_to_date函数](./mysql/mysql-src-analyze-str_to_date.md)

### LevelDB
- [leveldb工作原理](./leveldb/leveldb.md)
- [leveldb学习笔记](./leveldb/leveldb学习笔记1.md)

### Redis
- [redis基础](./redis/redis.md)
- [redis功能](./redis/redis-functions.md)
- [redis如何执行一条命令](./redis/how-to-exec-redis-command.md)
- [Redis中的持久化机制——AOF篇](./redis/redis-aof.md)
- [Redis中的持久化机制——RDB篇](./redis/redis-rdb.md)
- [Redis事务](./redis/redis-transaction.md)
- [Redis数据类型Hash实现原理](./redis/redis-hash.md)]
- [Redis数据类型String实现原理](./redis/redis-string.md)]

### TiDB
- [单机部署Tikv](./tidb/单机部署Tikv.md)

### [AntDB](https://antdb.net/us)
亚信科技的分布式数据库。基于PostgreSQL。曾经开源，但是github和gitee都已经搜不到了，目前已闭源？
- [antdb](./antdb/antdb.md)

### 行业相关
- [数据库开源项目参考](./数据库开源项目参考.md)
- [数据库行业认识](./数据库行业认识.md)
- [对OLTP、OLAP、HTAP的一点点思考](./对HTAP的一点点思考.md)


---

数据库领域公众号：
![内核思考](./.images/public-account-think-kernel.jpg)