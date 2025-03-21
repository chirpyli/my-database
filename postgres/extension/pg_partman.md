## PostgreSQL分区管理扩展——pg_partman
pg_partman是一个用于创建和管理表分区的PostgreSQL扩展。它提供了创建和管理基于时间的和基于数字的分区集的功能。从版本5.0.1开始，pg_partman仅支持基于声明的分区管理，而以前的触发器分区方法已被弃用。

PostgreSQL内置的声明式分区功能提供了创建分区表及其子表的命令。pg_partman基于这些内置功能，通过附加特性和增强功能简化分区管理。其核心功能之一是分区表自动维护（例如：按策略新增分区以及删除分区）。pg_partman还实现了将现有表转换为分区表或将分区表转换为现有表的功能。

大多数情况下，pg_partman内置的后台工作进程可自动执行分区维护任务，无需依赖外部调度器如cron。


### 安装
pg_partman（5.0.1版本及以上）的安装如下：
- 要求PostgreSQL版本>=14
- 推荐安装pg_jobmon拓展，若已安装，pg_partman将自动使用其任务监控功能。

1. 启用后台工作进程，在postgresql.conf中添加以下配置：
```shell
shared_preload_libraries = 'pg_partman_bgw'     # (change requires restart)

# 可选配置示例
pg_partman_bgw.interval = 3600    # 维护间隔（秒）
pg_partman_bgw.role = 'postgres'     # 执行角色
pg_partman_bgw.dbname = 'postgres'  # 目标数据库
```

2. 安装扩展
```sql
CREATE SCHEMA partman;
CREATE EXTENSION pg_partman SCHEMA partman;
```

3. 权限配置
pg_partman安装或者运行无需超级管理员权限，建议为pg_partman创建一个专用角色，并授予其所需的权限。例如：
```sql
CREATE ROLE partman_user WITH LOGIN;
GRANT ALL ON SCHEMA partman TO partman_user;
GRANT ALL ON ALL TABLES IN SCHEMA partman TO partman_user;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA partman TO partman_user;
GRANT EXECUTE ON ALL PROCEDURES IN SCHEMA partman TO partman_user;
GRANT ALL ON SCHEMA my_partition_schema TO partman_user;
GRANT TEMPORARY ON DATABASE mydb to partman_user; -- allow creation of temp tables to move data out of default
GRANT CREATE ON DATABASE mydb TO partman_user;
```

### 升级拓展
```sql
ALTER EXTENSION pg_partman UPDATE TO '<latest version>';
```

### 使用

所有分区默认创建DEFAULT分区以容纳尚未分区的数据。通过`check_default()`函数可监控默认表中的数据插入，`partition_data_*`系列函数可轻松将有效数据迁移至正确分区。

创建分区表后，可通过调用`create_parent`函数向`pg_partman`注册该表。此举会根据传递给函数的参数创建必要的分区。该`pg_partman`扩展还提供了`run_maintenance_proc`函数，您可以按计划调用该函数来自动管理分区。为确保根据需要创建正确的分区，请将此函数计划为定期运行（例如每小时）。您还可以确保自动删除分区。

#### create_parent函数
其中，函数`create_parent`是创建分区表的核心函数，会按用户传递的参数创建分区。并将配置信息存储在`part_config`表中。其定义如下：
```sql
CREATE FUNCTION @extschema@.create_parent(
    p_parent_table text             -- 父表的名称
    , p_control text                -- 分区键列名
    , p_interval text               -- 分区间隔（例如，1 day，3 months）
    , p_type text DEFAULT 'range'   -- 分区类型，默认为范围分区
    , p_epoch text DEFAULT 'none'       -- 纪元模式，默认为none，不启用，仅用于时间分区，将时间戳转换为整型存储
    , p_premake int DEFAULT 4           -- 预创建分区数，默认为4
    , p_start_partition text DEFAULT NULL       -- 起始分区值（例如时间值），默认为NULL
    , p_default_table boolean DEFAULT true      -- 是否创建默认分区，默认为true
    , p_automatic_maintenance text DEFAULT 'on'  -- 是否启用自动维护，默认为on
    , p_constraint_cols text[] DEFAULT NULL      -- 约束列，默认为NULL
    , p_template_table text DEFAULT NULL         -- 模板表名，默认为NULL
    , p_jobmon boolean DEFAULT true              -- 是否启用作业监控，默认为true
    , p_date_trunc_interval text DEFAULT NULL    -- 时间截断间隔，用于对齐分区边界，默认为NULL
    , p_control_not_null boolean DEFAULT true    -- 是否强制分区键列为NOT NULL
    , p_time_encoder text DEFAULT NULL           -- 时间编码函数（自定义序列化），默认为NULL
    , p_time_decoder text DEFAULT NULL           -- 时间解码函数（自定义反序列化），默认为NULL
)
    RETURNS boolean      -- 返回是否成功
    LANGUAGE plpgsql
```

#### part_config表
`part_config`表用于存储分区表的配置信息，包括父表、分区键、分区类型、分区间隔、预创建分区数等。维护函数`run_maintenance`可以根据配置信息进行自动创建和删除分区等操作。
```sql
-- part_config分区配置信息表
CREATE TABLE @extschema@.part_config (
    parent_table text NOT NULL          -- 父表的名称
    , control text NOT NULL              -- 分区键列名
    , time_encoder text                 -- 时间编码函数（自定义序列化）
    , time_decoder text                 -- 时间解码函数（自定义反序列化）
    , partition_interval text NOT NULL   -- 分区间隔
    , partition_type text NOT NULL       -- 分区类型
    , premake int NOT NULL DEFAULT 4     -- 预创建分区数
    , automatic_maintenance text NOT NULL DEFAULT 'on'    --    是否启用自动维护
    , template_table text                                  -- 模板表名
    , retention text                        -- 保留策略
    , retention_schema text                -- 保留策略模式
    , retention_keep_index boolean NOT NULL DEFAULT true    -- 是否保留索引
    , retention_keep_table boolean NOT NULL DEFAULT true    -- 是否保留表
    , epoch text NOT NULL DEFAULT 'none'                    -- 纪元模式    
    , constraint_cols text[]                                -- 约束列
    , optimize_constraint int NOT NULL DEFAULT 30           -- 约束优化
    , infinite_time_partitions boolean NOT NULL DEFAULT false  -- 是否允许无限时间分区
    , datetime_string text     -- 时间字符串格式
    , jobmon boolean NOT NULL DEFAULT true     -- 是否启用作业监控
    , sub_partition_set_full boolean NOT NULL DEFAULT false  -- 是否允许子分区
    , undo_in_progress boolean NOT NULL DEFAULT false        -- 是否启用撤销
    , inherit_privileges boolean DEFAULT false               -- 是否继承父表的权限
    , constraint_valid boolean DEFAULT true NOT NULL         -- 是否启用约束校验
    , ignore_default_data boolean NOT NULL DEFAULT true      -- 是否忽略默认数据
    , date_trunc_interval text                               -- 时间截断间隔
    , maintenance_order int                                   -- 维护顺序
    , retention_keep_publication boolean NOT NULL DEFAULT false  -- 是否保留 publication
    , maintenance_last_run timestamptz                     -- 最后一次维护时间
    , CONSTRAINT part_config_parent_table_pkey PRIMARY KEY (parent_table)  -- 主键约束
    , CONSTRAINT positive_premake_check CHECK (premake > 0)   -- 预创建分区数检查
);
```

#### run_maintenance函数
函数`run_maintenance`以及存储过程`run_maintenance_proc`用于运行分区维护任务，根据分区策略自动创建和删除分区。定义如下：
```sql
-- 函数用于运行分区维护任务
CREATE FUNCTION @extschema@.run_maintenance(
    p_parent_table text DEFAULT NULL
    -- If these defaults change reflect them in `run_maintenance_proc`!
    , p_analyze boolean DEFAULT false  -- 是否执行ANALYZE，维护后自动执行 ANALYZE，更新表的统计信息以优化查询性能。
    , p_jobmon boolean DEFAULT true  -- 是否启用作业监控，依赖pg_jobmon拓展
)
    RETURNS void
    LANGUAGE plpgsql

-- 创建一个存储过程，用于运行分区维护任务。
CREATE PROCEDURE @extschema@.run_maintenance_proc(
    p_wait int DEFAULT 0
    -- Keep these defaults in sync with `run_maintenance`!
    , p_analyze boolean DEFAULT false
    , p_jobmon boolean DEFAULT true
)
    LANGUAGE plpgsql
```
手动执行分区维护任务：
```sql
postgres=# select partman.run_maintenance();
 run_maintenance 
-----------------
 
(1 row)
```

#### 查看维护计划
可以通过表`part_config`查看分区表的维护计划。
```sql
-- 查询所有已配置的分区表及其维护参数
postgres=# select * from partman.part_config;
-[ RECORD 1 ]--------------+-----------------------------------------
parent_table               | partman_test.time_taptest_table
control                    | col3
time_encoder               | 
time_decoder               | 
partition_interval         | 1 day
partition_type             | range
premake                    | 4
automatic_maintenance      | on
template_table             | partman_test.time_taptest_table_template
retention                  | 
retention_schema           | 
retention_keep_index       | t
retention_keep_table       | t
epoch                      | none
constraint_cols            | 
optimize_constraint        | 30
infinite_time_partitions   | f
datetime_string            | YYYYMMDD
jobmon                     | t
sub_partition_set_full     | f
undo_in_progress           | f
inherit_privileges         | f
constraint_valid           | t
ignore_default_data        | t
date_trunc_interval        | 
maintenance_order          | 
retention_keep_publication | f
maintenance_last_run       | 2025-03-19 17:13:27.227666+08
```

#### 使用示例：按时间分区
对于原生分区，必须从一个已经设置为所需分区类型的父表开始。目前，`pg_partman`仅支持`RANGE`类型的分区（包括时间和服务ID）。你不能将一个非分区表转换为分区集的父表，这可能会使迁移变得困难。在此示例中，我们将从一个全新的表开始。在 PG11 及以上版本中，任何非唯一索引也可以添加到父表上，并且会自动创建在所有子表上。
1. 创建一个分区表：
```sql
CREATE SCHEMA IF NOT EXISTS partman_test;

CREATE TABLE partman_test.time_taptest_table
    (col1 int,
    col2 text default 'stuff',
    col3 timestamptz NOT NULL DEFAULT now())
PARTITION BY RANGE (col3);

CREATE INDEX ON partman_test.time_taptest_table (col3);

-- 查看分区表
postgres=# \d+ partman_test.time_taptest_table
                                      Partitioned table "partman_test.time_taptest_table"
 Column |           Type           | Collation | Nullable |    Default    | Storage  | Compression | Stats target | Description 
--------+--------------------------+-----------+----------+---------------+----------+-------------+--------------+-------------
 col1   | integer                  |           |          |               | plain    |             |              | 
 col2   | text                     |           |          | 'stuff'::text | extended |             |              | 
 col3   | timestamp with time zone |           | not null | now()         | plain    |             |              | 
Partition key: RANGE (col3)  -- 分区键
Indexes:
    "time_taptest_table_col3_idx" btree (col3)
Number of partitions: 0
```
唯一索引（包括主键）不能在原生分区表上创建，除非它们包含分区键。对于基于时间的分区，这通常行不通，因为这会限制每个子表中只能有一个时间戳值。`pg_partman`通过使用模板表来管理目前原生分区不支持的属性，从而帮助解决这个问题。

在这个例子中，我们将首先手动创建模板表，这样在运行`create_parent()`时，创建的初始子表将具有主键。如果你不向`pg_partman`提供模板表，它会在你安装扩展的模式中为你创建一个。但是，你添加到该模板的属性只会应用于该时间点后新创建的子表。你必须手动追溯地将这些属性应用到已经存在的子表上。
```sql
CREATE TABLE partman_test.time_taptest_table_template (LIKE partman_test.time_taptest_table);

ALTER TABLE partman_test.time_taptest_table_template ADD PRIMARY KEY (col1); -- 添加主键

-- 查看模板表
postgres=# \d+ partman_test.time_taptest_table_template
                                     Table "partman_test.time_taptest_table_template"
 Column |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
--------+--------------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 col1   | integer                  |           | not null |         | plain    |             |              | 
 col2   | text                     |           |          |         | extended |             |              | 
 col3   | timestamp with time zone |           | not null |         | plain    |             |              | 
Indexes:
    "time_taptest_table_template_pkey" PRIMARY KEY, btree (col1)
Access method: heap
```
2. 注册父表：`create_parent`函数将创建初始子表，并设置一些初始配置。它需要以下参数：
```sql
postgres=# SELECT partman.create_parent(
    p_parent_table := 'partman_test.time_taptest_table'      -- 父表名
    , p_control := 'col3'                                    -- 分区键
    , p_interval := '1 day'                                  -- 分区间隔
    , p_template_table := 'partman_test.time_taptest_table_template'    -- 模板表名
);
 create_parent 
---------------
 t
(1 row)

-- 查看part_config配置表，可以看到父表、分区键、分区类型、分区间隔等信息
postgres=# select * from partman.part_config;
-[ RECORD 1 ]--------------+-----------------------------------------
parent_table               | partman_test.time_taptest_table
control                    | col3
time_encoder               | 
time_decoder               | 
partition_interval         | 1 day
partition_type             | range
premake                    | 4
automatic_maintenance      | on
template_table             | partman_test.time_taptest_table_template
retention                  | 
retention_schema           | 
retention_keep_index       | t
retention_keep_table       | t
epoch                      | none
constraint_cols            | 
optimize_constraint        | 30
infinite_time_partitions   | f
datetime_string            | YYYYMMDD
jobmon                     | t
sub_partition_set_full     | f
undo_in_progress           | f
inherit_privileges         | f
constraint_valid           | t
ignore_default_data        | t
date_trunc_interval        | 
maintenance_order          | 
retention_keep_publication | f
maintenance_last_run       | 2025-03-19 17:13:27.227666+08

-- 查看创建的分区表，可以看到创建了很多分区以及default分区
postgres=# \d+ partman_test.time_taptest_table
                                      Partitioned table "partman_test.time_taptest_table"
 Column |           Type           | Collation | Nullable |    Default    | Storage  | Compression | Stats target | Description 
--------+--------------------------+-----------+----------+---------------+----------+-------------+--------------+-------------
 col1   | integer                  |           |          |               | plain    |             |              | 
 col2   | text                     |           |          | 'stuff'::text | extended |             |              | 
 col3   | timestamp with time zone |           | not null | now()         | plain    |             |              | 
Partition key: RANGE (col3)
Indexes:
    "time_taptest_table_col3_idx" btree (col3)
Partitions: partman_test.time_taptest_table_p20250315 FOR VALUES FROM ('2025-03-15 00:00:00+08') TO ('2025-03-16 00:00:00+08'),
            partman_test.time_taptest_table_p20250316 FOR VALUES FROM ('2025-03-16 00:00:00+08') TO ('2025-03-17 00:00:00+08'),
            partman_test.time_taptest_table_p20250317 FOR VALUES FROM ('2025-03-17 00:00:00+08') TO ('2025-03-18 00:00:00+08'),
            partman_test.time_taptest_table_p20250318 FOR VALUES FROM ('2025-03-18 00:00:00+08') TO ('2025-03-19 00:00:00+08'),
            partman_test.time_taptest_table_p20250319 FOR VALUES FROM ('2025-03-19 00:00:00+08') TO ('2025-03-20 00:00:00+08'),
            partman_test.time_taptest_table_p20250320 FOR VALUES FROM ('2025-03-20 00:00:00+08') TO ('2025-03-21 00:00:00+08'),
            partman_test.time_taptest_table_p20250321 FOR VALUES FROM ('2025-03-21 00:00:00+08') TO ('2025-03-22 00:00:00+08'),
            partman_test.time_taptest_table_p20250322 FOR VALUES FROM ('2025-03-22 00:00:00+08') TO ('2025-03-23 00:00:00+08'),
            partman_test.time_taptest_table_p20250323 FOR VALUES FROM ('2025-03-23 00:00:00+08') TO ('2025-03-24 00:00:00+08'),
            partman_test.time_taptest_table_default DEFAULT


-- 我们查看创建的子表
postgres=# \d+ partman_test.time_taptest_table_p20250319
                                       Table "partman_test.time_taptest_table_p20250319"
 Column |           Type           | Collation | Nullable |    Default    | Storage  | Compression | Stats target | Description 
--------+--------------------------+-----------+----------+---------------+----------+-------------+--------------+-------------
 col1   | integer                  |           | not null |               | plain    |             |              | 
 col2   | text                     |           |          | 'stuff'::text | extended |             |              | 
 col3   | timestamp with time zone |           | not null | now()         | plain    |             |              | 
Partition of: partman_test.time_taptest_table FOR VALUES FROM ('2025-03-19 00:00:00+08') TO ('2025-03-20 00:00:00+08')
Partition constraint: ((col3 IS NOT NULL) AND (col3 >= '2025-03-19 00:00:00+08'::timestamp with time zone) AND (col3 < '2025-03-20 00:00:00+08'::timestamp with time zone))
Indexes:
    "time_taptest_table_p20250319_pkey" PRIMARY KEY, btree (col1)
    "time_taptest_table_p20250319_col3_idx" btree (col3)
Access method: heap
```
向表中插入数据并进行查询测试一下：
```sql
postgres=# insert into partman_test.time_taptest_table values(1, 'jincheng');
INSERT 0 1
postgres=# select * from partman_test.time_taptest_table;
 col1 |   col2   |             col3              
------+----------+-------------------------------
    1 | jincheng | 2025-03-19 15:06:42.260531+08
(1 row)
-- 插入一个数据，可以看到数据被插入到了default分区中
postgres=# insert into partman_test.time_taptest_table values(2, 'taiyuan', now() - '1 week'::interval);
INSERT 0 1
postgres=# select * from partman_test.time_taptest_table;
 col1 |   col2   |             col3              
------+----------+-------------------------------
    1 | jincheng | 2025-03-19 15:06:42.260531+08
    2 | taiyuan  | 2025-03-12 15:54:33.82568+08
(2 rows)
-- 查询default分区中的数据
postgres=# select * from partman_test.time_taptest_table_default ;
 col1 |  col2   |             col3             
------+---------+------------------------------
    2 | taiyuan | 2025-03-12 15:54:33.82568+08
(1 row)
```