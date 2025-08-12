### PostgreSQL外存管理
数据库最终都是持久化存储的（除了内存数据库等），持久化就要将数据从内存Buffer落盘到外存。这里分析一下PostgreSQL中外存管理部分的内容。源码在`src/backend/storage/smgr`这一部分。


#### README
建议首先阅读一下`src/backend/storage/smgr/README`里的内容。中文翻译可参考文章[postgres外存管理之smgr](https://zhuanlan.zhihu.com/p/646825804)
```
src/backend/storage/smgr/README

Storage Managers
================

In the original Berkeley Postgres system, there were several storage managers,
of which only the "magnetic disk" manager remains. 
 The "magnetic disk" manager is itselfseriously misnamed, because actually 
 it supports any kind of device for which the operating system provides standard filesystem operations; which
these days is pretty much everything of interest.  However, we retain the
notion of a storage manager switch in case anyone ever wants to reintroduce
other kinds of storage managers.  Removing the switch layer would save
nothing noticeable anyway, since storage-access operations are surely far
more expensive than one extra layer of C function calls.

In Berkeley Postgres each relation was tagged with the ID of the storage
manager to use for it.  This is gone.  It would be probably more reasonable
to associate storage managers with tablespaces, should we ever re-introduce
multiple storage managers into the system catalogs.

The files in this directory, and their contents, are

    smgr.c	The storage manager switch dispatch code.  The routines in
		this file call the appropriate storage manager to do storage
		accesses requested by higher-level code.  smgr.c also manages
		the file handle cache (SMgrRelation table).

    md.c	The "magnetic disk" storage manager, which is really just
		an interface to the kernel's filesystem operations.

Note that md.c in turn relies on src/backend/storage/file/fd.c.


Relation Forks
==============

Since 8.4, a single smgr relation can be comprised of multiple physical
files, called relation forks. This allows storing additional metadata like
Free Space information in additional forks, which can be grown and truncated
independently of the main data file, while still treating it all as a single
physical relation in system catalogs.

It is assumed that the main fork, fork number 0 or MAIN_FORKNUM, always
exists. Fork numbers are assigned in src/include/common/relpath.h.
Functions in smgr.c and md.c take an extra fork number argument, in addition
to relfilenode and block number, to identify which relation fork you want to
access. Since most code wants to access the main fork, a shortcut version of
ReadBuffer that accesses MAIN_FORKNUM is provided in the buffer manager for
convenience.
```
截取README中比较重要的两句：
- 磁盘管理器不仅限于管理磁盘，实际上它支持任何设备，只要操作系统为该设备实现了标准文件系统操作接口。
- 虽然PG存储管理器目前仅有磁盘管理器，但依然保留了存储管理器（smgr）这个中间层，以便引入其他类型的存储管理器，删除中间层实际上并没有明显的收益，因为存储访问操作肯定远比多一层函数调用贵的多。

#### 存储管理器

实现了存储管理器分发调度接口，相当于是存储管理的一层抽象。所有对文件系统的操作都是由这里进行分发。我们看一下smgr.h中的函数声明：
```c++
extern void smgrinit(void);
extern SMgrRelation smgropen(RelFileNode rnode, BackendId backend);
extern bool smgrexists(SMgrRelation reln, ForkNumber forknum);
extern void smgrsetowner(SMgrRelation *owner, SMgrRelation reln);
extern void smgrclearowner(SMgrRelation *owner, SMgrRelation reln);
extern void smgrclose(SMgrRelation reln);
extern void smgrcloseall(void);
extern void smgrclosenode(RelFileNodeBackend rnode);
extern void smgrrelease(SMgrRelation reln);
extern void smgrreleaseall(void);
extern void smgrcreate(SMgrRelation reln, ForkNumber forknum, bool isRedo);
extern void smgrdosyncall(SMgrRelation *rels, int nrels);
extern void smgrdounlinkall(SMgrRelation *rels, int nrels, bool isRedo);
extern void smgrextend(SMgrRelation reln, ForkNumber forknum,
					   BlockNumber blocknum, char *buffer, bool skipFsync);
extern bool smgrprefetch(SMgrRelation reln, ForkNumber forknum,
						 BlockNumber blocknum);
extern void smgrread(SMgrRelation reln, ForkNumber forknum,
					 BlockNumber blocknum, char *buffer);
extern void smgrwrite(SMgrRelation reln, ForkNumber forknum,
					  BlockNumber blocknum, char *buffer, bool skipFsync);
extern void smgrwriteback(SMgrRelation reln, ForkNumber forknum,
						  BlockNumber blocknum, BlockNumber nblocks);
extern BlockNumber smgrnblocks(SMgrRelation reln, ForkNumber forknum);
extern BlockNumber smgrnblocks_cached(SMgrRelation reln, ForkNumber forknum);
extern void smgrtruncate(SMgrRelation reln, ForkNumber *forknum,
						 int nforks, BlockNumber *nblocks);
extern void smgrimmedsync(SMgrRelation reln, ForkNumber forknum);
extern void AtEOXact_SMgr(void);
extern bool ProcessBarrierSmgrRelease(void);
```
也就是说数据库与外存进行交互，都是通过这些接口实现的。我们以bgwriter为例，bgwriter需要将缓冲区中的页进行刷盘，我们看一下它的源码：
```c++
BackgroundWriterMain(void)  // bgwriter进程主流程
{
    //...
    if (sigsetjmp(local_sigjmp_buf, 1) != 0) // 错误处理
    {
        /* Close all open files after any error. */
		smgrcloseall();
    }

    for (;;)
    {
        BgBufferSync(&wb_context);  // 刷脏页落盘
        --> SyncOneBuffer(next_to_clean, true, wb_context);
            --> FlushBuffer(bufHdr, NULL);  // 具体脏页落盘的实现
                {
                    /* Find smgr relation for buffer */
                    if (reln == NULL)
                        reln = smgropen(buf->tag.rnode, InvalidBackendId);
                    // ...
                    /* bufToWrite is either the shared buffer or a copy, as appropriate.*/
                    smgrwrite(reln, buf->tag.forkNum, buf->tag.blockNum, bufToWrite, false);
                    // ...
                }
    }
    // ...
}
```
可以看到调用`smgrwrite`写入磁盘。


#### smgr.c

下面就是对存储管理抽象接口的定义，C语言中没有虚函数或者接口的概念，以函数指针的方式实现。
```c++
/*
 * This struct of function pointers defines the API between smgr.c and
 * any individual storage manager module. 
 */
typedef struct f_smgr
{
	void		(*smgr_init) (void);	/* may be NULL */
	void		(*smgr_shutdown) (void);	/* may be NULL */
	void		(*smgr_open) (SMgrRelation reln);
	void		(*smgr_close) (SMgrRelation reln, ForkNumber forknum);
	void		(*smgr_create) (SMgrRelation reln, ForkNumber forknum,
								bool isRedo);
	bool		(*smgr_exists) (SMgrRelation reln, ForkNumber forknum);
	void		(*smgr_unlink) (RelFileNodeBackend rnode, ForkNumber forknum,
								bool isRedo);
	void		(*smgr_extend) (SMgrRelation reln, ForkNumber forknum,
								BlockNumber blocknum, char *buffer, bool skipFsync);
	bool		(*smgr_prefetch) (SMgrRelation reln, ForkNumber forknum,
								  BlockNumber blocknum);
	void		(*smgr_read) (SMgrRelation reln, ForkNumber forknum,
							  BlockNumber blocknum, char *buffer);
	void		(*smgr_write) (SMgrRelation reln, ForkNumber forknum,
							   BlockNumber blocknum, char *buffer, bool skipFsync);
	void		(*smgr_writeback) (SMgrRelation reln, ForkNumber forknum,
								   BlockNumber blocknum, BlockNumber nblocks);
	BlockNumber (*smgr_nblocks) (SMgrRelation reln, ForkNumber forknum);
	void		(*smgr_truncate) (SMgrRelation reln, ForkNumber forknum,
								  BlockNumber nblocks);
	void		(*smgr_immedsync) (SMgrRelation reln, ForkNumber forknum);
} f_smgr;
```

具体实现，PG中仅实现了磁盘管理，具体函数的实现是在md.c中实现的。
```c++
static const f_smgr smgrsw[] = {
	/* magnetic disk */
	{
		.smgr_init = mdinit,
		.smgr_shutdown = NULL,
		.smgr_open = mdopen,
		.smgr_close = mdclose,
		.smgr_create = mdcreate,
		.smgr_exists = mdexists,
		.smgr_unlink = mdunlink,
		.smgr_extend = mdextend,
		.smgr_prefetch = mdprefetch,
		.smgr_read = mdread,
		.smgr_write = mdwrite,
		.smgr_writeback = mdwriteback,
		.smgr_nblocks = mdnblocks,
		.smgr_truncate = mdtruncate,
		.smgr_immedsync = mdimmedsync,
	}
};
```
仅实现了磁盘管理，所以存储管理器数组长度为1.
```c++ 
static const int NSmgr = lengthof(smgrsw);
```

我们先看一下，存储管理器的初始化与关闭，可以smgr只是一层抽象接口，最终实际调用执行的是具体的磁盘管理器。
```c++
/*
 *	smgrinit(), smgrshutdown() -- Initialize or shut down storage
 *								  managers.
 *
 * Note: smgrinit is called during backend startup (normal or standalone
 * case), *not* during postmaster start.  Therefore, any resources created
 * here or destroyed in smgrshutdown are backend-local.
 */
void smgrinit(void)
{
	int			i;

	for (i = 0; i < NSmgr; i++)
	{
		if (smgrsw[i].smgr_init)
			smgrsw[i].smgr_init();
	}

	/* register the shutdown proc */
	on_proc_exit(smgrshutdown, 0);
}

/*
 * on_proc_exit hook for smgr cleanup during backend shutdown
 */
static void smgrshutdown(int code, Datum arg)
{
	int			i;

	for (i = 0; i < NSmgr; i++)
	{
		if (smgrsw[i].smgr_shutdown)
			smgrsw[i].smgr_shutdown();
	}
}
```

我们列出几个比较重要的实现，其他的可参考PG源码smgr.c

###### smgropen
打开一个表对象，先查找表是否已打开，如果没有，则调用具体的磁盘管理器`smgrsw[reln->smgr_which].smgr_open(reln);`打开这个表。
```c++
/*	smgropen() -- Return an SMgrRelation object, creating it if need be.
 *		This does not attempt to actually open the underlying file. */
SMgrRelation smgropen(RelFileNode rnode, BackendId backend)
{
	RelFileNodeBackend brnode;
	SMgrRelation reln;
	bool		found;

	if (SMgrRelationHash == NULL)
	{
		/* First time through: initialize the hash table */
		HASHCTL		ctl;

		ctl.keysize = sizeof(RelFileNodeBackend);
		ctl.entrysize = sizeof(SMgrRelationData);
		SMgrRelationHash = hash_create("smgr relation table", 400,
									   &ctl, HASH_ELEM | HASH_BLOBS);
		dlist_init(&unowned_relns);
	}

	/* Look up or create an entry */
	brnode.node = rnode;
	brnode.backend = backend;
	reln = (SMgrRelation) hash_search(SMgrRelationHash,
									  (void *) &brnode,
									  HASH_ENTER, &found);

	/* Initialize it if not present before */
	if (!found)
	{
		/* hash_search already filled in the lookup key */
		reln->smgr_owner = NULL;
		reln->smgr_targblock = InvalidBlockNumber;
		for (int i = 0; i <= MAX_FORKNUM; ++i)
			reln->smgr_cached_nblocks[i] = InvalidBlockNumber;
		reln->smgr_which = 0;	/* we only have md.c at present */

		/* implementation-specific initialization */
		smgrsw[reln->smgr_which].smgr_open(reln);

		/* it has no owner yet */
		dlist_push_tail(&unowned_relns, &reln->node);
	}

	return reln;
}

/*  mdopen() -- Initialize newly-opened relation */
void mdopen(SMgrRelation reln)
{
	/* mark it not open */
	for (int forknum = 0; forknum <= MAX_FORKNUM; forknum++)
		reln->md_num_open_segs[forknum] = 0;
}

```

###### smgrcreate
创建一个新的表
```c++
/*
 *	smgrcreate() -- Create a new relation.
 *
 *		Given an already-created (but presumably unused) SMgrRelation,
 *		cause the underlying disk file or other storage for the fork
 *		to be created.
 */
void
smgrcreate(SMgrRelation reln, ForkNumber forknum, bool isRedo)
{
	smgrsw[reln->smgr_which].smgr_create(reln, forknum, isRedo);
}

/*
 *	mdcreate() -- Create a new relation on magnetic disk.
 *
 * If isRedo is true, it's okay for the relation to exist already.
 */
void mdcreate(SMgrRelation reln, ForkNumber forkNum, bool isRedo)
{
	MdfdVec    *mdfd;
	char	   *path;
	File		fd;

	if (isRedo && reln->md_num_open_segs[forkNum] > 0)
		return;					/* created and opened already... */

	Assert(reln->md_num_open_segs[forkNum] == 0);

	/*
	 * We may be using the target table space for the first time in this
	 * database, so create a per-database subdirectory if needed.
	 *
	 * XXX this is a fairly ugly violation of module layering, but this seems
	 * to be the best place to put the check.  Maybe TablespaceCreateDbspace
	 * should be here and not in commands/tablespace.c?  But that would imply
	 * importing a lot of stuff that smgr.c oughtn't know, either.
	 */
	TablespaceCreateDbspace(reln->smgr_rnode.node.spcNode,
							reln->smgr_rnode.node.dbNode,
							isRedo);

	path = relpath(reln->smgr_rnode, forkNum);

	fd = PathNameOpenFile(path, O_RDWR | O_CREAT | O_EXCL | PG_BINARY);

	if (fd < 0)
	{
		int			save_errno = errno;

		if (isRedo)
			fd = PathNameOpenFile(path, O_RDWR | PG_BINARY);
		if (fd < 0)
		{
			/* be sure to report the error reported by create, not open */
			errno = save_errno;
			ereport(ERROR,
					(errcode_for_file_access(),
					 errmsg("could not create file \"%s\": %m", path)));
		}
	}

	pfree(path);

	_fdvec_resize(reln, forkNum, 1);
	mdfd = &reln->md_seg_fds[forkNum][0];
	mdfd->mdfd_vfd = fd;
	mdfd->mdfd_segno = 0;

	if (!SmgrIsTemp(reln))
		register_dirty_segment(reln, forkNum, mdfd);
}
```


###### smgrextend

```c++
/*
 *	smgrextend() -- Add a new block to a file.
 *
 *		The semantics are nearly the same as smgrwrite(): write at the
 *		specified position.  However, this is to be used for the case of
 *		extending a relation (i.e., blocknum is at or beyond the current
 *		EOF).  Note that we assume writing a block beyond current EOF
 *		causes intervening file space to become filled with zeroes.
 */
void smgrextend(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, char *buffer, bool skipFsync)
{
	smgrsw[reln->smgr_which].smgr_extend(reln, forknum, blocknum, buffer, skipFsync);

	/* Normally we expect this to increase nblocks by one, but if the cached
	 * value isn't as expected, just invalidate it so the next call asks the kernel. */
	if (reln->smgr_cached_nblocks[forknum] == blocknum)
		reln->smgr_cached_nblocks[forknum] = blocknum + 1;
	else
		reln->smgr_cached_nblocks[forknum] = InvalidBlockNumber;
}
```

###### smgrread
读表中的指定块到buffer中。数据存储在表文件中，表文件又被切分成若干segment，每个segment最大为1G，超过1G则创建一个新的segment，每个segment按8k一个块，分为很多个块Block，然后元组就存储在块中。读指定块的时候，要首先找到表，再找到表的segment，再找块在segment中偏移的位置，然后再读8k的数据块。
```c++
/*
 *	smgrread() -- read a particular block from a relation into the supplied
 *				  buffer.
 *
 *		This routine is called from the buffer manager in order to
 *		instantiate pages in the shared buffer cache.  All storage managers
 *		return pages in the format that POSTGRES expects. */
void smgrread(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, char *buffer)
{
	smgrsw[reln->smgr_which].smgr_read(reln, forknum, blocknum, buffer);
}

/* mdread() -- Read the specified block from a relation. */
void mdread(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, char *buffer)
{
	off_t		seekpos;
	int			nbytes;
	MdfdVec    *v;
    // 1. 获取指定segment文件，targetseg = blkno / ((BlockNumber) RELSEG_SIZE);
	v = _mdfd_getseg(reln, forknum, blocknum, false, EXTENSION_FAIL | EXTENSION_CREATE_RECOVERY);
    // 2. 获取块在segment中的偏移量
	seekpos = (off_t) BLCKSZ * (blocknum % ((BlockNumber) RELSEG_SIZE));
    // 3. 读块数据
	nbytes = FileRead(v->mdfd_vfd, buffer, BLCKSZ, seekpos, WAIT_EVENT_DATA_FILE_READ);

	if (nbytes != BLCKSZ)
	{
		if (nbytes < 0)
			ereport(ERROR,
					(errcode_for_file_access(),
					 errmsg("could not read block %u in file \"%s\": %m",
							blocknum, FilePathName(v->mdfd_vfd))));

		/*
		 * Short read: we are at or past EOF, or we read a partial block at
		 * EOF.  Normally this is an error; upper levels should never try to
		 * read a nonexistent block.  However, if zero_damaged_pages is ON or
		 * we are InRecovery, we should instead return zeroes without
		 * complaining.  This allows, for example, the case of trying to
		 * update a block that was later truncated away. */
		if (zero_damaged_pages || InRecovery)
			MemSet(buffer, 0, BLCKSZ);
		else
			ereport(ERROR,(errcode(ERRCODE_DATA_CORRUPTED), errmsg("could not read block %u in file \"%s\": read only %d of %d bytes",	blocknum, FilePathName(v->mdfd_vfd),nbytes, BLCKSZ)));
	}
}



```
###### smgrwrite
将buffer写入磁盘中, 用于更新表中现有的块，要扩展表，需要调用smgrextend。
```c++
/*
 *	smgrwrite() -- Write the supplied buffer out.
 *
 *		This is to be used only for updating already-existing blocks of a
 *		relation (ie, those before the current EOF).  To extend a relation,
 *		use smgrextend().
 *
 *		This is not a synchronous write -- the block is not necessarily
 *		on disk at return, only dumped out to the kernel.  However,
 *		provisions will be made to fsync the write before the next checkpoint.
 *
 *		skipFsync indicates that the caller will make other provisions to
 *		fsync the relation, so we needn't bother.  Temporary relations also
 *		do not require fsync. */
void smgrwrite(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, char *buffer, bool skipFsync)
{
	smgrsw[reln->smgr_which].smgr_write(reln, forknum, blocknum, buffer, skipFsync);
}

/*
 *	mdwrite() -- Write the supplied block at the appropriate location.
 *
 *		This is to be used only for updating already-existing blocks of a
 *		relation (ie, those before the current EOF).  To extend a relation,
 *		use mdextend().
 */
void mdwrite(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, char *buffer, bool skipFsync)
{
	off_t		seekpos;
	int			nbytes;
	MdfdVec    *v;

	v = _mdfd_getseg(reln, forknum, blocknum, skipFsync, EXTENSION_FAIL | EXTENSION_CREATE_RECOVERY);

	seekpos = (off_t) BLCKSZ * (blocknum % ((BlockNumber) RELSEG_SIZE));

	nbytes = FileWrite(v->mdfd_vfd, buffer, BLCKSZ, seekpos, WAIT_EVENT_DATA_FILE_WRITE);

	if (nbytes != BLCKSZ)
	{
		if (nbytes < 0)
			ereport(ERROR,
					(errcode_for_file_access(),
					 errmsg("could not write block %u in file \"%s\": %m",
							blocknum, FilePathName(v->mdfd_vfd))));
		/* short write: complain appropriately */
		ereport(ERROR,
				(errcode(ERRCODE_DISK_FULL),
				 errmsg("could not write block %u in file \"%s\": wrote only %d of %d bytes",
						blocknum,
						FilePathName(v->mdfd_vfd),
						nbytes, BLCKSZ),
				 errhint("Check free disk space.")));
	}

	if (!skipFsync && !SmgrIsTemp(reln))
		register_dirty_segment(reln, forknum, v);
}
```

#### md.c
磁盘管理器具体实现，
```c++

/* md storage manager functionality */
extern void mdinit(void);
extern void mdopen(SMgrRelation reln);
extern void mdclose(SMgrRelation reln, ForkNumber forknum);
extern void mdcreate(SMgrRelation reln, ForkNumber forknum, bool isRedo);
extern bool mdexists(SMgrRelation reln, ForkNumber forknum);
extern void mdunlink(RelFileNodeBackend rnode, ForkNumber forknum, bool isRedo);
extern void mdextend(SMgrRelation reln, ForkNumber forknum,
					 BlockNumber blocknum, char *buffer, bool skipFsync);
extern bool mdprefetch(SMgrRelation reln, ForkNumber forknum,
					   BlockNumber blocknum);
extern void mdread(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum,
				   char *buffer);
extern void mdwrite(SMgrRelation reln, ForkNumber forknum,
					BlockNumber blocknum, char *buffer, bool skipFsync);
extern void mdwriteback(SMgrRelation reln, ForkNumber forknum,
						BlockNumber blocknum, BlockNumber nblocks);
extern BlockNumber mdnblocks(SMgrRelation reln, ForkNumber forknum);
extern void mdtruncate(SMgrRelation reln, ForkNumber forknum,
					   BlockNumber nblocks);
extern void mdimmedsync(SMgrRelation reln, ForkNumber forknum);

extern void ForgetDatabaseSyncRequests(Oid dbid);
extern void DropRelationFiles(RelFileNode *delrels, int ndelrels, bool isRedo);

/* md sync callbacks */
extern int	mdsyncfiletag(const FileTag *ftag, char *path);
extern int	mdunlinkfiletag(const FileTag *ftag, char *path);
extern bool mdfiletagmatches(const FileTag *ftag, const FileTag *candidate);
```