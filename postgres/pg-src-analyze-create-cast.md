### CREATE CAST源码分析

#### CREATE CAST用法
[CREATE CAST](https://www.postgresql.org/docs/13/sql-createcast.html) —— 定义一个用户自定义的类型转换
用法如下：
```sql
CREATE CAST (source_type AS target_type)
    WITH FUNCTION function_name [ (argument_type [, ...]) ]
    [ AS ASSIGNMENT | AS IMPLICIT ]

CREATE CAST (source_type AS target_type)
    WITHOUT FUNCTION
    [ AS ASSIGNMENT | AS IMPLICIT ]

CREATE CAST (source_type AS target_type)
    WITH INOUT
    [ AS ASSIGNMENT | AS IMPLICIT ]
```
如何使用以及用法请参考PostgreSQL文档中[CREATE CAST](https://www.postgresql.org/docs/13/sql-createcast.html)一节。下面我们主要分析一下其源码，看一下是如何实现的。

#### 源码分析
因为`CREATE CAST`属于Utility型语句，无需查询优化，其主流程如下：
```c++
exec_simple_query
--> pg_parse_query      // 语法解析
--> pg_analyze_and_rewrite   // 语义分析
    --> parse_analyze
    --> pg_rewrite_query
--> pg_plan_queries    // 生成执行计划
--> PortalStart
--> PortalRun          // 执行器执行
    --> PortalRunMulti
        --> PortalRunUtility
            --> ProcessUtility
                --> standard_ProcessUtility
                    --> ProcessUtilitySlow
                        --> CreateCast    // 进入CreateCast处理函数处理CREATE CAST语句
--> PortalDrop
```
主要的处理逻辑都在`CreateCast`函数中完成。后续会重点分析一下这个函数。

##### 解析部分
我们首先分析一下其语法解析部分，这部分比较简单，核心是`CreateCastStmt`的定义，定义如下：
```c++
/* ----------------------
 *	CREATE CAST Statement
 * ----------------------
 */
typedef struct CreateCastStmt
{
	NodeTag		type;
	TypeName   *sourcetype;
	TypeName   *targettype;
	ObjectWithArgs *func;
	CoercionContext context;
	bool		inout;
} CreateCastStmt;

```
`CREATE CAST`其在gram.y中定义的表示如下：
```c++
/*************************************************
 *
 *		CREATE CAST / DROP CAST
 *
 *************************************************/
CreateCastStmt: CREATE CAST '(' Typename AS Typename ')'
					WITH FUNCTION function_with_argtypes cast_context
				{
					CreateCastStmt *n = makeNode(CreateCastStmt);
					n->sourcetype = $4;
					n->targettype = $6;
					n->func = $10;
					n->context = (CoercionContext) $11;
					n->inout = false;
					$$ = (Node *)n;
				}
			| CREATE CAST '(' Typename AS Typename ')'
					WITHOUT FUNCTION cast_context
				{
					CreateCastStmt *n = makeNode(CreateCastStmt);
					n->sourcetype = $4;
					n->targettype = $6;
					n->func = NULL;
					n->context = (CoercionContext) $10;
					n->inout = false;
					$$ = (Node *)n;
				}
			| CREATE CAST '(' Typename AS Typename ')'
					WITH INOUT cast_context
				{
					CreateCastStmt *n = makeNode(CreateCastStmt);
					n->sourcetype = $4;
					n->targettype = $6;
					n->func = NULL;
					n->context = (CoercionContext) $10;
					n->inout = true;
					$$ = (Node *)n;
				}
		;

cast_context:  AS IMPLICIT_P					{ $$ = COERCION_IMPLICIT; }
		| AS ASSIGNMENT							{ $$ = COERCION_ASSIGNMENT; }
		| /*EMPTY*/								{ $$ = COERCION_EXPLICIT; }
		;


DropCastStmt: DROP CAST opt_if_exists '(' Typename AS Typename ')' opt_drop_behavior
				{
					DropStmt *n = makeNode(DropStmt);
					n->removeType = OBJECT_CAST;
					n->objects = list_make1(list_make2($5, $7));
					n->behavior = $9;
					n->missing_ok = $3;
					n->concurrent = false;
					$$ = (Node *)n;
				}
		;

opt_if_exists: IF_P EXISTS						{ $$ = true; }
		| /*EMPTY*/								{ $$ = false; }
		;
```
非常容易理解，下面我们重点分析一下`CreateCast`函数的实现。

##### 执行部分
用户通过`CREATE CAST`语句自定义一个类型转换，数据库肯定有个地方将这个转换的信息存起来，这个地方就是pg_cast系统表。pg_cast系统表存在数据类型转换路径，包括内建和用户自定义的。
```sql
postgres@postgres=# \d pg_cast
              Table "pg_catalog.pg_cast"
   Column    |  Type  | Collation | Nullable | Default 
-------------+--------+-----------+----------+---------
 oid         | oid    |           | not null | 
 castsource  | oid    |           | not null |             -- 源数据类型的OID
 casttarget  | oid    |           | not null |             -- 目标数据类型的OID
 castfunc    | oid    |           | not null |             -- 执行该转换的函数的OID。如果该转换方法不需要一个函数则存储0。
 castcontext | "char" |           | not null |             -- 指示该转换能被调用的环境
 castmethod  | "char" |           | not null |             -- 指示转换如何被执行。
Indexes:
    "pg_cast_oid_index" UNIQUE, btree (oid)
    "pg_cast_source_target_index" UNIQUE, btree (castsource, casttarget)
```

而`CreateCast`函数的主要内容就是将用户自定义的类型转换信息插入到pg_cast系统表中。
```c++
CreateCast
--> LookupFuncWithArgs  // 查到pg_proc系统表，看是否已存在
    --> LookupFuncNameInternal
        --> FuncnameGetCandidates
--> IsBinaryCoercible(Oid srctype, Oid targettype) //Check if srctype is binary-coercible to targettype.
--> CastCreate  // 将类型转换信息插入pg_cast系统表中
```

