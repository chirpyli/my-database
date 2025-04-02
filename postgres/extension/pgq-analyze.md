## PostgreSQL消息队列拓展PGQ实现原理分析
之前我们分析过另一款消息队列拓展PGMQ的实现原理，现在我们继续分析PGQ的实现原理。

PGQ是一款PostgreSQL消息队列拓展，有关其如何使用可参考文章[PostgreSQL消息队列拓展——PGQ](./pgq.md)。下面我们之间分析其实现原理。

### 队列的实现
消息队列，最重要的数据结构是队列，PGQ中，队列的实现是通过PostgreSQL的表来实现的。具体的我们看一下在PGQ中是如何实现的。

#### 创建队列
创建队列，实际上就是创建表来实现队列，具体实现如下：
```sql
create or replace function pgq.create_queue(i_queue_name text)
returns integer as $$
-- ----------------------------------------------------------------------
-- Function: pgq.create_queue(1)
--
--      Creates new queue with given name.
--
-- Returns:
--      0 - queue already exists
--      1 - queue created
-- Calls:
--      pgq.grant_perms(i_queue_name);
--      pgq.ticker(i_queue_name);
--      pgq.tune_storage(i_queue_name);
-- Tables directly manipulated:
--      insert - pgq.queue
--      create - pgq.event_N () inherits (pgq.event_template)
--      create - pgq.event_N_0 .. pgq.event_N_M () inherits (pgq.event_N) 实际上就是分区表
-- ----------------------------------------------------------------------
declare
    tblpfx   text;
    tblname  text;
    idxpfx   text;
    idxname  text;
    sql      text;
    id       integer;
    tick_seq text;
    ev_seq text;
    n_tables integer;
begin
    if i_queue_name is null then
        raise exception 'Invalid NULL value';
    end if;

    -- check if exists
    perform 1 from pgq.queue where queue_name = i_queue_name;
    if found then
        return 0;
    end if;

    -- insert event
    id := nextval('pgq.queue_queue_id_seq');
    tblpfx := 'pgq.event_' || id::text;
    idxpfx := 'event_' || id::text;
    tick_seq := 'pgq.event_' || id::text || '_tick_seq';
    ev_seq := 'pgq.event_' || id::text || '_id_seq';
    insert into pgq.queue (queue_id, queue_name,
            queue_data_pfx, queue_event_seq, queue_tick_seq)
        values (id, i_queue_name, tblpfx, ev_seq, tick_seq); -- 插入到队列信息表中

    select queue_ntables into n_tables from pgq.queue
        where queue_id = id;

    -- create seqs
    execute 'CREATE SEQUENCE ' || pgq.quote_fqname(tick_seq);
    execute 'CREATE SEQUENCE ' || pgq.quote_fqname(ev_seq);

    -- create data tables
    execute 'CREATE TABLE ' || pgq.quote_fqname(tblpfx) || ' () '
            || ' INHERITS (pgq.event_template)';
    for i in 0 .. (n_tables - 1) loop
        tblname := tblpfx || '_' || i::text;
        idxname := idxpfx || '_' || i::text || '_txid_idx';
        execute 'CREATE TABLE ' || pgq.quote_fqname(tblname) || ' () '
                || ' INHERITS (' || pgq.quote_fqname(tblpfx) || ')';
        execute 'ALTER TABLE ' || pgq.quote_fqname(tblname) || ' ALTER COLUMN ev_id '
                || ' SET DEFAULT nextval(' || quote_literal(ev_seq) || ')';
        execute 'create index ' || quote_ident(idxname) || ' on '
                || pgq.quote_fqname(tblname) || ' (ev_txid)';
    end loop; -- 创建队列数据表

    perform pgq.grant_perms(i_queue_name);

    perform pgq.ticker(i_queue_name);
    -- 设置表的fillfactor = 100，关闭autovacuum_enabled=off。
    perform pgq.tune_storage(i_queue_name);  

    return 1;
end;
$$ language plpgsql security definer;
```
可以看到，当创建一个队列时，其实是创建了一个继承自`pgq.event_template`的表，表名是`pgq.event_{queue_id}`，同时创建了两个序列，一个用于生成`tick_id`，一个用于生成`ev_id`，同时创建了多个继承自`pgq.event_{queue_id}`的表，表名是`pgq.event_{queue_id}_{i}`，其中i从0开始递增，用于存储事件数据。这里采用多个表来存储消息事件是出于性能考虑，单个表也能实现队列的功能。（这里补充说明一下，PGMQ中，也可使用分区表来实现队列）

#### 删除队列
删除队列，其实就是把创建队列所建立的表删除掉，删除队列需要注意，需要判断当前有无消费者在使用，如果存在消费者在使用，则不允许删除队列，除非是`x_force`为`true`强制删除的情况，则会注销所有消费者，然后删除队列。具体实现如下：
```sql
create or replace function pgq.drop_queue(x_queue_name text, x_force bool)
returns integer as $$
-- ----------------------------------------------------------------------
-- Function: pgq.drop_queue(2)
--
--     Drop queue and all associated tables.
--
-- Parameters:
--      x_queue_name    - queue name
--      x_force         - ignore (drop) existing consumers
-- Returns:
--      1 - success
-- Calls:
--      pgq.unregister_consumer(queue_name, consumer_name)
--      perform pgq.ticker(i_queue_name);
--      perform pgq.tune_storage(i_queue_name);
-- Tables directly manipulated:
--      delete - pgq.queue
--      drop - pgq.event_N (), pgq.event_N_0 .. pgq.event_N_M 
-- ----------------------------------------------------------------------
declare
    tblname  text;
    q record;
    num integer;
begin
    -- check if exists
    select * into q from pgq.queue
        where queue_name = x_queue_name
        for update;
    if not found then
        raise exception 'No such event queue';
    end if;

    if x_force then
        perform pgq.unregister_consumer(queue_name, consumer_name)
           from pgq.get_consumer_info(x_queue_name);
    else
        -- check if no consumers
        select count(*) into num from pgq.subscription
            where sub_queue = q.queue_id;
        if num > 0 then
            raise exception 'cannot drop queue, consumers still attached';
        end if;
    end if;

    -- drop data tables
    for i in 0 .. (q.queue_ntables - 1) loop
        tblname := q.queue_data_pfx || '_' || i::text;
        execute 'DROP TABLE ' || pgq.quote_fqname(tblname);
    end loop;
    execute 'DROP TABLE ' || pgq.quote_fqname(q.queue_data_pfx);

    -- delete ticks
    delete from pgq.tick where tick_queue = q.queue_id;

    -- drop seqs
    -- FIXME: any checks needed here?
    execute 'DROP SEQUENCE ' || pgq.quote_fqname(q.queue_tick_seq);
    execute 'DROP SEQUENCE ' || pgq.quote_fqname(q.queue_event_seq);

    -- delete event
    delete from pgq.queue
        where queue_name = x_queue_name;

    return 1;
end;
$$ language plpgsql security definer;
```

#### 事件模版表
`pgq.event_template`所有队列事件表都继承自该表，该表定义了队列事件的基本字段，包括事件id、事件类型、事件数据、事件时间、事件事务id等。
```sql
-- ----------------------------------------------------------------------
-- Table: pgq.event_template
--
--      Parent table for all event tables
--
-- Columns:
--      ev_id               - event's id, supposed to be unique per queue
--      ev_time             - when the event was inserted
--      ev_txid             - transaction id which inserted the event
--      ev_owner            - subscription id that wanted to retry this
--      ev_retry            - how many times the event has been retried, NULL for new events
--      ev_type             - consumer/producer can specify what the data fields contain
--      ev_data             - data field
--      ev_extra1           - extra data field
--      ev_extra2           - extra data field
--      ev_extra3           - extra data field
--      ev_extra4           - extra data field
-- ----------------------------------------------------------------------
create table pgq.event_template (
        ev_id               bigint          not null,
        ev_time             timestamptz     not null,

        ev_txid             bigint          not null default txid_current(),
        ev_owner            int4,
        ev_retry            int4,

        ev_type             text,
        ev_data             text,
        ev_extra1           text,
        ev_extra2           text,
        ev_extra3           text,
        ev_extra4           text
);
```

#### 队列的管理
队列的管理，主要是通过`pgq.queue`表来实现的，该表存储了队列的基本信息，包括队列的id、队列的名称、队列的数据表前缀、队列的事件序列、队列的tick序列等。每当创建一个队列时，都会在`pgq.queue`表中插入一条记录，记录了队列的基本信息。每当删除一个队列时，都会在`pgq.queue`表中删除一条记录。
```sql
-- ----------------------------------------------------------------------
-- Table: pgq.queue
--
--     Information about available queues
--
-- Columns:
--      queue_id                    - queue id for internal usage
--      queue_name                  - queue name visible outside
--      queue_ntables               - how many data tables the queue has 创建队列时创建了多个表，这个字段表示队列有多少个表
--      queue_cur_table             - which data table is currently active 创建队列时创建了多个表，这个字段表示当前活动的表
--      queue_rotation_period       - period for data table rotation 队列的轮换周期
--      queue_switch_step1          - tx when rotation happened 轮换时的事务id
--      queue_switch_step2          - tx after rotation was committed  
--      queue_switch_time           - time when switch happened 队列的轮换时间
--      queue_external_ticker       - ticks come from some external sources 是否来自外部源的tick
--      queue_ticker_paused         - ticker is paused 
--      queue_disable_insert        - disallow pgq.insert_event()
--      queue_ticker_max_count      - batch should not contain more events 触发tick的最小事件数
--      queue_ticker_max_lag        - events should not age more 触发tick的最大延迟
--      queue_ticker_idle_period    - how often to tick when no events happen 触发tick的最大空闲时间
--      queue_per_tx_limit          - Max number of events single TX can insert 
--      queue_data_pfx              - prefix for data table names 数据表的前缀
--      queue_event_seq             - sequence for event id's   
--      queue_tick_seq              - sequence for tick id's    
--      queue_extra_maint           - array of functon names to call during maintenance
-- ----------------------------------------------------------------------
create table pgq.queue (
        queue_id                    serial,
        queue_name                  text        not null,

        queue_ntables               integer     not null default 3,
        queue_cur_table             integer     not null default 0,
        queue_rotation_period       interval    not null default '2 hours',
        queue_switch_step1          bigint      not null default txid_current(),
        queue_switch_step2          bigint               default txid_current(),
        queue_switch_time           timestamptz not null default now(),

        queue_external_ticker       boolean     not null default false,
        queue_disable_insert        boolean     not null default false,
        queue_ticker_paused         boolean     not null default false,

        queue_ticker_max_count      integer     not null default 500,
        queue_ticker_max_lag        interval    not null default '3 seconds',
        queue_ticker_idle_period    interval    not null default '1 minute',
        queue_per_tx_limit          integer,

        queue_data_pfx              text        not null,
        queue_event_seq             text        not null,
        queue_tick_seq              text        not null,

        queue_extra_maint           text[],

        constraint queue_pkey primary key (queue_id),
        constraint queue_name_uq unique (queue_name)
);
```

### 消费者
消费者通过`pgq.register_consumer()`函数来注册，函数参数如下：
```sql
create or replace function pgq.register_consumer(
    x_queue_name text,
    x_consumer_id text)
returns integer as $$
```
实际上就是向表`pgq.subscription`插入一条记录，同时可通过`pgq.unregister_consumer()`函数来注销消费者，也就是删除`pgq.subscription`表中的记录。

可通过表`pgq.consumer`查询当前有哪些消费者，消费者表结构如下：
```sql
create table pgq.consumer (
        co_id       serial,      -- consumer id 消费者ID
        co_name     text        not null,  -- consumer name 消费者名称

        constraint consumer_pkey primary key (co_id),
        constraint consumer_name_uq UNIQUE (co_name)
);
```

#### 订阅
注册于注销消费者，都是通过表`pgq.subscription`来实现的，该表存储了消费者与队列的订阅关系，包括消费者与队列的订阅关系、消费者与队列的订阅关系对应的tick、消费者与队列的订阅关系对应的batch等。
```sql
-- ----------------------------------------------------------------------
-- Table: pgq.subscription
--
--      Consumer registration on a queue.
--
-- Columns:
--
--      sub_id          - subscription id for internal usage
--      sub_queue       - queue id
--      sub_consumer    - consumer's id
--      sub_last_tick   - last tick the consumer processed
--      sub_batch       - shortcut for queue_id/consumer_id/tick_id
--      sub_next_tick   - batch end pos
-- ----------------------------------------------------------------------
create table pgq.subscription (
        sub_id                          serial      not null,
        sub_queue                       int4        not null,
        sub_consumer                    int4        not null,
        sub_last_tick                   bigint,
        sub_active                      timestamptz not null default now(),
        sub_batch                       bigint,
        sub_next_tick                   bigint,

        constraint subscription_pkey primary key (sub_queue, sub_consumer),
        constraint subscription_batch_idx unique (sub_batch),
        constraint sub_queue_fkey foreign key (sub_queue)
                                   references pgq.queue (queue_id),
        constraint sub_consumer_fkey foreign key (sub_consumer)
                                   references pgq.consumer (co_id)
);
```

### 消息事件发布
用户可通过函数`pgq.insert_event`发布消息事件。其实现就是向创建队列时创建的`pgq.event_N_M`表中插入数据。
```sql
create or replace function pgq.insert_event(
    queue_name text, ev_type text, ev_data text,
    ev_extra1 text, ev_extra2 text, ev_extra3 text, ev_extra4 text)
returns bigint as $$
-- ----------------------------------------------------------------------
-- Function: pgq.insert_event(7)
--
--      Insert a event into queue with all the extra fields.
--
-- Parameters:
--      queue_name      - Name of the queue
--      ev_type         - User-specified type for the event
--      ev_data         - User data for the event
--      ev_extra1       - Extra data field for the event
--      ev_extra2       - Extra data field for the event
--      ev_extra3       - Extra data field for the event
--      ev_extra4       - Extra data field for the event
--
-- Returns:
--      Event ID
-- Calls:
--      pgq.insert_event_raw(11)
-- Tables directly manipulated:
--      insert - pgq.insert_event_raw(11), a C function, inserts into current event_N_M table
-- ----------------------------------------------------------------------
begin
    return pgq.insert_event_raw(queue_name, null, now(), null, null,
            ev_type, ev_data, ev_extra1, ev_extra2, ev_extra3, ev_extra4);
end;
$$ language plpgsql security definer;
```
我们可以通过`pgq.current_event_table`函数来查看当前队列的事件表名。然后直接查询该表，发现事件已经插入到事件表中。
```sql
-- 查询当前队列的事件表名
postgres=# select pgq.current_event_table('myqueue');
 current_event_table 
---------------------
 pgq.event_2_0
(1 row)
-- 插入事件存储在表中
postgres=# select * from pgq.event_2_0;
 ev_id |            ev_time            | ev_txid | ev_owner | ev_retry | ev_type  |      ev_data       | ev_extra1 | ev_extra2 | ev_extra3 | ev_extra4 
-------+-------------------------------+---------+----------+----------+----------+--------------------+-----------+-----------+-----------+-----------
     1 | 2025-03-21 17:20:15.532007+08 |     808 |          |          | insert   | insert a tuple     |           |           |           | 
     2 | 2025-03-21 17:20:28.98632+08  |     809 |          |          | update   | update a tuple     |           |           |           | 
     3 | 2025-03-21 17:20:39.131856+08 |     810 |          |          | delete   | delete a tuple     |           |           |           | 
     4 | 2025-03-24 15:00:31.981381+08 |     835 |          |          | hangzhou | hangzhou print log |           |           |           | 
(4 rows)
```

### 批处理
PGQ支持批量读取消息，我们看一下PGQ中批处理是怎么设计的。PGQ中通过`pgq.next_batch`函数获取下一个批次ID，然后通过`pgq.get_batch_events`函数获取该批次ID对应的消息事件。
```sql
-- 获取下一个批次ID
postgres=# select * from pgq.next_batch('myqueue','myconsumer');
 next_batch 
------------
          2
(1 row)
-- 我们看当前队列的事件表， 只有一个事件
postgres=# select * from pgq.event_4;
 ev_id |            ev_time            | ev_txid | ev_owner | ev_retry | ev_type |    ev_data     | ev_extra1 | ev_extra2 | ev_extra3 | ev_extra4 
-------+-------------------------------+---------+----------+----------+---------+----------------+-----------+-----------+-----------+-----------
     1 | 2025-04-02 15:02:13.052352+08 | 3994703 |          |          | insert  | insert a tuple |           |           |           | 
(1 row)
-- 此时插入一个事件
postgres=# select pgq.insert_event('myqueue','update','update a tuple');
 insert_event 
--------------
            2
(1 row)
-- 此时事件表中有两个事件
postgres=# select * from pgq.event_4;
 ev_id |            ev_time            | ev_txid | ev_owner | ev_retry | ev_type |    ev_data     | ev_extra1 | ev_extra2 | ev_extra3 | ev_extra4 
-------+-------------------------------+---------+----------+----------+---------+----------------+-----------+-----------+-----------+-----------
     1 | 2025-04-02 15:02:13.052352+08 | 3994703 |          |          | insert  | insert a tuple |           |           |           | 
     2 | 2025-04-02 16:06:53.584442+08 | 3994708 |          |          | update  | update a tuple |           |           |           | 
(2 rows)
-- 查询当前订阅信息
postgres=# select * from pgq.subscription;
 sub_id | sub_queue | sub_consumer | sub_last_tick |          sub_active           | sub_batch | sub_next_tick 
--------+-----------+--------------+---------------+-------------------------------+-----------+---------------
      3 |         4 |            3 |             1 | 2025-04-02 15:05:28.888255+08 |         2 |             2
(1 row)
-- 查看pgq.tick表，重点看其tick_snapshot字段，该字段表示该批次事件对应的快照
postgres=# select * from pgq.tick;
 tick_queue | tick_id |           tick_time           |  tick_snapshot   | tick_event_seq 
------------+---------+-------------------------------+------------------+----------------
          4 |       1 | 2025-04-02 14:58:24.807584+08 | 3994702:3994702: |              1
          4 |       2 | 2025-04-02 15:04:50.380946+08 | 3994705:3994705: |              1
(2 rows)
-- 获取该批次的事件，因事件2是在获取批次ID之后插入的，所以不会读取到
postgres=# select * from pgq.get_batch_events(2);
 ev_id |            ev_time            | ev_txid | ev_retry | ev_type |    ev_data     | ev_extra1 | ev_extra2 | ev_extra3 | ev_extra4 
-------+-------------------------------+---------+----------+---------+----------------+-----------+-----------+-----------+-----------
     1 | 2025-04-02 15:02:13.052352+08 | 3994703 |          | insert  | insert a tuple |           |           |           | 
(1 row)
-- 完成该批次
postgres=# select pgq.finish_batch(2);
 finish_batch 
--------------
            1
(1 row)
-- 查看订阅信息，sub_batch字段为空
postgres=# select * from pgq.subscription;
 sub_id | sub_queue | sub_consumer | sub_last_tick |          sub_active           | sub_batch | sub_next_tick 
--------+-----------+--------------+---------------+-------------------------------+-----------+---------------
      3 |         4 |            3 |             2 | 2025-04-02 16:23:06.923289+08 |           |              
(1 row)

-- 执行tick
postgres=# select pgq.ticker();
 ticker 
--------
      1
(1 row)
-- 查询tick表
postgres=# select * from pgq.tick ;
 tick_queue | tick_id |           tick_time           |  tick_snapshot   | tick_event_seq 
------------+---------+-------------------------------+------------------+----------------
          4 |       1 | 2025-04-02 14:58:24.807584+08 | 3994702:3994702: |              1
          4 |       2 | 2025-04-02 15:04:50.380946+08 | 3994705:3994705: |              1
          4 |       3 | 2025-04-02 16:26:27.368303+08 | 3994710:3994710: |              2
(3 rows)
-- 此时再次查询该批次的事件，此时会读取到事件2，而不会读取到事件1，因为事件1在获取批次ID之前已经完成
postgres=# select * from pgq.get_batch_events(3);
 ev_id |            ev_time            | ev_txid | ev_retry | ev_type |    ev_data     | ev_extra1 | ev_extra2 | ev_extra3 | ev_extra4 
-------+-------------------------------+---------+----------+---------+----------------+-----------+-----------+-----------+-----------
     2 | 2025-04-02 16:06:53.584442+08 | 3994708 |          | update  | update a tuple |           |           |           | 
(1 row)

```
可以看到PGQ中与PGMQ关于批量发送与批量读取的设计是有所不同的，但是核心原理是一致的，批量读就是从队列表中一次性读取多行数据，只不过在PGMQ中是通过读取前N行（ID最小的前N行）数据，而PGQ中是通过判断该批次的事务区间范围来判断。对比下来，PGMQ的设计更加简洁。

### tick
tick是pgq中的一个概念，它表示一个队列的周期性执行，用于触发队列的事件处理。 tick表结构如下：
```sql
-- ----------------------------------------------------------------------
-- Table: pgq.tick
--
--      Snapshots for event batching
--
-- Columns:
--      tick_queue      - queue id whose tick it is
--      tick_id         - ticks id (per-queue)
--      tick_time       - time when tick happened
--      tick_snapshot   - transaction state
--      tick_event_seq  - last value for event seq
-- ----------------------------------------------------------------------
create table pgq.tick (
        tick_queue                  int4            not null,
        tick_id                     bigint          not null,
        tick_time                   timestamptz     not null default now(),
        tick_snapshot               txid_snapshot   not null default txid_current_snapshot(),
        tick_event_seq              bigint          not null, -- may be NULL on upgraded dbs

        constraint tick_pkey primary key (tick_queue, tick_id),
        constraint tick_queue_fkey foreign key (tick_queue)
                                   references pgq.queue (queue_id)
);
```
由ticker决定批次。更多可参考[skytools](https://github.com/markokr/skytools)。

### 消息重试

消息重试是指当消息处理失败时，将消息重新放入队列中进行重试。其原理是将消息插入到`pgq.retry_queue`表中，当重试时间到达时，将消息重新插入到主队列中。

```sql
-- ----------------------------------------------------------------------
-- Table: pgq.retry_queue
--
--      Events to be retried.  When retry time reaches, they will
--      be put back into main queue.
--
-- Columns:
--      ev_retry_after          - time when it should be re-inserted to main queue
--      ev_queue                - queue id, used to speed up event copy into queue
--      *                       - same as pgq.event_template
-- ----------------------------------------------------------------------
create table pgq.retry_queue (
    ev_retry_after          timestamptz     not null,
    ev_queue                int4            not null,

    like pgq.event_template,

    constraint rq_pkey primary key (ev_owner, ev_id),
    constraint rq_queue_id_fkey foreign key (ev_queue)
                             references pgq.queue (queue_id)
);
```
我们执行消息重试，发现消息已经被插入到`pgq.retry_queue`表中。
```sql
postgres=# select pgq.batch_retry(2,120);
-[ RECORD 1 ]--
batch_retry | 1

postgres=# select * from pgq.retry_queue ;
-[ RECORD 1 ]--+------------------------------
ev_retry_after | 2025-04-02 15:16:29.850156+08
ev_queue       | 4
ev_id          | 1
ev_time        | 2025-04-02 15:02:13.052352+08
ev_txid        | 
ev_owner       | 3
ev_retry       | 1
ev_type        | insert
ev_data        | insert a tuple
ev_extra1      | 
ev_extra2      | 
ev_extra3      | 
ev_extra4      | 
```