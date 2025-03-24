## PostgreSQL消息队列拓展——PGQ
在分布式系统和异步任务处理中，消息队列是解耦服务、提升可靠性的核心组件。但当你的系统已经依赖PostgreSQL数据库时，引入RabbitMQ等独立消息中间件可能意味着更高的复杂性和运维成本。能否利用PostgreSQL数据库来构建消息队列，以减少运维成本呢？

PGMQ和PGQ就是PostgreSQL的消息队列拓展。[PGMQ](https://mp.weixin.qq.com/s/MJhS5KSLyUxlzhbqMvebiA)我们之前的文章中已有介绍，这里介绍另一款消息队列扩展PGQ。PGQ是一个通用消息队列拓展，可以解决实时事务的异步处理问题，比如在事务提交后，异步执行一些任务，比如发送邮件、更新缓存等。其核心是不会阻塞当前事务。实际上就是提供了类似消息队列的机制，提供纯SQL接口供开发者调用。

### PGQ解决了哪些问题？

**异步批量事务处理**：在事务提交后，异步执行一些任务，比如发送邮件、更新缓存等。这些任务通常不会阻塞当前事务，但需要确保这些任务在事务提交后执行。具体一点就是当你执行某些比如说`INSERT/DELETE/UPDATE`事务，你希望触发一些行为，但是这些行为不需要在事务`COMMIT`之前完成，希望在事务`COMMIT`之后不远的时间异步执行，而不阻塞当前的事务。

### 基本原理
数据库实现消息队列，并不是像RabbitMQ一样，实现了AMQP协议，而是通过数据库机制来实现类似消息队列的功能。对外提供的接口是SQL接口。 数据库实现消息队列，通常都是通过表来实现队列，PGQ也是基于表来实现消息队列。一个队列就是一张表，客户端发送消息到队列，实际上就是向表中插入了一条数据。消费者读消息，就是读这张抽象为队列的表，消费消息实际上就是从被抽象为队列的表中删除消息。为了方便用户使用，封装了一些API来操作队列，与RabbitMQ不同的是，这里提供的是纯SQL接口，用户可以直接使用SQL来操作队列，比如发送消息、消费消息等。

### 安装

安装扩展十分简单：
```sql
-- 创建 PGQ 扩展
CREATE EXTENSION pgq;
```
可通过函数`pgq.version()`查看当前pgq的版本。
```sql
postgres=# select pgq.version();
 version 
---------
 3.5.1
(1 row)
```

> 支持PG10~PG16

### PGQ的使用

#### 创建队列
使用消息队列，首先要创建一个消息队列，然后才能向队列中发送消息。
```sql
-- 创建队列
postgres=# select pgq.create_queue('myqueue');
 create_queue 
--------------
            1
(1 row)
```
为了查看队列的情况，PGQ中创建了表`pgq.queue`，用来存储已创建成功的队列信息。
```sql
postgres=# select * from pgq.queue;
-[ RECORD 1 ]------------+------------------------------
queue_id                 | 1
queue_name               | my_queue
queue_ntables            | 3
queue_cur_table          | 0
queue_rotation_period    | 02:00:00
queue_switch_step1       | 800
queue_switch_step2       | 800
queue_switch_time        | 2025-03-20 13:58:20.872887+08
queue_external_ticker    | f
queue_disable_insert     | f
queue_ticker_paused      | f
queue_ticker_max_count   | 500
queue_ticker_max_lag     | 00:00:03
queue_ticker_idle_period | 00:01:00
queue_per_tx_limit       | 
queue_data_pfx           | pgq.event_1
queue_event_seq          | pgq.event_1_id_seq
queue_tick_seq           | pgq.event_1_tick_seq
queue_extra_maint        | 
-[ RECORD 2 ]------------+------------------------------
queue_id                 | 2
queue_name               | myqueue
queue_ntables            | 3
queue_cur_table          | 0
queue_rotation_period    | 02:00:00
queue_switch_step1       | 807
queue_switch_step2       | 807
queue_switch_time        | 2025-03-21 17:19:50.565652+08
queue_external_ticker    | f
queue_disable_insert     | f
queue_ticker_paused      | f
queue_ticker_max_count   | 2
queue_ticker_max_lag     | 00:00:03
queue_ticker_idle_period | 00:01:00
queue_per_tx_limit       | 
queue_data_pfx           | pgq.event_2
queue_event_seq          | pgq.event_2_id_seq
queue_tick_seq           | pgq.event_2_tick_seq
queue_extra_maint        |        
```
也可以通过函数`pgq.get_queue_info`查看队列信息。
```sql
postgres=# select * from pgq.get_queue_info();
-[ RECORD 1 ]------------+------------------------------
queue_name               | myqueue
queue_ntables            | 3
queue_cur_table          | 0
queue_rotation_period    | 02:00:00
queue_switch_time        | 2025-03-21 17:19:50.565652+08
queue_external_ticker    | f
queue_ticker_paused      | f
queue_ticker_max_count   | 2
queue_ticker_max_lag     | 00:00:03
queue_ticker_idle_period | 00:01:00
ticker_lag               | 2 days 22:09:55.306126
ev_per_sec               | 
ev_new                   | 3
last_tick_id             | 1
-[ RECORD 2 ]------------+------------------------------
queue_name               | testqueue
queue_ntables            | 3
queue_cur_table          | 0
queue_rotation_period    | 02:00:00
queue_switch_time        | 2025-03-24 14:43:01.80398+08
queue_external_ticker    | f
queue_ticker_paused      | f
queue_ticker_max_count   | 500
queue_ticker_max_lag     | 00:00:03
queue_ticker_idle_period | 00:01:00
ticker_lag               | 00:46:44.067798
ev_per_sec               | 
ev_new                   | 0
last_tick_id             | 1
```

#### 注册消费者
可通过函数`pgq.register_consumer`注册消费者，消费者可以消费队列中的消息。
```sql
-- 注册消费者
postgres=# select pgq.register_consumer('myqueue','myconsumer');
 register_consumer 
-------------------
                 1
(1 row)
```
PGQ中创建了表`pgq.consumer`，用来存储已注册成功的消费者信息。
```sql
-- 查看消费者，
postgres=# select * from pgq.consumer;
 co_id |   co_name    
-------+--------------
     1 | myconsumer
     2 | testconsumer
(2 rows)
```
也可通过函数`pgq.get_consumer_info`查看消费者信息。
```sql
postgres=# select * from pgq.get_consumer_info();
 queue_name | consumer_name |          lag           |       last_seen        | last_tick | current_batch | next_tick | pending_events 
------------+---------------+------------------------+------------------------+-----------+---------------+-----------+----------------
 myqueue    | myconsumer    | 2 days 22:11:53.125704 | 2 days 22:10:33.319906 |         1 |               |           |              0
(1 row)
```


#### 发送消息
创建完队列后就可以发送消息了，PGQ可通过`pgq.insert_event`函数向队列中发送消息。
```sql  
postgres=# select pgq.insert_event('myqueue','hangzhou','hangzhou print log');
 insert_event 
--------------
            4
(1 row)
```
我们看一下其声明参数，需要传递三个参数，分别是队列名称、事件类型和事件数据。后面的参数是可选的。
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

-- 简化版
create or replace function pgq.insert_event(queue_name text, ev_type text, ev_data text)
returns bigint as $$
-- ----------------------------------------------------------------------
-- Function: pgq.insert_event(3)
--
--      Insert a event into queue.
--
-- Parameters:
--      queue_name      - Name of the queue
--      ev_type         - User-specified type for the event
--      ev_data         - User data for the event
--
-- Returns:
--      Event ID
-- Calls:
--      pgq.insert_event(7)
-- ----------------------------------------------------------------------
begin
    return pgq.insert_event(queue_name, ev_type, ev_data, null, null, null, null);
end;
$$ language plpgsql;
```

我们可以通过函数`pgq.current_event_table`来查看当前队列的表名。
```sql
-- 查看当前队列存储消息事件的表名
postgres=# select pgq.current_event_table('myqueue');
 current_event_table 
---------------------
 pgq.event_2_0
(1 row)
-- 查看表结构
postgres=# \d pgq.event_2_0;
                                         Table "pgq.event_2_0"
  Column   |           Type           | Collation | Nullable |                 Default                 
-----------+--------------------------+-----------+----------+-----------------------------------------
 ev_id     | bigint                   |           | not null | nextval('pgq.event_2_id_seq'::regclass)
 ev_time   | timestamp with time zone |           | not null | 
 ev_txid   | bigint                   |           | not null | txid_current()
 ev_owner  | integer                  |           |          | 
 ev_retry  | integer                  |           |          | 
 ev_type   | text                     |           |          | 
 ev_data   | text                     |           |          | 
 ev_extra1 | text                     |           |          | 
 ev_extra2 | text                     |           |          | 
 ev_extra3 | text                     |           |          | 
 ev_extra4 | text                     |           |          | 
Indexes:
    "event_2_0_txid_idx" btree (ev_txid)
Inherits: pgq.event_2

-- 查看具体的表，可以看到我们插入的消息已经存储到了该表中。
postgres=# select * from pgq.event_2_0;
 ev_id |            ev_time            | ev_txid | ev_owner | ev_retry | ev_type  |      ev_data       | ev_extra1 | ev_extra2 | ev_extra3 | ev_extra4 
-------+-------------------------------+---------+----------+----------+----------+--------------------+-----------+-----------+-----------+-----------
     1 | 2025-03-21 17:20:15.532007+08 |     808 |          |          | insert   | insert a tuple     |           |           |           | 
     2 | 2025-03-21 17:20:28.98632+08  |     809 |          |          | update   | update a tuple     |           |           |           | 
     3 | 2025-03-21 17:20:39.131856+08 |     810 |          |          | delete   | delete a tuple     |           |           |           | 
     4 | 2025-03-24 15:00:31.981381+08 |     835 |          |          | hangzhou | hangzhou print log |           |           |           | 
(4 rows)
```
也就是说，创建一个队列，其实就是创建了一些列的表，用来存储消息，以及存储一些元数据。而发送消息，实际上就是向这些表插入数据。

#### Ticker
[ticker](https://github.com/markokr/skytools/tree/master/sql/ticker)是PGQ消息处理的核心调度组件，一个轻量级的PostgreSQL任务队列工具，专门用于处理周期性任务和延时任务，负责周期性地将队列中的消息事件分批次推送给消费者。

深入可参考[pgq ticker](https://github.com/markokr/skytools/tree/master/sql/ticker)。

如果没有配置外部ticker，可通过SQL函数`pgq.ticker`直接调用ticker。
```sql
postgres=# select pgq.ticker();
 ticker 
--------
      0
(1 row)
```
> Notice: 如果没有配置外部ticker，或者手动执行`pgq.ticker()`，消费者读取不到消息。

#### 消费消息

消费消息，可先通过函数`pgq.next_batch`获取队列下一批次消息ID，再通过函数`pgq.get_batch_events`获取批次消息。最后通过函数`pgq.finish_batch`结束批次消息。
```sql
-- 获取下一批次消息ID
postgres=# select pgq.next_batch('myqueue','myconsumer');
 next_batch 
------------
          1
(1 row)
-- 获取批次消息
postgres=# select * from pgq.get_batch_events(1);
 ev_id |            ev_time            | ev_txid | ev_retry | ev_type  |      ev_data       | ev_extra1 | ev_extra2 | ev_extra3 | ev_extra4 
-------+-------------------------------+---------+----------+----------+--------------------+-----------+-----------+-----------+-----------
     1 | 2025-03-21 17:20:15.532007+08 |     808 |          | insert   | insert a tuple     |           |           |           | 
     2 | 2025-03-21 17:20:28.98632+08  |     809 |          | update   | update a tuple     |           |           |           | 
     3 | 2025-03-21 17:20:39.131856+08 |     810 |          | delete   | delete a tuple     |           |           |           | 
     4 | 2025-03-24 15:00:31.981381+08 |     835 |          | hangzhou | hangzhou print log |           |           |           | 
(4 rows)
-- 结束批次消息
postgres=# select pgq.finish_batch(1);
 finish_batch 
--------------
            1
(1 row)
```

#### 注销消费者
可通过函数`pgq.unregister_consumer`注销消费者，注销消费者后，消费者将不再消费队列中的消息。
```sql  
postgres=# select pgq.unregister_consumer('myqueue','myconsumer');
 unregister_consumer 
---------------------
                   1
(1 row)
-- 查看，myconsumer消费者已被删除
postgres=# select * from pgq.consumer;
 co_id |  co_name   
-------+------------
     1 | testconsumer
(1 row)

```

#### 删除队列
最后，删除队列，可通过函数`pgq.drop_queue`删除队列。
```sql
-- 删除队列
postgres=# select pgq.drop_queue('myqueue');
 drop_queue 
------------
 1
(1 row)
```

---
### 参考资料
[PGQ Tutorial](https://wiki.postgresql.org/wiki/PGQ_Tutorial)       
[pgq SQL接口说明](https://github.com/markokr/skytools/blob/master/doc/pgq-sql.txt)
[The PgQueue module](https://pypi.org/project/pgqueue/)
[PGQ: Queuing for Long-Running Jobs in Go Written Atop Postgres](https://thenewstack.io/pgq-queuing-for-long-running-jobs-in-go-written-atop-postgres/)
[The ticker daemon](https://wiki.postgresql.org/wiki/Londiste_Tutorial_(Skytools_2)#The_ticker_daemon)