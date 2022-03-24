### EXPLAIN显示执行计划源码分析
在Postgres中，显示执行计划可用explain命令，示例如下：
```sql
postgres@postgres=# explain select tableOid,ctid,name from students limit 2;
                            QUERY PLAN                             
-------------------------------------------------------------------
 Limit  (cost=0.00..0.04 rows=2 width=42)
   ->  Seq Scan on students  (cost=0.00..21.80 rows=1180 width=42)
(2 rows)
```
我们分析一下源码，看一下`EXPLAIN`是怎么显示执行计划的，选取了一条简单的查询语句。如果是数据库内核开发者，在新实现某个算子或者新增了某个Plan，这时还需要改一下执行计划显示相关代码，主要在explain.c中。我们分析一下其大概的过程。

#### 源码分析：
输入这条语句`explain select tableOid,ctid,name from students limit 2;`进行分析。

在进行源码跟踪前想一下，如果自己实现，应该怎么做呢？大概步骤如下：
1. 在gram.y中定义explain的语法，（包括新增关键字EXPLAIN）  定义`ExplainStmt`，语法分析
2. 因为目的是显示执行计划，所以肯定要先能正确的生成执行计划，然后才能进行显示。所以，接下来是生成执行计划的过程。 `ExplainStmt`生成`Query`，之后可调用`pg_plan_query`生成执行计划
3. 生成执行计划后，通过一个函数，入参为生成的执行计划Plan，输出为text字符串，打印出来。

后面我们跟踪一下源码，看源码是怎么实现的。主要代码在`src/backend/commands/explain.c`中。


#### 主要流程
我们看一下其执行流程
```c++
exec_simple_query(const char *query_string)
--> pg_parse_query(query_string);   // 1. 生成语法解析树，对应ExplainStmt
// 语义分析重写
--> pg_analyze_and_rewrite()    
    --> parse_analyze()         // 语义分析
        --> transformStmt()     // 语法解析树生成查询树
            --> transformExplainStmt()      // ExplanStmt  -->  Query
    --> pg_rewrite_query() 
// query->commandType == CMD_UTILITY，不进入pg_plan_query进行查询优化处理，直接进入PortalStart()
--> pg_plan_queries()   // explain为CMD_UTILITY直接将Query封装为PlannedStmt   
    --> pg_plan_query()   // 不进入查询优化，直接到执行器
--> PortalStart()
--> PortalRun()
    --> FillPortalStore()
        --> PortalRunUtility()
            --> ProcessUtility()
                --> standard_ProcessUtility()
                    --> ExplainQuery()
                        --> QueryRewrite()
                        --> ExplainBeginOutput()
                        --> ExplainOneQuery()
                            // 2. 生成explain后面的查询语句的执行计划Plan，对应的是select语句的执行计划
                            --> pg_plan_query()     
                            --> ExplainOnePlan()
                                --> ExplainPrintPlan()  // 3. 打印执行计划，输出字符串。
                                    --> ExplainNode()
                        --> ExplainEndOutput()
--> PortalDrop()
```

#### EXPLAIN语法定义
在gram.y中定义了explain的语法。
```sql
/*****************************************************************************
 *
 *		QUERY:
 *				EXPLAIN [ANALYZE] [VERBOSE] query
 *				EXPLAIN ( options ) query
 *
 ****************************************************************************/
ExplainStmt:
		EXPLAIN ExplainableStmt     -- explain的语法定义
				{
					ExplainStmt *n = makeNode(ExplainStmt);
					n->query = $2;
					n->options = NIL;
					$$ = (Node *) n;
				}
		| EXPLAIN analyze_keyword opt_verbose ExplainableStmt
				{
					ExplainStmt *n = makeNode(ExplainStmt);
					n->query = $4;
					n->options = list_make1(makeDefElem("analyze", NULL, @2));
					if ($3)
						n->options = lappend(n->options,
											 makeDefElem("verbose", NULL, @3));
					$$ = (Node *) n;
				}
		| EXPLAIN VERBOSE ExplainableStmt
				{
					ExplainStmt *n = makeNode(ExplainStmt);
					n->query = $3;
					n->options = list_make1(makeDefElem("verbose", NULL, @2));
					$$ = (Node *) n;
				}
		| EXPLAIN '(' explain_option_list ')' ExplainableStmt
				{
					ExplainStmt *n = makeNode(ExplainStmt);
					n->query = $5;
					n->options = $3;
					$$ = (Node *) n;
				}
		;

ExplainableStmt:
			SelectStmt      -- 可对以下这些语句进行执行计划显示
			| InsertStmt
			| UpdateStmt
			| DeleteStmt
			| MergeStmt
			| DeclareCursorStmt
			| CreateAsStmt
			| CreateMatViewStmt
			| RefreshMatViewStmt
			| ExecuteStmt					/* by default all are $$=$1 */
		;

explain_option_list:
			explain_option_elem
				{
					$$ = list_make1($1);
				}
			| explain_option_list ',' explain_option_elem
				{
					$$ = lappend($1, $3);
				}
		;

explain_option_elem:
			explain_option_name explain_option_arg
				{
					$$ = makeDefElem($1, $2, @1);
				}
		;

explain_option_name:
			NonReservedWord			{ $$ = $1; }
			| analyze_keyword		{ $$ = "analyze"; }
		;

explain_option_arg:
			opt_boolean_or_string	{ $$ = (Node *) makeString($1); }
			| NumericOnly			{ $$ = (Node *) $1; }
			| /* EMPTY */			{ $$ = NULL; }
		;
```

这块不再细述。继续跟踪源码。

#### 关键数据结构及代码
语法解析表示为`ExplainStmt`：
```c++
/* ----------------------
 *		Explain Statement
 *
 * The "query" field is initially a raw parse tree, and is converted to a
 * Query node during parse analysis.  Note that rewriting and planning
 * of the query are always postponed until execution.
 * ---------------------- */
typedef struct ExplainStmt
{
	NodeTag		type;
	Node	   *query;			/* the query (see comments above) */
	List	   *options;		/* list of DefElem nodes */
} ExplainStmt; 
```

在`transformExplainStmt`函数中将`ExplainStmt`转为查询树`Query`。
```c++
/*
 * transformExplainStmt -	transform an EXPLAIN Statement
 * EXPLAIN is like other utility statements in that we emit it as a CMD_UTILITY Query node;
 */
static Query *
transformExplainStmt(ParseState *pstate, ExplainStmt *stmt)
{
	Query	   *result;

	/* transform contained query, allowing SELECT INTO */
	stmt->query = (Node *) transformOptionalSelectInto(pstate, stmt->query);
    // 需要注意这里的处理，stmt->query经transformOptionalSelectInto处理前为SelectStmt结构，处理后为Query结构 
    // 这里必须要这样，进行语义分析，分析explain后面的查询语句语义是否正确，如果不正确，提前报错退出。
    // 后面要需要依据这个查询树生成执行计划。   
	/* represent the command as a utility Query */
	result = makeNode(Query);
	result->commandType = CMD_UTILITY;
	result->utilityStmt = (Node *) stmt;  //这里的stmt->query已经将ExplainStmt中query字段的语法解析树转为了查询树了。

	return result;
}
```

语义分析结束后进入到查询优化这里，在`pg_plan_queries`中，因为explain是`CMD_UTILITY`，所以直接由`Query`生成`PlannedStmt`。
```c++
List *pg_plan_queries(List *querytrees, const char *query_string, int cursorOptions, ParamListInfo boundParams)
{
	List	   *stmt_list = NIL;
	ListCell   *query_list;

	foreach(query_list, querytrees)
	{
		Query	   *query = lfirst_node(Query, query_list);
		PlannedStmt *stmt;

		if (query->commandType == CMD_UTILITY)
		{
			/* Utility commands require no planning. */
			stmt = makeNode(PlannedStmt);
			stmt->commandType = CMD_UTILITY;   // explain命令为utility，在这里封装PlannedStmt，供执行器执行。
			stmt->canSetTag = query->canSetTag;
			stmt->utilityStmt = query->utilityStmt;
			stmt->stmt_location = query->stmt_location;
			stmt->stmt_len = query->stmt_len;
		}
		else
			stmt = pg_plan_query(query, query_string, cursorOptions, boundParams);

		stmt_list = lappend(stmt_list, stmt);
	}

	return stmt_list;
}
```

后面就进入到执行器中了，具体执行到`ExplainQuery`中实现输出执行计划。
```c++
/* ExplainQuery -	  execute an EXPLAIN command*/
void ExplainQuery(ParseState *pstate, ExplainStmt *stmt, ParamListInfo params, DestReceiver *dest)
{
	ExplainState *es = NewExplainState();
	TupOutputState *tstate;
	List	   *rewritten;
	ListCell   *lc;
	bool		timing_set = false;
	bool		summary_set = false;

	/* Parse options list. */
	foreach(lc, stmt->options)
	{
		DefElem    *opt = (DefElem *) lfirst(lc);

		if (strcmp(opt->defname, "analyze") == 0)
			es->analyze = defGetBoolean(opt);
		else if (strcmp(opt->defname, "verbose") == 0)
			es->verbose = defGetBoolean(opt);
		else if (strcmp(opt->defname, "costs") == 0)
			es->costs = defGetBoolean(opt);
		else if (strcmp(opt->defname, "buffers") == 0)
			es->buffers = defGetBoolean(opt);
		else if (strcmp(opt->defname, "wal") == 0)
			es->wal = defGetBoolean(opt);
		else if (strcmp(opt->defname, "settings") == 0)
			es->settings = defGetBoolean(opt);
		else if (strcmp(opt->defname, "timing") == 0)
		{
			timing_set = true;
			es->timing = defGetBoolean(opt);
		}
		else if (strcmp(opt->defname, "summary") == 0)
		{
			summary_set = true;
			es->summary = defGetBoolean(opt);
		}
		else if (strcmp(opt->defname, "format") == 0)
		{
			char	   *p = defGetString(opt);

			if (strcmp(p, "text") == 0)
				es->format = EXPLAIN_FORMAT_TEXT;
			else if (strcmp(p, "xml") == 0)
				es->format = EXPLAIN_FORMAT_XML;
			else if (strcmp(p, "json") == 0)
				es->format = EXPLAIN_FORMAT_JSON;
			else if (strcmp(p, "yaml") == 0)
				es->format = EXPLAIN_FORMAT_YAML;
			else
				ereport(ERROR,(errcode(ERRCODE_INVALID_PARAMETER_VALUE), errmsg("unrecognized value for EXPLAIN option \"%s\": \"%s\"",opt->defname, p),parser_errposition(pstate, opt->location)));
		}
		else
			ereport(ERROR,(errcode(ERRCODE_SYNTAX_ERROR),errmsg("unrecognized EXPLAIN option \"%s\"", opt->defname),parser_errposition(pstate, opt->location)));
	}

	if (es->wal && !es->analyze)
		ereport(ERROR,(errcode(ERRCODE_INVALID_PARAMETER_VALUE),errmsg("EXPLAIN option WAL requires ANALYZE")));

	/* if the timing was not set explicitly, set default value */
	es->timing = (timing_set) ? es->timing : es->analyze;

	/* check that timing is used with EXPLAIN ANALYZE */
	if (es->timing && !es->analyze)
		ereport(ERROR,(errcode(ERRCODE_INVALID_PARAMETER_VALUE),errmsg("EXPLAIN option TIMING requires ANALYZE")));

	/* if the summary was not set explicitly, set default value */
	es->summary = (summary_set) ? es->summary : es->analyze;

	/* Parse analysis was done already, but we still have to run the rule
	 * rewriter.  We do not do AcquireRewriteLocks: we assume the query either
	 * came straight from the parser, or suitable locks were acquired by plancache.c.
	 *
	 * Because the rewriter and planner tend to scribble on the input, we make
	 * a preliminary copy of the source querytree.  This prevents problems in
	 * the case that the EXPLAIN is in a portal or plpgsql function and is executed repeatedly. */
	rewritten = QueryRewrite(castNode(Query, copyObject(stmt->query)));

	ExplainBeginOutput(es);	/* emit opening boilerplate */

	if (rewritten == NIL)
	{
		/* In the case of an INSTEAD NOTHING, tell at least that.  But in
		 * non-text format, the output is delimited, so this isn't necessary.*/
		if (es->format == EXPLAIN_FORMAT_TEXT)
			appendStringInfoString(es->str, "Query rewrites to nothing\n");
	}
	else
	{
		ListCell   *l;

		/* Explain every plan */
		foreach(l, rewritten)
		{
            // 重点在这里，
			ExplainOneQuery(lfirst_node(Query, l),CURSOR_OPT_PARALLEL_OK, NULL, es, pstate->p_sourcetext, params, pstate->p_queryEnv);

			/* Separate plans with an appropriate separator */
			if (lnext(rewritten, l) != NULL)
				ExplainSeparatePlans(es);
		}
	}

	/* emit closing boilerplate */
	ExplainEndOutput(es);
	Assert(es->indent == 0);

	/* output tuples */
	tstate = begin_tup_output_tupdesc(dest, ExplainResultDesc(stmt), &TTSOpsVirtual);
	if (es->format == EXPLAIN_FORMAT_TEXT)
		do_text_output_multiline(tstate, es->str->data);
	else
		do_text_output_oneline(tstate, es->str->data);
	end_tup_output(tstate);

	pfree(es->str->data);
}

/*
 * ExplainOneQuery -
 *	  print out the execution plan for one Query
 *
 * "into" is NULL unless we are explaining the contents of a CreateTableAsStmt.
 */
static void
ExplainOneQuery(Query *query, int cursorOptions, IntoClause *into, ExplainState *es,
				const char *queryString, ParamListInfo params, QueryEnvironment *queryEnv)
{
	/* planner will not cope with utility statements */
	if (query->commandType == CMD_UTILITY)
	{
		ExplainOneUtility(query->utilityStmt, into, es, queryString, params, queryEnv);
		return;
	}

	/* if an advisor plugin is present, let it manage things */
	if (ExplainOneQuery_hook)
		(*ExplainOneQuery_hook) (query, cursorOptions, into, es, queryString, params, queryEnv);
	else
	{
		PlannedStmt *plan;
		instr_time	planstart, planduration;
		BufferUsage bufusage_start, bufusage;

		if (es->buffers)
			bufusage_start = pgBufferUsage;
		INSTR_TIME_SET_CURRENT(planstart);

		/* plan the query */  // 生成查询语句的查询计划， 对应之前的是stmt->query
		plan = pg_plan_query(query, queryString, cursorOptions, params);

		INSTR_TIME_SET_CURRENT(planduration);
		INSTR_TIME_SUBTRACT(planduration, planstart);

		/* calc differences of buffer counters. */
		if (es->buffers)
		{
			memset(&bufusage, 0, sizeof(BufferUsage));
			BufferUsageAccumDiff(&bufusage, &pgBufferUsage, &bufusage_start);
		}

		/* run it (if needed) and produce output */ // 输出查询计划。
		ExplainOnePlan(plan, into, es, queryString, params, queryEnv, &planduration, (es->buffers ? &bufusage : NULL));
	}
}

/*
 * ExplainPrintPlan -
 *	  convert a QueryDesc's plan tree to text and append it to es->str
 *
 * The caller should have set up the options fields of *es, as well as
 * initializing the output buffer es->str.  Also, output formatting state
 * such as the indent level is assumed valid.  Plan-tree-specific fields
 * in *es are initialized here.
 *
 * NB: will not work on utility statements
 */
void
ExplainPrintPlan(ExplainState *es, QueryDesc *queryDesc)
{
	Bitmapset  *rels_used = NULL;
	PlanState  *ps;

	/* Set up ExplainState fields associated with this plan tree */
	Assert(queryDesc->plannedstmt != NULL);
	es->pstmt = queryDesc->plannedstmt;
	es->rtable = queryDesc->plannedstmt->rtable;
	ExplainPreScanNode(queryDesc->planstate, &rels_used);
	es->rtable_names = select_rtable_names_for_explain(es->rtable, rels_used);
	es->deparse_cxt = deparse_context_for_plan_tree(queryDesc->plannedstmt, es->rtable_names);
	es->printed_subplans = NULL;

	/*
	 * Sometimes we mark a Gather node as "invisible", which means that it's
	 * not to be displayed in EXPLAIN output.  The purpose of this is to allow
	 * running regression tests with force_parallel_mode=regress to get the
	 * same results as running the same tests with force_parallel_mode=off.
	 * Such marking is currently only supported on a Gather at the top of the
	 * plan.  We skip that node, and we must also hide per-worker detail data
	 * further down in the plan tree.
	 */
	ps = queryDesc->planstate;
	if (IsA(ps, GatherState) && ((Gather *) ps->plan)->invisible)
	{
		ps = outerPlanState(ps);
		es->hide_workers = true;
	}
	ExplainNode(ps, NIL, NULL, NULL, es);

	/*
	 * If requested, include information about GUC parameters with values that
	 * don't match the built-in defaults.
	 */
	ExplainPrintSettings(es);
}

/*
 * ExplainNode -
 *	  Appends a description of a plan tree to es->str
 *
 * planstate points to the executor state node for the current plan node.
 * We need to work from a PlanState node, not just a Plan node, in order to
 * get at the instrumentation data (if any) as well as the list of subplans.
 *
 * ancestors is a list of parent Plan and SubPlan nodes, most-closely-nested
 * first.  These are needed in order to interpret PARAM_EXEC Params.
 *
 * relationship describes the relationship of this plan node to its parent
 * (eg, "Outer", "Inner"); it can be null at top level.  plan_name is an
 * optional name to be attached to the node.
 *
 * In text format, es->indent is controlled in this function since we only
 * want it to change at plan-node boundaries (but a few subroutines will
 * transiently increment it).  In non-text formats, es->indent corresponds
 * to the nesting depth of logical output groups, and therefore is controlled
 * by ExplainOpenGroup/ExplainCloseGroup.
 */
static void
ExplainNode(PlanState *planstate, List *ancestors, const char *relationship, const char *plan_name, ExplainState *es)
{
    Plan	   *plan = planstate->plan;
	const char *pname;			/* node type name for text output */
	const char *sname;			/* node type name for non-text output */
	const char *strategy = NULL;
	const char *partialmode = NULL;
	const char *operation = NULL;
	const char *custom_name = NULL;
	ExplainWorkersState *save_workers_state = es->workers_state;
	int			save_indent = es->indent;
	bool		haschildren;

	/*
	 * Prepare per-worker output buffers, if needed.  We'll append the data in
	 * these to the main output string further down.
	 */
	if (planstate->worker_instrument && es->analyze && !es->hide_workers)
		es->workers_state = ExplainCreateWorkersState(planstate->worker_instrument->num_workers);
	else
		es->workers_state = NULL;

	/* Identify plan node type, and print generic details */
	switch (nodeTag(plan))
	{
		case T_Result:
			pname = sname = "Result";
			break;
		case T_ProjectSet:
			pname = sname = "ProjectSet";
			break;
		case T_ModifyTable:
			sname = "ModifyTable";
			switch (((ModifyTable *) plan)->operation)
			{
				case CMD_INSERT:
					pname = operation = "Insert";
					break;
				case CMD_UPDATE:
					pname = operation = "Update";
					break;
				case CMD_DELETE:
					pname = operation = "Delete";
					break;
				case CMD_MERGE:
					pname = operation = "Merge";
					break;
				default:
					pname = "???";
					break;
			}
			break;
    // 后面代码太多了，省略代码......
}
```



---

好了，就分析到这里吧。