### Greenplum源码分析 —— SELECT
这里我们简单分析一下在Greenplum中select查询的源码，PG的源码我们都比较熟了，这里重点看一下greenplum中有什么不同之处呢？或者说，在greenplum中，查询的大致流程是如何呢？对此我们这里通过一条简单的SQL语句来进行分析。

在进行分析之前，我们先回顾一下在Greenplum中SQL的执行过程，有个宏观的理解：
0. The system at rest
1. Client connects via the entry postmaster
2. Entry postmaster forks a new backend -- the QD 
3. QD connects to segment via the segment postmasters
4. Segment postmaster for initial gang of QEs
5. Client submits a query to the QD
6. QD plans query and submits plans to QEs
7. QD and QEs setup interconnect routes according to plan
8. QD and QEs execute their slices sending tuples up the slice tree
9. QEs return status to QD
10. QD returns result set and status to the client

#### 执行计划
我们先看一条简单的查询语句在Greenplum中的执行计划，可以看到，与PG最不一样的就是多了一个在master进行Gather Motion的操作。
```sql
postgres=# explain select * from t1;
                                   QUERY PLAN                                   
---------------------------------------------------------------------
 Gather Motion 2:1  (slice1; segments: 2)  (cost=0.00..21.02 rows=1001 width=8)
   ->  Seq Scan on t1  (cost=0.00..6.00 rows=500 width=8)
 Optimizer: Postgres query optimizer
(3 rows)

postgres=# explain select * from t1 limit 3;
                                    QUERY PLAN                                    
----------------------------------------------------------------------------------
 Limit  (cost=0.60..0.66 rows=3 width=8)
   ->  Gather Motion 2:1  (slice1; segments: 2)  (cost=0.60..0.73 rows=6 width=8)
         ->  Limit  (cost=0.60..0.64 rows=3 width=8)
               ->  Seq Scan on t1  (cost=0.00..6.00 rows=500 width=8)
 Optimizer: Postgres query optimizer
(5 rows)
```
我们以下面这条SQL语句为例进行分析。

#### 源码分析

greenplum是分布式数据库，相比PG，其要多节点协同工作，需要生成分布式执行计划，为了分布式执行计划的生成以及分发执行，gp相对pg增加了很多内容，比如Interconnect，比如重分布、广播等，这里面的东西还是非常多的，后续再进行分析。关于这部分的源码可以查看gpdb/src/backend/cdb中的内容，其中有个分发执行计划相关解释的[cdb/dispatcher/README.md](https://github.com/greenplum-db/gpdb/blob/main/src/backend/cdb/dispatcher/README.md)说明可以阅读一下。

关键数据结构：
```c++
/*
 * CdbMotionPath represents transmission of the child Path results
 * from a set of sending processes to a set of receiving processes.
 *
 * Normally, the distribution is determined by the 'locus' of the path.
 * However, if the distribution cannot be represented by a DistributionKeys,
 * an alternative representation is to mark the locus as Strewn, and list
 * the hash columns in 'policy'. In the normal case, 'policy' is not used.
 */
typedef struct CdbMotionPath
{
	Path		path;
    Path	   *subpath;
	bool		is_explicit_motion;

	GpPolicy   *policy;
} CdbMotionPath;
```

##### master节点源码分析
master节点主要是生成执行计划并分发给segments节点去执行，这里需要注意生成执行计划有两种优化器，一种是PG本身的优化器（GP对此做了分布式的一些修改），另一种是ORCA优化器，这里先不进行ORCA的分析，先看一下PG优化器下是如何工作的。

master节点主流程如下：

```c++
exec_simple_query
--> pg_parse_query
--> pg_plan_queries
	--> pg_plan_query
		--> planner
			--> standard_planner  // 选择pg的优化器进行分析，后续可分析orca
				--> subquery_planner
					--> grouping_planner
						--> query_planner
						--> create_preliminary_limit_path  // doing multi-phase limit
							--> create_limit_path
						--> CdbPathLocus_MakeSingleQE
							--> getgpsegmentCount   // 获取segment的个数
						--> cdbpath_create_motion_path
							--> cdbpath_cost_motion
						--> create_limit_path
						--> cdbpath_create_motion_path
				--> cdbllize_adjust_top_path
					--> cdbpath_create_motion_path
				--> create_plan
					--> create_motion_plan
						--> create_limit_plan
							--> create_motion_plan
								--> create_limit_plan
									--> create_scan_plan
										--> create_seqscan_plan
				--> cdbpathtoplan_create_flow
				--> cdbllize_decorate_subplans_with_motions
--> PortalStart
	--> ExecutorStart
		--> standard_ExecutorStart
			--> InitPlan
				--> ExecInitLimit
					--> ExecInitMotion
						--> ExecInitLimit
							--> ExecInitSeqScan
			--> CdbDispatchPlan    // 分发执行计划
				--> serializeParamsForDispatch
				--> cdbdisp_dispatchX
--> PortalRun
	--> ExecutorRun
		--> standard_ExecutorRun
			--> getCurrentSlice
			   /*
				* Run the plan locally.  There are three ways;
				*
				* 1. Do nothing
				* 2. Run a root slice
				* 3. Run a non-root slice on a QE.
				*
				* Here we decide what is our identity -- root slice, non-root
				* on QE or other (in which case we do nothing), and then run
				* the plan if required. For more information see
				* getGpExecIdentity() in execUtils.
				*/
			--> getGpExecIdentity
			--> ExecutePlan
				--> ExecLimit
					--> ExecLimit_guts
						--> ExecMotion
							--> execMotionUnsortedReceiver
								--> RecvTupleFrom	// Receive one tuple from a sender.
									--> getMotionNodeEntry
									--> htfifo_gettuple
--> PortalDrop
```

生成的执行计划： Limit-->Motion-->Limit-->SeqScan

##### segment节点源码分析

相比postgresql数据库，greenplum是由多个数据库实例协作完成具体SQL的执行的。具体的是master节点生成分布式执行计划，并将执行计划分发给segments节点，segments节点接收的是执行计划，不再是sql语句字符串，也就是说，在segments节点上，不用再进行词法分析，语法分析，执行计划生成了，直接接收master节点分发的执行计划执行。入口是`exec_mpp_query`函数。

我们可以看一下这个函数的参数，`const char * serializedPlantree, int serializedPlantreelen,`，这里接收的是执行计划序列化后的字符串，经过反序列化可还原为执行计划。
```c++
 	/* Deserialize the query execution plan (a PlannedStmt node), if there is one. */
	if (serializedPlantree != NULL && serializedPlantreelen > 0)
	{
		plan = (PlannedStmt *) deserializeNode(serializedPlantree,serializedPlantreelen);
		if (!plan || !IsA(plan, PlannedStmt))
			elog(ERROR, "MPPEXEC: receive invalid planned statement");
    }
```
有了执行计划后就可以在执行器执行了：
```c++
/*
 * exec_mpp_query
 *
 * Called in a qExec process to read and execute a query plan sent by CdbDispatchPlan().
 *
 * query_string -- optional query text (C string).
 * serializedPlantree[len] -- PlannedStmt node, or (NULL,0) if query provided.
 * serializedQueryDispatchDesc[len] -- QueryDispatchDesc node, or (NULL,0) if query provided.
 *
 * Caller may supply either a Query (representing utility command) or
 * a PlannedStmt (representing a planned DML command), but not both.
 */
static void
exec_mpp_query(const char *query_string,
			   const char * serializedPlantree, int serializedPlantreelen,
			   const char * serializedQueryDispatchDesc, int serializedQueryDispatchDesclen)
{
	CommandDest dest = whereToSendOutput;
	MemoryContext oldcontext;
	bool		save_log_statement_stats = log_statement_stats;
	bool		was_logged = false;
	char		msec_str[32];
	PlannedStmt	   *plan = NULL;
	QueryDispatchDesc *ddesc = NULL;
	CmdType		commandType = CMD_UNKNOWN;
	SliceTable *sliceTable = NULL;
	ExecSlice  *slice = NULL;
	ParamListInfo paramLI = NULL;

	SIMPLE_FAULT_INJECTOR("exec_mpp_query_start");

	Assert(Gp_role == GP_ROLE_EXECUTE);
	/* If we didn't get passed a query string, dummy something up for ps display and pg_stat_activity */
	if (query_string == NULL || strlen(query_string)==0)
		query_string = "mppexec";

	/* Report query to various monitoring facilities. */
	debug_query_string = query_string;

	pgstat_report_activity(STATE_RUNNING, query_string);

	/*
	 * We use save_log_statement_stats so ShowUsage doesn't report incorrect
	 * results because ResetUsage wasn't called.
	 */
	if (save_log_statement_stats)
		ResetUsage();

	/*
	 * Start up a transaction command.	All queries generated by the
	 * query_string will be in this same command block, *unless* we find a
	 * BEGIN/COMMIT/ABORT statement; we have to force a new xact command after
	 * one of those, else bad things will happen in xact.c. (Note that this
	 * will normally change current memory context.)
	 */
	start_xact_command();

	/*
	 * Zap any pre-existing unnamed statement.	(While not strictly necessary,
	 * it seems best to define simple-Query mode as if it used the unnamed
	 * statement and portal; this ensures we recover any storage used by prior
	 * unnamed operations.)
	 */
	drop_unnamed_stmt();

	/* Switch to appropriate context for constructing parsetrees. */
	oldcontext = MemoryContextSwitchTo(MessageContext);

 	/* Deserialize the query execution plan (a PlannedStmt node), if there is one. */
	if (serializedPlantree != NULL && serializedPlantreelen > 0)
	{
		plan = (PlannedStmt *) deserializeNode(serializedPlantree,serializedPlantreelen);
		if (!plan || !IsA(plan, PlannedStmt))
			elog(ERROR, "MPPEXEC: receive invalid planned statement");
    }

	/*
     * Deserialize the extra execution information (a QueryDispatchDesc node), if there is one.
     */
    if (serializedQueryDispatchDesc != NULL && serializedQueryDispatchDesclen > 0)
    {
		ddesc = (QueryDispatchDesc *) deserializeNode(serializedQueryDispatchDesc,serializedQueryDispatchDesclen);
		if (!ddesc || !IsA(ddesc, QueryDispatchDesc))
			elog(ERROR, "MPPEXEC: received invalid QueryDispatchDesc with planned statement");
		/*
		 * Deserialize and apply security context from QD.
		 */
		SetUserIdAndSecContext(GetUserId(), ddesc->secContext);

        sliceTable = ddesc->sliceTable;

		if (sliceTable)
		{
			int			i;

			if (!IsA(sliceTable, SliceTable) ||
				sliceTable->localSlice < 0 ||
				sliceTable->localSlice >= sliceTable->numSlices)
				elog(ERROR, "MPPEXEC: received invalid slice table: %d", sliceTable->localSlice);

			/* Identify slice to execute */
			for (i = 0; i < sliceTable->numSlices; i++)
			{
				slice = &sliceTable->slices[i];

				if (bms_is_member(qe_identifier, slice->processesMap))
					break;
			}
			if (i == sliceTable->numSlices)
				elog(ERROR, "could not find QE identifier in process map");
			sliceTable->localSlice = slice->sliceIndex;

			/* Set global sliceid variable for elog. */
			currentSliceId = sliceTable->localSlice;
		}

		if (ddesc->oidAssignments)
			AddPreassignedOids(ddesc->oidAssignments);
    }

	if ( !plan )
		elog(ERROR, "MPPEXEC: received neither Query nor Plan");

	/* Extract command type from the planned statement. */
	if (plan->commandType != CMD_SELECT &&
		plan->commandType != CMD_INSERT &&
		plan->commandType != CMD_UPDATE &&
		plan->commandType != CMD_DELETE &&
		plan->commandType != CMD_UTILITY)
		elog(ERROR, "MPPEXEC: received non-DML Plan");
	commandType = plan->commandType;

	if ( slice )
	{
		/* Non root slices don't need update privileges. */
		if (sliceTable->localSlice != slice->rootIndex)
		{
			ListCell       *rtcell;
			RangeTblEntry  *rte;
			AclMode         removeperms = ACL_INSERT | ACL_UPDATE | ACL_DELETE | ACL_SELECT_FOR_UPDATE;

			/* Just reading, so don't check INS/DEL/UPD permissions. */
			foreach(rtcell, plan->rtable)
			{
				rte = (RangeTblEntry *)lfirst(rtcell);
				if (rte->rtekind == RTE_RELATION &&
					0 != (rte->requiredPerms & removeperms))
					rte->requiredPerms &= ~removeperms;
			}
		}
	}


	if (log_statement != LOGSTMT_NONE)
	{
		/*
		 * TODO need to log SELECT INTO as DDL
		 */
		if (log_statement == LOGSTMT_ALL ||
			(plan->utilityStmt && log_statement == LOGSTMT_DDL) ||
			(plan && log_statement >= LOGSTMT_MOD))

		{
			ereport(LOG, (errmsg("statement: %s", query_string)
						   ));
			was_logged = true;
		}

	}

	/*
	 * Get (possibly 0) PARAM_EXTERN parameters. (PARAM_EXEC parameter
	 * will be handled later, in InitPlan()).
	 */
	if (ddesc && ddesc->paramInfo)
		paramLI = deserializeExternParams(ddesc->paramInfo);
	else
		paramLI = NULL;

	/* Switch back to transaction context to enter the loop. */
	MemoryContextSwitchTo(oldcontext);

	/*
	 * All unpacked and checked.  Process the command.
	 */
	{
		const char *commandTag;
		char		completionTag[COMPLETION_TAG_BUFSIZE];

		Portal		portal;
		DestReceiver *receiver;
		int16		format;

		/*
		 * Get the command name for use in status display (it also becomes the
		 * default completion tag, down inside PortalRun).	Set ps_status and
		 * do any special start-of-SQL-command processing needed by the
		 * destination.
		 */
		if (commandType == CMD_UTILITY)
			commandTag = "MPPEXEC UTILITY";
		else if (commandType == CMD_SELECT)
			commandTag = "MPPEXEC SELECT";
		else if (commandType == CMD_INSERT)
			commandTag = "MPPEXEC INSERT";
		else if (commandType == CMD_UPDATE)
			commandTag = "MPPEXEC UPDATE";
		else if (commandType == CMD_DELETE)
			commandTag = "MPPEXEC DELETE";
		else
			commandTag = "MPPEXEC";


		set_ps_display(commandTag, false);

		BeginCommand(commandTag, dest);

        /* Downgrade segworker process priority */
		if (gp_segworker_relative_priority != 0)
		{
			renice_current_process(PostmasterPriority + gp_segworker_relative_priority);
		}

		if (Debug_dtm_action == DEBUG_DTM_ACTION_FAIL_BEGIN_COMMAND &&
			CheckDebugDtmActionSqlCommandTag(commandTag))
		{
			ereport(ERROR,
					(errcode(ERRCODE_FAULT_INJECT),
					 errmsg("Raise ERROR for debug_dtm_action = %d, commandTag = %s",
							Debug_dtm_action, commandTag)));
		}

		/*
		 * If we are in an aborted transaction, reject all commands except
		 * COMMIT/ABORT.  It is important that this test occur before we try
		 * to do parse analysis, rewrite, or planning, since all those phases
		 * try to do database accesses, which may fail in abort state. (It
		 * might be safe to allow some additional utility commands in this
		 * state, but not many...)
		 */
		if (IsAbortedTransactionBlockState() /*&&*/
			/*!IsTransactionExitStmt(parsetree)*/)
			ereport(ERROR,
					(errcode(ERRCODE_IN_FAILED_SQL_TRANSACTION),
					 errmsg("current transaction is aborted, "
							"commands ignored until end of transaction block")));

		/* Make sure we are in a transaction command */
		start_xact_command();

		/*
		 * OK to analyze, rewrite, and plan this query.
		 *
		 * Switch to appropriate context for constructing querytrees (again,
		 * these must outlive the execution context).
		 */
		oldcontext = MemoryContextSwitchTo(MessageContext);

		CHECK_FOR_INTERRUPTS();

		/*
		 * Create unnamed portal to run the query or queries in. If there
		 * already is one, silently drop it.
		 */
		portal = CreatePortal("", true, true);
		/* Don't display the portal in pg_cursors */
		portal->visible = false;

		/*
		 * We don't have to copy anything into the portal, because everything
		 * we are passing here is in MessageContext, which will outlive the
		 * portal anyway.
		 */
		PortalDefineQuery(portal,
						  NULL,
						  query_string,
						  /*
						   * sourceTag is stored in parsetree, but the original parsetree isn't
						   * dispatched to QE, so set a generic T_Query here.
						   */
						  T_Query,
						  commandTag,
						  list_make1(plan),
						  NULL);

		/*
		 * Start the portal.
		 */
		PortalStart(portal, paramLI, 0, InvalidSnapshot, ddesc);

		/*
		 * Select text output format, the default.
		 */
		format = 0;
		PortalSetResultFormat(portal, 1, &format);

		/*
		 * Now we can create the destination receiver object.
		 */
		receiver = CreateDestReceiver(dest);
		if (dest == DestRemote)
			SetRemoteDestReceiverParams(receiver, portal);

		/*
		 * Switch back to transaction context for execution.
		 */
		MemoryContextSwitchTo(oldcontext);

		/*
		 * Run the portal to completion, and then drop it (and the receiver).
		 */
		(void) PortalRun(portal,
						 FETCH_ALL,
						 true, /* Effectively always top level. */
						 portal->run_once,
						 receiver,
						 receiver,
						 completionTag);

		/*
		 * If writer QE, sent current pgstat for tables to QD.
		 */
		if (Gp_role == GP_ROLE_EXECUTE && Gp_is_writer)
			pgstat_send_qd_tabstats();

		(*receiver->rDestroy) (receiver);

		PortalDrop(portal, false);

		/*
		 * Close down transaction statement before reporting command-complete.
		 * This is so that any end-of-transaction errors are reported before
		 * the command-complete message is issued, to avoid confusing
		 * clients who will expect either a command-complete message or an
		 * error, not one and then the other.
		 */
		finish_xact_command();

		if (Debug_dtm_action == DEBUG_DTM_ACTION_FAIL_END_COMMAND &&
			CheckDebugDtmActionSqlCommandTag(commandTag))
		{
			ereport(ERROR,
					(errcode(ERRCODE_FAULT_INJECT),
					 errmsg("Raise ERROR for debug_dtm_action = %d, commandTag = %s",
							Debug_dtm_action, commandTag)));
		}

		/*
		 * Tell client that we're done with this query.  Note we emit exactly
		 * one EndCommand report for each raw parsetree, thus one for each SQL
		 * command the client sent, regardless of rewriting. (But a command
		 * aborted by error will not send an EndCommand report at all.)
		 */
		EndCommand(completionTag, dest);
	}							/* end loop over parsetrees */

	/*
	 * Close down transaction statement, if one is open.
	 */
	finish_xact_command();

	/*
	 * Emit duration logging if appropriate.
	 */
	switch (check_log_duration(msec_str, was_logged))
	{
		case 1:
			ereport(LOG,
					(errmsg("duration: %s ms", msec_str),
					 errhidestmt(false)));
			break;
		case 2:
			ereport(LOG,
					(errmsg("duration: %s ms  statement: %s",
							msec_str, query_string),
					 errhidestmt(true)));
			break;
	}

	if (save_log_statement_stats)
		ShowUsage("QUERY STATISTICS");


	if (gp_enable_resqueue_priority)
	{
		BackoffBackendEntryExit();
	}

	debug_query_string = NULL;
}
```