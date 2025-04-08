## 关于PostgreSQL消息队列的思考
### 背景
需要调研PostgreSQL实现消息队列以替换RabbitMQ的方案。

给出各扩展和RabbitMQ的功能、性能对比，如果pgmq、pgq都无法满足，需要再分析一下PostgreSQL是否有其他功能可以替换RabbitMQ。


PostgreSQL异步通知机制，具有最简单的消息通知功能，但缺少可靠性保证，PostgreSQL异常重启后，消息会丢失，消息不是写入到表中，是通过SLRU缓冲池写入pg_notify目录下的临时段文件中，没有WAL日志，无法实现高可用。

### PostgreSQL实现消息队列

#### 实现方案
PostgreSQL目前的实现消息队列方案有：
- pgmq
- pgq

#### 基本原理
数据库实现消息队列，并不是像RabbitMQ一样，实现了AMQP协议，而是通过数据库机制来实现类似消息队列的功能。对外提供的接口是SQL接口。 数据库实现消息队列，通常都是通过表来实现队列，PGQ也是基于表来实现消息队列。一个队列就是一张表，客户端发送消息到队列，实际上就是向表中插入了一条数据。消费者读消息，就是读这张抽象为队列的表，消费消息实际上就是从被抽象为队列的表中删除消息。为了方便用户使用，封装了一些API来操作队列，与RabbitMQ不同的是，这里提供的是纯SQL接口，用户可以直接使用SQL来操作队列，比如发送消息、消费消息等。

#### 对替换RabbitMQ的思考
如果想要完整的替换RabbitMQ，如果用到了RabbitMQ中的复杂功能，则PostgreSQL中的替换方案基本不可行。无论是pgmq还是pgq，只支持基本的消息队列功能，没有提供丰富的路由等功能。想要对业务无感替换，是做不到的，提供的API为SQL接口。RabbitMQ为独立的专业消息中间件，而PostgreSQL是一个数据库，完全替换基本上不可行，但是，某些场景下，可以用PostgreSQL的消息队列来满足业务需求。重点应该是找到PostgreSQL消息队列适合的场景，而不是完全替换RabbitMQ。


### 哪些场景建议使用PostgreSQL消息队列
本身技术栈已经是PostgreSQL，并且对消息队列功能要求不高，比如不需要消息确认，不需要消息重试，不需要消息路由，不需要消息优先级，不需要消息过期等。因为RabbitMQ等独立消息中间件会增加运维成本。


如果本身的技术栈没有PostgreSQL，不推荐引入PostgreSQL消息队列，因为PostgreSQL的消息队列功能比较简单，如果要实现复杂的消息队列功能，则需要引入RabbitMQ等独立消息中间件。

其最佳的适用场景：
- 数据库事务内异步操作：需要与数据库事务强集成的异步任务处理场景（如订单创建后发送短信，订单支付成功后发送邮件）
- 轻量级消息队列：场景简单，不需要复杂路由、优先级、过期等高级功能的消息队列场景，只需简单消息队列功能即可满足需求，且消息量适中，不需要高并发、高吞吐量的消息队列场景。


### RabbitMQ与PGMQ的对比

#### 功能对比

|功能对比|RabbitMQ|PGMQ|PGQ|
|--|--|--|--|
|消息持久化|支持|支持|支持|
|消息确认机制|支持ACK/NACK机制|数据库事务保证，无需消息确认机制|数据库事务保证，无需消息确认机制|
|消息重试|支持|不支持（如果事务失败，业务层去处理重试的问题） |不支持（如果事务失败，业务层去处理重试的问题）|
|消息路由|支持多种路由规则，如fanout、direct、topic等|不支持|不支持|
|消息优先级|支持|不支持|不支持|
|消息过期|支持|不支持|不支持|
|事务| 支持AMQP协议事务机制、生产者确认机制| 支持数据库事务| 支持数据库事务|

因PGQ与PGMQ实现原理基本一致，故只对比PGMQ与RabbitMQ。

|特性 |PGMQ|RabbitMQ|
|--|--|--|
|架构与实现|PGMQ是构建在PostgreSQL数据库之上的消息队列系统，无需额外部署独立的消息队列服务|Erlang实现，独立的消息队列中间件，采用AMQP协议，单独部署和管理，可集群化以实现高可用和扩展性|
|性能| 性能受限于数据库性能，适合小规模的消息处理场景、高并发、大规模消息处理的场景下受限于PostgreSQL的写入性能 | 高并发、大规模消息处理的场景下，性能表现良好，延迟较低|
|可靠性| 继承PostgreSQL的可靠性，支持事务处理，可以保证消息的原子性和一致性。依赖数据库备份和恢复机制| 提供多种可靠性机制，如消息确认、持久化存储、支持集群镜像队列等，支持集群自动故障转移|
|协议支持|仅支持SQL接口| 支持多种消息协议，如AMQP、MQTT、STOMP等|
|消息持久化| 依赖PostgreSQL的持久化机制（消息实际上是存储在表中）| 支持磁盘持久化|
|消息路由| 简单队列模型、无复杂路由规则| 支持灵活的路由规则，如fanout、direct、topic等|
|消息确认机制| 支持visibility timeout、显式删除| 支持ACK/NACK机制|

#### 性能对比：

8核8G

##### RabbitMQ 测试结果：

基础场景：单生产者单消费者
```bash
# 消息大小 1KB，队列名为 "test_queue"
java -jar perf-test-2.23.0.jar -x 1 -y 1 -s 1024 -u "test_queue"
id: test-144514-761, sending rate avg: 26583 msg/s
id: test-144514-761, receiving rate avg: 26580 msg/s
id: test-144514-761, consumer latency min/median/75th/95th/99th 8766/23473/26926/37135/63106 µs
```
发送/接收吞吐量：26580 msg/s， 时延：37135 µs

高并发场景：10 生产者 + 10 消费者
```bash
# 持续 60 秒，消息大小 512B，预取值 500
java -jar perf-test-2.23.0.jar -x 10 -y 10 -s 512 --time 60 -q 500
id: test-145333-156, sending rate avg: 6400 msg/s
id: test-145333-156, receiving rate avg: 49924 msg/s
id: test-145333-156, consumer latency min/median/75th/95th/99th 66898/22712797/25327685/28434121/29969780 µs
```
发送吞吐量：6400 msg/s， 接收吞吐量：49924 msg/s， 时延：28434121 µs

##### PGMQ 测试结果：

发送吞吐量：
```bash
# 单生产者，持续60秒
postgres@slpc:~$ pgbench -f pgmq_test.sql -c 1 -T 60 -r postgres
pgbench (14.15)
transaction type: pgmq_test.sql
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 60 s
number of transactions actually processed: 89406
latency average = 0.671 ms
initial connection time = 4.058 ms
tps = 1490.185810 (without initial connection time)
statement latencies in milliseconds:
         0.671  select pgmq.send('test_queue','{"test":"test event"}');

# 10生产者，持续60秒
postgres@slpc:~$ pgbench -f pgmq_test.sql -c 10 -T 60 -r postgres
pgbench (14.15)
transaction type: pgmq_test.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
duration: 60 s
number of transactions actually processed: 187176
latency average = 3.202 ms
initial connection time = 62.007 ms
tps = 3122.647965 (without initial connection time)
statement latencies in milliseconds:
         3.101  select pgmq.send('test_queue','{"test":"test event"}');

# 100个生产者，持续60秒
postgres@slpc:~$ pgbench -f pgmq_test.sql -c 100 -T 60 -r postgres
pgbench (14.15)
transaction type: pgmq_test.sql
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 1
duration: 60 s
number of transactions actually processed: 488202
latency average = 12.207 ms
initial connection time = 539.254 ms
tps = 8191.970680 (without initial connection time)
statement latencies in milliseconds:
        10.595  select pgmq.send('test_queue','{"test":"test event"}');
```

消费吞吐量：
```bash 
postgres@slpc:~$ pgbench -f pgmq_test.sql -c 10 -T 60 -r postgres
pgbench (14.15)
starting vacuum...end.
transaction type: pgmq_test.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
duration: 60 s
number of transactions actually processed: 165059
latency average = 3.633 ms
initial connection time = 35.184 ms
tps = 2752.402197 (without initial connection time)
statement latencies in milliseconds:
         3.559  select pgmq.pop('test_queue');

```
消费pop较发送的tps低


---



以下是 **PostgreSQL PGMQ** 与 **RabbitMQ** 的对比分析，从架构、性能、适用场景等多个维度展开：

---

### 一、核心概念对比

| **特性**          | **PGMQ（基于 PostgreSQL）**                 | **RabbitMQ**                               |
|--------------------|---------------------------------------------|--------------------------------------------|
| **本质**          | 基于关系数据库的消息队列实现                 | 专用的消息代理（Message Broker）           |
| **协议**          | 无标准协议，通过 SQL 操作                     | 支持 AMQP 0.9.1、MQTT、STOMP 等协议        |
| **持久化**        | 强持久化（依赖 PostgreSQL 事务日志）         | 可配置持久化（消息和队列持久化到磁盘）      |
| **数据一致性**    | 强一致性（ACID 事务保证）                     | 最终一致性（异步刷盘时可能丢失部分消息）    |
| **部署复杂度**    | 无需额外组件（直接使用 PostgreSQL）           | 需独立部署消息服务器                        |

---

### 二、性能对比

| **指标**          | **PGMQ**                                   | **RabbitMQ**                               |
|--------------------|--------------------------------------------|--------------------------------------------|
| **吞吐量**        | 中等（受限于磁盘 I/O 和事务开销）            | 高（内存操作，可达 10万+/秒）               |
| **延迟**          | 较高（毫秒级，事务提交和索引查询开销）       | 极低（微秒级，内存处理）                    |
| **并发处理**      | 依赖 PostgreSQL 连接池和锁机制               | 原生支持高并发（Erlang 轻量级进程模型）      |
| **扩展性**        | 垂直扩展（提升 PostgreSQL 性能）             | 水平扩展（集群、镜像队列）                   |

---

### 三、功能特性对比

| **功能**              | **PGMQ**                                   | **RabbitMQ**                               |
|------------------------|--------------------------------------------|--------------------------------------------|
| **消息路由**          | 简单（基于 SQL 查询）                       | 复杂（支持 Direct、Topic、Fanout 等路由规则）|
| **消息确认机制**      | 需手动更新状态（依赖 SQL 事务）             | 自动 ACK/NACK（支持重试、死信队列）          |
| **消息顺序性**        | 严格有序（基于自增主键）                    | 不保证全局有序（除非单消费者单队列）         |
| **消息优先级**        | 不支持                                      | 支持（队列级别或消息级别优先级）             |
| **死信队列**          | 需手动实现（通过状态字段或触发器）           | 原生支持（Dead Letter Exchange）            |
| **监控与管理**        | 依赖 PostgreSQL 监控工具（如 `pg_stat`）    | 内置 Web UI、Prometheus 集成、HTTP API      |

---

### 四、适用场景对比

#### **1. 选择 PGMQ 的场景**  
- **强一致性需求**：消息处理需与业务数据保持 ACID 事务（如扣款后发消息）。  
- **简化架构**：已使用 PostgreSQL，希望减少组件依赖。  
- **低频但关键任务**：如对账、审计日志（允许稍高延迟）。  

**示例**：  
```sql
-- 在事务中同时更新业务表和消息表
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
INSERT INTO pgmq (payload) VALUES ('{"event": "payment", "amount": 100}');
COMMIT;
```

#### **2. 选择 RabbitMQ 的场景**  
- **高吞吐量需求**：如电商秒杀、实时日志处理。  
- **复杂消息路由**：需灵活的消息分发机制（如按 Topic 广播）。  
- **高可用性**：通过集群和镜像队列实现故障转移。  
- **异步解耦**：微服务间松耦合通信。  

**示例**：  
```python
# 生产者发送消息到 Exchange
channel.basic_publish(
    exchange='orders',
    routing_key='payment.success',
    body='{"order_id": 123}'
)

# 消费者订阅队列
channel.basic_consume(
    queue='payment_queue',
    on_message_callback=process_payment
)
```

---

### 五、优缺点总结

| **队列类型**       | **优点**                                  | **缺点**                                  |
|--------------------|-------------------------------------------|-------------------------------------------|
| **PGMQ**           | - 强一致性<br>- 与数据库深度集成<br>- 无需额外运维 | - 吞吐量较低<br>- 功能简单<br>- 依赖 SQL 性能 |
| **RabbitMQ**       | - 高吞吐、低延迟<br>- 功能丰富<br>- 易扩展   | - 需独立维护<br>- 学习成本高<br>- 最终一致性 |

---

### 六、混合架构建议

若场景需要兼顾 **强一致性** 和 **高吞吐量**，可结合两者：  
1. **关键事务消息**：使用 PGMQ 保证与数据库操作原子性。  
2. **异步处理消息**：通过 RabbitMQ 分发高吞吐任务。  

**数据流转示例**：  
1. 用户在 PostgreSQL 中下单，事务内写入 PGMQ。  
2. 消费者从 PGMQ 读取订单，处理完成后发送到 RabbitMQ 通知其他服务。  
3. 物流服务通过 RabbitMQ 接收消息并异步处理。  

---

### 七、性能优化技巧

#### **PGMQ 优化**  
1. **分区表**：按时间或状态分区，减少索引扫描范围。  
2. **批量提交**：合并多条消息插入为单个事务。  
3. **连接池**：使用 `pgbouncer` 减少连接开销。  

#### **RabbitMQ 优化**  
1. **镜像队列**：提升可用性和负载均衡。  
2. **预取限制（Prefetch）**：避免消费者过载。  
3. **持久化策略**：权衡性能与可靠性（非持久化队列速度更快）。  

---

### 八、总结

- **PGMQ** 是轻量级、与 PostgreSQL 深度集成的方案，适合事务性强、吞吐量适中的场景。  
- **RabbitMQ** 是专业级消息中间件，适合高吞吐、低延迟和复杂路由需求的场景。  

根据业务需求选择，或在混合架构中结合两者优势。


---

## PostgreSQL消息队列拓展PGMQ与PGQ的对比分析

