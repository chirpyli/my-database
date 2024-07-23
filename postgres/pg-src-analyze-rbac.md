### PostgreSQL源码分析——对象访问控制
PostgreSQL通过基于角色的访问控制（RBAC）机制来管理数据库对象的访问权限。RBAC是一种基于角色的访问控制模型，它将权限与角色关联，然后将角色授予用户。用户通过拥有角色来获得相应的权限。在PostgreSQL中，每个数据库对象都有一个或多个拥有者，拥有者拥有对该对象的完全控制权。除了拥有者外，还可以将其他用户或角色添加为对象的拥有者或成员。拥有者或成员可以通过授予或撤销权限来控制其他用户对对象的访问。PostgreSQL数据库对象包括表、视图、序列、索引、函数、存储过程等。当用户要对数据库对象进行任何操作时都必须先经过PostgreSQL的权限检查。同时，用户对象的权限发生变更时，需要对用户在对象上的操作权限进行维护和更新。


#### 示例
我们看一下具体的例子，默认postgres用户下创建一个表，插入数据，新创建一个用户hangzhou，在postgres用户没有赋予hangzhou用户表t1的读权限的情况下，hangzhou用户登录后无法读取表t1的数据。
```sql
--创建一个用户
postgres=# create user hangzhou with password 'hangzhou';
CREATE ROLE
-- 创建一个表
postgres=# create table t1(a int);
CREATE TABLE
-- 插入数据
postgres=# insert into t1 values(1);
INSERT 0 1
-- hangzhou用户登录
postgres@slpc: psql -U hangzhou -d postgres
psql (14.12)
Type "help" for help.
-- 没有读权限，被拒绝访问
postgres=> select * from t1;
ERROR:  permission denied for table t1

--通过grant授权
postgres=# grant select on t1 to hangzhou;
GRANT
-- 有权限了，可以访问了

postgres@slpc: psql -U hangzhou -d postgres
psql (14.12)
Type "help" for help.

postgres=> select * from t1;
 a 
---
 1
(1 row)

```


#### 源码分析

我们对`select * from t1`查询流程进行源码分析，看一下，是在哪里进行的权限检查，是在生成执行计划后，在执行前进行权限检查的。

```c++
exec_simple_query
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformSelectStmt
                --> transformFromClause
                    --> transformTableEntry
                        --> addRangeTableEntry
                            {
                                RangeTblEntry *rte = makeNode(RangeTblEntry);
                                rte->requiredPerms = ACL_SELECT;
                            }
    --> pg_rewrite_query
--> pg_plan_queries
    --> standard_planner
        --> subquery_planner
            --> grouping_planner
                --> query_planner
                    --> add_base_rels_to_query
                    --> make_one_rel
        --> create_plan
            --> create_plan_recurse
                --> create_scan_plan
--> PortalStart
    --> ExecutorStart
        --> InitPlan
            // Do permissions checks
            --> ExecCheckRTPerms   
                --> ExecCheckRTEPerms  // 对单个RTE进行权限检查
                    --> pg_class_aclmask
                        --> pg_class_aclmask_ext
                    --> pg_attribute_aclcheck
                        --> pg_attribute_aclcheck_ext
                            --> pg_attribute_aclmask_ext
                                --> aclmask
                --> aclcheck_error
--> PortalRun
--> PortalDrop
```

进行权限检查`ExecCheckRTPerms`，我们具体的看一下这个函数
```c++
/*
 * ExecCheckRTPerms
 *		Check access permissions for all relations listed in a range table.
 *
 * Returns true if permissions are adequate.  Otherwise, throws an appropriate
 * error if ereport_on_violation is true, or simply returns false otherwise.
 */
bool ExecCheckRTPerms(List *rangeTable, bool ereport_on_violation)
{
	ListCell   *l;
	bool		result = true;

	foreach(l, rangeTable)
	{
		RangeTblEntry *rte = (RangeTblEntry *) lfirst(l);

		result = ExecCheckRTEPerms(rte);
		if (!result)
		{
			Assert(rte->rtekind == RTE_RELATION);
			if (ereport_on_violation)
				aclcheck_error(ACLCHECK_NO_PRIV, get_relkind_objtype(get_rel_relkind(rte->relid)),
							   get_rel_name(rte->relid));
			return false;
		}
	}

	if (ExecutorCheckPerms_hook)
		result = (*ExecutorCheckPerms_hook) (rangeTable, ereport_on_violation);
	return result;
}

/*
 * ExecCheckRTEPerms
 *		Check access permissions for a single RTE.
 */
static bool ExecCheckRTEPerms(RangeTblEntry *rte)
{
	AclMode		requiredPerms;
	AclMode		relPerms;
	AclMode		remainingPerms;
	Oid			relOid;
	Oid			userid;

	/*
	 * Only plain-relation RTEs need to be checked here.  Function RTEs are
	 * checked when the function is prepared for execution.  Join, subquery,
	 * and special RTEs need no checks. */
	if (rte->rtekind != RTE_RELATION)
		return true;

	/* No work if requiredPerms is empty. */
	requiredPerms = rte->requiredPerms;
	if (requiredPerms == 0)
		return true;

	relOid = rte->relid;

	/* userid to check as: current user unless we have a setuid indication. */
	userid = rte->checkAsUser ? rte->checkAsUser : GetUserId();

	/*
	 * We must have *all* the requiredPerms bits, but some of the bits can be
	 * satisfied from column-level rather than relation-level permissions.
	 * First, remove any bits that are satisfied by relation permissions.
	 */
	relPerms = pg_class_aclmask(relOid, userid, requiredPerms, ACLMASK_ALL);
	remainingPerms = requiredPerms & ~relPerms;
	if (remainingPerms != 0)
	{
		int			col = -1;

		/* If we lack any permissions that exist only as relation permissions, we can fail straight away. */
		if (remainingPerms & ~(ACL_SELECT | ACL_INSERT | ACL_UPDATE))
			return false;

		/*
		 * Check to see if we have the needed privileges at column level.
		 *
		 * Note: failures just report a table-level error; it would be nicer
		 * to report a column-level error if we have some but not all of the
		 * column privileges.
		 */
		if (remainingPerms & ACL_SELECT)
		{
			/*
			 * When the query doesn't explicitly reference any columns (for
			 * example, SELECT COUNT(*) FROM table), allow the query if we
			 * have SELECT on any column of the rel, as per SQL spec.
			 */
			if (bms_is_empty(rte->selectedCols))
			{
				if (pg_attribute_aclcheck_all(relOid, userid, ACL_SELECT,
											  ACLMASK_ANY) != ACLCHECK_OK)
					return false;
			}

			while ((col = bms_next_member(rte->selectedCols, col)) >= 0)
			{
				/* bit #s are offset by FirstLowInvalidHeapAttributeNumber */
				AttrNumber	attno = col + FirstLowInvalidHeapAttributeNumber;

				if (attno == InvalidAttrNumber)
				{
					/* Whole-row reference, must have priv on all cols */
					if (pg_attribute_aclcheck_all(relOid, userid, ACL_SELECT,
												  ACLMASK_ALL) != ACLCHECK_OK)
						return false;
				}
				else
				{
					if (pg_attribute_aclcheck(relOid, attno, userid,
											  ACL_SELECT) != ACLCHECK_OK)
						return false;
				}
			}
		}

		/*
		 * Basically the same for the mod columns, for both INSERT and UPDATE
		 * privilege as specified by remainingPerms.
		 */
		if (remainingPerms & ACL_INSERT && !ExecCheckRTEPermsModified(relOid,
																	  userid,
																	  rte->insertedCols,
																	  ACL_INSERT))
			return false;

		if (remainingPerms & ACL_UPDATE && !ExecCheckRTEPermsModified(relOid,
																	  userid,
																	  rte->updatedCols,
																	  ACL_UPDATE))
			return false;
	}
	return true;
}
```

#### GRANT授权源码分析
我们对GRANT授权语句进行解析，并分析其源码实现。
```sql
grant select on t1 to hangzhou;
```
调用栈如下，
```c++
merge_acl_with_grant(Acl * old_acl, _Bool is_grant, _Bool grant_option, DropBehavior behavior, List * grantees, AclMode privileges, Oid grantorId, Oid ownerId) (src\backend\catalog\aclchk.c:200)
ExecGrant_Relation(InternalGrant * istmt) (src\backend\catalog\aclchk.c:1971)
ExecGrantStmt_oids(InternalGrant * istmt) (src\backend\catalog\aclchk.c:570)
ExecuteGrantStmt(GrantStmt * stmt) (src\backend\catalog\aclchk.c:555)
ProcessUtilitySlow(ParseState * pstate, PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\utility.c:1794)
standard_ProcessUtility(PlannedStmt * pstmt, const char * queryString, _Bool readOnlyTree, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\utility.c:965)
ProcessUtility(PlannedStmt * pstmt, const char * queryString, _Bool readOnlyTree, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\utility.c:527)
PortalRunUtility(Portal portal, PlannedStmt * pstmt, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\pquery.c:1155)
PortalRunMulti(Portal portal, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:1312)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:788)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1213)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (src\backend\tcop\postgres.c:4513)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4540)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4262)
ServerLoop() (src\backend\postmaster\postmaster.c:1748)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1420)
main(int argc, char ** argv) (src\backend\main\main.c:209)
```
grant语句，实际上就是更改acl列表中的内容。我们观察relacl的值，新增了权限信息
```sql
postgres=# select * from pg_class where relname='t1';
-[ RECORD 1 ]-------+------------------------------------------------
oid                 | 16385
relname             | t1
relnamespace        | 2200
reltype             | 16387
reloftype           | 0
relowner            | 10
relam               | 2
relfilenode         | 16385
reltablespace       | 0
relpages            | 0
reltuples           | -1
relallvisible       | 0
reltoastrelid       | 0
relhasindex         | f
relisshared         | f
relpersistence      | p
relkind             | r
relnatts            | 1
relchecks           | 0
relhasrules         | f
relhastriggers      | f
relhassubclass      | f
relrowsecurity      | f
relforcerowsecurity | f
relispopulated      | t
relreplident        | d
relispartition      | f
relrewrite          | 0
relfrozenxid        | 734
relminmxid          | 1
relacl              | {postgres=arwdDxt/postgres,hangzhou=r/postgres}
reloptions          | 
relpartbound        | 
```