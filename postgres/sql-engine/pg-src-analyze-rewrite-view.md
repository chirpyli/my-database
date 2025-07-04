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