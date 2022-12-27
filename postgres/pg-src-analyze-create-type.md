### create type 源码分析
PG中可以通过`CREATE TYPE`创建复合类型，我们分析一下其源码实现。

```sql
create type mytyp as (a int, b int);
```


#### 语法解析
```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
        --> base_yyparse
```
生成`CompositeTypeStmt`。定义如下：
```c++
/* ----------------------
 *		Create Type Statement, composite types
 * ----------------------
 */
typedef struct CompositeTypeStmt
{
	NodeTag		type;
	RangeVar   *typevar;		/* the composite type to be created */
	List	   *coldeflist;		/* list of ColumnDef nodes */
} CompositeTypeStmt;
```
gram.y中语法定义如下：
```c++
DefineStmt:
			CREATE TYPE_P any_name AS '(' OptTableFuncElementList ')'
				{
					CompositeTypeStmt *n = makeNode(CompositeTypeStmt);

					/* can't use qualified_name, sigh */
					n->typevar = makeRangeVarFromAnyName($3, @3, yyscanner);
					n->coldeflist = $6;
					$$ = (Node *)n;
				}
			| CREATE TYPE_P any_name AS ENUM_P '(' opt_enum_val_list ')'
				{
					CreateEnumStmt *n = makeNode(CreateEnumStmt);
					n->typeName = $3;
					n->vals = $7;
					$$ = (Node *)n;
				}

TableFuncElementList:
			TableFuncElement
				{
					$$ = list_make1($1);
				}
			| TableFuncElementList ',' TableFuncElement
				{
					$$ = lappend($1, $3);
				}
		;

TableFuncElement:	ColId Typename opt_collate_clause
				{
					ColumnDef *n = makeNode(ColumnDef);
					n->colname = $1;				
					n->typeName = $2;
					n->inhcount = 0;
					n->is_local = true;
					n->is_not_null = false;
					n->is_from_type = false;
					n->storage = 0;
					n->raw_default = NULL;
					n->cooked_default = NULL;
					n->collClause = (CollateClause *) $3;
					n->collOid = InvalidOid;
					n->constraints = NIL;
					n->location = @1;
					$$ = (Node *)n;
				}
		;
```

#### 语义分析
这部分因为是Utility语句，没有什么处理的。
```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
    --> parse_analyze
	    --> transformStmt
```
关键代码：
```c++
/*
 * transformStmt -  recursively transform a Parse tree into a Query tree.
 */
Query *transformStmt(ParseState *pstate, Node *parseTree)
{
	Query	   *result;
	switch (nodeTag(parseTree))
	{
		case T_InsertStmt:
			result = transformInsertStmt(pstate, (InsertStmt *) parseTree);
			break;
		case T_DeleteStmt:
			result = transformDeleteStmt(pstate, (DeleteStmt *) parseTree);
			break;
		case T_UpdateStmt:
			result = transformUpdateStmt(pstate, (UpdateStmt *) parseTree);
			break;
		case T_SelectStmt:
			{
				SelectStmt *n = (SelectStmt *) parseTree;

				if (n->valuesLists)
					result = transformValuesClause(pstate, n);
				else if (n->op == SETOP_NONE)
					result = transformSelectStmt(pstate, n);
				else
					result = transformSetOperationStmt(pstate, n);
			}
			break;

			/*
			 * Special cases
			 */
		case T_DeclareCursorStmt:
			result = transformDeclareCursorStmt(pstate,
												(DeclareCursorStmt *) parseTree);
			break;

		case T_ExplainStmt:
			result = transformExplainStmt(pstate,
										  (ExplainStmt *) parseTree);
			break;

		case T_CreateTableAsStmt:
			result = transformCreateTableAsStmt(pstate,
												(CreateTableAsStmt *) parseTree);
			break;

		case T_CallStmt:
			result = transformCallStmt(pstate,
									   (CallStmt *) parseTree);
			break;
		default:
			/*
			 * other statements don't require any transformation; just return
			 * the original parsetree with a Query node plastered on top.
			 */
			result = makeNode(Query);
			result->commandType = CMD_UTILITY;
			result->utilityStmt = (Node *) parseTree;   // 构造查询树，直接将抽象语法树赋值到utilityStmt字段中
			break;
	}

	result->querySource = QSRC_ORIGINAL;
	result->canSetTag = true;
		
	return result;
}
```

#### 执行计划生成
这部分因为不是可优化语句，没有什么处理。直接构造执行计划。
```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
    --> parse_analyze
	    --> transformStmt
--> pg_plan_queries
```
关键代码：
```c++
/*
 * Generate plans for a list of already-rewritten queries.
 *
 * For normal optimizable statements, invoke the planner.  For utility
 * statements, just make a wrapper PlannedStmt node.
 *
 * The result is a list of PlannedStmt nodes.
 */
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
			stmt->commandType = CMD_UTILITY;
			stmt->canSetTag = query->canSetTag;
			stmt->utilityStmt = query->utilityStmt;  // 直接构造执行计划树
			stmt->stmt_location = query->stmt_location;
			stmt->stmt_len = query->stmt_len;
		} else
			stmt = pg_plan_query(query, query_string, cursorOptions, boundParams);

		stmt_list = lappend(stmt_list, stmt);
	}

	return stmt_list;
}
```

#### 执行器
其实，其流程与`create table`十分类似。定义一个复合类型某种程度上就是定义一张新的表，表的rowtype就是复合类型。会看到创建一个复合类型的同时会创建一个同名的表。这个表可以在pg_class系统表中查到，但是通过\d等却查不到，也不能想表中插入数据，因为它并不是真正意义上的表，只是用表实现复合类型。

```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
    --> parse_analyze
	    --> transformStmt
--> pg_plan_queries

--> PortalStart
--> PortalRun
    --> standard_ProcessUtility
	    --> ProcessUtilitySlow
		    --> DefineCompositeType
```

关键代码：
```c++
/*-------------------------------------------------------------------
 * DefineCompositeType
 *
 * Create a Composite Type relation.
 * `DefineRelation' does all the work, we just provide the correct arguments!
 *
 * If the relation already exists, then 'DefineRelation' will abort the xact...
 *
 * Return type is the new type's object address.
 *-------------------------------------------------------------------*/
ObjectAddress DefineCompositeType(RangeVar *typevar, List *coldeflist)
{
	CreateStmt *createStmt = makeNode(CreateStmt);
	Oid			old_type_oid;
	Oid			typeNamespace;
	ObjectAddress address;

	/* now set the parameters for keys/inheritance etc. All of these are uninteresting for composite types... */
	createStmt->relation = typevar;
	createStmt->tableElts = coldeflist;
	createStmt->inhRelations = NIL;
	createStmt->constraints = NIL;
	createStmt->options = NIL;
	createStmt->oncommit = ONCOMMIT_NOOP;
	createStmt->tablespacename = NULL;
	createStmt->if_not_exists = false;

	/*
	 * Check for collision with an existing type name. If there is one and
	 * it's an autogenerated array, we can rename it out of the way.  This
	 * check is here mainly to get a better error message about a "type"
	 * instead of below about a "relation".
	 */
	typeNamespace = RangeVarGetAndCheckCreationNamespace(createStmt->relation, NoLock, NULL);
	RangeVarAdjustRelationPersistence(createStmt->relation, typeNamespace);
	old_type_oid = GetSysCacheOid2(TYPENAMENSP, Anum_pg_type_oid,
						CStringGetDatum(createStmt->relation->relname),
						ObjectIdGetDatum(typeNamespace));
	if (OidIsValid(old_type_oid))
	{
		if (!moveArrayTypeName(old_type_oid, createStmt->relation->relname, typeNamespace))
			ereport(ERROR, (errcode(ERRCODE_DUPLICATE_OBJECT),
					 errmsg("type \"%s\" already exists", createStmt->relation->relname)));
	}

	/* Finally create the relation.  This also creates the type. */
	DefineRelation(createStmt, RELKIND_COMPOSITE_TYPE, InvalidOid, &address, NULL);  // 通过创建表，创建新的复合类型，但不是用户能使用的表

	return address;
}
```