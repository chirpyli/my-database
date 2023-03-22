### 关于OLTP、OLAP、HTAP的一点点思考
最早出现的数据库是OLTP数据库，上层应用对数据库的主要需求可以概况为增删改查（INSERT, UPDATE, DELETE, SELECT）。典型的产品比如Oracle、MySQL、PostgreSQL等。后来随着信息化程度的提高以及数据量的增加，企业有了对数据进行分析的需求，比如聚合等，由此诞生了OLAP型数据库。典型的产品比如Greenplum、ClickHouse等。再后来，出现了HTAP的概念，实际上，不存在纯粹的HTAP数据库，更多的是AP数据库不断增强TP处理能力，比如Greenplum，可参考论文《Greenplum: A Hybrid Database for Transactional and Analytical Workloads》 ，TP数据库不断增强AP处理能力比如TiDB，可参考论文：《TiDB: A Raft-based HTAP Database》。

这里说不存在纯粹的HTAP数据库，并不是说没有，或者HTAP没有意义，而是说，HTAP想要同时达到TP以及AP的能力非常难。事实上，HTAP是非常有意义的，这里引用一段Greenplum论文中的描述：
- First, HTAP can reduce the waiting time of new data analysis tasks significantly, as there is no ETL transferring delay.
-  Second, HTAP systems can also reduce the overall business cost in terms of hardware and administration.

这两点都有巨大的商业价值：实时分析能力的增加，成本的降低，效率的提升。值得数据库厂商朝着这个目标去努力。

现有阶段实现HTAP的困难在于，目前基本不存在从零开始设计的HTAP数据库，大都是从TP向HTAP，AP向HTAP去走，这就存在问题了，TP数据库为了在其在事务型业务的性能，针对性的做了很多设计，比如执行计划缓存，索引，行存等，而在AP场景中，执行计划缓存没什么意义，索引的重要性下降，更多情况是全表扫描，列存相对行存在AP场景更有优势。所以要同时在两个方向都取得最佳的性能，难度是非常大的。即使是从零开始设计HTAP也是非常难以取舍的。

一个产品如果强调通用性，其某一单一方向往往难以达到极致。同理，HTAP的设计使得其难以在TP或者AP方向取得极致。ClickHouse就是一个典型，完全围绕AP场景设计，在单一方向取得极致。所以不存在谁更先进，谁是传统的，谁是未来，它们各自有其优势领域，由用户决定那个更适合自己的业务场景，既能满足业务需求同时综合成本最低。这里岔开一个话题，就是成本问题，商业数据库的用户必然面对成本问题，在同等性能指标或相差不大情况下，谁的成本低（这里的成本低不单指售价，也包括用户的全周期使用运维成本）谁的技术就更有优势，谁就更有机会在激烈的市场竞争中存活下来。一句话概括就是用最低的成本为用户解决问题，低成本本身就是一种技术优势。

数据库最终的发展，其引领方向的并不是数据库本身，而是数据库上层的应用业务需求。只要有上层应用业务需求的需要，这个方向就是对的，毕竟数据库就是拿来给业务用的。未来的业务应用一定是丰富的，业务需求也不再是单一的，所以未来的数据库也一定是各有特色（关系数据库、时序数据库、图数据库、向量数据库、多模数据库、NoSQL数据库、分布式数据库、分析型数据库、HTAP数据库、云原生数据库......），很难出现一款数据库包打天下的情况。

在TP、AP越来越成熟，竞争越来越激烈的情况下，向HTAP发展也是个很不错的方向。最后，希望国产数据库的未来越来越好，做出有世界影响力的产品。
