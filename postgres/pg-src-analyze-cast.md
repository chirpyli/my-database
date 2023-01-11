### PostgreSQL源码分析 —— CAST类型转换

我们分析一下PostgreSQL中显式类型转换是如何实现的，在PostgreSQL中，显式类型转换可以用`CAST(x AS typename)` 等同于 `x::typename`。 这里我们以`select cast (2 as numeric`为例进行分析。

#### 源码分析
在分析类型转换前，我们前面分析过`CREATE CAST`的源码，类型转换实质就是定义个类型转换函数，存储在pg_cast系统表中。
```sql
CREATE CAST (source_type AS target_type)
    WITH FUNCTION function_name [ (argument_type [, ...]) ]
    [ AS ASSIGNMENT | AS IMPLICIT ]
```
在进行类型转换时会调用这个定义的函数。理解了这个我们再继续分析下面的源码。

##### 主流程
```c++
exec_simple_query
--> pg_parse_query   // 语法解析，这块不再做说明，前面的很多文章中已经讲的比较多了
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformSelectStmt
                --> transformTargetList
                    --> transformTargetEntry
                        --> transformTypeCast // 将TypeCast节点转为FuncExpr节点，调用函数为类型转换函数
                            --> typenameTypeIdAndMod // 类型名获取类型oid，访问pg_type系统表
                            --> exprType // 获取输入表达式的类型oid
                            --> coerce_to_target_type  // 类型转换函数
                                --> can_coerce_type  // 判断能否进行类型转换
                                    --> find_coercion_pathway // 查pg_cast系统表，能否被转换
                                --> coerce_type  // 将表达式转换另一个类型
                                    --> find_coercion_pathway
                                    --> getBaseTypeAndTypmod
                                    --> build_coercion_expression // 构造强制转换表达式节点
                                        --> makeFuncExpr // 构造类型转换函数的调用节点
                                --> coerce_type_typmod
--> pg_plan_queries // 生成执行计划
    --> pg_plan_query
        --> standard_planner
            --> subquery_planner
                --> preprocess_expression  // 预处理targetList 的表达式
                    --> eval_const_expressions  // 常量表达式化简
                        --> simplify_function
                            --> evaluate_function
                                --> evaluate_expr
                                    --> ExecEvalExprSwitchContext
                                        --> ExecInterpExpr
                                            --> int4_numeric // 调用最终的类型转换函数
            --> create_plan
```

##### 语法解析方面
这里`select cast( 2 as numeric)`，构造`SelectStmt`，cast为target部分，构造`ResTarget`，再构造`TypeCast`节点。因为这部分前面的文章中分析的比较多了，这里不再叙述。`SelectStmt`->`ResTarget`->`TypeCast` 语法解析树表示。我们重点看一下`TypeCast`的定义：
```c++
/* TypeCast - a CAST expression */
typedef struct TypeCast
{
	NodeTag		type;
	Node	   *arg;			/* the expression being casted */ // 需要被转换的表达式
	TypeName   *typeName;		/* the target type */  // 转换成什么类型
	int			location;		/* token location, or -1 if unknown */
} TypeCast;
```
cast类型转换语法表示如下：
```c++
c_expr:		columnref { $$ = $1; }
			| func_expr
				{ $$ = $1; }
func_expr: 			
            | func_expr_common_subexpr
				{ $$ = $1; }
func_expr_common_subexpr:
			| CAST '(' a_expr AS Typename ')'
				{ $$ = makeTypeCast($3, $5, @1); }
			| EXTRACT '(' extract_list ')'
				{
					$$ = (Node *) makeFuncCall(SystemFuncName("date_part"), $3, @1);
				}
```
构造`TypeCast`节点：
```c++
static Node *makeTypeCast(Node *arg, TypeName *typename, int location)
{
	TypeCast *n = makeNode(TypeCast);
	n->arg = arg;
	n->typeName = typename;
	n->location = location;
	return (Node *) n;
}
```

##### 语义分析阶段
主流程：
```c++
exec_simple_query
--> pg_parse_query   // 语法解析，这块不再做说明，前面的很多文章中已经讲的比较多了
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformSelectStmt
                --> transformTargetList
                    --> transformTargetEntry
                        --> transformTypeCast
``` 
主要是`transformTypeCast`函数实现，前面的`SelectStmt`转换为`Query`， `ResTarget`转换为`TargetEntry`，这里的`TypeCast`转换为
```c++
/*
 * Handle an explicit CAST construct.
 *
 * Transform the argument, look up the type name, and apply any necessary
 * coercion function(s).
 */
static Node *
transformTypeCast(ParseState *pstate, TypeCast *tc)
{
	Node	   *result;
	Node	   *arg = tc->arg;
	Node	   *expr;
	Oid			inputType;
	Oid			targetType;
	int32		targetTypmod;
	int			location;

	/* Look up the type name first */
	typenameTypeIdAndMod(pstate, tc->typeName, &targetType, &targetTypmod);

	/*
	 * Look through any AEXPR_PAREN nodes that may have been inserted thanks
	 * to operator_precedence_warning.  Otherwise, ARRAY[]::foo[] behaves
	 * differently from (ARRAY[])::foo[].
	 */
	while (arg && IsA(arg, A_Expr) &&
		   ((A_Expr *) arg)->kind == AEXPR_PAREN)
		arg = ((A_Expr *) arg)->lexpr;

	/*
	 * If the subject of the typecast is an ARRAY[] construct and the target
	 * type is an array type, we invoke transformArrayExpr() directly so that
	 * we can pass down the type information.  This avoids some cases where
	 * transformArrayExpr() might not infer the correct type.  Otherwise, just
	 * transform the argument normally.
	 */
	if (IsA(arg, A_ArrayExpr))
	{
		Oid			targetBaseType;
		int32		targetBaseTypmod;
		Oid			elementType;

		/*
		 * If target is a domain over array, work with the base array type
		 * here.  Below, we'll cast the array type to the domain.  In the
		 * usual case that the target is not a domain, the remaining steps
		 * will be a no-op.
		 */
		targetBaseTypmod = targetTypmod;
		targetBaseType = getBaseTypeAndTypmod(targetType, &targetBaseTypmod);
		elementType = get_element_type(targetBaseType);
		if (OidIsValid(elementType))
		{
			expr = transformArrayExpr(pstate,
									  (A_ArrayExpr *) arg,
									  targetBaseType,
									  elementType,
									  targetBaseTypmod);
		}
		else
			expr = transformExprRecurse(pstate, arg);
	}
	else
		expr = transformExprRecurse(pstate, arg);

	inputType = exprType(expr);
	if (inputType == InvalidOid)
		return expr;			/* do nothing if NULL input */

	/*
	 * Location of the coercion is preferentially the location of the :: or
	 * CAST symbol, but if there is none then use the location of the type
	 * name (this can happen in TypeName 'string' syntax, for instance).
	 */
	location = tc->location;
	if (location < 0)
		location = tc->typeName->location;

	result = coerce_to_target_type(pstate, expr, inputType,
								   targetType, targetTypmod,
								   COERCION_EXPLICIT,
								   COERCE_EXPLICIT_CAST,
								   location);
	if (result == NULL)
		ereport(ERROR,
				(errcode(ERRCODE_CANNOT_COERCE),
				 errmsg("cannot cast type %s to %s",
						format_type_be(inputType),
						format_type_be(targetType)),
				 parser_coercion_errposition(pstate, location, expr)));

	return result;
}
```

在这里，需要额外将一个系统表pg_cast，存储数据类型转换路径，包括内建的和用户自定义的类型。用户通过`CREATE CAST`创建的类型转换也存储在pg_cast系统表中。

- oid oid  ： 行标识符
- castsource oid (references pg_type.oid) ： 源数据类型的OID
- casttarget oid (references pg_type.oid) ： 目标数据类型的OID
- castfunc oid (references pg_proc.oid) ： 执行该转换的函数的OID。如果该转换方法不需要一个函数则存储0。
- castcontext char ： 指示该转换能被调用的环境。 e表示仅能作为一个显式转换（使用CAST或::语法）。 a表示在赋值给目标列时隐式调用， 和显式调用一样。 i表示在表达式中隐式调用，和其他转换一样。
- castmethod char ： 指示转换如何被执行。 f表明使用castfunc中指定的函数。 i表明使用输入/输出函数。 b表明该类型是二进制可转换的，因此不需要转换。
```sql
-- 查一下 int转numeric  pg_cast中存在类型转换， 查pg_proc,转换函数为numeric
postgres@postgres=# select pg_cast.*,pg_proc.proname,pg_proc.pronamespace from pg_cast,pg_proc where pg_cast.castsource=23 and pg_cast.casttarget=1700 and pg_cast.castfunc = pg_proc.oid;
  oid  | castsource | casttarget | castfunc | castcontext | castmethod | proname | pronamespace 
-------+------------+------------+----------+-------------+------------+---------+--------------
 11337 |         23 |       1700 |     1740 | i           | f          | numeric |           11
(1 row)
```

##### 执行

在逻辑优化阶段，因为复合常量表达式化简，在这个阶段就调用了类型转换函数，返回常量值，到执行器时，仅仅是将常量值返回。
最终会调用`int4_numeric`转换函数完成类型转换。
```c++
--> pg_plan_queries // 生成执行计划
    --> pg_plan_query
        --> standard_planner
            --> subquery_planner
                --> preprocess_expression  // 预处理targetList 的表达式
                    --> eval_const_expressions  // 常量表达式化简
                        --> simplify_function
                            --> evaluate_function
                                --> evaluate_expr
                                    --> ExecEvalExprSwitchContext
                                        --> ExecInterpExpr
                                            --> int4_numeric // 调用最终的类型转换函数
            --> create_plan
```
```c++
/* ----------------------------------------------------------------------
 *
 * Type conversion functions
 *
 * ----------------------------------------------------------------------*/
Datum int4_numeric(PG_FUNCTION_ARGS)
{
	int32		val = PG_GETARG_INT32(0);
	Numeric		res;
	NumericVar	result;

	init_var(&result);

	int64_to_numericvar((int64) val, &result);

	res = make_result(&result);

	free_var(&result);

	PG_RETURN_NUMERIC(res);
}
```

其他的类型转换与之类似，只是部分细节不同。