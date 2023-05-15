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
	createStmt->relation = typevar;    // 可以看到这里将create type转为了 create table， 内部是通过建表实现的复合类型
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

我们继续看一下DefineRelation中的实现：
```c++
DefineRelation()
    --> heap_create_with_catalog()
        --> heap_create()
        --> AddNewRelationType()
            --> TypeCreate()
        --> TypeCreate()
```
下面这段代码要认真研读：
```c++
heap_create_with_catalog()
{
    // 省略代码...
    /* Since defining a relation also defines a complex type, we add a new
	 * system type corresponding to the new relation.  The OID of the type can
	 * be preselected by the caller, but if reltypeid is InvalidOid, we'll
	 * generate a new OID for it.
	 * NOTE: we could get a unique-index failure here, in case someone else is
	 * creating the same type name in parallel but hadn't committed yet when
	 * we checked for a duplicate name above.*/
	new_type_addr = AddNewRelationType(relname, relnamespace, relid, relkind, ownerid, reltypeid, new_array_oid);
	new_type_oid = new_type_addr.objectId;
	if (typaddress)
		*typaddress = new_type_addr;

	/* Now make the array type if wanted. */
	if (OidIsValid(new_array_oid))
	{
		char	   *relarrayname;
		relarrayname = makeArrayTypeName(relname, relnamespace);

		TypeCreate(new_array_oid,	/* force the type's OID to this */
				   relarrayname,	/* Array type name */
				   relnamespace,	/* Same namespace as parent */
				   InvalidOid,	/* Not composite, no relationOid */
				   0,			/* relkind, also N/A here */
				   ownerid,		/* owner's ID */
				   -1,			/* Internal size (varlena) */
				   TYPTYPE_BASE,	/* Not composite - typelem is */
				   TYPCATEGORY_ARRAY,	/* type-category (array) */
				   false,		/* array types are never preferred */
				   DEFAULT_TYPDELIM,	/* default array delimiter */
				   F_ARRAY_IN,	/* array input proc */
				   F_ARRAY_OUT, /* array output proc */
				   F_ARRAY_RECV,	/* array recv (bin) proc */
				   F_ARRAY_SEND,	/* array send (bin) proc */
				   InvalidOid,	/* typmodin procedure - none */
				   InvalidOid,	/* typmodout procedure - none */
				   F_ARRAY_TYPANALYZE,	/* array analyze procedure */
				   new_type_oid,	/* array element type - the rowtype */
				   true,		/* yes, this is an array type */
				   InvalidOid,	/* this has no array type */
				   InvalidOid,	/* domain base type - irrelevant */
				   NULL,		/* default value - none */
				   NULL,		/* default binary representation */
				   false,		/* passed by reference */
				   TYPALIGN_DOUBLE, /* alignment - must be the largest! */
				   TYPSTORAGE_EXTENDED, /* fully TOASTable */
				   -1,			/* typmod */
				   0,			/* array dimensions for typBaseType */
				   false,		/* Type NOT NULL */
				   InvalidOid);

		pfree(relarrayname);
	}
}

/* --------------------------------
 *		AddNewRelationType -
 *
 *		define a composite type corresponding to the new relation
 * --------------------------------
 */
static ObjectAddress AddNewRelationType(const char *typeName, Oid typeNamespace, Oid new_rel_oid, char new_rel_kind, Oid ownerid, Oid new_row_type, Oid new_array_type)
{
	return TypeCreate(new_row_type,	/* optional predetermined OID */
				   typeName,	/* type name */
				   typeNamespace,	/* type namespace */
				   new_rel_oid, /* relation oid */
				   new_rel_kind,	/* relation kind */
				   ownerid,		/* owner's ID */
				   -1,			/* internal size (varlena) */
				   TYPTYPE_COMPOSITE,	/* type-type (composite) */
				   TYPCATEGORY_COMPOSITE,	/* type-category (ditto) */
				   false,		/* composite types are never preferred */
				   DEFAULT_TYPDELIM,	/* default array delimiter */
				   F_RECORD_IN, /* input procedure */
				   F_RECORD_OUT,	/* output procedure */
				   F_RECORD_RECV,	/* receive procedure */
				   F_RECORD_SEND,	/* send procedure */
				   InvalidOid,	/* typmodin procedure - none */
				   InvalidOid,	/* typmodout procedure - none */
				   InvalidOid,	/* analyze procedure - default */
				   InvalidOid,	/* array element type - irrelevant */
				   false,		/* this is not an array type */
				   new_array_type,	/* array type if any */
				   InvalidOid,	/* domain base type - irrelevant */
				   NULL,		/* default value - none */
				   NULL,		/* default binary representation */
				   false,		/* passed by reference */
				   TYPALIGN_DOUBLE, /* alignment - must be the largest! */
				   TYPSTORAGE_EXTENDED, /* fully TOASTable */
				   -1,			/* typmod */
				   0,			/* array dimensions for typBaseType */
				   false,		/* Type NOT NULL */
				   InvalidOid);
}

// 创建类型，向pg_type系统表中插入数据
/* ----------------------------------------------------------------
 *		TypeCreate
 *
 *		This does all the necessary work needed to define a new type.
 *
 *		Returns the ObjectAddress assigned to the new type.
 *		If newTypeOid is zero (the normal case), a new OID is created;
 *		otherwise we use exactly that OID.
 * ---------------------------------------------------------------*/
ObjectAddress TypeCreate(Oid newTypeOid,
		   const char *typeName,
		   Oid typeNamespace,
		   Oid relationOid,		/* only for relation rowtypes */
		   char relationKind,	/* ditto */
		   Oid ownerId,
		   int16 internalSize,
		   char typeType,
		   char typeCategory,
		   bool typePreferred,
		   char typDelim,
		   Oid inputProcedure,
		   Oid outputProcedure,
		   Oid receiveProcedure,
		   Oid sendProcedure,
		   Oid typmodinProcedure,
		   Oid typmodoutProcedure,
		   Oid analyzeProcedure,
		   Oid elementType,
		   bool isImplicitArray,
		   Oid arrayType,
		   Oid baseType,
		   const char *defaultTypeValue,	/* human readable rep */
		   char *defaultTypeBin,	/* cooked rep */
		   bool passedByValue,
		   char alignment,
		   char storage,
		   int32 typeMod,
		   int32 typNDims,		/* Array dimensions for baseType */
		   bool typeNotNull,
		   Oid typeCollation)
{
    // ......
}
```