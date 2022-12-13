## PG源码分析——CREATE EXTENSION
本文分析一下CREATE EXTENSION的源码实现。顺带学习一下PG中的插件相关的原理。

### 语法解析层面
gram.y中的语法定义如下：
```sql
/************************************************************************
 *
 *		QUERY:
 *             CREATE EXTENSION extension
 *             [ WITH ] [ SCHEMA schema ] [ VERSION version ]
 *
 ***************************************************************************/
CreateExtensionStmt: CREATE EXTENSION name opt_with create_extension_opt_list
				{
					CreateExtensionStmt *n = makeNode(CreateExtensionStmt);
					n->extname = $3;
					n->if_not_exists = false;
					n->options = $5;
					$$ = (Node *) n;
				}
				| CREATE EXTENSION IF_P NOT EXISTS name opt_with create_extension_opt_list
				{
					CreateExtensionStmt *n = makeNode(CreateExtensionStmt);
					n->extname = $6;
					n->if_not_exists = true;
					n->options = $8;
					$$ = (Node *) n;
				}
		;

create_extension_opt_list:
			create_extension_opt_list create_extension_opt_item
				{ $$ = lappend($1, $2); }
			| /* EMPTY */
				{ $$ = NIL; }
		;

create_extension_opt_item:
			SCHEMA name
				{
					$$ = makeDefElem("schema", (Node *)makeString($2), @1);
				}
			| VERSION_P NonReservedWord_or_Sconst
				{
					$$ = makeDefElem("new_version", (Node *)makeString($2), @1);
				}
			| FROM NonReservedWord_or_Sconst
				{
					ereport(ERROR,(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),errmsg("CREATE EXTENSION ... FROM is no longer supported"),parser_errposition(@1));
				}
			| CASCADE
				{
					$$ = makeDefElem("cascade", (Node *)makeInteger(true), @1);
				}
		;
```
其核心就是要记录插件名字，有哪些扩展选项：
```c++
typedef struct CreateExtensionStmt
{
	NodeTag		type;
	char	   *extname;
	bool		if_not_exists;	/* just do nothing if it already exists? */
	List	   *options;		/* List of DefElem nodes */
} CreateExtensionStmt;
```
很明显这是一个UTILITY命令，不需要经过查询优化器进行优化，直接简单封装为查询树继而封装为执行计划树后进入执行器执行。

### 执行层面
生成执行计划树后，进入到执行器中进行处理，Utility语句会调用`standard_ProcessUtility`进行处理，对应的创建插件由`CreateExtension`函数实现。
```c++
/* CREATE EXTENSION*/
ObjectAddress CreateExtension(ParseState *pstate, CreateExtensionStmt *stmt)
{
	DefElem    *d_schema = NULL;
	DefElem    *d_new_version = NULL;
	DefElem    *d_cascade = NULL;
	char	   *schemaName = NULL;
	char	   *versionName = NULL;
	bool		cascade = false;
	ListCell   *lc;

	/* Check extension name validity before any filesystem access */
	check_valid_extension_name(stmt->extname);  // 检查插件名称是否有效

	/* Check for duplicate extension name.  The unique index on
	 * pg_extension.extname would catch this anyway, and serves as a backstop
	 * in case of race conditions; but this is a friendlier error message, and
	 * besides we need a check to support IF NOT EXISTS. */
	if (get_extension_oid(stmt->extname, true) != InvalidOid) // 从pg_extension系统表中查看是否有冲突
	{
		if (stmt->if_not_exists)
		{
			ereport(NOTICE,(errcode(ERRCODE_DUPLICATE_OBJECT), errmsg("extension \"%s\" already exists, skipping",stmt->extname)));
			return InvalidObjectAddress;
		}
		else
			ereport(ERROR,(errcode(ERRCODE_DUPLICATE_OBJECT), errmsg("extension \"%s\" already exists",stmt->extname)));
	}

	/* We use global variables to track the extension being created, so we can create only one extension at the same time. */
	if (creating_extension)
		ereport(ERROR,(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),errmsg("nested CREATE EXTENSION is not supported")));

	/* Deconstruct the statement option list */  // 解析options
	foreach(lc, stmt->options)
	{
		DefElem    *defel = (DefElem *) lfirst(lc);

		if (strcmp(defel->defname, "schema") == 0)
		{
			if (d_schema)
				ereport(ERROR,(errcode(ERRCODE_SYNTAX_ERROR),errmsg("conflicting or redundant options"), parser_errposition(pstate, defel->location)));
			d_schema = defel;
			schemaName = defGetString(d_schema);
		}
		else if (strcmp(defel->defname, "new_version") == 0)
		{
			if (d_new_version)
				ereport(ERROR,(errcode(ERRCODE_SYNTAX_ERROR), errmsg("conflicting or redundant options"), parser_errposition(pstate, defel->location)));
			d_new_version = defel;
			versionName = defGetString(d_new_version);
		}
		else if (strcmp(defel->defname, "cascade") == 0)
		{
			if (d_cascade)
				ereport(ERROR,(errcode(ERRCODE_SYNTAX_ERROR), errmsg("conflicting or redundant options"), parser_errposition(pstate, defel->location)));
			d_cascade = defel;
			cascade = defGetBoolean(d_cascade);
		}
		else
			elog(ERROR, "unrecognized option: %s", defel->defname);
	}

	/* Call CreateExtensionInternal to do the real work. */  // 实际创建是在这里实现的
	return CreateExtensionInternal(stmt->extname, schemaName, versionName, cascade, NIL, true);
}
```
我们接下来分析一下关键函数`CreateExtensionInternal`，源码如下：
```c++
/* CREATE EXTENSION worker
 *
 * When CASCADE is specified, CreateExtensionInternal() recurses if required
 * extensions need to be installed.  To sanely handle cyclic dependencies,
 * the "parents" list contains a list of names of extensions already being
 * installed, allowing us to error out if we recurse to one of those.  */
static ObjectAddress CreateExtensionInternal(char *extensionName,char *schemaName,const char *versionName,bool cascade,List *parents,bool is_create)
{
	char	   *origSchemaName = schemaName;
	Oid			schemaOid = InvalidOid;
	Oid			extowner = GetUserId();
	ExtensionControlFile *pcontrol;
	ExtensionControlFile *control;
	char	   *filename;
	struct stat fst;
	List	   *updateVersions;
	List	   *requiredExtensions;
	List	   *requiredSchemas;
	Oid			extensionOid;
	ObjectAddress address;
	ListCell   *lc;

	/* Read the primary control file. */
	pcontrol = read_extension_control_file(extensionName);  // 读*.control文件。在插件的目录里。

	/* Determine the version to install */
	if (versionName == NULL)
	{
		if (pcontrol->default_version)
			versionName = pcontrol->default_version;
		else
			ereport(ERROR,(errcode(ERRCODE_INVALID_PARAMETER_VALUE),errmsg("version to install must be specified")));
	}
	check_valid_version_name(versionName);

	/* Figure out which script(s) we need to run to install the desired
	 * version of the extension.  If we do not have a script that directly
	 * does what is needed, we try to find a sequence of update scripts that will get us there. */
	filename = get_extension_script_filename(pcontrol, NULL, versionName);
	if (stat(filename, &fst) == 0)
		updateVersions = NIL;		/* Easy, no extra scripts */
	else
	{
		/* Look for best way to install this version */
		List	   *evi_list;
		ExtensionVersionInfo *evi_start;
		ExtensionVersionInfo *evi_target;

		/* Extract the version update graph from the script directory */
		evi_list = get_ext_ver_list(pcontrol);

		/* Identify the target version */
		evi_target = get_ext_ver_info(versionName, &evi_list);

		/* Identify best path to reach target */
		evi_start = find_install_path(evi_list, evi_target, &updateVersions);

		/* Fail if no path ... */
		if (evi_start == NULL)
			ereport(ERROR,(errcode(ERRCODE_INVALID_PARAMETER_VALUE), errmsg("extension \"%s\" has no installation script nor update path for version \"%s\"",pcontrol->name, versionName)));

		versionName = evi_start->name;		/* Otherwise, install best starting point and then upgrade */
	}

	/* Fetch control parameters for installation target version */
	control = read_extension_aux_control_file(pcontrol, versionName);

	/* Determine the target schema to install the extension into*/
	if (schemaName)
		schemaOid = get_namespace_oid(schemaName, false);		/* If the user is giving us the schema name, it must exist already. */

	if (control->schema != NULL)
	{
		/* The extension is not relocatable and the author gave us a schema for it.
		 * Unless CASCADE parameter was given, it's an error to give a schema
		 * different from control->schema if control->schema is specified. */
		if (schemaName && strcmp(control->schema, schemaName) != 0 && !cascade)
			ereport(ERROR,(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),errmsg("extension \"%s\" must be installed in schema \"%s\"",control->name,	control->schema)));

		schemaName = control->schema;		/* Always use the schema from control file for current extension. */
		schemaOid = get_namespace_oid(schemaName, true);		/* Find or create the schema in case it does not exist. */
		if (!OidIsValid(schemaOid))
		{
			CreateSchemaStmt *csstmt = makeNode(CreateSchemaStmt);

			csstmt->schemaname = schemaName;
			csstmt->authrole = NULL;	/* will be created by current user */
			csstmt->schemaElts = NIL;
			csstmt->if_not_exists = false;
			CreateSchemaCommand(csstmt, "(generated CREATE SCHEMA command)",-1, -1);

			/* CreateSchemaCommand includes CommandCounterIncrement, so new schema is now visible. */
			schemaOid = get_namespace_oid(schemaName, false);
		}
	}
	else if (!OidIsValid(schemaOid))
	{
		/* Neither user nor author of the extension specified schema; use the current default creation namespace, which is the first explicit entry in the search_path. */
		List	   *search_path = fetch_search_path(false);

		if (search_path == NIL) /* nothing valid in search_path? */
			ereport(ERROR,(errcode(ERRCODE_UNDEFINED_SCHEMA),errmsg("no schema has been selected to create in")));
		schemaOid = linitial_oid(search_path);
		schemaName = get_namespace_name(schemaOid);
		if (schemaName == NULL) /* recently-deleted namespace? */
			ereport(ERROR,(errcode(ERRCODE_UNDEFINED_SCHEMA), errmsg("no schema has been selected to create in")));

		list_free(search_path);
	}

	/* Make note if a temporary namespace has been accessed in this transaction. */
	if (isTempNamespace(schemaOid))
		MyXactFlags |= XACT_FLAGS_ACCESSEDTEMPNAMESPACE;

	/* We don't check creation rights on the target namespace here.  If the
	 * extension script actually creates any objects there, it will fail if
	 * the user doesn't have such permissions.  But there are cases such as
	 * procedural languages where it's convenient to set schema = pg_catalog
	 * yet we don't want to restrict the command to users with ACL_CREATE for pg_catalog.*/

	/* Look up the prerequisite extensions, install them if necessary, and
	 * build lists of their OIDs and the OIDs of their target schemas. */
	requiredExtensions = NIL;
	requiredSchemas = NIL;
	foreach(lc, control->requires)
	{
		char	   *curreq = (char *) lfirst(lc);
		Oid			reqext, reqschema;

		reqext = get_required_extension(curreq,extensionName,origSchemaName,cascade,parents,is_create);
		reqschema = get_extension_schema(reqext);
		requiredExtensions = lappend_oid(requiredExtensions, reqext);
		requiredSchemas = lappend_oid(requiredSchemas, reqschema);
	}

    // 新插件信息插入到系统表pg_extension中。
	/* Insert new tuple into pg_extension, and create dependency entries. */
	address = InsertExtensionTuple(control->name, extowner,schemaOid, control->relocatable,versionName,PointerGetDatum(NULL),PointerGetDatum(NULL),requiredExtensions);
	extensionOid = address.objectId;

	/* Apply any control-file comment on extension*/
	if (control->comment != NULL)
		CreateComments(extensionOid, ExtensionRelationId, 0, control->comment);

	/* Execute the installation script file */  // 执行插件脚本中的sql语句。
	execute_extension_script(extensionOid, control, NULL, versionName, requiredSchemas,schemaName, schemaOid);

	/* If additional update scripts have to be executed, apply the updates as though a series of ALTER EXTENSION UPDATE commands were given*/
	ApplyExtensionUpdates(extensionOid, pcontrol, versionName, updateVersions, origSchemaName, cascade, is_create);

	return address;
}
```

在执行`execute_extension_script`函数时，不同的插件有所差别，但最后都会调用到`_PG_init`。


### 后续
插件创建完成后，可查系统表pg_extension查看：
```sql
postgres@postgres=# select * from pg_extension;
  oid  | extname | extowner | extnamespace | extrelocatable | extversion | extconfig | extcondition 
-------+---------+----------+--------------+----------------+------------+-----------+--------------
 16384 | plpgsql |       10 |           11 | f              | 1.0        |           | 
(1 row)
```
或者通`\x`进行查看：
```sql
postgres@postgres=# \dx
                 List of installed extensions
  Name   | Version |   Schema   |         Description          
---------+---------+------------+------------------------------
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(1 row)l
```
