
### PostgreSQL源码分析——创建用户

我们分析一下创建用户的源码，以`create user lulu with password '123456';`为例。

#### pg_authid系统表
创建一个用户，实际上就是向pg_authid系统表中插入一条数据，然后记录相应的权限、密码等信息。
```sql
postgres=# \d pg_authid
                        Table "pg_catalog.pg_authid"
     Column     |           Type           | Collation | Nullable | Default 
----------------+--------------------------+-----------+----------+---------
 oid            | oid                      |           | not null | 
 rolname        | name                     |           | not null | 
 rolsuper       | boolean                  |           | not null | 
 rolinherit     | boolean                  |           | not null | 
 rolcreaterole  | boolean                  |           | not null | 
 rolcreatedb    | boolean                  |           | not null | 
 rolcanlogin    | boolean                  |           | not null | 
 rolreplication | boolean                  |           | not null | 
 rolbypassrls   | boolean                  |           | not null | 
 rolconnlimit   | integer                  |           | not null | 
 rolpassword    | text                     | C         |          | 
 rolvaliduntil  | timestamp with time zone |           |          | 
Indexes:
    "pg_authid_oid_index" PRIMARY KEY, btree (oid), tablespace "pg_global"
    "pg_authid_rolname_index" UNIQUE CONSTRAINT, btree (rolname), tablespace "pg_global"
Tablespace: "pg_global"
```
其中，密码是通过哈希算法加密存储在rolpassword字段中的。提高了安全性。
```
postgres=# select * from pg_authid where rolname='sli';
-[ RECORD 1 ]--+--------------------------------------------------------------------------------------------------------------------------------------
oid            | 16447
rolname        | sli
rolsuper       | f
rolinherit     | t
rolcreaterole  | f
rolcreatedb    | f
rolcanlogin    | t
rolreplication | f
rolbypassrls   | f
rolconnlimit   | -1
rolpassword    | SCRAM-SHA-256$4096:BPUqhvXNUVcZQpKG5G/omw==$N2d3oPtrU4WogNN8tTMYKDfyY0T5dY+W0ZrsSV+U++g=:XNopc/EGypl1W80Be4UOZ2dMh4SkXEImzEAKXB2BYlY=
rolvaliduntil  | 
```

#### 口令认证
创建一个用户，并设置用户口令密码，当前支持MD5、SCRAM_SHA_256两种算法，推荐使用安全性更高的SCRAM_SHA_256。
```c++
/*
 * Types of password hashes or secrets.
 *
 * Plaintext passwords can be passed in by the user, in a CREATE/ALTER USER
 * command. They will be encrypted to MD5 or SCRAM-SHA-256 format, before
 * storing on-disk, so only MD5 and SCRAM-SHA-256 passwords should appear
 * in pg_authid.rolpassword. They are also the allowed values for the
 * password_encryption GUC.
 */
typedef enum PasswordType
{
	PASSWORD_TYPE_PLAINTEXT = 0,
	PASSWORD_TYPE_MD5,
	PASSWORD_TYPE_SCRAM_SHA_256
} PasswordType;
```
口令认证分为明文口令认证和加密口令认证。明文口令认证要求客户端提供一个未加密的口令进行认证，安全性较差，已经被禁止使用。
```sql
postgres=# create user lu with unencrypted password 'asdf';
ERROR:  UNENCRYPTED PASSWORD is no longer supported
LINE 1: create user lu with unencrypted password 'asdf';
                            ^
HINT:  Remove UNENCRYPTED to store the password in encrypted form instead.
```
加密口令认证要求客户端提供一个经过SCRAM-SHA-256加密的口令进行认证，该口令在传送过程中使用了结合salt的单向哈希加密，增强了安全性。
加密口令的过程： 客户端创建用户密码，用户设置的密码 + 数据库生成的随机数作为输入进行哈希，得到一个认证值，为什么一定要数据库生成一个随机数呢？主要是为了增加安全性。

#### 主流程
`CREATE USER`的主流程如下，实际实现是在`CreateRole`函数中实现的。
```c++
exec_simple_query(query_string);
--> pg_parse_query(query_string);
--> pg_analyze_and_rewrite_fixedparams(parsetree, query_string, NULL, 0, NULL);
--> pg_plan_queries(querytree_list, query_string, CURSOR_OPT_PARALLEL_OK, NULL);
--> PortalStart(portal, NULL, 0, InvalidSnapshot);
--> PortalRun(portal, FETCH_ALL, true, true, receiver, receiver, &qc);
    --> PortalRunMulti(portal, isTopLevel, false, dest, altdest, qc);
        --> PortalRunUtility(portal, pstmt, isTopLevel, false, dest, qc);
            --> ProcessUtility(pstmt, portal->sourceText, (portal->cplan != NULL),  isTopLevel ?PROCESS_UTILITY_TOPLEVEL : PROCESS_UTILITY_QUERY, portal->portalParams, portal->queryEnv, dest, qc);
                --> standard_ProcessUtility(pstmt, queryString, readOnlyTree,context, params, queryEnv,dest, qc);
                    --> CreateRole(pstate, (CreateRoleStmt *) parsetree);
                        // 密码检查钩子 
                        --> (*check_password_hook) (stmt->role,password,get_password_type(password),validUntil_datum,validUntil_null);
                        // 对密码进行加密
                        --> encrypt_password(Password_encryption, stmt->role, password);
                            --> pg_be_scram_build_secret(password); // 默认使用scram-sha-256算法
                                --> pg_saslprep(password, &prep_password);
                                // 产生随机数，由数据库产生随机数
                                --> pg_strong_random(saltbuf, SCRAM_DEFAULT_SALT_LEN)
                                // 由用户输入的密码 + 数据库生成的随机数 生成一个哈希值，进行后续的口令认证，存储在pg_authid.rolpassword中。
                                --> scram_build_secret(saltbuf, SCRAM_DEFAULT_SALT_LEN,SCRAM_DEFAULT_ITERATIONS, password,&errstr);
                                    --> scram_SaltedPassword(password, salt, saltlen, iterations, salted_password, errstr)

```

`CreateRole`函数实现如下：
```c++
Oid CreateRole(ParseState *pstate, CreateRoleStmt *stmt)
{
    Relation	pg_authid_rel;
    // ...
    	/* Extract options from the statement node tree */
	foreach(option, stmt->options)
	{
		DefElem    *defel = (DefElem *) lfirst(option);

		if (strcmp(defel->defname, "password") == 0)
		{
			if (dpassword)
				errorConflictingDefElem(defel, pstate);
			dpassword = defel;
		}
        // ...
    }

    if (dpassword && dpassword->arg)
		password = strVal(dpassword->arg);
    // ...
	pg_authid_rel = table_open(AuthIdRelationId, RowExclusiveLock);
	pg_authid_dsc = RelationGetDescr(pg_authid_rel);

    // 密码检查钩子，其中passwordcheck插件就是通过该钩子实现对用户输入的密码进行检查。
	if (check_password_hook && password)    
		(*check_password_hook) (stmt->role,
								password,
								get_password_type(password),
								validUntil_datum,
								validUntil_null);

    // 构造一个新的tuple，
	MemSet(new_record, 0, sizeof(new_record));
	MemSet(new_record_nulls, false, sizeof(new_record_nulls));

	new_record[Anum_pg_authid_rolname - 1] = DirectFunctionCall1(namein, CStringGetDatum(stmt->role));
	// 如果有设置密码，密码不允许为空
    if (password)
	{
		char	   *shadow_pass;
		const char *logdetail = NULL;

		/*
		 * Don't allow an empty password. Libpq treats an empty password the
		 * same as no password at all, and won't even try to authenticate. But
		 * other clients might, so allowing it would be confusing. By clearing
		 * the password when an empty string is specified, the account is
		 * consistently locked for all clients.
		 *
		 * Note that this only covers passwords stored in the database itself.
		 * There are also checks in the authentication code, to forbid an
		 * empty password from being used with authentication methods that
		 * fetch the password from an external system, like LDAP or PAM.
		 */
		if (password[0] == '\0' || plain_crypt_verify(stmt->role, password, "", &logdetail) == STATUS_OK)
		{
			ereport(NOTICE, (errmsg("empty string is not a valid password, clearing password")));
			new_record_nulls[Anum_pg_authid_rolpassword - 1] = true;
		}
		else
		{
			/* Encrypt the password to the requested format. */
			shadow_pass = encrypt_password(Password_encryption, stmt->role, password);
			new_record[Anum_pg_authid_rolpassword - 1] = CStringGetTextDatum(shadow_pass);
		}
	}
}
```



```c++
typedef struct CreateRoleStmt
{
	NodeTag		type;
	RoleStmtType stmt_type;		/* ROLE/USER/GROUP */
	char	   *role;			/* role name */
	List	   *options;		/* List of DefElem nodes */
} CreateRoleStmt;
```