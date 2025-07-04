## PostgreSQL视图实现原理
这里我们分析一下视图的实现原理。视图的实现共有2部分，其一：创建视图，其二：查询重写。这里我们分别分析一下其实现。

### 视图的基本实现原理
视图实现是基于规则系统实现的。如果理解了规则系统，那么视图的实现原理就非常简单了。

视图的大致过程：
- 创建视图：用户定义一个视图，实际上是创建了一个规则，向pg_rewrite系统表中插入了一条规则记录。
- 查询重写：用户执行一个查询，例如`SELECT * FROM myview;`，在rewrite阶段发现myview是一个视图，查找该视图的规则，并执行该规则，完成查询重写。

这里简要介绍一下规则系统，规则系统可应用于实现视图、行级安全性等。
```sql
postgres=# \h create rule
Command:     CREATE RULE
Description: define a new rewrite rule
Syntax:
CREATE [ OR REPLACE ] RULE name AS ON event
    TO table_name [ WHERE condition ]
    DO [ ALSO | INSTEAD ] { NOTHING | command | ( command ; command ... ) }

where event can be one of:

    SELECT | INSERT | UPDATE | DELETE
```
简要的说，规则系统就是创建一个规则，当用户执行指定的事件（SQL语句：SELECT | INSERT | UPDATE | DELET）时，会触发该规则，并执行指定的SQL语句。行为有2种：
- DO INSTEAD： 用规则中定义的动作替换原查询树中对规则所在表的引用
- DO ALSO：原始查询和规则动作都会被执行。



涉及到的系统表：
- `pg_class`系统表中存储表相关信息，其中的`relhasrules`字段表示该表是否有相关规则。
- `pg_rewrite`系统表中存储规则信息。

### 创建视图
我们知道PostgreSQL中的视图其实就是用规则系统来实现的。创建一个视图时，会按照视图的定义自动创建一个规则。在对视图进行查询时，则用相应的规则将对视图的查询改写成对基表的查询。下面我们将详细分析一下创建视图的源码。最核心的就是创建一条规则，插入到pg_rewrite系统表中。

以下面的例句为例进行分析：
```sql
create table t1(a int, b int);
-- 创建视图
create view vt1 as select * from t1;
```
其中创建视图的命令等同于:
```sql
create table vt1(a int, b int);
create rule _RETURN as on select to vt1 do instead select * from t1;
```

#### 语法解析部分
创建视图，生成抽象语法树`ViewStmt`，其中view字段表示视图名，query字段表示视图代表的查询语句
```c++
typedef struct ViewStmt
{
	NodeTag		type;
	RangeVar   *view;			/* 视图名 */
	List	   *aliases;		/* target column names */
	Node	   *query;			/* 查询树 */
	bool		replace;		/* 是否替换已存在的同名视图 */
	List	   *options;		/* options from WITH clause */
	ViewCheckOption withCheckOption;	/* WITH CHECK OPTION */
} ViewStmt;
```
因为创建视图不能进行优化，所以跳过优化器阶段，直接进入执行器执行。
#### 执行部分
主流程
```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
--> pg_plan_queries
--> PortalStart
--> PortalRun
    --> PortalRunUtility    // 创建视图语句执行
        --> ProcessUtility
            --> standard_ProcessUtility
                --> ProcessUtilitySlow
                    --> DefineView
                        --> parse_analyze // 将视图查询语句抽象语法树转为查询树
                            --> transformStmt
                                --> transformSelectStmt
                        --> DefineVirtualRelation   // 创建视图关系
                            --> DefineRelation
                                --> view_reloptions
                                --> heap_create_with_catalog
                                    --> heap_create
                                        --> RelationBuildLocalRelation
                                        // ... 不创建实际的存储空间， RELKIND_RELATION,RELKIND_MATVIEW才会创建
                            --> StoreViewQuery  /* 规则系统存储该视图规则 */
                                --> UpdateRangeTableOfViewParse
                                --> DefineViewRules // 创建该视图对应的规则
                                --> DefineQueryRewrite // 定义查询规则，视图是通过规则系统实现的
                                    --> InsertRule // 向pg_rewrite系统表中插入规则
                                    --> SetRelationRuleStatus   // 设置视图表pg_class的relhasrules字段为true     
--> PortalDrop
```
从上面的主流程可以看到，创建视图的最后是在pg_rewrite系统表中插入了一条规则，视图表SELECT时触发INSTEAD规则。

我们看一下其插入的结果：
```sql
postgres@postgres=# select * from pg_rewrite order by oid desc limit 1;
oid        | 16390
rulename   | _RETURN  -- 规则名
ev_class   | 16387   -- 视图vt1的表oid
ev_type    | 1
ev_enabled | O
is_instead | t       -- instead规则
ev_qual    | <> 
ev_action  | ({QUERY :commandType 1 :querySource 0 :canSetTag true :utilityStmt <> :resultRelation 0 :hasAggs false :hasWindowFuncs false :hasTargetSRFs false :hasSubLinks false :hasDistinctOn false :hasRecursive false :hasModifyingCTE false :hasForUpdate false :hasRowSecurity false :cteList <> :rtable ({RTE :alias {ALIAS :aliasname old :colnames <>} :eref {ALIAS :aliasname old :colnames ("a" "b")} :rtekind 0 :relid 16387 :relkind v :rellockmode 1 :tablesample <> :lateral false :inh false :inFromCl false :requiredPerms 0 :checkAsUser 0 :selectedCols (b) :insertedCols (b) :updatedCols (b) :extraUpdatedCols (b) :securityQuals <>} {RTE :alias {ALIAS :aliasname new :colnames <>} :eref {ALIAS :aliasname new :colnames ("a" "b")} :rtekind 0 :relid 16387 :relkind v :rellockmode 1 :tablesample <> :lateral false :inh false :inFromCl false :requiredPerms 0 :checkAsUser 0 :selectedCols (b) :insertedCols (b) :updatedCols (b) :extraUpdatedCols (b) :securityQuals <>} {RTE :alias <> :eref {ALIAS :aliasname t1 :colnames ("a" "b")} :rtekind 0 :relid 16384 :relkind r :rellockmode 1 :tablesample <> :lateral false :inh true :inFromCl true :requiredPerms 2 :checkAsUser 0 :selectedCols (b 8 9) :insertedCols (b) :updatedCols (b) :extraUpdatedCols (b) :securityQuals <>}) :jointree {FROMEXPR :fromlist ({RANGETBLREF :rtindex 3}) :quals <>} :targetList ({TARGETENTRY :expr {VAR :varno 3 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 3 :varattnosyn 1 :location 26} :resno 1 :resname a :ressortgroupref 0 :resorigtbl 16384 :resorigcol 1 :resjunk false} {TARGETENTRY :expr {VAR :varno 3 :varattno 2 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnosyn 3 :varattnosyn 2 :location 26} :resno 2 :resname b :ressortgroupref 0 :resorigtbl 16384 :resorigcol 2 :resjunk false}) :override 0 :onConflict <> :returningList <> :groupClause <> :groupingSets <> :havingQual <> :windowClause <> :distinctClause <> :sortClause <> :limitOffset <> :limitCount <> :limitOption 0 :rowMarks <> :setOperations <> :constraintDeps <> :withCheckOptions <>})  -- 替换为源表t1
```
再看一下系统表pg_class中表t1以及视图vt1中的信息：
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

下面我们看一下系统表pg_rewrite的定义，其存储重写规则。
```sql
postgres@postgres=# \d pg_rewrite
               Table "pg_catalog.pg_rewrite"
   Column   |     Type     | Collation | Nullable | Default 
------------+--------------+-----------+----------+---------
 oid        | oid          |           | not null | 
 rulename   | name         |           | not null |    -- 规则名称
 ev_class   | oid          |           | not null |    -- 使用该规则的表名称
 ev_type    | "char"       |           | not null |    -- 规则适用的命令类型
 ev_enabled | "char"       |           | not null | 
 is_instead | boolean      |           | not null |    -- 是否是instead规则
 ev_qual    | pg_node_tree | C         | not null |    -- 规则的条件表达式
 ev_action  | pg_node_tree | C         | not null |    -- 规则动作的查询树
Indexes:
    "pg_rewrite_oid_index" UNIQUE, btree (oid)
    "pg_rewrite_rel_rulename_index" UNIQUE, btree (ev_class, rulename)
```

规则工作原理：对于每个规则（ 一个pg_rewrite元组），该元组的ev_class属性表示该规则适用的表名，如果在该表上执行特定的命令（ev_type）且满足了规则的条件（ev_qual）时，用规则的动作（ev_action）替换原始命令的动作或者将规则的动作附加在原始命令之前。

创建视图时插入的是INSTEAD规则，INSTEAD规则执行的动作就是用规则中定义的动作代替原始的查询树中的对规则所在表的引用。具体的，在创建视图时，系统会自动按照其定义生成相应的规则，<font color=red>当查询涉及该视图时，查询重写模块都会用对应的规则对该查询进行重写，将对视图的查询改写为对基本表的查询</font>。在生成视图规则时，规则动作是视图创建命令中SELECT语句的拷贝，并且该规则时无条件的INSTEAD规则。

### 查询重写
查询重写所处的阶段是规则重写阶段，即在解析器之后，优化器之前，其主要的处理逻辑是重写查询树。
```txt
SQL --> parser（生成抽象语法树RawStmt）--> analyze（生成查询树Query）--> rewrite（规则重写）--> optimize（逻辑优化+物理优化）--> execute（执行SQL） 
```
我们通过实际代码调试来进行分析。

`select * from vt1`生成一个抽象语法树`SelectStmt`。其中视图名vt1通过`RangeVar`节点表示。再由`SelectStmt`经由语义分析转为查询树`Query`。
语法解析部分的源码主流程如下：
```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
            --> transformFromClause
                --> transformFromClauseItem
    --> pg_rewrite_query  // 查询重写阶段
        --> QueryRewrite   // 查询重写，视图重写为子查询是在这里发生的。
            // Step 1: 应用所有的非SELECT规则
            --> RewriteQuery
            // Step 2: 应用所有的RIR规则，视图在这里进行处理
            --> fireRIRrules
                --> ApplyRetrieveRule  // 展开ON SELECT规则
                    --> fireRIRrules    // 递归展开视图的_RETURN规则
--> pg_plan_queries
```
我们看一下其中的关键实现函数`fireRIRrules`的源码：
```c++
static Query *fireRIRrules(Query *parsetree, List *activeRIRs)
{
    // 遍历所有表，看有无对应的规则，如果有规则，则应用规则
	while (rt_index < list_length(parsetree->rtable))
	{
        RangeTblEntry *rte = rt_fetch(rt_index, parsetree->rtable)
        
        rel = table_open(rte->relid, NoLock); // 打开视图表
		rules = rel->rd_rules;  // 获取视图表的所有规则
		if (rules != NULL)
		{
			locks = NIL;
			for (i = 0; i < rules->numLocks; i++)
			{
				rule = rules->rules[i];
				if (rule->event != CMD_SELECT)
					continue;  // 只处理SELECT规则

				locks = lappend(locks, rule);
			}

			/* If we found any, apply them --- but first check for recursion! */
			if (locks != NIL)
			{
                // ...
				foreach(l, locks)
				{
					rule = lfirst(l);
                    // 对每个规则进行应用
					parsetree = ApplyRetrieveRule(parsetree, rule, rt_index, rel, activeRIRs);
				}

				activeRIRs = list_delete_last(activeRIRs);
			}
		}

		table_close(rel, NoLock);
    }
}
```
其中，视图定义的规则展开的关键函数为`ApplyRetrieveRule`，我们重点分析一下：
```c++
/* 应用规则系统， ON SELECT DO INSTEAD规则 */
static Query *ApplyRetrieveRule(Query *parsetree, RewriteRule *rule, int rt_index, Relation relation, List *activeRIRs)
{
	Query	   *rule_action;
	RangeTblEntry *rte, *subrte;

    // 递归地展开视图内部的所有视图引用，确保所有嵌套视图也被展开
	rule_action = fireRIRrules(rule_action, activeRIRs);  // 应用规则

    // 将视图查询作为子查询插入，并将该关系的原始 RTE 转换为 subquery RTE（子查询类型的RTE）
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
可以看到，视图实现的原理，最核心的思想就是查询重写，重写查询树，那么按什么规则重写呢？需要用户自己定义视图规则，每当进入查询重写阶段，就会遍历查询树中的表，如果表含有视图规则，则应该规则进行重写。这种机制的实现为用户以及管理员提供了灵活性，在开发以及运维等场景广泛应用。