### 子连接优化
子查询的逻辑优化内容很多，这里不打算在一篇内容中全部叙述清楚，我们先以`select * from t1 where id in (select a from t2);`这条SQL语句为例分析PostgreSQL对子连接的逻辑优化过程。在往下讲之前，先区别一下子连接与子查询，因为代码中分别会对子连接和子查询进行上拉操作。怎么区别呢？通常，以范围表的方式存在的称为子查询，以表达式的方式存在的称为子连接。实际上也可以通过子句所处的位置来区分子连接和子查询，出现在FROM关键字之后的子句时子查询语句，出现在WHERE等约束条件中的是子连接。

我们先通过执行计划进行观察分析，数据库通过`Hash Semi Join`对子连接进行了逻辑优化。
```sql
--查看执行计划
postgres=# explain select * from t1 where id in (select a from t2);
                          QUERY PLAN                           
---------------------------------------------------------------
 Hash Semi Join  (cost=1.09..2.21 rows=4 width=8)
   Hash Cond: (t1.id = t2.a)
   ->  Seq Scan on t1  (cost=0.00..1.06 rows=6 width=8)
   ->  Hash  (cost=1.04..1.04 rows=4 width=4)
         ->  Seq Scan on t2  (cost=0.00..1.04 rows=4 width=4)
```

#### 解析层分析
我们先分析子连接在解析层的表示（以这条语句为例，其他子连接形式类似）：
```c++
 /* A SubLink represents a subselect appearing in an expression, and in some
 * cases also the combining operator(s) just above it. */
typedef struct SubLink
{
	Expr		xpr;
	SubLinkType subLinkType;	/* see above */
	int			subLinkId;		/* ID (1..n); 0 if not MULTIEXPR */
	Node	   *testexpr;		/* outer-query test for ALL/ANY/ROWCOMPARE */
	List	   *operName;		/* originally specified operator name */
	Node	   *subselect;		/* subselect as Query* or raw parsetree */
	int			location;		/* token location, or -1 if unknown */
} SubLink;
```
语法定义如下：
```c++
// 匹配 where id in (select a from t2);
where_clause:
			WHERE a_expr							{ $$ = $2; }
			| /*EMPTY*/								{ $$ = NULL; }
		;
// 匹配 id in (select a from t2);
a_expr:		c_expr									{ $$ = $1; }
			| a_expr IN_P in_expr
				{
					/* in_expr returns a SubLink or a list of a_exprs */
					if (IsA($3, SubLink))
					{
						/* generate foo = ANY (subquery) */
						SubLink *n = (SubLink *) $3;
						n->subLinkType = ANY_SUBLINK;
						n->subLinkId = 0;
						n->testexpr = $1;   // 具体到上面的SQL语句是id column
						n->operName = NIL;		/* show it's IN not = ANY */
						n->location = @2;
						$$ = (Node *)n;
					}
					else
					{
						/* generate scalar IN expression */
						$$ = (Node *) makeSimpleA_Expr(AEXPR_IN, "=", $1, $3, @2);
					}
				}
// 匹配 (select a from t2)
in_expr:	select_with_parens
				{
					SubLink *n = makeNode(SubLink);
					n->subselect = $1;
					/* other fields will be filled later */
					$$ = (Node *)n;
				}
			| '(' expr_list ')'						{ $$ = (Node *)$2; }
		;
```

我们知道`"x IN (select)", convert to "x = ANY (select)"`这个转换是等价的。在解析阶段，会进行`IN`转`ANY`的处理，在调用过程如下：`transformSelectStmt`->`transformWhereClause`->`transformExpr`->`transformSubLink`。在`transformSublink`中进行这个转换处理：
```c++
static Node *transformSubLink(ParseState *pstate, SubLink *sublink)
{
	  Node	   *result = (Node *) sublink;
	  Query	   *qtree;

    // 省略中间代码......
		/* If the source was "x IN (select)", convert to "x = ANY (select)".*/
		if (sublink->operName == NIL)
			sublink->operName = list_make1(makeString("="));

    // 省略中间代码......

	  return result;
}      
```

理解了子连接在解析层的表示后，我们重点分析一下逻辑优化阶段的处理。


#### 逻辑优化
逻辑优化的基础是关系代数的等价变换，等价推理。有点类似于中学时数学中的简便运算思想。最容易理解的优化思想是：先选择后投影，比如选择下推的目的是减少中间计算结果从而达到优化的效果。在PG中，查询优化器的代码在`src/backend/optimizer`目录下：
```c++
optimizer
├── geqo  // 遗传算法，在连接较多的情况下会选择该算法，阈值可调整，默认geqo_threshold=12
├── path  // 物理优化
├── plan  // 优化器入口
├── prep  // 逻辑优化相关
│   ├── prepjointree.c  // 处理连接操作
│   ├── prepqual.c      // 处理选择条件
│   ├── preptlist.c     // 处理投影相关
│   └── prepunion.c     // 处理集合操作  
└── util  // 公共函数
```
逻辑优化的内容有很多，这里我们只分析子连接的逻辑优化（子连接上拉），将子连接转为SEMI-JOIN，使得子连接中的子查询有机会与父查询语句进行合并优化。我们看一下具体的下面的执行计划。可以看到将子查询转为了SEMI-JOIN的形式。
```sql
postgres=# explain select * from t1 where id in (select cid from school);
                            QUERY PLAN                            
------------------------------------------------------------------
 Hash Semi Join  (cost=1.09..2.21 rows=4 width=8)
   Hash Cond: (t1.id = school.cid)
   ->  Seq Scan on t1  (cost=0.00..1.06 rows=6 width=8)
   ->  Hash  (cost=1.04..1.04 rows=4 width=4)
         ->  Seq Scan on school  (cost=0.00..1.04 rows=4 width=4)
```
我们关闭`enable_hashjoin`再看一下执行计划：
```sql
postgres=# set enable_hashjoin=off;
SET
postgres=# explain select * from t1 where id in (select cid from school);
                             QUERY PLAN                             
--------------------------------------------------------------------
 Merge Semi Join  (cost=2.22..2.30 rows=4 width=8)
   Merge Cond: (t1.id = school.cid)
   ->  Sort  (cost=1.14..1.15 rows=6 width=8)
         Sort Key: t1.id
         ->  Seq Scan on t1  (cost=0.00..1.06 rows=6 width=8)
   ->  Sort  (cost=1.08..1.09 rows=4 width=4)
         Sort Key: school.cid
         ->  Seq Scan on school  (cost=0.00..1.04 rows=4 width=4)
```
关闭hash join后走的是merge join执行计划，我们再把enable_mergejoin关闭：
```sql
postgres=# set enable_mergejoin =off;
SET
postgres=# explain select * from t1 where id in (select cid from school);
                            QUERY PLAN                            
------------------------------------------------------------------
 Nested Loop Semi Join  (cost=0.00..2.47 rows=4 width=8)
   Join Filter: (t1.id = school.cid)
   ->  Seq Scan on t1  (cost=0.00..1.06 rows=6 width=8)
   ->  Materialize  (cost=0.00..1.06 rows=4 width=4)
         ->  Seq Scan on school  (cost=0.00..1.04 rows=4 width=4)
```
我们看到了逻辑优化后的结果，这里因为我自己构造的数据量很少，所以结果可能不是很明显，当数据量很大时结果对比会非常明显。我们再看一个例子：
```sql
postgres=# explain select * from t1 where id in (select cid from school where students > t1.id);
                          QUERY PLAN                          
--------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..4.23 rows=3 width=8)   /* 这里扫描节点的下层是子执行计划 */
   Filter: (SubPlan 1)
   SubPlan 1
     ->  Seq Scan on school  (cost=0.00..1.05 rows=1 width=4)
           Filter: (students > t1.id)
```
没有被优化为Semi Join。也就是说并不是所有的子连接都可以被上拉优化。是否被上拉取决于上拉后是否保持等价变换（正确性），有很多情况是不能被上拉的，另一个是上拉优化后是否执行效率更高，并不是所有的情况上拉后执行效率更高，具体的我们分析一下源码。


#### 源码分析
查询树生成执行计划是在`planner`调用完成，`planner`会调用`standard_planner`，我们继续往下看：
```c++
PlannedStmt *standard_planner(Query *parse, const char *query_string, int cursorOptions, ParamListInfo boundParams)
{
	  PlannedStmt *result;
    // 省略很多代码......
  
    /* primary planning entry point (may recurse for subqueries) */
    root = subquery_planner(glob, parse, NULL, false, tuple_fraction);  // 看这里

    final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
    best_path = get_cheapest_fractional_path(final_rel, tuple_fraction);

    top_plan = create_plan(root, best_path);  //创建计划
    // 省略很多代码......
  	return result; 
}

PlannerInfo *subquery_planner(PlannerGlobal *glob, Query *parse, PlannerInfo *parent_root, bool hasRecursion, double tuple_fraction)
{
    PlannerInfo *root;
    // 省略中间代码......
    /* Look for ANY and EXISTS SubLinks in WHERE and JOIN/ON clauses, and try
    * to transform them into joins.  Note that this step does not descend
    * into subqueries; if we pull up any subqueries below, their SubLinks are
    * processed just before pulling them up.*/
    if (parse->hasSubLinks)
      pull_up_sublinks(root);   // 子连接上拉优化，可以看到下面子查询也有上拉优化，区别是：子连接可以认为是出现在表达式中的子查询。
 
    pull_up_subqueries(root);   //  子查询上拉

    if (parse->setOperations)
      flatten_simple_union_all(root);   
	
	// 后面是其他优化，包括表达式优化等，这里不再列出.......
}
```
我们知道了子连接上拉优化的调用关系以及位置后具体分析一下其处理过程：
```c++
/* Attempt to pull up ANY and EXISTS SubLinks to be treated as	semijoins or anti-semijoins. */
void pull_up_sublinks(PlannerInfo *root)
{
	elog_node_display(NOTICE, "optimizer-before", root->parse->jointree, true);		// 可通过增加日志打印出优化前后的变化

	Node	   *jtnode;
	Relids		relids;

	jtnode = pull_up_sublinks_jointree_recurse(root,(Node *) root->parse->jointree,&relids);

	if (IsA(jtnode, FromExpr))
		root->parse->jointree = (FromExpr *) jtnode;
	else
		root->parse->jointree = makeFromExpr(list_make1(jtnode), NULL);

	elog_node_display(NOTICE, "optimizer-after", root->parse->jointree, true);
}
```
`pull_up_sublinks`核心实现是在`pull_up_sublinks_jointree_recurse`中完成的，我们继续往下看：
```c++
Node *pull_up_sublinks_jointree_recurse(PlannerInfo *root, Node *jtnode, Relids *relids)
{
    if (jtnode == NULL)
      *relids = NULL;
    else if (IsA(jtnode, RangeTblRef)) 
	{
      int	varno = ((RangeTblRef *) jtnode)->rtindex;
      *relids = bms_make_singleton(varno);        /* jtnode is returned unmodified */
    } 
    else if (IsA(jtnode, FromExpr)) 
    {       // 重点看这段代码，子连接上拉，需要重写FromExpr
      FromExpr   *f = (FromExpr *) jtnode;
      List	   *newfromlist = NIL;
      Relids		frelids = NULL;
      FromExpr   *newf;
      Node	   *jtlink;
      ListCell   *l;

      /* First, recurse to process children and collect their relids */
      foreach(l, f->fromlist)   // 对应处理SQL语句中的 from t1 
      {
        Node	   *newchild;
        Relids		childrelids;

        newchild = pull_up_sublinks_jointree_recurse(root,lfirst(l),&childrelids);
        newfromlist = lappend(newfromlist, newchild);     
        frelids = bms_join(frelids, childrelids);
      }
      /* Build the replacement FromExpr; no quals yet */
      newf = makeFromExpr(newfromlist, NULL);     // 需要重写where id in (select cid from school); 条件表达式，所以这里先赋NULL
      /* Set up a link representing the rebuilt jointree */
      jtlink = (Node *) newf;
	 // 重写where qual。需要重写成类似select * from t1 semi-join (select cid from school) as any_subquery where t1.id=any_subquery.cid这样的，注意没有semi-join这个语法。
      newf->quals = pull_up_sublinks_qual_recurse(root, f->quals,&jtlink, frelids, NULL, NULL);   

      *relids = frelids;
      jtnode = jtlink;
    }
    else if (IsA(jtnode, JoinExpr))
    {
       // 省略部分代码......
    }
    // 省略部分代码......
    return jtnode;
}
```
难点在`pull_up_sublinks_qual_recurse`的处理上，即怎么重写条件表达式。子连接有多种，我们只分析上面的语句，其他的代码相似，这里不再列出。

```c++
Node *pull_up_sublinks_qual_recurse(PlannerInfo *root, Node *node,Node **jtlink1, Relids available_rels1,Node **jtlink2, Relids available_rels2)
{
	if (node == NULL)
		return NULL;
	if (IsA(node, SubLink))
	{
		SubLink    *sublink = (SubLink *) node;
		JoinExpr   *j;
		Relids		child_rels;

		/* Is it a convertible ANY or EXISTS clause? */
		if (sublink->subLinkType == ANY_SUBLINK)    // 重点分析这种情况，其他情况类似
		{
			if ((j = convert_ANY_sublink_to_join(root, sublink,available_rels1)) != NULL)
			{
				/* Yes; insert the new join node into the join tree */
				j->larg = *jtlink1;
				*jtlink1 = (Node *) j;
				/* Recursively process pulled-up jointree nodes */
				j->rarg = pull_up_sublinks_jointree_recurse(root,j->rarg,&child_rels);

				/* Now recursively process the pulled-up quals.  Any inserted joins can get stacked onto either j->larg or j->rarg,depending on which rels they reference. */
				j->quals = pull_up_sublinks_qual_recurse(root,j->quals,&j->larg,available_rels1,&j->rarg,child_rels);
				/* Return NULL representing constant TRUE */
				return NULL;
			}
			if (available_rels2 != NULL &&(j = convert_ANY_sublink_to_join(root, sublink,available_rels2)) != NULL)
			{
				/* Yes; insert the new join node into the join tree */
				j->larg = *jtlink2;
				*jtlink2 = (Node *) j;
				/* Recursively process pulled-up jointree nodes */
				j->rarg = pull_up_sublinks_jointree_recurse(root,j->rarg,&child_rels);
				j->quals = pull_up_sublinks_qual_recurse(root,j->quals,&j->larg,available_rels2,&j->rarg,child_rels);
				/* Return NULL representing constant TRUE */
				return NULL;
			}
		}
		else if (sublink->subLinkType == EXISTS_SUBLINK)
		{
        // 省略代码.......
		}
		/* Else return it unmodified */
		return node;
	}
	if (is_notclause(node))
	{
		/* If the immediate argument of NOT is EXISTS, try to convert */
		SubLink    *sublink = (SubLink *) get_notclausearg((Expr *) node);
    // 省略代码......

		/* Else return it unmodified */
		return node;
	}
	if (is_andclause(node))
	{
		/* Recurse into AND clause */
    // 省略代码......
	}
	/* Stop if not an AND */
	return node;
}
```
还需要再思考一个问题，是所有的情况都可以进行上拉优化吗，显然不是的，前面的例子中已经看到了，那进行上拉优化需要那些限制条件呢？有3条：
- 子连接中的子查询不能使用其父查询中的任何Var类型变量
- 比较表达式（testexpr）中必须包含父查询中的任意一个Var类型变量
- 比较表达式中不能包含任何的易失函数

关于易失函数：PG中的函数有3种稳定性级别（VOLATILE、STABLE、IMMUTABLE），其中VOLATILE函数输入同样的参数会返回不同的结果，查询优化器通常不对含有VOLATILE函数的表达式进行优化，子连接不能提升。

```c++
// convert_ANY_sublink_to_join: try to convert an ANY SubLink to a join
JoinExpr *convert_ANY_sublink_to_join(PlannerInfo *root, SubLink *sublink,Relids available_rels)
{
	JoinExpr   *result;
	Query	   *parse = root->parse;
	Query	   *subselect = (Query *) sublink->subselect;
	Relids		upper_varnos;
	int			rtindex;
	ParseNamespaceItem *nsitem;
	RangeTblEntry *rte;
	RangeTblRef *rtr;
	List	   *subquery_vars;
	Node	   *quals;
	ParseState *pstate;

 	//ANY类型的子连接,子查询不能依赖父查询的任何变量,否则不能上拉
	if (contain_vars_of_level((Node *) subselect, 1))
		return NULL;

	//子查询的testexpr变量中必须含有父查询的某些Vars,否则不能上拉(join)
	upper_varnos = pull_varnos(root, sublink->testexpr);
	if (bms_is_empty(upper_varnos))
		return NULL;

	/* However, it can't refer to anything outside available_rels.*/
	if (!bms_is_subset(upper_varnos, available_rels))
		return NULL;

	// 组合操作符和左侧的表达式不能是易变(volatile)的,比如随机数等
	if (contain_volatile_functions(sublink->testexpr))
		return NULL;

	/* Create a dummy ParseState for addRangeTableEntryForSubquery */
	pstate = make_parsestate(NULL);

	// 子连接上拉后,子查询变成了上层的RTE,在这里构造,pull up the sub-select into upper range table.
	nsitem = addRangeTableEntryForSubquery(pstate,subselect, makeAlias("ANY_subquery", NIL),false,false);
	rte = nsitem->p_rte;
	parse->rtable = lappend(parse->rtable, rte);
	rtindex = list_length(parse->rtable);

	/* Form a RangeTblRef for the pulled-up sub-select.*/
	rtr = makeNode(RangeTblRef);
	rtr->rtindex = rtindex;

	/* Build a list of Vars representing the subselect outputs.*/
	subquery_vars = generate_subquery_vars(root,subselect->targetList,rtindex);

	/* Build the new join's qual expression 构造新的join的条件表达式*/
	quals = convert_testexpr(root, sublink->testexpr, subquery_vars);

	/* And finally, build the JoinExpr node.*/
	result = makeNode(JoinExpr);
	result->jointype = JOIN_SEMI;		// 变换为半连接
	result->isNatural = false;
	result->larg = NULL;		/* caller must fill this in */
	result->rarg = (Node *) rtr;
	result->usingClause = NIL;
	result->quals = quals;
	result->alias = NULL;
	result->rtindex = 0;		/* we don't need an RTE for it */

	return result;
}
```


#### 后记
这里只借助一条SQL语句分析了一下其逻辑优化的过程，其中还有很多内容并没有讲，比如EXIST类型子连接，相关子连接与非相关子连接的逻辑优化情况，这个可以结合执行计划和源码自行去分析，后面还会补充这方面的内容，这里不再展开。另外需要注意的是，有的子连接会提升为子查询，有的不会，（比如，ANY类型子连接一般会提升为子查询，EXIST类型子连接不会，直接提升为表与表的连接方式）。

针对上面的问题，举个例子：`select * from ttt where exists (select id from teacher where ttt.id > id);`
```sql
postgres=# explain (costs false) select * from ttt where exists (select id from teacher where ttt.id > id);
              QUERY PLAN              
--------------------------------------
 Nested Loop Semi Join
   Join Filter: (ttt.id > teacher.id)
   ->  Seq Scan on ttt
   ->  Materialize
         ->  Seq Scan on teacher
```
会被优化成类似`select * from ttt semi join teacher where ttt.id > id`的形式。

我们再举个例子：`select * from ttt where id > any (select id from teacher);`ANY子连接优化结果：
```sql
postgres=# explain (costs false) select * from ttt where id > any (select id from teacher);
              QUERY PLAN              
--------------------------------------
 Nested Loop Semi Join
   Join Filter: (ttt.id > teacher.id)
   ->  Seq Scan on ttt
   ->  Materialize
         ->  Seq Scan on teacher
```
在`pull_up_sublink`阶段，会将其优化为类似`select * from ttt semi join (select id from teacher) as ANY_subquery where ttt.id > ANY_subquery.id`的形式。也就是说将ANY类型的子连接转换为了子查询的形式，这里只做了一半，后一半是子查询优化，也就是会将子查询进行上拉。这一优化过程后面文章再继续分析子查询优化`pull_up_subquery`。所以说，子连接的优化学习还没有结束，后面还要继续分析学习子查询的逻辑优化过程。

还有一点：我们进行逻辑优化的目的是加快执行效率，并不是所有的子连接都可以进行上拉优化，也并不是所有的子连接都需要上拉（如果不上拉执行效率更高，比如非相关子查询的若干情况，这时候就不需要上拉），这一点要注意。

#### 附录
可通过在`pull_up_sublink`中增加日志`elog_node_display(NOTICE, "optimizer-before", root->parse->jointree, true);	`观察优化前后的变化。
##### IN类型转换日志
```sql
postgres=# select *from t1 where id in (select cid from school);
NOTICE:  optimizer-before:
DETAIL:     {FROMEXPR 
   :fromlist (
      {RANGETBLREF 
      :rtindex 1
      }
   )
   :quals 
      {SUBLINK 
      :subLinkType 2 
      :subLinkId 0 
      :testexpr 
         {OPEXPR 
         :opno 96 
         :opfuncid 65 
         :opresulttype 16 
         :opretset false 
         :opcollid 0 
         :inputcollid 0 
         :args (
            {VAR 
            :varno 1 
            :varattno 1 
            :vartype 23 
            :vartypmod -1 
            :varcollid 0 
            :varlevelsup 0 
            :varnosyn 1 
            :varattnosyn 1 
            :location 28
            }
            {PARAM 
            :paramkind 2 
            :paramid 1 
            :paramtype 23 
            :paramtypmod -1 
            :paramcollid 0 
            :location -1
            }
         )
         :location 31
         }
      :operName ("=")
      :subselect 
         {QUERY 
         :commandType 1 
         :querySource 0 
         :canSetTag true 
         :utilityStmt <> 
         :resultRelation 0 
         :hasAggs false 
         :hasWindowFuncs false 
         :hasTargetSRFs false 
         :hasSubLinks false 
         :hasDistinctOn false 
         :hasRecursive false 
         :hasModifyingCTE false 
         :hasForUpdate false 
         :hasRowSecurity false 
         :cteList <> 
         :rtable (
            {RTE 
            :alias <> 
            :eref 
               {ALIAS 
               :aliasname school 
               :colnames ("cid" "students")
               }
            :rtekind 0 
            :relid 16436 
            :relkind r 
            :rellockmode 1 
            :tablesample <> 
            :lateral false 
            :inh true 
            :inFromCl true 
            :requiredPerms 2 
            :checkAsUser 0 
            :selectedCols (b 8)
            :insertedCols (b)
            :updatedCols (b)
            :extraUpdatedCols (b)
            :securityQuals <>
            }
         )
         :jointree 
            {FROMEXPR 
            :fromlist (
               {RANGETBLREF 
               :rtindex 1
               }
            )
            :quals <>
            }
         :targetList (
            {TARGETENTRY 
            :expr 
               {VAR 
               :varno 1 
               :varattno 1 
               :vartype 23 
               :vartypmod -1 
               :varcollid 0 
               :varlevelsup 0 
               :varnosyn 1 
               :varattnosyn 1 
               :location 42
               }
            :resno 1 
            :resname cid 
            :ressortgroupref 0 
            :resorigtbl 16436 
            :resorigcol 1 
            :resjunk false
            }
         )
         :override 0 
         :onConflict <> 
         :returningList <> 
         :groupClause <> 
         :groupingSets <> 
         :havingQual <> 
         :windowClause <> 
         :distinctClause <> 
         :sortClause <> 
         :limitOffset <> 
         :limitCount <> 
         :limitOption 0 
         :rowMarks <> 
         :setOperations <> 
         :constraintDeps <> 
         :withCheckOptions <> 
         :stmt_location 0 
         :stmt_len 0 
         :hasRownum false 
         :hint_comment <>
         }
      :location 31
      }
   }
/*可以看到子连接转换为了JOIN形式*/
NOTICE:  optimizer-after:
DETAIL:     {FROMEXPR 
   :fromlist (
      {JOINEXPR 
      :jointype 4 		/* 4是JOIN_SEMI, 1 copy of each LHS row that has match(es) */
      :isNatural false 
      :larg 
         {FROMEXPR 
         :fromlist (
            {RANGETBLREF 
            :rtindex 1
            }
         )
         :quals <>
         }
      :rarg 
         {RANGETBLREF 
         :rtindex 2
         }
      :usingClause <> 
      :quals 
         {OPEXPR 
         :opno 96 
         :opfuncid 65 
         :opresulttype 16 
         :opretset false 
         :opcollid 0 
         :inputcollid 0 
         :args (
            {VAR 
            :varno 1 
            :varattno 1 
            :vartype 23 
            :vartypmod -1 
            :varcollid 0 
            :varlevelsup 0 
            :varnosyn 1 
            :varattnosyn 1 
            :location 28
            }
            {VAR 
            :varno 2 
            :varattno 1 
            :vartype 23 
            :vartypmod -1 
            :varcollid 0 
            :varlevelsup 0 
            :varnosyn 2 
            :varattnosyn 1 
            :location -1
            }
         )
         :location 31
         }
      :alias <> 
      :rtindex 0
      }
   )
   :quals <>
   }

```

##### ANY类型转换
我们再看一下ANY的转换：
```sql
postgres=# select * from ttt where id > any (select id from teacher);
NOTICE:  optimizer-before:
DETAIL:     {FROMEXPR 
   :fromlist (
      {RANGETBLREF 
      :rtindex 1
      }
   )
   :quals 
      {SUBLINK 
      :subLinkType 2 
      :subLinkId 0 
      :testexpr 
         {OPEXPR 
         :opno 521 
         :opfuncid 147 
         :opresulttype 16 
         :opretset false 
         :opcollid 0 
         :inputcollid 0 
         :args (
            {VAR 
            :varno 1 
            :varattno 1 
            :vartype 23 
            :vartypmod -1 
            :varcollid 0 
            :varlevelsup 0 
            :varnosyn 1 
            :varattnosyn 1 
            :location 24
            }
            {PARAM 
            :paramkind 2 
            :paramid 1 
            :paramtype 23 
            :paramtypmod -1 
            :paramcollid 0 
            :location -1
            }
         )
         :location 27
         }
      :operName (">")
      :subselect 
         {QUERY 
         :commandType 1 
         :querySource 0 
         :canSetTag true 
         :utilityStmt <> 
         :resultRelation 0 
         :hasAggs false 
         :hasWindowFuncs false 
         :hasTargetSRFs false 
         :hasSubLinks false 
         :hasDistinctOn false 
         :hasRecursive false 
         :hasModifyingCTE false 
         :hasForUpdate false 
         :hasRowSecurity false 
         :cteList <> 
         :rtable (
            {RTE 
            :alias <> 
            :eref 
               {ALIAS 
               :aliasname teacher 
               :colnames ("id" "name")
               }
            :rtekind 0 
            :relid 32829 
            :relkind r 
            :rellockmode 1 
            :tablesample <> 
            :lateral false 
            :inh true 
            :inFromCl true 
            :requiredPerms 2 
            :checkAsUser 0 
            :selectedCols (b 8)
            :insertedCols (b)
            :updatedCols (b)
            :extraUpdatedCols (b)
            :securityQuals <>
            }
         )
         :jointree 
            {FROMEXPR 
            :fromlist (
               {RANGETBLREF 
               :rtindex 1
               }
            )
            :quals <>
            }
         :targetList (
            {TARGETENTRY 
            :expr 
               {VAR 
               :varno 1 
               :varattno 1 
               :vartype 23 
               :vartypmod -1 
               :varcollid 0 
               :varlevelsup 0 
               :varnosyn 1 
               :varattnosyn 1 
               :location 41
               }
            :resno 1 
            :resname id 
            :ressortgroupref 0 
            :resorigtbl 32829 
            :resorigcol 1 
            :resjunk false
            }
         )
         :override 0 
         :onConflict <> 
         :returningList <> 
         :groupClause <> 
         :groupingSets <> 
         :havingQual <> 
         :windowClause <> 
         :distinctClause <> 
         :sortClause <> 
         :limitOffset <> 
         :limitCount <> 
         :limitOption 0 
         :rowMarks <> 
         :setOperations <> 
         :constraintDeps <> 
         :withCheckOptions <> 
         :stmt_location 0 
         :stmt_len 0 
         :hasRownum false 
         :hint_comment <>
         }
      :location 27
      }
   }
/* 转换后*/
NOTICE:  optimizer-after:
DETAIL:     {FROMEXPR 
   :fromlist (
      {JOINEXPR 
      :jointype 4 
      :isNatural false 
      :larg 
         {FROMEXPR 
         :fromlist (
            {RANGETBLREF 
            :rtindex 1
            }
         )
         :quals <>
         }
      :rarg 
         {RANGETBLREF 
         :rtindex 2
         }
      :usingClause <> 
      :quals 
         {OPEXPR 
         :opno 521 
         :opfuncid 147 
         :opresulttype 16 
         :opretset false 
         :opcollid 0 
         :inputcollid 0 
         :args (
            {VAR 
            :varno 1 
            :varattno 1 
            :vartype 23 
            :vartypmod -1 
            :varcollid 0 
            :varlevelsup 0 
            :varnosyn 1 
            :varattnosyn 1 
            :location 24
            }
            {VAR 
            :varno 2 
            :varattno 1 
            :vartype 23 
            :vartypmod -1 
            :varcollid 0 
            :varlevelsup 0 
            :varnosyn 2 
            :varattnosyn 1 
            :location -1
            }
         )
         :location 27
         }
      :alias <> 
      :rtindex 0
      }
   )
   :quals <>
   }
```