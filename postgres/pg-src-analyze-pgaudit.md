
## PostgreSQL审计插件pgaudit
在PostgreSQL中，提供了开源的审计插件[pgaudit](https://github.com/pgaudit/pgaudit)，但是其功能并不完善，只提供了基本的审计功能，对此，很多基于PG开发的商业数据库大多提供了丰富的审计功能。比如人大金仓，openGauss等。这里，我们不去讨论审计功能是否丰富，我们分析一下pgaudit的实现，已便我们后续开发自己的审计插件。


### pgaudit的使用
#### 安装

配置postgresql.conf，添加以下内容：
```shell
shared_preload_libraries = 'pgaudit'
```
创建插件：
```sql
CREATE EXTENSION pgaudit;
```
查看可配置参数：
```sql
postgres=# select name,setting from pg_settings where name ~ 'pgaudit';
            name            | setting 
----------------------------+---------
 pgaudit.log                | none		-- 指定会话审计日志记录哪些类型的语句
 pgaudit.log_catalog        | on
 pgaudit.log_client         | off
 pgaudit.log_level          | log
 pgaudit.log_parameter      | off
 pgaudit.log_relation       | off
 pgaudit.log_rows           | off
 pgaudit.log_statement      | on
 pgaudit.log_statement_once | off
 pgaudit.role               | 
(10 rows)
```

具体个参数的设置，可参考文档： https://github.com/pgaudit/pgaudit/blob/master/README.md


配置审计角色
```sql
postgres=# create role pgaudit with password 'pgaudit' login;
CREATE ROLE
postgres=# alter system set pgaudit.role = 'pgaudit';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# show pgaudit.role;
 pgaudit.role 
--------------
 pgaudit
(1 row)

```

实例：
```sql
设置记录write,以及ddl语句
postgres=# alter system set pgaudit.log = 'write,ddl';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```

```sql
postgres=# insert into t1 values(3,3);
INSERT 0 1
```
查看日志，后面那条是审计日志：
```log
2024-06-01 14:44:52.006 CST [27675] LOG:  statement: insert into t1 values(3,3);
2024-06-01 14:44:52.006 CST [27675] LOG:  AUDIT: SESSION,3,1,WRITE,INSERT,,,"insert into t1 values(3,3);",<not logged>
```

### 源码分析
总体上审计插件的设计思路有点类似于日志库的设计。首先是要获取审计日志信息，需要先通过配置设置审计哪些内容，比如审计DDL语句，还是DML语句，或者都进行审计。当然，这里审计规则可以设置的非常灵活，商业数据库的审计功能会更加的完善。然后还需要设置审计日志输出哪些内容，比如，语句类型，执行结果是否成功，执行影响的行数等等。配置好这些后，就是具体的获取审计日志的信息了。具体实现上就是在语句执行的过程中通过钩子函数捕获相应的信息，实现上要在不同的位置设置不同的钩子函数，比如DDL语句，则可在`ProcessUtility_hook`中执行自己的钩子函数，因为每个DDL语句都会走这里，具体的可以见下面的代码分析。
```c++
void ProcessUtility(PlannedStmt *pstmt,
			   const char *queryString,
			   bool readOnlyTree,
			   ProcessUtilityContext context,
			   ParamListInfo params,
			   QueryEnvironment *queryEnv,
			   DestReceiver *dest,
			   QueryCompletion *qc)
{
    // 审计插件钩子函数 pgaudit_ProcessUtility_hook
	if (ProcessUtility_hook)
		(*ProcessUtility_hook) (pstmt, queryString, readOnlyTree,
								context, params, queryEnv,
								dest, qc);
	else
		standard_ProcessUtility(pstmt, queryString, readOnlyTree,
								context, params, queryEnv,
								dest, qc);
}
```


#### 加载插件
总体源码并不多：当加载插件时，会执行`_PG_init`函数，该函数会定义GUC变量，以及安装钩子。还会执行`pgaudit--1.7.sql`。
GUC参数：
- pgaudit.log   
- pgaudit.log_catalog
- pgaudit.log_client
- pgaudit.log_level
- pgaudit.log_parameter
- pgaudit.log_relation
- pgaudit.log_rows
- pgaudit.log_statement
- pgaudit.log_statement_once
- pgaudit.role

钩子函数：
- pgaudit_ExecutorStart_hook;
- pgaudit_ExecutorCheckPerms_hook;
- pgaudit_ProcessUtility_hook;
- pgaudit_object_access_hook;
- pgaudit_ExecutorRun_hook;
- pgaudit_ExecutorEnd_hook;

如果要实现更多的功能，可以在更多的位置加钩子函数

```c++
/* Define GUC variables and install hooks upon module load. */
void _PG_init(void)
{
    /* Be sure we do initialization only once */
    static bool inited = false;  // 确保只被加载一次

    if (inited)
        return;

    /* Must be loaded with shared_preload_libraries */
    if (!process_shared_preload_libraries_in_progress)
        ereport(ERROR, (errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
                errmsg("pgaudit must be loaded via shared_preload_libraries")));

	// 下面是定义插件中的几个GUC参数
    /* Define pgaudit.log */
    DefineCustomStringVariable(
        "pgaudit.log",

        "Specifies which classes of statements will be logged by session audit "
        "logging. Multiple classes can be provided using a comma-separated "
        "list and classes can be subtracted by prefacing the class with a "
        "- sign.",

        NULL,
        &auditLog,
        "none",
        PGC_SUSET,
        GUC_LIST_INPUT | GUC_NOT_IN_SAMPLE,
        check_pgaudit_log,
        assign_pgaudit_log,
        NULL);

    // 其他参数

	// 安装钩子
    /* Install our hook functions after saving the existing pointers to preserve the chains. */
    next_ExecutorStart_hook = ExecutorStart_hook;
    ExecutorStart_hook = pgaudit_ExecutorStart_hook;

    // ...

    /* Log that the extension has completed initialization */
#ifndef EXEC_BACKEND
    ereport(LOG, (errmsg("pgaudit extension initialized")));
#else
    ereport(DEBUG1, (errmsg("pgaudit extension initialized")));
#endif /* EXEC_BACKEND */

    inited = true;
}
```

在加载插件时，执行`pgaudit--1.7.sql`，定义了2个触发器函数，2个事件触发器。函数的具体实现在`pgaudit.c`中
```sql
-- 当DDL语句执行结束时，执行该函数
CREATE FUNCTION pgaudit_ddl_command_end()
	RETURNS event_trigger
	SECURITY DEFINER
	SET search_path = 'pg_catalog, pg_temp'
	LANGUAGE C
	AS 'MODULE_PATHNAME', 'pgaudit_ddl_command_end';

-- ddl_command_end事件触发器
CREATE EVENT TRIGGER pgaudit_ddl_command_end
	ON ddl_command_end
	EXECUTE PROCEDURE pgaudit_ddl_command_end(); -- 当DDL语句执行结束时触发执行该函数

CREATE FUNCTION pgaudit_sql_drop()
	RETURNS event_trigger
	SECURITY DEFINER
	SET search_path = 'pg_catalog, pg_temp'
	LANGUAGE C
	AS 'MODULE_PATHNAME', 'pgaudit_sql_drop';

CREATE EVENT TRIGGER pgaudit_sql_drop
	ON sql_drop
	EXECUTE PROCEDURE pgaudit_sql_drop();   -- sql_drop事件触发器，删除对象操作触发

```

#### 调试DDL语句的审计
我们调试一条建表语句的审计过程

调用栈：
```c++
pgaudit.so!log_audit_event(AuditEventStackItem * stackItem) (contrib\pgaudit\pgaudit.c:704)
pgaudit.so!pgaudit_ddl_command_end(FunctionCallInfo fcinfo) (contrib\pgaudit\pgaudit.c:1734)
fmgr_security_definer(FunctionCallInfo fcinfo) (src\backend\utils\fmgr\fmgr.c:732)
EventTriggerInvoke(List * fn_oid_list, EventTriggerData * trigdata) (src\backend\commands\event_trigger.c:920)
EventTriggerDDLCommandEnd(Node * parsetree) (src\backend\commands\event_trigger.c:727)
ProcessUtilitySlow(ParseState * pstate, PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\utility.c:1923)
standard_ProcessUtility(PlannedStmt * pstmt, const char * queryString, _Bool readOnlyTree, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\utility.c:1074)
pgaudit.so!pgaudit_ProcessUtility_hook(PlannedStmt * pstmt, const char * queryString, _Bool readOnlyTree, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (contrib\pgaudit\pgaudit.c:1584)
ProcessUtility(PlannedStmt * pstmt, const char * queryString, _Bool readOnlyTree, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\utility.c:526)
PortalRunUtility(Portal portal, PlannedStmt * pstmt, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\pquery.c:1158)
PortalRunMulti(Portal portal, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:1315)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:791)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1250)
PostgresMain(const char * dbname, const char * username) (src\backend\tcop\postgres.c:4598)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4514)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4242)
ServerLoop() (src\backend\postmaster\postmaster.c:1809)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1481)
main(int argc, char ** argv) (src\backend\main\main.c:202)
```
中间最重要的3个函数`pgaudit_ProcessUtility_hook`、捕获DDL语句，添加`AuditEvent`项，后续不断地补充`AuditEvent`中个各项信息。
```c++
static void pgaudit_ProcessUtility_hook(PlannedStmt *pstmt,
                            const char *queryString,
                            bool readOnlyTree,
                            ProcessUtilityContext context,
                            ParamListInfo params,
                            QueryEnvironment *queryEnv,
                            DestReceiver *dest,
                            QueryCompletion *qc)
{
    AuditEventStackItem *stackItem = NULL;
    int64 stackId = 0;

    /* Don't audit substatements.  All the substatements we care about should be covered by the event triggers. */
    if (context <= PROCESS_UTILITY_QUERY && !IsAbortedTransactionBlockState())
    {
        /* Process top level utility statement */
        if (context == PROCESS_UTILITY_TOPLEVEL)
        {
            /* If the stack is not empty then the only allowed entries are open
             * select, show, and explain cursors */
            if (auditEventStack != NULL)
            {
                AuditEventStackItem *nextItem = auditEventStack;

                do
                {
                    if (nextItem->auditEvent.commandTag != T_SelectStmt &&
                        nextItem->auditEvent.commandTag != T_VariableShowStmt &&
                        nextItem->auditEvent.commandTag != T_ExplainStmt)
                    {
                        elog(ERROR, "pgaudit stack is not empty");
                    }

                    nextItem = nextItem->next;
                }
                while (nextItem != NULL);
            }

            stackItem = stack_push();
            stackItem->auditEvent.paramList = copyParamList(params);
        }
        else
            stackItem = stack_push();

        stackId = stackItem->stackId;
        stackItem->auditEvent.logStmtLevel = GetCommandLogLevel(pstmt->utilityStmt);
        stackItem->auditEvent.commandTag = nodeTag(pstmt->utilityStmt);
        stackItem->auditEvent.command = CreateCommandTag(pstmt->utilityStmt);
        stackItem->auditEvent.commandText = queryString;

        /* If this is a DO block log it before calling the next ProcessUtility hook. */
        if (auditLogBitmap & LOG_FUNCTION && stackItem->auditEvent.commandTag == T_DoStmt &&
            !IsAbortedTransactionBlockState())
            log_audit_event(stackItem);

        /*
         * If this is a create/alter extension command log it before calling
         * the next ProcessUtility hook. Otherwise, any warnings will be emitted
         * before the create/alter is logged and errors will prevent it from
         * being logged at all. */
        if (auditLogBitmap & LOG_DDL &&
            (stackItem->auditEvent.commandTag == T_CreateExtensionStmt ||
                stackItem->auditEvent.commandTag == T_AlterExtensionStmt) &&
            !IsAbortedTransactionBlockState())
            log_audit_event(stackItem);

        /*
         * A close will free the open cursor which will also free the close
         * audit entry. Immediately log the close and set stackItem to NULL so
         * it won't be logged later.
         */
        if (stackItem->auditEvent.commandTag == T_ClosePortalStmt)
        {
            if (auditLogBitmap & LOG_MISC && !IsAbortedTransactionBlockState())
                log_audit_event(stackItem);

            stackItem = NULL;
        }
    }

    /* Call the standard process utility chain. */
    if (next_ProcessUtility_hook)
        (*next_ProcessUtility_hook) (pstmt, queryString, readOnlyTree, context,
                                     params, queryEnv, dest, qc);
    else
        standard_ProcessUtility(pstmt, queryString, readOnlyTree, context,
                                params, queryEnv, dest, qc);

    /* Process the audit event if there is one.  Also check that this event
     * was not popped off the stack by a memory context being free'd elsewhere. */
    if (stackItem && !IsAbortedTransactionBlockState())
    {
        /* Make sure the item we want to log is still on the stack - if not
         * then something has gone wrong and an error will be raised. */
        stack_valid(stackId);

        /* Log the utility command if logging is on, the command has not
         * already been logged by another hook, and the transaction is not aborted. */
        if (auditLogBitmap != 0 && !stackItem->auditEvent.logged)
            log_audit_event(stackItem);
    }
}
```
记录了审计信息后，需要将其进行输出，在商业数据库中，可以存储在表中或者文件中，通过表等进行查看，在pgaudit插件中，近输出到日志文件中。
```c++
static void log_audit_event(AuditEventStackItem *stackItem)
{
    /* By default, put everything in the MISC class. */
    int class = LOG_MISC;
    const char *className = CLASS_MISC;
    MemoryContext contextOld;
    StringInfoData auditStr;

    /*
     * Skip logging script statements if an extension is currently being created
     * or altered. PostgreSQL reports the statement text for each statement in
     * the script as the entire script text, which can blow up the logs. The
     * create/alter statement will still be logged.
     *
     * Since a superuser is responsible for determining which extensions are
     * available, and in most cases installing them, it should not be necessary
     * to log each statement in the script.
     */
    if (creating_extension)
        return;

    /* If this event has already been logged don't log it again */
    if (stackItem->auditEvent.logged)
        return;

    // ...

    /*
     * Create the audit substring
     *
     * The type-of-audit-log and statement/substatement ID are handled below,
     * this string is everything else.
     */
    initStringInfo(&auditStr);
    append_valid_csv(&auditStr, GetCommandTagName(stackItem->auditEvent.command));

    appendStringInfoCharMacro(&auditStr, ',');
    append_valid_csv(&auditStr, stackItem->auditEvent.objectType);

    appendStringInfoCharMacro(&auditStr, ',');
    append_valid_csv(&auditStr, stackItem->auditEvent.objectName);

    // 其他信息

    /* Log rows affected */
    if (auditLogRows)
        appendStringInfo(&auditStr, "," INT64_FORMAT,
                         stackItem->auditEvent.rows);

    /*
     * Log the audit entry.  Note: use of INT64_FORMAT here is bad for
     * translatability, but we currently haven't got translation support in
     * pgaudit anyway. */
    ereport(auditLogClient ? auditLogLevel : LOG_SERVER_ONLY,
            (errmsg("AUDIT: %s," INT64_FORMAT "," INT64_FORMAT ",%s,%s",
                    stackItem->auditEvent.granted ?
                    AUDIT_TYPE_OBJECT : AUDIT_TYPE_SESSION,
                    stackItem->auditEvent.statementId,
                    stackItem->auditEvent.substatementId,
                    className,
                    auditStr.data),
                    errhidestmt(true),
                    errhidecontext(true)));

    stackItem->auditEvent.logged = true;

    MemoryContextSwitchTo(contextOld);
}
```

#### 调试DML语句的审计

查询语句的审计调用栈如下：
```c++
pgaudit.so!log_audit_event(AuditEventStackItem * stackItem) (contrib\pgaudit\pgaudit.c:654)
pgaudit.so!log_select_dml(Oid auditOid, List * rangeTabls) (contrib\pgaudit\pgaudit.c:1211)
pgaudit.so!pgaudit_ExecutorCheckPerms_hook(List * rangeTabls, _Bool abort) (contrib\pgaudit\pgaudit.c:1413)
ExecCheckRTPerms(List * rangeTable, _Bool ereport_on_violation) (src\backend\executor\execMain.c:591)
InitPlan(QueryDesc * queryDesc, int eflags) (src\backend\executor\execMain.c:820)
standard_ExecutorStart(QueryDesc * queryDesc, int eflags) (src\backend\executor\execMain.c:265)
pgaudit.so!pgaudit_ExecutorStart_hook(QueryDesc * queryDesc, int eflags) (contrib\pgaudit\pgaudit.c:1351)
ExecutorStart(QueryDesc * queryDesc, int eflags) (src\backend\executor\execMain.c:142)
PortalStart(Portal portal, ParamListInfo params, int eflags, Snapshot snapshot) (src\backend\tcop\pquery.c:517)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1211)
PostgresMain(const char * dbname, const char * username) (src\backend\tcop\postgres.c:4598)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4514)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4242)
ServerLoop() (src\backend\postmaster\postmaster.c:1809)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1481)
main(int argc, char ** argv) (src\backend\main\main.c:202)
```


#### 钩子函数

具体实现审计时，涉及到下面这些钩子函数：
```c++
static ExecutorCheckPerms_hook_type next_ExecutorCheckPerms_hook = NULL;
static ProcessUtility_hook_type next_ProcessUtility_hook = NULL;
static object_access_hook_type next_object_access_hook = NULL;
static ExecutorStart_hook_type next_ExecutorStart_hook = NULL;
/* The following hook functions are required to get rows */
static ExecutorRun_hook_type next_ExecutorRun_hook = NULL;
static ExecutorEnd_hook_type next_ExecutorEnd_hook = NULL;
```

```c++

/*
 * Hook ExecutorStart to get the query text and basic command type for queries
 * that do not contain a table and so can't be idenitified accurately in
 * ExecutorCheckPerms.
 */
static void
pgaudit_ExecutorStart_hook(QueryDesc *queryDesc, int eflags)
{
    AuditEventStackItem *stackItem = NULL;

    if (!internalStatement)
    {
        /* Push the audit even onto the stack */
        stackItem = stack_push();

        /* Initialize command using queryDesc->operation */
        switch (queryDesc->operation)
        {
            case CMD_SELECT:
                stackItem->auditEvent.logStmtLevel = LOGSTMT_ALL;
                stackItem->auditEvent.commandTag = T_SelectStmt;
                stackItem->auditEvent.command = CMDTAG_SELECT;
                break;

            case CMD_INSERT:
                stackItem->auditEvent.logStmtLevel = LOGSTMT_MOD;
                stackItem->auditEvent.commandTag = T_InsertStmt;
                stackItem->auditEvent.command = CMDTAG_INSERT;
                break;

            case CMD_UPDATE:
                stackItem->auditEvent.logStmtLevel = LOGSTMT_MOD;
                stackItem->auditEvent.commandTag = T_UpdateStmt;
                stackItem->auditEvent.command = CMDTAG_UPDATE;
                break;

            case CMD_DELETE:
                stackItem->auditEvent.logStmtLevel = LOGSTMT_MOD;
                stackItem->auditEvent.commandTag = T_DeleteStmt;
                stackItem->auditEvent.command = CMDTAG_DELETE;
                break;

            default:
                stackItem->auditEvent.logStmtLevel = LOGSTMT_ALL;
                stackItem->auditEvent.commandTag = T_Invalid;
                stackItem->auditEvent.command = CMDTAG_UNKNOWN;
                break;
        }

        /* Initialize the audit event */
        stackItem->auditEvent.commandText = queryDesc->sourceText;
        stackItem->auditEvent.paramList = copyParamList(queryDesc->params);
    }

    /* Call the previous hook or standard function */
    if (next_ExecutorStart_hook)
        next_ExecutorStart_hook(queryDesc, eflags);
    else
        standard_ExecutorStart(queryDesc, eflags);

    /*
     * Move the stack memory context to the query memory context.  This needs
     * to be done here because the query context does not exist before the
     * call to standard_ExecutorStart() but the stack item is required by
     * pgaudit_ExecutorCheckPerms_hook() which is called during
     * standard_ExecutorStart().
     */
    if (stackItem)
    {
        MemoryContextSetParent(stackItem->contextAudit,
                               queryDesc->estate->es_query_cxt);

        /* Set query context for tracking rows processed */
        if (auditLogRows)
            stackItem->auditEvent.queryContext = queryDesc->estate->es_query_cxt;
    }
}

/*
 * Hook ExecutorCheckPerms to do session and object auditing for DML.
 */
static bool pgaudit_ExecutorCheckPerms_hook(List *rangeTabls, bool abort)
{
    Oid auditOid;

    /* Get the audit oid if the role exists */
    auditOid = get_role_oid(auditRole, true);

    /* Log DML if the audit role is valid or session logging is enabled */
    if ((auditOid != InvalidOid || auditLogBitmap != 0) &&
        !IsAbortedTransactionBlockState())
    {
        /* If auditLogRows is on, wait for rows processed to be set */
        if (auditLogRows && auditEventStack != NULL)
        {
            /* Check if the top item is SELECT/INSERT for CREATE TABLE AS */
            if (auditEventStack->auditEvent.commandTag == T_SelectStmt &&
                auditEventStack->next != NULL &&
                auditEventStack->next->auditEvent.command == CMDTAG_CREATE_TABLE_AS &&
                auditEventStack->auditEvent.rangeTabls != NULL)
            {
                /*
                 * First, log the INSERT event for CREATE TABLE AS here.
                 * The SELECT event for CREATE TABLE AS will be logged
                 * in pgaudit_ExecutorEnd_hook() later to get rows.
                 */
                log_select_dml(auditOid, rangeTabls);
            }
            else
            {
                /* Save auditOid and rangeTabls to call log_select_dml()
                 * in pgaudit_ExecutorEnd_hook() later. */
                auditEventStack->auditEvent.auditOid = auditOid;
                auditEventStack->auditEvent.rangeTabls = rangeTabls;
            }
        }
        else
            log_select_dml(auditOid, rangeTabls);
    }

    /* Call the next hook function */
    if (next_ExecutorCheckPerms_hook &&
        !(*next_ExecutorCheckPerms_hook) (rangeTabls, abort))
        return false;

    return true;
}

/* Hook ExecutorRun to get rows processed by the current statement.*/
static void pgaudit_ExecutorRun_hook(QueryDesc *queryDesc, ScanDirection direction, uint64 count, bool execute_once)
{
    AuditEventStackItem *stackItem = NULL;

    /* Call the previous hook or standard function */
    if (next_ExecutorRun_hook)
        next_ExecutorRun_hook(queryDesc, direction, count, execute_once);
    else
        standard_ExecutorRun(queryDesc, direction, count, execute_once);

    if (auditLogRows && !internalStatement)
    {
        /* Find an item from the stack by the query memory context */
        stackItem = stack_find_context(queryDesc->estate->es_query_cxt);

        /* Accumulate the number of rows processed */
        if (stackItem != NULL)
            stackItem->auditEvent.rows += queryDesc->estate->es_processed;
    }
}

/*
 * Hook ExecutorEnd to get rows processed by the current statement.
 */
static void pgaudit_ExecutorEnd_hook(QueryDesc *queryDesc)
{
    AuditEventStackItem *stackItem = NULL;
    AuditEventStackItem *auditEventStackFull = NULL;

    if (auditLogRows && !internalStatement)
    {
        /* Find an item from the stack by the query memory context */
        stackItem = stack_find_context(queryDesc->estate->es_query_cxt);

        if (stackItem != NULL && stackItem->auditEvent.rangeTabls != NULL)
        {
            /* Reset auditEventStack to use in log_select_dml() */
            auditEventStackFull = auditEventStack;
            auditEventStack = stackItem;

            /* Log SELECT/DML audit entry */
            log_select_dml(stackItem->auditEvent.auditOid,
                           stackItem->auditEvent.rangeTabls);

            /* Switch back to the previous auditEventStack */
            auditEventStack = auditEventStackFull;
        }
    }

    /* Call the previous hook or standard function */
    if (next_ExecutorEnd_hook)
        next_ExecutorEnd_hook(queryDesc);
    else
        standard_ExecutorEnd(queryDesc);
}

/*
 * Hook object_access_hook to provide fully-qualified object names for function
 * calls.
 */
static void pgaudit_object_access_hook(ObjectAccessType access,
                            Oid classId,
                            Oid objectId,
                            int subId,
                            void *arg)
{
    if (auditLogBitmap & LOG_FUNCTION && access == OAT_FUNCTION_EXECUTE &&
        auditEventStack && !IsAbortedTransactionBlockState())
        log_function_execute(objectId);

    if (next_object_access_hook)
        (*next_object_access_hook) (access, classId, objectId, subId, arg);
}

```