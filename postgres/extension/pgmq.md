

## PostgreSQL轻量级消息队列PGMQ
在分布式系统中，消息队列通常用于实现消息的异步传递，提高系统的性能和可用性。常用的消息队列有RabbitMQ、RocketMQ、Kafka、ActiveMQ等。除了以上，PostgreSQL也可通过扩展支持消息队列，PGMQ就是是一个轻量级的PostgreSQL消息队列扩展，具有以下特性：
- 轻量级： 无需后台进程以及其他依赖，仅通过PostgreSQL函数以扩展形式实现
- 保证消息`exactly once`：在设置的可见性超时内，确保消息仅被消费者消费一次，避免重复或者丢失。
- 与AWS SQS和RSMQ的API兼容：提供与AWS SQS和RSMQ的API兼容的函数，方便迁移和集成。
- 消息持久化：消息默认保存在队列中（数据库表），直至被删除。
- 消息存档：可选择将消息存档，用于长期保留。


### PGMQ的使用

#### 安装pgmq扩展
支持PostgreSQL 14-17版本，安装方法如下：
```sql
postgres=# create extension pgmq;
CREATE EXTENSION
```

#### 创建队列
可通过`pgmq.create_queue`函数创建队列，每个队列都是一个数据库表，表名是队列名加上前缀'q_'。例如，创建一个名为'my_queue'的队列，则对应的表名为'q_my_queue'。
```sql
postgres=# select pgmq.create_queue('my_queue');
 pgmq.create_queue 
------------------
 
(1 row)

-- 查看队列对应的数据库表
postgres=# \d pgmq.q_my_queue;
                                   Table "pgmq.q_my_queue"
   Column    |           Type           | Collation | Nullable |           Default            
-------------+--------------------------+-----------+----------+------------------------------
 msg_id      | bigint                   |           | not null | generated always as identity
 read_ct     | integer                  |           | not null | 0
 enqueued_at | timestamp with time zone |           | not null | now()
 vt          | timestamp with time zone |           | not null | 
 message     | jsonb                    |           |          | 
 headers     | jsonb                    |           |          | 
Indexes:
    "q_my_queue_pkey" PRIMARY KEY, btree (msg_id)
    "q_my_queue_vt_idx" btree (vt)

```

#### 发送消息
可通过`pgmq.send_message`函数发送消息，消息会保存在队列中，并返回消息ID。例如，向'my_queue'队列发送一条消息'hello world'。
```sql
postgres=# select pgmq.send_message('my_queue', 'hello world');
 pgmq.send_message 
------------------
 
(1 row)

-- messages are sent as JSON
postgres=# SELECT * from pgmq.send(
  queue_name  => 'my_queue',
  msg         => '{"foo": "bar1"}'
);
 send 
------
    1
(1 row)

-- Optionally provide a delay
-- this message will be on the queue but unable to be consumed for 5 seconds
postgres=# SELECT * from pgmq.send(
  queue_name => 'my_queue',
  msg        => '{"foo": "bar2"}',
  delay      => 5
);
 send 
------
    2
(1 row)
```

#### 读消息
可通过`pgmq.read`函数读取消息，默认读取队列中的第一个消息，并返回消息ID、读取次数、入队时间、可见时间、消息内容、消息头。
```sql
-- 读2条消息，可见性超时30秒，返回2条消息，如果消息在30秒内没有被删除或者归档，则会重新可见，可被其他消费者读取
postgres=# SELECT * FROM pgmq.read(
  queue_name => 'my_queue',
  vt         => 30,
  qty        => 2
);
 msg_id | read_ct |          enqueued_at          |              vt               |     message     | headers 
--------+---------+-------------------------------+-------------------------------+-----------------+---------
      1 |       1 | 2025-03-21 17:50:54.221263+08 | 2025-03-21 17:53:38.303743+08 | {"foo": "bar1"} | 
      2 |       1 | 2025-03-21 17:51:25.216303+08 | 2025-03-21 17:53:38.303831+08 | {"foo": "bar2"} | 
(2 rows)

-- 30秒内读取，发现消息不可见，返回0条消息
postgres=# SELECT * FROM pgmq.read(
  queue_name => 'my_queue',
  vt         => 30,
  qty        => 2
);
 msg_id | read_ct | enqueued_at | vt | message | headers 
--------+---------+-------------+----+---------+---------
(0 rows)
```

#### 消费消息
可通过`pgmq.pop`函数读取消息后删除。
```sql
-- 弹出my_queue队列中的第一条消息
postgres=# SELECT * FROM pgmq.pop('my_queue');
 msg_id | read_ct |          enqueued_at          |              vt               |     message     | headers 
--------+---------+-------------------------------+-------------------------------+-----------------+---------
      1 |       3 | 2025-03-21 17:50:54.221263+08 | 2025-03-21 20:29:00.951687+08 | {"foo": "bar1"} | 
(1 row)

-- 消费消息后，消息从队列中删除，此时再次查询，可以看到刚刚消费的消息已经不存在了。
postgres=# SELECT * FROM pgmq.read(           
  queue_name => 'my_queue',
  vt         => 30,
  qty        => 2
);
 msg_id | read_ct |          enqueued_at          |              vt               |     message     | headers 
--------+---------+-------------------------------+-------------------------------+-----------------+---------
      2 |       4 | 2025-03-21 17:51:25.216303+08 | 2025-03-21 20:31:55.160707+08 | {"foo": "bar2"} | 
(1 row)
```

#### 归档消息
可通过`pgmq.archive`函数归档消息，即将消息从队列中删除，并保存到归档表中。归档表名为前缀`a_`加上队列名。
```sql
-- Archive message with msg_id=2
postgres=# SELECT pgmq.archive(
  queue_name => 'my_queue',
  msg_id     => 2
);
 archive 
---------
 t
(1 row)
```
也可同时归档多条消息，即`pgmq.archive(queue_name text, msg_ids bigint[])`。
```sql
-- 批量发送消息
postgres=# SELECT pgmq.send_batch(
  queue_name => 'my_queue',
  msgs       => ARRAY['{"foo": "bar3"}','{"foo": "bar4"}','{"foo": "bar5"}']::jsonb[]
);
 send_batch 
------------
          3
          4
          5
(3 rows)

-- 归档3条消息
postgres=# SELECT pgmq.archive(
  queue_name => 'my_queue',
  msg_ids    => ARRAY[3, 4, 5]
);
 archive 
---------
       3
       4
       5
(3 rows)

-- 查询归档表，可以看到我们归档的4条消息
postgres=# SELECT * FROM pgmq.a_my_queue;
 msg_id | read_ct |          enqueued_at          |          archived_at          |              vt               |     message     | headers 
--------+---------+-------------------------------+-------------------------------+-------------------------------+-----------------+---------
      2 |       4 | 2025-03-21 17:51:25.216303+08 | 2025-03-21 21:24:46.696399+08 | 2025-03-21 20:31:55.160707+08 | {"foo": "bar2"} | 
      3 |       0 | 2025-03-21 21:27:30.405284+08 | 2025-03-21 21:27:56.719586+08 | 2025-03-21 21:27:30.407251+08 | {"foo": "bar3"} | 
      4 |       0 | 2025-03-21 21:27:30.405284+08 | 2025-03-21 21:27:56.719586+08 | 2025-03-21 21:27:30.407251+08 | {"foo": "bar4"} | 
      5 |       0 | 2025-03-21 21:27:30.405284+08 | 2025-03-21 21:27:56.719586+08 | 2025-03-21 21:27:30.407251+08 | {"foo": "bar5"} | 
(4 rows)

-- 再次查询，可以看到刚刚归档的消息已经不存在了
postgres=# SELECT * FROM pgmq.read(
  queue_name => 'my_queue',
  vt         => 30,                                                          
  qty        => 2
);
 msg_id | read_ct | enqueued_at | vt | message | headers 
--------+---------+-------------+----+---------+---------
(0 rows)
```

#### 删除消息
可通过`pgmq.delete`函数删除消息，即从队列中删除消息。
```sql
-- 发送1条消息
postgres=# SELECT pgmq.send('my_queue', '{"foo": "bar6"}');
 send 
------
    6
(1 row)
-- 可以读到刚刚发送的消息
postgres=# SELECT * FROM pgmq.read(
  queue_name => 'my_queue',
  vt         => 30,
  qty        => 2
);
 msg_id | read_ct |          enqueued_at          |              vt               |     message     | headers 
--------+---------+-------------------------------+-------------------------------+-----------------+---------
      6 |       1 | 2025-03-21 21:31:49.375243+08 | 2025-03-21 21:32:23.991975+08 | {"foo": "bar6"} | 
(1 row)
-- 删除消息
postgres=# SELECT pgmq.delete('my_queue', 6);
 delete 
--------
 t
(1 row)
-- 再次查询，可以看到刚刚删除的消息已经不存在了
postgres=# SELECT * FROM pgmq.read(
  queue_name => 'my_queue',
  vt         => 30,
  qty        => 2
);
 msg_id | read_ct | enqueued_at | vt | message | headers 
--------+---------+-------------+----+---------+---------
(0 rows)
```

#### 删除队列
可通过`pgmq.drop`函数删除队列，即删除队列中的所有消息，并删除归档表。
```sql
postgres=# SELECT pgmq.drop_queue('my_queue');
 drop_queue 
------------
 t
(1 row)

-- 此时再次读队列，会报错，队列表不存在
postgres=# SELECT * FROM pgmq.read(
  queue_name => 'my_queue',
  vt         => 30,
  qty        => 2
);
ERROR:  relation "pgmq.q_my_queue" does not exist
LINE 5:             FROM pgmq.q_my_queue
                         ^
QUERY:  
        WITH cte AS
        (
            SELECT msg_id
            FROM pgmq.q_my_queue
            WHERE vt <= clock_timestamp() AND CASE
                WHEN '{}' != '{}'::jsonb THEN (message @> '{}')::integer
                ELSE 1
            END = 1
            ORDER BY msg_id ASC
            LIMIT $1
            FOR UPDATE SKIP LOCKED
        )
        UPDATE pgmq.q_my_queue m
        SET
            vt = clock_timestamp() + '00:00:30',
            read_ct = read_ct + 1
        FROM cte
        WHERE m.msg_id = cte.msg_id
        RETURNING m.msg_id, m.read_ct, m.enqueued_at, m.vt, m.message, m.headers;
        
CONTEXT:  PL/pgSQL function pgmq.read(text,integer,integer,jsonb) line 30 at RETURN QUERY

-- 归档表也删除了
postgres=# SELECT * FROM pgmq.a_my_queue;
ERROR:  relation "pgmq.a_my_queue" does not exist
LINE 1: SELECT * FROM pgmq.a_my_queue;
                      ^
```

#### Visibility Timeout
PGMQ通过可见性超时确保在特定时间窗口内，每条消息仅被消费者成功处理一次。当消费者通过pgmq.read函数读取消息时，需指定一个可见性超时时间，在次时间段内，该消息对其他消费者不可见，避免被重复消费。若消费者未在可见性超时内删除消息，或归档消息，则消息将重新变为可见状态，可被其他消费者读取。这可用于处理消费者崩溃或者超时的场景，防止消息丢失。

#### 其他API
除了上面常用的函数外，PGMQ还提供了一些其他的函数，可参考[PGMQ API文档](https://tembo.io/pgmq/api/sql/functions/)，这里不再一一赘述。

### PGMQ or RabbitMQ
提到消息队列，大家首先想到的可能是RabbitMQ，而PGMQ则是一个基于PostgreSQL的消息队列系统。下面是PGMQ和RabbitMQ的一些比较。以供技术方案选型时参考。

|特性 |PGMQ|RabbitMQ|
|--|--|--|
|架构与实现|PGMQ是构建在PostgreSQL数据库之上的消息队列系统，无需额外部署独立的消息队列服务|Erlang实现，独立的消息队列中间件，采用AMQP协议，单独部署和管理，可集群化以实现高可用和扩展性|
|性能|性能受限于数据库性能，适合小规模的消息处理场景，高并发、大规模消息处理的场景下受限于PostgreSQL的写入性能 | 高并发、大规模消息处理的场景下，性能表现良好，延迟较低|
|可靠性| 继承PostgreSQL的可靠性，支持事务处理，可以保证消息的原子性和一致性。依赖数据库备份和恢复机制| 提供多种可靠性机制，如消息确认、持久化存储、支持集群镜像队列等，支持集群自动故障转移|
|协议支持|仅支持SQL接口| 支持多种消息协议，如AMQP、MQTT、STOMP等|
|消息持久化| 依赖PostgreSQL的持久化机制（消息实际上是存储在表中）| 支持磁盘持久化|
|消息路由| 简单队列模型、无复杂路由规则| 支持灵活的路由规则，如fanout、direct、topic等|
|消息确认机制| 支持visibility timeout、显式删除| 支持ACK/NACK机制|


综合考虑，需要结合业务场景和需求，选择合适的消息队列系统。简单来说，如果业务不是很复杂，业务量不大，且不需要高并发、大规模消息处理的场景，那么pgmq也是一个不错的选择，能够简化技术栈，避免引入独立的消息中间件，减少运维成本，实现降本增效。简单来说就是很多情况下，没有必要非要引入重量级的消息中间件，完全可以用更轻量化的方案实现。



---

参考文档：
[PGMQ](https://tembo.io/pgmq/)    
[PGMQ API文档](https://tembo.io/pgmq/api/sql/functions/)
[RabbitMQ One broker to queue them all](https://www.rabbitmq.com/)
[RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials)