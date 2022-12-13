### Postgres中的视图源码分析——CREATE VIEW
我们知道PostgreSQL中的视图其实就是用规则来实现的。创建一个视图时，会按照视图的定义自动创建一个规则。在对视图进行查询时，则用相应的规则将对视图的查询改写成对基表的查询。下面我们将详细分析一下创建视图的源码。以下面的例句为例进行分析：
```sql
create table t1(a int, b int);
-- 创建视图
create view vt1 as select * from t1;
```
#### 语法解析部分
语法解析部分的源码主流程如下：
```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
        --> base_yyparse
--> pg_analyze_and_rewrite
    --> parse_analyze
        --> transformStmt
    --> pg_rewrite_query
        --> QueryRewrite
--> pg_plan_queries
```
创建视图，生成抽象语法树`ViewStmt`，其中view字段表示视图名，query字段表示视图代表的查询语句
```c++
typedef struct ViewStmt
{
	NodeTag		type;
	RangeVar   *view;			/* the view to be created */
	List	   *aliases;		/* target column names */
	Node	   *query;			/* the SELECT query (as a raw parse tree) */
	bool		replace;		/* replace an existing view? */
	List	   *options;		/* options from WITH clause */
	ViewCheckOption withCheckOption;	/* WITH CHECK OPTION */
} ViewStmt;
```
```c++
/*****************************************************************************
 *
 *	QUERY:
 *		CREATE [ OR REPLACE ] [ TEMP ] VIEW <viewname> '('target-list ')'
 *			AS <query> [ WITH [ CASCADED | LOCAL ] CHECK OPTION ]
 *
 *****************************************************************************/
ViewStmt: CREATE OptTemp VIEW qualified_name opt_column_list opt_reloptions
				AS SelectStmt opt_check_option
				{
					ViewStmt *n = makeNode(ViewStmt);
					n->view = $4;   // 视图名
					n->view->relpersistence = $2;
					n->aliases = $5;
					n->query = $8;  // 查询语句
					n->replace = false;
					n->options = $6;
					n->withCheckOption = $9;
					$$ = (Node *) n;
				}
        // ...  其他的不再列出
```

因为创建视图不能进行优化，所以跳过优化器阶段，直接进入执行器执行。

#### 执行部分

```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
--> pg_plan_queries
--> PortalStart
--> PortalRun
    --> PortalRunUtility    // Execute a utility statement inside a portal.
        --> ProcessUtility
            --> standard_ProcessUtility
                --> ProcessUtilitySlow
                    --> DefineView
                        --> parse_analyze // 将视图查询语句抽象语法树转为查询树
                            --> transformStmt
                                --> transformSelectStmt
                        --> DefineVirtualRelation   // Create the view relation
                        --> DefineRelation
                            --> view_reloptions
                            --> heap_create_with_catalog
                                --> heap_create
                                    --> RelationBuildLocalRelation
                                    // ... 不创建实际的存储空间， RELKIND_RELATION,RELKIND_MATVIEW才会创建
                        --> StoreViewQuery  /* Store the query for the view */
                            --> UpdateRangeTableOfViewParse
                            --> DefineViewRules // Now create the rules associated with the view.
                            --> DefineQueryRewrite // 定义查询规则，视图是通过规则系统实现的
                                --> InsertRule // 向pg_rewrite系统表中插入规则
                                --> SetRelationRuleStatus                
--> PortalDrop
```
从上面的主流程可以看到，创建视图的最后是在pg_rewrite系统表中国插入了一条SELECT规则同时是INSTEAD规则。

我们看一下其插入的结果：
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

创建视图时插入的是INSTEAD规则，INSTEAD规则执行的动作就是用规则中定义的动作代替原始的查询树中的对规则所在表的引用。具体的，在创建视图时，系统会自动按照其定义生成相应的规则，当查询涉及该视图时，查询重写模块都会用对应的规则对该查询进行重写，将对视图的查询改写为对基本表的查询。在生成视图规则时，规则动作是视图创建命令中SELECT语句的拷贝，并且该规则时无条件的INSTEAD规则。
