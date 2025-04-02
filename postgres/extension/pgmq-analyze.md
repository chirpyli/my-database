## PostgreSQL轻量级消息队列PGMQ实现原理分析
在上面篇文章中[PostgreSQL轻量级消息队列PGMQ](../pgmq.md)中，我们介绍了PGMQ的简单使用，本文将分析PGMQ的实现原理。


### 数据库实现消息队列的初步原理

数据库实现消息队列，并不是像RabbitMQ一样，实现了AMQP协议，而是通过数据库机制来实现类似消息队列的功能。对外提供的接口是SQL接口。 <u>数据库实现消息队列，是通过表来实现队列。</u> 一个队列就是一张表，客户端发送消息到队列，实际上就是向表中插入了一条数据。消费者读消息，就是读这张抽象为队列的表，消费消息实际上就是从被抽象为队列的表中删除消息。为了方便用户使用，除了提供SQL接口的方式，还封装了一些API来操作队列，例如[Rust库pgmq](https://crates.org.cn/crates/pgmq)，[Python库tembo-pgmq-python](https://tembo.io/blog/pgmq-with-python)等。


### 消息定义
一条消息，我们需要定义一个格式来存储，在PGMQ中消息类型定义如下：
```sql
-- This type has the shape of a message in a queue, and is often returned by
-- pgmq functions that return messages
CREATE TYPE pgmq.message_record AS (
    msg_id BIGINT,
    read_ct INTEGER,
    enqueued_at TIMESTAMP WITH TIME ZONE,
    vt TIMESTAMP WITH TIME ZONE,
    message JSONB,
    headers JSONB
);
```
在PGMQ中消息按如下格式在队列中存储。也即在表中的存储：

| Attribute Name   | Type       | Description                |
| :---             |    :----   |                       :--- |
| msg_id           | bigint     | Unique ID of the message   |
| read_ct          | bigint     | Number of times the message has been read. Increments on read().   |
| enqueued_at           |  timestamp with time zone     | time that the message was inserted into the queue   |
| vt           | timestamp with time zone      | Timestamp when the message will become available for consumers to read   |
| message           | jsonb      | The message payload   |

例如:
```sql
 msg_id | read_ct |          enqueued_at          |              vt               |      message       
--------+---------+-------------------------------+-------------------------------+--------------------
      1 |       1 | 2023-10-28 19:06:19.941509-05 | 2023-10-28 19:06:27.419392-05 | {"hello": "world"}
```

我们可以查看创建的队列表结构如下：
```sql
postgres=# \d pgmq.q_myqueue;
                                    Table "pgmq.q_myqueue"
   Column    |           Type           | Collation | Nullable |           Default            
-------------+--------------------------+-----------+----------+------------------------------
 msg_id      | bigint                   |           | not null | generated always as identity -- 消息ID
 read_ct     | integer                  |           | not null | 0                       -- 读取次数
 enqueued_at | timestamp with time zone |           | not null | now()                     -- 入队时间
 vt          | timestamp with time zone |           | not null |                           -- 有效时间
 message     | jsonb                    |           |          |                           -- 消息内容
 headers     | jsonb                    |           |          |                           -- 消息头
Indexes:
    "q_myqueue_pkey" PRIMARY KEY, btree (msg_id)                  -- 主键索引
    "q_myqueue_vt_idx" btree (vt)                                 -- 有效时间索引
```

### 队列的实现

我们分析一下队列的实现，在PGMQ中，队列是通过表来实现的，每个队列都是一个数据库表，表名是队列名加上前缀`q_`。例如，创建一个名为`my_queue`的队列，则会创建对应队列表名为`q_my_queue`，同时会创建归档表名为`a_my_queue`，以及向表`pgmq.meta`中插入一条元数据。实现代码如下：
```sql
-- 定义创建队列的函数接口pgmq.create
CREATE FUNCTION pgmq.create(queue_name TEXT)
RETURNS void AS $$
BEGIN
    PERFORM pgmq.create_non_partitioned(queue_name);
END;
$$ LANGUAGE plpgsql;
-- 创建非分区队列
CREATE FUNCTION pgmq.create_non_partitioned(queue_name TEXT)
RETURNS void AS $$
DECLARE
  qtable TEXT := pgmq.format_table_name(queue_name, 'q');
  qtable_seq TEXT := qtable || '_msg_id_seq';
  atable TEXT := pgmq.format_table_name(queue_name, 'a');
BEGIN
  PERFORM pgmq.validate_queue_name(queue_name);

  EXECUTE FORMAT(
    $QUERY$
    CREATE TABLE IF NOT EXISTS pgmq.%I (
        msg_id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
        read_ct INT DEFAULT 0 NOT NULL,
        enqueued_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
        vt TIMESTAMP WITH TIME ZONE NOT NULL,
        message JSONB,
        headers JSONB
    ) -- 创建队列表，表名是队列名加上前缀'q_'
    $QUERY$,
    qtable
  );

  EXECUTE FORMAT(
    $QUERY$
    CREATE TABLE IF NOT EXISTS pgmq.%I (
      msg_id BIGINT PRIMARY KEY,
      read_ct INT DEFAULT 0 NOT NULL,
      enqueued_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
      archived_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
      vt TIMESTAMP WITH TIME ZONE NOT NULL,
      message JSONB,
      headers JSONB
    ); -- 创建归档表，表名是队列名加上前缀'a_'
    $QUERY$,
    atable
  );

  IF NOT pgmq._belongs_to_pgmq(qtable) THEN
      EXECUTE FORMAT('ALTER EXTENSION pgmq ADD TABLE pgmq.%I', qtable);
      EXECUTE FORMAT('ALTER EXTENSION pgmq ADD SEQUENCE pgmq.%I', qtable_seq);
  END IF;

  IF NOT pgmq._belongs_to_pgmq(atable) THEN
      EXECUTE FORMAT('ALTER EXTENSION pgmq ADD TABLE pgmq.%I', atable);
  END IF;

  EXECUTE FORMAT(
    $QUERY$
    CREATE INDEX IF NOT EXISTS %I ON pgmq.%I (vt ASC);
    $QUERY$,
    qtable || '_vt_idx', qtable
  ); -- 队列表创建索引

  EXECUTE FORMAT(
    $QUERY$
    CREATE INDEX IF NOT EXISTS %I ON pgmq.%I (archived_at);
    $QUERY$,
    'archived_at_idx_' || queue_name, atable
  ); -- 归档表创建索引

  EXECUTE FORMAT(
    $QUERY$
    INSERT INTO pgmq.meta (queue_name, is_partitioned, is_unlogged)
    VALUES (%L, false, false)
    ON CONFLICT
    DO NOTHING;
    $QUERY$,
    queue_name
  );  -- 创建元数据，向表meta中插入一条记录，记录当前队列的信息。
END;
$$ LANGUAGE plpgsql;
```

除了需要创建队列对应的表，还需要对元数据做管理，比如记录当前有哪些队列？队列是否分区？是否是unlogged表？等等。PGMQ中，通过一张元数据表`pgmq.meta`来记录这些信息。
```sql
-- Table where queues and metadata about them is stored
CREATE TABLE pgmq.meta (
    queue_name VARCHAR UNIQUE NOT NULL,
    is_partitioned BOOLEAN NOT NULL,
    is_unlogged BOOLEAN NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL
);
```
我们可以创建一个队列，查看一下对应的元数据信息：
```sql
-- 查看队列信息
postgres=# select * from pgmq.list_queues();
 queue_name | is_partitioned | is_unlogged | created_at 
------------+----------------+-------------+------------
(0 rows)
-- 创建一个队列
postgres=# select pgmq.create('test_queue');
 create 
--------
 
(1 row)
-- 查看队列信息
postgres=# select * from pgmq.list_queues();
 queue_name | is_partitioned | is_unlogged |          created_at           
------------+----------------+-------------+-------------------------------
 test_queue | f              | f           | 2025-03-26 16:54:40.242321+08
(1 row)
-- 也可以直接通过pgmq.meta查看队列信息
postgres=# select * from pgmq.meta;
 queue_name | is_partitioned | is_unlogged |          created_at           
------------+----------------+-------------+-------------------------------
 test_queue | f              | f           | 2025-03-26 16:54:40.242321+08
(1 row)
```

### 队列中怎么保证消息的顺序呢？ 
<u>PGMQ中，可以保证消息的顺序性，但不支持消息优先级。</u> 通过消息ID（msg_id字段）来保证消息的顺序，消息ID是自增的（`generated always as identity`），所以消息ID越小，消息越早到达。发送消息的时候，向表中插入数据，消息ID是自增的，所以消息ID越小，消息越早到达，消息ID越大，消息越晚到达。而读取消息的时候，会依次读取消息ID最小的记录，所以消息的顺序是保证的。具体代码的实现可以参考下面的发送消息以及读取消息的实现。

### 发送消息
发送消息，实际上就是向队列表中插入一条记录，消息会保存在队列表中，其实现如下：
```sql
-- send: actual implementation
CREATE FUNCTION pgmq.send(
    queue_name TEXT,     -- 队列名
    msg JSONB,           -- 消息内容
    headers JSONB,       -- 消息头
    delay TIMESTAMP WITH TIME ZONE     -- 延迟时间   
) RETURNS SETOF BIGINT AS $$
DECLARE
    sql TEXT;
    qtable TEXT := pgmq.format_table_name(queue_name, 'q');
BEGIN
    sql := FORMAT(
        $QUERY$
        INSERT INTO pgmq.%I (vt, message, headers)
        VALUES ($2, $1, $3)
        RETURNING msg_id;
        $QUERY$,
        qtable
    );   -- 向队列表插入消息，返回消息ID
    RETURN QUERY EXECUTE sql USING msg, delay, headers;
END;
$$ LANGUAGE plpgsql;
```

#### 支持批量发送消息
支持批量发送消息，其实现如下：
```sql
-- send_batch: actual implementation
CREATE FUNCTION pgmq.send_batch(
    queue_name TEXT,
    msgs JSONB[],
    headers JSONB[],
    delay TIMESTAMP WITH TIME ZONE
) RETURNS SETOF BIGINT AS $$
DECLARE
    sql TEXT;
    qtable TEXT := pgmq.format_table_name(queue_name, 'q');
BEGIN
    sql := FORMAT(
        $QUERY$
        INSERT INTO pgmq.%I (vt, message, headers)
        SELECT $2, unnest($1), unnest(coalesce($3, ARRAY[]::jsonb[]))
        RETURNING msg_id;
        $QUERY$,
        qtable
    );
    RETURN QUERY EXECUTE sql USING msgs, delay, headers;
END;
$$ LANGUAGE plpgsql;
```

#### 怎么确定消息是否发生成功呢？
返回消息ID则消息发送成功，由事务保证，如果消息发送失败，事务会回滚，所以消息不会插入到队列表中。不支持消息重试，需要业务自己实现。

#### 延迟队列
支持延迟队列，发送消息时，可设置延迟时间，延迟时间到达后，消息变为可见，可以被消费者读取。

### 读消息

读消息`pgmq.read`，实际上就是从队列表中读`qty`条记录，并更新`read_ct`字段，设置在`vt`时间内不可见，如果这些消息在`vt`时间内没有被删除或者归档，则将再次变为可见，并可以被另一个消费者读取。**支持批量读**，只需要在读时设置`qty`即可。其实现如下：
```sql
-- read
-- reads a number of messages from a queue, setting a visibility timeout on them
CREATE FUNCTION pgmq.read(
    queue_name TEXT,    -- 队列名
    vt INTEGER,         -- 设置不可见时间
    qty INTEGER,        -- 读取消息的数量
    conditional JSONB DEFAULT '{}'    -- 条件
)
RETURNS SETOF pgmq.message_record AS $$
DECLARE
    sql TEXT;
    qtable TEXT := pgmq.format_table_name(queue_name, 'q');
BEGIN
    sql := FORMAT(
        $QUERY$
        WITH cte AS
        (
            SELECT msg_id
            FROM pgmq.%I
            WHERE vt <= clock_timestamp() AND CASE
                WHEN %L != '{}'::jsonb THEN (message @> %2$L)::integer
                ELSE 1
            END = 1
            ORDER BY msg_id ASC   -- 按消息ID升序排序，保证消息的顺序
            LIMIT $1           -- 读几条消息
            FOR UPDATE SKIP LOCKED  -- 跳过被锁定行，仅处理未锁定的行，高并发场景避免阻塞提高效率
        )
        UPDATE pgmq.%I m
        SET
            vt = clock_timestamp() + %L,
            read_ct = read_ct + 1   -- 更新read_ct字段
        FROM cte
        WHERE m.msg_id = cte.msg_id
        RETURNING m.msg_id, m.read_ct, m.enqueued_at, m.vt, m.message, m.headers;
        $QUERY$,
        qtable, conditional, qtable, make_interval(secs => vt)
    );
    RETURN QUERY EXECUTE sql USING qty;
END;
$$ LANGUAGE plpgsql;
```
在PGMQ中，<u>缺少消息通知机制</u>，即，当队列中有新消息到达，无法通知消费者，对此，消费者需要定时去读取队列，这会导致消息的延迟，所以PGMQ中提供了`pgmq.read_with_poll`函数，该函数会定时读取队列，直到读取到消息或者超时，其实现如下：
```sql
---- read_with_poll
---- reads a number of messages from a queue, setting a visibility timeout on them
CREATE FUNCTION pgmq.read_with_poll(
    queue_name TEXT,
    vt INTEGER,
    qty INTEGER,
    max_poll_seconds INTEGER DEFAULT 5,
    poll_interval_ms INTEGER DEFAULT 100,
    conditional JSONB DEFAULT '{}'
)
RETURNS SETOF pgmq.message_record AS $$
DECLARE
    r pgmq.message_record;
    stop_at TIMESTAMP;
    sql TEXT;
    qtable TEXT := pgmq.format_table_name(queue_name, 'q');
BEGIN
    stop_at := clock_timestamp() + make_interval(secs => max_poll_seconds);
    LOOP
      IF (SELECT clock_timestamp() >= stop_at) THEN
        RETURN;   -- 如果超时，返回
      END IF;

      sql := FORMAT(
          $QUERY$
          WITH cte AS
          (
              SELECT msg_id
              FROM pgmq.%I
              WHERE vt <= clock_timestamp() AND CASE
                  WHEN %L != '{}'::jsonb THEN (message @> %2$L)::integer
                  ELSE 1
              END = 1
              ORDER BY msg_id ASC
              LIMIT $1
              FOR UPDATE SKIP LOCKED
          )
          UPDATE pgmq.%I m
          SET
              vt = clock_timestamp() + %L,
              read_ct = read_ct + 1
          FROM cte
          WHERE m.msg_id = cte.msg_id
          RETURNING m.msg_id, m.read_ct, m.enqueued_at, m.vt, m.message, m.headers;
          $QUERY$,
          qtable, conditional, qtable, make_interval(secs => vt)
      );

      FOR r IN
        EXECUTE sql USING qty
      LOOP
        RETURN NEXT r;
      END LOOP;
      IF FOUND THEN  -- 如果有消息，返回
        RETURN;
      ELSE     -- 如果没有消息，休眠后继续循环
        PERFORM pg_sleep(poll_interval_ms::numeric / 1000);
      END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### 弹出消息
可以弹出消息，即从队列表中读取消息后从队列中删除消息，与读消息的区别是，读消息后，消息仍然在队列中，而弹出消息后，消息从队列中删除，其实现如下：
```sql
-- pop a single message
CREATE FUNCTION pgmq.pop(queue_name TEXT)
RETURNS SETOF pgmq.message_record AS $$
DECLARE
    sql TEXT;
    result pgmq.message_record;
    qtable TEXT := pgmq.format_table_name(queue_name, 'q');
BEGIN
    sql := FORMAT(
        $QUERY$
        WITH cte AS
            (
                SELECT msg_id
                FROM pgmq.%I
                WHERE vt <= now()
                ORDER BY msg_id ASC
                LIMIT 1
                FOR UPDATE SKIP LOCKED
            )
        DELETE from pgmq.%I
        WHERE msg_id = (select msg_id from cte)
        RETURNING *;
        $QUERY$,
        qtable, qtable
    );
    RETURN QUERY EXECUTE sql;
END;
$$ LANGUAGE plpgsql;
```

### 关于多生产者多消费者的问题
PGMQ支持多生产者多消费者，但是PGMQ中不需要明确声明生产者或者注册消费者，任何向队列发送消息的都是生产者，任何从队列中读取消息的都是消费者。


### 消息归档
支持消息归档，当消息从消息队列归档到归档表中，消息从消息队列表中删除。实现如下：
```sql
---- removes a message from the queue, and sends it to the archive, where its
---- saved permanently.
CREATE FUNCTION pgmq.archive(
    queue_name TEXT,
    msg_id BIGINT
)
RETURNS BOOLEAN AS $$
DECLARE
    sql TEXT;
    result BIGINT;
    qtable TEXT := pgmq.format_table_name(queue_name, 'q');
    atable TEXT := pgmq.format_table_name(queue_name, 'a');
BEGIN
    sql := FORMAT(
        $QUERY$
        WITH archived AS (
            DELETE FROM pgmq.%I
            WHERE msg_id = $1
            RETURNING msg_id, vt, read_ct, enqueued_at, message, headers
        )
        INSERT INTO pgmq.%I (msg_id, vt, read_ct, enqueued_at, message, headers)
        SELECT msg_id, vt, read_ct, enqueued_at, message, headers
        FROM archived
        RETURNING msg_id;
        $QUERY$,
        qtable, atable
    );
    EXECUTE sql USING msg_id INTO result;
    RETURN NOT (result IS NULL);
END;
$$ LANGUAGE plpgsql;
```

### 消息删除
消息的删除，就是从消息队列表中删除消息，实现如下：
```sql    
---- deletes a message id from the queue permanently
CREATE FUNCTION pgmq.delete(
    queue_name TEXT,
    msg_id BIGINT
)
RETURNS BOOLEAN AS $$
DECLARE
    sql TEXT;
    result BIGINT;
    qtable TEXT := pgmq.format_table_name(queue_name, 'q');
BEGIN
    sql := FORMAT(
        $QUERY$
        DELETE FROM pgmq.%I
        WHERE msg_id = $1
        RETURNING msg_id
        $QUERY$,
        qtable
    );
    EXECUTE sql USING msg_id INTO result;
    RETURN NOT (result IS NULL);
END;
$$ LANGUAGE plpgsql;
```

### 队列管理
在使用队列时 ，我们还需要查看队列的信息，比如队列中有没有积压消息等，可通过函数`pgmq.metrics`以及`pgmq.metrics_all`查看队列的信息，示例如下：
```sql
postgres=# select * from pgmq.metrics('myqueue');
-[ RECORD 1 ]--------+------------------------------
queue_name           | myqueue
queue_length         | 3
newest_msg_age_sec   | 977
oldest_msg_age_sec   | 1009
total_messages       | 3
scrape_time          | 2025-03-27 11:16:55.540944+08
queue_visible_length | 3
```

查看当前有哪些队列：
```sql
-- list queues
CREATE FUNCTION pgmq."list_queues"()
RETURNS SETOF pgmq.queue_record AS $$
BEGIN
  RETURN QUERY SELECT * FROM pgmq.meta;
END
$$ LANGUAGE plpgsql;
```

清空某个队列中的消息：
```sql
-- purge queue, deleting all entries in it.
CREATE OR REPLACE FUNCTION pgmq."purge_queue"(queue_name TEXT)
RETURNS BIGINT AS $$
DECLARE
  deleted_count INTEGER;
  qtable TEXT := pgmq.format_table_name(queue_name, 'q');
BEGIN
  -- Get the row count before truncating
  EXECUTE format('SELECT count(*) FROM pgmq.%I', qtable) INTO deleted_count;

  -- Use TRUNCATE for better performance on large tables
  EXECUTE format('TRUNCATE TABLE pgmq.%I', qtable);

  -- Return the number of purged rows
  RETURN deleted_count;
END
$$ LANGUAGE plpgsql;
```

删除队列，删除队列时，会删除队列的元数据，消息队列表，归档表，消息id序列，以及消息队列的分区表，实现如下：
```sql
CREATE FUNCTION pgmq.drop_queue(queue_name TEXT)
RETURNS BOOLEAN AS $$
DECLARE
    qtable TEXT := pgmq.format_table_name(queue_name, 'q');
    qtable_seq TEXT := qtable || '_msg_id_seq';
    fq_qtable TEXT := 'pgmq.' || qtable;
    atable TEXT := pgmq.format_table_name(queue_name, 'a');
    fq_atable TEXT := 'pgmq.' || atable;
    partitioned BOOLEAN;
BEGIN
    EXECUTE FORMAT(
        $QUERY$
        SELECT is_partitioned FROM pgmq.meta WHERE queue_name = %L
        $QUERY$,
        queue_name
    ) INTO partitioned;

    EXECUTE FORMAT(
        $QUERY$
        ALTER EXTENSION pgmq DROP TABLE pgmq.%I
        $QUERY$,
        qtable
    );

    EXECUTE FORMAT(
        $QUERY$
        ALTER EXTENSION pgmq DROP SEQUENCE pgmq.%I
        $QUERY$,
        qtable_seq
    );

    EXECUTE FORMAT(
        $QUERY$
        ALTER EXTENSION pgmq DROP TABLE pgmq.%I
        $QUERY$,
        atable
    );

    EXECUTE FORMAT(
        $QUERY$
        DROP TABLE IF EXISTS pgmq.%I
        $QUERY$,
        qtable
    );

    EXECUTE FORMAT(
        $QUERY$
        DROP TABLE IF EXISTS pgmq.%I
        $QUERY$,
        atable
    );

     IF EXISTS (
          SELECT 1
          FROM information_schema.tables
          WHERE table_name = 'meta' and table_schema = 'pgmq'
     ) THEN
        EXECUTE FORMAT(
            $QUERY$
            DELETE FROM pgmq.meta WHERE queue_name = %L
            $QUERY$,
            queue_name
        );
     END IF;

     IF partitioned THEN
        EXECUTE FORMAT(
          $QUERY$
          DELETE FROM %I.part_config where parent_table in (%L, %L)
          $QUERY$,
          pgmq._get_pg_partman_schema(), fq_qtable, fq_atable
        );
     END IF;

    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

### 消息管理
可对消息的可见性进行设置，原理就是修改其`vt`字段，实现如下：
```sql
-- Sets vt of a message, returns it
CREATE FUNCTION pgmq.set_vt(queue_name TEXT, msg_id BIGINT, vt INTEGER)
RETURNS SETOF pgmq.message_record AS $$
DECLARE
    sql TEXT;
    result pgmq.message_record;
    qtable TEXT := pgmq.format_table_name(queue_name, 'q');
BEGIN
    sql := FORMAT(
        $QUERY$
        UPDATE pgmq.%I
        SET vt = (now() + %L)
        WHERE msg_id = %L
        RETURNING *;
        $QUERY$,
        qtable, make_interval(secs => vt), msg_id
    );
    RETURN QUERY EXECUTE sql;
END;
$$ LANGUAGE plpgsql;
```

### Rust客户端
为了方便用户的使用，除了提供SQL接口，还提供了Rust客户端，用户可以方便的使用Rust语言来操作消息队列。提供了两种客户端，一种是`pgmq::PGMQueueExt`，这种是封装了PGMQ扩展提供的SQL接口，另一种是`pgmq::PGMQueue`，这种是无需安装PGMQ扩展，直接可连接PostgreSQL数据库使用，相当于`pgmq::PGMQueue`根据PGMQ的实现原理，不通过plpgsql来实现，而是自己用SQL来实现。用户可以根据自己的需求选择合适的客户端。

具体的代码这里不再分析，只以创建队列和读队列为例，这里都是异步操作，需要使用`tokio`运行时来执行，示例如下：
```rust
/// Main controller for interacting with a managed by the PGMQ Postgres extension.
#[derive(Clone, Debug)]
pub struct PGMQueueExt {
    pub url: String,
    pub connection: Pool<Postgres>,
}

impl PGMQueueExt {
    pub async fn create_with_cxn<'c, E: sqlx::Executor<'c, Database = Postgres>>(
        &self,
        queue_name: &str,
        executor: E,
    ) -> Result<bool, PgmqError> {
        check_input(queue_name)?;
        sqlx::query!("SELECT * from pgmq.create($1::text);", queue_name)
            .execute(executor)
            .await?;
        Ok(true)
    }
    /// Errors when there is any database error and Ok(false) when the queue already exists.
    pub async fn create(&self, queue_name: &str) -> Result<bool, PgmqError> {
        self.create_with_cxn(queue_name, &self.connection).await?;
        Ok(true)
    }

        pub async fn read_with_cxn<
        'c,
        E: sqlx::Executor<'c, Database = Postgres>,
        T: for<'de> Deserialize<'de>,
    >(
        &self,
        queue_name: &str,
        vt: i32,
        executor: E,
    ) -> Result<Option<Message<T>>, PgmqError> {
        check_input(queue_name)?;
        let row = sqlx::query!(
            "SELECT * from pgmq.read($1::text, $2, $3)",
            queue_name,
            vt,
            1
        )
        .fetch_optional(executor)
        .await?;
        match row {
            Some(row) => {
                // happy path - successfully read a message
                let raw_msg = row.message.expect("no message");
                let parsed_msg = serde_json::from_value::<T>(raw_msg)?;
                Ok(Some(Message {
                    msg_id: row.msg_id.expect("msg_id missing from queue table"),
                    vt: row.vt.expect("vt missing from queue table"),
                    read_ct: row.read_ct.expect("read_ct missing from queue table"),
                    enqueued_at: row
                        .enqueued_at
                        .expect("enqueued_at missing from queue table"),
                    message: parsed_msg,
                }))
            }
            None => {
                // no message found
                Ok(None)
            }
        }
    }

    pub async fn read<T: for<'de> Deserialize<'de>>(
        &self,
        queue_name: &str,
        vt: i32,
    ) -> Result<Option<Message<T>>, PgmqError> {
        self.read_with_cxn(queue_name, vt, &self.connection).await
    }
}
```

---
参考文档：
[Postgres消息队列](https://www.dongaigc.com/p/tembo-io/pgmq)