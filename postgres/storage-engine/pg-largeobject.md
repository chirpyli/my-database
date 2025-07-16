## PostgreSQL大对象存储机制
PostgreSQL中如何存储大对象呢？比如视频、图片等。前面我们分析过TOAST机制，即超大字段存储机制，从原理上来说，理解了TOAST机制，PostgreSQL的大对象存储机制就更容易理解了。其原理是一致的。

### 大对象存储实现原理
在前面分析的TOAST机制中，当数据大于指定大小时，会进行TOAST处理，将数据存储在TOAST表中。对于大对象存储原理基本相同，不同的是所有的大对象都存储在系统表pg_largeobject中。对于一个超大的对象，其大小会超多堆表页大小，所以需要将对象分块进行存储，切分时给每个分块一个编号，每个小块作为一个元组，存储在pg_largeobject表中，一个分块元组大小为2KB，这样一个页可以存储4个元组。
```sql
-- 查看pg_largeobject表结构
postgres=# \d pg_largeobject;
         Table "pg_catalog.pg_largeobject"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 loid   | oid     |           | not null |     -- 大对象OID，用于分別是那个大对象
 pageno | integer |           | not null |     -- 页码，类似于TOAST机制中的chunk_seq，大对象的分块顺序编号
 data   | bytea   |           | not null |     -- 大对象实际数据（当前块的数据，被切分为多块）
Indexes:
    "pg_largeobject_loid_pn_index" PRIMARY KEY, btree (loid, pageno)  -- 索引
```
在读取大对象时，会根据大对象的唯一OID去从pg_largeobject表中查询出所有分块的页码和数据，然后按照页码进行排序，将所有分块的数据拼接成完整的大对象数据。

对于大对象存储的数据的最大的大小，我们可以计算一下，pageno字段的范围是`0-2^32-1`，那么一个分块的大小为2KB，一个大对象最大的大小=`2^32*2KB=4TB`。也就是说，一个大对象最多可以存储4TB的数据。

为什么一个块设置为2KB呢？PostgreSQL给出了如下的描述：
```c++
/*
 * 每个大对象的“页面”（元组）可以容纳的数据大小
 * 我们本可以将其设置为 BLCKSZ 减去一些开销的值，但将其设为较小的值似乎更好，
 * 这样在页面元组被更新时不会占用太多空间。注意，这个值是有意设置得足够大，
 * 以触发元组的 toaster（自动压缩机制），这样我们会尝试对页面元组进行内联压缩。
 *（但除非用户为 pg_largeobject 创建 toast 表，否则它们不会被移出主表。）
 * 另外，将页面大小设为 2 的幂似乎是明智之举，因为客户端通常会以 2 的幂大小
 * 发送数据块。这可以避免因部分页面写入导致的不必要的元组更新。
 * 注意：更改 LOBLKSIZE 需要重新初始化数据库（initdb）。
 */
#define LOBLKSIZE		(BLCKSZ / 4)

// 定义了大对象最大的大小
#define MAX_LARGE_OBJECT_SIZE	((int64) INT_MAX * LOBLKSIZE)
```


### 大对象操作示例
这里我们通过SQL来演示大对象的操作，实际应用时可通过libpq客户端调用相应的API来实现。
```sql
-- 建表
postgres=# create table bigobj(name text, data oid);
CREATE TABLE
-- 插入一个大对象
postgres=# insert into bigobj values ('initdb',lo_import('/home/postgres/psql/bin/initdb'));
INSERT 0 1
-- 查询
postgres=# select * from bigobj;
  name  | data  
--------+-------
 initdb | 20123
(1 row)
-- 查看pg_largeobject系统表中的数据，大对象存储在pg_largeobject系统表中，被切分为多个块
postgres=# select loid,pageno from pg_largeobject;
 loid  | pageno 
-------+--------
 20123 |      0
 20123 |      1
 20123 |      2
 20123 |      3
 20123 |      4
 20123 |      5

-- 导出大对象
postgres=# select lo_export(bigobj.data,'/home/postgres/initdb') from bigobj where name = 'initdb';
 lo_export 
-----------
         1
(1 row)
-- 可以看到导出的initdb文件
postgres@slpc:~$ ls
initdb                                           psql
```
实际上，服务端将大对象存储在了系统表pg_largeobject中，我们看一下
```sql
-- 查看initdb大对象的大小，大小为367272字节
postgres@slpc:~$ ll initdb 
-rw-r--r--. 1 postgres postgres 367272  7月 16 17:48 initdb

-- 查询大对象，大对象被切分为了180块， 每块大小为2k，计算 367272/2048 = 179.33203125，也就是说需要180块去存储
postgres=# select count(*) from pg_largeobject where loid = 20123;
 count 
-------
   180
(1 row)

```

使用时需要注意，需要用户自己管理对大对象OID的引用，删除包含大对象OID的记录时，不会自动删除pg_largeobject中的大对象，需要用户显示调用`lo_unlink`来删除pg_largeobject中的大对象。
```sql
-- 删除包含大对象OID的记录
postgres=# delete from bigobj ;
DELETE 1
-- 此时大对象依旧存储在pg_largeobject中，并没有被删除，如果需要删除，需用用户额外处理
postgres=# select count(*) from pg_largeobject where loid=20123;
 count 
-------
   180
(1 row)
-- 删除大对象
postgres=# select lo_unlink(20123);
 lo_unlink 
-----------
         1
(1 row)
-- 可以观察到大对象被删除了
postgres=# select count(*) from pg_largeobject where loid=20123;
 count 
-------
     0
(1 row)

```

### 大对象与TOAST的比较
在PostgreSQL中，大对象和TOAST都可以存储大数据（值得是数据的大小比较大的数据），两者的核心原理是基本相同的，但是在实际应用是也存在着差异。比如，前面提到的，对于TOAST机制来说，它对用户是透明的，用户无需显式进行操作，当用户创建含有变长数据类型的列时，PostgreSQL会自动为表创建一个TOAST表，当数据超过阈值时会自动触发TOAST机制，进行压缩或线外存储至toast表中，而对于大对象，需要用户显式调用SQL或者libpq API进行操作。对于TOAST机制来说，其存储的对象每次进行操作是完整的读取与写入整个对象，而对于大对象来说，其可以分块操作，其操作的最小单元是块，每次操作时不必完全解析整个大对象，支持流式读写。


### 源码分析
我们以将一个大对象写入为例，分析其实现过程。大对象的写入，其实就是将大对象切分成块然后插入到表pg_largeobject中的过程。期间需要创建大对象OID，构建分块元组数据。
```c++
int inv_write(LargeObjectDesc *obj_desc, const char *buf, int nbytes)
{
	int			nwritten = 0;
	int			n;
	int			off;
	int			len;
	int32		pageno = (int32) (obj_desc->offset / LOBLKSIZE);
	ScanKeyData skey[2];
	SysScanDesc sd;
	HeapTuple	oldtuple;
	Form_pg_largeobject olddata;
	bool		neednextpage;
	bytea	   *datafield;
	bool		pfreeit;
	union
	{
		bytea		hdr;
		/* this is to make the union big enough for a LO data chunk: */
		char		data[LOBLKSIZE + VARHDRSZ];
		/* ensure union is aligned well enough: */
		int32		align_it;
	}			workbuf;
	char	   *workb = VARDATA(&workbuf.hdr);
	HeapTuple	newtup;
	Datum		values[Natts_pg_largeobject];
	bool		nulls[Natts_pg_largeobject];
	bool		replace[Natts_pg_largeobject];
	
	open_lo_relation();         // 打开大对象表

    CatalogIndexState indstate = CatalogOpenIndexes(lo_heap_r);

    // 索引键
	ScanKeyInit(&skey[0],
				Anum_pg_largeobject_loid,
				BTEqualStrategyNumber, F_OIDEQ,
				ObjectIdGetDatum(obj_desc->id));

	ScanKeyInit(&skey[1],
				Anum_pg_largeobject_pageno,
				BTGreaterEqualStrategyNumber, F_INT4GE,
				Int32GetDatum(pageno));

	sd = systable_beginscan_ordered(lo_heap_r, lo_index_r,
									obj_desc->snapshot, 2, skey);

	oldtuple = NULL;
	olddata = NULL;
	neednextpage = true;

    // 循环执行，直到所有数据完成写入
	while (nwritten < nbytes)
	{
		/*
		 * If possible, get next pre-existing page of the LO.  We expect the
		 * indexscan will deliver these in order --- but there may be holes.
		 */
		if (neednextpage)
		{
			if ((oldtuple = systable_getnext_ordered(sd, ForwardScanDirection)) != NULL)
			{
				if (HeapTupleHasNulls(oldtuple))	/* paranoia */
					elog(ERROR, "null field found in pg_largeobject");
				olddata = (Form_pg_largeobject) GETSTRUCT(oldtuple);
				Assert(olddata->pageno >= pageno);
			}
			neednextpage = false;
		}

		/*
		 * If we have a pre-existing page, see if it is the page we want to
		 * write, or a later one.
		 */
		if (olddata != NULL && olddata->pageno == pageno)
		{
			/*
			 * Update an existing page with fresh data.
			 *
			 * First, load old data into workbuf
			 */
			getdatafield(olddata, &datafield, &len, &pfreeit);
			memcpy(workb, VARDATA(datafield), len);
			if (pfreeit)
				pfree(datafield);

			/*
			 * Fill any hole
			 */
			off = (int) (obj_desc->offset % LOBLKSIZE);
			if (off > len)
				MemSet(workb + len, 0, off - len);

			/*
			 * Insert appropriate portion of new data
			 */
			n = LOBLKSIZE - off;
			n = (n <= (nbytes - nwritten)) ? n : (nbytes - nwritten);
			memcpy(workb + off, buf + nwritten, n);
			nwritten += n;
			obj_desc->offset += n;
			off += n;
			/* compute valid length of new page */
			len = (len >= off) ? len : off;
			SET_VARSIZE(&workbuf.hdr, len + VARHDRSZ);

			/*
			 * Form and insert updated tuple
			 */
			memset(values, 0, sizeof(values));
			memset(nulls, false, sizeof(nulls));
			memset(replace, false, sizeof(replace));
			values[Anum_pg_largeobject_data - 1] = PointerGetDatum(&workbuf);
			replace[Anum_pg_largeobject_data - 1] = true;
			newtup = heap_modify_tuple(oldtuple, RelationGetDescr(lo_heap_r),
									   values, nulls, replace);
			CatalogTupleUpdateWithInfo(lo_heap_r, &newtup->t_self, newtup,
									   indstate);
			heap_freetuple(newtup);

			/*
			 * We're done with this old page.
			 */
			oldtuple = NULL;
			olddata = NULL;
			neednextpage = true;
		}
		else
		{
			/*
			 * Write a brand new page.
			 *
			 * First, fill any hole
			 */
			off = (int) (obj_desc->offset % LOBLKSIZE);
			if (off > 0)
				MemSet(workb, 0, off);

			/*
			 * Insert appropriate portion of new data
			 */
			n = LOBLKSIZE - off;
			n = (n <= (nbytes - nwritten)) ? n : (nbytes - nwritten);
			memcpy(workb + off, buf + nwritten, n);
			nwritten += n;
			obj_desc->offset += n;
			/* compute valid length of new page */
			len = off + n;
			SET_VARSIZE(&workbuf.hdr, len + VARHDRSZ);

			/*
			 * Form and insert updated tuple
			 */
			memset(values, 0, sizeof(values));
			memset(nulls, false, sizeof(nulls));
			values[Anum_pg_largeobject_loid - 1] = ObjectIdGetDatum(obj_desc->id);
			values[Anum_pg_largeobject_pageno - 1] = Int32GetDatum(pageno);
			values[Anum_pg_largeobject_data - 1] = PointerGetDatum(&workbuf);
			newtup = heap_form_tuple(lo_heap_r->rd_att, values, nulls);
			CatalogTupleInsertWithInfo(lo_heap_r, newtup, indstate);
			heap_freetuple(newtup);
		}
		pageno++;
	}

	systable_endscan_ordered(sd);

	CatalogCloseIndexes(indstate);

	/*
	 * Advance command counter so that my tuple updates will be seen by later
	 * large-object operations in this transaction.
	 */
	CommandCounterIncrement();

	return nwritten;
}
```