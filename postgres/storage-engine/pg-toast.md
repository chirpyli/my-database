
### PostgreSQL中的TOAST机制

#### 如何存储超大字段？
PostgreSQL中数据存储在堆表heap中，堆表一个页为8k，但是如果一个元组的大小非常大或者超过了页大小怎么办呢？你不能强行把这个超大的元组强行放进页中，因为会破坏页的结构，怎么办呢？只能是想办法放在堆表页外的其他地方，这个地方就是Toast表。对应的解决这一问题的相关实现被称为TOAST（The Oversized-Attribute Storage Technique）超尺寸字段存储技巧机制。

如果一个元组超过一定大小，那么就会将部分变长属性值存放在外部toast表中。但是数据页都是有一定大小限制的，toast表中也可能存在某个属性值的大小超过页大小的情况，怎么解决呢？通过分片的形式，将分片大小限制在页size的范围内，然后通过多个分片组合存储，通过多个页就可以存储一个比较大的属性值。那么这个阈值是多少呢?
```c++
#define TOAST_TUPLES_PER_PAGE	4  // 期望一个页中能够存储的toast处理后的元组数
#define TOAST_TUPLE_THRESHOLD	MaximumBytesPerTuple(TOAST_TUPLES_PER_PAGE)

#define TOAST_TUPLE_TARGET		TOAST_TUPLE_THRESHOLD
```
也就是说当元组大小超过`TOAST_TUPLE_THRESHOLD`的时候就会触发toast处理，toast会尝试将元组压缩或存储在外部toast表中，使其不超过`TOAST_TUPLE_TARGET`字节。


那么什么时候创建toast表呢？toast表无需用户创建，对用户而言，其感知不到toast表的存在，toast表是在用户创建表时，如果创建的表的属性中有属性是可以toast的（比如text这种变长数据类型），那么将会为该表管理一个toast表，该toast表存储这些超大属性。对于大属性，PG引入了下列多种存储策略：
- PLAIN：避免压缩和行外存储。只有那些不需要TOAST策略就能存放的数据类型允许选择（例如 int 类型），而对于text这类要求存储长度超过页大小的类型，是不允许采用此策略的。
- EXTENDED：允许压缩和行外存储。一般会先压缩，如果还是太大，就会行外存储。
- EXTERNAL：允许行外存储，但不许压缩。类似字符串这种会对数据的一部分进行操作的字段，采用此策略可能获得更高的性能，因为不需要读取出整行数据再解压。
- MAIN：允许压缩，但不许行外存储。不过实际上，为了保证过大数据的存储，行外存储在其它方式（例如压缩）都无法满足需求的情况下，作为最后手段还是会被启动。因此理解为：尽量不使用行外存储更贴切。

<font color=blue>通常，为了保证存储密度，PG一页（大小为8K字节）至少存储四个元组。</font>因此理论上，一个元组大小阈值最大为：（8K字节 - 页面头部）/ 4。进行TOAST存储时，我们依次应用以下算法原则，并在该行不再超过阈值时立即停止：
1. 首先，我们从“最长”属性到“最短”属性，通过“EXTERNAL”和“EXTENDED”策略来遍历属性。如果EXTENDED属性被压缩（如果有效的话），并且如果值本身超过页面的四分之一，它将立即进入TOAST表。External属性的处理方式相同，但未压缩。
2. 如果在第一遍之后行版本仍不适合该页面，则将带有“EXTERNAL”和“EXTENDED”策略的其余属性传输到TOAST表。
3. 如果这也没有帮助，可以尝试使用MAIN策略压缩属性，但将其保留在表的页中。
4. 在步骤3之后，该行仍然不够短时，MAIN属性才能进入TOAST表。
有时，更改某些列的策略可能很有用。例如，如果事先知道无法压缩列中的数据，则可以为其设置“EXTERNAL”策略，这样可以避免不必要的压缩尝试，从而节省时间。请注意，TOAST仅适用于表，不适用于索引。这对要索引的键的大小施加了限制。

```sql
-- 创建表，含有text类型的属性，将会有一个关联的toast表。
postgres=# create table t3(a int, b text, c text);
CREATE TABLE
postgres=# \d+ t3
                                            Table "public.t3"
 Column |  Type   | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
--------+---------+-----------+----------+---------+----------+-------------+--------------+-------------
 a      | integer |           |          |         | plain    |             |              | 
 b      | text    |           |          |         | extended |             |              | 
 c      | text    |           |          |         | extended |             |              | 
Access method: heap

postgres=# \x
Expanded display is on.
postgres=# select * from pg_class where relname='t3';
-[ RECORD 1 ]-------+--------
oid                 | 20042
relname             | t3
relnamespace        | 2200
reltype             | 20044
reloftype           | 0
relowner            | 10
relam               | 2
relfilenode         | 20042
reltablespace       | 0
relpages            | 0
reltuples           | -1
relallvisible       | 0
reltoastrelid       | 20045		-- toast表
relhasindex         | f
relisshared         | f
relpersistence      | p
relkind             | r
relnatts            | 3

-- 查看toast表信息
postgres=# select * from pg_class where oid=20045;
-[ RECORD 1 ]-------+---------------
oid                 | 20045
relname             | pg_toast_20042   -- toast表名
relnamespace        | 99               -- pg_toast命名空间
reltype             | 0
reloftype           | 0
relowner            | 10
relam               | 2
relfilenode         | 20045


-- 查看toast表结构
postgres=# \d+ pg_toast.pg_toast_20042
TOAST table "pg_toast.pg_toast_20042"
   Column   |  Type   | Storage 
------------+---------+---------
 chunk_id   | oid     | plain      -- 为整个toast数据分配的OID
 chunk_seq  | integer | plain      -- 序列号，存储该片段在整个TOAST数据中的位置
 chunk_data | bytea   | plain      -- 存储该片段的实际数据
Owning table: "public.t3"
Indexes:
    "pg_toast_20042_index" PRIMARY KEY, btree (chunk_id, chunk_seq)
Access method: heap

-- 设置字段b为external，不进行压缩，可存储在行外存储中
postgres=# alter table t3 alter b set storage external;
ALTER TABLE
postgres=# \d+ t3
                                            Table "public.t3"
 Column |  Type   | Collation | Nullable | Default | Storage  | Compression | Stats target | Descripti
on 
--------+---------+-----------+----------+---------+----------+-------------+--------------+----------
---
 a      | integer |           |          |         | plain    |             |              | 
 b      | text    |           |          |         | external |             |              | 
 c      | text    |           |          |         | extended |             |              | 
Access method: heap

-- 元组大小没有超过阈值，没有触发toast机制，无需放入toast表中，放入toast表是有代价的，并不是extended策略就一定存储在外部
postgres=# insert into t3 values(1, '12345678','12345678');
INSERT 0 1
-- 查看toast表数据
postgres=# select * from pg_toast.pg_toast_20042;
 chunk_id | chunk_seq | chunk_data 
----------+-----------+------------
(0 rows)

-- 插入8k数据，超过了阈值，数据会被切片存储到toast表中
postgres=# insert into t3 values(2,repeat('12345678',1024));
INSERT 0 1
-- 查看toast表，发现数据被切片了。1996*+208=8192字节，数据被切片成了5片
postgres=# select chunk_id,chunk_seq,length(chunk_data),left(encode(chunk_data,'escape')::text,8) || '...' || right(encode(chunk_data,'escape')::text, 8) from pg_toast.pg_toast_20042;
 chunk_id | chunk_seq | length |      ?column?       
----------+-----------+--------+---------------------
    20048 |         0 |   1996 | 12345678...56781234
    20048 |         1 |   1996 | 56781234...12345678
    20048 |         2 |   1996 | 12345678...56781234
    20048 |         3 |   1996 | 56781234...12345678
    20048 |         4 |    208 | 12345678...12345678
(5 rows)

```

#### 什么时候创建toast表
什么时候创建toast表呢？创建toast表的过程对用户来说是透明无感的，<font color=blue>在创建表的时候，如果含有可toast的属性，那么就会在创建表后，创建toast表，并将toast表的oid信息对原表在pg_class.reltoastrelid字段进行更新。</font>
```c++
	CreateStmt *cstmt = (CreateStmt *) stmt;
	Datum		toast_options;

	/* 创建表 */
	address = DefineRelation(cstmt, RELKIND_RELATION, InvalidOid, NULL, queryString);

	/* parse and validate reloptions for the toast table */
	toast_options = transformRelOptions((Datum) 0,cstmt->options,"toast",validnsps,true,false);
	(void) heap_reloptions(RELKIND_TOASTVALUE, toast_options, true);

    /* 创建toast表，并与主表关联，更新主表的pg_class.reltoastrelid */
	NewRelationCreateToastTable(address.objectId,toast_options); 
```

#### toast表结构

在建表时，如果属性是可以toast的，则会为该表创建关联的toast表，并创建一个索引，表名为`pg_toast_<reloid>`。
```c++
	/* Create the toast table and its index */
	snprintf(toast_relname, sizeof(toast_relname),
			 "pg_toast_%u", relOid);
	snprintf(toast_idxname, sizeof(toast_idxname),
			 "pg_toast_%u_index", relOid);
```

该表需要存储超大属性值，那么在存储与查询时，怎么知道当前元组的属性值在toast表中的那个位置呢？这就需要我们设计toast表结构：
```sql
postgres=# \d+ pg_toast.pg_toast_20042
TOAST table "pg_toast.pg_toast_20042"
   Column   |  Type   | Storage 
------------+---------+---------
 chunk_id   | oid     | plain      -- 为整个toast数据分配的OID
 chunk_seq  | integer | plain      -- 序列号，存储该片段在整个TOAST数据中的位置
 chunk_data | bytea   | plain      -- 存储该片段的实际数据
Owning table: "public.t3"
Indexes:
    "pg_toast_20042_index" PRIMARY KEY, btree (chunk_id, chunk_seq)
Access method: heap
```
<font color=blue>我们通过chunk_id来辨识是属于哪个元组的数据，</font>每个被存储在toast的数据都会被分配一个OID，通过这个OID可以在toast表中找到属于该toast数据的所有片段，进而可以重组该数据。前面讲过为了解决超大数据可能会超过页大小的问题，采用分片存储，将原有的超大数据切分为多个片段chunk，通过chunk_seq为分片分配顺序，将分片数据保存在chunk_data中。这里还有一个问题，那就是分片大小？分片的大小为`TOAST_MAX_CHUNK_SIZE`字节，每个分片都作为独立的元组存储在toast表中。
```c++
#define EXTERN_TUPLES_PER_PAGE	4	/* toast表中一个页中切片的数量，这个值是可以被修改的 */

#define EXTERN_TUPLE_MAX_SIZE	MaximumBytesPerTuple(EXTERN_TUPLES_PER_PAGE)
/* 每个切片的最大大小 */
#define TOAST_MAX_CHUNK_SIZE	\
	(EXTERN_TUPLE_MAX_SIZE -							\
	 MAXALIGN(SizeofHeapTupleHeader) -					\
	 sizeof(Oid) -										\
	 sizeof(int32) -									\
	 VARHDRSZ)
```

那么具体源码实现中，通过`varatt_external`来表示toast指针。也就是通过这个来定位到toast表数据。该结构体描述了一个存储在toast表中的数据的元信息：
```c++
typedef struct varatt_external
{
	int32		va_rawsize;		/* 原数据大小（包含头部） */
	uint32		va_extinfo;		/* 外存储的数据大小，不包含头部和压缩方法 */
	Oid			va_valueid;		/* 在toast表中的唯一OID */
	Oid			va_toastrelid;	/* 存储该值的toast表的relid */
}			varatt_external;
```

总结访问toast表的流程：首先从表的toast属性中获取<font color=red>toast指针</font>，然后通过toast指针找到toast表，再通过toast数据的oid在toast表中找到所有的分片并按序号拼装起来得到完整的toast数据。

#### 元组插入/更新时的TOAST操作

在插入或者更新时，对于含有大字段的列，可能会进行toast处理，在插入时，会判断元组大小`tup->t_len`是否超过了阈值`TOAST_TUPLE_THRESHOLD`，如果超过了，则触发TOAST处理，否则不处理。
```c++
// 如果元组大小超过了阈值或者包含了来自其他关系已经进行过toast处理的外部属性，则进行toast处理
static HeapTuple heap_prepare_insert(Relation relation, HeapTuple tup, TransactionId xid, CommandId cid, int options)
{
	// ...

	if (relation->rd_rel->relkind != RELKIND_RELATION &&
		relation->rd_rel->relkind != RELKIND_MATVIEW)
	{
		/* toast table entries should never be recursively toasted */
		Assert(!HeapTupleHasExternal(tup));
		return tup;
	}
	else if (HeapTupleHasExternal(tup) || tup->t_len > TOAST_TUPLE_THRESHOLD)
		return heap_toast_insert_or_update(relation, tup, NULL, options);
	else
		return tup;
}
```
调用栈如下：
```c++
heap_toast_insert_or_update(Relation rel, HeapTuple newtup, HeapTuple oldtup, int options) (src\backend\access\heap\heaptoast.c:98)
heap_prepare_insert(Relation relation, HeapTuple tup, TransactionId xid, CommandId cid, int options) (src\backend\access\heap\heapam.c:2323)
heap_insert(Relation relation, HeapTuple tup, CommandId cid, int options, BulkInsertState bistate) (src\backend\access\heap\heapam.c:2117)
heapam_tuple_insert(Relation relation, TupleTableSlot * slot, CommandId cid, int options, BulkInsertState bistate) (src\backend\access\heap\heapam_handler.c:252)
table_tuple_insert(Relation rel, TupleTableSlot * slot, CommandId cid, int options, struct BulkInsertStateData * bistate) (src\include\access\tableam.h:1374)
ExecInsert(ModifyTableState * mtstate, ResultRelInfo * resultRelInfo, TupleTableSlot * slot, TupleTableSlot * planSlot, EState * estate, _Bool canSetTag) (src\backend\executor\nodeModifyTable.c:1031)
```
我们详细看一下`heap_toast_insert_or_update`函数的实现逻辑：
```c++
HeapTuple heap_toast_insert_or_update(Relation rel, HeapTuple newtup, HeapTuple oldtup, int options)
{
	HeapTuple	result_tuple;
	TupleDesc	tupleDesc;
	int			numAttrs;
	bool		toast_isnull[MaxHeapAttributeNumber];
	Datum		toast_values[MaxHeapAttributeNumber];
	ToastTupleContext ttc;
	
	// 将元组数据提取到values数组中		
	heap_deform_tuple(newtup, tupleDesc, toast_values, toast_isnull);
	
	// 初始化ToastTupleContext
	toast_tuple_init(&ttc);

	/*
	 * 压缩和线外存储直到元组长度符合要求
	 * 1. 对EXTENDED的属性进行压缩，将非常大的EXTENED或EXTERNAL的属性进行线外存储
	 * 2. 将EXTENDED或EXTERNAL的属性进行线外存储
	 * 3. 对MAIN的属性进行压缩
	 * 4. 如果还不满足则将MAIN的属性进行线外存储
	 */

	// 计算元组的大小，如果元组大小大于阈值，就一直进行，直到其他条件不满足
	// 1. 对EXTENDED的属性进行压缩，将非常大的EXTENED或EXTERNAL的属性进行线外存储
	while (heap_compute_data_size(tupleDesc, toast_values, toast_isnull) > maxDataLen)
	{
		// 找到最大的属性
		int	biggest_attno = toast_tuple_find_biggest_attribute(&ttc, true, false);
		if (biggest_attno < 0)
			break;

		/* 如果属性为EXTENDED，则先尝试进行压缩 */
		if (TupleDescAttr(tupleDesc, biggest_attno)->attstorage == TYPSTORAGE_EXTENDED)
			toast_tuple_try_compression(&ttc, biggest_attno);
		else
		{
			/* 如果是EXTERNAL, 则不进行压缩，跳过压缩 */
			toast_attr[biggest_attno].tai_colflags |= TOASTCOL_INCOMPRESSIBLE;
		}

		/* 如果该属性（压缩后，如有）本身就超过了阈值，则立即进行线外存储
		 * 这样可以避免在只有一个长字段和几个短字段的元组时，对其他字段进行无意义的压缩 */
		if (toast_attr[biggest_attno].tai_size > maxDataLen && rel->rd_rel->reltoastrelid != InvalidOid)
			toast_tuple_externalize(&ttc, biggest_attno, options);
	}

	/* 2. 对EXTENDED或EXTERNAL的属性进行线外存储 */
	while (heap_compute_data_size(tupleDesc,toast_values, toast_isnull) > maxDataLen && rel->rd_rel->reltoastrelid != InvalidOid)
	{
		int	biggest_attno = toast_tuple_find_biggest_attribute(&ttc, false, false);
		if (biggest_attno < 0)
			break;
		toast_tuple_externalize(&ttc, biggest_attno, options);
	}

	/* 3. 对MAIN属性进行压缩 */
	while (heap_compute_data_size(tupleDesc, toast_values, toast_isnull) > maxDataLen)
	{
		int	biggest_attno = toast_tuple_find_biggest_attribute(&ttc, true, true);
		if (biggest_attno < 0)
			break;

		toast_tuple_try_compression(&ttc, biggest_attno);
	}

	/* 4. 如果还没有满足要求，则对MAIN进行线外存储 */
	while (heap_compute_data_size(tupleDesc, toast_values, toast_isnull) > maxDataLen && rel->rd_rel->reltoastrelid != InvalidOid)
	{
		int	biggest_attno = toast_tuple_find_biggest_attribute(&ttc, false, true);
		if (biggest_attno < 0)
			break;

		toast_tuple_externalize(&ttc, biggest_attno, options);
	}

	// 如果需要进行线外存储，则构造新的tuple
	if ((ttc.ttc_flags & TOAST_NEEDS_CHANGE) != 0)
	{
		// ...
		heap_fill_tuple(tupleDesc,toast_values,toast_isnull,(char *) new_data + new_header_len,new_data_len,&(new_data->t_infomask),((ttc.ttc_flags & TOAST_HAS_NULLS) != 0) ?new_data->t_bits : NULL);
	}
	else
		result_tuple = newtup;
	
	toast_tuple_cleanup(&ttc);
	return result_tuple;
}
```
那么具体的怎么将线外存储的数据存储到toast表中的呢？我们看一下`toast_tuple_externalize`函数的实现：
```c++
void toast_tuple_externalize(ToastTupleContext *ttc, int attribute, int options)
{
	Datum	   *value = &ttc->ttc_values[attribute];
	Datum		old_value = *value;
	ToastAttrInfo *attr = &ttc->ttc_attr[attribute];

	attr->tai_colflags |= TOASTCOL_IGNORE;
	*value = toast_save_datum(ttc->ttc_rel, old_value, attr->tai_oldexternal, options);
	if ((attr->tai_colflags & TOASTCOL_NEEDS_FREE) != 0)
		pfree(DatumGetPointer(old_value));
	attr->tai_colflags |= TOASTCOL_NEEDS_FREE;
	ttc->ttc_flags |= (TOAST_NEEDS_CHANGE | TOAST_NEEDS_FREE);
}
```
其实现具体在`toast_save_datum`函数，将一个大字段分割成片（chunk），插入到toast表，并返回一个外部存储指针作为新的字段值
```c++
Datum toast_save_datum(Relation rel, Datum value, struct varlena *oldexternal, int options)
{
	// ...
	struct varatt_external toast_pointer;
	union
	{
		struct varlena hdr;
		/* this is to make the union big enough for a chunk: */
		char		data[TOAST_MAX_CHUNK_SIZE + VARHDRSZ];
		/* ensure union is aligned well enough: */
		int32		align_it;
	}			chunk_data;

	/* 打开toast表以及索引 */
	toastrel = table_open(rel->rd_rel->reltoastrelid, RowExclusiveLock);
	validIndex = toast_open_indexes(toastrel, RowExclusiveLock, &toastidxs, &num_indexes);

	// ...

	/* 设置value OID，  */
	if (!OidIsValid(rel->rd_toastoid))
	{
		/* normal case: just choose an unused OID */
		toast_pointer.va_valueid = GetNewOidWithIndex(toastrel, RelationGetRelid(toastidxs[validIndex]), (AttrNumber) 1);
	}
	else
	{
		// ...
		if (toast_pointer.va_valueid == InvalidOid)
		{
			// 找到一个未使用的不与已存在的value OID冲突的OID
			do
			{
				toast_pointer.va_valueid = GetNewOidWithIndex(toastrel, RelationGetRelid(toastidxs[validIndex]), (AttrNumber) 1);
			} while (toastid_valueid_exists(rel->rd_toastoid, toast_pointer.va_valueid));
		}
	}

	/* Initialize constant parts of the tuple data */
	t_values[0] = ObjectIdGetDatum(toast_pointer.va_valueid);
	t_values[2] = PointerGetDatum(&chunk_data);

	/* 分片chunk插入toast表中，一个分片chunk就是toast表的一个tuple */
	while (data_todo > 0)
	{
		int			i;

		/* Calculate the size of this chunk */
		chunk_size = Min(TOAST_MAX_CHUNK_SIZE, data_todo);

		/* 构造一个分片chunk tuple */
		t_values[1] = Int32GetDatum(chunk_seq++);
		SET_VARSIZE(&chunk_data, chunk_size + VARHDRSZ);
		memcpy(VARDATA(&chunk_data), data_p, chunk_size);
		toasttup = heap_form_tuple(toasttupDesc, t_values, t_isnull);

		heap_insert(toastrel, toasttup, mycid, options, NULL);	/* 插入toast表 */

		// 插入索引
		for (i = 0; i < num_indexes; i++)
		{
			/* Only index relations marked as ready can be updated */
			if (toastidxs[i]->rd_index->indisready)
				index_insert(toastidxs[i], t_values, t_isnull, &(toasttup->t_self), toastrel, toastidxs[i]->rd_index->indisunique ? UNIQUE_CHECK_YES : UNIQUE_CHECK_NO, false, NULL);
		}

		/* Free memory */
		heap_freetuple(toasttup);

		/* 移动到下一个分片chunk */
		data_todo -= chunk_size;
		data_p += chunk_size;
	}

	/* 关闭索引以及toast表 */
	toast_close_indexes(toastidxs, num_indexes, NoLock);
	table_close(toastrel, NoLock);

	/* 构造toast指针，返回 */
	result = (struct varlena *) palloc(TOAST_POINTER_SIZE);
	SET_VARTAG_EXTERNAL(result, VARTAG_ONDISK);
	memcpy(VARDATA_EXTERNAL(result), &toast_pointer, sizeof(toast_pointer));

	return PointerGetDatum(result);
}
```

#### 读取toast数据
在查询涉及被toast过的数据时，需要读取toast数据并进行恢复（解压缩、将分片进行组装为完整数据）。读取的过程其实就是插入的逆过程，根据toast属性中存储的toast指针，从toast表中读取所有分片，再根据序号将分片组装为完整数据，再判断是否为经过压缩的数据，如果是则需要进行解压缩。
具体可分析`detoast_attr`函数。这里不在详细进行分析。

```c++
toast_decompress_datum(struct varlena * attr) (src\backend\access\common\detoast.c:485)
detoast_attr(struct varlena * attr) (src\backend\access\common\detoast.c:173)
pg_detoast_datum_packed(struct varlena * datum) (src\backend\utils\fmgr\fmgr.c:1757)
text_to_cstring(const text * t) (src\backend\utils\adt\varlena.c:225)
textout(FunctionCallInfo fcinfo) (src\backend\utils\adt\varlena.c:574)
FunctionCall1Coll(FmgrInfo * flinfo, Oid collation, Datum arg1) (src\backend\utils\fmgr\fmgr.c:1138)
OutputFunctionCall(FmgrInfo * flinfo, Datum val) (src\backend\utils\fmgr\fmgr.c:1575)
printtup(TupleTableSlot * slot, DestReceiver * self) (src\backend\access\common\printtup.c:357)
ExecutePlan(QueryDesc * queryDesc, CmdType operation, _Bool sendTuples, uint64 numberTuples, ScanDirection direction, DestReceiver * dest) (src\backend\executor\execMain.c:1586)
standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\execMain.c:360)
ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\execMain.c:309)
PortalRunSelect(Portal portal, _Bool forward, long count, DestReceiver * dest) (src\backend\tcop\pquery.c:919)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:763)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1217)
```