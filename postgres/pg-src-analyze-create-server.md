## PostgreSQL源码分析——CREATE SERVER

我们分析一下外部数据包装器中创建外部服务器的`CREATE SERVER`命令的实现源码。前面已经分析过很多DDL的语法实现。这里简单描述一下大致过程。

主流程如下所示：
```c++
exec_simple_query(query_string);
--> pg_parse_query(query_string);
    --> raw_parser(query_string, RAW_PARSE_DEFAULT);
        --> base_yyparse(yyscanner);
--> pg_analyze_and_rewrite_fixedparams(parsetree, query_string, NULL, 0, NULL);
--> pg_plan_queries(querytree_list, query_string,	CURSOR_OPT_PARALLEL_OK, NULL);
--> PortalStart(portal, NULL, 0, InvalidSnapshot);
--> PortalRun(portal, FETCH_ALL, true, true, receiver, receiver, &qc);
    --> standard_ProcessUtility(pstmt, queryString, readOnlyTree,context, params, queryEnv,	dest, qc);
        --> CreateForeignServer((CreateForeignServerStmt *) parsetree);
            --> table_open(ForeignServerRelationId, RowExclusiveLock);
            --> get_foreign_server_oid(stmt->servername, true);
            --> transformGenericOptions(ForeignServerRelationId, PointerGetDatum(NULL), stmt->options, fdw->fdwvalidator);
            --> CatalogTupleInsert(rel, tuple);
            --> table_close(rel, RowExclusiveLock);
--> PortalDrop(portal, false);
```


###### 语法解析层面
gram.y语法定义：
```c++
/*****************************************************************************
 *
 *		QUERY:
 *             CREATE SERVER name [TYPE] [VERSION] [OPTIONS]
 *
 *****************************************************************************/

CreateForeignServerStmt: CREATE SERVER name opt_type opt_foreign_server_version
						 FOREIGN DATA_P WRAPPER name create_generic_options
				{
					CreateForeignServerStmt *n = makeNode(CreateForeignServerStmt);

					n->servername = $3;
					n->servertype = $4;
					n->version = $5;
					n->fdwname = $9;
					n->options = $10;
					n->if_not_exists = false;
					$$ = (Node *) n;
				}
				| CREATE SERVER IF_P NOT EXISTS name opt_type opt_foreign_server_version
						 FOREIGN DATA_P WRAPPER name create_generic_options
				{
					CreateForeignServerStmt *n = makeNode(CreateForeignServerStmt);

					n->servername = $6;
					n->servertype = $7;
					n->version = $8;
					n->fdwname = $12;
					n->options = $13;
					n->if_not_exists = true;
					$$ = (Node *) n;
				}
		;

opt_type:
			TYPE_P Sconst			{ $$ = $2; }
			| /*EMPTY*/				{ $$ = NULL; }
		;


foreign_server_version:
			VERSION_P Sconst		{ $$ = $2; }
		|	VERSION_P NULL_P		{ $$ = NULL; }
		;

opt_foreign_server_version:
			foreign_server_version	{ $$ = $1; }
			| /*EMPTY*/				{ $$ = NULL; }
		;
```

###### create foreign server实现
下面是创建外部服务器的主要实现函数，核心就是向`pg_foreign_server`系统表中插入一条记录。
```sql
postgres=# select * from pg_foreign_server ;
  oid  |     srvname      | srvowner | srvfdw | srvtype | srvversion | srvacl |                    srvoptions                    
-------+------------------+----------+--------+---------+------------+--------+--------------------------------------------------
 16820 | s1               |       10 |  16811 |         |            |        | 
 16825 | foreign_pgserver |       10 |  16815 |         |            |        | {host=192.168.109.133,port=6432,dbname=postgres}
(2 rows)
```
具体实现如下：
1. 需要现在外部服务器命名唯一，不能与已有名重复
2. 解析options
```c++
/* Create a foreign server */
ObjectAddress CreateForeignServer(CreateForeignServerStmt *stmt)
{
	Relation	rel;
	Datum		srvoptions;
	Datum		values[Natts_pg_foreign_server];
	bool		nulls[Natts_pg_foreign_server];
	HeapTuple	tuple;
	Oid			srvId;
	Oid			ownerId;
	AclResult	aclresult;
	ObjectAddress myself;
	ObjectAddress referenced;
	ForeignDataWrapper *fdw;

	rel = table_open(ForeignServerRelationId, RowExclusiveLock);

	/* For now the owner cannot be specified on create. Use effective user ID. */
	ownerId = GetUserId();

	/* Check that there is no other foreign server by this name.  If there is
	 * one, do nothing if IF NOT EXISTS was specified. */
	srvId = get_foreign_server_oid(stmt->servername, true);
	if (OidIsValid(srvId))
	{
		if (stmt->if_not_exists)
		{
			/* If we are in an extension script, insist that the pre-existing
			 * object be a member of the extension, to avoid security risks. */
			ObjectAddressSet(myself, ForeignServerRelationId, srvId);
			checkMembershipInCurrentExtension(&myself);

			/* OK to skip */
			ereport(NOTICE,
					(errcode(ERRCODE_DUPLICATE_OBJECT),
					 errmsg("server \"%s\" already exists, skipping",stmt->servername)));
			table_close(rel, RowExclusiveLock);
			return InvalidObjectAddress;
		}
		else
			ereport(ERROR,
					(errcode(ERRCODE_DUPLICATE_OBJECT),
					 errmsg("server \"%s\" already exists",stmt->servername)));
	}

	/* Check that the FDW exists and that we have USAGE on it. Also get the
	 * actual FDW for option validation etc. */
	fdw = GetForeignDataWrapperByName(stmt->fdwname, false);

	aclresult = pg_foreign_data_wrapper_aclcheck(fdw->fdwid, ownerId, ACL_USAGE);
	if (aclresult != ACLCHECK_OK)
		aclcheck_error(aclresult, OBJECT_FDW, fdw->fdwname);

	/* Insert tuple into pg_foreign_server. */
	memset(values, 0, sizeof(values));
	memset(nulls, false, sizeof(nulls));

	srvId = GetNewOidWithIndex(rel, ForeignServerOidIndexId, Anum_pg_foreign_server_oid);
	values[Anum_pg_foreign_server_oid - 1] = ObjectIdGetDatum(srvId);
	values[Anum_pg_foreign_server_srvname - 1] = DirectFunctionCall1(namein, CStringGetDatum(stmt->servername));
	values[Anum_pg_foreign_server_srvowner - 1] = ObjectIdGetDatum(ownerId);
	values[Anum_pg_foreign_server_srvfdw - 1] = ObjectIdGetDatum(fdw->fdwid);

	/* Add server type if supplied */
	if (stmt->servertype)
		values[Anum_pg_foreign_server_srvtype - 1] = CStringGetTextDatum(stmt->servertype);
	else
		nulls[Anum_pg_foreign_server_srvtype - 1] = true;

	/* Add server version if supplied */
	if (stmt->version)
		values[Anum_pg_foreign_server_srvversion - 1] = CStringGetTextDatum(stmt->version);
	else
		nulls[Anum_pg_foreign_server_srvversion - 1] = true;

	/* Start with a blank acl */
	nulls[Anum_pg_foreign_server_srvacl - 1] = true;

	/* Add server options */
	srvoptions = transformGenericOptions(ForeignServerRelationId, PointerGetDatum(NULL), stmt->options, fdw->fdwvalidator);

	if (PointerIsValid(DatumGetPointer(srvoptions)))
		values[Anum_pg_foreign_server_srvoptions - 1] = srvoptions;
	else
		nulls[Anum_pg_foreign_server_srvoptions - 1] = true;

	tuple = heap_form_tuple(rel->rd_att, values, nulls);

	CatalogTupleInsert(rel, tuple);

	heap_freetuple(tuple);

	/* record dependencies */
	myself.classId = ForeignServerRelationId;
	myself.objectId = srvId;
	myself.objectSubId = 0;

	referenced.classId = ForeignDataWrapperRelationId;
	referenced.objectId = fdw->fdwid;
	referenced.objectSubId = 0;
	recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);

	recordDependencyOnOwner(ForeignServerRelationId, srvId, ownerId);

	/* dependency on extension */
	recordDependencyOnCurrentExtension(&myself, false);

	/* Post creation hook for new foreign server */
	InvokeObjectPostCreateHook(ForeignServerRelationId, srvId, 0);

	table_close(rel, RowExclusiveLock);

	return myself;
}
```

##### drop server的流程
`drop server`主流程如下：
```c++
exec_simple_query(query_string);
--> pg_parse_query(query_string);
    --> raw_parser(query_string, RAW_PARSE_DEFAULT);
        --> base_yyparse(yyscanner);
--> pg_analyze_and_rewrite_fixedparams(parsetree, query_string, NULL, 0, NULL);
--> pg_plan_queries(querytree_list, query_string,	CURSOR_OPT_PARALLEL_OK, NULL);
--> PortalStart(portal, NULL, 0, InvalidSnapshot);
--> PortalRun(portal, FETCH_ALL, true, true, receiver, receiver, &qc);
    --> standard_ProcessUtility(pstmt, queryString, readOnlyTree,context, params, queryEnv,	dest, qc);
		--> ExecDropStmt((DropStmt *) parsetree, isTopLevel);
			--> RemoveObjects(stmt);
				--> get_object_address(stmt->removeType, object, &relation, AccessExclusiveLock, stmt->missing_ok);
					--> get_object_address_unqualified(objtype, castNode(String, object), missing_ok);
				--> performMultipleDeletions(objects, stmt->behavior, 0);
					--> deleteObjectsInList(targetObjects, &depRel, flags);
						--> deleteOneObject(thisobj, depRel, flags);
							--> doDeletion(object, flags);
								--> DropObjectById(object);
--> PortalDrop(portal, false);
```

###### 语法定义
```c++
DropStmt:  DROP drop_type_name name_list opt_drop_behavior
				{
					DropStmt *n = makeNode(DropStmt);

					n->removeType = $2;
					n->missing_ok = false;
					n->objects = $3;
					n->behavior = $4;
					n->concurrent = false;
					$$ = (Node *) n;
				}

drop_type_name:
			ACCESS METHOD							{ $$ = OBJECT_ACCESS_METHOD; }
			| EVENT TRIGGER							{ $$ = OBJECT_EVENT_TRIGGER; }
			| EXTENSION								{ $$ = OBJECT_EXTENSION; }
			| FOREIGN DATA_P WRAPPER				{ $$ = OBJECT_FDW; }
			| opt_procedural LANGUAGE				{ $$ = OBJECT_LANGUAGE; }
			| PUBLICATION							{ $$ = OBJECT_PUBLICATION; }
			| SCHEMA								{ $$ = OBJECT_SCHEMA; }
			| SERVER								{ $$ = OBJECT_FOREIGN_SERVER; }
		;

opt_drop_behavior:
			CASCADE						{ $$ = DROP_CASCADE; }
			| RESTRICT					{ $$ = DROP_RESTRICT; }
			| /* EMPTY */				{ $$ = DROP_RESTRICT; /* default */ }
		;
```

执行删除对象：
```c++
/*
 * Drop one or more objects.
 *
 * We don't currently handle all object types here.  Relations, for example,
 * require special handling, because (for example) indexes have additional
 * locking requirements.
 *
 * We look up all the objects first, and then delete them in a single
 * performMultipleDeletions() call.  This avoids unnecessary DROP RESTRICT
 * errors if there are dependencies between them.
 */
void RemoveObjects(DropStmt *stmt)
{
	ObjectAddresses *objects;
	ListCell   *cell1;

	objects = new_object_addresses();

	foreach(cell1, stmt->objects)
	{
		ObjectAddress address;
		Node	   *object = lfirst(cell1);
		Relation	relation = NULL;
		Oid			namespaceId;

		/* Get an ObjectAddress for the object. */
		address = get_object_address(stmt->removeType,
									 object,
									 &relation,
									 AccessExclusiveLock,
									 stmt->missing_ok);

		/*
		 * Issue NOTICE if supplied object was not found.  Note this is only
		 * relevant in the missing_ok case, because otherwise
		 * get_object_address would have thrown an error.
		 */
		if (!OidIsValid(address.objectId))
		{
			Assert(stmt->missing_ok);
			does_not_exist_skipping(stmt->removeType, object);
			continue;
		}

		/*
		 * Although COMMENT ON FUNCTION, SECURITY LABEL ON FUNCTION, etc. are
		 * happy to operate on an aggregate as on any other function, we have
		 * historically not allowed this for DROP FUNCTION.
		 */
		if (stmt->removeType == OBJECT_FUNCTION)
		{
			if (get_func_prokind(address.objectId) == PROKIND_AGGREGATE)
				ereport(ERROR,
						(errcode(ERRCODE_WRONG_OBJECT_TYPE),
						 errmsg("\"%s\" is an aggregate function",
								NameListToString(castNode(ObjectWithArgs, object)->objname)),
						 errhint("Use DROP AGGREGATE to drop aggregate functions.")));
		}

		/* Check permissions. */
		namespaceId = get_object_namespace(&address);
		if (!OidIsValid(namespaceId) ||
			!pg_namespace_ownercheck(namespaceId, GetUserId()))
			check_object_ownership(GetUserId(), stmt->removeType, address,
								   object, relation);

		/*
		 * Make note if a temporary namespace has been accessed in this
		 * transaction.
		 */
		if (OidIsValid(namespaceId) && isTempNamespace(namespaceId))
			MyXactFlags |= XACT_FLAGS_ACCESSEDTEMPNAMESPACE;

		/* Release any relcache reference count, but keep lock until commit. */
		if (relation)
			table_close(relation, NoLock);

		add_exact_object_address(&address, objects);
	}

	/* Here we really delete them. */
	performMultipleDeletions(objects, stmt->behavior, 0);

	free_object_addresses(objects);
}
```