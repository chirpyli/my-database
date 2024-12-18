## Aurora论文研读笔记
云原生数据库必读论文：[Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases](https://pdos.csail.mit.edu/6.824/papers/aurora.pdf)。在此学习研读一下。因英文水平以及数据库理解能力有限，一定存在较多不准确的地方，仅供批判参考。

为什么会有云原生数据库？底层基础设施出现了变化，出现了云计算，就一定会出现在相对于的底层基础设施（云计算）基础上构建的数据库（云原生数据库）。为什么是亚马逊诞生Aurora，因为它是最早一批开创云计算技术的公司，它有对云原生数据库最直接的需要。云原生数据库，就是要根据云计算的特点，设计出符合云计算的资源**弹性**管理特征的数据库。

我们下面看一下Aurora是怎么设计的，其中有一些关键词，论文阅读过程中，仔细体会。Aurora，基于MySQL，一写多读（最多支持15个只读节点），共享存储，计算存储分离。核心理念：“Log is Database”。


### ABSTRACT 摘要
Amazon Aurora is a relational database service for OLTP workloads offered as part of Amazon Web Services (AWS). In this paper, we describe the architecture of Aurora and the design considerations leading to that architecture. **We believe the central constraint in high throughput data processing has moved from compute and storage to the network.** <font color=red>Aurora brings a novel architecture to the relational database to address this constraint, most notably by pushing redo processing to a multi-tenant scale out storage service, purpose-built for Aurora. We describe how doing so not only reduces network traffic, but also allows for fast crash recovery, failovers to replicas without loss of data, and fault-tolerant, self-healing storage.</font> We then describe how Aurora achieves consensus on durable state across numerous storage nodes using an efficient asynchronous scheme, avoiding expensive and chatty recovery protocols. Finally, having operated Aurora as a production service for over 18 months, we share lessons we have learned from our customers on what modern cloud applications expect from their database tier.

Aurora是亚马逊云服务推出的云原生数据库服务，在MySQL基础上实现了计算存储分离架构，主要面向OLTP场景。Aurora认为在云环境下，数据库的最大瓶颈不再是计算或者存储资源，而是网络。基于此，Aurora设计了一个新的架构（计算存储分离）来解决此问题，将redo日志下推到存储层，这样不仅仅是降低了网络IO，而且带来其他的好处，比如快速故障恢复，读节点快速切换为读写节点，容错，存储自愈。此外，还讲述了存储节点如何实现保持一致。

架构上，相比单机数据库，其实现了计算存储分离架构，这样设计，是为了服务云计算“弹性”特征，计算按需使用，随时可以增加或者减少计算节点，存储则是“池化”，按需使用。

### 1. INTRODUCTION 引言


IT workloads are increasingly moving to public cloud providers. Significant reasons for this industry-wide transition include the ability to provision capacity on a flexible on-demand basis and to pay for this capacity using an operational expense as opposed to capital expense model. Many IT workloads require a relational OLTP database; providing equivalent or superior capabilities to on-premise databases is critical to support this secular transition.

越来越多的IT业务正在上云，需要一款匹配或者超越本地数据库的OLTP关系型数据库支持云上业务。

In modern distributed cloud services, resilience and scalability are increasingly achieved by decoupling compute from storage and by replicating storage across multiple nodes. Doing so lets us handle operations such as replacing misbehaving or unreachable hosts, adding replicas, failing over from a writer to a replica, scaling the size of a database instance up or down, etc.

在现代分布式云服务中，弹性和可扩展性通常是通过将计算与存储分离，并通过多节点间复制来实现。通过这种方式，可以处理诸如异常、主机不可达、增加副本、故障切换、数据库的扩容等等。

The I/O bottleneck faced by traditional database systems changes in this environment. Since I/Os can be spread across many nodes and many disks in a multi-tenant fleet, the individual disks and nodes are no longer hot. **Instead, the bottleneck moves to the network between the database tier requesting I/Os and the storage tier that performs these I/Os.** Beyond the basic bottlenecks of packets per second (PPS) and bandwidth, there is amplification of traffic since a performant database will issue writes out to the storage fleet in parallel. The performance of the outlier storage node, disk or network path can dominate response time.

在此背景下（云服务），传统数据库面临的I/O瓶颈正在发生变化。由于I/O操作可分布在多租户机群中的多个节点的多个磁盘上，因此，单机磁盘已不再是瓶颈。相反，瓶颈转移到了请求I/O的数据库层和执行这些I/O的存储层之间的网络上。底层存储节点的磁盘和网络决定了数据库的响应时间。

Although most operations in a database can overlap with each other, there are several situations that require synchronous operations. These result in stalls and context switches. One such situation is a disk read due to a miss in the database buffer cache. A reading thread cannot continue until its read completes. A cache miss may also incur the extra penalty of evicting and flushing a dirty cache page to accommodate the new page. Background processing such as checkpointing and dirty page writing can 
reduce the occurrence of this penalty, but can also cause stalls, context switches and resource contention.

尽管数据库中大多数操作都可以并发执行，但在某些情况下需要同步操作。这会导致停滞和上下文切换。一种典型的场景是因buffer未命中而导致的磁盘读，读线程在读完成前无法继续进行后续操作。Buffer未命中还可能导致因为了读取新页而将脏页进行刷盘已避免Buffer溢出。后台处理比如checkpoint以及刷脏页可以降低触发此种情况的频率，但是仍可能会引发停滞、上下文切换以及资源竞争。

Transaction commits are another source of interference; a stall in committing one transaction can inhibit others from progressing. Handling commits with multi-phase synchronization protocols such as 2-phase commit (2PC) is challenging in a cloud-scale distributed system. These protocols are intolerant of failure and high-scale distributed systems have a continual “background noise” of hard and soft failures. They are also high latency, as high scale systems are distributed across multiple data centers.

事务提交是另一种影响性能的干扰项，在提交一个事务时出现阻塞会阻塞其他事务的处理。在云分布式系统中，使用类似2阶段提交之类的多阶段同步协议是一个巨大的挑战。这些协议对错误的容忍度低，而大规模分布式系统中软硬件故障是常态化存在的。除此之外，因为大规模分布式系统经常跨越多个数据中心，存在很高的时延，这对性能同样影响巨大。

In this paper, we describe Amazon Aurora, a new database service that addresses the above issues by more aggressively leveraging the redo log across a highly-distributed cloud environment. We use a novel service-oriented architecture (see Figure 1) with a multi-tenant scale-out storage service that abstracts a virtualized segmented redo log and is loosely coupled to a fleet of database instances. **Although each instance still includes most of the components of a traditional kernel (query processor, transactions, locking, buffer cache, access methods and undo management) several functions (redo logging, durable storage, crash recovery, and backup/restore) are off-loaded to the storage service.**

对此，本文提出了一种新型数据库Aurora，旨在通过Redo日志来解决云环境中存在的上述问题。我们使用了一种新型的面向服务的架构，如下图所示。该存储服务抽象出一种虚拟化的分段redo日志并与数据库实例解耦。尽管每个数据库实例仍包含传统数据库的大部分功能组件 （query processor、transactions、locking、buffer cache、access methods、undo management），但是将其中一些功能（redo logging、durable storage、crash recovery、backup/restore）剥离到了存储服务中。

![image](https://img2020.cnblogs.com/blog/1616773/202008/1616773-20200803163744790-2118598358.png)

Our architecture has three significant advantages over traditional approaches. First, by building storage as an independent fault-tolerant and self-healing service across multiple data-centers, we protect the database from performance variance and transient or permanent failures at either the networking or storage tiers. We observe that a failure in durability can be modeled as a long-lasting availability event, and an availability event can be modeled as a long-lasting performance variation – a well-designed system can treat each of these uniformly. Second, by only writing redo log records to storage, we are able to reduce network IOPS by an order of magnitude. Once we removed this bottleneck, we were able to aggressively optimize numerous other points of contention, obtaining significant throughput improvements over the base MySQL code base from which we started. Third, we move some of the most complex and critical functions (backup and redo recovery) from one-time expensive operations in the database engine to continuous asynchronous operations amortized across a large distributed fleet. This yields near-instant crash recovery without checkpointing as well as inexpensive backups that do not interfere with foreground processing.

相比传统架构，我们的架构有3点优势：
- 通过构建跨多个数据中心的、可以容错和自愈的存储服务，避免了数据库受网络或存储层故障带来的影响。
- 通过仅写redo日志到存储，将网络IOPS降低了一个数据级
- 将一些复杂和关键的功能（备份、redo恢复）转移到了一个大型分布式集群中异步进行处理，带来了一些好处，故障恢复快

In this paper, we describe three contributions:
1. How to reason about durability at cloud scale and how to design quorum systems that are resilient to correlated failures. (Section 2).  存储部分，云环境怎么设计基于quorum协议的存储系统以实现持久性、故障容忍。
2. How to leverage smart storage by offloading the lower quarter of a traditional database to this tier. (Section 3).  怎么实现Log is database，计算存储如何分离
3. How to eliminate multi-phase synchronization, crash recovery and checkpointing in distributed storage (Section 4). 如何减少多阶段同步，崩溃恢复以及checkpoint

We then show how we bring these three ideas together to design the overall architecture of Aurora in Section 5, followed by a review of our performance results in Section 6 and the lessons we have learned in Section 7. Finally, we briefly survey related workin Section 8 and present concluding remarks in Section 9.

介绍一下后面几章节论文讲述的内容，比如整体架构，性能测试情况等。


### 2. DURABILITY AT SCALE 大规模系统如何实现数据持久性

If a database system does nothing else, it must satisfy the contract that data, once written, can be read. Not all systems do. In this section, we discuss the rationale behind our quorum model, why we segment storage, and how the two, in combination, provide not only durability, availability and reduction of jitter, but also help us solve the operational issues of managing a storage fleet at scale.

为啥采用Quorum模型？下面将解释为啥？

#### 2.1 Replication and Correlated Failures 复制以及故障容错

Instance lifetime does not correlate well with storage lifetime. Instances fail. Customers shut them down. They resize them up and down based on load. For these reasons, it helps to decouple the storage tier from the compute tier.
数据库实例的生命周期与数据存储的生命周期无必然联系。数据库实例故障时，用户可以将其关闭。用户可以根据负载情况调整实例配置。对此，是有利于计算存储分离的。

Once you do so, those storage nodes and disks can also fail. They therefore must be replicated in some form to provide resiliency to failure. In a large-scale cloud environment, there is a continuous low level background noise of node, disk and network path failures. Each failure can have a different duration and a different blast radius. For example, one can have a transient lack of network availability to a node, temporary downtime on a reboot, or a permanent failure of a disk, a node, a rack, a leaf or a spine network switch, or even a data center.
一旦进行计算存储分离，存储节点和磁盘也可能发生故障，所以必须实现数据复制以提供容错能力。在大规模云环境中，节点、磁盘、网络故障是常态化的，每种故障可能持续的时间不同，影响范围不同。例如，会出现短暂的网络不可达的情况，重启导致的临时停机，磁盘、节点、机架甚至数据中心的永久故障。

One approach to tolerate failures in a replicated system is to use a quorum-based voting protocol as described in. If each of the V copies of a replicated data item is assigned a vote, a read or write operation must respectively obtain a read quorum of Vr votes or a write quorum of Vw votes. To achieve consistency, the quorums must obey two rules. First, each read must be aware of the most recent write, formulated as Vr + Vw > V. This rule ensures the set of nodes used for a read intersects with the set of nodes used for a write and the read quorum contains at least one location with the newest version. Second, each write must be aware of the most recent write to avoid conflicting writes, formulated as Vw > V/2.
实现故障容错的一种方式是通过复制，使用Quorum协议。假设有V个副本，为了保证一致性，需满足下面两点：
- Vr + Vw > V， 就是说读副本数 + 写副本数 > 总副本数，这样就能保证每次可以读到拥有最新写数据的节点
- Vw > V/2  每次写都要保证能获取到上次写的最新数据，避免写冲突。

A common approach to tolerate the loss of a single node is to replicate data to (V = 3) nodes and rely on a write quorum of 2/3 (Vw = 2) and a read quorum of 2/3 (Vr = 2).
一种常见方法是3副本，写2读2。

We believe 2/3 quorums are inadequate. To understand why, let’s first understand the concept of an Availability Zone (AZ) in AWS. An AZ is a subset of a Region that is connected to other AZs in the region through low latency links but is isolated for most faults, including power, networking, software deployments, flooding, etc. Distributing data replicas across AZs ensures that typical failure modalities at scale only impact one data replica. This implies that one can simply place each of the three replicas in a different AZ, and be tolerant to large-scale events in addition to the smaller individual failures.
但我们认为2/3 Quorum是不够的。为啥？我们需要先了解AZ的概念。AZ是相对独立的，AZ直接的故障一般是互相隔离的，比如电源、网络、软件部署、洪灾等。将数据副本分布在不同的AZ中，可以尽可能的将故障限定在一个AZ中，从而能够容忍较小故障。

However, in a large storage fleet, the background noise of failures implies that, at any given moment in time, some subset of disks or nodes may have failed and are being repaired. These failures may be spread independently across nodes in each of AZ A, B and C. However, the failure of AZ C, due to a fire, roof failure, flood, etc, will break quorum for any of the replicas that concurrently have failures in AZ A or AZ B. At that point, in a 2/3 read quorum model, we will have lost two copies and will be unable to determine if the third is up to date. In other words, while the individual failures of replicas in each of the AZs are uncorrelated, the failure of an AZ is a correlated failure of all disks and nodes in that AZ. **Quorums need to tolerate an AZ failure as well as concurrently occuring background noise failures.**

在大型存储系统中，故障是无法避免的，意味着在任何时候都有可能会发生故障或者正在修复中。这些故障或许独立分布在每个AZ中的节点上。然而，可能因为火灾、地震、洪水等原因导致AZ C故障，此时如果AZ A或者B中的副本节点也碰巧发生故障，那么就打破了Quorum的要求。此时，在2/3Quorum下，我们将丢失两份副本，并无法确定第三份副本是否是最新的。换言之，虽然每个AZ之间单个副本故障是不相关的，但是AZ的故障与该AZ中的所有节点和磁盘故障都相关。Quorum需要容忍AZ故障以及同时发生的节点等背景噪声故障。


In Aurora, we have chosen a design point of tolerating (a) losing an entire AZ and one additional node (AZ+1) without losing data, and (b) losing an entire AZ without impacting the ability to write data. We achieve this by replicating each data item 6 ways across 3 AZs with 2 copies of each item in each AZ. We use a quorum model with 6 votes (V = 6), a write quorum of 4/6 (Vw = 4), and a read quorum of 3/6 (Vr = 3). With such a model, we can (a) lose a single AZ and one additional node (a failure of 3 nodes) without losing read availability, and (b) lose any two nodes, including a single AZ failure and maintain write availability. Ensuring read quorum enables us to rebuild write quorum by adding additional replica copies.





---

### 参考文档
[Amazon Aurora 深度探索](https://codechina.gitcode.host/programmer/practice-of-big-data/16-Amazon-Aurora.html)
[Aurora：如何设计云原生关系数据库](https://iswade.github.io/translate/Aurora_design_cloud_native_database/)
[【译】Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases 上篇](https://zhuanlan.zhihu.com/p/208680604)
[Aurora论文理解](https://www.jianshu.com/p/dd6aa53c3af5)
[Amazon Aurora 论文笔记](https://www.cnblogs.com/zzk0/p/13425353.html)
