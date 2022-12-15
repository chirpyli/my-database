## citus建分布表分析

### 实现思路
citus中有distributed table, reference table, local table三种表，我们重点分析一下distributed table的建表过程。即分布表建表过程。这个过程与分区表思路有点类似，不同的是分布表是要将一张表分割为多个分片分布在不同的worker节点上，这样就可以通过部署更多的机器实现水平扩展。在分析源码前，我们先讲一下其大致的实现思路，就是CN节点负责表的元数据管理，worker节点上单独建表，但是每个表(shard)是分布表的其中一个分片，所有worker节点上的分片组合起来就是一张分布表。
```sql
-- cn dn1 dn2 3个节点
-- CN 节点建表
create table t1(a int, b int);
-- cn上执行创建分布表，shard_count=2
select create_distributed_table('t1','a');  
-- 在cn上会根据shard_count划分为2个分片，构造Task，发往worker节点
-- 发往worker节点的命令为："SELECT worker_apply_shard_ddl_command (shardid, 'CREATE TABLE public.t1 (a integer, b integer) USING heap')"
-- worker 1 会创建一个分片
-- worker 2 会创建一个分片
```

### 源码分析
我们构造如下分析语句，进行源码分析
```sql
create table t1(a int, b int);
select create_distributed_table('t1','a');
```
CN节点调用的流程如下：
```c++
exec_simple_query
--> pg_parse_query               // 语法解析
--> pg_analyze_and_rewrite       // 语义分析
	--> parse_analyze
		--> transformStmt
			--> transformSelectStmt
				--> transformTargetList
					--> transformTargetEntry
						--> transformFuncCall
--> pg_plan_queries    // 生成执行计划
	--> pg_plan_query
		--> standard_planner
			--> subquery_planner
			--> create_plan
--> PortalStart
	--> ExecutorStart
		--> InitPlan
			--> ExecInitNode
--> PortalRun
	--> ExecutorRun
		--> CitusExecutorRun
			--> standard_ExecutorRun
				--> ExecutePlan
					--> ExecResult
						--> ExecProject
							--> ExecInterpExpr
								--> create_distributed_table // 调用citus创建分布表函数
--> PortalDrop
```
现在到了最建分布表最关键的部分，看一下citus中`create_distributed_table`函数的实现：
```c++
// CN 节点执行`select create_distributed_table('t1','a');`
create_distributed_table
--> EnsureCitusTableCanBeCreated
--> LookupDistributionMethod
--> CreateDistributedTable	// this function creates distributed table metadata, creates shards and copies local data to shards.
	--> LockRelationOid
	--> EnsureTableNotDistributed
	--> PropagatePrerequisiteObjectsForDistributedTable
	--> DecideReplicationModel
	--> BuildDistributionKeyFromColumnName
	--> ColocationIdForNewTable
	--> EnsureRelationCanBeDistributed
	--> EnsureReferenceTablesExistOnAllNodes
	--> TableEmpty
	--> InsertIntoPgDistPartition
	--> CreateHashDistributedTableShards  // 创建shards分片
		--> CreateShardsWithRoundRobinPolicy // 具体创建shards的函数
			--> CheckHashPartitionedTable
			--> LoadShardList
			--> DistributedTablePlacementNodeList
			--> SortList
			--> GetNextShardId // 获取分片ID
			--> InsertShardRow  // 插入pg_dist_shard表中
			--> InsertShardPlacementRows
				--> InsertShardPlacementRow // 插入pg_dist_placement
				--> LoadShardPlacement
			--> CreateShardsOnWorkers //核心函数，creates shards on worker nodes
				--> GetFullTableCreationCommands
					--> GetPreLoadTableCreationCommands
						--> pg_get_tableschemadef_string	// 获取表定义字符串
						--> pg_get_tablecolumnoptionsdef_string
						--> makeTableDDLCommandString
				--> LoadShardInterval
				--> RelationShardListForShardCreate
				--> WorkerCreateShardCommandList
					--> GetShardedTableDDLCommand	//构造"SELECT worker_apply_shard_ddl_command (103012, 'CREATE TABLE public.t1 (a integer, b integer) USING heap')"
					--> GetShardedTableDDLCommandString
				--> SetTaskQueryStringList  // 构造Task
				--> ExecuteUtilityTaskListExtended
					--> DecideTransactionPropertiesForTaskList
						--> TaskListCannotBeExecutedInTransaction
						--> DistributedExecutionRequiresRollback
						--> TaskListRequires2P
					--> ExecuteTaskListExtended
						--> CreateDistributedExecution
							--> ShouldExecuteTasksLocally
						--> EnsureCompatibleLocalExecutionState
						--> StartDistributedExecution
							--> UseCoordinatedTransaction
							--> Use2PCForCoordinatedTransaction
							--> EnsureTaskExecutionAllowed
								--> IsTaskExecutionAllowed
									--> InLocalTaskExecutionOnShard
									--> MaybeInRemoteTaskExecution
						--> RunDistributedExecution
							--> AssignTasksToConnectionsOrWorkerPool
								--> PlacementAccessListForTask
								--> Activate2PCIfModifyingTransactionExpandsToNewNode
							--> ManageWorkerPool
						--> FinishDistributedExecution
```
在CN节点上会构造Task，发往Worker节点上执行，其执行语句为在worker节点上创建分片（建表）。我们分析一下`worker_apply_shard_ddl_command`的执行过程。
```c++
// worker节点执行"SELECT worker_apply_shard_ddl_command (103012, 'CREATE TABLE public.t1 (a integer, b integer) USING heap')"
worker_apply_shard_ddl_command
--> ParseTreeNode // 由sql string 转为RawStmt，抽象语法树
	--> ParseTreeRawStmt
		--> pg_parse_query
			--> raw_parser
				--> base_yyparse
--> RelayEventExtendNames  // 扩展表名为tablename_shardid
	--> SetSchemaNameIfNotExist // 加模式名
	--> AppendShardIdToName     // 表名后面加shardid
--> ProcessUtilityParseTree		// 执行建表
	--> ProcessUtility

```

