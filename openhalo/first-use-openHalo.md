杭州易景数通科技有限公司开源了其羲和数据库[openHalo](https://www.openhalo.org/)，github地址: https://github.com/HaloTech-Co-Ltd/openHalo.git， openHalo基于PostgreSQL开发，其最大的特点是兼容MySQL，支持MySQL，PostgreSQL两种协议。其支持以通过MySQL客户端连接openHalo数据库实例。另外，其同一数据库实例可同时支持MySQL客户端与PostgreSQL客户端连接，openHalo会自动识别客户端类型，选择合适的协议进行通信，选择合适的解析器、优化器、执行器进行处理。

![image](https://www.openhalo.org/images/howitworks.png)


### 安装与使用

1. 下载源码：
```shell
git clone https://github.com/HaloTech-Co-Ltd/openHalo.git
```

2. 编译安装
```shell
./configure --prefix=/home/postgres/openhalo --enable-debug --with-uuid=ossp --with-icu CFLAGS=-O0
make && make install
cd contrib
make && make install
```

3. 初始化数据库实例
```shell
initdb -D openhalo-data/
# 查看数据库实例目录，PG内核版本为14
postgres@slpc:~/openhalo/openhalo-data$ ls
base          pg_hba.conf    pg_notify     pg_stat      pg_twophase  postgresql.auto.conf
global        pg_ident.conf  pg_replslot   pg_stat_tmp  PG_VERSION   postgresql.conf
pg_commit_ts  pg_logical     pg_serial     pg_subtrans  pg_wal
pg_dynshmem   pg_multixact   pg_snapshots  pg_tblspc    pg_xact
postgres@slpc:~/openhalo/openhalo-data$ cat PG_VERSION 
14
```

4. 修改postgresql.conf配置文件
开源版本兼容MySQL。Oralce,DB2,SQLServer,Sybase等数据库的兼容性仅企业版支持。
```shell
# MySQL相关GUC参数
#mysql.listener_on = true                           # (enable MySQL listener; change requires restart)
#mysql.port = 3306                              # (second_port is for MySQL mode; change requires restart)
#mysql.halo_mysql_version = '5.7.32-log'        # (change requires restart)
#mysql.ci_collation = true                      # (change requires restart)
#mysql.explicit_defaults_for_timestamp = false  # (change requires restart)
#mysql.auto_rollback_tx_on_error = false        # (change requires restart)
#mysql.halo_mysql_ci_collation = 0              # (change requires restart)
#mysql.column_name_case_control = 0             # (change requires restart)
#mysql.max_allowed_packet = 536870912B          # (512MB, change requires restart)
#mysql.support_multiple_table_update = true     # (change requires restart)

database_compat_mode = 'mysql'         # database compat mode, values can be:
                                                                                # postgresql (default)
                                        # mysql (MySQL mode)
                                                                                # oracle (support in Enterprise Edition)
                                                                                # db2 (support in Enterprise Edition)
                                                                                # sqlserver (support in Enterprise Edition)
                                                                                # sybase (support in Enterprise Edition)
standard_parserengine_auxiliary = 'on' # on, off, yes, no, true or false
```

5. 配置环境变量
```shell
sudo mkdir /var/run/openhalo 
sudo chown postgres:postgres /var/run/openhalo
export PGDATA=/home/postgres/openhalo/openhalo-data
export PATH=/home/postgres/openhalo/bin:$PATH
export LD_LIBRARY_PATH=/home/postgres/openhalo/lib:$LD_LIBRARY_PATH
```

6. 启动数据库实例
```shell
pg_ctl start -D $PGDATA
```

7. 连接数据库实例，可以看到其PostgreSQL内核版本为14.10
```sql
postgres@slpc:~/openhalo$ psql 
psql (1.0.14.10)
Type "help" for help.
halo0root=# show database_compat_mode ;
 database_compat_mode 
----------------------
 mysql
(1 row)
```

8. 安装`aux_mysql`扩展，该扩展为MySQL兼容辅助扩展模块
```sql
-- 安装aux_mysql扩展
halo0root=# create extension aux_mysql cascade;
NOTICE:  installing required extension "uuid-ossp"
CREATE EXTENSION
halo0root=# \dx
                            List of installed extensions
   Name    | Version |   Schema   |                   Description                   
-----------+---------+------------+-------------------------------------------------
 aux_mysql | 1.4     | public     | MySQL Supplementary Extension
 plpgsql   | 1.0     | pg_catalog | PL/pgSQL procedural language
 uuid-ossp | 1.1     | public     | generate universally unique identifiers (UUIDs)
(3 rows)
```

9. 安装mysql客户端，通过MySQL客户端连接openHalo数据库实例
```shell
sudo apt-get install mysql-client
# 通过mysql客户端连接openHalo数据库实例
postgres@slpc:~/openhalo/openhalo-data$ mysql -P 3306 -h 127.0.0.1
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.7.32-log MySQL Server (GPL)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> select version();
+------------+
| version    |
+------------+
| 5.7.32-log |
+------------+
1 row in set (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mys_sys            |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

```

10. 创建数据库
```sql
mysql> create database halo0;
Query OK, 1 row affected (0.00 sec)
```

### 初步源码分析

openHalo最大的特色就是[兼容MySQL](http://47.99.242.185/III.%E5%A4%9A%E6%A8%A1%E5%BC%8F%E5%85%BC%E5%AE%B9/MySQL%E5%85%BC%E5%AE%B9/)，我们初步分析一下openHalo是如何兼容MySQL的。首先是`aux_mysql`扩展，该扩展中包含了兼容MySQL所需的函数、运算符等，这部分大概有三万多行SQL代码。
```c++
typedef enum
{
	POSTGRESQL_COMPAT_MODE,
	MYSQL_COMPAT_MODE,
} DatabaseCompatModeType;
```
另外，实现了MySQL语法解析器，兼容MySQL语法，以及其他SQL命令相关的实现。
```c++
List *raw_parser(const char *str, RawParseMode mode)
{
	List *raw_parsetree = NIL;
	MemoryContext	oldctx = CurrentMemoryContext;

	if (parserengine == NULL)   // 如果没有指定解析引擎，则使用标准解析引擎
		parserengine = GetStandardParserEngine();

	Assert(parserengine != NULL);
	Assert(parserengine->raw_parser != NULL);

	PG_TRY();
	{
		raw_parsetree = parserengine->raw_parser(str, mode); // 调用解析引擎的解析函数
	}
	PG_CATCH();
	{
		if (raw_parsetree == NIL && parserengine->is_standard_parser == false
			&& (standard_parserengine_auxiliary == STANDARDARD_PARSERENGINE_AUXILIARY_ON 
				&& parserengine->need_standard_parser == true))
		{
			MemoryContextSwitchTo(oldctx);
			FlushErrorState();
			MemoryContextSwitchTo(oldctx);

			raw_parsetree = GetStandardParserEngine()->raw_parser(str, mode);
		}
		else
			PG_RE_THROW();
	}
	PG_END_TRY();
	return raw_parsetree;
}
```
为了支持多款解析器引擎，openHalo定义了`ParserRoutine`结构体，该结构体中包含了多种接口函数，用于处理不同解析器的解析、语法分析等。实现时需要把原先PostgreSQL解析相关的接口抽象出来。
```c++
/* API for parser engine */
typedef struct ParserRoutine
{
    NodeTag		type;
    bool        is_standard_parser;
    bool        need_standard_parser;

    const ScanKeywordList     *keywordlist;
    const uint16              *keyword_tokens;
    const uint8               *keyword_categories;
    
    raw_parser_function                           raw_parser;
    transformStmt_function                        transformStmt;
    transformSelectStmt_function                  transformSelectStmt;
    transformInsertStmt_function                  transformInsertStmt;
    transformDeleteStmt_function                  transformDeleteStmt;
    transformUpdateStmt_function                  transformUpdateStmt;
    transformSetOperationStmt_function            transformSetOperationStmt;

    transformCallStmt_function                    transformCallStmt;
    transformOptionalSelectInto_function          transformOptionalSelectInto;
    transformSetOperationTree_function            transformSetOperationTree;
    analyze_requires_snapshot_function            analyze_requires_snapshot;
    parse_sub_analyze_function                    parse_sub_analyze;


    transformGroupClause_function                 transformGroupClause;

    transformDistinctClause_function              transformDistinctClause;
    transformOnConflictArbiter_function           transformOnConflictArbiter;

    transformExpr_function                        transformExpr;

    transformCreateStmt_function                  transformCreateStmt;
    transformAlterTableStmt_function              transformAlterTableStmt;
    assign_query_collations_function              assign_query_collations;
    assign_list_collations_function               assign_list_collations;
    assign_expr_collations_function               assign_expr_collations;
    select_common_collation_function              select_common_collation;

    preTransformWindowAgg_fn                      preTransformWindowAgg;
    finalTransformColumnRef_fn                    finalTransformColumnRef;

    ParseFuncOrColumn_function                    ParseFuncOrColumn;
    func_get_detail_function                      func_get_detail;
    make_fn_arguments_function                    make_fn_arguments;
    RewriteInterface                              rewrite;
} ParserRoutine;
```
MySQL语法解析器调用栈如下：
```c++
mys_yyparse(core_yyscan_t yyscanner) (src\backend\parser\mysql\mys_gram.c:35413)
mys_raw_parser(const char * str, RawParseMode mode) (src\backend\parser\mysql\mys_parser.c:353)
raw_parser(const char * str, RawParseMode mode) (src\backend\parser\parser.c:66)
pg_parse_query(const char * query_string) (src\backend\tcop\postgres.c:641)
standard_exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1115)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres2.c:65)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (src\backend\tcop\postgres.c:4802)
mainFunc(Port * port, int argc, char ** argv) (src\backend\adapter\mysql\adapter.c:808)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4572)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4297)
ServerLoop() (src\backend\postmaster\postmaster.c:1773)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1442)
main(int argc, char ** argv) (src\backend\main\main.c:209)
```
我们实际调试一条INSERT语句，看一下其调用栈：
```c++
mys_transformInsertStmt(ParseState * pstate, InsertStmt * stmt) (src\backend\parser\mysql\mys_analyze.c:956)
transformInsertStmt(ParseState * pstate, InsertStmt * stmt) (src\backend\parser\analyze2.c:70)
mys_transformStmt(ParseState * pstate, Node * parseTree) (src\backend\parser\mysql\mys_analyze.c:191)
transformStmt(ParseState * pstate, Node * parseTree) (src\backend\parser\parser_ep.c:77)
mys_transformOptionalSelectInto(ParseState * pstate, Node * parseTree) (src\backend\parser\mysql\mys_analyze.c:149)
transformOptionalSelectInto(ParseState * pstate, Node * parseTree) (src\backend\parser\analyze2.c:257)
transformTopLevelStmt(ParseState * pstate, RawStmt * parseTree) (src\backend\parser\analyze.c:253)
standard_parse_analyze(RawStmt * parseTree, const char * sourceText, Oid * paramTypes, int numParams, QueryEnvironment * queryEnv) (src\backend\parser\analyze.c:162)
parse_analyze(RawStmt * parseTree, const char * sourceText, Oid * paramTypes, int numParams, QueryEnvironment * queryEnv) (src\backend\parser\parser_ep.c:45)
pg_analyze_and_rewrite(RawStmt * parsetree, const char * query_string, Oid * paramTypes, int numParams, QueryEnvironment * queryEnv) (src\backend\tcop\postgres.c:695)
standard_exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1257)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres2.c:65)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (src\backend\tcop\postgres.c:4802)
mainFunc(Port * port, int argc, char ** argv) (src\backend\adapter\mysql\adapter.c:808)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4572)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4297)
ServerLoop() (src\backend\postmaster\postmaster.c:1773)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1442)
main(int argc, char ** argv) (src\backend\main\main.c:209)
```

openHalo单个数据库实例可同时支持多种数据库模式，包括MySQL、PostgreSQL等，因此需要支持多种解析器引擎。当客户端连接时，openHalo会根据客户端的请求选择对应的解析引擎。每当客户端连接时，会fork一个新的进程，用于处理客户端的请求，该进程在初始化时会调用`InitPostgres`函数，在该过程中会调用`InitParserEngine()`选择合适的解析器。同时在该函数中，会进行一系列的初始化操作，包括执行`InitExecutorEngine()`选择合适的执行引擎，执行`InitPlannerEngine()`选择合适的优化器引擎。
我们看一下INSERT语句执行器相关调用栈：
```c++
ExecModifyTable(PlanState * pstate) (src\backend\executor\mys_nodeModifyTable.c:3063)
ExecProcNodeFirst(PlanState * node) (src\backend\executor\execProcnode.c:464)
ExecProcNode(PlanState * node) (src\include\executor\executor.h:260)
standard_ExecutePlan(EState * estate, PlanState * planstate, _Bool use_parallel_mode, CmdType operation, _Bool sendTuples, uint64 numberTuples, ScanDirection direction, DestReceiver * dest, _Bool execute_once) (src\backend\executor\execMain.c:1567)
ExecutePlan(EState * estate, PlanState * planstate, _Bool use_parallel_mode, CmdType operation, _Bool sendTuples, uint64 numberTuples, ScanDirection direction, DestReceiver * dest, _Bool execute_once) (src\backend\executor\execMain.c:2989)
mys_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\mys_execMain.c:220)
ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (src\backend\executor\execMain.c:315)
ProcessQuery(PlannedStmt * plan, const char * sourceText, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\pquery.c:160)
PortalRunMulti(Portal portal, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:1294)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:793)
standard_exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1341)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres2.c:65)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (src\backend\tcop\postgres.c:4802)
mainFunc(Port * port, int argc, char ** argv) (src\backend\adapter\mysql\adapter.c:808)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4572)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4297)
ServerLoop() (src\backend\postmaster\postmaster.c:1773)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1442)
main(int argc, char ** argv) (src\backend\main\main.c:209)
```


除了语法方面，openHalo还支持MySQL协议。为了支持多种协议，openHalo将原有PostgreSQL协议中相关接口进行了抽象，定义了`ProtocolInterface`结构体，该结构体中包含了多种接口函数，用于处理不同协议的连接、认证、消息处理等。
```c++
typedef struct ProtocolRoutine ProtocolInterface;

/*
 * ProtocolRoutine
 *
 * 	All the callbacks implementing a specific wire protocol
 *  AWS Babelfish compatible
 */
typedef void    (*fn_listen_init)(void);
typedef int		(*fn_accept)(pgsocket server_fd, struct Port *port);
typedef void	(*fn_close)(pgsocket server_fd);
typedef void	(*fn_init)(void);
typedef int		(*fn_start)(struct Port *port);
typedef void	(*fn_authenticate)(struct Port *port, const char **username);
typedef void	(*fn_mainfunc)(struct Port *port, int argc, char *argv[]) pg_attribute_noreturn();
typedef void	(*fn_send_message)(ErrorData *edata);
typedef void	(*fn_send_cancel_key)(int pid, int32 key);
typedef void	(*fn_comm_reset)(void);
typedef bool	(*fn_is_reading_msg)(void);
typedef void	(*fn_send_ready_for_query)(CommandDest dest);
typedef int		(*fn_read_command)(StringInfo inBuf);
typedef void	(*fn_end_command)(QueryCompletion *qc, CommandDest dest);
typedef bool	(*fn_printtup)(TupleTableSlot *slot, DestReceiver *self, CommandTag  commandTag);
typedef void	(*fn_printtup_startup)(DestReceiver *self, int operation, TupleDesc typeinfo, CommandTag  commandTag);
typedef void	(*fn_printtup_shutdown)(DestReceiver *self);
typedef void	(*fn_printtup_destroy)(DestReceiver *self);
typedef int		(*fn_process_command)(int *first_char, StringInfo inBuf);
typedef void	(*fn_report_param_status)(const char *name, char *val);

typedef struct ProtocolRoutine {
	NodeTag						type;
	
    fn_listen_init              listen_init;
	fn_accept					accept;
	fn_close					close;
	fn_init						init;
	fn_start					start;
	fn_authenticate				authenticate;
	fn_mainfunc					mainfunc;
	fn_send_message			    send_message;
	fn_send_cancel_key			send_cancel_key;
	fn_comm_reset				comm_reset;
	fn_is_reading_msg			is_reading_msg;
	fn_send_ready_for_query		send_ready_for_query;
	fn_read_command				read_command;
	fn_end_command				end_command;
	fn_printtup					printtup;
	fn_printtup_startup			printtup_startup;
	fn_printtup_shutdown		printtup_shutdown;
	fn_printtup_destroy			printtup_destroy;
	fn_process_command			process_command;
	fn_report_param_status		report_param_status;
} ProtocolRoutine;
```
其中MySQL协议接口定义如下：
```c++
static ProtocolInterface protocolHandler = {
    .type = T_MySQLProtocol,
    .listen_init = initListen,
    .accept = acceptConn,
    .close = closeListen,
	.init = initServer,
    .start = startServer,
	.authenticate = authenticate,
	.mainfunc = mainFunc,
	.send_message = sendErrorMessage,
	.send_cancel_key = mysqlSendCancelKey,
	.comm_reset = NULL,
	.is_reading_msg = NULL,
	.send_ready_for_query = sendReadyForQuery,
	.read_command = readCommand,
	.end_command = endCommand,
	.printtup = printTup,
	.printtup_startup = printTupStartup,
	.printtup_shutdown = printTupShutdown,
	.printtup_destroy = printTupDestroy,
	.process_command = processCommand,
    .report_param_status = reportParamStatus
};
```
其他方面，可通过代码调试来查看具体的实现逻辑。这里不再详细介绍。

### 源码结构分析
在openHalo的源码中，为兼容MySQL涉及如下代码，当前目前整理的肯定不够全面，后续再不断进行补充完善：
```c++
openHalo
--> contrib/aux_mysql          // MySQL兼容辅助扩展模块， 大概有三万多行SQL代码
            --> aux_mysql--1.1--1.2.sql
            --> aux_mysql--1.1.sql
            --> aux_mysql--1.2--1.3.sql
            --> aux_mysql--1.3--1.4.sql         
--> src/backend/parser/parsereng.c   // 解析引擎相关代码
--> src/backend/optimizer/plan/planner_engine.c  // 优化器引擎相关代码
--> src/backend/executor/executor_engine.c  // 执行引擎相关代码
--> src/backend/executor/mys_execMain.c
--> src/backend/executor/mys_execPartition.c
--> src/backend/executor/mys_execProcnode.c
--> src/backend/executor/mys_executor.c     // mysql执行引擎相关代码
--> src/backend/executor/mys_nodeModifyTable.c
--> src/backend/parser/mysql     // MySQL语法解析器相关代码， 大概有三万多行代码
                       --> check_myskeywords.pl
                       --> mys_analyze.c
                       --> mys_gram.y  // MySQL语法解析器
                       --> mys_keywords.c
                       --> mys_parse_agg.c
                       --> mys_parse_clause.c
                       --> mys_parse_expr.c
                       --> mys_parse_func.c
                       --> mys_parse_oper.c
                       --> mys_parse_utilcmd.c
                       --> mys_parser.c  // MySQL语法解析器入口
                       --> mys_scan.l  // lexical scanner for MySQL
--> src/backend/parse/analyze2.c   // extra routines to parse analyze
--> src/backend/adapter     // MySQL协议适配器 8k左右代码
                --> adapter.c
                --> errorConvertor.c
                --> netTransceiver.c
                --> pwdEncryptDecrypt.c
                --> systemVar.c
                --> userLogonAuth.c
                --> uuidShort.c
--> src/backend/commands/mysql
                         --> mys_prepare.c  // 实现MySQL的PREPARE语句
                         --> mys_sequence.c // 兼容MySQL sequence相关的处理
                         --> mys_tablecmds.c
                         --> mys_uservar.c
--> src/backend/commands/trigger2.c         // 实现MySQL触发器相关功能
--> src/backend/postmaster/postmaster2.c
--> src/backend/tcop/mysql
                     --> mys_utility.c
--> src/backend/tcop/postgres2.c

--> src/include/parser/mysql
--> src/include/optimizer/pathnode2.h
--> src/include/optimizer/plan_engine.h
--> src/include/parser/parserapi.h    // 解析器API
--> src/include/postmaster/protocol_interface.h
--> src/include/postmaster/postmaster2.h  
--> src/include/utils/mysql
                      --> mys_adtext.h
                      --> mys_date.h
                      --> mys_ri_trigger.h
                      --> mys_ruleutils.h
                      --> mys_timestamp.h
                      --> mys_varlena.h


```

### 个人思考
个人认为，openHalo数据库中将数据库协议，解析器，执行器进行抽象以支持多种数据库协议多种解析器来实现数据库兼容性是非常好的设计。在进行数据库内核开发时可以借鉴这种设计思想。


参考文档：
[多模式兼容](http://47.99.242.185/III.%E5%A4%9A%E6%A8%A1%E5%BC%8F%E5%85%BC%E5%AE%B9/)
