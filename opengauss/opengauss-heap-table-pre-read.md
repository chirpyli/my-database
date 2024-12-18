## openGauss支持堆表预读

在对堆表进行扫描进行顺序页面读取时，通过一次行读入多个页面，来减少因频繁读单页的IO损耗，提升对堆表进行线性扫描的性能表现。

[堆表支持预读](https://docs.opengauss.org/zh/docs/6.0.0/docs/AboutopenGauss/%E5%A0%86%E8%A1%A8%E6%94%AF%E6%8C%81%E9%A2%84%E8%AF%BB.html)

在对数据库中的堆表进行线性扫描时，系统会将磁盘中的页面逐一读入内存。如果待扫描的堆表数据量巨大，频繁地访问磁盘会导致显著的性能损耗。为了解决这一问题，引入了预读功能。预读是指在对磁盘文件进行扫描时，操作系统不是逐个页面地读取，而是通过一次磁盘IO操作读取多个页面，这样可以显著减少因单页访问引起的频繁IO损耗。在数据库环境中，此功能同样适用于对堆表的线性扫描，可以一次性将多个页面读入内存，从而降低磁盘IO的次数。在执行lazy vacuum对堆表进行清理时，利用预读功能也可以加快扫描和清理过程。

用户可以根据自己的运行环境和业务需求来决定是否启用此功能，并适当调整参数大小。经验表明，在处理超过10GB数据的堆表时，启用预读功能能够有效提高线性扫描和lazy vacuum的性能。

与PolarDB实现基本相同，在一些细节实现有所不同：
1. GUC参数，相比Polardb，多了一个`vacuum_bulk_read_size`，单独为vacuum设置了一个GUC参数
2. 因为PolarDB依赖于PolarFS，而openGauss可使用单机文件系统，所以文件读取方式不同，PolarDB使用PolarFS的接口，openGauss使用VFS接口。

其他基本一致。代码实现见[opengauss-heap-pre-read.patch](./opengauss-heap-pre-read.patch)