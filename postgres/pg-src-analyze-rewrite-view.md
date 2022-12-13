## Postgres源码分析——视图查询重写
这里我们分析一下查询重写的过程，主要分析视图的查询重写的过程。通过以下语句进行分析：
```sql
create table t1(a int, b int);
insert into t1 values(1,1);
-- 创建视图
create view vt1 as select * from t1;
-- 查询
select * from vt1;
```
#### 查询重写过程分析
`select * from vt1`生成一个抽象语法树`SelectStmt`。其中视图名vt1通过`RangeVar`节点表示。再由`SelectStmt`经由语义分析转为查询树`Query`。
语法解析部分的源码主流程如下：
```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformFromClause
                --> transformFromClauseItem
    --> pg_rewrite_query
        --> QueryRewrite   // 查询重写，视图重写为子查询是在这里发生的。
            // Step 1: Apply all non-SELECT rules possibly getting 0 or many queries
            --> RewriteQuery
            // Step 2: Apply all the RIR rules on each query
            --> fireRIRrules
                --> ApplyRetrieveRule  // expand an ON SELECT rule
                    --> fireRIRrules    // Recursively expand any view references inside the view.
            /* Step 3: Determine which, if any, of the resulting queries is supposed to set
	                   the command-result tag; and update the canSetTag fields accordingly. */
--> pg_plan_queries
```
视图定义的规则展开的关键函数为`ApplyRetrieveRule`，我们重点分析一下：
```c++
/* ApplyRetrieveRule - expand an ON SELECT rule*/
static Query *ApplyRetrieveRule(Query *parsetree, RewriteRule *rule, int rt_index, Relation relation, List *activeRIRs)
{
	Query	   *rule_action;
	RangeTblEntry *rte, *subrte;
	RowMarkClause *rc;

    // ...

	/*
	 * Make a modifiable copy of the view query, and acquire needed locks on
	 * the relations it mentions.  Force at least RowShareLock for all such
	 * rels if there's a FOR [KEY] UPDATE/SHARE clause affecting this view.
	 */
	rule_action = copyObject(linitial(rule->actions));

	AcquireRewriteLocks(rule_action, true, (rc != NULL));

	/*
	 * Recursively expand any view references inside the view.
	 *
	 * Note: this must happen after markQueryForLocking.  That way, any UPDATE
	 * permission bits needed for sub-views are initially applied to their
	 * RTE_RELATION RTEs by markQueryForLocking, and then transferred to their
	 * OLD rangetable entries by the action below (in a recursive call of this
	 * routine).
	 */
	rule_action = fireRIRrules(rule_action, activeRIRs);  // 规则应用

	/*
	 * Now, plug the view query in as a subselect, converting the relation's
	 * original RTE to a subquery RTE.
	 */
	rte = rt_fetch(rt_index, parsetree->rtable);

	rte->rtekind = RTE_SUBQUERY;  // 视图会通过子查询进行替换，对应的就是由RTE_SUBQUERY替换掉原先的RTE_RELATION
	rte->subquery = rule_action;
	rte->security_barrier = RelationIsSecurityView(relation);
	/* Clear fields that should not be set in a subquery RTE */
	rte->relid = InvalidOid;
	rte->relkind = 0;
	rte->rellockmode = 0;
	rte->tablesample = NULL;
	rte->inh = false;			/* must not be set for a subquery */

	/*
	 * We move the view's permission check data down to its rangetable. The
	 * checks will actually be done against the OLD entry therein.
	 */
	subrte = rt_fetch(PRS2_OLD_VARNO, rule_action->rtable);
	Assert(subrte->relid == relation->rd_id);
	subrte->requiredPerms = rte->requiredPerms;
	subrte->checkAsUser = rte->checkAsUser;
	subrte->selectedCols = rte->selectedCols;
	subrte->insertedCols = rte->insertedCols;
	subrte->updatedCols = rte->updatedCols;
	subrte->extraUpdatedCols = rte->extraUpdatedCols;

    // ...
	return parsetree;
}
```
可以看到，最核心的部分就是将RangeTblEntry由视图转换为子查询。

分析完源码，我们再回过头来研究一下其过程：
执行创建视图命令后`create view vt1 as select * from t1;`，向pg_rewrite系统表中插入了一套数据：
```sql
postgres@postgres=# select * from pg_rewrite order by oid desc limit 1;
oid        | 16390
rulename   | _RETURN
ev_class   | 16387   -- 视图vt1的表oid
ev_type    | 1
ev_enabled | O
is_instead | t       -- instead规则
ev_qual    | <>
ev_action  | ({QUERY :commandType 1 :querySource 0 :canSetTag true :utilityStmt <> :resultRelation 0 :hasAggs false :hasWindowFuncs false :hasTargetSRFs false :hasSubLinks false :hasDistinctOn false :hasRecursive false :hasModifyingCTE false :hasForUpdate false :hasRowSecurity false :cteList <> :rtable ({RTE :alias {ALIAS :aliasname old :colnames <>} :eref {ALIAS :aliasname old :colnames ("a" "b")} :rtekind 0 :relid 16387 :relkind v :rellockmode 1 :tablesample <> :lateral false :inh false :inFromCl false :requiredPerms 0 :checkAsUser 0 :selectedCols (b) :insertedCols (b) :updatedCols (b) :extraUpdatedCols (b) :securityQuals <>} {RTE :alias {ALIAS :aliasname new :colnames <>} :eref {ALIAS :aliasname new :colnames ("a" "b")} :rtekind 0 :relid 16387 :relkind v :rellockmode 1 :tablesample <> :lateral false :inh false :inFromCl false :requiredPerms 0 :checkAsUser 0 :selectedCols (b) :insertedCols (b) :updatedCols (b) :extraUpdatedCols (b) :securityQuals <>} {RTE :alias <> :eref {ALIAS :aliasname t1 :colnames ("a" "b")} :rtekind 0 :relid 16384 :relkind r :rellockmode 1 :tablesample <> :lateral false :inh true :inFromCl true :requiredPerms 2 :checkAsUser 0 :selectedCols (b 8 9) :insertedCols (b) :updatedCols (b) :extraUpdatedCols (b) :securityQuals <>}) :jointree {FROMEXPR :fromlist ({RANGETBLREF :rtindex 3}) :quals <>} :targetList ({TARGETENTRY :expr {VAR :varno 3 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 3 :varattnosyn 1 :location 26} :resno 1 :resname a :ressortgroupref 0 :resorigtbl 16384 :resorigcol 1 :resjunk false} {TARGETENTRY :expr {VAR :varno 3 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 3 :varattnosyn 2 :location 26} :resno 2 :resname b :ressortgroupref 0 :resorigtbl 16384 :resorigcol 2 :resjunk false}) :override 0 :onConflict <> :returningList <> :groupClause <> :groupingSets <> :havingQual <> :windowClause <> :distinctClause <> :sortClause <> :limitOffset <> :limitCount <> :limitOption 0 :rowMarks <> :setOperations <> :constraintDeps <> :withCheckOptions <>})
```
建表以及创建视图后，会向pg_class系统表插入如下的信息。
```sql
postgres@postgres=# select oid,relname,relkind,relhasrules,relrewrite from pg_class where relname='vt1';
-[ RECORD 1 ]------
oid         | 16387     -- 表OID
relname     | vt1       -- 视图名  
relkind     | v         -- 表示视图
relhasrules | t         -- 表是否定义了规则
relrewrite  | 0

postgres@postgres=# select oid,relname,relkind,relhasrules,relrewrite from pg_class where relname='t1';
-[ RECORD 1 ]------
oid         | 16384     -- 表OID
relname     | t1        -- 表名
relkind     | r         -- 表示是普通表
relhasrules | f         -- 表是否定义了规则
relrewrite  | 0
```
当执行`select * from vt1`时，先查pg_class系统表，找到表`vt1`类型为视图，同时该表定义了规则。查看pg_rewrite系统表，找到该表定义的规则的类型以及行为，应用规则。