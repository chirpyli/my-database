### Postgres源码分析——绑定变量
这里分析一下函数中应用绑定变量的问题，但实际应用场景中，不推荐这么使用。
```sql
prepare divplan2(int,int) as select div($1,$2);
execute divplan2(4,2);
```

#### 语法解析
分别分析prepare语句以及execute语句。
gram.y中定义
```c++
/*******************************************************************
 *
 *		QUERY:
 *				PREPARE <plan_name> [(args, ...)] AS <query>
 *
 *****************************************************************/

PrepareStmt: PREPARE name prep_type_clause AS PreparableStmt
				{
					PrepareStmt *n = makeNode(PrepareStmt);
					n->name = $2;
					n->argtypes = $3;
					n->query = $5;
					$$ = (Node *) n;
				}
		;

prep_type_clause: '(' type_list ')'			{ $$ = $2; }
				| /* EMPTY */				{ $$ = NIL; }
		;

PreparableStmt:
			SelectStmt
			| InsertStmt
			| UpdateStmt
			| DeleteStmt					/* by default all are $$=$1 */
			| MergeStmt
		;

/******************************************************************
 *
 * EXECUTE <plan_name> [(params, ...)]
 *
 *****************************************************************/
ExecuteStmt: EXECUTE name execute_param_clause
				{
					ExecuteStmt *n = makeNode(ExecuteStmt);
					n->name = $2;
					n->params = $3;
					$$ = (Node *) n;
				}

execute_param_clause: '(' expr_list ')'				{ $$ = $2; }
					| /* EMPTY */					{ $$ = NIL; }
					;
```

#### 主流程

prepare，execute语句代码主流程如下，
```c++
main
--> PostmasterMain
    --> ServerLoop
        --> BackendStartup
            --> BackendRun
                --> PostgresMain
                    --> exec_simple_query
```
```c++
exec_simple_query
--> pg_parse_query
--> pg_analyze_and_rewrite
--> pg_plan_queries
--> PortalStart
--> PortalRun
    --> PortalRunUtiliey
        --> standard_ProcessUtility
            --> PrepareQuery      // prepare语句
            --> ExecuteQuery      // execute语句
            --> DeallocateQuery
```

##### prepare语句
`prepare divplan2(int,int) as select div($1,$2);`
Prepare语句主流程如下：

```c++
PrepareQuery
--> CreateCachedPlan  // 创建CachedPlanSource
--> parse_analyze_varparams  // 语义分析
    --> parse_variable_parameters
    --> transformTopLevelStmt
        --> transformStmt
            --> transformSelectStmt
                --> transformTargetList
                    --> transformTargetEntry
                        --> transformExpr
                            --> transformFuncCall   // 解析函数调用
                                --> transformParamRef  // 解析绑定变量参数，构造Param节点,作为函数参数
                                    --> variable_paramref_hook  
                                --> ParseFuncOrColumn  // 构造FuncExpr， 获取函数oid,返回值类型oid
                                    --> func_get_detail
                                        --> FuncnameGetCandidates // 通过函数名获取后续函数列表
                                        --> func_match_argtypes
                                        --> func_select_candidate
                                    --> make_fn_arguments
                                        --> coerce_type
                                            --> variable_coerce_param_hook
    --> check_variable_parameters
--> QueryRewrite    // 查询重写
--> CompleteCachedPlan   // 创建plan cache entry
--> StorePreparedStatement // 存到哈希表中
    --> InitQueryHashTable /* Initialize the hash table, if necessary */
        --> hash_create    
    --> hash_search  /* Add entry to hash table */
    --> SaveCachedPlan // save a cached plan permanently
        --> ReleaseGenericPlan
        --> dlist_push_tail
```
##### execute语句
`execute divplan2(4,2);`
Execute主流程如下：
```c++
ExecuteQuery
--> FetchPreparedStatement // 哈希表中查找是否有已缓存的执行计划
    --> hash_search
--> CreateExecutorState
--> EvaluateParams   // 获取绑定变量参数，返回ParamListInfo
    --> ExecPrepareExprList
        --> ExecPrepareExpr
            --> expression_planner
                --> eval_const_expressions
            --> ExecInitExpr
    --> makeParamList  获取到参数值4,2
--> CreateNewPortal
    --> CreatePortal
--> GetCachedPlan
    --> RevalidateCachedQuery
    --> choose_custom_plan  // choose whether to use custom or generic plan
    --> if (!customplan) // 走generic plan
        --> if (CheckCachedPlan)
                // 直接获取已有的有效generic plan
            else
            --> BuildCachedPlan
                --> pg_plan_queries
                    --> pg_plan_query
                        --> planner
                            --> standard_planner
                                --> subquery_planner
                                    --> preprocess_expression   // 复合常量化简，直接调用函数
                                        --> eval_const_expressions
                                            --> simplify_function  
                                                --> evaluate_function
                                                    --> evaluate_expr
            --> cached_plan_cost
    --> if (customplan)  // 走custom plan
        --> BuildCachedPlan
            --> pg_plan_queries
                --> pg_plan_query
                    --> planner
                        --> standard_planner
                            --> subquery_planner
                                --> preprocess_qual_conditions
                                    --> eval_const_expressions
                            --> create_plan

--> PortalDefineQuery
--> PortalStart
    --> ExecutorStart
        --> InitPlan
--> PortalRun
    --> ExecutorRun
        --> standard_ExecutorRun
            --> ExecutePlan

--> PortalDrop
    --> ExecutorEnd
        --> standard_ExecutorEnd
            --> ExecEndPlan
    --> PortalReleaseCachedPlan  /* drop cached plan reference, if any */
```

