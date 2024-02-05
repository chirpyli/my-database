### PostgreSQL源码分析——创建分区表
分区表，可以认为是逻辑上一张表，但实际上是将逻辑上的一张表，分割为了多个物理表，每个物理表是逻辑表中的一部分，组合起来就是一张表。所以在实现分区表时，实际上是创建了多张物理表，但是逻辑上抽象为了一张表。下面我们就分析一下PostgreSQL中创建分区表的源码，示例代码如下：
```sql
-- 创建父表
create table ta(
    id    int,
    value date
) partition by range (value);

-- 创建分区表
create table ta_2020 partition of ta for values from ('2020-01-01') to ('2021-01-01');      -- from inclusive   to exclusize
create table ta_2021 partition of ta for values from ('2021-01-01') to ('2022-01-01');
```
建表后，我们可通过查看pg_class系统表，与分区表相关的字段
- relkind ：表类型（r = 普通表， i = 索引， S = 序列， t = TOAST表， v = 视图， m = 物化视图， c = 组合类型， f = 外部表， p = 分区表， I = 分区索引）
- relispartition ：如果表或索引是一个分区，则为真
- relpartbound ：表示一个分区（见relispartition），分区边界的内部表达
我们看ta以及子分区t_2020的pg_class系统表信息如下：
```sql
--查看ta表信息
postgres=# select relname,relkind,relispartition,relpartbound from pg_class where relname = 'ta';
-[ RECORD 1 ]--+---
relname        | ta
relkind        | p    -- 表示此表为分区表
relispartition | f
relpartbound   | 
-- 查看子分区信息
postgres=# select relname,relkind,relispartition,relpartbound from pg_class where relname = 'ta_2020';
-[ RECORD 1 ]
relname        | ta_2020
relkind        | r    -- 表示普通表
relispartition | t    -- 表示是子分区
relpartbound   | {PARTITIONBOUNDSPEC :strategy r :is_default false :modulus 0 :remainder 0 :listdatums <> :lowerdatums ({PARTITIONRANGEDATUM :kind 0 :value {CONST :consttype 1082 :consttypmod -1 :constcollid 0 :constlen 4 :constbyval true :constisnull false :location 54 :constvalue 4 [ -119 28 0 0 0 0 0 0 ]} :location 54}) :upperdatums ({PARTITIONRANGEDATUM :kind 0 :value {CONST :consttype 1082 :consttypmod -1 :constcollid 0 :constlen 4 :constbyval true :constisnull false :location 72 :constvalue 4 [ -9 29 0 0 0 0 0 0 ]} :location 72}) :location 48}  -- 分区边界的内部表达，节点树
```
还可查看系统表pg_partitioned_table，查看分区表信息：
```sql
postgres=# select * from pg_partitioned_table;
-[ RECORD 1 ]-+------
partrelid     | 16394    -- 分区表oid
partstrat     | r        -- 分区策略  h = 哈希分区表，l = 列表分区表，r = 范围分区表
partnatts     | 1        -- 分区键中的列数
partdefid     | 0        -- 这个分区表的默认分区的pg_class项的OID，如果这个分区表没有默认分区则为零。
partattrs     | 2        -- 这是一个长度为partnatts值的数组，它指示哪些表列是分区键的组成部分。 例如，值1 3表示第一个和第三个表列组成了分区键。这个数组中的零表示对应的分区键列是一个表达式而不是简单的列引用。
partclass     | 3122     -- 对于分区键中的每一个列，这个域包含要使用的操作符类的OID
partcollation | 0        -- 对于分区键中的每一个列，这个域包含要用于分区的排序规则的OID，如果该列不是一种可排序数据类型则为零。
partexprs     |          -- 非简单列引用的分区键列的表达式树（以nodeToString()的表达方式）。 这是一个列表，partattrs中每一个零项都有一个元素。如果所有分区键列都是简单列引用，则这个域为空。
```
另外，可查看系统表pg_inherits（继承表相关信息）相关信息：
```sql
postgres=# select * from pg_inherits;
-[ RECORD 1 ]----+------
inhrelid         | 16397   -- 子表oid
inhparent        | 16394   -- 父表oid
inhseqno         | 1
inhdetachpending | f
-[ RECORD 2 ]----+------
inhrelid         | 16400
inhparent        | 16394
inhseqno         | 1
inhdetachpending | f

```
一定要理解这些分区表相关的系统表，熟悉建表流程的话就会知道，建一个表，实际上就是在系统表中插入表相关的元信息，让数据库系统知道这个表名字是什么，有哪些列，列数据类型是什么，有哪些约束等等，将各自的信息存储在pg_class，pg_attribute等系统表中，分区表也是一样，要存储分布键信息，分布类型，分区的边界是什么，子分区有哪些，这些信息。


以PostgreSQL-15.1版本代码进行分析：
建表主流程如下：
```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
--> pg_analyze_and_rewrite_fixedparams
    --> parse_analyze_fixedparams
        --> transformStmt
    --> pg_rewrite_query
--> pg_plan_queries
--> PortalStart
--> PortalRun
    --> standard_ProcessUtility
        --> ProcessUtilitySlow
            --> transformCreateStmt // 处理CreateStmt，处理列，约束等
                --> transformColumnDefinition // 处理列
                    --> transformConstraintAttrs
                --> transformTableConstraint  // 处理约束
                --> transformIndexConstraints  // 处理索引
                    --> transformIndexConstraint
                --> transformFKConstraints     // 处理外键
                --> transformCheckConstraints   // 处理check约束
                --> transformExtendedStatistics
            --> DefineRelation
--> PortalDrop
```


下面我们逐步进行分析。

#### 解析部分
因为`CREATE TABLE`属于Utility语句，其语义分析阶段其实是在执行器阶段执行的，我们后续再分析。解析阶段主流程如下：
```c++
exec_simple_query
--> pg_parse_query
    --> raw_parser
--> pg_analyze_and_rewrite_fixedparams
    --> parse_analyze_fixedparams
        --> transformStmt
    --> pg_rewrite_query
--> pg_plan_queries
```
最关键的是构造语法解析树`CreateStmt`:
```c++
typedef struct CreateStmt
{
	NodeTag		type;
	RangeVar   *relation;		/* relation to create */
	List	   *tableElts;		/* column definitions (list of ColumnDef) */
	List	   *inhRelations;	/* relations to inherit from (list of
								 * RangeVar) */
	PartitionBoundSpec *partbound;	/* FOR VALUES clause */ // 分区表相关
	PartitionSpec *partspec;	/* PARTITION BY clause */  // 分区表相关
	TypeName   *ofTypename;		/* OF typename */
	List	   *constraints;	/* constraints (list of Constraint nodes) */
	List	   *options;		/* options from WITH clause */
	OnCommitAction oncommit;	/* what do we do at COMMIT? */
	char	   *tablespacename; /* table space to use, or NULL */
	char	   *accessMethod;	/* table access method */
	bool		if_not_exists;	/* just do nothing if it already exists? */
} CreateStmt;
```
其中与分区表相关的字段是`partbound`和`partspec`，即：`PartitionSpec`和`PartitionBoundSpec`：
```c++
/*
 * PartitionSpec - parse-time representation of a partition key specification
 *
 * This represents the key space we will be partitioning on.
 */
typedef struct PartitionSpec
{
	NodeTag		type;
	char	   *strategy;		/* partitioning strategy ('hash', 'list' or
								 * 'range') */ //分区策略
	List	   *partParams;		/* List of PartitionElems */  // 指定分区键
	int			location;		/* token location, or -1 if unknown */
} PartitionSpec;

/* Internal codes for partitioning strategies */
#define PARTITION_STRATEGY_HASH		'h'
#define PARTITION_STRATEGY_LIST		'l'
#define PARTITION_STRATEGY_RANGE	'r'

/*
 * PartitionBoundSpec - a partition bound specification
 *
 * This represents the portion of the partition key space assigned to a
 * particular partition.  These are stored on disk in pg_class.relpartbound.
 */
struct PartitionBoundSpec
{
	NodeTag		type;

	char		strategy;		/* see PARTITION_STRATEGY codes above */
	bool		is_default;		/* is it a default partition bound? */

	/* Partitioning info for HASH strategy: */
	int			modulus;
	int			remainder;

	/* Partitioning info for LIST strategy: */
	List	   *listdatums;		/* List of Consts (or A_Consts in raw tree) */

	/* Partitioning info for RANGE strategy: */
	List	   *lowerdatums;	/* List of PartitionRangeDatums */
	List	   *upperdatums;	/* List of PartitionRangeDatums */

	int			location;		/* token location, or -1 if unknown */
};

/*
 * PartitionElem - parse-time representation of a single partition key
 *
 * expr can be either a raw expression tree or a parse-analyzed expression.
 * We don't store these on-disk, though.
 */
typedef struct PartitionElem
{
	NodeTag		type;
	char	   *name;			/* name of column to partition on, or NULL */
	Node	   *expr;			/* expression to partition on, or NULL */
	List	   *collation;		/* name of collation; NIL = default */
	List	   *opclass;		/* name of desired opclass; NIL = default */
	int			location;		/* token location, or -1 if unknown */
} PartitionElem;
```

在gram.y中建表语法定义如下：
```c++
/*****************************************************************************
 *
 *		QUERY :
 *				CREATE TABLE relname
 *
 *****************************************************************************/

CreateStmt:	CREATE OptTemp TABLE qualified_name '(' OptTableElementList ')'
			OptInherit OptPartitionSpec table_access_method_clause OptWith
			OnCommitOption OptTableSpace
				{
					CreateStmt *n = makeNode(CreateStmt);

					$4->relpersistence = $2;
					n->relation = $4;
					n->tableElts = $6;
					n->inhRelations = $8;
					n->partspec = $9;
					n->ofTypename = NULL;
					n->constraints = NIL;
					n->accessMethod = $10;
					n->options = $11;
					n->oncommit = $12;
					n->tablespacename = $13;
					n->if_not_exists = false;
					$$ = (Node *) n;
				}
		| CREATE OptTemp TABLE qualified_name PARTITION OF qualified_name
			OptTypedTableElementList PartitionBoundSpec OptPartitionSpec
			table_access_method_clause OptWith OnCommitOption OptTableSpace
				{
					CreateStmt *n = makeNode(CreateStmt);

					$4->relpersistence = $2;
					n->relation = $4;
					n->tableElts = $8;
					n->inhRelations = list_make1($7);
					n->partbound = $9;
					n->partspec = $10;
					n->ofTypename = NULL;
					n->constraints = NIL;
					n->accessMethod = $11;
					n->options = $12;
					n->oncommit = $13;
					n->tablespacename = $14;
					n->if_not_exists = false;
					$$ = (Node *) n;
				}
        // ...
		;

/* Optional partition key specification */
OptPartitionSpec: PartitionSpec	{ $$ = $1; }
			| /*EMPTY*/			{ $$ = NULL; }
		;

PartitionSpec: PARTITION BY ColId '(' part_params ')'
				{
					PartitionSpec *n = makeNode(PartitionSpec);

					n->strategy = $3;
					n->partParams = $5;
					n->location = @1;

					$$ = n;
				}
		;

part_params:	part_elem						{ $$ = list_make1($1); }
			| part_params ',' part_elem			{ $$ = lappend($1, $3); }
		;

part_elem: ColId opt_collate opt_class
				{
					PartitionElem *n = makeNode(PartitionElem);

					n->name = $1;
					n->expr = NULL;
					n->collation = $2;
					n->opclass = $3;
					n->location = @1;
					$$ = n;
				}
			| func_expr_windowless opt_collate opt_class
				{
					PartitionElem *n = makeNode(PartitionElem);

					n->name = NULL;
					n->expr = $1;
					n->collation = $2;
					n->opclass = $3;
					n->location = @1;
					$$ = n;
				}
			| '(' a_expr ')' opt_collate opt_class
				{
					PartitionElem *n = makeNode(PartitionElem);

					n->name = NULL;
					n->expr = $2;
					n->collation = $4;
					n->opclass = $5;
					n->location = @1;
					$$ = n;
				}
		;

PartitionBoundSpec:
			/* a HASH partition */
			FOR VALUES WITH '(' hash_partbound ')'
				{
					ListCell   *lc;
					PartitionBoundSpec *n = makeNode(PartitionBoundSpec);

					n->strategy = PARTITION_STRATEGY_HASH;
					n->modulus = n->remainder = -1;

					foreach (lc, $5)
					{
						DefElem    *opt = lfirst_node(DefElem, lc);

						if (strcmp(opt->defname, "modulus") == 0)
						{
							if (n->modulus != -1)
								ereport(ERROR,
										(errcode(ERRCODE_DUPLICATE_OBJECT),
										 errmsg("modulus for hash partition provided more than once"),
										 parser_errposition(opt->location)));
							n->modulus = defGetInt32(opt);
						}
						else if (strcmp(opt->defname, "remainder") == 0)
						{
							if (n->remainder != -1)
								ereport(ERROR,
										(errcode(ERRCODE_DUPLICATE_OBJECT),
										 errmsg("remainder for hash partition provided more than once"),
										 parser_errposition(opt->location)));
							n->remainder = defGetInt32(opt);
						}
						else
							ereport(ERROR,
									(errcode(ERRCODE_SYNTAX_ERROR),
									 errmsg("unrecognized hash partition bound specification \"%s\"",
											opt->defname),
									 parser_errposition(opt->location)));
					}

					if (n->modulus == -1)
						ereport(ERROR,
								(errcode(ERRCODE_SYNTAX_ERROR),
								 errmsg("modulus for hash partition must be specified")));
					if (n->remainder == -1)
						ereport(ERROR,
								(errcode(ERRCODE_SYNTAX_ERROR),
								 errmsg("remainder for hash partition must be specified")));

					n->location = @3;

					$$ = n;
				}

			/* a LIST partition */
			| FOR VALUES IN_P '(' expr_list ')'
				{
					PartitionBoundSpec *n = makeNode(PartitionBoundSpec);

					n->strategy = PARTITION_STRATEGY_LIST;
					n->is_default = false;
					n->listdatums = $5;
					n->location = @3;

					$$ = n;
				}

			/* a RANGE partition */
			| FOR VALUES FROM '(' expr_list ')' TO '(' expr_list ')'
				{
					PartitionBoundSpec *n = makeNode(PartitionBoundSpec);

					n->strategy = PARTITION_STRATEGY_RANGE;
					n->is_default = false;
					n->lowerdatums = $5;
					n->upperdatums = $9;
					n->location = @3;

					$$ = n;
				}

			/* a DEFAULT partition */
			| DEFAULT
				{
					PartitionBoundSpec *n = makeNode(PartitionBoundSpec);

					n->is_default = true;
					n->location = @1;

					$$ = n;
				}
		;

hash_partbound_elem:
		NonReservedWord Iconst
			{
				$$ = makeDefElem($1, (Node *) makeInteger($2), @1);
			}
		;

hash_partbound:
		hash_partbound_elem
			{
				$$ = list_make1($1);
			}
		| hash_partbound ',' hash_partbound_elem
			{
				$$ = lappend($1, $3);
			}
		;
```


#### 执行部分
这里我们只描述一下分区表创建的大致过程，分区表相关的详细代码非常多，可查看`backend/catalog/partition.c`、`backend/partitioning/partbounds.c`、`backend/partitioning/partdesc.c`、`backend/partitioning/partprune.c`、`tablecmds.c`、`parse_utilcmds.c`等等。更详细的可以看PostgreSQL源码中分区表功能实现的提交记录[Implement table partitioning.](https://github.com/postgres/postgres/commit/f0e44751d7175fa3394da2c8f85e3ceb3cdbfe63)，当然，分区表相关功能的实现是多个版本不断完善的结果，并不是这个提交全部完成的，仅供参考。并且，分区表并不仅仅是创建就完了，而是含有分区表相关的一系列优化，可在`backend/partitioning/`中查看，需要我们去不断深入理解，而且PostgreSQL还在不断的对分区表进行优化。

创建分区表主流程:
```c++
--> PortalStart
--> PortalRun
    --> standard_ProcessUtility
        --> ProcessUtilitySlow
            --> transformCreateStmt // 处理CreateStmt，处理列，约束等
                --> transformColumnDefinition // 处理列
                    --> transformConstraintAttrs
                --> transformTableConstraint  // 处理约束
                --> transformIndexConstraints  // 处理索引
                    --> transformIndexConstraint
                --> transformFKConstraints     // 处理外键
                --> transformCheckConstraints   // 处理check约束
                --> transformExtendedStatistics
            --> DefineRelation
                --> heap_create_with_catalog
                    --> heap_create
                // 分区处理相关代码
                // 处理stmt->partbound
                --> transformPartitionBound  
                    --> RelationGetPartitionKey
                    --> get_partition_strategy
                    --> transformPartitionBoundValue
                    --> transformPartitionRangeBounds
                        --> validateInfiniteBounds
                --> check_new_partition_bound
                --> StorePartitionBound // Update pg_class tuple of rel to store the partition bound and set relispartition to true
                --> StoreCatalogInheritance // 向系统表pg_inherits插入信息
                // 处理stmt->partspec
                --> transformPartitionSpec
                --> ComputePartitionAttrs
                --> StorePartitionKey // 向pg_partitioned_table中插入分区键等信息
--> PortalDrop
```
这里，我们将`DefineRelation`函数单独拿出来进行分析分区表的相关代码：
```c++
ObjectAddress DefineRelation(CreateStmt *stmt, char relkind, Oid ownerId, ObjectAddress *typaddress, const char *queryString)
{
    // ... 

    if (stmt->partspec != NULL)
	{
		if (relkind != RELKIND_RELATION)
			elog(ERROR, "unexpected relkind: %d", (int) relkind);

		relkind = RELKIND_PARTITIONED_TABLE;  // 分区表
		partitioned = true;
	}
	else
		partitioned = false;

	/* Select tablespace to use: an explicitly indicated one, or (in the case
	 * of a partitioned table) the parent's, if it has one. */
	if (stmt->tablespacename)
	{
		tablespaceId = get_tablespace_oid(stmt->tablespacename, false);

		if (partitioned && tablespaceId == MyDatabaseTableSpace)
			ereport(ERROR,
					(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
					 errmsg("cannot specify default tablespace for partitioned relations")));
	}
	else if (stmt->partbound)
	{
		/* For partitions, when no other tablespace is specified, we default
		 * the tablespace to the parent partitioned table's. */
		Assert(list_length(inheritOids) == 1);
		tablespaceId = get_rel_tablespace(linitial_oid(inheritOids));
	}
	else
		tablespaceId = InvalidOid;

	/* Parse and validate reloptions, if any.*/
	reloptions = transformRelOptions((Datum) 0, stmt->options, NULL, validnsps, true, false);

	switch (relkind)
	{
		case RELKIND_VIEW:
			(void) view_reloptions(reloptions, true);
			break;
		case RELKIND_PARTITIONED_TABLE:
			(void) partitioned_table_reloptions(reloptions, true);
			break;
		default:
			(void) heap_reloptions(relkind, reloptions, true);
	}

	/*
	 * Create the relation.  Inherited defaults and constraints are passed in
	 * for immediate handling --- since they don't need parsing, they can be
	 * stored immediately.
	 */
	relationId = heap_create_with_catalog(relname, namespaceId, tablespaceId, InvalidOid,
										  InvalidOid, ofTypeId, ownerId, accessMethodId, descriptor,
										  list_concat(cookedDefaults, old_constraints), relkind,
										  stmt->relation->relpersistence, false, false, stmt->oncommit,
										  reloptions, true, allowSystemTableMods, false, InvalidOid, typaddress);


	/*
	 * Open the new relation and acquire exclusive lock on it.  This isn't
	 * really necessary for locking out other backends (since they can't see
	 * the new rel anyway until we commit), but it keeps the lock manager from
	 * complaining about deadlock risks.
	 */
	rel = relation_open(relationId, AccessExclusiveLock);

	/*
	 * Now add any newly specified column default and generation expressions
	 * to the new relation.  These are passed to us in the form of raw
	 * parsetrees; we need to transform them to executable expression trees
	 * before they can be added. The most convenient way to do that is to
	 * apply the parser's transformExpr routine, but transformExpr doesn't
	 * work unless we have a pre-existing relation. So, the transformation has
	 * to be postponed to this final step of CREATE TABLE.
	 *
	 * This needs to be before processing the partitioning clauses because
	 * those could refer to generated columns.
	 */
	if (rawDefaults)
		AddRelationNewConstraints(rel, rawDefaults, NIL,
								  true, true, false, queryString);

	/*
	 * Make column generation expressions visible for use by partitioning.
	 */
	CommandCounterIncrement();

    // 处理分区表信息
	/* Process and store partition bound, if any. */
	if (stmt->partbound)
	{
		PartitionBoundSpec *bound;
		ParseState *pstate;
		Oid			parentId = linitial_oid(inheritOids),
					defaultPartOid;
		Relation	parent,
					defaultRel = NULL;
		ParseNamespaceItem *nsitem;

		/* Already have strong enough lock on the parent */
		parent = table_open(parentId, NoLock);

		/*
		 * We are going to try to validate the partition bound specification
		 * against the partition key of parentRel, so it better have one.
		 */
		if (parent->rd_rel->relkind != RELKIND_PARTITIONED_TABLE)
			ereport(ERROR,
					(errcode(ERRCODE_INVALID_OBJECT_DEFINITION),
					 errmsg("\"%s\" is not partitioned",
							RelationGetRelationName(parent))));

		/*
		 * The partition constraint of the default partition depends on the
		 * partition bounds of every other partition. It is possible that
		 * another backend might be about to execute a query on the default
		 * partition table, and that the query relies on previously cached
		 * default partition constraints. We must therefore take a table lock
		 * strong enough to prevent all queries on the default partition from
		 * proceeding until we commit and send out a shared-cache-inval notice
		 * that will make them update their index lists.
		 *
		 * Order of locking: The relation being added won't be visible to
		 * other backends until it is committed, hence here in
		 * DefineRelation() the order of locking the default partition and the
		 * relation being added does not matter. But at all other places we
		 * need to lock the default relation before we lock the relation being
		 * added or removed i.e. we should take the lock in same order at all
		 * the places such that lock parent, lock default partition and then
		 * lock the partition so as to avoid a deadlock.
		 */
		defaultPartOid = get_default_oid_from_partdesc(RelationGetPartitionDesc(parent,true));
		if (OidIsValid(defaultPartOid))
			defaultRel = table_open(defaultPartOid, AccessExclusiveLock);

		/* Transform the bound values */
		pstate = make_parsestate(NULL);
		pstate->p_sourcetext = queryString;

		/*
		 * Add an nsitem containing this relation, so that transformExpr
		 * called on partition bound expressions is able to report errors
		 * using a proper context.
		 */
		nsitem = addRangeTableEntryForRelation(pstate, rel, AccessShareLock,
											   NULL, false, false);
		addNSItemToQuery(pstate, nsitem, false, true, true);

		bound = transformPartitionBound(pstate, parent, stmt->partbound);

		/*
		 * Check first that the new partition's bound is valid and does not
		 * overlap with any of existing partitions of the parent.
		 */
		check_new_partition_bound(relname, parent, bound, pstate);

		/*
		 * If the default partition exists, its partition constraints will
		 * change after the addition of this new partition such that it won't
		 * allow any row that qualifies for this new partition. So, check that
		 * the existing data in the default partition satisfies the constraint
		 * as it will exist after adding this partition.
		 */
		if (OidIsValid(defaultPartOid))
		{
			check_default_partition_contents(parent, defaultRel, bound);
			/* Keep the lock until commit. */
			table_close(defaultRel, NoLock);
		}

		/* Update the pg_class entry. */
		StorePartitionBound(rel, parent, bound);  // 向pg_class中写入分区表信息

		table_close(parent, NoLock);
	}

	/* Store inheritance information for new rel. */
	StoreCatalogInheritance(relationId, inheritOids, stmt->partbound != NULL);

	/* Process the partitioning specification (if any) and store the partition
	 * key information into the catalog. */
	if (partitioned)
	{
		ParseState *pstate;
		char		strategy;
		int			partnatts;
		AttrNumber	partattrs[PARTITION_MAX_KEYS];
		Oid			partopclass[PARTITION_MAX_KEYS];
		Oid			partcollation[PARTITION_MAX_KEYS];
		List	   *partexprs = NIL;

		pstate = make_parsestate(NULL);
		pstate->p_sourcetext = queryString;

		partnatts = list_length(stmt->partspec->partParams);

		/* Protect fixed-size arrays here and in executor */
		if (partnatts > PARTITION_MAX_KEYS)
			ereport(ERROR,
					(errcode(ERRCODE_TOO_MANY_COLUMNS), errmsg("cannot partition using more than %d columns", PARTITION_MAX_KEYS)));

		/*
		 * We need to transform the raw parsetrees corresponding to partition
		 * expressions into executable expression trees.  Like column defaults
		 * and CHECK constraints, we could not have done the transformation
		 * earlier.
		 */
		stmt->partspec = transformPartitionSpec(rel, stmt->partspec, &strategy);

		ComputePartitionAttrs(pstate, rel, stmt->partspec->partParams,
							  partattrs, &partexprs, partopclass,
							  partcollation, strategy);

		StorePartitionKey(rel, strategy, partnatts, partattrs, partexprs,
						  partopclass, partcollation);

		/* make it all visible */
		CommandCounterIncrement();
	}

	/*
	 * If we're creating a partition, create now all the indexes, triggers,
	 * FKs defined in the parent.
	 *
	 * We can't do it earlier, because DefineIndex wants to know the partition
	 * key which we just stored.
	 */
	if (stmt->partbound)
	{
		Oid			parentId = linitial_oid(inheritOids);
		Relation	parent;
		List	   *idxlist;
		ListCell   *cell;

		/* Already have strong enough lock on the parent */
		parent = table_open(parentId, NoLock);
		idxlist = RelationGetIndexList(parent);

		/*
		 * For each index in the parent table, create one in the partition
		 */
		foreach(cell, idxlist)
		{
			Relation	idxRel = index_open(lfirst_oid(cell), AccessShareLock);
			AttrMap    *attmap;
			IndexStmt  *idxstmt;
			Oid			constraintOid;

			if (rel->rd_rel->relkind == RELKIND_FOREIGN_TABLE)
			{
				if (idxRel->rd_index->indisunique)
					ereport(ERROR,
							(errcode(ERRCODE_WRONG_OBJECT_TYPE),
							 errmsg("cannot create foreign partition of partitioned table \"%s\"",
									RelationGetRelationName(parent)),
							 errdetail("Table \"%s\" contains indexes that are unique.",
									   RelationGetRelationName(parent))));
				else
				{
					index_close(idxRel, AccessShareLock);
					continue;
				}
			}

			attmap = build_attrmap_by_name(RelationGetDescr(rel),
										   RelationGetDescr(parent));
			idxstmt =
				generateClonedIndexStmt(NULL, idxRel,
										attmap, &constraintOid);
			DefineIndex(RelationGetRelid(rel),
						idxstmt,
						InvalidOid,
						RelationGetRelid(idxRel),
						constraintOid,
						false, false, false, false, false);

			index_close(idxRel, AccessShareLock);
		}

		list_free(idxlist);

		/*
		 * If there are any row-level triggers, clone them to the new
		 * partition.
		 */
		if (parent->trigdesc != NULL)
			CloneRowTriggersToPartition(parent, rel);

		/*
		 * And foreign keys too.  Note that because we're freshly creating the
		 * table, there is no need to verify these new constraints.
		 */
		CloneForeignKeyConstraints(NULL, parent, rel);

		table_close(parent, NoLock);
	}

	/*
	 * Now add any newly specified CHECK constraints to the new relation. Same
	 * as for defaults above, but these need to come after partitioning is set
	 * up.
	 */
	if (stmt->constraints)
		AddRelationNewConstraints(rel, NIL, stmt->constraints,
								  true, true, false, queryString);

	ObjectAddressSet(address, RelationRelationId, relationId);

	/*
	 * Clean up.  We keep lock on new relation (although it shouldn't be
	 * visible to anyone else anyway, until commit).
	 */
	relation_close(rel, NoLock);

	return address;
}
```

#### 创建表的物理文件
数据库中的数据最终是要持久化的，创建一个表，实际上就是创建一个文件，在这个文件中存储元组信息。我们创建了一个`t1`的表，然后查看一下表的存储路径，其文件名即为表oid，表文件被切分为若干segment，每个segment不超过1G，segment又被切分为8k的块，元组存储在块Block中。
```c++
postgres=# create table t1(a int);
CREATE TABLE
postgres=# select pg_relation_filepath('t1');
 pg_relation_filepath 
----------------------
 base/5/16431
(1 row)
```

建表的过程伴随着物理表文件的创建，我们看一下源码调用栈，在`RelationCreateStorage`函数中创建具体的物理表文件在磁盘上。
```c++
RelationCreateStorage(RelFileNode rnode, char relpersistence, _Bool register_delete) (src\backend\catalog\storage.c:127)
heapam_relation_set_new_filenode(Relation rel, const RelFileNode * newrnode, char persistence, TransactionId * freezeXid, MultiXactId * minmulti) (src\backend\access\heap\heapam_handler.c:594)
table_relation_set_new_filenode(Relation rel, const RelFileNode * newrnode, char persistence, TransactionId * freezeXid, MultiXactId * minmulti) (src\include\access\tableam.h:1598)
heap_create(const char * relname, Oid relnamespace, Oid reltablespace, Oid relid, Oid relfilenode, Oid accessmtd, TupleDesc tupDesc, char relkind, char relpersistence, _Bool shared_relation, _Bool mapped_relation, _Bool allow_system_table_mods, TransactionId * relfrozenxid, MultiXactId * relminmxid, _Bool create_storage) (src\backend\catalog\heap.c:388)
heap_create_with_catalog(const char * relname, Oid relnamespace, Oid reltablespace, Oid relid, Oid reltypeid, Oid reloftypeid, Oid ownerid, Oid accessmtd, TupleDesc tupdesc, List * cooked_constraints, char relkind, char relpersistence, _Bool shared_relation, _Bool mapped_relation, OnCommitAction oncommit, Datum reloptions, _Bool use_user_acl, _Bool allow_system_table_mods, _Bool is_internal, Oid relrewrite, ObjectAddress * typaddress) (src\backend\catalog\heap.c:1275)
DefineRelation(CreateStmt * stmt, char relkind, Oid ownerId, ObjectAddress * typaddress, const char * queryString) (src\backend\commands\tablecmds.c:963)
ProcessUtilitySlow(ParseState * pstate, PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\utility.c:1171)
standard_ProcessUtility(PlannedStmt * pstmt, const char * queryString, _Bool readOnlyTree, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\utility.c:1074)
ProcessUtility(PlannedStmt * pstmt, const char * queryString, _Bool readOnlyTree, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\utility.c:530)
PortalRunUtility(Portal portal, PlannedStmt * pstmt, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, QueryCompletion * qc) (src\backend\tcop\pquery.c:1158)
PortalRunMulti(Portal portal, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:1315)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (src\backend\tcop\pquery.c:791)
exec_simple_query(const char * query_string) (src\backend\tcop\postgres.c:1250)
PostgresMain(const char * dbname, const char * username) (src\backend\tcop\postgres.c:4598)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4514)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4242)
ServerLoop() (src\backend\postmaster\postmaster.c:1809)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1481)
main(int argc, char ** argv) (src\backend\main\main.c:202)
```

我们看一下函数`RelationCreateStorage`实现：
```c++
/*
 * RelationCreateStorage
 *		Create physical storage for a relation.
 *
 * Create the underlying disk file storage for the relation. This only
 * creates the main fork; additional forks are created lazily by the
 * modules that need them.
 *
 * This function is transactional. The creation is WAL-logged, and if the
 * transaction aborts later on, the storage will be destroyed.  A caller
 * that does not want the storage to be destroyed in case of an abort may
 * pass register_delete = false.
 */
SMgrRelation RelationCreateStorage(RelFileNode rnode, char relpersistence, bool register_delete)
{
	SMgrRelation srel;
	BackendId	backend;
	bool		needs_wal;

	switch (relpersistence)
	{
		case RELPERSISTENCE_TEMP:
			backend = BackendIdForTempRelations();
			needs_wal = false;
			break;
		case RELPERSISTENCE_UNLOGGED:
			backend = InvalidBackendId;
			needs_wal = false;
			break;
		case RELPERSISTENCE_PERMANENT:
			backend = InvalidBackendId;
			needs_wal = true;
			break;
		default:
			elog(ERROR, "invalid relpersistence: %c", relpersistence);
			return NULL;		/* placate compiler */
	}

	srel = smgropen(rnode, backend);
	smgrcreate(srel, MAIN_FORKNUM, false);

	if (needs_wal)
		log_smgrcreate(&srel->smgr_rnode.node, MAIN_FORKNUM);

	/* Add the relation to the list of stuff to delete at abort, if we are asked to do so. */
	if (register_delete)
	{
		PendingRelDelete *pending;

		pending = (PendingRelDelete *) MemoryContextAlloc(TopMemoryContext, sizeof(PendingRelDelete));
		pending->relnode = rnode;
		pending->backend = backend;
		pending->atCommit = false;	/* delete if abort */
		pending->nestLevel = GetCurrentTransactionNestLevel();
		pending->next = pendingDeletes;
		pendingDeletes = pending;
	}

	if (relpersistence == RELPERSISTENCE_PERMANENT && !XLogIsNeeded())
	{
		Assert(backend == InvalidBackendId);
		AddPendingSync(&rnode);
	}

	return srel;
}
```


#### 后记
对分区表的理解，创建分区表仅仅是第一步，对分区表相关的优化才是重点，想一想，为什么会出现分区表，也是为了对相关场景的优化才出来的。所以，对分区表优化部分的理解十分重要，对应用端应用分区表并对相关场景进行优化也十分有益。