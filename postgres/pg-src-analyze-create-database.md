### PostgreSQL源码分析——CREATE DATABASE
这里我们分析一下在PostgreSQL中创建数据库的源码，在分析源码之前，最好先阅读《PostgreSQL指南内幕探索》的第一章，数据库集簇、数据库和数据表，弄清其空间布局，理解PG中，数据库、表、元组是怎么布局的。通俗一点说，创建一个数据库相当于创建一个数据目录，在这个目录下，存放该数据库中定义的表、索引等对象。在数据库中，存放数据最基础的是表，我们定义了很多表用来存储用户的数据，每个表是一个文件（大于1G的表会分割为多个文件），而表文件内部被划分为固定长度的页，在页中存放元组，这里可以仔细想一下为什么要这么设计？页的大小设置为多少合适？这里涉及到很多方面，比如磁盘，固态硬盘的物理特性，每次读写的最小单元是扇区或块，操作系统为了性能，往往不会按最小单元进行读写，会批量读取，比如8k为一个块进行读取，数据库为了性能会设有缓冲区，用来将最近使用或最常使用的数据缓存起来，这样能避免频繁进行磁盘读取，（PG中使用时钟扫描算法），如果表不划分为页的话，缓存，性能等都不是最合理的方案，实现索引怎么实现的？ 这里可以再读一下《数据库系统内幕》相关章节。

#### 预分析

这我们再分析源码之前预先想一下`CREATE DATABASE`需要做什么，向pg_database系统表中插入一条数据，数据库名，数据库Oid等信息，创建一个数据库目录（内容从模板数据库copy，目录名为数据库oid）。我们分析一下源码看一下是不是这样。

```sql
-- 查看当前数据库
postgres@postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)

-- 查询系统表
postgres@postgres=# select oid,datname,datdba,encoding from pg_database;
  oid  |  datname  | datdba | encoding 
-------+-----------+--------+----------
 14204 | postgres  |     10 |        6
     1 | template1 |     10 |        6
 14203 | template0 |     10 |        6
(3 rows)

-- 查看数据目录，目录以数据库oid命名
[postgres@slpc base]$ ls
1  14203  14204


postgres@postgres=# \d pg_database;
               Table "pg_catalog.pg_database"
    Column     |   Type    | Collation | Nullable | Default 
---------------+-----------+-----------+----------+---------
 oid           | oid       |           | not null | 
 datname       | name      |           | not null | 
 datdba        | oid       |           | not null | 
 encoding      | integer   |           | not null | 
 datcollate    | name      |           | not null | 
 datctype      | name      |           | not null | 
 datistemplate | boolean   |           | not null | 
 datallowconn  | boolean   |           | not null | 
 datconnlimit  | integer   |           | not null | 
 datlastsysoid | oid       |           | not null | 
 datfrozenxid  | xid       |           | not null | 
 datminmxid    | xid       |           | not null | 
 dattablespace | oid       |           | not null | 
 datacl        | aclitem[] |           |          | 
Indexes:
    "pg_database_datname_index" UNIQUE, btree (datname), tablespace "pg_global"
    "pg_database_oid_index" UNIQUE, btree (oid), tablespace "pg_global"
Tablespace: "pg_global"

```

`CREATE DATABASE`语法使用如下：
```sql
CREATE DATABASE name
    [ WITH ] [ OWNER [=] user_name ]
           [ TEMPLATE [=] template ]
           [ ENCODING [=] encoding ]
           [ STRATEGY [=] strategy ] ]
           [ LOCALE [=] locale ]
           [ LC_COLLATE [=] lc_collate ]
           [ LC_CTYPE [=] lc_ctype ]
           [ ICU_LOCALE [=] icu_locale ]
           [ LOCALE_PROVIDER [=] locale_provider ]
           [ COLLATION_VERSION = collation_version ]
           [ TABLESPACE [=] tablespace_name ]
           [ ALLOW_CONNECTIONS [=] allowconn ]
           [ CONNECTION LIMIT [=] connlimit ]
           [ IS_TEMPLATE [=] istemplate ]
           [ OID [=] oid ]
```
pg_database系统表参考文档：https://www.postgresql.org/docs/current/catalog-pg-database.html
语法参考文档：https://www.postgresql.org/docs/current/sql-createdatabase.html




#### 源码分析
其语法相关可参考：
##### 解析层
关键定义：
```c++
/* ----------------------
 *		Createdb Statement
 * ---------------------- */
typedef struct CreatedbStmt
{
	NodeTag		type;
	char	   *dbname;			/* name of database to create */
	List	   *options;		/* List of DefElem nodes */
} CreatedbStmt;
```


语法定义(gram.y)：
```c++
CreatedbStmt:
			CREATE DATABASE name opt_with createdb_opt_list
				{
					CreatedbStmt *n = makeNode(CreatedbStmt);

					n->dbname = $3;	// 最重要的是告诉数据库，创建的数据库名是什么
					n->options = $5; // 创建数据库的一些选项，比如encoding, tablespace, location等等
					$$ = (Node *) n;
				}
		;
```

##### 执行层 
`CREATE DATABASE`属于非优化语句，跳过优化器，直接进入执行器执行对应的操作。
```c++
exec_simple_query
--> pg_parse_query
	--> raw_parser
--> pg_analyze_and_rewrite
	--> parse_analyze
		--> transformStmt
--> pg_plan_queries
--> PortalStart
--> PortalRun
	--> PortalRunMulti
		--> ProcessUtility
			--> standard_ProcessUtility
				--> createdb		// 执行创建数据库操作
--> PortalDrop
```
下面是`createdb`函数的具体代码，其中一点需要注意，PG采用从模板数据库复制的方式来创建新的数据库，没有指定的话，默认模板是template1数据库。

创建数据库代码如下：
```c++
/* CREATE DATABASE */
Oid createdb(ParseState *pstate, const CreatedbStmt *stmt)
{
	Oid			src_dboid;
	Oid			src_owner;
	int			src_encoding = -1;
	char	   *src_collate = NULL;
	char	   *src_ctype = NULL;
	char	   *src_iculocale = NULL;
	char		src_locprovider = '\0';
	char	   *src_collversion = NULL;
	bool		src_istemplate;
	bool		src_allowconn;
	TransactionId src_frozenxid = InvalidTransactionId;
	MultiXactId src_minmxid = InvalidMultiXactId;
	Oid			src_deftablespace;
	volatile Oid dst_deftablespace;
	Relation	pg_database_rel;
	HeapTuple	tuple;
	Datum		new_record[Natts_pg_database];
	bool		new_record_nulls[Natts_pg_database];
	Oid			dboid = InvalidOid;
	Oid			datdba;
	ListCell   *option;
	DefElem    *dtablespacename = NULL;
	DefElem    *downer = NULL;
	DefElem    *dtemplate = NULL;
	DefElem    *dencoding = NULL;
	DefElem    *dlocale = NULL;
	DefElem    *dcollate = NULL;
	DefElem    *dctype = NULL;
	DefElem    *diculocale = NULL;
	DefElem    *dlocprovider = NULL;
	DefElem    *distemplate = NULL;
	DefElem    *dallowconnections = NULL;
	DefElem    *dconnlimit = NULL;
	DefElem    *dcollversion = NULL;
	DefElem    *dstrategy = NULL;
	char	   *dbname = stmt->dbname;
	char	   *dbowner = NULL;
	const char *dbtemplate = NULL;
	char	   *dbcollate = NULL;
	char	   *dbctype = NULL;
	char	   *dbiculocale = NULL;
	char		dblocprovider = '\0';
	char	   *canonname;
	int			encoding = -1;
	bool		dbistemplate = false;
	bool		dballowconnections = true;
	int			dbconnlimit = -1;
	char	   *dbcollversion = NULL;
	int			notherbackends;
	int			npreparedxacts;
	CreateDBStrategy dbstrategy = CREATEDB_WAL_LOG;
	createdb_failure_params fparms;

	/* Extract options from the statement node tree */
	foreach(option, stmt->options)
	{
		DefElem    *defel = (DefElem *) lfirst(option);

		if (strcmp(defel->defname, "tablespace") == 0)
		{
			if (dtablespacename)
				errorConflictingDefElem(defel, pstate);
			dtablespacename = defel;
		}
		else if (strcmp(defel->defname, "owner") == 0)
		{
			if (downer)
				errorConflictingDefElem(defel, pstate);
			downer = defel;
		}
		else if (strcmp(defel->defname, "template") == 0)
		{
			if (dtemplate)
				errorConflictingDefElem(defel, pstate);
			dtemplate = defel;
		}
		else if (strcmp(defel->defname, "encoding") == 0)
		{
			if (dencoding)
				errorConflictingDefElem(defel, pstate);
			dencoding = defel;
		}
		else if (strcmp(defel->defname, "locale") == 0)
		{
			if (dlocale)
				errorConflictingDefElem(defel, pstate);
			dlocale = defel;
		}
		else if (strcmp(defel->defname, "lc_collate") == 0)
		{
			if (dcollate)
				errorConflictingDefElem(defel, pstate);
			dcollate = defel;
		}
		else if (strcmp(defel->defname, "lc_ctype") == 0)
		{
			if (dctype)
				errorConflictingDefElem(defel, pstate);
			dctype = defel;
		}
		else if (strcmp(defel->defname, "icu_locale") == 0)
		{
			if (diculocale)
				errorConflictingDefElem(defel, pstate);
			diculocale = defel;
		}
		else if (strcmp(defel->defname, "locale_provider") == 0)
		{
			if (dlocprovider)
				errorConflictingDefElem(defel, pstate);
			dlocprovider = defel;
		}
		else if (strcmp(defel->defname, "is_template") == 0)
		{
			if (distemplate)
				errorConflictingDefElem(defel, pstate);
			distemplate = defel;
		}
		else if (strcmp(defel->defname, "allow_connections") == 0)
		{
			if (dallowconnections)
				errorConflictingDefElem(defel, pstate);
			dallowconnections = defel;
		}
		else if (strcmp(defel->defname, "connection_limit") == 0)
		{
			if (dconnlimit)
				errorConflictingDefElem(defel, pstate);
			dconnlimit = defel;
		}
		else if (strcmp(defel->defname, "collation_version") == 0)
		{
			if (dcollversion)
				errorConflictingDefElem(defel, pstate);
			dcollversion = defel;
		}
		else if (strcmp(defel->defname, "location") == 0)
		{
			ereport(WARNING,
					(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
					 errmsg("LOCATION is not supported anymore"),
					 errhint("Consider using tablespaces instead."),
					 parser_errposition(pstate, defel->location)));
		}
		else if (strcmp(defel->defname, "oid") == 0)
		{
			dboid = defGetObjectId(defel);

			/*
			 * We don't normally permit new databases to be created with
			 * system-assigned OIDs. pg_upgrade tries to preserve database
			 * OIDs, so we can't allow any database to be created with an OID
			 * that might be in use in a freshly-initialized cluster created
			 * by some future version. We assume all such OIDs will be from
			 * the system-managed OID range.
			 *
			 * As an exception, however, we permit any OID to be assigned when
			 * allow_system_table_mods=on (so that initdb can assign system
			 * OIDs to template0 and postgres) or when performing a binary
			 * upgrade (so that pg_upgrade can preserve whatever OIDs it finds
			 * in the source cluster).
			 */
			if (dboid < FirstNormalObjectId &&
				!allowSystemTableMods && !IsBinaryUpgrade)
				ereport(ERROR,
						(errcode(ERRCODE_INVALID_PARAMETER_VALUE)),
						errmsg("OIDs less than %u are reserved for system objects", FirstNormalObjectId));
		}
		else if (strcmp(defel->defname, "strategy") == 0)
		{
			if (dstrategy)
				errorConflictingDefElem(defel, pstate);
			dstrategy = defel;
		}
		else
			ereport(ERROR,
					(errcode(ERRCODE_SYNTAX_ERROR),
					 errmsg("option \"%s\" not recognized", defel->defname),
					 parser_errposition(pstate, defel->location)));
	}

	if (downer && downer->arg)
		dbowner = defGetString(downer);
	if (dtemplate && dtemplate->arg)
		dbtemplate = defGetString(dtemplate);
	if (dencoding && dencoding->arg)
	{
		const char *encoding_name;

		if (IsA(dencoding->arg, Integer))
		{
			encoding = defGetInt32(dencoding);
			encoding_name = pg_encoding_to_char(encoding);
			if (strcmp(encoding_name, "") == 0 ||
				pg_valid_server_encoding(encoding_name) < 0)
				ereport(ERROR,
						(errcode(ERRCODE_UNDEFINED_OBJECT),
						 errmsg("%d is not a valid encoding code",
								encoding),
						 parser_errposition(pstate, dencoding->location)));
		}
		else
		{
			encoding_name = defGetString(dencoding);
			encoding = pg_valid_server_encoding(encoding_name);
			if (encoding < 0)
				ereport(ERROR,
						(errcode(ERRCODE_UNDEFINED_OBJECT),
						 errmsg("%s is not a valid encoding name",
								encoding_name),
						 parser_errposition(pstate, dencoding->location)));
		}
	}
	if (dlocale && dlocale->arg)
	{
		dbcollate = defGetString(dlocale);
		dbctype = defGetString(dlocale);
	}
	if (dcollate && dcollate->arg)
		dbcollate = defGetString(dcollate);
	if (dctype && dctype->arg)
		dbctype = defGetString(dctype);
	if (diculocale && diculocale->arg)
		dbiculocale = defGetString(diculocale);
	if (dlocprovider && dlocprovider->arg)
	{
		char	   *locproviderstr = defGetString(dlocprovider);

		if (pg_strcasecmp(locproviderstr, "icu") == 0)
			dblocprovider = COLLPROVIDER_ICU;
		else if (pg_strcasecmp(locproviderstr, "libc") == 0)
			dblocprovider = COLLPROVIDER_LIBC;
		else
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_OBJECT_DEFINITION),
					 errmsg("unrecognized locale provider: %s",
							locproviderstr)));
	}
	if (distemplate && distemplate->arg)
		dbistemplate = defGetBoolean(distemplate);
	if (dallowconnections && dallowconnections->arg)
		dballowconnections = defGetBoolean(dallowconnections);
	if (dconnlimit && dconnlimit->arg)
	{
		dbconnlimit = defGetInt32(dconnlimit);
		if (dbconnlimit < -1)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
					 errmsg("invalid connection limit: %d", dbconnlimit)));
	}
	if (dcollversion)
		dbcollversion = defGetString(dcollversion);

	/* obtain OID of proposed owner */
	if (dbowner)
		datdba = get_role_oid(dbowner, false);
	else
		datdba = GetUserId();

	/*
	 * To create a database, must have createdb privilege and must be able to
	 * become the target role (this does not imply that the target role itself
	 * must have createdb privilege).  The latter provision guards against
	 * "giveaway" attacks.  Note that a superuser will always have both of
	 * these privileges a fortiori.
	 */
	if (!have_createdb_privilege())
		ereport(ERROR,
				(errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
				 errmsg("permission denied to create database")));

	check_is_member_of_role(GetUserId(), datdba);

	/*
	 * Lookup database (template) to be cloned, and obtain share lock on it.
	 * ShareLock allows two CREATE DATABASEs to work from the same template
	 * concurrently, while ensuring no one is busy dropping it in parallel
	 * (which would be Very Bad since we'd likely get an incomplete copy
	 * without knowing it).  This also prevents any new connections from being
	 * made to the source until we finish copying it, so we can be sure it
	 * won't change underneath us.
	 */
	if (!dbtemplate)
		dbtemplate = "template1";	/* Default template database name */

	if (!get_db_info(dbtemplate, ShareLock,
					 &src_dboid, &src_owner, &src_encoding,
					 &src_istemplate, &src_allowconn,
					 &src_frozenxid, &src_minmxid, &src_deftablespace,
					 &src_collate, &src_ctype, &src_iculocale, &src_locprovider,
					 &src_collversion))
		ereport(ERROR,
				(errcode(ERRCODE_UNDEFINED_DATABASE),
				 errmsg("template database \"%s\" does not exist",
						dbtemplate)));

	/*
	 * Permission check: to copy a DB that's not marked datistemplate, you
	 * must be superuser or the owner thereof.
	 */
	if (!src_istemplate)
	{
		if (!pg_database_ownercheck(src_dboid, GetUserId()))
			ereport(ERROR,
					(errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
					 errmsg("permission denied to copy database \"%s\"",
							dbtemplate)));
	}

	/* Validate the database creation strategy. */
	if (dstrategy && dstrategy->arg)
	{
		char	   *strategy;

		strategy = defGetString(dstrategy);
		if (strcmp(strategy, "wal_log") == 0)
			dbstrategy = CREATEDB_WAL_LOG;
		else if (strcmp(strategy, "file_copy") == 0)
			dbstrategy = CREATEDB_FILE_COPY;
		else
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
					 errmsg("invalid create database strategy \"%s\"", strategy),
					 errhint("Valid strategies are \"wal_log\", and \"file_copy\".")));
	}

	/* If encoding or locales are defaulted, use source's setting */
	if (encoding < 0)
		encoding = src_encoding;
	if (dbcollate == NULL)
		dbcollate = src_collate;
	if (dbctype == NULL)
		dbctype = src_ctype;
	if (dblocprovider == '\0')
		dblocprovider = src_locprovider;
	if (dbiculocale == NULL && dblocprovider == COLLPROVIDER_ICU)
		dbiculocale = src_iculocale;

	/* Some encodings are client only */
	if (!PG_VALID_BE_ENCODING(encoding))
		ereport(ERROR,
				(errcode(ERRCODE_WRONG_OBJECT_TYPE),
				 errmsg("invalid server encoding %d", encoding)));

	/* Check that the chosen locales are valid, and get canonical spellings */
	if (!check_locale(LC_COLLATE, dbcollate, &canonname))
		ereport(ERROR,
				(errcode(ERRCODE_WRONG_OBJECT_TYPE),
				 errmsg("invalid locale name: \"%s\"", dbcollate)));
	dbcollate = canonname;
	if (!check_locale(LC_CTYPE, dbctype, &canonname))
		ereport(ERROR,
				(errcode(ERRCODE_WRONG_OBJECT_TYPE),
				 errmsg("invalid locale name: \"%s\"", dbctype)));
	dbctype = canonname;

	check_encoding_locale_matches(encoding, dbcollate, dbctype);

	if (dblocprovider == COLLPROVIDER_ICU)
	{
		if (!(is_encoding_supported_by_icu(encoding)))
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
					 errmsg("encoding \"%s\" is not supported with ICU provider",
							pg_encoding_to_char(encoding))));

		/*
		 * This would happen if template0 uses the libc provider but the new
		 * database uses icu.
		 */
		if (!dbiculocale)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
					 errmsg("ICU locale must be specified")));

		check_icu_locale(dbiculocale);
	}
	else
	{
		if (dbiculocale)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_OBJECT_DEFINITION),
					 errmsg("ICU locale cannot be specified unless locale provider is ICU")));
	}

	/*
	 * Check that the new encoding and locale settings match the source
	 * database.  We insist on this because we simply copy the source data ---
	 * any non-ASCII data would be wrongly encoded, and any indexes sorted
	 * according to the source locale would be wrong.
	 *
	 * However, we assume that template0 doesn't contain any non-ASCII data
	 * nor any indexes that depend on collation or ctype, so template0 can be
	 * used as template for creating a database with any encoding or locale.
	 */
	if (strcmp(dbtemplate, "template0") != 0)
	{
		if (encoding != src_encoding)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
					 errmsg("new encoding (%s) is incompatible with the encoding of the template database (%s)",
							pg_encoding_to_char(encoding),
							pg_encoding_to_char(src_encoding)),
					 errhint("Use the same encoding as in the template database, or use template0 as template.")));

		if (strcmp(dbcollate, src_collate) != 0)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
					 errmsg("new collation (%s) is incompatible with the collation of the template database (%s)",
							dbcollate, src_collate),
					 errhint("Use the same collation as in the template database, or use template0 as template.")));

		if (strcmp(dbctype, src_ctype) != 0)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
					 errmsg("new LC_CTYPE (%s) is incompatible with the LC_CTYPE of the template database (%s)",
							dbctype, src_ctype),
					 errhint("Use the same LC_CTYPE as in the template database, or use template0 as template.")));

		if (dblocprovider != src_locprovider)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
					 errmsg("new locale provider (%s) does not match locale provider of the template database (%s)",
							collprovider_name(dblocprovider), collprovider_name(src_locprovider)),
					 errhint("Use the same locale provider as in the template database, or use template0 as template.")));

		if (dblocprovider == COLLPROVIDER_ICU)
		{
			Assert(dbiculocale);
			Assert(src_iculocale);
			if (strcmp(dbiculocale, src_iculocale) != 0)
				ereport(ERROR,
						(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
						 errmsg("new ICU locale (%s) is incompatible with the ICU locale of the template database (%s)",
								dbiculocale, src_iculocale),
						 errhint("Use the same ICU locale as in the template database, or use template0 as template.")));
		}
	}

	/*
	 * If we got a collation version for the template database, check that it
	 * matches the actual OS collation version.  Otherwise error; the user
	 * needs to fix the template database first.  Don't complain if a
	 * collation version was specified explicitly as a statement option; that
	 * is used by pg_upgrade to reproduce the old state exactly.
	 *
	 * (If the template database has no collation version, then either the
	 * platform/provider does not support collation versioning, or it's
	 * template0, for which we stipulate that it does not contain
	 * collation-using objects.)
	 */
	if (src_collversion && !dcollversion)
	{
		char	   *actual_versionstr;

		actual_versionstr = get_collation_actual_version(dblocprovider, dblocprovider == COLLPROVIDER_ICU ? dbiculocale : dbcollate);
		if (!actual_versionstr)
			ereport(ERROR,
					(errmsg("template database \"%s\" has a collation version, but no actual collation version could be determined",
							dbtemplate)));

		if (strcmp(actual_versionstr, src_collversion) != 0)
			ereport(ERROR,
					(errmsg("template database \"%s\" has a collation version mismatch",
							dbtemplate),
					 errdetail("The template database was created using collation version %s, "
							   "but the operating system provides version %s.",
							   src_collversion, actual_versionstr),
					 errhint("Rebuild all objects in the template database that use the default collation and run "
							 "ALTER DATABASE %s REFRESH COLLATION VERSION, "
							 "or build PostgreSQL with the right library version.",
							 quote_identifier(dbtemplate))));
	}

	if (dbcollversion == NULL)
		dbcollversion = src_collversion;

	/*
	 * Normally, we copy the collation version from the template database.
	 * This last resort only applies if the template database does not have a
	 * collation version, which is normally only the case for template0.
	 */
	if (dbcollversion == NULL)
		dbcollversion = get_collation_actual_version(dblocprovider, dblocprovider == COLLPROVIDER_ICU ? dbiculocale : dbcollate);

	/* Resolve default tablespace for new database */
	if (dtablespacename && dtablespacename->arg)
	{
		char	   *tablespacename;
		AclResult	aclresult;

		tablespacename = defGetString(dtablespacename);
		dst_deftablespace = get_tablespace_oid(tablespacename, false);
		/* check permissions */
		aclresult = pg_tablespace_aclcheck(dst_deftablespace, GetUserId(),
										   ACL_CREATE);
		if (aclresult != ACLCHECK_OK)
			aclcheck_error(aclresult, OBJECT_TABLESPACE,
						   tablespacename);

		/* pg_global must never be the default tablespace */
		if (dst_deftablespace == GLOBALTABLESPACE_OID)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
					 errmsg("pg_global cannot be used as default tablespace")));

		/*
		 * If we are trying to change the default tablespace of the template,
		 * we require that the template not have any files in the new default
		 * tablespace.  This is necessary because otherwise the copied
		 * database would contain pg_class rows that refer to its default
		 * tablespace both explicitly (by OID) and implicitly (as zero), which
		 * would cause problems.  For example another CREATE DATABASE using
		 * the copied database as template, and trying to change its default
		 * tablespace again, would yield outright incorrect results (it would
		 * improperly move tables to the new default tablespace that should
		 * stay in the same tablespace).
		 */
		if (dst_deftablespace != src_deftablespace)
		{
			char	   *srcpath;
			struct stat st;

			srcpath = GetDatabasePath(src_dboid, dst_deftablespace);

			if (stat(srcpath, &st) == 0 &&
				S_ISDIR(st.st_mode) &&
				!directory_is_empty(srcpath))
				ereport(ERROR,
						(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
						 errmsg("cannot assign new default tablespace \"%s\"",
								tablespacename),
						 errdetail("There is a conflict because database \"%s\" already has some tables in this tablespace.",
								   dbtemplate)));
			pfree(srcpath);
		}
	}
	else
	{
		/* Use template database's default tablespace */
		dst_deftablespace = src_deftablespace;
		/* Note there is no additional permission check in this path */
	}

	/*
	 * If built with appropriate switch, whine when regression-testing
	 * conventions for database names are violated.  But don't complain during
	 * initdb.
	 */
#ifdef ENFORCE_REGRESSION_TEST_NAME_RESTRICTIONS
	if (IsUnderPostmaster && strstr(dbname, "regression") == NULL)
		elog(WARNING, "databases created by regression test cases should have names including \"regression\"");
#endif

	/*
	 * Check for db name conflict.  This is just to give a more friendly error
	 * message than "unique index violation".  There's a race condition but
	 * we're willing to accept the less friendly message in that case.
	 */
	if (OidIsValid(get_database_oid(dbname, true)))
		ereport(ERROR,
				(errcode(ERRCODE_DUPLICATE_DATABASE),
				 errmsg("database \"%s\" already exists", dbname)));

	/*
	 * The source DB can't have any active backends, except this one
	 * (exception is to allow CREATE DB while connected to template1).
	 * Otherwise we might copy inconsistent data.
	 *
	 * This should be last among the basic error checks, because it involves
	 * potential waiting; we may as well throw an error first if we're gonna
	 * throw one.
	 */
	if (CountOtherDBBackends(src_dboid, &notherbackends, &npreparedxacts))
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_IN_USE),
				 errmsg("source database \"%s\" is being accessed by other users",
						dbtemplate),
				 errdetail_busy_db(notherbackends, npreparedxacts)));

	/*
	 * Select an OID for the new database, checking that it doesn't have a
	 * filename conflict with anything already existing in the tablespace
	 * directories.
	 */
	pg_database_rel = table_open(DatabaseRelationId, RowExclusiveLock);

	/*
	 * If database OID is configured, check if the OID is already in use or
	 * data directory already exists.
	 */
	if (OidIsValid(dboid))
	{
		char	   *existing_dbname = get_database_name(dboid);

		if (existing_dbname != NULL)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE)),
					errmsg("database OID %u is already in use by database \"%s\"",
						   dboid, existing_dbname));

		if (check_db_file_conflict(dboid))
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_PARAMETER_VALUE)),
					errmsg("data directory with the specified OID %u already exists", dboid));
	}
	else
	{
		/* Select an OID for the new database if is not explicitly configured. */
		do
		{
			dboid = GetNewOidWithIndex(pg_database_rel, DatabaseOidIndexId,
									   Anum_pg_database_oid);
		} while (check_db_file_conflict(dboid));
	}

	/*
	 * Insert a new tuple into pg_database.  This establishes our ownership of
	 * the new database name (anyone else trying to insert the same name will
	 * block on the unique index, and fail after we commit).
	 */

	Assert((dblocprovider == COLLPROVIDER_ICU && dbiculocale) ||
		   (dblocprovider != COLLPROVIDER_ICU && !dbiculocale));

	/* Form tuple */
	MemSet(new_record, 0, sizeof(new_record));
	MemSet(new_record_nulls, false, sizeof(new_record_nulls));

	new_record[Anum_pg_database_oid - 1] = ObjectIdGetDatum(dboid);
	new_record[Anum_pg_database_datname - 1] =
		DirectFunctionCall1(namein, CStringGetDatum(dbname));
	new_record[Anum_pg_database_datdba - 1] = ObjectIdGetDatum(datdba);
	new_record[Anum_pg_database_encoding - 1] = Int32GetDatum(encoding);
	new_record[Anum_pg_database_datlocprovider - 1] = CharGetDatum(dblocprovider);
	new_record[Anum_pg_database_datistemplate - 1] = BoolGetDatum(dbistemplate);
	new_record[Anum_pg_database_datallowconn - 1] = BoolGetDatum(dballowconnections);
	new_record[Anum_pg_database_datconnlimit - 1] = Int32GetDatum(dbconnlimit);
	new_record[Anum_pg_database_datfrozenxid - 1] = TransactionIdGetDatum(src_frozenxid);
	new_record[Anum_pg_database_datminmxid - 1] = TransactionIdGetDatum(src_minmxid);
	new_record[Anum_pg_database_dattablespace - 1] = ObjectIdGetDatum(dst_deftablespace);
	new_record[Anum_pg_database_datcollate - 1] = CStringGetTextDatum(dbcollate);
	new_record[Anum_pg_database_datctype - 1] = CStringGetTextDatum(dbctype);
	if (dbiculocale)
		new_record[Anum_pg_database_daticulocale - 1] = CStringGetTextDatum(dbiculocale);
	else
		new_record_nulls[Anum_pg_database_daticulocale - 1] = true;
	if (dbcollversion)
		new_record[Anum_pg_database_datcollversion - 1] = CStringGetTextDatum(dbcollversion);
	else
		new_record_nulls[Anum_pg_database_datcollversion - 1] = true;

	/*
	 * We deliberately set datacl to default (NULL), rather than copying it
	 * from the template database.  Copying it would be a bad idea when the
	 * owner is not the same as the template's owner.
	 */
	new_record_nulls[Anum_pg_database_datacl - 1] = true;

	tuple = heap_form_tuple(RelationGetDescr(pg_database_rel),
							new_record, new_record_nulls);

	CatalogTupleInsert(pg_database_rel, tuple);

	/*
	 * Now generate additional catalog entries associated with the new DB
	 */

	/* Register owner dependency */
	recordDependencyOnOwner(DatabaseRelationId, dboid, datdba);

	/* Create pg_shdepend entries for objects within database */
	copyTemplateDependencies(src_dboid, dboid);

	/* Post creation hook for new database */
	InvokeObjectPostCreateHook(DatabaseRelationId, dboid, 0);

	/*
	 * If we're going to be reading data for the to-be-created database into
	 * shared_buffers, take a lock on it. Nobody should know that this
	 * database exists yet, but it's good to maintain the invariant that a
	 * lock an AccessExclusiveLock on the database is sufficient to drop all
	 * of its buffers without worrying about more being read later.
	 *
	 * Note that we need to do this before entering the
	 * PG_ENSURE_ERROR_CLEANUP block below, because createdb_failure_callback
	 * expects this lock to be held already.
	 */
	if (dbstrategy == CREATEDB_WAL_LOG)
		LockSharedObject(DatabaseRelationId, dboid, 0, AccessShareLock);

	/*
	 * Once we start copying subdirectories, we need to be able to clean 'em
	 * up if we fail.  Use an ENSURE block to make sure this happens.  (This
	 * is not a 100% solution, because of the possibility of failure during
	 * transaction commit after we leave this routine, but it should handle
	 * most scenarios.)
	 */
	fparms.src_dboid = src_dboid;
	fparms.dest_dboid = dboid;
	fparms.strategy = dbstrategy;

	PG_ENSURE_ERROR_CLEANUP(createdb_failure_callback,
							PointerGetDatum(&fparms));
	{
		/*
		 * If the user has asked to create a database with WAL_LOG strategy
		 * then call CreateDatabaseUsingWalLog, which will copy the database
		 * at the block level and it will WAL log each copied block.
		 * Otherwise, call CreateDatabaseUsingFileCopy that will copy the
		 * database file by file.
		 */
		if (dbstrategy == CREATEDB_WAL_LOG)
			CreateDatabaseUsingWalLog(src_dboid, dboid, src_deftablespace,
									  dst_deftablespace);
		else
			CreateDatabaseUsingFileCopy(src_dboid, dboid, src_deftablespace,
										dst_deftablespace);

		/*
		 * Close pg_database, but keep lock till commit.
		 */
		table_close(pg_database_rel, NoLock);

		/*
		 * Force synchronous commit, thus minimizing the window between
		 * creation of the database files and committal of the transaction. If
		 * we crash before committing, we'll have a DB that's taking up disk
		 * space but is not in pg_database, which is not good.
		 */
		ForceSyncCommit();
	}
	PG_END_ENSURE_ERROR_CLEANUP(createdb_failure_callback,
								PointerGetDatum(&fparms));

	return dboid;
}
```