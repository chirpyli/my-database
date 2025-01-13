## PostgreSQL源码分析 —— pg_control

### pg_control为什么会存在？
为啥会有pg_control这么个文件呢？ pg_control是PostgreSQL中一个很重要的文件，我们之前讲到过PostgreSQL的启动过程，启动过程中很重要的一项工作就是故障恢复，启动startup进程，回放WAL日志进行故障恢复，而从哪里开始进行回放呢？我怎么知道起点在哪里呢？这个位置的保存一定是在磁盘中，而不是在内存中，假设数据库因故障崩溃，内存中的数据会丢失，所以，只有checkpointer进程在做checkpoint操作时不断的更新pg_control文件，使之持久化保存，数据库启动进行故障恢复时，读取该文件，获得故障恢复的起始位置。  

除了保存检查点信息，还保存一些其他的状态等信息，用于数据库启动等。比如数据库状态信息，系统表版本号等。

看下面的代码，数据库启动时会检查pg_control文件，如果文件被损坏，数据库就会启动失败。

```c++
main()
--> MemoryContextInit()          // 初始化内存上下文： TopMemoryContext、ErrorContext
--> PostmasterMain(argc, argv);  // Postmaster main entry point
    --> pqsignal_pm(SIGCHLD, reaper);	/* handle child termination */ // 注册信号处理函数
    --> checkDataDir();          // 检查数据目录
        --> ValidatePgVersion(DataDir); // 检查PG_VERSION文件，PG实例版本是否与程序兼容
    --> checkControlFile();             // 检查pg_control文件
    --> CreateDataDirLockFile(true);    // 创建postmaster.pid文件
    --> LocalProcessControlFile(false); // 读pg_control，到ControlFileData中
        --> ReadControlFile();

```
startup进程从pg_control中获取故障恢复起点：
```c++
StartupProcessMain(void)
--> StartupXLOG();
    --> ValidateXLOGDirectoryStructure();   // 检查pg_wal是否存在
    -->	readRecoverySignalFile();           // 依据standby.signal和recovery.signal是否存在，判断进入何种状态
	  --> validateRecoveryParameters();
    if (read_backup_label(&checkPointLoc, &backupEndRequired, &backupFromStandby))
    {
        // 如果backup_label文件存在，则表示从备份文件中进行恢复（例如使用pg_basebackup进行备份）
        // 此种情况，设置backup_label，而不是用pg_control，为啥呢？下面就是解释
         /*
          * If we see a backup_label during recovery, we assume that we are recovering
          * from a backup dump file, and we therefore roll forward from the checkpoint
          * identified by the label file, NOT what pg_control says.  This avoids the
          * problem that pg_control might have been archived one or more checkpoints
          * later than the start of the dump, and so if we rely on it as the start
          * point, we will fail to restore a consistent database state.
          */
    }
    else
    {
        /* Get the last valid checkpoint record. */
        checkPointLoc = ControlFile->checkPoint;            // 从pg_control中获取检查点信息
        RedoStartLSN = ControlFile->checkPointCopy.redo;
        record = ReadCheckpointRecord(xlogreader, checkPointLoc, 1, true);
                 --> XLogBeginRead(xlogreader, RecPtr);     // Begin reading WAL at 'RecPtr'.
                 --> ReadRecord(xlogreader, LOG, true);     // Attempt to read the next XLOG record.
                     for (;;)
                     {
                          XLogReadRecord(xlogreader, &errormsg);    // Attempt to read an XLOG record.
                     }
    }
```

### pg_control文件的内容
pg_control保存在PGDATA/global/pg_control中，我们通过pg_controldata查看其具体内容：
其中有2个是非常重要的，
```shell
Latest checkpoint location:           0/D04FD00       -- 最后一次的checkpoint位置
Latest checkpoint's REDO location:    0/D04FCC8
```
怎么解释呢？可以参考一张图片!image[](https://img2020.cnblogs.com/blog/1660349/202105/1660349-20210531223332830-1896329496.png)
就是说checkpoint操作是需要一定时间的，在开始进行checkpoint时，先记录当前点为Latest checkpoint REDO location，当完成刷盘操作后，把checkpoint相关信息也生成一条WAL记录，再把这个WAL记录也写入WAL日志文件中，位置就是Latest checkpoint location。


```shell
postgres@slpc:~/pgsql$ ./bin/pg_controldata -D masternode/
pg_control version number:            1300
Catalog version number:               202107181                 -- 系统表版本号
Database system identifier:           7242131451622390647       -- 数据库系统标识符，用于标识一套数据库系统，物理复制的主备库拥有相同的数据库唯一标识串，initdb时生成
Database cluster state:               in production
pg_control last modified:             2023年08月03日 星期四 18时17分24秒
Latest checkpoint location:           0/D04FD00       -- 最后一次的checkpoint位置
Latest checkpoint's REDO location:    0/D04FCC8
Latest checkpoint's REDO WAL file:    00000001000000000000000D
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
Latest checkpoint's NextXID:          0:25998
Latest checkpoint's NextOID:          24641
Latest checkpoint's NextMultiXactId:  1
Latest checkpoint's NextMultiOffset:  0
Latest checkpoint's oldestXID:        726
Latest checkpoint's oldestXID's DB:   1
Latest checkpoint's oldestActiveXID:  25998
Latest checkpoint's oldestMultiXid:   1
Latest checkpoint's oldestMulti's DB: 1
Latest checkpoint's oldestCommitTsXid:0
Latest checkpoint's newestCommitTsXid:0
Time of latest checkpoint:            2023年08月03日 星期四 18时17分18秒
Fake LSN counter for unlogged rels:   0/3E8
Minimum recovery ending location:     0/0         -- 备机用，用于指定当备考异常终止再启动时，只有应用WAL日志过指定点后才能对外提供只读服务，否则，用户读到的数据可能会不一致。
Min recovery ending loc's timeline:   0
Backup start location:                0/0
Backup end location:                  0/0
End-of-backup record required:        no
wal_level setting:                    replica
wal_log_hints setting:                off
max_connections setting:              100
max_worker_processes setting:         8
max_wal_senders setting:              10
max_prepared_xacts setting:           0
max_locks_per_xact setting:           64
track_commit_timestamp setting:       off
Maximum data alignment:               8
Database block size:                  8192
Blocks per segment of large relation: 131072
WAL block size:                       8192
Bytes per WAL segment:                16777216
Maximum length of identifiers:        64
Maximum columns in an index:          32
Maximum size of a TOAST chunk:        1996
Size of a large-object chunk:         2048
Date/time type storage:               64-bit integers
Float8 argument passing:              by value
Data page checksum version:           0
Mock authentication nonce:            5faa9a77552e9f84e1dadd05fabcc97d121cc211905bd02386eb881dd347c198
```
关于其具体含义，可以看下面的定义，更多可参考`src/include/catalog/pg_control.h`的定义。
```c++
/* Contents of pg_control. */
typedef struct ControlFileData
{
	/* Unique system identifier --- to ensure we match up xlog files with the installation that produced them. */
	uint64		system_identifier;
	uint32		pg_control_version; /* PG_CONTROL_VERSION */
	uint32		catalog_version_no; /* see catversion.h */

	/* System status data */
	DBState		state;			/* see enum above */
	pg_time_t	time;			/* time stamp of last pg_control update */
	XLogRecPtr	checkPoint;		/* last check point record ptr */
	CheckPoint	checkPointCopy; /* copy of last check point record */
	XLogRecPtr	unloggedLSN;	/* current fake LSN value, for unlogged rels */

	/*
	 * These two values determine the minimum point we must recover up to
	 * before starting up:
	 *
	 * minRecoveryPoint is updated to the latest replayed LSN whenever we
	 * flush a data change during archive recovery. That guards against
	 * starting archive recovery, aborting it, and restarting with an earlier
	 * stop location. If we've already flushed data changes from WAL record X
	 * to disk, we mustn't start up until we reach X again. Zero when not
	 * doing archive recovery.
	 *
	 * backupStartPoint is the redo pointer of the backup start checkpoint, if
	 * we are recovering from an online backup and haven't reached the end of
	 * backup yet. It is reset to zero when the end of backup is reached, and
	 * we mustn't start up before that. A boolean would suffice otherwise, but
	 * we use the redo pointer as a cross-check when we see an end-of-backup
	 * record, to make sure the end-of-backup record corresponds the base
	 * backup we're recovering from.
	 *
	 * backupEndPoint is the backup end location, if we are recovering from an
	 * online backup which was taken from the standby and haven't reached the
	 * end of backup yet. It is initialized to the minimum recovery point in
	 * pg_control which was backed up last. It is reset to zero when the end
	 * of backup is reached, and we mustn't start up before that.
	 *
	 * If backupEndRequired is true, we know for sure that we're restoring
	 * from a backup, and must see a backup-end record before we can safely
	 * start up. If it's false, but backupStartPoint is set, a backup_label
	 * file was found at startup but it may have been a leftover from a stray
	 * pg_start_backup() call, not accompanied by pg_stop_backup().
	 */
	XLogRecPtr	minRecoveryPoint;
	TimeLineID	minRecoveryPointTLI;
	XLogRecPtr	backupStartPoint;
	XLogRecPtr	backupEndPoint;
	bool		backupEndRequired;

	/* Parameter settings that determine if the WAL can be used for archival or hot standby. */
	int			wal_level;
	bool		wal_log_hints;
	int			MaxConnections;
	int			max_worker_processes;
	int			max_wal_senders;
	int			max_prepared_xacts;
	int			max_locks_per_xact;
	bool		track_commit_timestamp;

	/*
	 * This data is used to check for hardware-architecture compatibility of
	 * the database and the backend executable.  We need not check endianness
	 * explicitly, since the pg_control version will surely look wrong to a
	 * machine of different endianness, but we do need to worry about MAXALIGN
	 * and floating-point format.  (Note: storage layout nominally also
	 * depends on SHORTALIGN and INTALIGN, but in practice these are the same
	 * on all architectures of interest.)
	 *
	 * Testing just one double value is not a very bulletproof test for
	 * floating-point compatibility, but it will catch most cases.
	 */
	uint32		maxAlign;		/* alignment requirement for tuples */
	double		floatFormat;	/* constant 1234567.0 */
#define FLOATFORMAT_VALUE	1234567.0

	/* This data is used to make sure that configuration of this database is compatible with the backend executable.*/
	uint32		blcksz;			/* data block size for this DB */
	uint32		relseg_size;	/* blocks per segment of large relation */
	uint32		xlog_blcksz;	/* block size within WAL files */
	uint32		xlog_seg_size;	/* size of each WAL segment */
	uint32		nameDataLen;	/* catalog name field width */
	uint32		indexMaxKeys;	/* max number of columns in an index */
	uint32		toast_max_chunk_size;	/* chunk size in TOAST tables */
	uint32		loblksize;		/* chunk size in pg_largeobject */

	bool		float8ByVal;	/* float8, int8, etc pass-by-value? */

	/* Are data pages protected by checksums? Zero if no checksum version */
	uint32		data_checksum_version;

	/* Random nonce, used in authentication requests that need to proceed
	 * based on values that are cluster-unique, like a SASL exchange that
	 * failed at an early stage. */
	char		mock_authentication_nonce[MOCK_AUTH_NONCE_LEN];
	pg_crc32c	crc;   	/* CRC of all above ... MUST BE LAST! */
} ControlFileData;
```


### pg_controldata 
可通过`pg_controldata -D masternode/`这种形式查看pg_control文件的内容，我们看一下其主流程，就是读pg_control文件，然后将内容进行解析。
```c++
main(int argc, char *argv[])
{
  	ControlFileData *ControlFile;
  	/* get a copy of the control file */
    ControlFile = get_controlfile(DataDir, &crc_ok);
    printf(_("pg_control version number:            %u\n"), ControlFile->pg_control_version);
    printf(_("Catalog version number:               %u\n"), ControlFile->catalog_version_no);
    printf(_("Database system identifier:           %llu\n"), (unsigned long long) ControlFile->system_identifier);
    printf(_("Database cluster state:               %s\n"), dbState(ControlFile->state));
    printf(_("pg_control last modified:             %s\n"), pgctime_str);
    printf(_("Latest checkpoint location:           %X/%X\n"), LSN_FORMAT_ARGS(ControlFile->checkPoint));
    printf(_("Latest checkpoint's REDO location:    %X/%X\n"), LSN_FORMAT_ARGS(ControlFile->checkPointCopy.redo));
    printf(_("Latest checkpoint's REDO WAL file:    %s\n"), xlogfilename);
    // 其他信息...
}
// 读pg_control文件到ControlFileData中
ControlFileData *get_controlfile(const char *DataDir, bool *crc_ok_p)
{
    ControlFileData *ControlFile;
    ControlFile = palloc(sizeof(ControlFileData));
    fd = open(ControlFilePath, O_RDONLY | PG_BINARY, 0);
    r = read(fd, ControlFile, sizeof(ControlFileData));
    close(fd);

    return ControlFile;
}
```


### 现象观测

初始化数据库时，initdb后，我们看一下其pg_control的状态
```shell
postgres@slpc:~/pgsql$ ./bin/pg_controldata -D pgdata/
pg_control version number:            1300
Catalog version number:               202107181
Database system identifier:           7264750828496047334
Database cluster state:               shut down                                 -- shut down状态
pg_control last modified:             2023年08月08日 星期二 09时03分33秒
Latest checkpoint location:           0/167E598
Latest checkpoint's REDO location:    0/167E598
Latest checkpoint's REDO WAL file:    000000010000000000000001
```

我们通过一些命令来加深对此的理解。
```sql
-- 创建插件，用于观测buffer
postgres=# create extension pg_buffercache ;
CREATE EXTENSION
-- 手动执行一次checkpoint。
postgres=# checkpoint;
CHECKPOINT
-- 查看当前WAL位置
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/D082C30
(1 row)
-- 建表
postgres=# create table t1 as select * from t2;
SELECT 10000
-- 查看buffer，可以看到blocknumber从0到54，共计55个脏页
postgres=# select * from pg_buffercache where isdirty and relfilenode = 't1'::regclass::oid;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
      405 |       16500 |          1663 |       13010 |             0 |              0 | t       |          1 |                0
      406 |       16500 |          1663 |       13010 |             0 |              1 | t       |          1 |                0
      407 |       16500 |          1663 |       13010 |             0 |              2 | t       |          1 |                0
      408 |       16500 |          1663 |       13010 |             0 |              3 | t       |          1 |                0
      409 |       16500 |          1663 |       13010 |             0 |              4 | t       |          1 |                0
      410 |       16500 |          1663 |       13010 |             0 |              5 | t       |          1 |                0
      411 |       16500 |          1663 |       13010 |             0 |              6 | t       |          1 |                0
      412 |       16500 |          1663 |       13010 |             0 |              7 | t       |          1 |                0
      413 |       16500 |          1663 |       13010 |             0 |              8 | t       |          1 |                0
      414 |       16500 |          1663 |       13010 |             0 |              9 | t       |          1 |                0
      415 |       16500 |          1663 |       13010 |             0 |             10 | t       |          1 |                0
      416 |       16500 |          1663 |       13010 |             0 |             11 | t       |          1 |                0
      417 |       16500 |          1663 |       13010 |             0 |             12 | t       |          1 |                0
      418 |       16500 |          1663 |       13010 |             0 |             13 | t       |          1 |                0
      419 |       16500 |          1663 |       13010 |             0 |             14 | t       |          1 |                0
      420 |       16500 |          1663 |       13010 |             0 |             15 | t       |          1 |                0
      421 |       16500 |          1663 |       13010 |             0 |             16 | t       |          1 |                0
      422 |       16500 |          1663 |       13010 |             0 |             17 | t       |          1 |                0
      423 |       16500 |          1663 |       13010 |             0 |             18 | t       |          1 |                0
      424 |       16500 |          1663 |       13010 |             0 |             19 | t       |          1 |                0
      425 |       16500 |          1663 |       13010 |             0 |             20 | t       |          1 |                0
      426 |       16500 |          1663 |       13010 |             0 |             21 | t       |          1 |                0
      427 |       16500 |          1663 |       13010 |             0 |             22 | t       |          1 |                0
      428 |       16500 |          1663 |       13010 |             0 |             23 | t       |          1 |                0
      429 |       16500 |          1663 |       13010 |             0 |             24 | t       |          1 |                0
      430 |       16500 |          1663 |       13010 |             0 |             25 | t       |          1 |                0
      431 |       16500 |          1663 |       13010 |             0 |             26 | t       |          1 |                0
      432 |       16500 |          1663 |       13010 |             0 |             27 | t       |          1 |                0
      433 |       16500 |          1663 |       13010 |             0 |             28 | t       |          1 |                0
      434 |       16500 |          1663 |       13010 |             0 |             29 | t       |          1 |                0
      435 |       16500 |          1663 |       13010 |             0 |             30 | t       |          1 |                0
      436 |       16500 |          1663 |       13010 |             0 |             31 | t       |          1 |                0
      437 |       16500 |          1663 |       13010 |             0 |             32 | t       |          1 |                0
      438 |       16500 |          1663 |       13010 |             0 |             33 | t       |          1 |                0
      439 |       16500 |          1663 |       13010 |             0 |             34 | t       |          1 |                0
      440 |       16500 |          1663 |       13010 |             0 |             35 | t       |          1 |                0
      441 |       16500 |          1663 |       13010 |             0 |             36 | t       |          1 |                0
      442 |       16500 |          1663 |       13010 |             0 |             37 | t       |          1 |                0
      443 |       16500 |          1663 |       13010 |             0 |             38 | t       |          1 |                0
      444 |       16500 |          1663 |       13010 |             0 |             39 | t       |          1 |                0
      445 |       16500 |          1663 |       13010 |             0 |             40 | t       |          1 |                0
      446 |       16500 |          1663 |       13010 |             0 |             41 | t       |          1 |                0
      447 |       16500 |          1663 |       13010 |             0 |             42 | t       |          1 |                0
      448 |       16500 |          1663 |       13010 |             0 |             43 | t       |          1 |                0
      449 |       16500 |          1663 |       13010 |             0 |             44 | t       |          1 |                0
      450 |       16500 |          1663 |       13010 |             0 |             45 | t       |          1 |                0
      451 |       16500 |          1663 |       13010 |             0 |             46 | t       |          1 |                0
      452 |       16500 |          1663 |       13010 |             0 |             47 | t       |          1 |                0
      453 |       16500 |          1663 |       13010 |             0 |             48 | t       |          1 |                0
      454 |       16500 |          1663 |       13010 |             0 |             49 | t       |          1 |                0
      455 |       16500 |          1663 |       13010 |             0 |             50 | t       |          1 |                0
      456 |       16500 |          1663 |       13010 |             0 |             51 | t       |          1 |                0
      457 |       16500 |          1663 |       13010 |             0 |             52 | t       |          1 |                0
      458 |       16500 |          1663 |       13010 |             0 |             53 | t       |          1 |                0
      459 |       16500 |          1663 |       13010 |             0 |             54 | t       |          1 |                0
(55 rows)
-- 执行一次checkpoint
postgres=# checkpoint;
CHECKPOINT
-- 查看脏页，脏页都被刷盘
postgres=# select * from pg_buffercache where relfilenode = 't1'::regclass::oid and isdirty;
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends 
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+------------------
(0 rows)
-- 查看检查点
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/D14D578
(1 row)
-- 查看当前lsn对应的wal日志文件
postgres=# SELECT file_name, upper(to_hex(file_offset)) file_offset FROM pg_walfile_name_offset('0/D14D578');
        file_name         | file_offset 
--------------------------+-------------
 00000001000000000000000D | 14D578
(1 row)

```
我们通过pg_waldump查看一下WAL日志记录，比较长，截取了其中一段：
```sql
postgres@slpc:~/pgsql$ ./bin/pg_waldump -p masternode/pg_wal/ -s 0/D082C30 -e 0/D14D578 00000001000000000000000D

-- ...
rmgr: Heap        len (rec/tot):     71/    71, tx:      26000, lsn: 0/0D09AF20, prev 0/0D09AEF0, desc: INSERT+INIT off 1 flags 0x00, blkref #0: rel 1663/13010/16500 blk 0
rmgr: Heap        len (rec/tot):     71/    71, tx:      26000, lsn: 0/0D09AF68, prev 0/0D09AF20, desc: INSERT off 2 flags 0x00, blkref #0: rel 1663/13010/16500 blk 0
rmgr: Heap        len (rec/tot):     71/    71, tx:      26000, lsn: 0/0D09AFB0, prev 0/0D09AF68, desc: INSERT off 3 flags 0x00, blkref #0: rel 1663/13010/16500 blk 0
 -- ...
rmgr: Heap        len (rec/tot):     71/    71, tx:      26000, lsn: 0/0D14B398, prev 0/0D14B350, desc: INSERT off 10 flags 0x00, blkref #0: rel 1663/13010/16500 blk 54
rmgr: Transaction len (rec/tot):    437/   437, tx:      26000, lsn: 0/0D14B3E0, prev 0/0D14B398, desc: COMMIT 2023-08-04 16:29:03.649432 CST; inval msgs: catcache 76 catcache 75 catcache 76 catcache 75 catcache 51 catcache 50 catcache 7 catcache 6 catcache 7 catcache 6 catcache 7 catcache 6 catcache 7 catcache 6 catcache 7 catcache 6 catcache 7 catcache 6 catcache 7 catcache 6 catcache 7 catcache 6 snapshot 2608 relcache 16500
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/0D14B598, prev 0/0D14B3E0, desc: RUNNING_XACTS nextXid 26001 latestCompletedXid 26000 oldestRunningXid 26001
rmgr: Heap2       len (rec/tot):     59/  7791, tx:          0, lsn: 0/0D14B5D0, prev 0/0D14B598, desc: PRUNE latestRemovedXid 0 nredirected 0 ndead 1, blkref #0: rel 1663/13010/1255 blk 25 FPW
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/0D14D458, prev 0/0D14B5D0, desc: RUNNING_XACTS nextXid 26001 latestCompletedXid 26000 oldestRunningXid 26001
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/0D14D490, prev 0/0D14D458, desc: RUNNING_XACTS nextXid 26001 latestCompletedXid 26000 oldestRunningXid 26001
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/0D14D4C8, prev 0/0D14D490, desc: CHECKPOINT_ONLINE redo 0/D14D490; tli 1; prev tli 1; fpw true; xid 0:26001; oid 24641; multi 1; offset 0; oldest xid 726 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 26001; online
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/0D14D540, prev 0/0D14D4C8, desc: RUNNING_XACTS nextXid 26001 latestCompletedXid 26000 oldestRunningXid 26001


```



参考文档：
[PostgreSQL故障恢复能力之检查点（Checkpoint）](https://www.cnblogs.com/mingfan/p/14829511.html)
[He3DB恢复过程源码分析系列](https://zhuanlan.zhihu.com/p/568727825)

