### PostgreSQL源码分析 —— 除法/ 

我们分析一下在PostgreSQL数据库中，是如何实现除法操作的。其他的运算符操作与之类似。以下面的语句为例：
```sql
select 2/4;
```
#### 源码分析
主流程如下：
```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
--> pg_plan_queries
--> PortalStart
--> PortalRun
--> PortalDrop
```

##### 解析部分
```c++
exec_simple_query
--> pg_parse_query   // 构造语法解析树SelectStmt-->ResTarget-->A_Expr
    --> raw_parser
        --> base_yyparse
```
对`select 2/4`的语法解析部分，比较简单。构造`SelectStmt`节点，只存在target部分，构造`targetList`字段， target为`ResTarget`，在`ResTarget`中构造val字段，为2/4的表达式节点表示`A_Expr`。在运算符的语法解析表示上，gram.y中定义的源码如下：
```sql
simple_select:
		SELECT opt_all_clause opt_target_list
			into_clause from_clause where_clause 
			group_clause having_clause window_clause
		{
			SelectStmt *n = makeNode(SelectStmt);
				n->targetList = $4;
                        -- ...
		}
/*****************************************************************************
 *
 *	target list for SELECT
 *
 *****************************************************************************/
opt_target_list: target_list		{ $$ = $1; }
			| /* EMPTY */	{ $$ = NIL; }
		;

target_list:
		target_el	{ $$ = list_make1($1); }
		| target_list ',' target_el	{ $$ = lappend($1, $3); }
		;
target_el:	
		a_expr
		{
		        $$ = makeNode(ResTarget);
			$$->name = NULL;
			$$->indirection = NIL;
			$$->val = (Node *)$1;
			$$->location = @1;
		}
a_expr:		c_expr	{ $$ = $1; }
		/*
		 * These operators must be called out explicitly in order to make use
		 * of bison's automatic operator-precedence handling.  All other
		 * operator names are handled by the generic productions using "Op",
		 * below; and all those operators will have the same precedence.
		 *
		 * If you add more explicitly-known operators, be sure to add them
		 * also to b_expr and to the MathOp list below.
		 */
		| '+' a_expr		%prec UMINUS
			{ $$ = (Node *) makeSimpleA_Expr(AEXPR_OP, "+", NULL, $2, @1); }
		| '-' a_expr		%prec UMINUS
			{ $$ = doNegate($2, @1); }
		| a_expr '+' a_expr
			{ $$ = (Node *) makeSimpleA_Expr(AEXPR_OP, "+", $1, $3, @2); }
		| a_expr '-' a_expr
			{ $$ = (Node *) makeSimpleA_Expr(AEXPR_OP, "-", $1, $3, @2); }
		| a_expr '*' a_expr
			{ $$ = (Node *) makeSimpleA_Expr(AEXPR_OP, "*", $1, $3, @2); }
		| a_expr '/' a_expr
			{ $$ = (Node *) makeSimpleA_Expr(AEXPR_OP, "/", $1, $3, @2); }
```
构造了一个`A_Expr`节点：
```c++
/*
 * makeSimpleA_Expr -
 *		As above, given a simple (unqualified) operator name
 */
A_Expr *makeSimpleA_Expr(A_Expr_Kind kind, char *name, Node *lexpr, Node *rexpr, int location)
{
	A_Expr	   *a = makeNode(A_Expr);

	a->kind = kind;
	a->name = list_make1(makeString((char *) name));
	a->lexpr = lexpr;
	a->rexpr = rexpr;
	a->location = location;
	return a;
}
```
##### 语义分析阶段
```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformSelectStmt     // transforms a Select Statement
                --> transformTargetList /* Turns a list of ResTarget's into a list of TargetEntry's. */
                    --> transformTargetEntry
                        --> transformExpr
                            --> transformAExprOp
                        --> makeTargetEntry
                            --> make_op // 构造操作符表达式Expr节点OpExpr
                                --> oper // search for a binary operator
                                    --> enforce_generic_type_consistency
                                    --> make_fn_arguments
--> pg_plan_queries
    --> standard_planner
        --> subquery_planner
            --> pull_up_subqueries
            --> preprocess_expression
                // 化简常量表达式，如果可以的话，immutable才行
                --> eval_const_expressions
                    --> set_opfuncid
                    --> simplify_function
                        --> expand_function_arguments
                        --> evaluate_function   // try to pre-evaluate a function call
                            --> evaluate_expr   // 预执行一个常量表达式
                                --> CreateExecutorState
                                --> ExecInitExpr
                                    --> ExecInitExprRec
                                        --> ExecInitFunc
                                    --> ExecReadyExpr
                                --> ExecEvalExprSwitchContext // 执行获取一个常量值
                                    --> ExecInterpExpr
                                        --> int4div  // 调用除法实现函数 计算结果
                                --> makeConst // 将执行常量表达式获得的结果转换为Const常量节点返回

            --> grouping_planner
                --> query_planner
                    --> build_simple_rel
                        --> create_empty_pathtarget // 构造PathTarget
                        --> create_group_result_path  // 生成GroupResultPath
                --> apply_scanjoin_target_to_paths
                    --> create_projection_path
        --> create_plan
            --> create_projection_plan
                --> create_group_result_plan
--> PortalStart
    --> ExecutorStart
        --> InitPlan
            --> ExecInitResult
--> PortalRun
    --> ExecutorRun
        --> standard_ExecutorRun
            --> ExecutePlan
                --> ExecResult
                    --> ExecProject
--> PortalDrop
```

在这个阶段，会找到`/`除法操作符的oid，即查系统表pg_operaotr，在函数`oper`中，我们下面查看一下系统表，查询一下除法`/`操作符，这个操作符你一定意义上可以认为是函数，除法函数。需要输入参数，输出结果。
> [PG文档：10.2. 操作符](http://postgres.cn/docs/13/typeconv-oper.html)
```sql
postgres=# select * from pg_operator where oprname = '/';
 oid  | oprname | oprnamespace | oprowner | oprkind | oprcanmerge | oprcanhash | oprleft | oprright | oprresult | oprcom | oprnegate |    oprcode    | oprrest | oprjoin 
------+---------+--------------+----------+---------+-------------+------------+---------+----------+-----------+--------+-----------+---------------+---------+---------
  527 | /       |           11 |       10 | b       | f           | f          |      21 |       21 |        21 |      0 |         0 | int2div       | -       | -
  528 | /       |           11 |       10 | b       | f           | f          |      23 |       23 |        23 |      0 |         0 | int4div       | -       | -
  546 | /       |           11 |       10 | b       | f           | f          |      21 |       23 |        23 |      0 |         0 | int24div      | -       | -
  547 | /       |           11 |       10 | b       | f           | f          |      23 |       21 |        23 |      0 |         0 | int42div      | -       | -
  588 | /       |           11 |       10 | b       | f           | f          |     700 |      700 |       700 |      0 |         0 | float4div     | -       | -
  593 | /       |           11 |       10 | b       | f           | f          |     701 |      701 |       701 |      0 |         0 | float8div     | -       | -
  687 | /       |           11 |       10 | b       | f           | f          |      20 |       20 |        20 |      0 |         0 | int8div       | -       | -
  691 | /       |           11 |       10 | b       | f           | f          |      20 |       23 |        20 |      0 |         0 | int84div      | -       | -
  695 | /       |           11 |       10 | b       | f           | f          |      23 |       20 |        20 |      0 |         0 | int48div      | -       | -
  821 | /       |           11 |       10 | b       | f           | f          |      20 |       21 |        20 |      0 |         0 | int82div      | -       | -
  825 | /       |           11 |       10 | b       | f           | f          |      21 |       20 |        20 |      0 |         0 | int28div      | -       | -
  734 | /       |           11 |       10 | b       | f           | f          |     600 |      600 |       600 |      0 |         0 | point_div     | -       | -
  739 | /       |           11 |       10 | b       | f           | f          |     602 |      600 |       602 |      0 |         0 | path_div_pt   | -       | -
  807 | /       |           11 |       10 | b       | f           | f          |     603 |      600 |       603 |      0 |         0 | box_div       | -       | -
  844 | /       |           11 |       10 | b       | f           | f          |     790 |      700 |       790 |      0 |         0 | cash_div_flt4 | -       | -
  909 | /       |           11 |       10 | b       | f           | f          |     790 |      701 |       790 |      0 |         0 | cash_div_flt8 | -       | -
 3347 | /       |           11 |       10 | b       | f           | f          |     790 |       20 |       790 |      0 |         0 | cash_div_int8 | -       | -
  913 | /       |           11 |       10 | b       | f           | f          |     790 |       23 |       790 |      0 |         0 | cash_div_int4 | -       | -
  915 | /       |           11 |       10 | b       | f           | f          |     790 |       21 |       790 |      0 |         0 | cash_div_int2 | -       | -
 3825 | /       |           11 |       10 | b       | f           | f          |     790 |      790 |       701 |      0 |         0 | cash_div_cash | -       | -
 1118 | /       |           11 |       10 | b       | f           | f          |     700 |      701 |       701 |      0 |         0 | float48div    | -       | -
 1128 | /       |           11 |       10 | b       | f           | f          |     701 |      700 |       701 |      0 |         0 | float84div    | -       | -
 1519 | /       |           11 |       10 | b       | f           | f          |     718 |      600 |       718 |      0 |         0 | circle_div_pt | -       | -
 1585 | /       |           11 |       10 | b       | f           | f          |    1186 |      701 |      1186 |      0 |         0 | interval_div  | -       | -
 1761 | /       |           11 |       10 | b       | f           | f          |    1700 |     1700 |      1700 |      0 |         0 | numeric_div   | -       | -
(25 rows)

```
可以查到，`select 2/4;`对应的是oid=528,实现该操作符的函数是`int4div`，oprcode表示实现该操作符的函数，可通过查看系统表pg_proc查看该函数的详细信息：
```sql
postgres@postgres=# select * from pg_proc where proname='int4div';
-[ RECORD 1 ]---+--------
oid             | 154
proname         | int4div              -- 函数名
pronamespace    | 11
proowner        | 10
prolang         | 12
procost         | 1
prorows         | 0
provariadic     | 0
prosupport      | -
prokind         | f
prosecdef       | f
proleakproof    | f
proisstrict     | t
proretset       | f
provolatile     | i
proparallel     | s
pronargs        | 2
pronargdefaults | 0
prorettype      | 23                    -- 返回参数
proargtypes     | 23 23                 -- 入参
proallargtypes  | 
proargmodes     | 
proargnames     | 
proargdefaults  | 
protrftypes     | 
prosrc          | int4div               -- 函数source 
probin          | 
proconfig       | 
proacl          | 
```
其中除法表达式`A_Expr`经过语义分析后，通过`OpExpr`进行表示：
```c++
/*
 * OpExpr - expression node for an operator invocation
 *
 * Semantically, this is essentially the same as a function call.
 * 语义上，等同于一个函数调用
 * Note that opfuncid is not necessarily filled in immediately on creation
 * of the node.  The planner makes sure it is valid before passing the node
 * tree to the executor, but during parsing/planning opfuncid can be 0.
 */
typedef struct OpExpr
{
	Expr	xpr;
	Oid	opno;		/* PG_OPERATOR OID of the operator */   //对应pg_operator.oid
	Oid	opfuncid;	/* PG_PROC OID of underlying function */ // 实现该操作符的函数oid
	Oid	opresulttype;	/* PG_TYPE OID of result value */   // 返回值类型
	bool	opretset;	/* true if operator returns set */
	Oid	opcollid;	/* OID of collation of result */
	Oid	inputcollid;	/* OID of collation that operator should use */
	List	*args;		/* arguments to the operator (1 or 2) */  // 参数
	int	location;	/* token location, or -1 if unknown */
} OpExpr;

```

除法的实现函数：
```c++
Datum int4div(PG_FUNCTION_ARGS)
{
	int32		arg1 = PG_GETARG_INT32(0);
	int32		arg2 = PG_GETARG_INT32(1);
	int32		result;

	if (arg2 == 0)
	{
		ereport(ERROR,(errcode(ERRCODE_DIVISION_BY_ZERO),errmsg("division by zero")));
		/* ensure compiler realizes we mustn't reach the division (gcc bug) */
		PG_RETURN_NULL();
	}

	/*
	 * INT_MIN / -1 is problematic, since the result can't be represented on a
	 * two's-complement machine.  Some machines produce INT_MIN, some produce
	 * zero, some throw an exception.  We can dodge the problem by recognizing
	 * that division by -1 is the same as negation.
	 */
	if (arg2 == -1)
	{
		if (unlikely(arg1 == PG_INT32_MIN))
			ereport(ERROR,(errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE), errmsg("integer out of range")));
		result = -arg1;
		PG_RETURN_INT32(result);
	}

	/* No overflow is possible */

	result = arg1 / arg2;

	PG_RETURN_INT32(result);
}
```

从上面的分析过程来看，其实运算符操作与调用一个函数是差不多的。基本可以认为是等同。