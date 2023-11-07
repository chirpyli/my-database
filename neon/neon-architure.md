Neon， Serverless Postgres。 可以认为是一款云原生PostgreSQL数据库。计算存储分离架构。


架构上，可分为计算节点（PostgreSQL）和存储引擎（Neon）。存储主要包含2部分：

*   PageServer: 计算节点的弹性存储后端
*   Safekeepers： Safekeepers组成一个WAL服务，接收计算节点的WAL日志，在被PageServer处理前以及上传到云存储前持久化存储。



### 计算节点

计算节点改动很少，具体改动点可参考[Postgres core changes](https://github.com/neondatabase/neon/blob/main/docs/core_changes.md#postgres-core-changes)。负责解析用户输入的SQL，生成WAL日志，返回给用户计算结果。

一写多读，无状态，源码：


### Pageserver

这块是Neon的实现重点之一。功能点如下：

*   存储和管理页数据，存储数据到S3完成持久化，从S3获取需要的数据
*   响应计算节点的读页请求，GetPage@LSN
*   从WAL serice接收WAL日志并解析
*   回放WAL日志
*
具体实现时，PageServer包含如下线程：
```
                                           | WAL
                                           V
                                   +--------------+
                                   |              |
                                   | WAL receiver |
                                   |              |
                                   +--------------+
                                                                                 ......
                  +---------+                              +--------+            .    .
                  |         |                              |        |            .    .
 GetPage@LSN      |         |                              | backup |  ------->  . S3 .
------------->    |  Page   |         repository           |        |            .    .
                  | Service |                              +--------+            .    .
   page           |         |                                                    ......
<-------------    |         |
                  +---------+     +-----------+     +--------------------+
                                  | WAL redo  |     | Checkpointing,     |
                  +----------+    | processes |     | Garbage collection |
                  |          |    +-----------+     +--------------------+
                  |   HTTP   |
                  | mgmt API |
                  |          |
                  +----------+

Legend:

+--+
|  |   A thread or multi-threaded service
+--+

--->   Data flow
<---
```

这部分的内容很多。我们看每个线程的功能：
- Page Service: 监听计算节点的页请求，GetPage@LSN，返回指定的页，通过libpq协议与计算节点进行通信。
- WAL Receiver: 连接到safekeep日志服务中，使用PG物理流复制协议，接收WAL日志
- Backup service: 备份数据到S3中，也可以存储到本地盘（一般用于测试）
- repository: 存储了所有的页版本以及需要的WAL，（多版本页）
- WAL redo processes: 重放WAL日志，满足GetPage@LSN时，重放到指定LSN，同时，不断地重放WAL日志，避免每个页累积太多的WAL日志
- Checkpointing / Garbage collection: 周期性的刷内存中累积的WAL日志到磁盘中，以便WAL safekeeper可以删除旧的WAL日志。同时可以释放内存去接收新的WAL日志。

#### 存储
Page Server会接收Safekeeper的WAL日志，存储在内存中，因为内存空间是有限的，需要周期性的将内存中的WAL刷盘到本地盘中，最终会被存储到云存储中。
```
Cloud Storage                   Page Server                           Safekeeper
                        L1               L0             Memory            WAL

+----+               +----+----+
|AAAA|               |AAAA|AAAA|      +---+-----+         |
+----+               +----+----+      |   |     |         |AA
|BBBB|               |BBBB|BBBB|      |BB | AA  |         |BB
+----+----+          +----+----+      |C  | BB  |         |CC
|CCCC|CCCC|  <----   |CCCC|CCCC| <--- |D  | CC  |  <---   |DDD     <----   ADEBAABED
+----+----+          +----+----+      |   | DDD |         |E
|DDDD|DDDD|          |DDDD|DDDD|      |E  |     |         |
+----+----+          +----+----+      |   |     |
|EEEE|               |EEEE|EEEE|      +---+-----+
+----+               +----+----+
```


### WAL Service

计算存储分离架构，计算节点生成WAL日志，存储层回放WAL日志更新数据页。中间并不是计算节点之间发送WAL日志到存储层，而是先将WAL日志发送到 WAL Service，负责持久化和多副本高可靠。Pageserver再从WAL Service中消费WAL日志。

```
+--------------+              +------------------+
|              |     WAL      |                  |
| Compute node |  ----------> |   WAL Service    |
|              |              |                  |
+--------------+              +------------------+
                                     |
                                     |
                                     | WAL
                                     |
                                     |
                                     V
                              +--------------+
                              |              |
                              | Pageservers  |
                              |              |
                              +--------------+
```


WAL service由多个safekeepers构成，每个safekeepers拥有一份WAL日志拷贝，由Paxos算法保证一致性。


```
  +-------------------------------------------+
  | WAL Service                               |
  |                                           |
  |                                           |
  |  +------------+                           |
  |  | safekeeper |                           |
  |  +------------+                           |
  |                                           |
  |  +------------+                           |
  |  | safekeeper |                           |
  |  +------------+                           |
  |                                           |
  |  +------------+                           |
  |  | safekeeper |                           |
  |  +------------+                           |
  |                                           |
  +-------------------------------------------+
```


主节点连接到WAL safekeepers，作为发送方。对此，需要在Postgres计算主节点生成一个wal proposer工作进程负责发送WAL日志，接收方是safekeeper。

Page Server连接WAL safekeeper，使用与PG主备流复制相同的协议。所以，你也可以直接连接Postgres主节点用于测试。

在生成环境部署上，一般将多个safekeeper节点部署到不同的节点上。

具体工作中，wal proposer为Poxos共识算法中的提议者角色proposer，safekeeper为acceptor角色。更多可参考[WAL proposer-safekeeper communication consensus protocol.](https://github.com/neondatabase/neon/blob/main/docs/safekeeper-protocol.md)。在具体的设计中，为了降低对PG本身的修改，使用了标准的流复制，广播WAL到safekeepers。safekeeper之间不交互数据，WAL proposer建立与safekeepers的连接，任何一个safekeeper都可以作为WAL server。

目前计算节点是一主多读的架构，不支持多写，主节点必须是唯一的，为了解决脑裂的问题，新主的生成须经过选举，safekeeper只会接收合法的主节点的WAL，避免了safekeeper可能会收到旧主WAL消息的问题。
Handshake的主要目的是收集投票，避免脑裂（新主和旧主同时存在）。过程如下：
1. 广播信息给所有safekeeper（wal segment size, system_id, ...）
2. 收到safekeeper回应的消息
3. 一旦达到合法的投票数，达成一个新主节点的共识，广播新NodeId(max(term)+1, server.uuid)给所有safekeepers
4. safekeeper一旦收到新主proposed的NodeId，与本地存储的nodeid进行对比，如果不比本地的小，则更新nodeid到本地控制文件
5. 如果足够的safekeeper达成一致，则服务端认为握手成功结束，新主已产生，切换到recovery阶段。




### Proxy

Postgres协议代理/路由。 可接受psql的连接请求，并认证检查。更多可参考[Proxy](https://github.com/neondatabase/neon/blob/main/proxy/README.md)。



---
参考文档
[云原生存算分离数据库Neon的架构决策](https://mp.weixin.qq.com/s/JcLuX4v-CT4hHnLae_gMLA)