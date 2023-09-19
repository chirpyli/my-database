### PostgreSQL源码分析——基础备份

进行基础备份有2中方式，可使用pg_basebackup工具或其他备份工具进行备份，另一种是使用底层命令进行基础备份。pg_basebackup等工具其实是封装了底层命令，所以，为了更好的理解基础备份的过程，这里我们使用底层命令进行备份。并分析其中的源码实现。


#### 基础备份过程
备份的方式有多种，可以进行SQL Dump，也可以停止数据库实例，对实例物理文件进行复制拷贝，有其各自的优缺点与适用场景。这里的基础备份，其中一个最大的优势就是可以不停机，不停业务进行物理备份，在备份过程中，不需要获取表上的锁，正常业务受备份的影响较小。另外非常强大的一个功能就是PIRT，后面再去分析，这里我们分析一下基础备份的全过程。

基础备份的过程如下：
1. 连接到数据库
2. 执行`select pg_start_backup('lable')`命令。（会强制发生一次checkpoint，并将检查点记录到backup_label文件中）
3. 执行备份，把数据目录进行复制（包含backup_label）
4. 执行`select pg_stop_backup`命令，（删除backup_label文件，并在WAL日志中写入一条`XLOG_BACKUP_END`的记录，当备节点回放到该记录时，就知道备份结束了，数据达到了一致点，可以对外提供服务了）
5. 备份过程中产生的WAL日志进行复制


#### 操作执行过程分析
在分析源码之前，我们先执行基础备份操作过程，进行基础备份，帮助我们理解其中的备份过程。
1. initdb，创建数据库
查看pg_control
```sql
postgres@slpc:~/pgsql$ pg_controldata -D pgdata/
pg_control version number:            1300
Catalog version number:               202107181
Database system identifier:           7279971345653503170
Database cluster state:               shut down
pg_control last modified:             2023年09月18日 星期一 09时26分56秒
Latest checkpoint location:           0/167E598
Latest checkpoint's REDO location:    0/167E598
Latest checkpoint's REDO WAL file:    000000010000000000000001
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
```
查看WAL日志：
```
postgres@slpc:~/pgsql$ pg_waldump -p pgdata/pg_wal/ 000000010000000000000001
// 省略...
rmgr: Transaction len (rec/tot):     66/    66, tx:        732, lsn: 0/0167E550, prev 0/0167E4B0, desc: COMMIT 2023-09-18 09:26:56.640405 CST; inval msgs: snapshot 2396
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/0167E598, prev 0/0167E550, desc: CHECKPOINT_SHUTDOWN redo 0/167E598; tli 1; prev tli 1; fpw true; xid 0:733; oid 13011; multi 1; offset 0; oldest xid 726 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 0; shutdown
```
2. 启动数据库
3. 连接数据库，建表，插入数据
4. 执行`pg_start_backup('bak1')`函数
```sql
postgres@slpc:~/pgsql/pgdata/pg_wal$ psql -p 7432
psql (14.8)
Type "help" for help.

postgres=# create table t1(a int);
CREATE TABLE
postgres=# insert into t1 values(1);
INSERT 0 1
postgres=# select pg_start_backup('bak1');
 pg_start_backup 
-----------------
 0/2000028
(1 row)
```
首先是日志文件发生切换，切换后再执行checkpoint操作
```sql
postgres@slpc:~/pgsql/pgdata/pg_wal$ ls   
000000010000000000000001  archive_status
postgres@slpc:~/pgsql/pgdata/pg_wal$ ls   -- 强制切换WAL段，回收WAL文件, 从000000010000000000000002开始，后的WAL文件都要拷贝到备份文件中，回收的WAL文件则不需要
000000010000000000000002  000000010000000000000003  archive_status

查看日志，观察运行过程， 执行过程中，会进行checkpoint操作：
```sql
2023-09-18 10:12:21.139 CST [417435] DEBUG:  00000: attempting to remove WAL segments older than log file 000000000000000000000001
2023-09-18 10:12:21.139 CST [417435] LOCATION:  RemoveOldXlogFiles, xlog.c:4114
2023-09-18 10:12:21.141 CST [417435] DEBUG:  00000: recycled write-ahead log file "000000010000000000000001"
2023-09-18 10:12:21.141 CST [417435] LOCATION:  RemoveXlogFile, xlog.c:4256
2023-09-18 10:12:21.141 CST [417435] DEBUG:  00000: SlruScanDirectory invoking callback on pg_subtrans/0000
2023-09-18 10:12:21.141 CST [417435] LOCATION:  SlruScanDirectory, slru.c:1574
2023-09-18 10:12:21.141 CST [417435] LOG:  00000: checkpoint complete: wrote 31 buffers (0.2%); 0 WAL file(s) added, 0 removed, 1 recycled; write=2.846 s, sync=0.005 s, total=2.860 s; sync files=22, longest=0.004 s, average=0.001 s; distance=9734 kB, estimate=9734 kB
2023-09-18 10:12:21.141 CST [417435] LOCATION:  LogCheckpointEnd, xlog.c:8925
2023-09-18 10:12:39.283 CST [417436] DEBUG:  00000: snapshot of 0+0 running transaction ids (lsn 0/2000148 oldest xid 735 latest complete 734 next xid 735)
```
观察wal日志：
```sql
postgres@slpc:~/pgsql$ pg_waldump -p pgdata/pg_wal/ 000000010000000000000002
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/02000028, prev 0/01696D18, desc: RUNNING_XACTS nextXid 735 latestCompletedXid 734 oldestRunningXid 735
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/02000060, prev 0/02000028, desc: RUNNING_XACTS nextXid 735 latestCompletedXid 734 oldestRunningXid 735
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/02000098, prev 0/02000060, desc: CHECKPOINT_ONLINE redo 0/2000028; tli 1; prev tli 1; fpw true; xid 0:735; oid 24576; multi 1; offset 0; oldest xid 726 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 735; online
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/02000110, prev 0/02000098, desc: RUNNING_XACTS nextXid 735 latestCompletedXid 734 oldestRunningXid 735
rmgr: Heap        len (rec/tot):     54/   150, tx:        735, lsn: 0/02000148, prev 0/02000110, desc: INSERT off 2 flags 0x00, blkref #0: rel 1663/13010/16384 blk 0 FPW
rmgr: Transaction len (rec/tot):     34/    34, tx:        735, lsn: 0/020001E0, prev 0/02000148, desc: COMMIT 2023-09-18 10:23:57.688476 CST
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/02000208, prev 0/020001E0, desc: RUNNING_XACTS nextXid 736 latestCompletedXid 735 oldestRunningXid 736
```
观察pg_control
```sql
postgres@slpc:~/pgsql$ pg_controldata -D pgdata/
pg_control version number:            1300
Catalog version number:               202107181
Database system identifier:           7279971345653503170
Database cluster state:               in production
pg_control last modified:             2023年09月18日 星期一 10时12分21秒
Latest checkpoint location:           0/2000098        -- 最新检测点位置 
Latest checkpoint's REDO location:    0/2000028
Latest checkpoint's REDO WAL file:    000000010000000000000002   -- REDO WAL文件，即checkpoint REDO location开始的文件
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
Latest checkpoint's NextXID:          0:735
Latest checkpoint's NextOID:          24576
```
生成backup_label文件(非常重要，后续从备份文件中进行恢复时，从这里记录的位置开始，而不是读取pg_control文件中的位置)：
```sql
postgres@slpc:~/pgsql/pgdata$ cat backup_label 
START WAL LOCATION: 0/2000028 (file 000000010000000000000002)
CHECKPOINT LOCATION: 0/2000098
BACKUP METHOD: pg_start_backup
BACKUP FROM: primary
START TIME: 2023-09-18 10:12:21 CST
LABEL: bak1
START TIMELINE: 1
```

5. 拷贝数据库实例到备份文件

6. 执行`pg_stop_backup()`，结束基础备份
```sql
postgres=# select pg_stop_backup();
NOTICE:  WAL archiving is not enabled; you must ensure that all required WAL segments are copied through other means to complete the backup
 pg_stop_backup 
----------------
 0/2000268
(1 row)
```
观察日志
```
2023-09-18 10:47:41.095 CST [447083] DEBUG:  00000: removing WAL backup history file "000000010000000000000002.00000028.backup"
2023-09-18 10:47:41.095 CST [447083] LOCATION:  CleanupBackupHistory, xlog.c:4375
2023-09-18 10:47:41.095 CST [447083] NOTICE:  00000: WAL archiving is not enabled; you must ensure that all required WAL segments are copied through other means to complete the backup
2023-09-18 10:47:41.095 CST [447083] LOCATION:  do_pg_stop_backup, xlog.c:11912
2023-09-18 10:47:41.263 CST [417436] DEBUG:  00000: snapshot of 0+0 running transaction ids (lsn 0/3000060 oldest xid 736 latest complete 735 next xid 736)
```
查看wal日志：
```sql
postgres@slpc:~/pgsql$ pg_waldump -p pgdata/pg_wal/ 000000010000000000000002
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/02000028, prev 0/01696D18, desc: RUNNING_XACTS nextXid 735 latestCompletedXid 734 oldestRunningXid 735
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/02000060, prev 0/02000028, desc: RUNNING_XACTS nextXid 735 latestCompletedXid 734 oldestRunningXid 735
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/02000098, prev 0/02000060, desc: CHECKPOINT_ONLINE redo 0/2000028; tli 1; prev tli 1; fpw true; xid 0:735; oid 24576; multi 1; offset 0; oldest xid 726 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 735; online
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/02000110, prev 0/02000098, desc: RUNNING_XACTS nextXid 735 latestCompletedXid 734 oldestRunningXid 735
rmgr: Heap        len (rec/tot):     54/   150, tx:        735, lsn: 0/02000148, prev 0/02000110, desc: INSERT off 2 flags 0x00, blkref #0: rel 1663/13010/16384 blk 0 FPW
rmgr: Transaction len (rec/tot):     34/    34, tx:        735, lsn: 0/020001E0, prev 0/02000148, desc: COMMIT 2023-09-18 10:23:57.688476 CST
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/02000208, prev 0/020001E0, desc: RUNNING_XACTS nextXid 736 latestCompletedXid 735 oldestRunningXid 736
rmgr: XLOG        len (rec/tot):     34/    34, tx:          0, lsn: 0/02000240, prev 0/02000208, desc: BACKUP_END 0/2000028
rmgr: XLOG        len (rec/tot):     24/    24, tx:          0, lsn: 0/02000268, prev 0/02000240, desc: SWITCH 
postgres@slpc:~/pgsql$ pg_waldump -p pgdata/pg_wal/ 000000010000000000000003
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/03000028, prev 0/02000268, desc: RUNNING_XACTS nextXid 736 latestCompletedXid 735 oldestRunningXid 736
```
删除了源数据库实例中的`backup_label`文件，因为这个是给备库用的，已经被拷贝到了备份文件中，等待恢复使用。

7. 备份文件进行恢复
启动备份的数据库实例，读backup_label文件，
观察日志：
```
2023-09-18 11:09:59.964 CST [1237713] LOG:  00000: starting PostgreSQL 14.8 on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0, 64-bit
2023-09-18 11:09:59.965 CST [1237713] LOG:  00000: listening on IPv4 address "0.0.0.0", port 7431
2023-09-18 11:09:59.965 CST [1237713] LOG:  00000: listening on IPv6 address "::", port 7431
2023-09-18 11:09:59.970 CST [1237713] LOG:  00000: listening on Unix socket "/tmp/.s.PGSQL.7431"
2023-09-18 11:09:59.976 CST [1237717] LOG:  00000: database system was interrupted; last known up at 2023-09-18 10:12:21 CST
2023-09-18 11:09:59.976 CST [1237717] LOCATION:  StartupXLOG, xlog.c:6585
2023-09-18 11:09:59.976 CST [1237717] DEBUG:  00000: removing all temporary WAL segments
2023-09-18 11:09:59.976 CST [1237717] LOCATION:  RemoveTempXlogFiles, xlog.c:4070
2023-09-18 11:09:59.993 CST [1237717] DEBUG:  00000: backup time 2023-09-18 10:12:21 CST in file "backup_label"
2023-09-18 11:09:59.993 CST [1237717] LOCATION:  read_backup_label, xlog.c:12143
2023-09-18 11:09:59.993 CST [1237717] DEBUG:  00000: backup label bak1 in file "backup_label"
2023-09-18 11:09:59.993 CST [1237717] LOCATION:  read_backup_label, xlog.c:12148
2023-09-18 11:09:59.993 CST [1237717] DEBUG:  00000: backup timeline 1 in file "backup_label"
2023-09-18 11:09:59.993 CST [1237717] LOCATION:  read_backup_label, xlog.c:12165
2023-09-18 11:09:59.993 CST [1237717] DEBUG:  00000: checkpoint record is at 0/2000098
2023-09-18 11:09:59.993 CST [1237717] LOCATION:  StartupXLOG, xlog.c:6729
2023-09-18 11:09:59.993 CST [1237717] DEBUG:  00000: redo record is at 0/2000028; shutdown false
2023-09-18 11:09:59.993 CST [1237717] LOCATION:  StartupXLOG, xlog.c:6936
2023-09-18 11:09:59.993 CST [1237717] DEBUG:  00000: next transaction ID: 735; next OID: 24576
2023-09-18 11:09:59.993 CST [1237717] LOCATION:  StartupXLOG, xlog.c:6940
2023-09-18 11:09:59.994 CST [1237717] DEBUG:  00000: next MultiXactId: 1; next MultiXactOffset: 0
2023-09-18 11:09:59.994 CST [1237717] LOCATION:  StartupXLOG, xlog.c:6944
2023-09-18 11:09:59.994 CST [1237717] DEBUG:  00000: oldest unfrozen transaction ID: 726, in database 1
2023-09-18 11:09:59.994 CST [1237717] LOCATION:  StartupXLOG, xlog.c:6947
2023-09-18 11:09:59.994 CST [1237717] DEBUG:  00000: oldest MultiXactId: 1, in database 1
2023-09-18 11:09:59.994 CST [1237717] LOCATION:  StartupXLOG, xlog.c:6950
2023-09-18 11:09:59.994 CST [1237717] DEBUG:  00000: commit timestamp Xid oldest/newest: 0/0
2023-09-18 11:09:59.994 CST [1237717] LOCATION:  StartupXLOG, xlog.c:6953
2023-09-18 11:09:59.994 CST [1237717] DEBUG:  00000: transaction ID wrap limit is 2147484373, limited by database with OID 1
2023-09-18 11:09:59.994 CST [1237717] LOCATION:  SetTransactionIdLimit, varsup.c:427
2023-09-18 11:09:59.994 CST [1237717] DEBUG:  00000: MultiXactId wrap limit is 2147483648, limited by database with OID 1
2023-09-18 11:09:59.994 CST [1237717] LOCATION:  SetMultiXactIdLimit, multixact.c:2283
2023-09-18 11:09:59.994 CST [1237717] DEBUG:  00000: starting up replication slots
2023-09-18 11:09:59.994 CST [1237717] LOCATION:  StartupReplicationSlots, slot.c:1394
2023-09-18 11:09:59.994 CST [1237717] DEBUG:  00000: xmin required by slots: data 0, catalog 0
2023-09-18 11:09:59.994 CST [1237717] LOCATION:  ProcArraySetReplicationSlotXmin, procarray.c:3984
2023-09-18 11:09:59.994 CST [1237717] DEBUG:  00000: starting up replication origin progress state
2023-09-18 11:09:59.994 CST [1237717] LOCATION:  StartupReplicationOrigin, origin.c:706
2023-09-18 11:09:59.996 CST [1237717] DEBUG:  00000: resetting unlogged relations: cleanup 1 init 0
2023-09-18 11:09:59.996 CST [1237717] LOCATION:  ResetUnloggedRelations, reinit.c:55
2023-09-18 11:10:00.008 CST [1237717] LOG:  00000: redo starts at 0/2000028
2023-09-18 11:10:00.008 CST [1237717] LOCATION:  StartupXLOG, xlog.c:7387
2023-09-18 11:10:00.008 CST [1237717] DEBUG:  00000: end of backup reached
2023-09-18 11:10:00.008 CST [1237717] CONTEXT:  WAL redo at 0/2000240 for XLOG/BACKUP_END: 0/2000028
2023-09-18 11:10:00.008 CST [1237717] LOCATION:  xlog_redo, xlog.c:10595
2023-09-18 11:10:00.010 CST [1237717] LOG:  00000: consistent recovery state reached at 0/2000268    到达一致性点，也就是pg_stop_backup的位置
2023-09-18 11:10:00.010 CST [1237717] LOCATION:  CheckRecoveryConsistency, xlog.c:8331      
```
观察WAL文件
```sql
postgres@slpc:~/pgsql$ pg_waldump -p pgbak/pg_wal/ 000000010000000000000003
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/03000028, prev 0/02000268, desc: RUNNING_XACTS nextXid 736 latestCompletedXid 735 oldestRunningXid 736
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/03000060, prev 0/03000028, desc: CHECKPOINT_SHUTDOWN redo 0/3000060; tli 1; prev tli 1; fpw true; xid 0:736; oid 24576; multi 1; offset 0; oldest xid 726 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 0; shutdown
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/030000D8, prev 0/03000060, desc: RUNNING_XACTS nextXid 736 latestCompletedXid 735 oldestRunningXid 736
```

8. 检测备份的数据库实例是否启动成功
```sql
postgres@slpc:~/pgsql/pgdata$ psql -p 7431
psql (14.8)
Type "help" for help.

postgres=# \d
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

postgres=# select * from t1;
 a 
---
 1
 2
(2 rows)
```

下面我们进行源码分析

#### pg_start_backup
`pg_start_backup`开始为制作基础备份进行准备工作，恢复过程从重做点开始，因此`pg_start_backup`必须执行检查点，以便在制作基础备份开始的时刻显式创建一个重做点。这个检查点的位置需要保存在非`pg_control`文件中，因为备份过程中，业务并没有停，期间可能会执行多次常规检查点。
```c++
pg_start_backup ( label text [, fast boolean [, exclusive boolean ]] ) → pg_lsn
```
准备开始在线备份。唯一需要的参数是用于备份的任意用户定义的标签。(通常，备份转储文件将存储在这个名称下。) 如果可选的第二个参数被指定为true，它将指定尽可能快地执行`pg_start_backup`。这将强制产生一个即时检查点，这将导致I/O操作突增，从而降低并发执行的查询的速度。第三个可选参数指定是执行排他或非排他备份(默认为排他备份)。在排他模式下使用时，该函数将写一个备份标签文件(backup_label)，如果pg_tblspc/目录中有任何链接， 则将一个表空间映射文件(tablespace_map)写入数据库集群的数据目录，然后执行检查点，然后返回备份的开始写-提前日志位置。 (用户可以忽略这个结果值，但在有用的情况下会提供它。) 在非排他模式下使用时，这些文件的内容将由`pg_stop_backup`函数返回，并且应该由用户复制到备份区域。



源码分析，调用`pg_start_backup`，调用的中间过程略，直接看函数实现。
```c++
pg_start_backup
--> do_pg_start_backup
```
`pg_start_backup`函数实现如下：

```c++
/*
 * pg_start_backup: set up for taking an on-line backup dump
 *
 * Essentially what this does is to create a backup label file in $PGDATA,
 * where it will be archived as part of the backup dump.  The label file
 * contains the user-supplied label string (typically this would be used
 * to tell where the backup dump will be stored) and the starting time and
 * starting WAL location for the dump.
 */
Datum pg_start_backup(PG_FUNCTION_ARGS)
{
	text	   *backupid = PG_GETARG_TEXT_PP(0);    // 参数1：用来唯一标识这次备份操作的任意字符串
   // 默认情况下，pg_start_backup可能需要较长的时间完成。 这是因为它会执行一个检查点，并且该检查点所需要的 I/O 将会分散到一段 显著的时间上，默认情况下是你的检查点间隔（见配置参数 checkpoint_completion_target）的一半。这通常 是你所想要的，因为它可以最小化对查询处理的影响。如果你想要尽可能快地 开始备份，请把第二个参数改成true，这将会发出一个立即的检查点并且使用尽可能多的I/O。
	bool		fast = PG_GETARG_BOOL(1);            
	bool		exclusive = PG_GETARG_BOOL(2);  // 开始一次非排他基础备份
	char	   *backupidstr;
	XLogRecPtr	startpoint;
	SessionBackupState status = get_backup_status();

	backupidstr = text_to_cstring(backupid);

	if (status == SESSION_BACKUP_NON_EXCLUSIVE)
		ereport(ERROR, (errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE), errmsg("a backup is already in progress in this session")));

	if (exclusive)  // 是否排他备份
	{
		startpoint = do_pg_start_backup(backupidstr, fast, NULL, NULL, NULL, NULL);
	}
	else
	{
		MemoryContext oldcontext;

		/* Label file and tablespace map file need to be long-lived, since they are read in pg_stop_backup. */
		oldcontext = MemoryContextSwitchTo(TopMemoryContext);
		label_file = makeStringInfo();
		tblspc_map_file = makeStringInfo();
		MemoryContextSwitchTo(oldcontext);

		register_persistent_abort_backup_handler();

		startpoint = do_pg_start_backup(backupidstr, fast, NULL, label_file,	NULL, tblspc_map_file);
	}

	PG_RETURN_LSN(startpoint);   // 返回LSN
}
```
实际实现在`do_pg_start_backup`中，主要工作：
- 强制开启full_page_writes = on， 备份结束再还原
- 切换到一个新的WAL日志文件，命名规则如下： （方便进行日志归档，拷贝等操作）
```c++
/* Generate a WAL segment file name.*/
#define XLogFileName(fname, tli, logSegNo, wal_segsz_bytes)	\
	snprintf(fname, MAXFNAMELEN, "%08X%08X%08X", tli,		\
			 (uint32) ((logSegNo) / XLogSegmentsPerXLogId(wal_segsz_bytes)), \
			 (uint32) ((logSegNo) % XLogSegmentsPerXLogId(wal_segsz_bytes)))
```
- 进行checkpoint
- 构造backup_lable文件，存储检查点位置等信息
返回最小的WAL LSN，以及timeline。这个LSN表示备份恢复需要的起始WAL日志的位置。
```c++
/*
 * do_pg_start_backup
 *
 * Utility function called at the start of an online backup. It creates the
 * necessary starting checkpoint and constructs the backup label file.

 * Returns the minimum WAL location that must be present to restore from this
 * backup, and the corresponding timeline ID in *starttli_p.
 */
XLogRecPtr
do_pg_start_backup(const char *backupidstr, bool fast, TimeLineID *starttli_p,
				   StringInfo labelfile, List **tablespaces, StringInfo tblspcmapfile)
{
	bool		exclusive = (labelfile == NULL);
	bool		backup_started_in_recovery = false;
	XLogRecPtr	checkpointloc;
	XLogRecPtr	startpoint;
	TimeLineID	starttli;
	pg_time_t	stamp_time;
	char		strfbuf[128];
	char		xlogfilename[MAXFNAMELEN];
	XLogSegNo	_logSegNo;
	struct stat stat_buf;
	FILE	   *fp;

	backup_started_in_recovery = RecoveryInProgress();

    // 在恢复阶段，不能进行排他备份
	/* Currently only non-exclusive backup can be taken during recovery.*/
	if (backup_started_in_recovery && exclusive)
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("recovery is in progress"),
				 errhint("WAL control functions cannot be executed during recovery.")));

	/* During recovery, we don't need to check WAL level. Because, if WAL
	 * level is not sufficient, it's impossible to get here during recovery. */
	if (!backup_started_in_recovery && !XLogIsNeeded())
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("WAL level not sufficient for making an online backup"),
				 errhint("wal_level must be set to \"replica\" or \"logical\" at server start.")));

    // ...

	/*
	 * Mark backup active in shared memory.  We must do full-page WAL writes
	 * during an on-line backup even if not doing so at other times, because
	 * it's quite possible for the backup dump to obtain a "torn" (partially
	 * written) copy of a database page if it reads the page concurrently with
	 * our write to the same page.  This can be fixed as long as the first
	 * write to the page in the WAL sequence is a full-page write. Hence, we
	 * turn on forcePageWrites and then force a CHECKPOINT, to ensure there
	 * are no dirty pages in shared memory that might get dumped while the
	 * backup is in progress without having a corresponding WAL record.  (Once
	 * the backup is complete, we need not force full-page writes anymore,
	 * since we expect that any pages not modified during the backup interval
	 * must have been correctly captured by the backup.)
	 *
	 * Note that forcePageWrites has no effect during an online backup from
	 * the standby.
	 *
	 * We must hold all the insertion locks to change the value of
	 * forcePageWrites, to ensure adequate interlocking against
	 * XLogInsertRecord().
	 */
	WALInsertLockAcquireExclusive();
	if (exclusive)
	{
		/* At first, mark that we're now starting an exclusive backup, to
		 * ensure that there are no other sessions currently running pg_start_backup() or pg_stop_backup(). */
		if (XLogCtl->Insert.exclusiveBackupState != EXCLUSIVE_BACKUP_NONE)
		{
			WALInsertLockRelease();
			ereport(ERROR,(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE), errmsg("a backup is already in progress"), errhint("Run pg_stop_backup() and try again.")));
		}
		XLogCtl->Insert.exclusiveBackupState = EXCLUSIVE_BACKUP_STARTING;
	}
	else
		XLogCtl->Insert.nonExclusiveBackups++;
	XLogCtl->Insert.forcePageWrites = true;   /* 强制开启full_page_writes */
	WALInsertLockRelease();

	/* Ensure we release forcePageWrites if fail below */
	PG_ENSURE_ERROR_CLEANUP(pg_start_backup_callback, (Datum) BoolGetDatum(exclusive));
	{
		bool		gotUniqueStartpoint = false;
		DIR		   *tblspcdir;
		struct dirent *de;
		tablespaceinfo *ti;
		int			datadirpathlen;

		/*
		 * Force an XLOG file switch before the checkpoint, to ensure that the
		 * WAL segment the checkpoint is written to doesn't contain pages with
		 * old timeline IDs.  That would otherwise happen if you called
		 * pg_start_backup() right after restoring from a PITR archive: the
		 * first WAL segment containing the startup checkpoint has pages in
		 * the beginning with the old timeline ID.  That can cause trouble at
		 * recovery: we won't have a history file covering the old timeline if
		 * pg_wal directory was not included in the base backup and the WAL
		 * archive was cleared too before starting the backup.
		 *
		 * This also ensures that we have emitted a WAL page header that has
		 * XLP_BKP_REMOVABLE off before we emit the checkpoint record.
		 * Therefore, if a WAL archiver (such as pglesslog) is trying to
		 * compress out removable backup blocks, it won't remove any that
		 * occur after this point.
		 *
		 * During recovery, we skip forcing XLOG file switch, which means that
		 * the backup taken during recovery is not available for the special
		 * recovery case described above.
		 */
		if (!backup_started_in_recovery)
			RequestXLogSwitch(false);     // 切换到一个新的WAL日志文件，默认是16MB后才切换

		do
		{
			bool		checkpointfpw;
      		// 进行强制checkpoint
			/*
			 * Force a CHECKPOINT.  Aside from being necessary to prevent torn
			 * page problems, this guarantees that two successive backup runs
			 * will have different checkpoint positions and hence different
			 * history file names, even if nothing happened in between.
			 *
			 * During recovery, establish a restartpoint if possible. We use
			 * the last restartpoint as the backup starting checkpoint. This
			 * means that two successive backup runs can have same checkpoint
			 * positions.
			 *
			 * Since the fact that we are executing do_pg_start_backup()
			 * during recovery means that checkpointer is running, we can use
			 * RequestCheckpoint() to establish a restartpoint.
			 *
			 * We use CHECKPOINT_IMMEDIATE only if requested by user (via
			 * passing fast = true).  Otherwise this can take awhile.
			 */
			RequestCheckpoint(CHECKPOINT_FORCE | CHECKPOINT_WAIT | (fast ? CHECKPOINT_IMMEDIATE : 0));

			/*
			 * Now we need to fetch the checkpoint record location, and also
			 * its REDO pointer.  The oldest point in WAL that would be needed
			 * to restore starting from the checkpoint is precisely the REDO pointer. */
			LWLockAcquire(ControlFileLock, LW_SHARED);
			checkpointloc = ControlFile->checkPoint;            // 获取最新的检查点信息
			startpoint = ControlFile->checkPointCopy.redo;  
			starttli = ControlFile->checkPointCopy.ThisTimeLineID;
			checkpointfpw = ControlFile->checkPointCopy.fullPageWrites;
			LWLockRelease(ControlFileLock);

      // ...

			/*
			 * If two base backups are started at the same time (in WAL sender
			 * processes), we need to make sure that they use different
			 * checkpoints as starting locations, because we use the starting
			 * WAL location as a unique identifier for the base backup in the
			 * end-of-backup WAL record and when we write the backup history
			 * file. Perhaps it would be better generate a separate unique ID
			 * for each backup instead of forcing another checkpoint, but
			 * taking a checkpoint right after another is not that expensive
			 * either because only few buffers have been dirtied yet.
			 */
			WALInsertLockAcquireExclusive();
			if (XLogCtl->Insert.lastBackupStart < startpoint)
			{
				XLogCtl->Insert.lastBackupStart = startpoint;
				gotUniqueStartpoint = true;
			}
			WALInsertLockRelease();
		} while (!gotUniqueStartpoint);

		XLByteToSeg(startpoint, _logSegNo, wal_segment_size);   //Compute a segment number from an XLogRecPtr.
		XLogFileName(xlogfilename, starttli, _logSegNo, wal_segment_size);  // 生成WAL日志文件名

		/* Construct tablespace_map file.   */
		if (tblspcmapfile == NULL)
			tblspcmapfile = makeStringInfo();

		datadirpathlen = strlen(DataDir);

		/* Collect information about all tablespaces */
		tblspcdir = AllocateDir("pg_tblspc");
		while ((de = ReadDir(tblspcdir, "pg_tblspc")) != NULL)
		{
            // ...
		}
		FreeDir(tblspcdir);

        //创建backup_label文件，构造信息
		/* Construct backup label file. */
		if (labelfile == NULL)
			labelfile = makeStringInfo();

		/* Use the log timezone here, not the session timezone */
		stamp_time = (pg_time_t) time(NULL);
		pg_strftime(strfbuf, sizeof(strfbuf),
					"%Y-%m-%d %H:%M:%S %Z",
					pg_localtime(&stamp_time, log_timezone));
		appendStringInfo(labelfile, "START WAL LOCATION: %X/%X (file %s)\n",
						 LSN_FORMAT_ARGS(startpoint), xlogfilename);
		appendStringInfo(labelfile, "CHECKPOINT LOCATION: %X/%X\n",
						 LSN_FORMAT_ARGS(checkpointloc));
		appendStringInfo(labelfile, "BACKUP METHOD: %s\n",
						 exclusive ? "pg_start_backup" : "streamed");
		appendStringInfo(labelfile, "BACKUP FROM: %s\n",
						 backup_started_in_recovery ? "standby" : "primary");
		appendStringInfo(labelfile, "START TIME: %s\n", strfbuf);
		appendStringInfo(labelfile, "LABEL: %s\n", backupidstr);
		appendStringInfo(labelfile, "START TIMELINE: %u\n", starttli);

    	// 写backup_lable文件到磁盘
		/* Okay, write the file, or return its contents to caller. */
		if (exclusive)
		{
			/* Check for existing backup label --- implies a backup is already
			 * running.  (XXX given that we checked exclusiveBackupState
			 * above, maybe it would be OK to just unlink any such label file?) */
			if (stat(BACKUP_LABEL_FILE, &stat_buf) != 0)
			{
				if (errno != ENOENT)
					ereport(ERROR, (errcode_for_file_access(), errmsg("could not stat file \"%s\": %m", BACKUP_LABEL_FILE)));
			}
			else
				ereport(ERROR,
						(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
						 errmsg("a backup is already in progress"),
						 errhint("If you're sure there is no backup in progress, remove file \"%s\" and try again.",
								 BACKUP_LABEL_FILE)));

			fp = AllocateFile(BACKUP_LABEL_FILE, "w");

			if (!fp)
				ereport(ERROR,(errcode_for_file_access(), errmsg("could not create file \"%s\": %m",BACKUP_LABEL_FILE)));
			if (fwrite(labelfile->data, labelfile->len, 1, fp) != 1 ||fflush(fp) != 0 ||pg_fsync(fileno(fp)) != 0 ||ferror(fp) ||FreeFile(fp))
				ereport(ERROR,(errcode_for_file_access(), errmsg("could not write file \"%s\": %m",BACKUP_LABEL_FILE)));
			/* Allocated locally for exclusive backups, so free separately */
			pfree(labelfile->data);
			pfree(labelfile);

			/* Write backup tablespace_map file. */
			if (tblspcmapfile->len > 0)
			{
				if (stat(TABLESPACE_MAP, &stat_buf) != 0)
				{
					if (errno != ENOENT)
						ereport(ERROR,(errcode_for_file_access(), errmsg("could not stat file \"%s\": %m",TABLESPACE_MAP)));
				}
				else
					ereport(ERROR,
							(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
							 errmsg("a backup is already in progress"),
							 errhint("If you're sure there is no backup in progress, remove file \"%s\" and try again.",
									 TABLESPACE_MAP)));

				fp = AllocateFile(TABLESPACE_MAP, "w");

				if (!fp)
					ereport(ERROR,(errcode_for_file_access(), errmsg("could not create file \"%s\": %m",TABLESPACE_MAP)));
				if (fwrite(tblspcmapfile->data, tblspcmapfile->len, 1, fp) != 1 ||
					fflush(fp) != 0 ||pg_fsync(fileno(fp)) != 0 ||ferror(fp) ||FreeFile(fp))
					ereport(ERROR,
							(errcode_for_file_access(),
							 errmsg("could not write file \"%s\": %m",
									TABLESPACE_MAP)));
			}

			/* Allocated locally for exclusive backups, so free separately */
			pfree(tblspcmapfile->data);
			pfree(tblspcmapfile);
		}
	}
	PG_END_ENSURE_ERROR_CLEANUP(pg_start_backup_callback, (Datum) BoolGetDatum(exclusive));

	/*
	 * Mark that start phase has correctly finished for an exclusive backup.
	 * Session-level locks are updated as well to reflect that state.
	 *
	 * Note that CHECK_FOR_INTERRUPTS() must not occur while updating backup
	 * counters and session-level lock. Otherwise they can be updated
	 * inconsistently, and which might cause do_pg_abort_backup() to fail.
	 */
	if (exclusive)
	{
		WALInsertLockAcquireExclusive();
		XLogCtl->Insert.exclusiveBackupState = EXCLUSIVE_BACKUP_IN_PROGRESS;

		/* Set session-level lock */
		sessionBackupState = SESSION_BACKUP_EXCLUSIVE;
		WALInsertLockRelease();
	}
	else
		sessionBackupState = SESSION_BACKUP_NON_EXCLUSIVE;

	/* We're done.  As a convenience, return the starting WAL location.*/
	if (starttli_p)
		*starttli_p = starttli;
	return startpoint;
}
```

执行如下命令：
```sql
postgres=# select pg_start_backup('bak1');
 pg_start_backup 
-----------------
 7/F7000148
(1 row)

-- 生成的backup_label文件内容
postgres@slpc:~/pgsql/pgdata$ cat backup_label 
START WAL LOCATION: 7/F7000148 (file 0000000100000007000000F7)
CHECKPOINT LOCATION: 7/F7000180
BACKUP METHOD: pg_start_backup
BACKUP FROM: primary
START TIME: 2023-09-15 15:05:13 CST
LABEL: bak1
START TIMELINE: 1
```

#### pg_stop_backup
结束备份操作，主要内容如下：
- 如果强制开启了full_page_writes，则关闭
- 写入一条备份结束的XLOG记录
- 切换WAL段文件
- 创建一个备份历史记录文件
- 删除backup_label文件， 这个文件最开始是放在源数据库实例目录下，必须删除，不然源数据库重启时，会读该文件从而影响正常的恢复过程。

```c++
pg_stop_backup ( exclusive boolean [, wait_for_archive boolean ] ) → setof record ( lsn pg_lsn, labelfile text, spcmapfile text )
```
完成排他或非排他联机备份。exclusive参数必须与前面的`pg_start_backup`调用相匹配。 在排他备份中， `pg_stop_backup`删除备份标签文件，如果存在，则删除`pg_start_backup`创建的表空间映射文件。 在非排他备份中，这些文件的所需内容将作为函数结果的一部分返回，并且应该写入备份区域(不在数据目录)中的文件。

还有一个可选的boolean类型的第二个参数。如果为假，则该函数将在备份完成后立即返回，而无需等待WAL被归档。 这种行为只有在独立监控WAL归档的备份软件中才有用。否则，使备份一致所需的WAL可能会丢失，从而使备份无效。 默认情况下或当该参数为真时，pg_stop_backup将在启用归档时等待WAL被归档。 (在备用状态下，这意味着只有当archive_mode = always时，它才会等待。 如果主节点上的写活动很少，那么可以在主节点上运行pg_switch_wal来触发立即段切换。)

当在主节点上执行时，这个函数还会在预写式日志归档区域中创建一个备份历史文件。 历史文件包括给予pg_start_backup的标签，备份的开始和结束写前预写式日志的位置，以及备份的开始和结束时间。 记录完结束位置后，当前的预写式日志插入点自动移到下一个预写式日志文件，以便结束的预写式日志文件可以立即归档，从而完成备份。

该函数的结果是一条记录。lsn列保持备份的结束预写式日志位置(可以再忽略)。 当结束排他备份时，第二和第三列为NULL;在非排他备份之后，它们保持标签和表空间映射文件所需的内容。

还有另外一个函数，无参数。
```c++
pg_stop_backup () → pg_lsn
```
结束执行排他在线备份。这个简化版本等同于pg_stop_backup(true, true)，只是它只返回pg_lsn结果。

源码如下：
```c++
/*
 * pg_stop_backup: finish taking an on-line backup dump
 *
 * We write an end-of-backup WAL record, and remove the backup label file
 * created by pg_start_backup, creating a backup history file in pg_wal
 * instead (whence it will immediately be archived). The backup history file
 * contains the same info found in the label file, plus the backup-end time
 * and WAL location.
 *
 * Note: this version is only called to stop an exclusive backup. The function
 *		 pg_stop_backup_v2 (overloaded as pg_stop_backup in SQL) is called to stop non-exclusive backups.
 */
Datum pg_stop_backup(PG_FUNCTION_ARGS)
{
	XLogRecPtr	stoppoint;
	SessionBackupState status = get_backup_status();

	if (status == SESSION_BACKUP_NON_EXCLUSIVE)
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("non-exclusive backup in progress"),
				 errhint("Did you mean to use pg_stop_backup('f')?")));

	/*
	 * Exclusive backups were typically started in a different connection, so
	 * don't try to verify that status of backup is set to
	 * SESSION_BACKUP_EXCLUSIVE in this function. Actual verification that an
	 * exclusive backup is in fact running is handled inside
	 * do_pg_stop_backup.
	 */
	stoppoint = do_pg_stop_backup(NULL, true, NULL);

	PG_RETURN_LSN(stoppoint);
}

/*
 * do_pg_stop_backup
 *
 * Utility function called at the end of an online backup. It cleans up the
 * backup state and can optionally wait for WAL segments to be archived.
 *
 * If labelfile is NULL, this stops an exclusive backup. Otherwise this stops
 * the non-exclusive backup specified by 'labelfile'.
 *
 * Returns the last WAL location that must be present to restore from this
 * backup, and the corresponding timeline ID in *stoptli_p.
 */
XLogRecPtr
do_pg_stop_backup(char *labelfile, bool waitforarchive, TimeLineID *stoptli_p)
{
	bool		exclusive = (labelfile == NULL);
	bool		backup_started_in_recovery = false;
	XLogRecPtr	startpoint;
	XLogRecPtr	stoppoint;
	TimeLineID	stoptli;
	pg_time_t	stamp_time;
	char		strfbuf[128];
	char		histfilepath[MAXPGPATH];
	char		startxlogfilename[MAXFNAMELEN];
	char		stopxlogfilename[MAXFNAMELEN];
	char		lastxlogfilename[MAXFNAMELEN];
	char		histfilename[MAXFNAMELEN];
	char		backupfrom[20];
	XLogSegNo	_logSegNo;
	FILE	   *lfp;
	FILE	   *fp;
	char		ch;
	int			seconds_before_warning;
	int			waits = 0;
	bool		reported_waiting = false;
	char	   *remaining;
	char	   *ptr;
	uint32		hi,lo;

  // ...

	if (exclusive)
	{
		/*
		 * At first, mark that we're now stopping an exclusive backup, to
		 * ensure that there are no other sessions currently running
		 * pg_start_backup() or pg_stop_backup().
		 */
		WALInsertLockAcquireExclusive();
		if (XLogCtl->Insert.exclusiveBackupState != EXCLUSIVE_BACKUP_IN_PROGRESS)
		{
			WALInsertLockRelease();
			ereport(ERROR,
					(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
					 errmsg("exclusive backup not in progress")));
		}
		XLogCtl->Insert.exclusiveBackupState = EXCLUSIVE_BACKUP_STOPPING;
		WALInsertLockRelease();

		/*
		 * Remove backup_label. In case of failure, the state for an exclusive
		 * backup is switched back to in-progress.
		 */
		PG_ENSURE_ERROR_CLEANUP(pg_stop_backup_callback, (Datum) BoolGetDatum(exclusive));
		{
      // ...
      // 删除backup_label文件
			/*
			 * Close and remove the backup label file
			 */
			if (r != 1 || ferror(lfp) || FreeFile(lfp))
				ereport(ERROR,
						(errcode_for_file_access(),
						 errmsg("could not read file \"%s\": %m",
								BACKUP_LABEL_FILE)));
			durable_unlink(BACKUP_LABEL_FILE, ERROR);

			/*
			 * Remove tablespace_map file if present, it is created only if
			 * there are tablespaces.
			 */
			durable_unlink(TABLESPACE_MAP, DEBUG1);
		}
		PG_END_ENSURE_ERROR_CLEANUP(pg_stop_backup_callback, (Datum) BoolGetDatum(exclusive));
	}

	/*
	 * OK to update backup counters, forcePageWrites and session-level lock.
	 *
	 * Note that CHECK_FOR_INTERRUPTS() must not occur while updating them.
	 * Otherwise they can be updated inconsistently, and which might cause
	 * do_pg_abort_backup() to fail.
	 */
	WALInsertLockAcquireExclusive();
	if (exclusive)
	{
		XLogCtl->Insert.exclusiveBackupState = EXCLUSIVE_BACKUP_NONE;
	}
	else
	{
		/*
		 * The user-visible pg_start/stop_backup() functions that operate on
		 * exclusive backups can be called at any time, but for non-exclusive
		 * backups, it is expected that each do_pg_start_backup() call is
		 * matched by exactly one do_pg_stop_backup() call.
		 */
		Assert(XLogCtl->Insert.nonExclusiveBackups > 0);
		XLogCtl->Insert.nonExclusiveBackups--;
	}

	if (XLogCtl->Insert.exclusiveBackupState == EXCLUSIVE_BACKUP_NONE &&
		XLogCtl->Insert.nonExclusiveBackups == 0)
	{
		XLogCtl->Insert.forcePageWrites = false;    // 关闭强制full_page_writes
	}

	/*
	 * Clean up session-level lock.
	 *
	 * You might think that WALInsertLockRelease() can be called before
	 * cleaning up session-level lock because session-level lock doesn't need
	 * to be protected with WAL insertion lock. But since
	 * CHECK_FOR_INTERRUPTS() can occur in it, session-level lock must be
	 * cleaned up before it.
	 */
	sessionBackupState = SESSION_BACKUP_NONE;

	WALInsertLockRelease();

	/*
	 * Read and parse the START WAL LOCATION line (this code is pretty crude,
	 * but we are not expecting any variability in the file format).
	 */
	if (sscanf(labelfile, "START WAL LOCATION: %X/%X (file %24s)%c",
			   &hi, &lo, startxlogfilename,
			   &ch) != 4 || ch != '\n')
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("invalid data in file \"%s\"", BACKUP_LABEL_FILE)));
	startpoint = ((uint64) hi) << 32 | lo;
	remaining = strchr(labelfile, '\n') + 1;	/* %n is not portable enough */

	/*
	 * Parse the BACKUP FROM line. If we are taking an online backup from the
	 * standby, we confirm that the standby has not been promoted during the
	 * backup.
	 */
	ptr = strstr(remaining, "BACKUP FROM:");
	if (!ptr || sscanf(ptr, "BACKUP FROM: %19s\n", backupfrom) != 1)
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("invalid data in file \"%s\"", BACKUP_LABEL_FILE)));
	if (strcmp(backupfrom, "standby") == 0 && !backup_started_in_recovery)
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("the standby was promoted during online backup"),
				 errhint("This means that the backup being taken is corrupt "
						 "and should not be used. "
						 "Try taking another online backup.")));

	/*
	 * During recovery, we don't write an end-of-backup record. We assume that
	 * pg_control was backed up last and its minimum recovery point can be
	 * available as the backup end location. Since we don't have an
	 * end-of-backup record, we use the pg_control value to check whether
	 * we've reached the end of backup when starting recovery from this
	 * backup. We have no way of checking if pg_control wasn't backed up last
	 * however.
	 *
	 * We don't force a switch to new WAL file but it is still possible to
	 * wait for all the required files to be archived if waitforarchive is
	 * true. This is okay if we use the backup to start a standby and fetch
	 * the missing WAL using streaming replication. But in the case of an
	 * archive recovery, a user should set waitforarchive to true and wait for
	 * them to be archived to ensure that all the required files are
	 * available.
	 *
	 * We return the current minimum recovery point as the backup end
	 * location. Note that it can be greater than the exact backup end
	 * location if the minimum recovery point is updated after the backup of
	 * pg_control. This is harmless for current uses.
	 *
	 * XXX currently a backup history file is for informational and debug
	 * purposes only. It's not essential for an online backup. Furthermore,
	 * even if it's created, it will not be archived during recovery because
	 * an archiver is not invoked. So it doesn't seem worthwhile to write a
	 * backup history file during recovery.
	 */
	if (backup_started_in_recovery)
	{
      // ...
	}
	else
	{
    // 写入一条备份结束XLOG记录
		/* Write the backup-end xlog record */
		XLogBeginInsert();
		XLogRegisterData((char *) (&startpoint), sizeof(startpoint));
		stoppoint = XLogInsert(RM_XLOG_ID, XLOG_BACKUP_END);
		stoptli = ThisTimeLineID;

		/*
		 * Force a switch to a new xlog segment file, so that the backup is
		 * valid as soon as archiver moves out the current segment file. */
		RequestXLogSwitch(false);   // 切换日志段文件，以便尽快归档，减少等待归档结束的时间

		XLByteToPrevSeg(stoppoint, _logSegNo, wal_segment_size);
		XLogFileName(stopxlogfilename, stoptli, _logSegNo, wal_segment_size);

		/* Use the log timezone here, not the session timezone */
		stamp_time = (pg_time_t) time(NULL);
		pg_strftime(strfbuf, sizeof(strfbuf),"%Y-%m-%d %H:%M:%S %Z", pg_localtime(&stamp_time, log_timezone));

		/* Write the backup history file */
		XLByteToSeg(startpoint, _logSegNo, wal_segment_size);
		BackupHistoryFilePath(histfilepath, stoptli, _logSegNo,  startpoint, wal_segment_size);
		fp = AllocateFile(histfilepath, "w");
		if (!fp)
			ereport(ERROR,
					(errcode_for_file_access(),
					 errmsg("could not create file \"%s\": %m",
							histfilepath)));
		fprintf(fp, "START WAL LOCATION: %X/%X (file %s)\n",
				LSN_FORMAT_ARGS(startpoint), startxlogfilename);
		fprintf(fp, "STOP WAL LOCATION: %X/%X (file %s)\n",
				LSN_FORMAT_ARGS(stoppoint), stopxlogfilename);

		/* Transfer remaining lines including label and start timeline to history file.*/
		fprintf(fp, "%s", remaining);
		fprintf(fp, "STOP TIME: %s\n", strfbuf);
		fprintf(fp, "STOP TIMELINE: %u\n", stoptli);
		if (fflush(fp) || ferror(fp) || FreeFile(fp))
			ereport(ERROR, (errcode_for_file_access(), errmsg("could not write file \"%s\": %m", histfilepath)));

		/* Clean out any no-longer-needed history files.  As a side effect,
		 * this will post a .ready file for the newly created history file,
		 * notifying the archiver that history file may be archived immediately. */
		CleanupBackupHistory();
	}

  // 等待归档结束
	/*
	 * If archiving is enabled, wait for all the required WAL files to be
	 * archived before returning. If archiving isn't enabled, the required WAL
	 * needs to be transported via streaming replication (hopefully with
	 * wal_keep_size set high enough), or some more exotic mechanism like
	 * polling and copying files from pg_wal with script. We have no knowledge
	 * of those mechanisms, so it's up to the user to ensure that he gets all
	 * the required WAL.
	 *
	 * We wait until both the last WAL file filled during backup and the
	 * history file have been archived, and assume that the alphabetic sorting
	 * property of the WAL files ensures any earlier WAL files are safely
	 * archived as well.
	 *
	 * We wait forever, since archive_command is supposed to work and we
	 * assume the admin wanted his backup to work completely. If you don't
	 * wish to wait, then either waitforarchive should be passed in as false,
	 * or you can set statement_timeout.  Also, some notices are issued to
	 * clue in anyone who might be doing this interactively. */

	if (waitforarchive && ((!backup_started_in_recovery && XLogArchivingActive()) || (backup_started_in_recovery && XLogArchivingAlways())))
	{
		XLByteToPrevSeg(stoppoint, _logSegNo, wal_segment_size);
		XLogFileName(lastxlogfilename, stoptli, _logSegNo, wal_segment_size);

		XLByteToSeg(startpoint, _logSegNo, wal_segment_size);
		BackupHistoryFileName(histfilename, stoptli, _logSegNo, startpoint, wal_segment_size);

		seconds_before_warning = 60;
		waits = 0;

		while (XLogArchiveIsBusy(lastxlogfilename) || XLogArchiveIsBusy(histfilename))
		{
			CHECK_FOR_INTERRUPTS();

			if (!reported_waiting && waits > 5)
			{
				ereport(NOTICE, (errmsg("base backup done, waiting for required WAL segments to be archived")));
				reported_waiting = true;
			}

			pgstat_report_wait_start(WAIT_EVENT_BACKUP_WAIT_WAL_ARCHIVE);
			pg_usleep(1000000L);
			pgstat_report_wait_end();

			if (++waits >= seconds_before_warning)
			{
				seconds_before_warning *= 2;	/* This wraps in >10 years... */
				ereport(WARNING,
						(errmsg("still waiting for all required WAL segments to be archived (%d seconds elapsed)",
								waits),
						 errhint("Check that your archive_command is executing properly.  "
								 "You can safely cancel this backup, "
								 "but the database backup will not be usable without all the WAL segments.")));
			}
		}

		ereport(NOTICE,
				(errmsg("all required WAL segments have been archived")));
	}
	else if (waitforarchive)
		ereport(NOTICE,
				(errmsg("WAL archiving is not enabled; you must ensure that all required WAL segments are copied through other means to complete the backup")));

	/* We're done.  As a convenience, return the ending WAL location.*/
	if (stoptli_p)
		*stoptli_p = stoptli;
	return stoppoint;
}
```

#### 恢复源码分析
执行备份：
```sql
postgres=# select pg_start_backup('bak3');
 pg_start_backup 
-----------------
 0/6000060
(1 row)

postgres=# insert into t1 values(5);
INSERT 0 1
postgres=# select pg_stop_backup();
NOTICE:  WAL archiving is not enabled; you must ensure that all required WAL segments are copied through other means to complete the backup
 pg_stop_backup 
----------------
 0/60002D8
(1 row)

```
查看日志：
```sql
postgres@slpc:~/pgsql/pgdata/pg_wal$ pg_waldump -p ../pg_wal 000000010000000000000006
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/06000028, prev 0/05000110, desc: RUNNING_XACTS nextXid 738 latestCompletedXid 737 oldestRunningXid 738
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/06000060, prev 0/06000028, desc: RUNNING_XACTS nextXid 738 latestCompletedXid 737 oldestRunningXid 738
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/06000098, prev 0/06000060, desc: CHECKPOINT_ONLINE redo 0/6000060; tli 1; prev tli 1; fpw true; xid 0:738; oid 16387; multi 1; offset 0; oldest xid 726 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 738; online
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/06000110, prev 0/06000098, desc: RUNNING_XACTS nextXid 738 latestCompletedXid 737 oldestRunningXid 738
rmgr: Heap        len (rec/tot):     54/   258, tx:        738, lsn: 0/06000148, prev 0/06000110, desc: INSERT off 5 flags 0x00, blkref #0: rel 1663/13010/16384 blk 0 FPW
rmgr: Transaction len (rec/tot):     34/    34, tx:        738, lsn: 0/06000250, prev 0/06000148, desc: COMMIT 2023-09-18 14:40:06.694650 CST
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/06000278, prev 0/06000250, desc: RUNNING_XACTS nextXid 739 latestCompletedXid 738 oldestRunningXid 739
rmgr: XLOG        len (rec/tot):     34/    34, tx:          0, lsn: 0/060002B0, prev 0/06000278, desc: BACKUP_END 0/6000060
rmgr: XLOG        len (rec/tot):     24/    24, tx:          0, lsn: 0/060002D8, prev 0/060002B0, desc: SWITCH  
```
查看backup_label文件：
```sql
postgres@slpc:~/pgsql/pgbak2$ cat backup_label 
START WAL LOCATION: 0/6000060 (file 000000010000000000000006)
CHECKPOINT LOCATION: 0/6000098
BACKUP METHOD: pg_start_backup
BACKUP FROM: primary
START TIME: 2023-09-18 14:39:50 CST
LABEL: bak3
START TIMELINE: 1
```
恢复源码分析：
启动备份数据库，检测到有backup_label文件时，则认为是从一个备份文件中进行恢复，读取backup_label中的检查点信息，而不是从pg_control中读取。
```c++
main(int argc, char *argv[])
--> PostmasterMain(argc, argv);
	--> LocalProcessControlFile(false);		// 读pg_control文件
	--> StartupPID = StartupDataBase();		// 启动startup子进程
		--> StartupProcessMain();
			--> StartupXLOG();
				--> ValidateXLOGDirectoryStructure();   // Verify that pg_wal and pg_wal/archive_status exist.
				--> readRecoverySignalFile();		// Check for signal files, and if so set up state for offline recovery
				--> validateRecoveryParameters();
				--> XLogReaderAllocate    // Allocate and initialize a new XLogReader.
				// 是否存在backup_label文件，如果存在的话，则认为是从一个备份文件进行恢复
				--> read_backup_label(&checkPointLoc, &backupEndRequired, &backupFromStandby)
				--> record = ReadCheckpointRecord(xlogreader, checkPointLoc, 0, true); //回放的起点为backup_label中的检查点
					--> XLogBeginRead(xlogreader, RecPtr);   // Begin reading WAL at 'RecPtr'.
					--> record = ReadRecord(xlogreader, LOG, true);
									for (;;)
									{
										record = XLogReadRecord(xlogreader, &errormsg);		// Attempt to read an XLOG record.
									}
				--> StartupCLOG();
				/* REDO */
				if (InRecovery)
				{
					UpdateControlFile();
					CheckRecoveryConsistency();
					if (checkPoint.redo < RecPtr)
					{
						/* back up to find the record */
						XLogBeginRead(xlogreader, checkPoint.redo);
						record = ReadRecord(xlogreader, PANIC, false);
					} else {
						/* just have to read next record after CheckPoint */
						record = ReadRecord(xlogreader, LOG, false);
					}

					if (record != NULL)
					{
						/* main redo apply loop */
						do  // 回放日志
						{	
							// 判断否已达到指定恢复位置，PITR用
							if (recoveryStopsBefore(xlogreader))
							{
								reachedRecoveryTarget = true;
								break;
							}

							/* Now apply the WAL record itself */
							RmgrTable[record->xl_rmid].rm_redo(xlogreader);
						}
					}
				}

```
核心函数`StartupXLOG`源码分析：
```c++
void StartupXLOG(void)
{
	// ...

	/* Set up XLOG reader facility */
	MemSet(&private, 0, sizeof(XLogPageReadPrivate));
	xlogreader = XLogReaderAllocate(wal_segment_size, NULL, XL_ROUTINE(.page_read = &XLogPageRead, .segment_open = NULL, .segment_close = wal_segment_close), &private);

	// 读backup_label文件
	if (read_backup_label(&checkPointLoc, &backupEndRequired, &backupFromStandby))
	{
		/* Archive recovery was requested, and thanks to the backup label
		 * file, we know how far we need to replay to reach consistency. Enter
		 * archive recovery directly. */
		InArchiveRecovery = true;

		/* When a backup_label file is present, we want to roll forward from
		 * the checkpoint it identifies, rather than using pg_control. */
		record = ReadCheckpointRecord(xlogreader, checkPointLoc, 0, true);
		if (record != NULL)
		{
			memcpy(&checkPoint, XLogRecGetData(xlogreader), sizeof(CheckPoint));

			/* Make sure that REDO location exists. This may not be the case
			 * if there was a crash during an online backup, which left a
			 * backup_label around that references a WAL segment that's already been archived. */
			if (checkPoint.redo < checkPointLoc)
			{
				XLogBeginRead(xlogreader, checkPoint.redo);
				if (!ReadRecord(xlogreader, LOG, false))
					ereport(FATAL,(errmsg("could not find redo location referenced by checkpoint record"), errhint("If you are restoring from a backup, touch \"%s/recovery.signal\" and add required recovery options.\n" "If you are not restoring from a backup, try removing the file \"%s/backup_label\".\n" "Be careful: removing \"%s/backup_label\" will result in a corrupt cluster if restoring from a backup.", DataDir, DataDir, DataDir)));
			}
		}
		else
		{
			ereport(FATAL,
					(errmsg("could not locate required checkpoint record"),
					 errhint("If you are restoring from a backup, touch \"%s/recovery.signal\" and add required recovery options.\n"
							 "If you are not restoring from a backup, try removing the file \"%s/backup_label\".\n"
							 "Be careful: removing \"%s/backup_label\" will result in a corrupt cluster if restoring from a backup.", DataDir, DataDir, DataDir)));
			wasShutdown = false;	/* keep compiler quiet */
		}

		/* set flag to delete it later */
		haveBackupLabel = true;
	}
	else  // 如果没有backup_label文件，则读pg_control文件，在备机恢复的场景中，如果丢失了backup_label文件，而读取了pg_control文件中的检查点，则会因为回放位置不对，无法达成数据一致，恢复失败。
	{
		/* Get the last valid checkpoint record. */
		checkPointLoc = ControlFile->checkPoint;
		RedoStartLSN = ControlFile->checkPointCopy.redo;
		record = ReadCheckpointRecord(xlogreader, checkPointLoc, 1, true);
		if (record != NULL)
		{
			ereport(DEBUG1,(errmsg_internal("checkpoint record is at %X/%X", LSN_FORMAT_ARGS(checkPointLoc))));
		}
		else
		{
			/*
			 * We used to attempt to go back to a secondary checkpoint record
			 * here, but only when not in standby mode. We now just fail if we
			 * can't read the last checkpoint because this allows us to
			 * simplify processing around checkpoints.
			 */
			ereport(PANIC,(errmsg("could not locate a valid checkpoint record")));
		}
		memcpy(&checkPoint, XLogRecGetData(xlogreader), sizeof(CheckPoint));
	}

	/* REDO */
	if (InRecovery)
	{
		/*
		 * Set backupStartPoint if we're starting recovery from a base backup.
		 *
		 * Also set backupEndPoint and use minRecoveryPoint as the backup end
		 * location if we're starting recovery from a base backup which was
		 * taken from a standby. In this case, the database system status in
		 * pg_control must indicate that the database was already in recovery.
		 * Usually that will be DB_IN_ARCHIVE_RECOVERY but also can be
		 * DB_SHUTDOWNED_IN_RECOVERY if recovery previously was interrupted
		 * before reaching this point; e.g. because restore_command or primary_conninfo were faulty.
		 *
		 * Any other state indicates that the backup somehow became corrupted and we can't sensibly continue with recovery.
		 */
		if (haveBackupLabel)
		{
			ControlFile->backupStartPoint = checkPoint.redo;		// 从基础备份中恢复
			ControlFile->backupEndRequired = backupEndRequired;

			if (backupFromStandby)
			{
				if (dbstate_at_startup != DB_IN_ARCHIVE_RECOVERY &&
					dbstate_at_startup != DB_SHUTDOWNED_IN_RECOVERY)
					ereport(FATAL,
							(errmsg("backup_label contains data inconsistent with control file"),
							 errhint("This means that the backup is corrupted and you will "
									 "have to use another backup for recovery.")));
				ControlFile->backupEndPoint = ControlFile->minRecoveryPoint;
			}
		}

		UpdateControlFile();	// 更新pg_control，主要是将Backup start location写入


		/*
		 * We're in recovery, so unlogged relations may be trashed and must be
		 * reset.  This should be done BEFORE allowing Hot Standby
		 * connections, so that read-only backends don't try to read whatever
		 * garbage is left over from before.
		 */
		ResetUnloggedRelations(UNLOGGED_RELATION_CLEANUP);

		/* Initialize resource managers */
		for (rmid = 0; rmid <= RM_MAX_ID; rmid++)
		{
			if (RmgrTable[rmid].rm_startup != NULL)
				RmgrTable[rmid].rm_startup();
		}

		CheckRecoveryConsistency();		// Checks if recovery has reached a consistent state.
	
		/*
		 * Find the first record that logically follows the checkpoint --- it
		 * might physically precede it, though. */
		if (checkPoint.redo < RecPtr)
		{
			/* back up to find the record */
			XLogBeginRead(xlogreader, checkPoint.redo);
			record = ReadRecord(xlogreader, PANIC, false);
		}
		else
		{
			/* just have to read next record after CheckPoint */
			record = ReadRecord(xlogreader, LOG, false);
		}

		if (record != NULL)
		{
			// 在这里进行实质的日志回放
			/* main redo apply loop */
			do
			{
				bool		switchedTLI = false;
				// 用于PITR，判断是否已经回放到了指定的Target
				/* Have we reached our recovery target? */
				if (recoveryStopsBefore(xlogreader))
				{
					reachedRecoveryTarget = true;
					break;
				}

				/* Now apply the WAL record itself */
				RmgrTable[record->xl_rmid].rm_redo(xlogreader);		// 调用standby_redo,xlog_redo,heap_redo，xact_redo等，进行回放，

				/* Allow read-only connections if we're consistent now */
				CheckRecoveryConsistency();

				/* Exit loop if we reached inclusive recovery target */
				if (recoveryStopsAfter(xlogreader))
				{
					reachedRecoveryTarget = true;
					break;
				}
				
				/* Else, try to fetch the next WAL record */
				record = ReadRecord(xlogreader, LOG, false);	
			} while (record != NULL);  // 直到结束
		}
	}

	/*
	 * Determine where to start writing WAL next.
	 *
	 * When recovery ended in an incomplete record, write a WAL record about
	 * that and continue after it.  In all other cases, re-fetch the last
	 * valid or last applied record, so we can identify the exact endpoint of
	 * what we consider the valid portion of WAL.
	 */
	XLogBeginRead(xlogreader, LastRec);
	record = ReadRecord(xlogreader, PANIC, false);
	EndOfLog = EndRecPtr;

	// ...

}
```
一直回放到`XLOG_BACKUP_END`，
```c++
/*
 * XLOG resource manager's routines
 *
 * Definitions of info values are in include/catalog/pg_control.h, though
 * not all record types are related to control file updates.
 */
void xlog_redo(XLogReaderState *record)
{
	uint8		info = XLogRecGetInfo(record) & ~XLR_INFO_MASK;
	XLogRecPtr	lsn = record->EndRecPtr;


	if (info == XLOG_NEXTOID)
	{
		// ...
	}
	else if (info == XLOG_CHECKPOINT_SHUTDOWN)
	{
		CheckPoint	checkPoint;
		memcpy(&checkPoint, XLogRecGetData(record), sizeof(CheckPoint));

		// ...

		RecoveryRestartPoint(&checkPoint);
	}
	else if (info == XLOG_CHECKPOINT_ONLINE)
	{
		CheckPoint	checkPoint;
		memcpy(&checkPoint, XLogRecGetData(record), sizeof(CheckPoint));
		// ...

		RecoveryRestartPoint(&checkPoint);
	}
	else if (info == XLOG_OVERWRITE_CONTRECORD)
	{
		xl_overwrite_contrecord xlrec;

		memcpy(&xlrec, XLogRecGetData(record), sizeof(xl_overwrite_contrecord));
		VerifyOverwriteContrecord(&xlrec, record);
	}
	else if (info == XLOG_END_OF_RECOVERY)
	{
		xl_end_of_recovery xlrec;

		memcpy(&xlrec, XLogRecGetData(record), sizeof(xl_end_of_recovery));

		/*
		 * For Hot Standby, we could treat this like a Shutdown Checkpoint,
		 * but this case is rarer and harder to test, so the benefit doesn't
		 * outweigh the potential extra cost of maintenance.
		 */

		/*
		 * We should've already switched to the new TLI before replaying this
		 * record.
		 */
		if (xlrec.ThisTimeLineID != ThisTimeLineID)
			ereport(PANIC,
					(errmsg("unexpected timeline ID %u (should be %u) in checkpoint record",
							xlrec.ThisTimeLineID, ThisTimeLineID)));
	}
	else if (info == XLOG_NOOP)
	{
		/* nothing to do here */
	}
	else if (info == XLOG_SWITCH)
	{
		/* nothing to do here */
	}
	else if (info == XLOG_RESTORE_POINT)
	{
		/* nothing to do here */
	}
	else if (info == XLOG_FPI || info == XLOG_FPI_FOR_HINT)
	{
		/*
		 * Full-page image (FPI) records contain nothing else but a backup
		 * block (or multiple backup blocks). Every block reference must
		 * include a full-page image - otherwise there would be no point in
		 * this record.
		 *
		 * No recovery conflicts are generated by these generic records - if a
		 * resource manager needs to generate conflicts, it has to define a
		 * separate WAL record type and redo routine.
		 *
		 * XLOG_FPI_FOR_HINT records are generated when a page needs to be
		 * WAL- logged because of a hint bit update. They are only generated
		 * when checksums are enabled. There is no difference in handling
		 * XLOG_FPI and XLOG_FPI_FOR_HINT records, they use a different info
		 * code just to distinguish them for statistics purposes.
		 */
		for (uint8 block_id = 0; block_id <= record->max_block_id; block_id++)
		{
			Buffer		buffer;

			if (XLogReadBufferForRedo(record, block_id, &buffer) != BLK_RESTORED)
				elog(ERROR, "unexpected XLogReadBufferForRedo result when restoring backup block");
			UnlockReleaseBuffer(buffer);
		}
	}
	else if (info == XLOG_BACKUP_END)	// 回放到这里，结束备份恢复过程
	{
		XLogRecPtr	startpoint;

		memcpy(&startpoint, XLogRecGetData(record), sizeof(startpoint));

		if (ControlFile->backupStartPoint == startpoint)
		{
			/*
			 * We have reached the end of base backup, the point where
			 * pg_stop_backup() was done. The data on disk is now consistent.
			 * Reset backupStartPoint, and update minRecoveryPoint to make
			 * sure we don't allow starting up at an earlier point even if
			 * recovery is stopped and restarted soon after this.
			 */
			elog(DEBUG1, "end of backup reached");

			LWLockAcquire(ControlFileLock, LW_EXCLUSIVE);

			if (ControlFile->minRecoveryPoint < lsn)
			{
				ControlFile->minRecoveryPoint = lsn;
				ControlFile->minRecoveryPointTLI = ThisTimeLineID;
			}
			ControlFile->backupStartPoint = InvalidXLogRecPtr;
			ControlFile->backupEndRequired = false;
			UpdateControlFile();

			LWLockRelease(ControlFileLock);
		}
	}
	else if (info == XLOG_PARAMETER_CHANGE)
	{
		// ...
	}
	else if (info == XLOG_FPW_CHANGE)
	{
		// ...
	}
}
```


---
参考文档：
[9.27. 系统管理函数](http://postgres.cn/docs/14/functions-admin.html#FUNCTIONS-ADMIN-BACKUP)


