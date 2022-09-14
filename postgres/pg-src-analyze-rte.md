
这里单独分析一下Postgres中的范围表`RangeTblEntry`。当我们需要到数据库中查询某些数据时，一般需要通过查一张表，然后返回表中的数据，比如`select * from t1;`，这里的表t1就可以理解为范围表。除了可以从表中返回数据，我们还可以是从子查询，函数，Values等返回查询数据。例如：
```sql
select * from t1;  -- RangeVar
select * from generate_series(2,4);  -- RangeFunction
select * from (values(1,1),(2,2)) as a;  -- RangeValues
select * from (select * from t1);   -- RangeSubquery  ，后面会被优化
-- ......
```
具体的我们可以看一下Postgres源码中范围表类型的定义，有如下几种类型：
```c++
typedef enum RTEKind
{
	RTE_RELATION,				/* ordinary relation reference */
	RTE_SUBQUERY,				/* subquery in FROM */
	RTE_JOIN,					/* join */
	RTE_FUNCTION,				/* function in FROM */
	RTE_TABLEFUNC,				/* TableFunc(.., column list) */
	RTE_VALUES,					/* VALUES (<exprlist>), (<exprlist>), ... */
	RTE_CTE,					/* common table expr (WITH list element) */
	RTE_NAMEDTUPLESTORE,		/* tuplestore, e.g. for AFTER triggers */
	RTE_RESULT					/* RTE represents an empty FROM clause; such
								 * RTEs are added by the planner, they're not
								 * present during parsing or rewriting */
} RTEKind;
```

#### 语法解析层面表示
在语法解析层面，查询语句的表示为`SelectStmt`，经过语义分析后转为查询树`Query`。

```c++
simple_select:
			SELECT opt_all_clause opt_target_list
			into_clause from_clause where_clause
			group_clause having_clause window_clause
				{
					SelectStmt *n = makeNode(SelectStmt);
					n->targetList = $3;
					n->intoClause = $4;
					n->fromClause = $5;
					n->whereClause = $6;
					n->groupClause = ($7)->list;
					n->havingClause = $8;
					n->windowClause = $9;
					$$ = (Node *)n;
				}
from_clause:
			FROM from_list							{ $$ = $2; }
			| /*EMPTY*/								{ $$ = NIL; }
		;

from_list:
			table_ref								{ $$ = list_make1($1); }
			| from_list ',' table_ref				{ $$ = lappend($1, $3); }
		;

/*
 * table_ref is where an alias clause can be attached.
 */
table_ref:	relation_expr opt_alias_clause
				{
					$1->alias = $2;
					$$ = (Node *) $1;
				}
			| relation_expr opt_alias_clause tablesample_clause
				{
					RangeTableSample *n = (RangeTableSample *) $3;
					$1->alias = $2;
					/* relation_expr goes inside the RangeTableSample node */
					n->relation = (Node *) $1;
					$$ = (Node *) n;
				}
			| func_table func_alias_clause
				{
					RangeFunction *n = (RangeFunction *) $1;
					n->alias = linitial($2);
					n->coldeflist = lsecond($2);
					$$ = (Node *) n;
				}
			| select_with_parens opt_alias_clause
				{
					RangeSubselect *n = makeNode(RangeSubselect);
					n->lateral = false;
					n->subquery = $1;
					n->alias = $2;

					$$ = (Node *) n;
				}
			| joined_table
				{
					$$ = (Node *) $1;
				}
			| '(' joined_table ')' alias_clause
				{
					$2->alias = $4;
					$$ = (Node *) $2;
				}
		;
```
也就是说我们查询的时候，可以是从表、子查询、函数、Values等返回待查询的数据。对应的抽象语法树结构定义如下：
- RangeVar      —— 表示表，主要是表名，模式名
- RangeSubselect  —— 子查询，  主要有一个query的字段保存子查询的SelectStmt
- RangeFunction —— 函数，要保存对应的FuncCall，函数调用
- RangeTableFunc
- RangeTableSample
- JoinExpr
这里我们并不把所有的都列出，仅列出如下比较有典型意义的：
```c++
/*
 * RangeVar - range variable, used in FROM clauses
 *
 * Also used to represent table names in utility statements; there, the alias
 * field is not used, and inh tells whether to apply the operation
 * recursively to child tables.  In some contexts it is also useful to carry
 * a TEMP table indication here.
 */
typedef struct RangeVar
{
	NodeTag		type;
	char	   *catalogname;	/* the catalog (database) name, or NULL */
	char	   *schemaname;		/* the schema name, or NULL */
	char	   *relname;		/* the relation/sequence name */
	bool		inh;			/* expand rel by inheritance? recursively act
								 * on children? */
	char		relpersistence; /* see RELPERSISTENCE_* in pg_class.h */
	Alias	   *alias;			/* table alias & optional column aliases */
	int			location;		/* token location, or -1 if unknown */
} RangeVar;

/*
 * RangeFunction - function call appearing in a FROM clause
 *
 * functions is a List because we use this to represent the construct
 * ROWS FROM(func1(...), func2(...), ...).  Each element of this list is a
 * two-element sublist, the first element being the untransformed function
 * call tree, and the second element being a possibly-empty list of ColumnDef
 * nodes representing any columndef list attached to that function within the
 * ROWS FROM() syntax.
 *
 * alias and coldeflist represent any alias and/or columndef list attached
 * at the top level.  (We disallow coldeflist appearing both here and
 * per-function, but that's checked in parse analysis, not by the grammar.)
 */
typedef struct RangeFunction
{
	NodeTag		type;
	bool		lateral;		/* does it have LATERAL prefix? */
	bool		ordinality;		/* does it have WITH ORDINALITY suffix? */
	bool		is_rowsfrom;	/* is result of ROWS FROM() syntax? */
	List	   *functions;		/* per-function information, see above */
	Alias	   *alias;			/* table alias & optional column aliases */
	List	   *coldeflist;		/* list of ColumnDef nodes to describe result
								 * of function returning RECORD */
} RangeFunction;
```

#### 语义分析层面表示
用户输入的SQL字符串，主要通过函数名，表名等表示需要查的“表”，这个表是广义上的表（在from子句的都可以认为是表）。我们前面通过语法解析已经将SQL字符串转成了数据库可以处理的抽象语法树SelectStmt结构体。而数据库内核，其对象的表示，比如表，函数，类型等都是通过OID（整型）去表示的，所以就需要将抽象语法树做进一步的处理，通过查系统表（pg_class、pg_proc等）将对应的名字转换为OID表示，同时进行语义检查语句是否正确，对应的源码就是`pg_analyze->transformStmt`。 具体的，

`FuncCall` -> `FuncExpr`。
```c++
typedef struct FuncCall
{
	NodeTag		type;
	List	   *funcname;	/* qualified name of function */   保存函数名信息
	List	   *args;			/* the arguments (list of exprs) */  保存参数信息
} FuncCall;

/*
 * FuncExpr - expression node for a function call
 */
typedef struct FuncExpr
{
	Expr		xpr;
	Oid			funcid;			/* PG_PROC OID of the function */   函数oid   pg_proc
	Oid			funcresulttype;    /* PG_TYPE OID of result value */    函数返回类型oid pg_type
	bool		funcretset;		    /* true if function returns set */
	bool		funcvariadic;	        /* true if variadic arguments have been
								 * combined into an array last argument */
	CoercionForm funcformat;	    /* how to display this function call */
	Oid			funccollid;		/* OID of collation of result */
	Oid			inputcollid;	    /* OID of collation that function should use */
	List	        *args;			        /* arguments to the function */
	int			location;		    
} FuncExpr;
```

`RangeFunction` ——> `RangeTblEntry`
```c++
typedef struct RangeFunction
{
	NodeTag		type;
	bool		lateral;		/* does it have LATERAL prefix? */
	bool		ordinality;		/* does it have WITH ORDINALITY suffix? */
	bool		is_rowsfrom;	/* is result of ROWS FROM() syntax? */
	List	   *functions;		/* per-function information, see above */
	Alias	   *alias;			/* table alias & optional column aliases */
	List	   *coldeflist;		/* list of ColumnDef nodes to describe result
								 * of function returning RECORD */
} RangeFunction;

// 范围表的定义： 对不同的类型，都有对应的字段保存相关的信息。
typedef struct RangeTblEntry
{
	NodeTag		type;
	RTEKind		rtekind;		/* see above */

	/* Fields valid for a plain relation RTE (else zero):*/
	Oid			relid;			/* OID of the relation */
	char		relkind;		/* relation kind (see pg_class.relkind) */
	int			rellockmode;	/* lock level that query requires on the rel */
	struct TableSampleClause *tablesample;	/* sampling info, or NULL */

	/* Fields valid for a subquery RTE (else NULL):*/
	Query	   *subquery;		/* the sub-query */     //子查询
	bool		security_barrier;	/* is from security_barrier view? */

	/* Fields valid for a function RTE (else NIL/zero):*/
	List	   *functions;		/* list of RangeTblFunction nodes */   //函数
	bool		funcordinality; /* is this called WITH ORDINALITY? */

	/* Fields valid for a TableFunc RTE (else NULL):*/
	TableFunc  *tablefunc;

	/* Fields valid for a values RTE (else NIL):*/
	List	   *values_lists;	/* list of expression lists */ // Values

	// ......

} RangeTblEntry;
```

`SelectStmt` ——> `Query`
因为`SelectStmt`以及`Query`前面的文章中已经分析过多次，这里就不再列出。到此，我们得到了经过语义分析后的查询树`Query`，后面就可以进行查询优化，生成执行计划，进而进入执行器执行了。对于范围表`RangeTblEntry`，在进入优化阶段，主要的处理就是表扫描路径，对此都要生成一个基表信息`RelOptInfo`，后续再对这个数据结构进行详细分析，放到后续优化器相关章节。

