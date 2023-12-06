## pg_basebackup分析

涉及到的代码主要在`src/backend/replication`以及`bin/pg_basebackup`中。

我们知道pg_basebackup是一个进行基础备份的工具，除了使用这个工具，还可以用底层API的方式进行基础备份，主要过程如下：
1. 连接到数据库
2. 执行`select pg_start_backup('lable')`命令。（会强制发生一次checkpoint，并将检查点记录到backup_label文件中）
3. 执行备份，把数据目录进行复制（包含backup_label）
4. 执行`select pg_stop_backup`命令，（删除backup_label文件，并在WAL日志中写入一条`XLOG_BACKUP_END`的记录，当备节点回放到该记录时，就知道备份结束了，数据达到了一致点，可以对外提供服务了）
5. 备份过程中产生的WAL日志进行复制

其实，pg_basebackup工具就是对底层API的封装，其主要过程是相同的，但具体到代码，并不是直接调用的`pg_start_backup`,`pg_stop_backup`函数，而是通过一些命令的形式，这些特殊的命令定义在`src/backend/replication/repl_gram.y`中，后面我们会进行分析。

### 主流程
pg_basebackup执行基础备份的主要流程如下，其中，涉及到libpq协议与服务端进行连接，通信，向服务端发送一些特殊的命令语句，这些命令的解析在`src/backend/replication/repl_gram.y`中可以查看到具体的语法定义。主流程如下：

```c++
main(int argc, char **argv)
--> GetConnection();   // 连接服务端，（例如主节点）
    --> PQconnectdbParams(keywords, values, true);
--> BaseBackup();      // 执行基础备份
    --> GenerateRecoveryConfig(conn, replication_slot);   // 用于生成primary_conninfo配置信息
        --> PQconninfo(pgconn);
        // 向服务端发送IDENTIFY_SYSTEM命令，返回timeline和system identifier. 
    --> RunIdentifySystem(conn, &sysidentifier, &latesttli, NULL, NULL) 

    basebkp = psprintf("BASE_BACKUP LABEL '%s' %s %s %s %s %s %s %s %s %s",
				 escaped_label,
				 estimatesize ? "PROGRESS" : "",
				 includewal == FETCH_WAL ? "WAL" : "",
				 fastcheckpoint ? "FAST" : "",
				 includewal == NO_WAL ? "" : "NOWAIT",
				 maxrate_clause ? maxrate_clause : "",
				 format == 't' ? "TABLESPACE_MAP" : "",
				 verify_checksums ? "" : "NOVERIFY_CHECKSUMS",
				 manifest_clause ? manifest_clause : "",
				 manifest_checksums_clause);

    // 向服务端发送执行命令"BASE_BACKUP LABEL 'pg14bak' PROGRESS   NOWAIT    MANIFEST 'yes' "
    --> PQsendQuery(conn, basebkp)    
    --> PQgetResult(conn);    // Get the starting WAL location

    --> StartLogStreamer(xlogstart, starttli, sysidentifier);
        --> CreateReplicationSlot(param->bgconn, replication_slot, NULL, temp_replication_slot, true, true, false)
            // 执行命令 "CREATE_REPLICATION_SLOT \"pg_basebackup_2553309\" TEMPORARY PHYSICAL RESERVE_WAL"
            --> PQexec(conn, query->data);


      // Start a child process and tell it to start streaming.
    	// 创建一个单独的子进程用于日志传输
      bgchild = fork();
      if (bgchild == 0)
      {
          /* in child process */
          LogStreamerMain(param);
          --> ReceiveXlogStream(param->bgconn, &stream)   // Receive a log stream starting at the specified position.
      }

    if (!writing_to_stdout && manifest)
        ReceiveBackupManifest(conn);    // receive backup manifest

```

整个的过程，最重要的有3点：
1. 在进行备份前，执行一次checkpoint，记录开始的位置，在服务端接收到`BASE_BACKUP LABLE`命令后，生成备份标签文件`backup_lable`，这个文件最重要的作用是记录数据库恢复的起始位置。当启动备份实例时，会读该文件进行恢复。
2. 复制数据库数据文件
3. 日志复制

我们可以看一下`backup_lable`文件中的内容：
```shell
postgres@slpc:~/pgsql/pgbak$ ls
backup_label      pg_commit_ts   pg_notify     pg_subtrans  postgresql.auto.conf
backup_manifest   pg_dynshmem    pg_replslot   pg_tblspc    postgresql.conf
base              pg_hba.conf    pg_serial     pg_twophase  standby.signal
current_logfiles  pg_ident.conf  pg_snapshots  PG_VERSION
global            pg_logical     pg_stat       pg_wal
log               pg_multixact   pg_stat_tmp   pg_xact
postgres@slpc:~/pgsql/pgbak$ cat backup_label 
START WAL LOCATION: 0/C000028 (file 00000001000000000000000C)    备份开始时日志的位置
CHECKPOINT LOCATION: 0/C000060                                   检查点的位置
BACKUP METHOD: streamed                                          备份方法
BACKUP FROM: primary                                             备份源    
START TIME: 2023-08-02 17:15:58 CST                              备份开始的物理时间
LABEL: pg14bak                                                   备份标签
START TIMELINE: 1
```

### 具体过程
仅看上面的主流程还是有一些不清楚的地方的。

这里有个很重要的命令`BASE_BACKUP LABEL`，备份命令，获取XLOG的存放路径和备份开始时日志的位置，那么服务端这块是怎么处理的呢？我们看一下服务端的相关代码：

```c++
void PostgresMain(int argc, char *argv[], const char *dbname, const char *username)
{
    // ...

    if (am_walsender)
        WalSndSignals();

    /* Perform initialization specific to a WAL sender process. */
    if (am_walsender)
      InitWalSender();

    for (;;)
    {
		    firstchar = ReadCommand(&input_message);
        switch (firstchar)
        {
          case 'Q':			/* simple query */
            {
              const char *query_string;

              /* Set statement_timestamp() */
              SetCurrentStatementStartTimestamp();

              query_string = pq_getmsgstring(&input_message);
              pq_getmsgend(&input_message);

              if (am_walsender)
              {
                if (!exec_replication_command(query_string))
                  exec_simple_query(query_string);
              }
              else
                exec_simple_query(query_string);

              send_ready_for_query = true;
            }
            break;
        }
}

/* Execute an incoming replication command.*/
bool exec_replication_command(const char *cmd_string)
{
    // ...

	/* Looks like a WalSender command, so parse it. */
	parse_rc = replication_yyparse();
	if (parse_rc != 0)
		ereport(ERROR,
				(errcode(ERRCODE_SYNTAX_ERROR),
				 errmsg_internal("replication command parser returned %d",
								 parse_rc)));


    switch (cmd_node->type)
    {
      case T_IdentifySystemCmd:
        cmdtag = "IDENTIFY_SYSTEM";
        set_ps_display(cmdtag);
        IdentifySystem();
        EndReplicationCommand(cmdtag);
        break;

      case T_BaseBackupCmd:
        cmdtag = "BASE_BACKUP";
        set_ps_display(cmdtag);
        PreventInTransactionBlock(true, cmdtag);
        SendBaseBackup((BaseBackupCmd *) cmd_node);
        EndReplicationCommand(cmdtag);
        break;

      case T_CreateReplicationSlotCmd:
        cmdtag = "CREATE_REPLICATION_SLOT";
        set_ps_display(cmdtag);
        CreateReplicationSlot((CreateReplicationSlotCmd *) cmd_node);
        EndReplicationCommand(cmdtag);
        break;

      case T_DropReplicationSlotCmd:
        cmdtag = "DROP_REPLICATION_SLOT";
        set_ps_display(cmdtag);
        DropReplicationSlot((DropReplicationSlotCmd *) cmd_node);
        EndReplicationCommand(cmdtag);
        break;

      case T_StartReplicationCmd:
        {
          StartReplicationCmd *cmd = (StartReplicationCmd *) cmd_node;

          cmdtag = "START_REPLICATION";
          set_ps_display(cmdtag);
          PreventInTransactionBlock(true, cmdtag);

          if (cmd->kind == REPLICATION_KIND_PHYSICAL)
            StartReplication(cmd);
          else
            StartLogicalReplication(cmd);

          /* dupe, but necessary per libpqrcv_endstreaming */
          EndReplicationCommand(cmdtag);

          Assert(xlogreader != NULL);
          break;
        }

      case T_TimeLineHistoryCmd:
        cmdtag = "TIMELINE_HISTORY";
        set_ps_display(cmdtag);
        PreventInTransactionBlock(true, cmdtag);
        SendTimeLineHistory((TimeLineHistoryCmd *) cmd_node);
        EndReplicationCommand(cmdtag);
        break;

      case T_VariableShowStmt:
        {
          DestReceiver *dest = CreateDestReceiver(DestRemoteSimple);
          VariableShowStmt *n = (VariableShowStmt *) cmd_node;

          cmdtag = "SHOW";
          set_ps_display(cmdtag);

          /* syscache access needs a transaction environment */
          StartTransactionCommand();
          GetPGVariable(n->name, dest);
          CommitTransactionCommand();
          EndReplicationCommand(cmdtag);
        }
        break;

      default:
        elog(ERROR, "unrecognized replication command node tag: %u",
          cmd_node->type);
    }

    // ...
}
```
我们继续看一下`SendBaseBackup`函数的实现。
```c++
/*
 * SendBaseBackup() - send a complete base backup.
 *
 * The function will put the system into backup mode like pg_start_backup()
 * does, so that the backup is consistent even though we read directly from
 * the filesystem, bypassing the buffer cache.
 */
void
SendBaseBackup(BaseBackupCmd *cmd)
{
	basebackup_options opt;
	SessionBackupState status = get_backup_status();

	if (status == SESSION_BACKUP_NON_EXCLUSIVE)
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("a backup is already in progress in this session")));

	parse_basebackup_options(cmd->options, &opt);

	WalSndSetState(WALSNDSTATE_BACKUP);

	if (update_process_title)
	{
		char		activitymsg[50];

		snprintf(activitymsg, sizeof(activitymsg), "sending backup \"%s\"",
				 opt.label);
		set_ps_display(activitymsg);
	}

	perform_base_backup(&opt);
}
```
主流程如下：
```c++
SendBaseBackup(BaseBackupCmd *cmd)
--> perform_base_backup(&opt);
    // creates the necessary starting checkpoint and constructs the backup label file.
    --> do_pg_start_backup(opt->label, opt->fastcheckpoint, &starttli, labelfile, &tablespaces, tblspc_map_file);
        --> RequestCheckpoint(CHECKPOINT_FORCE | CHECKPOINT_WAIT | (fast ? CHECKPOINT_IMMEDIATE : 0));
            --> CreateCheckPoint(flags | CHECKPOINT_IMMEDIATE);
    --> do_pg_stop_backup(labelfile->data, !opt->nowait, &endtli);
```
