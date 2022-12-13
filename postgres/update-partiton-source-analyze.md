### update分区表源码分析
以`update tc set c = 1`语句为例，分析update分区表的过程。 分区表的update与非分区表的流程是不一样的。我们先看一下其执行计划的不同：
```sql
--非分区表update执行计划
postgres@postgres=# explain update dtea set id = 0;
                          QUERY PLAN                          
--------------------------------------------------------------
 Update on dtea  (cost=0.00..19.00 rows=900 width=68)
   ->  Seq Scan on dtea  (cost=0.00..19.00 rows=900 width=68)
(2 rows)

--分区表update执行计划
postgres@postgres=# explain update tc set c = 1;
                               QUERY PLAN                               
------------------------------------------------------------------------
 Update on tc  (cost=0.00..2.06 rows=6 width=16)
   Update on tc_1_prt_tc1 tc_1
   Update on tc_1_prt_tc2 tc_2
   ->  Seq Scan on tc_1_prt_tc1 tc_1  (cost=0.00..1.03 rows=3 width=16)
   ->  Seq Scan on tc_1_prt_tc2 tc_2  (cost=0.00..1.03 rows=3 width=16)
(5 rows)
```

### 源码分析：
关于解析层的代码分析这里不再列出，重点是优化器以及执行器中对分区部分的代码分析：
```c++
// 优化器的调用：
void exec_simple_query(const char *query_string)
--> pg_parse_query(query_string)
--> pg_analyze_and_rewrite()        // 语义分析查询重写
--> pg_plan_queries()				// 基于查询树生成查询计划树
      --> pg_plan_query()
	      --> planner()  /* call the optimizer */
			--> standard_planner()
// 计划执行部分
--> PortalStart()
--> PortalRun()
```

#### 优化器部分源码分析
```c++
//--------由查询器Query生成执行计划PlannedStmt----------------
PlannedStmt *standard_planner(Query *parse, const char *query_string, int cursorOptions,ParamListInfo boundParams)
{
      PlannerInfo *root;
      RelOptInfo *final_rel;
	Path	   *best_path;
      Plan	   *top_plan;
      // 省略 并行检查等代码......

      /* primary planning entry point (may recurse for subqueries) */
	root = subquery_planner(glob, parse, NULL,false, tuple_fraction);

	/* Select best Path and turn it into a Plan */
	final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
	best_path = get_cheapest_fractional_path(final_rel, tuple_fraction);

	top_plan = create_plan(root, best_path);  // 由最佳路径生成执行计划 Path --> Plan

	// 整理计划树，生成的完整计划经过计划树整理后就可以交给查询执行器去执行了
	top_plan = set_plan_references(root, top_plan);
      // 省略并行等代码......
}

// 由Query生成PlannerInfo
PlannerInfo *subquery_planner(PlannerGlobal *glob, Query *parse,PlannerInfo *parent_root,bool hasRecursion, double tuple_fraction)
{
      if (parse->hasSubLinks)
		pull_up_sublinks(root);
      
      pull_up_subqueries(root);

      // 省略其他很多代码......

	/* Do the main planning.  
     * If we have an inherited target relation, that needs special processing, 
     * else go straight to grouping_planner.*/
	if (parse->resultRelation &&rt_fetch(parse->resultRelation, parse->rtable)->inh)
		inheritance_planner(root);    // update/delete分区表在这里进行处理,里面会调用grouping_planner
	else
		grouping_planner(root, false, tuple_fraction);

}

// 重点分析这里开始的代码：
void inheritance_planner(PlannerInfo *root)
{
	Query	   *parse = root->parse;
	int			top_parentRTindex = parse->resultRelation;
	List	   *select_rtable;
	List	   *select_appinfos;
	List	   *child_appinfos;
	List	   *old_child_rtis;
	List	   *new_child_rtis;
	Bitmapset  *subqueryRTindexes;
	Index		next_subquery_rti;
	int			nominalRelation = -1;
	Index		rootRelation = 0;
	List	   *final_rtable = NIL;
	List	   *final_rowmarks = NIL;
	List	   *final_appendrels = NIL;
	int			save_rel_array_size = 0;
	RelOptInfo **save_rel_array = NULL;
	AppendRelInfo **save_append_rel_array = NULL;
	List	   *subpaths = NIL;
	List	   *subroots = NIL;
	List	   *resultRelations = NIL;
	List	   *withCheckOptionLists = NIL;
	List	   *returningLists = NIL;
	List	   *rowMarks;
	RelOptInfo *final_rel;
	ListCell   *lc;
	ListCell   *lc2;
	Index		rti;
	RangeTblEntry *parent_rte;
	Bitmapset  *parent_relids;
	Query	  **parent_parses;

	// 也就是说只有分区表 update/delete语句才走这里进行处理
	Assert(parse->commandType == CMD_UPDATE || parse->commandType == CMD_DELETE);	/* Should only get here for UPDATE or DELETE */

	/** We generate a modified instance of the original Query for each target
	 * relation, plan that, and put all the plans into a list that will be
	 * controlled by a single ModifyTable node.  All the instances share the
	 * same rangetable, but each instance must have its own set of subquery
	 * RTEs within the finished rangetable because (1) they are likely to get
	 * scribbled on during planning, and (2) it's not inconceivable that
	 * subqueries could get planned differently in different cases.  We need
	 * not create duplicate copies of other RTE kinds, in particular not the
	 * target relations, because they don't have either of those issues.  Not
	 * having to duplicate the target relations is important because doing so
	 * (1) would result in a rangetable of length O(N^2) for N targets, with
	 * at least O(N^3) work expended here; and (2) would greatly complicate
	 * management of the rowMarks list.
	 *
	 * To begin with, generate a bitmapset of the relids of the subquery RTEs.*/
	subqueryRTindexes = NULL;
	rti = 1;
	foreach(lc, parse->rtable)
	{
		RangeTblEntry *rte = lfirst_node(RangeTblEntry, lc);

		if (rte->rtekind == RTE_SUBQUERY)
			subqueryRTindexes = bms_add_member(subqueryRTindexes, rti);
		rti++;
	}

	/* If the parent RTE is a partitioned table, we should use that as the
	 * nominal target relation, because the RTEs added for partitioned tables
	 * (including the root parent) as child members of the inheritance set do
	 * not appear anywhere else in the plan, so the confusion explained below
	 * for non-partitioning inheritance cases is not possible.*/
	parent_rte = rt_fetch(top_parentRTindex, parse->rtable);
	Assert(parent_rte->inh);
	if (parent_rte->relkind == RELKIND_PARTITIONED_TABLE)
	{
		nominalRelation = top_parentRTindex;
		rootRelation = top_parentRTindex;
	}

	/* Before generating the real per-child-relation plans, do a cycle of
	 * planning as though the query were a SELECT.  
     * The objective here is to find out which child relations need to be processed, using the same
	 * expansion and pruning logic as for a SELECT.  
     *
     *  We'll then pull out the RangeTblEntry-s generated for the child rels, and make use of the
	 * AppendRelInfo entries for them to guide the real planning. 
     *  
     * (This is rather inefficient; we could perhaps stop short of making a full Path
	 * tree.  But this whole function is inefficient and slated for
	 * destruction, so let's not contort query_planner for that.) */
	{
		PlannerInfo *subroot;
		subroot = makeNode(PlannerInfo);
		memcpy(subroot, root, sizeof(PlannerInfo));		/** Flat-copy the PlannerInfo to prevent modification of the original.*/

		/* Make a deep copy of the parsetree for this planning cycle to mess
		 * around with, and change it to look like a SELECT.  (Hack alert: the
		 * target RTE still has updatedCols set if this is an UPDATE, so that
		 * expand_partitioned_rtentry will correctly update subroot->partColsUpdated.)*/
		subroot->parse = copyObject(root->parse);   
		subroot->parse->commandType = CMD_SELECT;   // 先以查询的方式生成计划，目的是看那个分区需要被处理
		subroot->parse->resultRelation = 0;

		/* Ensure the subroot has its own copy of the original append_rel_list, since it'll be scribbled on.  (Note that at this point, the list only contains AppendRelInfos for flattened UNION ALL subqueries.)*/
		subroot->append_rel_list = copyObject(root->append_rel_list);

		/* Generate Path(s) for accessing this result relation */
		grouping_planner(subroot, true, 0.0 /* retrieve all tuples */ );  // 对父表生成PlannerInfo

		/* Extract the info we need. */		// 提取我们以select方式获取的信息，为的是利用分区的优化，裁减掉不需要处理的分区，只保留需要处理的分区
		select_rtable = subroot->parse->rtable;
		select_appinfos = subroot->append_rel_list;
	}

    foreach(lc, select_appinfos)      // 对分区表进行处理
	{
		AppendRelInfo *appinfo = lfirst_node(AppendRelInfo, lc);
		RangeTblEntry *child_rte;

		child_appinfos = lappend(child_appinfos, appinfo);		/* remember relevant AppendRelInfos for use below */
		parse->rtable = lappend(parse->rtable, child_rte);		/* and append it to the original rtable */
	}    

	/* And now we can get on with generating a plan for each child table.*/
	foreach(lc, child_appinfos)	// 分别对每个子分区生成子执行计划，最后生成ModifyTable执行计划
	{
		AppendRelInfo *appinfo = lfirst_node(AppendRelInfo, lc);
		Index		this_subquery_rti = next_subquery_rti;
		Query	   *parent_parse;
		PlannerInfo *subroot;
		RangeTblEntry *child_rte;
		RelOptInfo *sub_final_rel;
		Path	   *subpath;

		/* expand_inherited_rtentry() always processes a parent before any of
		 * that parent's children, so the parent query for this relation
		 * should already be available. */
		parent_parse = parent_parses[appinfo->parent_relid];

		/*
		 * We need a working copy of the PlannerInfo so that we can control
		 * propagation of information back to the main copy.
		 */
		subroot = makeNode(PlannerInfo);
		memcpy(subroot, root, sizeof(PlannerInfo));     // 需要注意这里，copy的方式

		/* Generate modified query with this rel as target.  We first apply
		 * adjust_appendrel_attrs, which copies the Query and changes
		 * references to the parent RTE to refer to the current child RTE,
		 * then fool around with subquery RTEs.*/
		subroot->parse = (Query *)adjust_appendrel_attrs(subroot,(Node *) parent_parse, 1, &appinfo);


		/* If this child is further partitioned, remember it as a parent.
		 * Since a partitioned table does not have any data, we don't need to
		 * create a plan for it, and we can stop processing it here.  We do,
		 * however, need to remember its modified PlannerInfo for use when
		 * processing its children, since we'll update their varnos based on
		 * the delta from immediate parent to child, not from top to child.
		 *
		 * Note: a very non-obvious point is that we have not yet added
		 * duplicate subquery RTEs to the subroot's rtable.  We mustn't,
		 * because then its children would have two sets of duplicates,
		 * confusing matters.*/
		if (child_rte->inh)
		{
			Assert(child_rte->relkind == RELKIND_PARTITIONED_TABLE);
			parent_parses[appinfo->child_relid] = subroot->parse;
			continue;
		}

		/*
		 * Set the nominal target relation of the ModifyTable node if not
		 * already done.  If the target is a partitioned table, we already set
		 * nominalRelation to refer to the partition root, above.  For
		 * non-partitioned inheritance cases, we'll use the first child
		 * relation (even if it's excluded) as the nominal target relation.
		 * Because of the way expand_inherited_rtentry works, that should be
		 * the RTE representing the parent table in its role as a simple
		 * member of the inheritance set.
		 *
		 * It would be logically cleaner to *always* use the inheritance
		 * parent RTE as the nominal relation; but that RTE is not otherwise
		 * referenced in the plan in the non-partitioned inheritance case.
		 * Instead the duplicate child RTE created by expand_inherited_rtentry
		 * is used elsewhere in the plan, so using the original parent RTE
		 * would give rise to confusing use of multiple aliases in EXPLAIN
		 * output for what the user will think is the "same" table.  OTOH,
		 * it's not a problem in the partitioned inheritance case, because
		 * there is no duplicate RTE for the parent.
		 */
		if (nominalRelation < 0)
			nominalRelation = appinfo->child_relid;

		/*
		 * As above, each child plan run needs its own append_rel_list and
		 * rowmarks, which should start out as pristine copies of the
		 * originals.  There can't be any references to UPDATE/DELETE target
		 * rels in them; but there could be subquery references, which we'll
		 * fix up in a moment.
		 */
		subroot->append_rel_list = copyObject(root->append_rel_list);
		subroot->rowMarks = copyObject(root->rowMarks);

		/*
		 * If this isn't the first child Query, adjust Vars and jointree
		 * entries to reference the appropriate set of subquery RTEs.
		 */
		if (final_rtable != NIL && subqueryRTindexes != NULL)
		{
			int			oldrti = -1;

			while ((oldrti = bms_next_member(subqueryRTindexes, oldrti)) >= 0)
			{
				Index		newrti = next_subquery_rti++;

				ChangeVarNodes((Node *) subroot->parse, oldrti, newrti, 0);
				ChangeVarNodes((Node *) subroot->append_rel_list,
							   oldrti, newrti, 0);
				ChangeVarNodes((Node *) subroot->rowMarks, oldrti, newrti, 0);
			}
		}

		/* Generate Path(s) for accessing this result relation */
		grouping_planner(subroot, true, 0.0 /* retrieve all tuples */ );        // 对分区表生成PlannerInfo

		/* * Select cheapest path in case there's more than one.  We always run
		 * modification queries to conclusion, so we care only for the*/
		sub_final_rel = fetch_upper_rel(subroot, UPPERREL_FINAL, NULL);
		set_cheapest(sub_final_rel);
		subpath = sub_final_rel->cheapest_total_path;

		/* * If this child rel was excluded by constraint exclusion, exclude it from the result plan.*/
		if (IS_DUMMY_REL(sub_final_rel))
			continue;

		/*
		 * If this is the first non-excluded child, its post-planning rtable
		 * becomes the initial contents of final_rtable; otherwise, copy its
		 * modified subquery RTEs into final_rtable, to ensure we have sane
		 * copies of those.  Also save the first non-excluded child's version
		 * of the rowmarks list; we assume all children will end up with
		 * equivalent versions of that.  Likewise for append_rel_list.
		 */
		if (final_rtable == NIL)
		{
			final_rtable = subroot->parse->rtable;
			final_rowmarks = subroot->rowMarks;
			final_appendrels = subroot->append_rel_list;
		}
            else
            {
			if (subqueryRTindexes != NULL)
			{
				int			oldrti = -1;

				while ((oldrti = bms_next_member(subqueryRTindexes, oldrti)) >= 0)
				{
					Index		newrti = this_subquery_rti++;
					RangeTblEntry *subqrte;
					ListCell   *newrticell;

					subqrte = rt_fetch(newrti, subroot->parse->rtable);
					newrticell = list_nth_cell(final_rtable, newrti - 1);
					lfirst(newrticell) = subqrte;
				}
			}
		}

		/* We need to collect all the RelOptInfos from all child plans into
		 * the main PlannerInfo, since setrefs.c will need them.  We use the
		 * last child's simple_rel_array, so we have to propagate forward the
		 * RelOptInfos that were already built in previous children.*/
		Assert(subroot->simple_rel_array_size >= save_rel_array_size);
		for (rti = 1; rti < save_rel_array_size; rti++)
		{
			RelOptInfo *brel = save_rel_array[rti];

			if (brel)
				subroot->simple_rel_array[rti] = brel;
		}
		save_rel_array_size = subroot->simple_rel_array_size;
		save_rel_array = subroot->simple_rel_array;
		save_append_rel_array = subroot->append_rel_array;

		/* Make sure any initplans from this rel get into the outer list. Note
		 * we're effectively assuming all children generate the same init_plans.*/
		root->init_plans = subroot->init_plans;

		/* Build list of sub-paths */
		subpaths = lappend(subpaths, subpath);

		/* Build list of modified subroots, too */
		subroots = lappend(subroots, subroot);

		/* Build list of target-relation RT indexes */
		resultRelations = lappend_int(resultRelations, appinfo->child_relid);

	} 

	/* Result path must go into outer query's FINAL upperrel */
	final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);

	/* We don't currently worry about setting final_rel's consider_parallel
	 * flag in this case, nor about allowing FDWs or create_upper_paths_hook
	 * to get control here. */
	if (subpaths == NIL)
	{
		/* * We managed to exclude every child rel, so generate a dummy path
		 * representing the empty set.  Although it's clear that no data will
		 * be updated or deleted, we will still need to have a ModifyTable
		 * node so that any statement triggers are executed.  (This could be
		 * cleaner if we fixed nodeModifyTable.c to support zero child nodes,
		 * but that probably wouldn't be a net win.)*/
		Path	   *dummy_path;

		/* tlist processing never got done, either */
		root->processed_tlist = preprocess_targetlist(root);
		final_rel->reltarget = create_pathtarget(root, root->processed_tlist);

		/* Make a dummy path, cf set_dummy_rel_pathlist() */
		dummy_path = (Path *) create_append_path(NULL, final_rel, NIL, NIL,NIL, NULL, 0, false,NIL, -1);

		/* These lists must be nonempty to make a valid ModifyTable node */
		subpaths = list_make1(dummy_path);
		subroots = list_make1(root);
		resultRelations = list_make1_int(parse->resultRelation);
	}
	else
	{
		/* Put back the final adjusted rtable into the master copy of the
		 * Query.  (We mustn't do this if we found no non-excluded children,
		 * since we never saved an adjusted rtable at all.)*/
		parse->rtable = final_rtable;
		root->simple_rel_array_size = save_rel_array_size;
		root->simple_rel_array = save_rel_array;
		root->append_rel_array = save_append_rel_array;

		/* Must reconstruct master's simple_rte_array, too */
		root->simple_rte_array = (RangeTblEntry **)
			palloc0((list_length(final_rtable) + 1) * sizeof(RangeTblEntry *));
		rti = 1;
		foreach(lc, final_rtable)
		{
			RangeTblEntry *rte = lfirst_node(RangeTblEntry, lc);
			root->simple_rte_array[rti++] = rte;
		}

		/* Put back adjusted rowmarks and appendrels, too */
		root->rowMarks = final_rowmarks;
		root->append_rel_list = final_appendrels;
	}

	/* Create Path representing a ModifyTable to do the UPDATE/DELETE work */
	add_path(final_rel, (Path *)
			 create_modifytable_path(root, final_rel,
									 parse->commandType,
									 parse->canSetTag,
									 nominalRelation,
									 rootRelation,
									 root->partColsUpdated,
									 resultRelations,
									 subpaths,
									 subroots,
									 withCheckOptionLists,
									 returningLists,
									 rowMarks,
									 NULL,
									 assign_special_exec_param(root)));             
}


void grouping_planner(PlannerInfo *root, bool inheritance_update, double tuple_fraction)
{
      /* Generate the best unsorted and presorted paths for the scan/join
	 * portion of this Query, ie the processing represented by the
	 * FROM/WHERE clauses.  (Note there may not be any presorted paths.)
	 * We also generate (in standard_qp_callback) pathkey representations
	 * of the query's sort clause, distinct clause, etc. */
	current_rel = query_planner(root, standard_qp_callback, &qp_extra);
	
      apply_scanjoin_target_to_paths(root, current_rel, scanjoin_targets,scanjoin_targets_contain_srfs,scanjoin_target_parallel_safe,scanjoin_target_same_exprs);

	if (parse->sortClause)
	{
		current_rel = create_ordered_paths(root,current_rel,final_target,final_target_parallel_safe, have_postponed_srfs ? -1.0 : limit_tuples);

	}      
}
```

```c++
grouping_planner()
// 为查询中的扫描/连接部分生成最优的未排序/预排序路径(如FROM/WHERE语句表示的处理过程)
--> query_planner
      --> add_base_rels_to_query(root, (Node *) parse->jointree);
      --> add_other_rels_to_query(root);
      --> make_one_rel(root, joinlist);   //生成RelOptInfo
            --> set_base_rel_sizes(root);
            --> set_base_rel_pathlists(root);
                  --> set_rel_pathlist(root, rel, rti, root->simple_rte_array[rti]);
                        --> set_append_rel_pathlist(root, rel, rti, rte);
                              --> add_paths_to_append_rel(root, rel, live_childrels);/* Add paths to the append relation. */
            --> make_rel_from_joinlist(root, joinlist);
--> apply_scanjoin_target_to_paths
--> create_ordered_paths
--> create_limit_path
```


#### 执行器部分源码分析
重点分析`ExecModifyTable`和`ExecUpdate`的实现。 需要注意，执行计划与算子执行理解的时候需要结合在一起理解，执行计划的生成很大程度上是算子的实现决定的。`ExecModifyTable`算子去更新表的时候，每个分区各自生成子执行计划，然后依次取每个分区的执行计划，依次执行。所以在执行计划生成层面，是需要为每个子分区单独生成各自的子分区执行计划。

```c++
/* ----------------------------------------------------------------
 *	   ExecModifyTable
 *		Perform table modifications as required, and return RETURNING results if needed.
 * ---------------------------------------------------------------- */
static TupleTableSlot *ExecModifyTable(PlanState *pstate)
{
	ModifyTableState *node = castNode(ModifyTableState, pstate);
	PartitionTupleRouting *proute = node->mt_partition_tuple_routing;
	EState	   *estate = node->ps.state;
	CmdType		operation = node->operation;
	ResultRelInfo *saved_resultRelInfo;
	ResultRelInfo *resultRelInfo;
	PlanState  *subplanstate;
	JunkFilter *junkfilter;
	TupleTableSlot *slot;
	TupleTableSlot *planSlot;
	ItemPointer tupleid;
	ItemPointerData tuple_ctid;
	HeapTupleData oldtupdata;
	HeapTuple	oldtuple;

	CHECK_FOR_INTERRUPTS();

	/* This should NOT get called during EvalPlanQual; we should have passed a
	 * subplan tree to EvalPlanQual, instead.  Use a runtime test not just
	 * Assert because this condition is easy to miss in testing. */
	if (estate->es_epq_active != NULL)
		elog(ERROR, "ModifyTable should not be called during EvalPlanQual");

	/* If we've already completed processing, don't try to do more.  We need
	 * this test because ExecPostprocessPlan might call us an extra time, and
	 * our subplan's nodes aren't necessarily robust against being called
	 * extra times.*/
	if (node->mt_done)
		return NULL;

	/* On first call, fire BEFORE STATEMENT triggers before proceeding.*/
	if (node->fireBSTriggers)
	{
		fireBSTriggers(node);
		node->fireBSTriggers = false;
	}

	/* Preload local variables */
	resultRelInfo = node->resultRelInfo + node->mt_whichplan;
	subplanstate = node->mt_plans[node->mt_whichplan];
	junkfilter = resultRelInfo->ri_junkFilter;

	/* es_result_relation_info must point to the currently active result relation while we are within this ModifyTable node.  
	 * Even though ModifyTable nodes can't be nested statically, they can be nested
	 * dynamically (since our subplan could include a reference to a modifying
	 * CTE).  So we have to save and restore the caller's value.*/
	saved_resultRelInfo = estate->es_result_relation_info;
	estate->es_result_relation_info = resultRelInfo;

	/* Fetch rows from subplan(s), and execute the required table modification for each row.*/
	for (;;)
	{
		/* Reset the per-output-tuple exprcontext.  This is needed because
		 * triggers expect to use that context as workspace.  It's a bit ugly
		 * to do this below the top level of the plan, however.  We might need to rethink this later.*/
		ResetPerTupleExprContext(estate);

		/* Reset per-tuple memory context used for processing on conflict and
		 * returning clauses, to free any expression evaluation storage allocated in the previous cycle. */
		if (pstate->ps_ExprContext)
			ResetExprContext(pstate->ps_ExprContext);

		planSlot = ExecProcNode(subplanstate);
		if (TupIsNull(planSlot))
		{
			/* advance to next subplan if any */
			node->mt_whichplan++;	// 分区表的update，每个分区分布对应一个subplan，当执行完一个分区再执行下一个分区
			if (node->mt_whichplan < node->mt_nplans)
			{
				resultRelInfo++;
				subplanstate = node->mt_plans[node->mt_whichplan];
				junkfilter = resultRelInfo->ri_junkFilter;
				estate->es_result_relation_info = resultRelInfo;
				EvalPlanQualSetPlan(&node->mt_epqstate, subplanstate->plan, node->mt_arowmarks[node->mt_whichplan]);
				/* Prepare to convert transition tuples from this child. */
				if (node->mt_transition_capture != NULL) {
					node->mt_transition_capture->tcs_map = tupconv_map_for_subplan(node, node->mt_whichplan);
				}
				if (node->mt_oc_transition_capture != NULL) {
					node->mt_oc_transition_capture->tcs_map = tupconv_map_for_subplan(node, node->mt_whichplan);
				}
				continue;
			}
			else
				break;
		}

		/* Ensure input tuple is the right format for the target relation.*/
		if (node->mt_scans[node->mt_whichplan]->tts_ops != planSlot->tts_ops) {
			ExecCopySlot(node->mt_scans[node->mt_whichplan], planSlot);
			planSlot = node->mt_scans[node->mt_whichplan];
		}

		/* If resultRelInfo->ri_usesFdwDirectModify is true, all we need to do here is compute the RETURNING expressions.*/
		if (resultRelInfo->ri_usesFdwDirectModify)
		{
			Assert(resultRelInfo->ri_projectReturning);
			slot = ExecProcessReturning(resultRelInfo->ri_projectReturning, RelationGetRelid(resultRelInfo->ri_RelationDesc), NULL, planSlot);

			estate->es_result_relation_info = saved_resultRelInfo;
			return slot;
		}

		EvalPlanQualSetSlot(&node->mt_epqstate, planSlot);
		slot = planSlot;

		tupleid = NULL;
		oldtuple = NULL;
		if (junkfilter != NULL)
		{
			/* extract the 'ctid' or 'wholerow' junk attribute.*/
			if (operation == CMD_UPDATE || operation == CMD_DELETE)
			{
				char		relkind;
				Datum		datum;
				bool		isNull;

				relkind = resultRelInfo->ri_RelationDesc->rd_rel->relkind;
				if (relkind == RELKIND_RELATION || relkind == RELKIND_MATVIEW)
				{
					datum = ExecGetJunkAttribute(slot,junkfilter->jf_junkAttNo,&isNull);
					/* shouldn't ever get a null result... */
					if (isNull)
						elog(ERROR, "ctid is NULL");

					tupleid = (ItemPointer) DatumGetPointer(datum);
					tuple_ctid = *tupleid;	/* be sure we don't free ctid!! */
					tupleid = &tuple_ctid;
				}
				/* Use the wholerow attribute, when available, to reconstruct the old relation tuple.*/
				else if (AttributeNumberIsValid(junkfilter->jf_junkAttNo))
				{
					datum = ExecGetJunkAttribute(slot,junkfilter->jf_junkAttNo,&isNull);
					/* shouldn't ever get a null result... */
					if (isNull)
						elog(ERROR, "wholerow is NULL");

					oldtupdata.t_data = DatumGetHeapTupleHeader(datum);
					oldtupdata.t_len = HeapTupleHeaderGetDatumLength(oldtupdata.t_data);
					ItemPointerSetInvalid(&(oldtupdata.t_self));
					/* Historically, view triggers see invalid t_tableOid. */
					oldtupdata.t_tableOid = (relkind == RELKIND_VIEW) ? InvalidOid : RelationGetRelid(resultRelInfo->ri_RelationDesc);
					oldtuple = &oldtupdata;
				}
				else
					Assert(relkind == RELKIND_FOREIGN_TABLE);
			}

			/* apply the junkfilter if needed. */
			if (operation != CMD_DELETE)
				slot = ExecFilterJunk(junkfilter, slot);
		}

		switch (operation)
		{
			case CMD_INSERT:
				if (proute)				/* Prepare for tuple routing if needed. */
					slot = ExecPrepareTupleRouting(node, estate, proute, resultRelInfo, slot);
				slot = ExecInsert(node, slot, planSlot, NULL, estate->es_result_relation_info, estate, node->canSetTag);
				if (proute)				/* Revert ExecPrepareTupleRouting's state change. */
					estate->es_result_relation_info = resultRelInfo;
				break;
			case CMD_UPDATE:
				slot = ExecUpdate(node, tupleid, oldtuple, slot, planSlot,
								  &node->mt_epqstate, estate, node->canSetTag);
				break;
			case CMD_DELETE:
				slot = ExecDelete(node, tupleid, oldtuple, planSlot,
								  &node->mt_epqstate, estate,
								  true, node->canSetTag, false /* changingPart */ , NULL, NULL);
				break;
			default:
				elog(ERROR, "unknown operation");
				break;
		}

		/* If we got a RETURNING result, return it to caller.  We'll continue the work on next call.*/
		if (slot) {
			estate->es_result_relation_info = saved_resultRelInfo;
			return slot;
		}
	}

	estate->es_result_relation_info = saved_resultRelInfo;	/* Restore es_result_relation_info before exiting */
	fireASTriggers(node);	/* We're done, but fire AFTER STATEMENT triggers before exiting.*/

	node->mt_done = true;

	return NULL;
}
```


```c++
/* ----------------------------------------------------------------
 *		ExecUpdate
 *
 *		note: we can't run UPDATE queries with transactions off because UPDATEs are actually INSERTs and our
 *		scan will mistakenly loop forever, updating the tuple it just inserted..  This should be fixed but until it
 *		is, we don't want to get stuck in an infinite loop which corrupts your database..
 *
 *		When updating a table, tupleid identifies the tuple to update and oldtuple is NULL.  
 *
 *		Returns RETURNING result if any, otherwise NULL.
 * ----------------------------------------------------------------*/
static TupleTableSlot *
ExecUpdate(ModifyTableState *mtstate,
		   ItemPointer tupleid,
		   HeapTuple oldtuple,
		   TupleTableSlot *slot,
		   TupleTableSlot *planSlot,
		   EPQState *epqstate,
		   EState *estate,
		   bool canSetTag)
{
	ResultRelInfo *resultRelInfo;
	Relation	resultRelationDesc;
	TM_Result	result;
	TM_FailureData tmfd;
	List	   *recheckIndexes = NIL;
	TupleConversionMap *saved_tcs_map = NULL;

	/* abort the operation if not running transactions*/
	if (IsBootstrapProcessingMode())
		elog(ERROR, "cannot UPDATE during bootstrap");

	ExecMaterializeSlot(slot);

	/* get information on the (current) result relation*/
	resultRelInfo = estate->es_result_relation_info;
	resultRelationDesc = resultRelInfo->ri_RelationDesc;

	/* BEFORE ROW UPDATE Triggers */
	if (resultRelInfo->ri_TrigDesc && resultRelInfo->ri_TrigDesc->trig_update_before_row)
	{
		if (!ExecBRUpdateTriggers(estate, epqstate, resultRelInfo, tupleid, oldtuple, slot))
			return NULL;		/* "do nothing" */
	}

	/* INSTEAD OF ROW UPDATE Triggers */
	if (resultRelInfo->ri_TrigDesc && resultRelInfo->ri_TrigDesc->trig_update_instead_row)
	{
		if (!ExecIRUpdateTriggers(estate, resultRelInfo, oldtuple, slot))
			return NULL;		/* "do nothing" */
	}
	else if (resultRelInfo->ri_FdwRoutine)
	{
		/* Compute stored generated columns*/
		if (resultRelationDesc->rd_att->constr && resultRelationDesc->rd_att->constr->has_generated_stored)
			ExecComputeStoredGenerated(estate, slot, CMD_UPDATE);

		/* update in foreign table: let the FDW do it*/
		slot = resultRelInfo->ri_FdwRoutine->ExecForeignUpdate(estate, resultRelInfo, slot, planSlot);

		if (slot == NULL)		/* "do nothing" */
			return NULL;

		/* AFTER ROW Triggers or RETURNING expressions might reference the
		 * tableoid column, so (re-)initialize tts_tableOid before evaluating them. */
		slot->tts_tableOid = RelationGetRelid(resultRelationDesc);
	}
	else
	{
		LockTupleMode lockmode;
		bool		partition_constraint_failed;
		bool		update_indexes;

		/* Constraints might reference the tableoid column, so (re-)initialize
		 * tts_tableOid before evaluating them.*/
		slot->tts_tableOid = RelationGetRelid(resultRelationDesc);

		/* Compute stored generated columns*/
		if (resultRelationDesc->rd_att->constr && resultRelationDesc->rd_att->constr->has_generated_stored)
			ExecComputeStoredGenerated(estate, slot, CMD_UPDATE);

		/*
		 * Check any RLS UPDATE WITH CHECK policies
		 *
		 * If we generate a new candidate tuple after EvalPlanQual testing, we
		 * must loop back here and recheck any RLS policies and constraints.
		 * (We don't need to redo triggers, however.  If there are any BEFORE
		 * triggers then trigger.c will have done table_tuple_lock to lock the
		 * correct tuple, so there's no need to do them again.) */
lreplace:;

		/* ensure slot is independent, consider e.g. EPQ */
		ExecMaterializeSlot(slot);

		/* If partition constraint fails, this row might get moved to another
		 * partition, in which case we should check the RLS CHECK policy just
		 * before inserting into the new partition, rather than doing it here.
		 * This is because a trigger on that partition might again change the
		 * row.  So skip the WCO checks if the partition constraint fails. */
		partition_constraint_failed = resultRelInfo->ri_PartitionCheck && !ExecPartitionCheck(resultRelInfo, slot, estate, false);

		if (!partition_constraint_failed && resultRelInfo->ri_WithCheckOptions != NIL)
		{
			/* ExecWithCheckOptions() will skip any WCOs which are not of the kind we are looking for at this point. */
			ExecWithCheckOptions(WCO_RLS_UPDATE_CHECK, resultRelInfo, slot, estate);
		}

		/* If a partition check failed, try to move the row into the right partition.*/
		if (partition_constraint_failed)
		{
			bool		tuple_deleted;
			TupleTableSlot *ret_slot;
			TupleTableSlot *orig_slot = slot;
			TupleTableSlot *epqslot = NULL;
			PartitionTupleRouting *proute = mtstate->mt_partition_tuple_routing;
			int			map_index;
			TupleConversionMap *tupconv_map;

			/* Disallow an INSERT ON CONFLICT DO UPDATE that causes the
			 * original row to migrate to a different partition.  Maybe this
			 * can be implemented some day, but it seems a fringe feature with
			 * little redeeming value.*/
			if (((ModifyTable *) mtstate->ps.plan)->onConflictAction == ONCONFLICT_UPDATE)
				ereport(ERROR,
						(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
						 errmsg("invalid ON UPDATE specification"),
						 errdetail("The result tuple would appear in a different partition than the original tuple.")));

			/* When an UPDATE is run on a leaf partition, we will not have
			 * partition tuple routing set up. In that case, fail with
			 * partition constraint violation error.*/
			if (proute == NULL)
				ExecPartitionCheckEmitError(resultRelInfo, slot, estate);

			/* Row movement, part 1.  Delete the tuple, but skip RETURNING
			 * processing. We want to return rows from INSERT.*/
			ExecDelete(mtstate, tupleid, oldtuple, planSlot, epqstate, estate, false, false /* canSetTag */ , true /* changingPart */ , &tuple_deleted, &epqslot);

			/* For some reason if DELETE didn't happen (e.g. trigger prevented
			 * it, or it was already deleted by self, or it was concurrently
			 * deleted by another transaction), then we should skip the insert
			 * as well; otherwise, an UPDATE could cause an increase in the
			 * total number of rows across all partitions, which is clearly wrong.
			 *
			 * For a normal UPDATE, the case where the tuple has been the
			 * subject of a concurrent UPDATE or DELETE would be handled by
			 * the EvalPlanQual machinery, but for an UPDATE that we've
			 * translated into a DELETE from this partition and an INSERT into
			 * some other partition, that's not available, because CTID chains
			 * can't span relation boundaries.  We mimic the semantics to a
			 * limited extent by skipping the INSERT if the DELETE fails to
			 * find a tuple. This ensures that two concurrent attempts to
			 * UPDATE the same tuple at the same time can't turn one tuple
			 * into two, and that an UPDATE of a just-deleted tuple can't resurrect it.*/
			if (!tuple_deleted)
			{
				/*
				 * epqslot will be typically NULL.  But when ExecDelete()
				 * finds that another transaction has concurrently updated the
				 * same row, it re-fetches the row, skips the delete, and
				 * epqslot is set to the re-fetched tuple slot. In that case,
				 * we need to do all the checks again.
				 */
				if (TupIsNull(epqslot))
					return NULL;
				else
				{
					slot = ExecFilterJunk(resultRelInfo->ri_junkFilter, epqslot);
					goto lreplace;
				}
			}

			/* Updates set the transition capture map only when a new subplan
			 * is chosen.  But for inserts, it is set for each row. So after
			 * INSERT, we need to revert back to the map created for UPDATE;
			 * otherwise the next UPDATE will incorrectly use the one created
			 * for INSERT.  So first save the one created for UPDATE. */
			if (mtstate->mt_transition_capture)
				saved_tcs_map = mtstate->mt_transition_capture->tcs_map;

			/* resultRelInfo is one of the per-subplan resultRelInfos.  So we
			 * should convert the tuple into root's tuple descriptor, since
			 * ExecInsert() starts the search from root.  The tuple conversion
			 * map list is in the order of mtstate->resultRelInfo[], so to
			 * retrieve the one for this resultRel, we need to know the
			 * position of the resultRel in mtstate->resultRelInfo[]. */
			map_index = resultRelInfo - mtstate->resultRelInfo;
			Assert(map_index >= 0 && map_index < mtstate->mt_nplans);
			tupconv_map = tupconv_map_for_subplan(mtstate, map_index);
			if (tupconv_map != NULL)
				slot = execute_attr_map_slot(tupconv_map->attrMap, slot, mtstate->mt_root_tuple_slot);

			/* Prepare for tuple routing, making it look like we're inserting into the root. */
			Assert(mtstate->rootResultRelInfo != NULL);
			slot = ExecPrepareTupleRouting(mtstate, estate, proute, mtstate->rootResultRelInfo, slot);

			ret_slot = ExecInsert(mtstate, slot, planSlot,
								  orig_slot, resultRelInfo,
								  estate, canSetTag);

			/* Revert ExecPrepareTupleRouting's node change. */
			estate->es_result_relation_info = resultRelInfo;
			if (mtstate->mt_transition_capture)
			{
				mtstate->mt_transition_capture->tcs_original_insert_tuple = NULL;
				mtstate->mt_transition_capture->tcs_map = saved_tcs_map;
			}

			return ret_slot;
		}

		/* Check the constraints of the tuple.  We've already checked the
		 * partition constraint above; however, we must still ensure the tuple
		 * passes all other constraints, so we will call ExecConstraints() and
		 * have it validate all remaining checks.*/
		if (resultRelationDesc->rd_att->constr)
			ExecConstraints(resultRelInfo, slot, estate);

		/* replace the heap tuple
		 *
		 * Note: if es_crosscheck_snapshot isn't InvalidSnapshot, we check
		 * that the row to be updated is visible to that snapshot, and throw a
		 * can't-serialize error if not. This is a special-case behavior
		 * needed for referential integrity updates in transaction-snapshot mode transactions. */
		result = table_tuple_update(resultRelationDesc, tupleid, slot, estate->es_output_cid,
									estate->es_snapshot, estate->es_crosscheck_snapshot, true /* wait for commit */ ,&tmfd, &lockmode, &update_indexes);

		switch (result)
		{
			case TM_SelfModified:

				/* The target tuple was already updated or deleted by the
				 * current command, or by a later command in the current
				 * transaction.  The former case is possible in a join UPDATE
				 * where multiple tuples join to the same target tuple. This
				 * is pretty questionable, but Postgres has always allowed it:
				 * we just execute the first update action and ignore
				 * additional update attempts.
				 *
				 * The latter case arises if the tuple is modified by a
				 * command in a BEFORE trigger, or perhaps by a command in a
				 * volatile function used in the query.  In such situations we
				 * should not ignore the update, but it is equally unsafe to
				 * proceed.  We don't want to discard the original UPDATE
				 * while keeping the triggered actions based on it; and we
				 * have no principled way to merge this update with the
				 * previous ones.  So throwing an error is the only safe
				 * course.
				 *
				 * If a trigger actually intends this type of interaction, it
				 * can re-execute the UPDATE (assuming it can figure out how)
				 * and then return NULL to cancel the outer update.*/
				if (tmfd.cmax != estate->es_output_cid)
					ereport(ERROR,(errcode(ERRCODE_TRIGGERED_DATA_CHANGE_VIOLATION),
							 errmsg("tuple to be updated was already modified by an operation triggered by the current command"),
							 errhint("Consider using an AFTER trigger instead of a BEFORE trigger to propagate changes to other rows.")));

				/* Else, already updated by self; nothing to do */
				return NULL;

			case TM_Ok:
				break;

			case TM_Updated:
				{
					TupleTableSlot *inputslot;
					TupleTableSlot *epqslot;

					if (IsolationUsesXactSnapshot())
						ereport(ERROR,(errcode(ERRCODE_T_R_SERIALIZATION_FAILURE),errmsg("could not serialize access due to concurrent update")));

					/* Already know that we're going to need to do EPQ, so fetch tuple directly into the right slot. */
					inputslot = EvalPlanQualSlot(epqstate, resultRelationDesc,resultRelInfo->ri_RangeTableIndex);

					result = table_tuple_lock(resultRelationDesc, tupleid, estate->es_snapshot,inputslot, estate->es_output_cid, lockmode, LockWaitBlock, TUPLE_LOCK_FLAG_FIND_LAST_VERSION,&tmfd);

					switch (result)
					{
						case TM_Ok:
							Assert(tmfd.traversed);
							epqslot = EvalPlanQual(epqstate, resultRelationDesc, resultRelInfo->ri_RangeTableIndex, inputslot);
							if (TupIsNull(epqslot))
								/* Tuple not passing quals anymore, exiting... */
								return NULL;

							slot = ExecFilterJunk(resultRelInfo->ri_junkFilter, epqslot);
							goto lreplace;

						case TM_Deleted:
							/* tuple already deleted; nothing to do */
							return NULL;

						case TM_SelfModified:

							/*
							 * This can be reached when following an update chain from a tuple updated by another session,
							 * reaching a tuple that was already updated in this transaction. If previously modified by
							 * this command, ignore the redundant update, otherwise error out.
							 *
							 * See also TM_SelfModified response to table_tuple_update() above.*/
							if (tmfd.cmax != estate->es_output_cid)
								ereport(ERROR,(errcode(ERRCODE_TRIGGERED_DATA_CHANGE_VIOLATION),
										 errmsg("tuple to be updated was already modified by an operation triggered by the current command"),errhint("Consider using an AFTER trigger instead of a BEFORE trigger to propagate changes to other rows.")));
							return NULL;

						default:
							/* see table_tuple_lock call in ExecDelete() */
							elog(ERROR, "unexpected table_tuple_lock status: %u", result);
							return NULL;
					}
				}

				break;

			case TM_Deleted:
				if (IsolationUsesXactSnapshot())
					ereport(ERROR,(errcode(ERRCODE_T_R_SERIALIZATION_FAILURE),errmsg("could not serialize access due to concurrent delete")));
				/* tuple already deleted; nothing to do */
				return NULL;

			default:
				elog(ERROR, "unrecognized table_tuple_update status: %u",
					 result);
				return NULL;
		}

		/* insert index entries for tuple if necessary */
		if (resultRelInfo->ri_NumIndices > 0 && update_indexes)
			recheckIndexes = ExecInsertIndexTuples(slot, estate, false, NULL, NIL);
	}

	if (canSetTag)
		(estate->es_processed)++;

	/* AFTER ROW UPDATE Triggers */
	ExecARUpdateTriggers(estate, resultRelInfo, tupleid, oldtuple, slot,recheckIndexes,mtstate->operation == CMD_INSERT ?mtstate->mt_oc_transition_capture : mtstate->mt_transition_capture);

	list_free(recheckIndexes);

	/* Check any WITH CHECK OPTION constraints from parent views.  We are
	 * required to do this after testing all constraints and uniqueness
	 * violations per the SQL spec, so we do it after actually updating the
	 * record in the heap and all indexes.
	 *
	 * ExecWithCheckOptions() will skip any WCOs which are not of the kind we
	 * are looking for at this point. */
	if (resultRelInfo->ri_WithCheckOptions != NIL)
		ExecWithCheckOptions(WCO_VIEW_CHECK, resultRelInfo, slot, estate);

	if (resultRelInfo->ri_projectReturning)	/* Process RETURNING if present */
		return ExecProcessReturning(resultRelInfo->ri_projectReturning,RelationGetRelid(resultRelationDesc),slot, planSlot);

	return NULL;
}
```
再往下就是涉及到存储引擎的部分了，我们重点看一下其对外的接口输入参数。重点是这4个参数：
- relation - table to be modified (caller must hold suitable lock) （要更新的那个表）
- otid - TID of old tuple to be replaced （要更新的元组ID，对应的是老的元组，更新后相当于是插入一条新元组，老元组的tid值要更新为新的tid值）
- slot - newly constructed tuple data to store （新元组的值）
- cid - update command ID (used for visibility test, and stored into cmax/cmin if successful) （cid值，事务相关）
执行器层面的更新算子是建立在存储引擎提供的底层`table_tuple_update`接口之上的。是我们编写`ExecUpdate`以及`ExecModifyTable`的基础。
```c++
/*
 * Update a tuple.

 * Input parameters:
 *	relation - table to be modified (caller must hold suitable lock)
 *	otid - TID of old tuple to be replaced
 *	slot - newly constructed tuple data to store
 *	cid - update command ID (used for visibility test, and stored into cmax/cmin if successful)
 *	crosscheck - if not InvalidSnapshot, also check old tuple against this
 *	wait - true if should wait for any conflicting update to commit/abort

 * Output parameters:
 *	tmfd - filled in failure cases (see below)
 *	lockmode - filled with lock mode acquired on tuple
 *  update_indexes - in success cases this is set to true if new index entries are required for this tuple
 *
 * Normal, successful return value is TM_Ok, which means we did actually update it. */
static inline TM_Result
table_tuple_update(Relation rel, ItemPointer otid, TupleTableSlot *slot, CommandId cid, 
				   Snapshot snapshot, Snapshot crosscheck, bool wait, TM_FailureData *tmfd, LockTupleMode *lockmode, bool *update_indexes)
{
	return rel->rd_tableam->tuple_update(rel, otid, slot, cid, 
										 snapshot, crosscheck, wait, tmfd, lockmode, update_indexes);
}
```