### 逻辑优化——常量表达式化简
常量表达式可以进行化简，可降低执行器计算表达式的代价。在逻辑优化阶段，会判断是否可以进行常量表达式化简，如果可以，则在执行器执行之前就预先对常量表达式树进行计算，计算出常量后，以新计算出的常量表达式代替原有的表达式。当进入执行器时，此时表达式已被替换为常量，避免了在执行器中频繁的计算表达式。

#### 化简示例
我们以下面的SQL为例，条件表达式为`a > 1 + 1`，是常量表达式，可以进行化简。
```sql
-- 常量表达式可以进行化简
postgres=# explain select * from t1 where a >  1 + 1;   
                     QUERY PLAN                      
-----------------------------------------------------
 Seq Scan on t1  (cost=0.00..17.50 rows=998 width=8)
   Filter: (a > 2)   -- 进入执行器中， 表达式已被提前替换为常量，每扫描一个元组，不用再进行1+1的表达式计算了
(2 rows)
-- 不满足常量表达式化简的条件
postgres=# explain select * from t1 where a >  1 + random();  
                               QUERY PLAN                               
------------------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..25.00 rows=333 width=8)
   Filter: ((a)::double precision > ('1'::double precision + random()))     --  执行器中每扫描一个元组，都要进行一次表达式计算
(2 rows)
```
除了常量表达式，常量函数也可以在逻辑优化阶段提前执行，计算得到常量，用常量替换原有的函数表达式。

#### 源码分析
我们以`select * from t1 where a = 1 + 1;`这条语句为例，分析一下在PostgreSQL中是如何进行化简的。主流程如下
```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
    --> transformStmt
        --> transformSelectStmt
            --> transformWhereClause
                --> transformExpr
--> pg_plan_queries
    --> pg_plan_query
        --> planner
            --> standard_planner
                --> subquery_planner
                    --> preprocess_qual_conditions
                        --> preprocess_expression
                            --> eval_const_expressions  // 在这里完成常量表达式化简，完成1+1表达式替换为常量2
                                --> eval_const_expressions_mutator
                                    --> simplify_function
                                        --> evaluate_function
                                            --> evaluate_expr  // 1+1的表达式被计算为常量2
                --> create_plan
--> PortalStart
--> PortalRun  // 执行器执行
--> PortalDrop
```
可以看到，在逻辑优化阶段，常量表达式已被化简为常量，进入执行器时，表达式树已被替换为常量。下面为详细实现。
```c++
/*
 * preprocess_qual_conditions
 *		Recursively scan the query's jointree and do subquery_planner's
 *		preprocessing work on each qual condition found therein.
 */
static void preprocess_qual_conditions(PlannerInfo *root, Node *jtnode)
{
	if (jtnode == NULL)
		return;
	if (IsA(jtnode, RangeTblRef))
	{
		/* nothing to do here */
	}
	else if (IsA(jtnode, FromExpr))
	{
		FromExpr   *f = (FromExpr *) jtnode;
		ListCell   *l;

		foreach(l, f->fromlist)
			preprocess_qual_conditions(root, lfirst(l));

		f->quals = preprocess_expression(root, f->quals, EXPRKIND_QUAL);
	}
	else if (IsA(jtnode, JoinExpr))
	{
        // ...
	}
	else
		elog(ERROR, "unrecognized node type: %d",
			 (int) nodeTag(jtnode));
}

/*
 * preprocess_expression
 *		Do subquery_planner's preprocessing work for an expression,
 *		which can be a targetlist, a WHERE clause (including JOIN/ON
 *		conditions), a HAVING clause, or a few other things.
 */
static Node *preprocess_expression(PlannerInfo *root, Node *expr, int kind)
{
    // ...

	/*
	 * Simplify constant expressions.  For function RTEs, this was already
	 * done by preprocess_function_rtes.  
	 */
	if (kind != EXPRKIND_RTFUNC)
		expr = eval_const_expressions(root, expr);

	/* If it's a qual or havingQual, canonicalize it. */
	if (kind == EXPRKIND_QUAL)
		expr = (Node *) canonicalize_qual((Expr *) expr, false);

    // ...

	/*
	 * If it's a qual or havingQual, convert it to implicit-AND format. (We
	 * don't want to do this before eval_const_expressions, since the latter
	 * would be unable to simplify a top-level AND correctly. Also,
	 * SS_process_sublinks expects explicit-AND format.)
	 */
	if (kind == EXPRKIND_QUAL)
		expr = (Node *) make_ands_implicit((Expr *) expr);

	return expr;
}

/*--------------------
 * eval_const_expressions
 *
 * Reduce any recognizably constant subexpressions of the given
 * expression tree, for example "2 + 2" => "4".  More interestingly,
 * we can reduce certain boolean expressions even when they contain
 * non-constant subexpressions: "x OR true" => "true" no matter what
 * the subexpression x is.  */
Node *eval_const_expressions(PlannerInfo *root, Node *node)
{
	eval_const_expressions_context context;

	if (root)
		context.boundParams = root->glob->boundParams;	/* bound Params */
	else
		context.boundParams = NULL;
	context.root = root;		/* for inlined-function dependencies */
	context.active_fns = NIL;	/* nothing being recursively simplified */
	context.case_val = NULL;	/* no CASE being examined */
	context.estimate = false;	/* safe transformations only */
	return eval_const_expressions_mutator(node, &context);
}

/* Recursive guts of eval_const_expressions/estimate_expression_value */
static Node *eval_const_expressions_mutator(Node *node,
							   eval_const_expressions_context *context)
{
	if (node == NULL)
		return NULL;
	switch (nodeTag(node))
	{
		case T_FuncExpr:
			{
                // ...
			}
		case T_OpExpr:
			{
				OpExpr	   *expr = (OpExpr *) node;
				List	   *args = expr->args;
				Expr	   *simple;
				OpExpr	   *newexpr;

				/* Need to get OID of underlying function.  Okay to scribble on input to this extent. */
				set_opfuncid(expr);

				/*
				 * Code for op/func reduction is pretty bulky, so split it out
				 * as a separate function. */
				simple = simplify_function(expr->opfuncid,
										   expr->opresulttype, -1,
										   expr->opcollid,
										   expr->inputcollid,
										   &args,
										   false,
										   true,
										   true,
										   context);
				if (simple)		/* successfully simplified it */
					return (Node *) simple;

				/*
				 * If the operator is boolean equality or inequality, we know
				 * how to simplify cases involving one constant and one
				 * non-constant argument.
				 */
				if (expr->opno == BooleanEqualOperator ||
					expr->opno == BooleanNotEqualOperator)
				{
					simple = (Expr *) simplify_boolean_equality(expr->opno,
																args);
					if (simple) /* successfully simplified it */
						return (Node *) simple;
				}

				/*
				 * The expression cannot be simplified any further, so build
				 * and return a replacement OpExpr node using the
				 * possibly-simplified arguments.
				 */
				newexpr = makeNode(OpExpr);
				newexpr->opno = expr->opno;
				newexpr->opfuncid = expr->opfuncid;
				newexpr->opresulttype = expr->opresulttype;
				newexpr->opretset = expr->opretset;
				newexpr->opcollid = expr->opcollid;
				newexpr->inputcollid = expr->inputcollid;
				newexpr->args = args;
				newexpr->location = expr->location;
				return (Node *) newexpr;
			}
        // ...
		default:
			break;
	}

	/*
	 * For any node type not handled above, copy the node unchanged but
	 * const-simplify its subexpressions.  This is the correct thing for node
	 * types whose behavior might change between planning and execution, such
	 * as CurrentOfExpr.  It's also a safe default for new node types not
	 * known to this routine.
	 */
	return ece_generic_processing(node);
}

/*
 * Subroutine for eval_const_expressions: try to simplify a function call
 * (which might originally have been an operator; we don't care)
 */
static Expr *
simplify_function(Oid funcid, Oid result_type, int32 result_typmod,
				  Oid result_collid, Oid input_collid, List **args_p,
				  bool funcvariadic, bool process_args, bool allow_non_const,
				  eval_const_expressions_context *context)
{
    // ...
	/* Now attempt simplification of the function call proper. */
	newexpr = evaluate_function(funcid, result_type, result_typmod,
								result_collid, input_collid,
								args, funcvariadic,
								func_tuple, context);

    // ...

	return newexpr;
}

/*
 * evaluate_function: try to pre-evaluate a function call
 */
static Expr *
evaluate_function(Oid funcid, Oid result_type, int32 result_typmod,
				  Oid result_collid, Oid input_collid, List *args,
				  bool funcvariadic,
				  HeapTuple func_tuple,
				  eval_const_expressions_context *context)
{
    // ...

	return evaluate_expr((Expr *) newexpr, result_type, result_typmod,
						 result_collid);
}

/*
 * evaluate_expr: pre-evaluate a constant expression
 *
 * We use the executor's routine ExecEvalExpr() to avoid duplication of
 * code and ensure we get the same result as the executor would get.
 */
Expr *
evaluate_expr(Expr *expr, Oid result_type, int32 result_typmod,
			  Oid result_collation)
{
	EState	   *estate;
	ExprState  *exprstate;
	MemoryContext oldcontext;
	Datum		const_val;
	bool		const_is_null;
	int16		resultTypLen;
	bool		resultTypByVal;

	/* To use the executor, we need an EState. */
	estate = CreateExecutorState();

	/* We can use the estate's working context to avoid memory leaks. */
	oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);

	/* Make sure any opfuncids are filled in. */
	fix_opfuncids((Node *) expr);

	/*
	 * Prepare expr for execution.  (Note: we can't use ExecPrepareExpr
	 * because it'd result in recursively invoking eval_const_expressions.)
	 */
	exprstate = ExecInitExpr(expr, NULL);

	/*
	 * And evaluate it.
	 *
	 * It is OK to use a default econtext because none of the ExecEvalExpr()
	 * code used in this situation will use econtext.  That might seem
	 * fortuitous, but it's not so unreasonable --- a constant expression does
	 * not depend on context, by definition, n'est ce pas?
	 */
	const_val = ExecEvalExprSwitchContext(exprstate,
										  GetPerTupleExprContext(estate),
										  &const_is_null);  // 计算表达式 1 + 1

	/* Get info needed about result datatype */
	get_typlenbyval(result_type, &resultTypLen, &resultTypByVal);

	/* Get back to outer memory context */
	MemoryContextSwitchTo(oldcontext);

	/*
	 * Must copy result out of sub-context used by expression eval.
	 *
	 * Also, if it's varlena, forcibly detoast it.  This protects us against
	 * storing TOAST pointers into plans that might outlive the referenced
	 * data.  (makeConst would handle detoasting anyway, but it's worth a few
	 * extra lines here so that we can do the copy and detoast in one step.)
	 */
	if (!const_is_null)
	{
		if (resultTypLen == -1)
			const_val = PointerGetDatum(PG_DETOAST_DATUM_COPY(const_val));
		else
			const_val = datumCopy(const_val, resultTypByVal, resultTypLen);
	}

	/* Release all the junk we just created */
	FreeExecutorState(estate);

	/* Make the constant result node. */
	return (Expr *) makeConst(result_type, result_typmod, result_collation,
							  resultTypLen,
							  const_val, const_is_null,
							  resultTypByVal);
}
```

>如果对源码不熟悉，可以补充看一下这里关于解析层的代码，有助于理解PostgreSQL源码中表达式处理相关的逻辑。

#### 补充解析层相关代码

我们看一下在解析层表达式是如何表示的，`1+1`表示为`A_Expr`，类型为`AEXPR_OP`。`where a = 1 + 1`则是`A_Expr`，左子树为变量`a`，右子树为`1+1`的表达式`A_Expr`。
```c++
where_clause:
			WHERE a_expr		{ $$ = $2; }
			| /*EMPTY*/			{ $$ = NULL; }
		;
a_expr:		c_expr			{ $$ = $1; }
			| a_expr '+' a_expr     // 表示 1 + 1
				{ $$ = (Node *) makeSimpleA_Expr(AEXPR_OP, "+", $1, $3, @2); }
			| a_expr '=' a_expr
				{ $$ = (Node *) makeSimpleA_Expr(AEXPR_OP, "=", $1, $3, @2); }
c_expr:		columnref		{ $$ = $1; }
			| AexprConst	{ $$ = $1; }
AexprConst: Iconst
				{
					$$ = makeIntConst($1, @1);
				}
Iconst:		ICONST	{ $$ = $1; };       // 1 

A_Expr *
makeSimpleA_Expr(A_Expr_Kind kind, char *name, Node *lexpr, Node *rexpr, int location)
{
	A_Expr	   *a = makeNode(A_Expr);

	a->kind = kind;
	a->name = list_make1(makeString((char *) name));
	a->lexpr = lexpr;
	a->rexpr = rexpr;
	a->location = location;
	return a;
}

typedef struct A_Expr
{
	NodeTag		type;
	A_Expr_Kind kind;			/* see above */
	List	   *name;			/* possibly-qualified name of operator */
	Node	   *lexpr;			/* left argument, or NULL if none */
	Node	   *rexpr;			/* right argument, or NULL if none */
	int			location;		/* token location, or -1 if unknown */
} A_Expr;

/* A_Const - a literal constant */
typedef struct A_Const
{
	NodeTag		type;
	Value		val;			/* value (includes type info, see value.h) */
	int			location;		/* token location, or -1 if unknown */
} A_Const;
```
