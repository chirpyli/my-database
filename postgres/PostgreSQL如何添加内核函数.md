在PostgreSQL内核的学习过程中，可以尝试向内核中添加一些函数，扩展PostgreSQL的功能。同时可以增加自己对PG内核的理解。这里我们以简单的添加一个helloworld函数为例，分析一下这个过程中涉及到的相关源码。

### PostgreSQL添加pg_helloworld函数
这里总结一下如何向PostgreSQL中添加内核函数，以`helloworld`为例，添加一个内核函数`pg_helloworld`，显示`Hello PostgreSQL!`。在添加之前，我们输入`select pg_helloworld()`，因为PostgreSQL中没有该内核函数，所以显示如下错误：
```sql
postgres=# select pg_helloworld();
ERROR:  function pg_helloworld() does not exist
LINE 1: select pg_helloworld();
               ^
HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
```

这里我们实现这个函数。过程如下：
1. 在`src/include/catalog/pg_proc.dat`中添加如下声明
```c++
# add function helloworld()    by chirpyli
{ oid => '9999', descr => 'Hello PostgreSQL',
  proname => 'pg_helloworld', prorettype => 'text',
  proargtypes => '', prosrc => 'pg_helloworld' },
```
其中含义如下：
oid：对象id，唯一不重复  
descr：函数描述信息
proname：函数名称        
prorettype：返回值类型
proargtypes：参数列表    
prosrc：函数名称

2. 在`src/backend/utils/adt/pseudotypes.c`中添加函数`pg_helloworld`
```c++
/*
* pg_helloworld
* function to show 'Hello PostgreSQL!'
*/
Datum
pg_helloworld(PG_FUNCTION_ARGS)
{
	char str[] = "Hello PostgreSQL!";
	PG_RETURN_TEXT_P(cstring_to_text(str));
}
```
这里说明一下参数，能够直接用SQL语句调用的函数（prosrc），他的参数必须是`PG_FUNCTION_ARGS`，其定义(`src/include/fmgr.h`)如下：
```c++
/* Standard parameter list for fmgr-compatible functions */
#define PG_FUNCTION_ARGS	FunctionCallInfo fcinfo
typedef struct FunctionCallInfoBaseData *FunctionCallInfo;
typedef struct FunctionCallInfoBaseData
{
	FmgrInfo   *flinfo;			/* ptr to lookup info used for this call */
	fmNodePtr	context;		/* pass info about context of call */
	fmNodePtr	resultinfo;		/* pass or return extra info about result */
	Oid			fncollation;	/* collation for function to use */
#define FIELDNO_FUNCTIONCALLINFODATA_ISNULL 4
	bool		isnull;			/* function must set true if result is NULL */
	short		nargs;			/* # arguments actually passed */
#define FIELDNO_FUNCTIONCALLINFODATA_ARGS 6
	NullableDatum args[FLEXIBLE_ARRAY_MEMBER];
} FunctionCallInfoBaseData;

typedef Datum (*PGFunction) (FunctionCallInfo fcinfo);

typedef struct FmgrInfo
{
	PGFunction	fn_addr;		/* pointer to function or handler to be called */
	Oid			fn_oid;			/* OID of function (NOT of handler, if any) */
	short		fn_nargs;		/* number of input args (0..FUNC_MAX_ARGS) */
	bool		fn_strict;		/* function is "strict" (NULL in => NULL out) */
	bool		fn_retset;		/* function returns a set */
	unsigned char fn_stats;		/* collect stats if track_functions > this */
	void	   *fn_extra;		/* extra space for use by handler */
	MemoryContext fn_mcxt;		/* memory context to store fn_extra in */
	fmNodePtr	fn_expr;		/* expression parse tree for call, or NULL */
} FmgrInfo;
```

3. 验证是否添加成功，重新编译`make && make install`，初始化数据库`initdb`，`psql`连接数据库，`select pg_helloworld()`查看是否添加成功，结果如下，添加成功。
```sql
postgres=# select pg_helloworld();
   pg_helloworld   
-------------------
 Hello PostgreSQL!
(1 row)

```

### 源码分析
上面成功的添加了`pg_helloworld`函数后，我们深入思考一下，进行源码分析，看一下其中的细节。数据库处理函数大概的流程是用户发起了调用函数的SQL语句，PG要解析SQL语句，生成语法解析树，首先要识别出是调用系统函数，然后在`pg_proc`系统表中查询是否有该函数，这个过程是在语义分析阶段做的，最后生成计划树。我们一步一步进行源码分析。具体分析跟踪源码的时候可以用`select pg_backend_pid()`进行分析。

#### 解析部分
这部分主要是在词法语法分析阶段，识别出是调用函数。关于SQL调用的前期过程以及词法分析过程可参考上一篇[PostgreSQL中表名，列名的长度限制](https://mp.weixin.qq.com/s/xpfYqkB4_loJvchG5s5fuA)，里面有相关的源码分析。这里不再细述。这里只关心解析出函数部分。

**解析部分调用主流程如下：**
```c++
main(int argc, char *argv[])
--> PostmasterMain(argc, argv);
    --> ServerLoop();
        --> BackendStartup(port);
            --> BackendRun(port);
                --> PostgresMain(ac, av, port->database_name, port->user_name);
                    --> for (;;)        // 在这里不断接收客户端的请求，处理
                        --> exec_simple_query(const char *query_string)     
                            --> pg_parse_query(query_string)    // 解析SQL
                                --> raw_parser(query_string)
                                    // gram.c中宏定义 #define yyparse         base_yyparse，实际调用yyparse()
                                    --> base_yyparse(yyscanner)     

```

下面分析一下匹配`select pg_helloworld()`的过程，这个过程主要是分析`gram.y`。下面的流程是其匹配的过程，不熟悉的话可以倒着看。
```c++
// 忽略其他代码......
%%
// The target production for the whole parse.
stmtblock:	stmtmulti
			{
				pg_yyget_extra(yyscanner)->parsetree = $1;
			}
		;
stmtmulti:	stmtmulti ';' stmt      // 识别多个SQL语句
				{
					if ($1 != NIL)
					{
						/* update length of previous stmt */
						updateRawStmtEnd(llast_node(RawStmt, $1), @2);
					}
					if ($3 != NULL)
						$$ = lappend($1, makeRawStmt($3, @2 + 1));
					else
						$$ = $1;
				}
			| stmt
				{
					if ($1 != NULL)
						$$ = list_make1(makeRawStmt($1, 0));
					else
						$$ = NIL;
				}
		;        

stmt :		// 省略很多其他语句的代码......
		    AlterEventTrigStmt	
            | SelectStmt        // 重点关注select
            | /*EMPTY*/
				{ $$ = NULL; }

SelectStmt: select_no_parens			%prec UMINUS
			| select_with_parens		%prec UMINUS
		;

select_no_parens:
			simple_select						{ $$ = $1; }
            | // 忽略其他匹配的代码......

simple_select:
			SELECT opt_all_clause opt_target_list
			into_clause from_clause where_clause
			group_clause having_clause window_clause
				{
					SelectStmt *n = makeNode(SelectStmt);	// select 匹配SelectSmtm
					n->targetList = $3;		// 这个是后面的pg_helloworld()，ResTarget
					n->intoClause = $4;
					n->fromClause = $5;
					n->whereClause = $6;
					n->groupClause = $7;
					n->havingClause = $8;
					n->windowClause = $9;
					$$ = (Node *)n;
				}

opt_target_list: target_list						{ $$ = $1; }
			| /* EMPTY */							{ $$ = NIL; }
		;

target_list:
			target_el								{ $$ = list_make1($1); }
			| target_list ',' target_el				{ $$ = lappend($1, $3); }
		;

target_el:
			// 省略部分代码......
			a_expr
				{
					$$ = makeNode(ResTarget);
					$$->name = NULL;
					$$->indirection = NIL;
					$$->val = (Node *)$1;		// 对应上面的a_expr，对应下面的func_expr
					$$->location = @1;
				}
		;

a_expr:		c_expr									{ $$ = $1; }
            | // 忽略其他匹配的代码......

c_expr:		columnref								{ $$ = $1; }
			| AexprConst							{ $$ = $1; }
			| case_expr
				{ $$ = $1; }
			| func_expr
				{ $$ = $1; }
            | // 忽略其他匹配的代码......            

func_expr: func_application within_group_clause filter_clause over_clause
				{
					FuncCall *n = (FuncCall *) $1;
					// 省略代码......
					n->agg_filter = $3;
					n->over = $4;
					$$ = (Node *) n;
				}
		;

// 匹配select pg_helloworld()
func_application: func_name '(' ')'
				{
					$$ = (Node *) makeFuncCall($1, NIL, @1);
				}

func_name:	type_function_name
					{ $$ = list_make1(makeString($1)); }
			| ColId indirection
					{
						$$ = check_func_name(lcons(makeString($1), $2),
											 yyscanner);
					}
		;

/* Type/function identifier --- names that can be type or function names.*/
type_function_name:	IDENT							{ $$ = $1; }
			| unreserved_keyword					{ $$ = pstrdup($1); }
			| type_func_name_keyword				{ $$ = pstrdup($1); }
		;        

%%
// 忽略其他代码......
```
上面定义的`gram.y`，实际代码调用的是其生成的`gram.c`，具体的会调用`yyparse()`，里面会调用`yylex`进行词法分析。这里面主要是根据在`gram.y`文件中规则段定义的规则生成的C代码，大的方面可以理解为里面有个switch-case，每个分支对于一个规则匹配。
```c++
int yyparse (core_yyscan_t yyscanner) {
// 忽略部分代码...
yybackup:
// 忽略部分代码...
  if (yychar == YYEMPTY) {
      YYDPRINTF ((stderr, "Reading a token: "));
      yychar = yylex (&yylval, &yylloc, yyscanner);     // 调用yylex
   }
// 忽略部分代码...
yyreduce:   
  switch (yyn) {
    // 忽略部分代码.....，这里只列出最关键的匹配,其他匹配代码见gram.y
  case 1663: {
					SelectStmt *n = makeNode(SelectStmt);		// 匹配SelectStmt
					n->targetList = (yyvsp[-6].list);		// 函数表达式是在这里
					n->intoClause = (yyvsp[-5].into);
					n->fromClause = (yyvsp[-4].list);
					n->whereClause = (yyvsp[-3].node);
					n->groupClause = (yyvsp[-2].list);
					n->havingClause = (yyvsp[-1].node);
					n->windowClause = (yyvsp[0].list);
					(yyval.node) = (Node *)n;
				}
    break;	
    case 2039: {
        (yyval.node) = (Node *) makeFuncCall((yyvsp[-2].list), NIL, (yylsp[-2]));
        }
        break; 
    case 2270:
        { (yyval.list) = list_make1(makeString((yyvsp[0].str))); }
        break;        
    case 2299:
        { (yyval.str) = (yyvsp[0].str); }
        break;           
    // 忽略其他代码......    
  }

}

FuncCall *makeFuncCall(List *name, List *args, int location) {
	FuncCall   *n = makeNode(FuncCall);

	n->funcname = name;
	n->args = args;
	n->agg_order = NIL;
	n->agg_filter = NULL;
	n->agg_within_group = false;
	n->agg_star = false;
	n->agg_distinct = false;
	n->func_variadic = false;
	n->over = NULL;
	n->location = location;
	return n;
}

typedef struct FuncCall {
	NodeTag		type;
	List	   *funcname;		/* qualified name of function */
	List	   *args;			/* the arguments (list of exprs) */
	List	   *agg_order;		/* ORDER BY (list of SortBy) */
	Node	   *agg_filter;		/* FILTER clause, if any */
	bool		agg_within_group;	/* ORDER BY appeared in WITHIN GROUP */
	bool		agg_star;		/* argument was really '*' */
	bool		agg_distinct;	/* arguments were labeled DISTINCT */
	bool		func_variadic;	/* last argument was labeled VARIADIC */
	struct WindowDef *over;		/* OVER clause, if any */
	int			location;		/* token location, or -1 if unknown */
} FuncCall;
```
针对`select pg_helloworld()`上面的过程解析出来是一个`SelectStmt`节点,`SelectStmt`中有个`targetList`字段（里面是`ResTarget`），

> 注意上面的分析只是针对`select pg_helloworld()`的分析过程，如果是其他语句，其过程是不同的，字段的值类型等是不同的。

到这里，我们清楚了`select pg_helloworld()`的语法分析部分。我们继续分析，PG是如何查找函数是否存在的。 数据库中所有内部函数信息都存储在系统表`pg_proc`中，所以肯定是要访问这个系统表的。在`pg_parse_query`后返回语法解析树，需要经过`parse_analyze`转为查询树`Query`。在这个转换的过程中，会调用`transformStmt`，在这个过程中会进行查表，我们后面看一下debug过程。注意，因为这个函数没有参数，所以参数处理的代码就没有跟踪。查询树经过`pg_plan_queries`生成计划树，在`PortalRun`中执行。具体的，要调用`ExecInterpExpr`，可以看一下这个函数的内部实现，在这个函数里面再去调用自己新添加的函数。下面是具体的分析调用过程：
```c++
void exec_simple_query(const char *query_string)
	// 生成语法解析树，只会进行语法检查，不进行语义检查，输入未定义的函数，在这里不会报错
    --> pg_parse_query(query_string)
        --> raw_parser(query_string)    // 里面具体内容上面已经分析过了
    --> pg_analyze_and_rewrite()        // 语义分析查询重写
		// 语义分析，将语法解析树转为查询树Query tree.  函数是否存在，在这里进行检查，确保语义是对的，能够被执行
		--> parse_analyze()		
			--> transformTopLevelStmt() 
				--> transformOptionalSelectInto() 
					--> transformStmt()
						--> transformSelectStmt()
							--> transformTargetList()	// 在这里赋值query->targetList 字段
								--> transformTargetEntry()
									--> transformExpr()
										--> transformExprRecurse()
										--> transformFuncCall()		// 重点，
											// Parse a function call
											--> ParseFuncOrColumn()		// 如果没有找到函数的话，报错
												// Find the named function in the system catalogs.
												--> func_get_detail()	// 如果没有找到返回FUNCDETAIL_NOTFOUND
													--> FuncnameGetCandidates()	// 会返回函数的oid
														--> DeconstructQualifiedName()	// 获得函数名pg_helloworld
														--> SearchSysCacheList1()	// PG设置了高速缓存Cache来提高系统表的访问效率
															--> SearchCatCacheList()	// 对系统表的访问后面文章单独会讲
																// 如果cache中找到，返回，如果cache中没有找到，就进入下面扫描pg_proc系统表
																--> table_open(cache->cc_reloid, AccessShareLock);	// reloid=1255 CATALOG(pg_proc,1255,ProcedureRelationId) 
																--> systable_beginscan()
												--> make_fn_arguments()	// 参数处理，这里先忽略
												// 生成FuncExpr  
									--> makeTargetEntry()	// 里面干的就是将FuncExpr节点放到TargetEntry的expr字段中
		--> pg_rewrite_query()	// 查询重写
			--> QueryRewrite()
	--> pg_plan_queries()		// 基于查询树生成查询计划树
		--> pg_plan_query()
			--> planner()  /* call the optimizer */
				--> standard_planner()
					--> subquery_planner()
						--> preprocess_expression()	 // select pg_helloworld这里没做任何处理，只是为了下面的，知道常量表达式是怎么处理的
							--> eval_const_expressions()	// 常量表达式在这里处理
					--> create_plan()
						--> create_plan_recurse()
							--> create_projection_plan()
	// 计划执行部分
	--> PortalStart()
	--> PortalRun()
		--> PortalRunSelect()
			--> ExecutorRun()
				--> standard_ExecutorRun()
					--> ExecutePlan()
						--> ExecProcNode()
							--> ExecProcNodeFirst()
								--> ExecResult()
									--> ExecProject()
										// Same as ExecEvalExpr, but get into the right allocation context explicitly.
										--> ExecEvalExprSwitchContext()	// 到这里是关键部分了
											--> ExecInterpExprStillValid()
												--> ExecInterpExpr()	// 执行
													--> pg_backend_pid()	// 这里是select pg_backend_pid时的调用


```
到这里，应该清楚的知道整体的流程了，当然还是有很多很多细节无法一一描述清楚，待进一步学习理解。

```c++
#define EEO_CASE(name)		CASE_##name:

/* Evaluate expression identified by "state" in the execution context
 * given by "econtext".  *isnull is set to the is-null flag for the result,
 * and the Datum value is the function result.*/
static Datum ExecInterpExpr(ExprState *state, ExprContext *econtext, bool *isnull) {
		// 因为select pg_helloworld()，函数中的实现是返回一个字符串常量，所以没有走下面的流程，验证走EEO_CASE(EEOP_FUNCEXPR)的话可以用select pg_backend_pid()来验证一下。
		EEO_CASE(EEOP_CONST)
		{
			*op->resnull = op->d.constval.isnull;
			*op->resvalue = op->d.constval.value;

			EEO_NEXT();
		}
		/*
		 * Function-call implementations. Arguments have previously been
		 * evaluated directly into fcinfo->args.
		 *
		 * As both STRICT checks and function-usage are noticeable performance
		 * wise, and function calls are a very hot-path (they also back
		 * operators!), it's worth having so many separate opcodes.
		 *
		 * Note: the reason for using a temporary variable "d", here and in
		 * other places, is that some compilers think "*op->resvalue = f();"
		 * requires them to evaluate op->resvalue into a register before
		 * calling f(), just in case f() is able to modify op->resvalue
		 * somehow.  The extra line of code can save a useless register spill
		 * and reload across the function call.
		 */
		EEO_CASE(EEOP_FUNCEXPR)
		{
			FunctionCallInfo fcinfo = op->d.func.fcinfo_data;
			Datum		d;

			fcinfo->isnull = false;
			d = op->d.func.fn_addr(fcinfo);		// 调用自己添加的函数
			*op->resvalue = d;
			*op->resnull = fcinfo->isnull;

			EEO_NEXT();
		}
}

static inline Datum ExecEvalExprSwitchContext(ExprState *state, ExprContext *econtext, bool *isNull) {
	Datum		retDatum;
	MemoryContext oldContext;

	oldContext = MemoryContextSwitchTo(econtext->ecxt_per_tuple_memory);
	retDatum = state->evalfunc(state, econtext, isNull);
	MemoryContextSwitchTo(oldContext);
	return retDatum;
}

/* FuncExpr - expression node for a function call */
typedef struct FuncExpr {
	Expr		xpr;
	Oid			funcid;			/* PG_PROC OID of the function */
	Oid			funcresulttype; /* PG_TYPE OID of result value */
	bool		funcretset;		/* true if function returns set */
	bool		funcvariadic;	/* true if variadic arguments have been
								 * combined into an array last argument */
	CoercionForm funcformat;	/* how to display this function call */
	Oid			funccollid;		/* OID of collation of result */
	Oid			inputcollid;	/* OID of collation that function should use */
	List	   *args;			/* arguments to the function */
	int			location;		/* token location, or -1 if unknown */
} FuncExpr;

```


---

- [用户定义的函数](http://www.postgres.cn/docs/13/xfunc.html)
- [PostgreSQL开发之Hello, PG!!!](https://pgfans.cn/a?id=417)
- [PostgreSQL的系统函数分析记录](https://my.oschina.net/Suregogo/blog/311648?tdsourcetag=s_pcqq_aiomsg)
- [PostgreSQL Executor(5): 表达式](https://www.jianshu.com/p/19d74529b170?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)
