## PostgreSQL源码分析——INSERT

这里我们对INSERT语句进行分析， 只分析其宏观过程，具体细节后续再分析。我们就分析下面的语句的执行过程。

```sql
insert into t1 values(4,4);
```

### 主流程
主流程如下：
```c++
exec_simple_query
--> pg_parse_query   //语法解析
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformInsertStmt  //语义分析

--> pg_plan_queries    //查询优化，生成执行计划
    --> pg_plan_query
        --> standard_planner
            --> subquery_planner
                --> grouping_planner
                    --> query_planner  
                        --> build_simple_rel
                        --> make_one_rel
                    --> create_modifytable_path // INSERT/UPDATE/DELETE都会生成ModifyTablePath
            --> create_plan
--> PortalStart
--> PortalRun
    --> ProcessQuery
        --> ExecutorStart
            --> InitPlan
                --> ExecInitModifyTable
        --> ExecutorRun
            --> ExecutePlan
                --> ExecModifyTable
                    --> planSlot = ExecProcNode(subplanstate); // 执行子执行计划Result，获取要插入的tuple值， 对应values(4,4)
                    --> ExecInitInsertProjection
                    --> ExecGetInsertNewTuple
                    --> ExecInsert  // 执行插入
                        --> table_tuple_insert
        --> ExecutorEnd
--> PortalDrop
```

### 查询优化

这里已经分析的比较多了，可参考之前的源码分析文章[Postgres源码分析——UPDATE](https://www.modb.pro/db/477696)。

### 执行器
执行插入，首先是在Bufer缓冲区中获取空闲页，如果Buffer没有指定表的页，则新增一个空白页，向页中插入一条元组。在插入元组前，需要构造元组所须的隐藏字段，事务ID，cid等。标记Buffer中的页为脏页（后台进程会处理脏页进行刷盘—）。之后是写入WAL日志。需要注意的是写WAL日志也是先写入WAL Buffer，再刷盘的。并不是直接写文件。


主流程：
```c++
ExecInsert  // 执行插入
--> table_tuple_insert
    --> heap_insert
        /* *******  Buffer中向Page插入一条tuple ***********/
        --> GetCurrentTransactionId()
        --> heap_prepare_insert //  Prepares a tuple for insertion
        --> RelationGetBufferForTuple // Buffer中获取足够插入tuple大小的页，
            --> GetPageWithFreeSpace
                --> fsm_search  // 结合fsm，查找含有足够空闲size的页
            // 如果fsm没有足够信息，尝试last page，避免one-tuple-per-page syndrome during bootstrapping or in a recently-started system.
        --> CheckForSerializableConflictIn
        --> RelationPutHeapTuple // 向页中插入tuple
            --> BufferGetPage(buffer);
            --> PageAddItemExtended
        --> MarkBufferDirty
        /****** 写WAL日志 ***********/
        --> XLogBeginInsert
        --> XLogRegisterData
        --> XLogRegisterBuffer
        --> XLogRegisterBufData
        --> XLogSetRecordFlags
        --> XLogInsert
            --> XLogRecordAssemble  // 由前面的信息生成日志记录
            --> XLogInsertRecord    // 插入WAL日志中
                --> CopyXLogRecordToWAL(rechdr->xl_tot_len, isLogSwitch, rdata,StartPos, EndPos);
                    -->  GetXLogBuffer(CurrPos)
                --> XLogFlush(EndPos) // Ensure that all XLOG data through the given position is flushed to disk.
                    --> XLogWrite // 写入WAL日志文件
                
        --> PageSetLSN(page, recptr);
```


关键数据结构：

```c++
// HeapTupleData is an in-memory data structure that points to a tuple.
typedef struct HeapTupleData
{
	uint32		t_len;			/* length of *t_data */
	ItemPointerData t_self;		/* SelfItemPointer */
	Oid			t_tableOid;		/* table the tuple came from */
#define FIELDNO_HEAPTUPLEDATA_DATA 3
	HeapTupleHeader t_data;		/* -> tuple header and data */
} HeapTupleData;

```
下面我们就重点看看元组是怎么插入到表中的。理解页面布局的话，下面的代码就很容易理解，可以参考《PostgreSQL指南：内幕探索》中第1.3章节。

```c++
// Insert a tuple from a slot into table AM routine.
// rel: 被插入的表
// slot: 插入的数据
static inline void 
table_tuple_insert(Relation rel, TupleTableSlot *slot, CommandId cid, int options, struct BulkInsertStateData *bistate)
{
	rel->rd_tableam->tuple_insert(rel, slot, cid, options, bistate);    
}

static void
heapam_tuple_insert(Relation relation, TupleTableSlot *slot, CommandId cid, int options, BulkInsertState bistate)
{
	bool		shouldFree = true;
	HeapTuple	tuple = ExecFetchSlotHeapTuple(slot, true, &shouldFree);

	/* Update the tuple with table oid */
	slot->tts_tableOid = RelationGetRelid(relation);
	tuple->t_tableOid = slot->tts_tableOid;

	/* Perform the insertion, and copy the resulting ItemPointer */
	heap_insert(relation, tuple, cid, options, bistate);
	ItemPointerCopy(&tuple->t_self, &slot->tts_tid);

	if (shouldFree)
		pfree(tuple);
}

/*
 *	heap_insert		- insert tuple into a heap
 *
 * The new tuple is stamped with current transaction ID and the specified command ID.
 */
void
heap_insert(Relation relation, HeapTuple tup, CommandId cid, int options, BulkInsertState bistate)
{
	TransactionId xid = GetCurrentTransactionId();  // 获取事务XID
	HeapTuple	heaptup;
	Buffer		buffer;
	Buffer		vmbuffer = InvalidBuffer;
	bool		all_visible_cleared = false;

	/*
	 * Fill in tuple header fields and toast the tuple if necessary.
	 *
	 * Note: below this point, heaptup is the data we actually intend to store
	 * into the relation; tup is the caller's original untoasted data.
	 */
	heaptup = heap_prepare_insert(relation, tup, xid, cid, options);

	/*
	 * Find buffer to insert this tuple into.  If the page is all visible,
	 * this will also pin the requisite visibility map page.*/
    // Buffer中获取足够插入tuple大小的页， 要大于heaptup->t_len
	buffer = RelationGetBufferForTuple(relation, heaptup->t_len,
									   InvalidBuffer, options, bistate,
									   &vmbuffer, NULL);

	CheckForSerializableConflictIn(relation, NULL, InvalidBlockNumber);

	/* NO EREPORT(ERROR) from here till changes are logged */
	START_CRIT_SECTION();

    // 向页中插入tuple
	RelationPutHeapTuple(relation, buffer, heaptup,
						 (options & HEAP_INSERT_SPECULATIVE) != 0);

    MarkBufferDirty(buffer);

    /* XLOG stuff */
	if (RelationNeedsWAL(relation))
	{
        XLogBeginInsert();
		XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);

		xlhdr.t_infomask2 = heaptup->t_data->t_infomask2;
		xlhdr.t_infomask = heaptup->t_data->t_infomask;
		xlhdr.t_hoff = heaptup->t_data->t_hoff;

		/* note we mark xlhdr as belonging to buffer; if XLogInsert decides to
		 * write the whole page to the xlog, we don't need to store
		 * xl_heap_header in the xlog. */
		XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);
		XLogRegisterBufData(0, (char *) &xlhdr, SizeOfHeapHeader);
		/* PG73FORMAT: write bitmap [+ padding] [+ oid] + data */
		XLogRegisterBufData(0,
							(char *) heaptup->t_data + SizeofHeapTupleHeader,
							heaptup->t_len - SizeofHeapTupleHeader);

		/* filtering by origin on a row level is much more efficient */
		XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);
        // 把数据写入WAL Buffer
		recptr = XLogInsert(RM_HEAP_ID, info);

		PageSetLSN(page, recptr);
    }
}
```

对于WAL日志部分内容，可参考《PostgreSQL技术内幕：事务处理深度探索》中的第4章。


### 页中插入一条元组
这个函数帮助我们理解Page, tuple，物理结构，看一条元组怎么插入页中

```c++
/*
 * RelationPutHeapTuple - place tuple at specified page
 *
 * !!! EREPORT(ERROR) IS DISALLOWED HERE !!!  Must PANIC on failure!!!
 *
 * Note - caller must hold BUFFER_LOCK_EXCLUSIVE on the buffer.
 */
void
RelationPutHeapTuple(Relation relation,
					 Buffer buffer,
					 HeapTuple tuple,
					 bool token)
{
	Page		pageHeader;
	OffsetNumber offnum;
    // ...

	/* Add the tuple to the page */
	pageHeader = BufferGetPage(buffer);

	offnum = PageAddItem(pageHeader, (Item) tuple->t_data,
						 tuple->t_len, InvalidOffsetNumber, false, true);

	/* Update tuple->t_self to the actual position where it was stored */
	ItemPointerSet(&(tuple->t_self), BufferGetBlockNumber(buffer), offnum);

    // ...
}
```
具体插入是在`PageAddItemExtended`函数中实现的。
```c++
OffsetNumber
PageAddItemExtended(Page page,
					Item item,
					Size size,
					OffsetNumber offsetNumber,
					int flags)
{
	PageHeader	phdr = (PageHeader) page;
	Size		alignedSize;
	int			lower;
	int			upper;
	ItemId		itemId;
	OffsetNumber limit;
	bool		needshuffle = false;

	/*
	 * Be wary about corrupted page pointers
	 */
	if (phdr->pd_lower < SizeOfPageHeaderData ||
		phdr->pd_lower > phdr->pd_upper ||
		phdr->pd_upper > phdr->pd_special ||
		phdr->pd_special > BLCKSZ)
		ereport(PANIC,
				(errcode(ERRCODE_DATA_CORRUPTED),
				 errmsg("corrupted page pointers: lower = %u, upper = %u, special = %u",
						phdr->pd_lower, phdr->pd_upper, phdr->pd_special)));

	/*
	 * Select offsetNumber to place the new item at
	 */
	limit = OffsetNumberNext(PageGetMaxOffsetNumber(page));

	/* was offsetNumber passed in? */
	if (OffsetNumberIsValid(offsetNumber))
	{
		/* yes, check it */
		if ((flags & PAI_OVERWRITE) != 0)
		{
			if (offsetNumber < limit)
			{
				itemId = PageGetItemId(phdr, offsetNumber);
				if (ItemIdIsUsed(itemId) || ItemIdHasStorage(itemId))
				{
					elog(WARNING, "will not overwrite a used ItemId");
					return InvalidOffsetNumber;
				}
			}
		}
		else
		{
			if (offsetNumber < limit)
				needshuffle = true; /* need to move existing linp's */
		}
	}
	else
	{
		/* offsetNumber was not passed in, so find a free slot */
		/* if no free slot, we'll put it at limit (1st open slot) */
		if (PageHasFreeLinePointers(phdr))
		{
			/*
			 * Scan line pointer array to locate a "recyclable" (unused)
			 * ItemId.
			 *
			 * Always use earlier items first.  PageTruncateLinePointerArray
			 * can only truncate unused items when they appear as a contiguous
			 * group at the end of the line pointer array.
			 */
			for (offsetNumber = FirstOffsetNumber;
				 offsetNumber < limit;	/* limit is maxoff+1 */
				 offsetNumber++)
			{
				itemId = PageGetItemId(phdr, offsetNumber);

				/*
				 * We check for no storage as well, just to be paranoid;
				 * unused items should never have storage.  Assert() that the
				 * invariant is respected too.
				 */
				Assert(ItemIdIsUsed(itemId) || !ItemIdHasStorage(itemId));

				if (!ItemIdIsUsed(itemId) && !ItemIdHasStorage(itemId))
					break;
			}
			if (offsetNumber >= limit)
			{
				/* the hint is wrong, so reset it */
				PageClearHasFreeLinePointers(phdr);
			}
		}
		else
		{
			/* don't bother searching if hint says there's no free slot */
			offsetNumber = limit;
		}
	}

	/* Reject placing items beyond the first unused line pointer */
	if (offsetNumber > limit)
	{
		elog(WARNING, "specified item offset is too large");
		return InvalidOffsetNumber;
	}

	/* Reject placing items beyond heap boundary, if heap */
	if ((flags & PAI_IS_HEAP) != 0 && offsetNumber > MaxHeapTuplesPerPage)
	{
		elog(WARNING, "can't put more than MaxHeapTuplesPerPage items in a heap page");
		return InvalidOffsetNumber;
	}

	/*
	 * Compute new lower and upper pointers for page, see if it'll fit.
	 *
	 * Note: do arithmetic as signed ints, to avoid mistakes if, say,
	 * alignedSize > pd_upper.
	 */
	if (offsetNumber == limit || needshuffle)
		lower = phdr->pd_lower + sizeof(ItemIdData);
	else
		lower = phdr->pd_lower;

	alignedSize = MAXALIGN(size);

	upper = (int) phdr->pd_upper - (int) alignedSize;

	if (lower > upper)
		return InvalidOffsetNumber;

	/*
	 * OK to insert the item.  First, shuffle the existing pointers if needed.
	 */
	itemId = PageGetItemId(phdr, offsetNumber);

	if (needshuffle)
		memmove(itemId + 1, itemId,
				(limit - offsetNumber) * sizeof(ItemIdData));

	/* set the line pointer */
	ItemIdSetNormal(itemId, upper, size);

	/*
	 * Items normally contain no uninitialized bytes.  Core bufpage consumers
	 * conform, but this is not a necessary coding rule; a new index AM could
	 * opt to depart from it.  However, data type input functions and other
	 * C-language functions that synthesize datums should initialize all
	 * bytes; datumIsEqual() relies on this.  Testing here, along with the
	 * similar check in printtup(), helps to catch such mistakes.
	 *
	 * Values of the "name" type retrieved via index-only scans may contain
	 * uninitialized bytes; see comment in btrescan().  Valgrind will report
	 * this as an error, but it is safe to ignore.
	 */
	VALGRIND_CHECK_MEM_IS_DEFINED(item, size);

	/* copy the item's data onto the page */
	memcpy((char *) page + upper, item, size);

	/* adjust page header */
	phdr->pd_lower = (LocationIndex) lower;
	phdr->pd_upper = (LocationIndex) upper;

	return offsetNumber;
}
```

分析数据库的源码，有一个困难，那就是源码实在是太多了，内容太多，所以每次分析就只能有一个侧重点，不可能面面都去分析。最好是抓住一个功能点分析，不要漫天去分析代码。