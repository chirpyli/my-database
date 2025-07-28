## PostgreSQL源码分析—— pg_cron插件
pg_cron插件是PostgreSQL的一个定时任务调度插件，其主要功能是让用户能够在数据库内部设定定时任务，与操作系统里的cron服务功能类似。


### pg_cron用法
定时任务用法基本等同于cron，与Linux中cron不同的是需要通过函数来完成定时任务的添加`cron.schedule`、删除`cron.unschedule`、查看`cron.jon`定时任务的命令，具体用法如下：

安装插件：
```sql
CREATE EXTENSION pg_cron;
```
增加/ 删除定时任务：
```sql
-- 每当分钟值为00分钟时执行一次插入1条数据
postgres=# select cron.schedule('insert-t1','0 * * * *','insert into t1 values(1)');
 schedule 
----------
        1
(1 row)

-- 每30秒执行一次插入1条数据
postgres=# select cron.schedule('insert-t1','30 seconds','insert into t1 values(1)');
 schedule 
----------
        1
(1 row)

-- 每小时执行一次插入1条数据
postgres=# select cron.schedule('insert-t1-job3','@hourly','insert into t1 values(3)');
 schedule 
----------
        3
(1 row)

-- 开机定时任务，执行vacuum
postgres=# select cron.schedule('job-reboot','@reboot','vacuum');
 schedule 
----------
        4
(1 row)

-- 通过`cron.unschedule`删除定时任务
postgres=# select cron.unschedule(1);
 unschedule 
------------
 t
(1 row)

-- 删除定时任务
postgres=# select cron.unschedule('job-reboot');
 unschedule 
------------
 t
(1 row)
```
查询定时任务，定时任务存储在表`cron.job`中，可以通过查询表来查看定时任务：
```sql
-- cron.job表结构
postgres=# \d cron.job
                                Table "cron.job"
  Column  |  Type   | Collation | Nullable |               Default               
----------+---------+-----------+----------+-------------------------------------
 jobid    | bigint  |           | not null | nextval('cron.jobid_seq'::regclass)
 schedule | text    |           | not null | 
 command  | text    |           | not null | 
 nodename | text    |           | not null | 'localhost'::text
 nodeport | integer |           | not null | inet_server_port()
 database | text    |           | not null | current_database()
 username | text    |           | not null | CURRENT_USER
 active   | boolean |           | not null | true
 jobname  | text    |           |          | 
Indexes:
    "job_pkey" PRIMARY KEY, btree (jobid)
    "jobname_username_uniq" UNIQUE CONSTRAINT, btree (jobname, username)
Policies:
    POLICY "cron_job_policy"
      USING ((username = CURRENT_USER))
Triggers:
    cron_job_cache_invalidate AFTER INSERT OR DELETE OR UPDATE OR TRUNCATE ON cron.job FOR EACH STATEMENT EXECUTE FUNCTION cron.job_cache_invalidate()

-- 查询定时任务
postgres=# select * from cron.job;
 jobid |  schedule  |         command          | nodename  | nodeport | database | username | active |    job
name     
-------+------------+--------------------------+-----------+----------+----------+----------+--------+-------
---------
     1 | 30 seconds | insert into t1 values(1) | localhost |     5432 | postgres | postgres | t      | insert
-t1
     3 | @hourly    | insert into t1 values(3) | localhost |     5432 | postgres | postgres | t      | insert
-t1-job3
(2 rows)

```

### pg_cron实现源码分析
在PostgreSQL启动时，会启动一个pg_cron的后台background进程，该进程在数据库运行期间会一直保持活跃，专门负责监控和调度所有配置好的定时任务。
```c++
main(int argc, char *argv[])
--> PostmasterMain(argc, argv);
    --> process_shared_preload_libraries();
        --> _PG_init(void); // 调用pg_cron插件的初始化函数
				// 注册回调函数，当relation cache发生变化时，会调用该回调函数
			--> CacheRegisterRelcacheCallback(InvalidateJobCacheCallback, (Datum) 0); 
            --> RegisterBackgroundWorker(&worker); // 注册后台工作进程
    --> maybe_start_bgworkers(); // 启动后台工作进程
			// 判断是否需要启动后台工作进程
        --> if (bgworker_should_start_now(rw->rw_worker.bgw_start_time);) 
            {
                StartBackgroundWorker(rw); // 启动后台工作进程
                --> AssignPostmasterChildSlot(B_BG_WORKER); // 分配后台工作进程的slot
                --> postmaster_child_launch(B_BG_WORKER, bn->child_slot,
										 (char *) &rw->rw_worker, sizeof(BackgroundWorker), NULL);
            }
```
我们看一下函数`postmaster_child_launch`的实现：
```c++
pid_t postmaster_child_launch(BackendType child_type, int child_slot, char *startup_data, size_t startup_data_len, ClientSocket *client_sock)
{
	pid_t pid = fork_process();
	if (pid == 0)				/* child */
	{
		/* Close the postmaster's sockets */
		ClosePostmasterPorts(child_type == B_LOGGER);

		/* Detangle from postmaster */
		InitPostmasterChild();

		/* Detach shared memory if not needed. */
		if (!child_process_kinds[child_type].shmem_attach)
		{
			dsm_detach_all();
			PGSharedMemoryDetach();
		}

		MemoryContextSwitchTo(TopMemoryContext);

		MyPMChildSlot = child_slot;
		if (client_sock)
		{
			MyClientSocket = palloc(sizeof(ClientSocket));
			memcpy(MyClientSocket, client_sock, sizeof(ClientSocket));
		}

		/* Run the appropriate Main function */
		child_process_kinds[child_type].main_fn(startup_data, startup_data_len);
		pg_unreachable();		/* main_fn never returns */
	}

	return pid;
}
```
启动进程调用栈如下：
```c++
pg_cron.so!PgCronLauncherMain(Datum arg) (contrib\pg_cron\src\pg_cron.c:576)
BackgroundWorkerMain(char * startup_data, size_t startup_data_len) (src\backend\postmaster\bgworker.c:842)
postmaster_child_launch(BackendType child_type, int child_slot, char * startup_data, size_t startup_data_len, ClientSocket * client_sock) (src\backend\postmaster\launch_backend.c:274)
StartBackgroundWorker(RegisteredBgWorker * rw) (src\backend\postmaster\postmaster.c:4082)
maybe_start_bgworkers() (src\backend\postmaster\postmaster.c:4247)
LaunchMissingBackgroundProcesses() (src\backend\postmaster\postmaster.c:3335)
ServerLoop() (src\backend\postmaster\postmaster.c:1703)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1386)
main(int argc, char ** argv) (src\backend\main\main.c:230)
```

pg_cron插件会在数据库中创建一个名为`cron.job`的表，用于存储所有定时任务的配置信息。当pg_cron后台进程启动时，它会从`cron.job`表中读取所有定时任务的配置信息生成`JobList`，并根据配置信息创建相应的定时任务`TaskList`。在数据库运行期间，pg_cron后台进程会定期检查所有定时任务的执行时间，并在执行时间到达时执行相应的定时任务。

pg_cron后台进程实现
```c++
PgCronLauncherMain(Datum arg)
--> BackgroundWorkerInitializeConnection(CronTableDatabaseName, NULL, 0); // 初始化数据库连接
--> InitializeJobMetadataCache();	// 初始化定时任务元数据cron.job缓存
	--> CreateCronJobHash()
--> InitializeTaskStateHash();		// 初始化定时任务状态Hash表
	--> CreateCronTaskHash()
--> while (!got_sigterm)	// 只要没有收到终止信号就一直执行
	{
		/*
		 * Both CronReloadConfig and CronJobCacheValid are triggered by SIGHUP.
		 * ProcessConfigFile should come first, because RefreshTaskHash depends
		 * on settings that might have changed.
		 */
		if (!CronJobCacheValid)	// 如果cron.job relcache失效，则刷新定时任务元数据缓存
		{
			// reloads the cron jobs from the cron.job table.
			RefreshTaskHash();
			--> ResetJobMetadataCache();
			--> LoadCronJobList(); 	// 从cron.job表中读取所有定时任务的配置信息，生成JobList, CronJob为表中的一条定时任务信息
			--> foreach(jobCell, jobList)	// 遍历jobList，生成taskList
				{
					task = GetCronTask(job->jobId);
							// 初始化定时任务，初始状态设为CRON_TASK_WAITING
						   --> InitializeCronTash(task, jobId); 
				}
		}

		taskList = CurrentTaskList();
		currentTime = GetCurrentTimestamp();

		// 启动所有待执行的定时任务，先判断是否有reboot任务
		StartAllPendingRuns(taskList, currentTime);

		WaitForCronTasks(taskList);
		ManageCronTasks(taskList, currentTime); // 管理定时任务
		--> ManageCronTask(task, currentTime);	// 定时任务调度状态机，根据定时任务当前的状态，决定下一步的动作

	}
```
定时任务调度状态机：
```c++
typedef enum
{
	CRON_TASK_WAITING = 0,	// 等待状态，默认状态，如果条件不满足，则跳过该任务的调度，如果条件满足，则进入START状态
	CRON_TASK_START = 1,   // 启动状态，构建任务的连接信息，并进行连接测试，如果连接成功，则进入CONNECTING状态，否则进入ERROR状态
	CRON_TASK_CONNECTING = 2, // 连接状态，如果连接成功，则进入SENDING状态，否则进入ERROR状态
	CRON_TASK_SENDING = 3,	// 发送状态，如果所有条件都满足，将定时任务文本发送至PostgreSQL服务器，进入RUNNING状态，否则进入ERROR状态
	CRON_TASK_RUNNING = 4,	// 检查任务是否激活，连接是否正常。如果所有条件都满足，接收传回的任务结果并进入DONE状态，否则跳出等待进入ERROR状态。
	CRON_TASK_RECEIVING = 5,
	CRON_TASK_DONE = 6,
	CRON_TASK_ERROR = 7,  	// 任务失败，进入DONE状态。
	CRON_TASK_BGW_START = 8,	// 如果是通过BackgroundWorker启动定时任务，则启动后台进程，进入BGW_RUNNING状态
	CRON_TASK_BGW_RUNNING = 9
} CronTaskState;
```

![image](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/0739469271/CAEQYBiBgMDbms._2xgiIDI2Mjg5MmFmNTY0ZDRjY2JhM2Y0OTM1ZTRiOTU5ZDAx4052908_20231027135246.926.svg)


定时任务状态机调用栈：
```c++
libpq.so.5!PQsendQueryInternal(PGconn * conn, const char * query, _Bool newQuery) (src\interfaces\libpq\fe-exec.c:1430)
libpq.so.5!PQsendQuery(PGconn * conn, const char * query) (src\interfaces\libpq\fe-exec.c:1418)
pg_cron.so!ManageCronTask(CronTask * task, TimestampTz currentTime) (contrib\pg_cron\src\pg_cron.c:1652)
pg_cron.so!ManageCronTasks(List * taskList, TimestampTz currentTime) (contrib\pg_cron\src\pg_cron.c:1274)
pg_cron.so!PgCronLauncherMain(Datum arg) (contrib\pg_cron\src\pg_cron.c:666)
BackgroundWorkerMain(char * startup_data, size_t startup_data_len) (src\backend\postmaster\bgworker.c:842)
postmaster_child_launch(BackendType child_type, int child_slot, char * startup_data, size_t startup_data_len, ClientSocket * client_sock) (src\backend\postmaster\launch_backend.c:274)
StartBackgroundWorker(RegisteredBgWorker * rw) (src\backend\postmaster\postmaster.c:4082)
maybe_start_bgworkers() (src\backend\postmaster\postmaster.c:4247)
LaunchMissingBackgroundProcesses() (src\backend\postmaster\postmaster.c:3335)
ServerLoop() (src\backend\postmaster\postmaster.c:1703)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1386)
main(int argc, char ** argv) (src\backend\main\main.c:230)
```
任务调度
![image](../images/pg_cron_task_schedule.png)

---
参考文档：
[pg_cron（定时任务）](https://help.aliyun.com/zh/polardb/polardb-for-postgresql/pg-cron)