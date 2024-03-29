# my-database
database study, include PostgreSQL, distribute system  and so on.


## 目录

### SQL
- [SQL必知必会笔记](./sql/SQL必知必会笔记.md)
- [SQL查询](./sql/sql-select.md)


### SQL引擎

#### 解析器
- [词法分析与flex](./postgres/词法分析与flex.md)

#### 优化器

- [逻辑优化——常量表达式化简](./query-optimizer/simplify_const_expression.md)

### PostgreSQL

- [PostgreSQL源码分析——数据页校验](./postgres/pg-src-analyze-checksum.md)
- [PostgreSQL源码分析——外存管理](./postgres/pg-src-analyze-smgr.md)

#### 源码分析——DML


- [PostgreSQL中列名表名长度限制](./postgres/PostgreSQL中表名列名长度限制.md)
- [PostgreSQL如何添加内核函数](./postgres/PostgreSQL如何添加内核函数.md)
- [SQL解析器](./postgres/SQL解析器.md)
- [查询优化之子连接优化](./postgres/查询优化之子连接优化.md)
- [逻辑优化——子查询优化](./query-optimizer/subquery-optimizer.md)
- [Postgres源码分析——scan.l](./pg-src-analyze-scaan.l.md)
- [Postgres源码分析——UPDATE](./postgres/pg-src-analyze-update.md)
- [Postgres源码分析——INSERT](./postgres/pg-src-analyze-insert.md)
- [Postgres源码分析——执行计划EXPLAIN](./postgres/pg-src-analyze-explain.md)

- [Postgres源码分析——UPDATE分区表](./postgres/update-partiton-source-analyze.md)
- [Postgres源码分析——SeqScan](./postgres/pg-src-analyze-seqscan.md)
- [Postgres源码分析——IndexScan](./postgres/pg-src-analyze-indexscan.md)
- [Postgres源码分析—— ValueScan](./postgres/pg-src-analyze-valuescan.md)
- [Postgres源码分析 —— FunctionScan](./postgres/pg-src-analyze-create-function.md)

- [Postgres源码分析——存储过程调用CALL](./postgres/pg-src-analyze-call.md)
- [Postgres源码分析——CREATE FUNCTION](./postgres/pg-src-analyze-create-function.md)

- [Postgres源码分析——CTE](./postgres/pg-src-analyze-cte.md)
- [Postgres源码分析——绑定变量](./postgres/pg-src-analyze-cacheplan-1.md)
- [Postgres源码分析——COPY](./postgres/pg-src-analyze-copy.md)

#### 源码分析 —— 类型系统
- [Postgres源码分析——CREATE TYPE](./postgres/pg-src-analyze-create-type.md)
- [Postgres源码分析——CREATE CAST](./postgres/pg-src-analyze-create-cast.md)
- [Postgres源码分析——CAST类型转换](./postgres/pg-src-analyze-cast.md)
- [Postgres源码分析——除法运算符](./postgres/pg-src-analyze-operator-div.md)

#### 源码分析 —— 建表相关

- [Postgres源码分析——CREATE DATABASE](./postgres/pg-src-analyze-create-database.md)
- [Postgres源码分析——CREATE SCHEMA](./postgres/pg-src-analyze-create-schema.md)
- [Postgres源码分析——CREATE VIEW](./postgres/pg-src-analyze-create-view.md)
- [Postgres源码分析——创建分区表](./postgres/pg-src-analyze-create-partition-table.md)
- [Postgres源码分析——建表含有序列](./postgres/pg-src-analyze-create-table-series.md)
- [Postgres源码分析 —— 范围表](./postgres/pg-src-analyze-cte.md)

#### 源码分析 —— 日志相关
- [Postgres源码分析——CHECKPOINT](./postgres/pg-src-analyze-checkpoint.md)
- [Postgres源码分析——基础备份](./postgres/pg-src-analyze-basebackup.md)
- [Postgres源码分析——日志归档](./postgres/pg-src-analyze-archiver.md)
- [Postgres源码分析——WAL日志（一）](./postgres/pg-src-analyze-wal-part1.md)
- [Postgres源码分析——WAL日志（二）](./postgres/pg-src-analyze-wal-part2.md)
- [Postgres源码分析——WAL日志清理](./postgres/pg-src-analyze-wal-clean.md)
#### 源码分析 —— 后台进程
- [Postgres源码分析——bgwriter](./postgres/pg-src-analyze-bgwriter.md)
- [Postgres源码分析——启动流程](./postgres/pg-src-analyze-start.md)


#### 源码分析 —— DCL
- [Postgres源码分析——创建用户](./postgres/pg-src-analyze-create-user.md)
- [Postgres源码分析——口令认证](./postgres/pg-src-analyze-auth.md)

#### 源码分析 —— 工具
- [Postgres源码分析——pg_basebackup](./postgres/pg-src-analyze-pg_basebackup.md)
- [Postgres源码分析——initdb](./postgres/pg-src-analyze-initdb.md)
- [Postgres源码分析——psql](./postgres/pg-src-analyze-psql.md)


### greenplum
- [Greenplum执行计划分析](./greenplum/gp-plan-analyze.md)
- [Greenplum源码分析——简单查询SELECT](./greenplum/gp-src-analyze-select.md)
### mysql
- [MySQL源码分析——除法运算符](./mysql/mysql-src-analyze-operator-div.md)
- [MySQL源码分析——str_to_date函数](./mysql/mysql-src-analyze-str_to_date.md)
### leveldb
- [leveldb工作原理](./leveldb/leveldb.md)
- [leveldb学习笔记](./leveldb/leveldb学习笔记1.md)

### TiDB
- [单机部署Tikv](./tidb/单机部署Tikv.md)

### 行业相关
- [数据库开源项目参考](./数据库开源项目参考.md)
- [数据库行业认识](./数据库行业认识.md)
- [对OLTP、OLAP、HTAP的一点点思考](./对HTAP的一点点思考.md)