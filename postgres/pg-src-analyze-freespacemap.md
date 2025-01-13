### PostgreSQL源码分析——空闲空间映射表

#### 为什么需要空闲空间映射表？
随着表不断地执行插入和删除元组，元组块中必然产生空闲空间，当需要新插入一个元组时，会存在一个问题，这个元组插入那个页面呢？如果遍历整个表，找到第一个有空闲空间的页面，那么效率会非常低，因此PostgreSQL引入了空闲空间映射表，用于快速定位一个有足够空间存储元组的页面，或者确定没有这样的页面，从而需要扩展一个页面。PostgreSQL源代码中的README是这样说的The purpose of the free space map is to quickly locate a page with enough free space to hold a tuple to be stored; or to determine that no such page exists and the relation must be extended by one page.翻译过来就是为了快速找到一个有足够空间存储元组的页面，或者确定没有这样的页面，从而需要扩展一个页面。

#### FSM如何设计？
为了加快查找速度，FSM文件应该尽量的小，我们知道一个页是8KB大小，也就是最大空闲空间不会超过8192，表示8192需要2个字节，为了压缩FSM文件的大小，我们用一个字节去表示每个页的空闲空间大小，一个字节有8位，最大表示255，8192/256=32，所以，实际FSM文件中存储的值需要乘以32为其实际大小。0表示没有空闲空间。

那么具体的FSM文件是怎么组织的呢？最简单的方式是采用一个大数组的方式。
```c++
typedef uint32 BlockNumber;

#define InvalidBlockNumber		((BlockNumber) 0xFFFFFFFF)

#define MaxBlockNumber			((BlockNumber) 0xFFFFFFFE)
```
PostgreSQL中最多可以有2^32-1个数据页，每个数据页需要用1个字节表示空闲空间大小，则需要4G的空间，在如此大的空间查找到空闲空间最大的块，采用数组的方式查找效率是非常低的，因此，PostgreSQL采用了一种树状结构来组织FSM文件，FSM文件由8KB的FSM块页组成，在FSM块之间使用3层树的结构，第0层和第1层位辅助层，第2层FSM中用于实际存放各表块的空闲空间值。每层构造最大堆，每个FSM块内构造一个局部的最大堆二叉树。
```
For example:

    4
 4     2
3 4   0 2    <- This level represents heap pages
```
为什么是三层结构？ 每个FSM块默认大学8KB，除去必要的文件块头部信息，FSM块中剩下的空间全部用来存储块内的最大堆二叉树，每个叶子节点用一个字节表示空闲空间大小，有完全二叉树的性质可以计算出来，每个FSM块大概可以保存4000个叶子节点，两层结构的话，可以保存4000*4000 < 2^32个叶子节点。
[image](https://www.postgresql.org/message-id/attachment/23510/fsm-drawing.png)

三层的话4000*4000*4000 > 2^32，所以需要三层结构。


#### 表插入元组的过程
这里只分析大概的流程，实际向表中插入一条元组很复杂，涉及到很多细节，比如锁，WAL日志，事务等等。向表中插入元组，首先要确定插入到那个页中，这个页需要有大于新增的元组的大小的空间空间，可需要调用`GetPageWithFreeSpace(Relation rel, Size spaceNeeded)`返回含有指定空闲空间大小的块号。
```c++
ExecInsert
--> table_tuple_insert
    --> heapam_tuple_insert
        // 向heap表中插入元组
        --> heap_insert
            --> RelationGetBufferForTuple(relation, heaptup->t_len,...)   // 获取一个可插入tuple的数据页，页的空闲空间需要大于heaptup->t_len
                --> GetPageWithFreeSpace(relation, targetFreeSpace); // 获取一个有足够空间存储元组的页面
                --> page = BufferGetPage(buffer);   // 获取页面
            
                --> pageFreeSpace = PageGetHeapFreeSpace(page); // 获取页面空闲空间大小
                --> RelationSetTargetBlock(relation, targetBlock); // 设置目标块
            --> RelationPutHeapTuple(relation, buffer, heaptup,(options & HEAP_INSERT_SPECULATIVE) != 0); // 插入元组
            --> MarkBufferDirty(buffer);    // 标记buffer为脏页
            // ... 插入WAL日志

```

#### 如何计算一个页的空闲空间大小？
根据堆表的页面布局，可用的空闲空间大小等于`pd_upper - pd_lower - sizeof(ItemIdData)`。
```c++
Size PageGetFreeSpace(Page page)
{
	int			space;

	/*
	 * Use signed arithmetic here so that we behave sensibly if pd_lower >
	 * pd_upper.
	 */
	space = (int) ((PageHeader) page)->pd_upper -
		(int) ((PageHeader) page)->pd_lower;

	if (space < (int) sizeof(ItemIdData))
		return 0;
	space -= sizeof(ItemIdData);

	return (Size) space;
}
```

#### 什么时候更新FSM呢？

我们知道在PostgreSQL中，可通过auto-vacuum或者手动执行vacuum来进行空间回收，vacuum会对表中的页进行清理，清理后重新计算表空间大小，更新FSM文件。
```c++
vacuum_rel
--> vacuum_open_relation  // 执行vacuum时，需要对表加锁,vacuum full则需要加AccessExclusiveLock，否则加ShareUpdateExclusiveLock
--> table_relation_vacuum(rel, params, vac_strategy);
    --> heap_vacuum_rel(rel, params, bstrategy)
        --> lazy_scan_heap(vacrel, params, aggressive);     // 执行lazy vacuum 区别于full vacuum
            --> for (blkno = 0; blkno < nblokcs; blkno++)
                {
                    // 获取块号对应的buffer
                    buf = ReadBufferExtended(vacrel->rel, MAIN_FORKNUM, blkno, RBM_NORMAL, vacrel->bstrategy);

                    page = BufferGetPage(buf);

                    // 裁剪页面
                    lazy_scan_prune(vacrel, buf, blkno, page, vistest, &prunestate);
                    --> heap_page_prune

                    Size freespace = PageGetHeapFreeSpace(page);
                    RecordPageWithFreeSpace(vacrel->rel, blkno, freespace);
                    --> fsm_set_and_search(rel, addr, slot, new_cat, 0);
                        --> fsm_readbuf(rel, addr, true);
                        --> fsm_set_avail(page, slot, newValue)
                }

```