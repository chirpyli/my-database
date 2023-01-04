### PostgreSQL源码分析——建表含有序列
#### 建表时含有序列
之前分析过`CREATE TABLE`语句的创建过程，这里，分析一下当建表中的列有serial时是如何处理的。以下面的例句为例：
```sql
-- 建表，含有序列
create table t1(a int, b serial);

-- 可以看到建表时，同时创建了一个序列t1_b_seq
postgres@postgres=# \d
                      List of relations
   Schema   |       Name       |       Type        |  Owner   
------------+------------------+-------------------+----------
 public     | t1               | table             | postgres
 public     | t1_b_seq         | sequence          | postgres
-- 查看这个序列，owned by t1, 当删除t1时，会同步删除t1_b_seq
postgres@postgres=# \d t1_b_seq 
                      Sequence "public.t1_b_seq"
  Type   | Start | Minimum |  Maximum   | Increment | Cycles? | Cache 
---------+-------+---------+------------+-----------+---------+-------
 integer |     1 |       1 | 2147483647 |         1 | no      |     1
Owned by: public.t1.b
-- 查看表t1, b列的默认值为序列t1_b_seq的nextval
postgres@postgres=# \d t1
                            Table "public.t1"
 Column |  Type   | Collation | Nullable |            Default            
--------+---------+-----------+----------+-------------------------------
 a      | integer |           |          | 
 b      | integer |           | not null | nextval('t1_b_seq'::regclass)
-- 查看系统表pg_class
postgres@postgres=# select oid,relname,relkind from pg_class where relname='t1_b_seq';
  oid  | relname  | relkind 
-------+----------+---------
 25813 | t1_b_seq | S       -- 可以看到表中类型为序列
(1 row)
-- 查看系统表pg_sequence 可查询序列信息
postgres@postgres=# select * from pg_sequence where seqrelid = 25813;
 seqrelid | seqtypid | seqstart | seqincrement |   seqmax   | seqmin | seqcache | seqcycle 
----------+----------+----------+--------------+------------+--------+----------+----------
    25813 |       23 |        1 |            1 | 2147483647 |      1 |        1 | f
(1 row)


```
从上面的结果来看，我们大致就能清楚其创建过程了，先创建序列t1_b_seq，再创建表t1，设置t1中b列的默认值为序列的下一个值。下面我们就跟踪源码，看一下是否和上面分析的一样。

#### 源码分析
因前面文章中已分析过建表的解析过程，这里直接上主流程，
```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
--> pg_analyze_and_rewrite_fixedparams
    --> parse_analyze_fixedparams
        --> transformStmt
    --> pg_rewrite_query
--> pg_plan_queries
--> PortalStart
--> PortalRun
    --> standard_ProcessUtility
        --> ProcessUtilitySlow
            --> case T_CreateStmt:
              // 建表流程
              --> transformCreateStmt // 处理CreateStmt，处理列，约束等
                  --> transformColumnDefinition // 处理列
                      // 如果该列是序列，则
                      // 1. 创建CreateSeqStmt节点
                      // 2. 创建Constraint约束，CONSTR_DEFAULT, 默认值为序列的nextval
                      // 3. 创建Constraint约束，CONSTR_NOTNULL
                      --> generateSerialExtraStmts
                          --> ChooseRelationName // 构造序列名: tablename_columnname_seq
                  --> transformTableConstraint  // 处理约束
              --> DefineRelation
            --> case T_CreateSeqStmt:
              // 创建序列
              --> DefineSeqence  // Creates a new sequence relation
                  --> init_params // 处理序列的相关设置
                  --> DefineRelation
                  --> 向pg_sequence系统表中插入序列信息
               
--> PortalDrop
```

需要单独分析一下函数`ProcessUtilitySlow`，在这个函数中，负责处理建表，建序列的逻辑。在处理`CreateStmt`时，会新创建一个`CreateSeqStmt`节点，用于创建序列，再创建表。
```c++
typedef struct CreateSeqStmt
{
	NodeTag		type;
	RangeVar   *sequence;		/* the sequence to create */ // 序列名
	List	   *options;
	Oid			ownerId;		/* ID of owner, or InvalidOid for default */ // 序列属于谁
	bool		for_identity;
	bool		if_not_exists;	/* just do nothing if it already exists? */
} CreateSeqStmt;
```
建表过程流程如下：
```c++
/*
 * The "Slow" variant of ProcessUtility should only receive statements
 * supported by the event triggers facility.  Therefore, we always
 * perform the trigger support calls if the context allows it.
 */
static void
ProcessUtilitySlow(ParseState *pstate,
				   PlannedStmt *pstmt,
				   const char *queryString,
				   ProcessUtilityContext context,
				   ParamListInfo params,
				   QueryEnvironment *queryEnv,
				   DestReceiver *dest,
				   QueryCompletion *qc)
{
	Node	   *parsetree = pstmt->utilityStmt;
	bool		isTopLevel = (context == PROCESS_UTILITY_TOPLEVEL);
	bool		isCompleteQuery = (context != PROCESS_UTILITY_SUBCOMMAND);
	bool		needCleanup;
	bool		commandCollected = false;
	ObjectAddress address;
	ObjectAddress secondaryObject = InvalidObjectAddress;

	/* All event trigger calls are done only when isCompleteQuery is true */
	needCleanup = isCompleteQuery && EventTriggerBeginCompleteQuery();

	/* PG_TRY block is to ensure we call EventTriggerEndCompleteQuery */
	PG_TRY();
	{
		if (isCompleteQuery)
			EventTriggerDDLCommandStart(parsetree);

		switch (nodeTag(parsetree))
		{
			case T_CreateStmt:
			case T_CreateForeignTableStmt:
				{
					List	   *stmts;
					RangeVar   *table_rv = NULL;

          // 建表语句语义分析，会处理serial，增加一个CreateSeqStmt节点
					/* Run parse analysis ... */
					stmts = transformCreateStmt((CreateStmt *) parsetree,	queryString);

					/*
					 * ... and do it.  We can't use foreach() because we may
					 * modify the list midway through, so pick off the
					 * elements one at a time, the hard way.
					 */
					while (stmts != NIL)
					{
						Node	   *stmt = (Node *) linitial(stmts);

						stmts = list_delete_first(stmts);

						if (IsA(stmt, CreateStmt))
						{
							CreateStmt *cstmt = (CreateStmt *) stmt;
							Datum		toast_options;
							static char *validnsps[] = HEAP_RELOPT_NAMESPACES;

							/* Remember transformed RangeVar for LIKE */
							table_rv = cstmt->relation;

							/* Create the table itself */
							address = DefineRelation(cstmt, RELKIND_RELATION, InvalidOid, NULL, queryString);
							EventTriggerCollectSimpleCommand(address,
															 secondaryObject,
															 stmt);

							/*
							 * Let NewRelationCreateToastTable decide if this
							 * one needs a secondary relation too.
							 */
							CommandCounterIncrement();

							/* parse and validate reloptions for the toast table */
							toast_options = transformRelOptions((Datum) 0,
																cstmt->options,
																"toast",
																validnsps,
																true,
																false);
							(void) heap_reloptions(RELKIND_TOASTVALUE,
												   toast_options,
												   true);

							NewRelationCreateToastTable(address.objectId,
														toast_options);
						}
						else if (IsA(stmt, CreateForeignTableStmt))
						{
                // ...
						}
						else if (IsA(stmt, TableLikeClause))
						{
                // ...
						}
						else
						{
							/*
							 * Recurse for anything else.  Note the recursive
							 * call will stash the objects so created into our event trigger context.
							 */
							PlannedStmt *wrapper;

							wrapper = makeNode(PlannedStmt);
							wrapper->commandType = CMD_UTILITY;
							wrapper->canSetTag = false;
							wrapper->utilityStmt = stmt;
							wrapper->stmt_location = pstmt->stmt_location;
							wrapper->stmt_len = pstmt->stmt_len;

							ProcessUtility(wrapper, queryString, false, PROCESS_UTILITY_SUBCOMMAND, params, NULL, None_Receiver, NULL);
						}

						/* Need CCI between commands */
						if (stmts != NIL)
							CommandCounterIncrement();
					}

					/*
					 * The multiple commands generated here are stashed
					 * individually, so disable collection below.
					 */
					commandCollected = true;
				}
				break;

      // 创建序列
			case T_CreateSeqStmt:
				address = DefineSequence(pstate, (CreateSeqStmt *) parsetree);
				break;

			default:
				elog(ERROR, "unrecognized node type: %d",
					 (int) nodeTag(parsetree));
				break;
		}
	}
  // ...
}
```
处理序列列的代码在函数`transformColumnDefinition`中，负责创建`CreateSeqStmt`，创建Default约束，创建not null约束。
```c++
/*
 * transformColumnDefinition -
 *		transform a single ColumnDef within CREATE TABLE
 *		Also used in ALTER TABLE ADD COLUMN
 */
static void transformColumnDefinition(CreateStmtContext *cxt, ColumnDef *column)
{
	bool		is_serial;
	bool		saw_nullable;
	bool		saw_default;
	bool		saw_identity;
	bool		saw_generated;
	ListCell   *clist;

	cxt->columns = lappend(cxt->columns, column);

	/* Check for SERIAL pseudo-types */
	is_serial = false;
	if (column->typeName && list_length(column->typeName->names) == 1 && !column->typeName->pct_type)
	{
		char	   *typname = strVal(linitial(column->typeName->names));

		if (strcmp(typname, "smallserial") == 0 || strcmp(typname, "serial2") == 0)
		{
			is_serial = true;
			column->typeName->names = NIL;
			column->typeName->typeOid = INT2OID;
		}
		else if (strcmp(typname, "serial") == 0 || strcmp(typname, "serial4") == 0)
		{
			is_serial = true;
			column->typeName->names = NIL;
			column->typeName->typeOid = INT4OID;
		}
		else if (strcmp(typname, "bigserial") == 0 || strcmp(typname, "serial8") == 0)
		{
			is_serial = true;
			column->typeName->names = NIL;
			column->typeName->typeOid = INT8OID;
		}

		/*
		 * We have to reject "serial[]" explicitly, because once we've set
		 * typeid, LookupTypeName won't notice arrayBounds.  We don't need any
		 * special coding for serial(typmod) though.
		 */
		if (is_serial && column->typeName->arrayBounds != NIL)
			ereport(ERROR,
					(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
					 errmsg("array of serial is not implemented"),
					 parser_errposition(cxt->pstate,
										column->typeName->location)));
	}

	/* Do necessary work on the column type declaration */
	if (column->typeName)
		transformColumnType(cxt, column);

	/* Special actions for SERIAL pseudo-types */
	if (is_serial)
	{
		char	   *snamespace;
		char	   *sname;
		char	   *qstring;
		A_Const    *snamenode;
		TypeCast   *castnode;
		FuncCall   *funccallnode;
		Constraint *constraint;

    // 创建CreateSeqStmt节点
		generateSerialExtraStmts(cxt, column,
								 column->typeName->typeOid, NIL,
								 false, false,
								 &snamespace, &sname);

		/*
		 * Create appropriate constraints for SERIAL.  We do this in full,
		 * rather than shortcutting, so that we will detect any conflicting
		 * constraints the user wrote (like a different DEFAULT).
		 *
		 * Create an expression tree representing the function call
		 * nextval('sequencename').  We cannot reduce the raw tree to cooked
		 * form until after the sequence is created, but there's no need to do
		 * so.
		 */
		qstring = quote_qualified_identifier(snamespace, sname);
		snamenode = makeNode(A_Const);
		snamenode->val.node.type = T_String;
		snamenode->val.sval.sval = qstring;
		snamenode->location = -1;
		castnode = makeNode(TypeCast);
		castnode->typeName = SystemTypeName("regclass");
		castnode->arg = (Node *) snamenode;
		castnode->location = -1;
		funccallnode = makeFuncCall(SystemFuncName("nextval"), list_make1(castnode),COERCE_EXPLICIT_CALL,-1);
		constraint = makeNode(Constraint);
		constraint->contype = CONSTR_DEFAULT; // 默认约束
		constraint->location = -1;
		constraint->raw_expr = (Node *) funccallnode; // 默认值为序列的nextval
		constraint->cooked_expr = NULL;
		column->constraints = lappend(column->constraints, constraint);

		constraint = makeNode(Constraint);
		constraint->contype = CONSTR_NOTNULL;
		constraint->location = -1;
		column->constraints = lappend(column->constraints, constraint);
	}

	/* Process column constraints, if any... */
	transformConstraintAttrs(cxt, column->constraints);

	saw_nullable = false;
	saw_default = false;
	saw_identity = false;
	saw_generated = false;

	foreach(clist, column->constraints)
	{
		Constraint *constraint = lfirst_node(Constraint, clist);

		switch (constraint->contype)
		{
			case CONSTR_NULL:
				if (saw_nullable && column->is_not_null)
					ereport(ERROR,
							(errcode(ERRCODE_SYNTAX_ERROR),
							 errmsg("conflicting NULL/NOT NULL declarations for column \"%s\" of table \"%s\"",
									column->colname, cxt->relation->relname),
							 parser_errposition(cxt->pstate, constraint->location)));
				column->is_not_null = false;
				saw_nullable = true;
				break;

			case CONSTR_NOTNULL:
				if (saw_nullable && !column->is_not_null)
					ereport(ERROR,
							(errcode(ERRCODE_SYNTAX_ERROR),
							 errmsg("conflicting NULL/NOT NULL declarations for column \"%s\" of table \"%s\"",
									column->colname, cxt->relation->relname),
							 parser_errposition(cxt->pstate,	constraint->location)));
				column->is_not_null = true;
				saw_nullable = true;
				break;

			case CONSTR_DEFAULT:
				if (saw_default)
					ereport(ERROR,
							(errcode(ERRCODE_SYNTAX_ERROR),
							 errmsg("multiple default values specified for column \"%s\" of table \"%s\"",
									column->colname, cxt->relation->relname),
							 parser_errposition(cxt->pstate,constraint->location)));
				column->raw_default = constraint->raw_expr;

				saw_default = true;
				break;

			case CONSTR_IDENTITY:
				{
					Type		ctype;
					Oid			typeOid;

					if (cxt->ofType)
						ereport(ERROR,
								(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
								 errmsg("identity columns are not supported on typed tables")));
					if (cxt->partbound)
						ereport(ERROR,
								(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
								 errmsg("identity columns are not supported on partitions")));

					ctype = typenameType(cxt->pstate, column->typeName, NULL);
					typeOid = ((Form_pg_type) GETSTRUCT(ctype))->oid;
					ReleaseSysCache(ctype);

					if (saw_identity)
						ereport(ERROR,
								(errcode(ERRCODE_SYNTAX_ERROR),
								 errmsg("multiple identity specifications for column \"%s\" of table \"%s\"",
										column->colname, cxt->relation->relname),
								 parser_errposition(cxt->pstate,
													constraint->location)));

					generateSerialExtraStmts(cxt, column,
											 typeOid, constraint->options,
											 true, false,
											 NULL, NULL);

					column->identity = constraint->generated_when;
					saw_identity = true;

					/* An identity column is implicitly NOT NULL */
					if (saw_nullable && !column->is_not_null)
						ereport(ERROR,
								(errcode(ERRCODE_SYNTAX_ERROR),
								 errmsg("conflicting NULL/NOT NULL declarations for column \"%s\" of table \"%s\"",
										column->colname, cxt->relation->relname),
								 parser_errposition(cxt->pstate,
													constraint->location)));
					column->is_not_null = true;
					saw_nullable = true;
					break;
				}

			case CONSTR_GENERATED:
				if (cxt->ofType)
					ereport(ERROR,
							(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
							 errmsg("generated columns are not supported on typed tables")));
				if (cxt->partbound)
					ereport(ERROR,
							(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
							 errmsg("generated columns are not supported on partitions")));

				if (saw_generated)
					ereport(ERROR,
							(errcode(ERRCODE_SYNTAX_ERROR),
							 errmsg("multiple generation clauses specified for column \"%s\" of table \"%s\"",
									column->colname, cxt->relation->relname),
							 parser_errposition(cxt->pstate,
												constraint->location)));
				column->generated = ATTRIBUTE_GENERATED_STORED;
				column->raw_default = constraint->raw_expr;

				saw_generated = true;
				break;

      // ...

			default:
				elog(ERROR, "unrecognized constraint type: %d", constraint->contype);
				break;
		}
    // ...
	}

  // ...
}
```
具体的生成`CreateSeqStmt`节点的函数：
```c++
/*
 * generateSerialExtraStmts
 *		Generate CREATE SEQUENCE and ALTER SEQUENCE ... OWNED BY statements
 *		to create the sequence for a serial or identity column.
 *
 * This includes determining the name the sequence will have.  The caller
 * can ask to get back the name components by passing non-null pointers
 * for snamespace_p and sname_p.
 */
static void
generateSerialExtraStmts(CreateStmtContext *cxt, ColumnDef *column,
						 Oid seqtypid, List *seqoptions,
						 bool for_identity, bool col_exists,
						 char **snamespace_p, char **sname_p)
{
	ListCell   *option;
	DefElem    *nameEl = NULL;
	Oid			snamespaceid;
	char	   *snamespace;
	char	   *sname;
	CreateSeqStmt *seqstmt;
	AlterSeqStmt *altseqstmt;
	List	   *attnamelist;

	/*
	 * Determine namespace and name to use for the sequence.
	 *
	 * First, check if a sequence name was passed in as an option.  This is
	 * used by pg_dump.  Else, generate a name.
	 *
	 * Although we use ChooseRelationName, it's not guaranteed that the
	 * selected sequence name won't conflict; given sufficiently long field
	 * names, two different serial columns in the same table could be assigned
	 * the same sequence name, and we'd not notice since we aren't creating
	 * the sequence quite yet.  In practice this seems quite unlikely to be a
	 * problem, especially since few people would need two serial columns in
	 * one table.
	 */
	foreach(option, seqoptions)
	{
		DefElem    *defel = lfirst_node(DefElem, option);

		if (strcmp(defel->defname, "sequence_name") == 0)
		{
			if (nameEl)
				ereport(ERROR,
						(errcode(ERRCODE_SYNTAX_ERROR),
						 errmsg("conflicting or redundant options")));
			nameEl = defel;
		}
	}

	if (nameEl)
	{
		RangeVar   *rv = makeRangeVarFromNameList(castNode(List, nameEl->arg));

		snamespace = rv->schemaname;
		if (!snamespace)
		{
			/* Given unqualified SEQUENCE NAME, select namespace */
			if (cxt->rel)
				snamespaceid = RelationGetNamespace(cxt->rel);
			else
				snamespaceid = RangeVarGetCreationNamespace(cxt->relation);
			snamespace = get_namespace_name(snamespaceid);
		}
		sname = rv->relname;
		/* Remove the SEQUENCE NAME item from seqoptions */
		seqoptions = list_delete_ptr(seqoptions, nameEl);
	}
	else
	{
		if (cxt->rel)
			snamespaceid = RelationGetNamespace(cxt->rel);
		else
		{
			snamespaceid = RangeVarGetCreationNamespace(cxt->relation);
			RangeVarAdjustRelationPersistence(cxt->relation, snamespaceid);
		}
		snamespace = get_namespace_name(snamespaceid);
    // 序列名
		sname = ChooseRelationName(cxt->relation->relname, column->colname, "seq", snamespaceid, false);
	}

	ereport(DEBUG1,
			(errmsg("%s will create implicit sequence \"%s\" for serial column \"%s.%s\"",
					cxt->stmtType, sname,
					cxt->relation->relname, column->colname)));

	/*
	 * Build a CREATE SEQUENCE command to create the sequence object, and add
	 * it to the list of things to be done before this CREATE/ALTER TABLE.
	 */
  // 构造CreateSeqStmt节点
	seqstmt = makeNode(CreateSeqStmt);
	seqstmt->for_identity = for_identity;
	seqstmt->sequence = makeRangeVar(snamespace, sname, -1);  // 序列名
	seqstmt->options = seqoptions;

	/*
	 * If a sequence data type was specified, add it to the options.  Prepend
	 * to the list rather than append; in case a user supplied their own AS
	 * clause, the "redundant options" error will point to their occurrence,
	 * not our synthetic one.
	 */
	if (seqtypid)
		seqstmt->options = lcons(makeDefElem("as",
											 (Node *) makeTypeNameFromOid(seqtypid, -1), -1),
								 seqstmt->options);

	/*
	 * If this is ALTER ADD COLUMN, make sure the sequence will be owned by
	 * the table's owner.  The current user might be someone else (perhaps a
	 * superuser, or someone who's only a member of the owning role), but the
	 * SEQUENCE OWNED BY mechanisms will bleat unless table and sequence have
	 * exactly the same owning role.
	 */
	if (cxt->rel)
		seqstmt->ownerId = cxt->rel->rd_rel->relowner;
	else
		seqstmt->ownerId = InvalidOid;

	cxt->blist = lappend(cxt->blist, seqstmt); // 可以看到是放在before list中，在建表的前面创建序列

	/*
	 * Store the identity sequence name that we decided on.  ALTER TABLE ...
	 * ADD COLUMN ... IDENTITY needs this so that it can fill the new column
	 * with values from the sequence, while the association of the sequence
	 * with the table is not set until after the ALTER TABLE.
	 */
	column->identitySequence = seqstmt->sequence;

	/*
	 * Build an ALTER SEQUENCE ... OWNED BY command to mark the sequence as
	 * owned by this column, and add it to the appropriate list of things to
	 * be done along with this CREATE/ALTER TABLE.  In a CREATE or ALTER ADD
	 * COLUMN, it must be done after the statement because we don't know the
	 * column's attnum yet.  But if we do have the attnum (in AT_AddIdentity),
	 * we can do the marking immediately, which improves some ALTER TABLE
	 * behaviors.
	 */
	altseqstmt = makeNode(AlterSeqStmt);
	altseqstmt->sequence = makeRangeVar(snamespace, sname, -1);
	attnamelist = list_make3(makeString(snamespace), makeString(cxt->relation->relname), makeString(column->colname));
	altseqstmt->options = list_make1(makeDefElem("owned_by", (Node *) attnamelist, -1));
	altseqstmt->for_identity = for_identity;

	if (col_exists)
		cxt->blist = lappend(cxt->blist, altseqstmt);
	else
		cxt->alist = lappend(cxt->alist, altseqstmt);

	if (snamespace_p)
		*snamespace_p = snamespace;
	if (sname_p)
		*sname_p = sname;
}

```
最后看一下这个函数`DefineSequence`创建序列，其中还是需要调用函数`DefineRelation`来实现。
```c++
/*
 * DefineSequence
 *				Creates a new sequence relation
 */
ObjectAddress
DefineSequence(ParseState *pstate, CreateSeqStmt *seq)
{
	FormData_pg_sequence seqform;
	FormData_pg_sequence_data seqdataform;
	bool		need_seq_rewrite;
	List	   *owned_by;
	CreateStmt *stmt = makeNode(CreateStmt);
	Oid			seqoid;
	ObjectAddress address;
	Relation	rel;
	HeapTuple	tuple;
	TupleDesc	tupDesc;
	Datum		value[SEQ_COL_LASTCOL];
	bool		null[SEQ_COL_LASTCOL];
	Datum		pgs_values[Natts_pg_sequence];
	bool		pgs_nulls[Natts_pg_sequence];
	int			i;

	/* Unlogged sequences are not implemented -- not clear if useful. */
	if (seq->sequence->relpersistence == RELPERSISTENCE_UNLOGGED)
		ereport(ERROR,
				(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
				 errmsg("unlogged sequences are not supported")));

	/*
	 * If if_not_exists was given and a relation with the same name already
	 * exists, bail out. (Note: we needn't check this when not if_not_exists,
	 * because DefineRelation will complain anyway.)
	 */
	if (seq->if_not_exists)
	{
		RangeVarGetAndCheckCreationNamespace(seq->sequence, NoLock, &seqoid);
		if (OidIsValid(seqoid))
		{
			/*
			 * If we are in an extension script, insist that the pre-existing
			 * object be a member of the extension, to avoid security risks.
			 */
			ObjectAddressSet(address, RelationRelationId, seqoid);
			checkMembershipInCurrentExtension(&address);

			/* OK to skip */
			ereport(NOTICE,
					(errcode(ERRCODE_DUPLICATE_TABLE),
					 errmsg("relation \"%s\" already exists, skipping",
							seq->sequence->relname)));
			return InvalidObjectAddress;
		}
	}

	/* Check and set all option values */
	init_params(pstate, seq->options, seq->for_identity, true,
				&seqform, &seqdataform,
				&need_seq_rewrite, &owned_by);

	/*
	 * Create relation (and fill value[] and null[] for the tuple)
	 */
	stmt->tableElts = NIL;
	for (i = SEQ_COL_FIRSTCOL; i <= SEQ_COL_LASTCOL; i++)
	{
		ColumnDef  *coldef = makeNode(ColumnDef);

		coldef->inhcount = 0;
		coldef->is_local = true;
		coldef->is_not_null = true;
		coldef->is_from_type = false;
		coldef->storage = 0;
		coldef->raw_default = NULL;
		coldef->cooked_default = NULL;
		coldef->collClause = NULL;
		coldef->collOid = InvalidOid;
		coldef->constraints = NIL;
		coldef->location = -1;

		null[i - 1] = false;

		switch (i)
		{
			case SEQ_COL_LASTVAL:
				coldef->typeName = makeTypeNameFromOid(INT8OID, -1);
				coldef->colname = "last_value";
				value[i - 1] = Int64GetDatumFast(seqdataform.last_value);
				break;
			case SEQ_COL_LOG:
				coldef->typeName = makeTypeNameFromOid(INT8OID, -1);
				coldef->colname = "log_cnt";
				value[i - 1] = Int64GetDatum((int64) 0);
				break;
			case SEQ_COL_CALLED:
				coldef->typeName = makeTypeNameFromOid(BOOLOID, -1);
				coldef->colname = "is_called";
				value[i - 1] = BoolGetDatum(false);
				break;
		}
		stmt->tableElts = lappend(stmt->tableElts, coldef);
	}

	stmt->relation = seq->sequence;
	stmt->inhRelations = NIL;
	stmt->constraints = NIL;
	stmt->options = NIL;
	stmt->oncommit = ONCOMMIT_NOOP;
	stmt->tablespacename = NULL;
	stmt->if_not_exists = seq->if_not_exists;

	address = DefineRelation(stmt, RELKIND_SEQUENCE, seq->ownerId, NULL, NULL);
	seqoid = address.objectId;
	Assert(seqoid != InvalidOid);

	rel = table_open(seqoid, AccessExclusiveLock);
	tupDesc = RelationGetDescr(rel);

	/* now initialize the sequence's data */
	tuple = heap_form_tuple(tupDesc, value, null);
	fill_seq_with_data(rel, tuple);

	/* process OWNED BY if given */
	if (owned_by)
		process_owned_by(rel, owned_by, seq->for_identity);

	table_close(rel, NoLock);

	/* fill in pg_sequence */
	rel = table_open(SequenceRelationId, RowExclusiveLock);
	tupDesc = RelationGetDescr(rel);

	memset(pgs_nulls, 0, sizeof(pgs_nulls));

	pgs_values[Anum_pg_sequence_seqrelid - 1] = ObjectIdGetDatum(seqoid);
	pgs_values[Anum_pg_sequence_seqtypid - 1] = ObjectIdGetDatum(seqform.seqtypid);
	pgs_values[Anum_pg_sequence_seqstart - 1] = Int64GetDatumFast(seqform.seqstart);
	pgs_values[Anum_pg_sequence_seqincrement - 1] = Int64GetDatumFast(seqform.seqincrement);
	pgs_values[Anum_pg_sequence_seqmax - 1] = Int64GetDatumFast(seqform.seqmax);
	pgs_values[Anum_pg_sequence_seqmin - 1] = Int64GetDatumFast(seqform.seqmin);
	pgs_values[Anum_pg_sequence_seqcache - 1] = Int64GetDatumFast(seqform.seqcache);
	pgs_values[Anum_pg_sequence_seqcycle - 1] = BoolGetDatum(seqform.seqcycle);

	tuple = heap_form_tuple(tupDesc, pgs_values, pgs_nulls);
	CatalogTupleInsert(rel, tuple);

	heap_freetuple(tuple);
	table_close(rel, RowExclusiveLock);

	return address;
}
```