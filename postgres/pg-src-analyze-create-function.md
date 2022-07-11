### Postgres源码分析——create function

我们分析一下创建函数的源码，对应的测试语句如下：
```sql
 create function madd(integer, integer) returns integer as 'select $1 + $2;' language sql immutable returns null on null input;

 create function atax(total int, OUT result int) as $$ begin result := total * 2; end; $$ language plpgsql;

 create procedure login()  language plpgsql as $$ begin insert into dstu values(1,'a','a'); end;$$ ;
```

在分析源码之前，我们前面讲过[如何添加内核函数](https://mp.weixin.qq.com/s?__biz=MzU3NTk0ODkyNw==&mid=2247483955&idx=1&sn=1715c439e6c23e2e24c031c421ef1190&chksm=fd1a1323ca6d9a35a6ec85269c226b68c0985aa576383184428bf5deb2125404db3c593bb560&cur_album_id=1997508294004785155&scene=190#rd)，与之不同的是这里是用户自定义的函数，但有些方面是相通的，用户自定义的函数也需要存储在系统表pg_proc中，函数调用的过程也是相通的。所以在阅读本文前，最好先阅读一下如何添加内核函数这篇文章，理解起来会快很多。

#### 调用栈
输入调试语句`create function madd(integer, integer) returns integer as 'select $1 + $2;' language sql immutable returns null on null input;`，进行调试，我们先打印一下其调用栈，看一下其主要流程。
```c++
pg_parse_query(const char * query_string) (\postgres\src\backend\tcop\postgres.c:646)
fmgr_sql_validator(FunctionCallInfo fcinfo) (\postgres\src\backend\catalog\pg_proc.c:895)
FunctionCall1Coll(FmgrInfo * flinfo, Oid collation, Datum arg1) (\postgres\src\backend\utils\fmgr\fmgr.c:1142)
OidFunctionCall1Coll(Oid functionId, Oid collation, Datum arg1) (\postgres\src\backend\utils\fmgr\fmgr.c:1420)
ProcedureCreate(const char * procedureName, Oid procNamespace, _Bool replace, _Bool returnsSet, Oid returnType, Oid proowner, Oid languageObjectId, Oid languageValidator, const char * prosrc, const char * probin, char prokind, _Bool security_definer, _Bool isLeakProof, _Bool isStrict, char volatility, char parallel, oidvector * parameterTypes, Datum allParameterTypes, Datum parameterModes, Datum parameterNames, List * parameterDefaults, Datum trftypes, Datum proconfig, Oid prosupport, float4 procost, float4 prorows) (\postgres\src\backend\catalog\pg_proc.c:702)
CreateFunction(ParseState * pstate, CreateFunctionStmt * stmt) (\postgres\src\backend\commands\functioncmds.c:1152)
ProcessUtilitySlow(ParseState * pstate, PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (\postgres\src\backend\tcop\utility.c:1898)
standard_ProcessUtility(PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (\postgres\src\backend\tcop\utility.c:1305)
ProcessUtility(PlannedStmt * pstmt, const char * queryString, ProcessUtilityContext context, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (\postgres\src\backend\tcop\utility.c:753)
PortalRunUtility(Portal portal, PlannedStmt * pstmt, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, QueryCompletion * qc) (\postgres\src\backend\tcop\pquery.c:1161)
PortalRunMulti(Portal portal, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (\postgres\src\backend\tcop\pquery.c:1307)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (\postgres\src\backend\tcop\pquery.c:783)
exec_simple_query(const char * query_string) (\postgres\src\backend\tcop\postgres.c:1326)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (\postgres\src\backend\tcop\postgres.c:4445)
BackendRun(Port * port) (\postgres\src\backend\postmaster\postmaster.c:4988)
BackendStartup(Port * port) (\postgres\src\backend\postmaster\postmaster.c:4672)
ServerLoop() (\postgres\src\backend\postmaster\postmaster.c:1959)
PostmasterMain(int argc, char ** argv) (\postgres\src\backend\postmaster\postmaster.c:1495)
main(int argc, char ** argv) (\postgres\src\backend\main\main.c:231)
```
前面的处理过程应该很熟了，这条语句为UTILITY语句，经过语法分析后直接进入执行器中由`CreateFunction`进行处理。后面我们详细分析一下。

#### 语法解析
我们先看一下在gram.y中定义的CREATE FUNCTION语法。
```sql
/*****************************************************************************
 *
 *		QUERY:
 *				create [or replace] function <fname>
 *						[(<type-1> { , <type-n>})]
 *						returns <type-r>
 *						as <filename or code in language as appropriate>
 *						language <lang> [with parameters]
 *
 *****************************************************************************/

CreateFunctionStmt:
			CREATE opt_or_replace FUNCTION func_name func_args_with_defaults
			RETURNS func_return createfunc_opt_list
				{
					CreateFunctionStmt *n = makeNode(CreateFunctionStmt);
					n->is_procedure = false;
					n->replace = $2;
					n->funcname = $4;
					n->parameters = $5;
					n->returnType = $7;
					n->options = $8;
					$$ = (Node *)n;
				}
			| CREATE opt_or_replace FUNCTION func_name func_args_with_defaults
			  RETURNS TABLE '(' table_func_column_list ')' createfunc_opt_list
				{
					CreateFunctionStmt *n = makeNode(CreateFunctionStmt);
					n->is_procedure = false;
					n->replace = $2;
					n->funcname = $4;
					n->parameters = mergeTableFuncParameters($5, $9);
					n->returnType = TableFuncTypeName($9);
					n->returnType->location = @7;
					n->options = $11;
					$$ = (Node *)n;
				}
			| CREATE opt_or_replace FUNCTION func_name func_args_with_defaults
			  createfunc_opt_list
				{
					CreateFunctionStmt *n = makeNode(CreateFunctionStmt);
					n->is_procedure = false;
					n->replace = $2;
					n->funcname = $4;
					n->parameters = $5;
					n->returnType = NULL;
					n->options = $6;
					$$ = (Node *)n;
				}
			| CREATE opt_or_replace PROCEDURE func_name func_args_with_defaults
			  createfunc_opt_list
				{
					CreateFunctionStmt *n = makeNode(CreateFunctionStmt);
					n->is_procedure = true;
					n->replace = $2;
					n->funcname = $4;
					n->parameters = $5;
					n->returnType = NULL;
					n->options = $6;
					$$ = (Node *)n;
				}
		;

 -- 省略后续代码......
```

其中`CreateFunctionStmt`的定义如下：
```c++
/* ----------------------
 *		Create Function Statement
 * ----------------------*/
typedef struct CreateFunctionStmt
{
	NodeTag		type;
	bool		is_procedure;	/* it's really CREATE PROCEDURE */
	bool		replace;		/* T => replace if already exists */
	List	   *funcname;		/* qualified name of function to create */
	List	   *parameters;		/* a list of FunctionParameter */
	TypeName   *returnType;		/* the return type */
	List	   *options;		/* a list of DefElem */
} CreateFunctionStmt;

typedef enum FunctionParameterMode
{
	/* the assigned enum values appear in pg_proc, don't change 'em! */
	FUNC_PARAM_IN = 'i',		/* input only */
	FUNC_PARAM_OUT = 'o',		/* output only */
	FUNC_PARAM_INOUT = 'b',		/* both */
	FUNC_PARAM_VARIADIC = 'v',	/* variadic (always input) */
	FUNC_PARAM_TABLE = 't'		/* table function output column */
} FunctionParameterMode;

typedef struct FunctionParameter
{
	NodeTag		type;
	char	   *name;			/* parameter name, or NULL if not given */
	TypeName   *argType;		/* TypeName for parameter type */
	FunctionParameterMode mode; /* IN/OUT/etc */
	Node	   *defexpr;		/* raw default expr, or NULL if not given */
} FunctionParameter;

// 上面 select $1 + $2 参数表示
/* ParamRef - specifies a $n parameter reference*/
typedef struct ParamRef
{
	NodeTag		type;
	int			number;			/* the number of the parameter */
	int			location;		/* token location, or -1 if unknown */
} ParamRef;
```

#### 语义分析执行计划生成

前面生成了`CreateFunctionStmt`后，经`parse_analyze`转为查询树。在这里，`create function`为UTILITY语句最终在调用`transformStmt`时直接由语法解析树直接转为查询树`Query`。同时，因为为UTILITY语句，在调用`pg_plan_queries`生成执行计划时，无需经过`pg_plan_query`优化器过程，直接将查询树封装为执行计划树`PlannedStmt`再到执行器去执行。


#### 执行器

生成了执行计划树后，进入执行器执行，最终调用到`CreateFunction`，实现创建函数：`PortalRun`->`PortalRunMulti`->`PortalRunUtility`->`ProcessUtility`
->`standard_ProcessUtility`->`ProcessUtilitySlow`->`CreateFunction`。

我们研读一下`CreateFunction`的代码：
```c++
/* CreateFunction
 *	 Execute a CREATE FUNCTION (or CREATE PROCEDURE) utility statement. */
ObjectAddress CreateFunction(ParseState *pstate, CreateFunctionStmt *stmt)
{
	char	   *probin_str;
	char	   *prosrc_str;
	Oid			prorettype;
	bool		returnsSet;
	char	   *language;
	Oid			languageOid, languageValidator;
	Node	   *transformDefElem = NULL;
	char	   *funcname;
	Oid			namespaceId;
	AclResult	aclresult;
	oidvector  *parameterTypes;
	ArrayType  *allParameterTypes;
	ArrayType  *parameterModes;
	ArrayType  *parameterNames;
	List	   *parameterDefaults;
	Oid			variadicArgType;
	List	   *trftypes_list = NIL;
	ArrayType  *trftypes;
	Oid			requiredResultType;
	bool		isWindowFunc,isStrict,security,isLeakProof;
	char		volatility, parallel;
	ArrayType  *proconfig;
	float4		procost, prorows;
	Oid			prosupport;
	HeapTuple	languageTuple;
	Form_pg_language languageStruct;
	List	   *as_clause;

	/* Convert list of names to a name and namespace */
	namespaceId = QualifiedNameGetCreationNamespace(stmt->funcname, &funcname);

	/* Check we have creation rights in target namespace */
	aclresult = pg_namespace_aclcheck(namespaceId, GetUserId(), ACL_CREATE);  // 权限检查
	if (aclresult != ACLCHECK_OK)
		aclcheck_error(aclresult, OBJECT_SCHEMA, get_namespace_name(namespaceId));

	/* Set default attributes */
	isWindowFunc = false;
	isStrict = false;
	security = false;
	isLeakProof = false;
	volatility = PROVOLATILE_VOLATILE;
	proconfig = NULL;
	procost = -1;				/* indicates not set */
	prorows = -1;				/* indicates not set */
	prosupport = InvalidOid;
	parallel = PROPARALLEL_UNSAFE;

	/* Extract non-default attributes from stmt->options list */
	compute_function_attributes(pstate, stmt->is_procedure, stmt->options, &as_clause, &language, &transformDefElem, &isWindowFunc, &volatility, &isStrict, &security, &isLeakProof, &proconfig, &procost, &prorows, &prosupport, &parallel);

	/* Look up the language and validate permissions */ // 检索语言并验证授权
	languageTuple = SearchSysCache1(LANGNAME, PointerGetDatum(language));
	if (!HeapTupleIsValid(languageTuple))
		ereport(ERROR, (errcode(ERRCODE_UNDEFINED_OBJECT), errmsg("language \"%s\" does not exist", language), (extension_file_exists(language) ? errhint("Use CREATE EXTENSION to load the language into the database.") : 0)));

	languageStruct = (Form_pg_language) GETSTRUCT(languageTuple);
	languageOid = languageStruct->oid;

	if (languageStruct->lanpltrusted) {
		AclResult	aclresult = pg_language_aclcheck(languageOid, GetUserId(), ACL_USAGE);		/* if trusted language, need USAGE privilege */
		if (aclresult != ACLCHECK_OK)
			aclcheck_error(aclresult, OBJECT_LANGUAGE, NameStr(languageStruct->lanname));
	} else {
		if (!superuser()) /* if untrusted language, must be superuser */
			aclcheck_error(ACLCHECK_NO_PRIV, OBJECT_LANGUAGE, NameStr(languageStruct->lanname));
	}

	languageValidator = languageStruct->lanvalidator;
	ReleaseSysCache(languageTuple);

	/* Only superuser is allowed to create leakproof functions because
	 * leakproof functions can see tuples which have not yet been filtered out
	 * by security barrier views or row level security policies. */
	if (isLeakProof && !superuser())
		ereport(ERROR, (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE), errmsg("only superuser can define a leakproof function")));

	if (transformDefElem) {
		ListCell   *lc;
		foreach(lc, castNode(List, transformDefElem)) {
			Oid			typeid = typenameTypeId(NULL, lfirst_node(TypeName, lc));
			Oid			elt = get_base_element_type(typeid);

			typeid = elt ? elt : typeid;
			get_transform_oid(typeid, languageOid, false);
			trftypes_list = lappend_oid(trftypes_list, typeid);
		}
	}

	/* Convert remaining parameters of CREATE to form wanted by ProcedureCreate. */
	interpret_function_parameter_list(pstate, stmt->parameters, languageOid, stmt->is_procedure ? OBJECT_PROCEDURE : OBJECT_FUNCTION, &parameterTypes, &allParameterTypes, &parameterModes, &parameterNames, &parameterDefaults, &variadicArgType, &requiredResultType);

	if (stmt->is_procedure) {
		prorettype = requiredResultType ? requiredResultType : VOIDOID;
		returnsSet = false;
	} else if (stmt->returnType) {
		/* explicit RETURNS clause */
		compute_return_type(stmt->returnType, languageOid, &prorettype, &returnsSet);
		if (OidIsValid(requiredResultType) && prorettype != requiredResultType)
			ereport(ERROR, (errcode(ERRCODE_INVALID_FUNCTION_DEFINITION), errmsg("function result type must be %s because of OUT parameters", format_type_be(requiredResultType))));
	} else if (OidIsValid(requiredResultType)) {
		prorettype = requiredResultType; /* default RETURNS clause from OUT parameters */
		returnsSet = false;
	} else {
		ereport(ERROR, (errcode(ERRCODE_INVALID_FUNCTION_DEFINITION), errmsg("function result type must be specified")));
		/* Alternative possibility: default to RETURNS VOID */
		prorettype = VOIDOID;
		returnsSet = false;
	}

	if (list_length(trftypes_list) > 0) {
		ListCell   *lc;
		Datum	   *arr = palloc(list_length(trftypes_list) * sizeof(Datum));
		int i = 0;
		foreach(lc, trftypes_list)
			arr[i++] = ObjectIdGetDatum(lfirst_oid(lc));
		trftypes = construct_array(arr, list_length(trftypes_list), OIDOID, sizeof(Oid), true, TYPALIGN_INT);
	}
	else
		trftypes = NULL;		/* store SQL NULL instead of empty array */

	interpret_AS_clause(languageOid, language, funcname, as_clause, &prosrc_str, &probin_str);

	/* Set default values for COST and ROWS depending on other parameters;
	 * reject ROWS if it's not returnsSet.  NB: pg_dump knows these default
	 * values, keep it in sync if you change them.*/
	if (procost < 0)
	{
		/* SQL and PL-language functions are assumed more expensive */
		if (languageOid == INTERNALlanguageId || languageOid == ClanguageId)
			procost = 1;
		else
			procost = 100;
	}
	if (prorows < 0) {
		if (returnsSet)
			prorows = 1000;
		else
			prorows = 0;		/* dummy value if not returnsSet */
	} else if (!returnsSet)
		ereport(ERROR, (errcode(ERRCODE_INVALID_PARAMETER_VALUE), errmsg("ROWS is not applicable when function does not return a set")));

	/* And now that we have all the parameters, and know we're permitted to do so, go ahead and create the function.*/
	return ProcedureCreate(funcname, namespaceId, stmt->replace, returnsSet,
						   prorettype, GetUserId(), languageOid, languageValidator,
						   prosrc_str,	/* converted to text later */
						   probin_str,	/* converted to text later */
						   stmt->is_procedure ? PROKIND_PROCEDURE : (isWindowFunc ? PROKIND_WINDOW : PROKIND_FUNCTION),
						   security, isLeakProof, isStrict, volatility, parallel, parameterTypes,
						   PointerGetDatum(allParameterTypes), PointerGetDatum(parameterModes), PointerGetDatum(parameterNames), parameterDefaults,
						   PointerGetDatum(trftypes), PointerGetDatum(proconfig), prosupport, procost, prorows);
}
```
其实在`ProcedureCreate`之前都是创建函数/存储过程的准备工作，检查是否满足权限，参数的处理等，真正创建的是`ProcedureCreate`函数。文章开头讲过，创建函数/存储过程，其实是存储在系统表`pg_proc`中的。之后调用的时候，去查系统表，有该函数/存储过程，再完成后续调用过程。插入系统表`pg_proc`的过程是在`ProcedureCreate`函数中完成的。
```c++
/* ----------------------------------------------------------------
 *		ProcedureCreate
 * Note: allParameterTypes, parameterModes, parameterNames, trftypes, and proconfig
 * are either arrays of the proper types or NULL.  We declare them Datum,
 * not "ArrayType *", to avoid importing array.h into pg_proc.h.
 * ----------------------------------------------------------------*/
ObjectAddress ProcedureCreate(const char *procedureName, Oid procNamespace,
				bool replace, bool returnsSet,
				Oid returnType, Oid proowner,
				Oid languageObjectId, Oid languageValidator,
				const char *prosrc, const char *probin, char prokind,
				bool security_definer, bool isLeakProof, bool isStrict,
				char volatility,char parallel, oidvector *parameterTypes,
				Datum allParameterTypes,Datum parameterModes,Datum parameterNames,List *parameterDefaults,
				Datum trftypes,Datum proconfig,
				Oid prosupport,float4 procost,float4 prorows)
{
	Oid			retval;
	int			parameterCount, allParamCount, i;
	Oid		   *allParams;
	char	   *paramModes = NULL;
	Oid			variadicType = InvalidOid;
	Acl		   *proacl = NULL;
	Relation	rel;
	HeapTuple	tup, oldtup;
	bool		nulls[Natts_pg_proc];
	Datum		values[Natts_pg_proc];
	bool		replaces[Natts_pg_proc];
	NameData	procname;
	TupleDesc	tupDesc;
	bool		is_update;
	ObjectAddress myself, referenced;
	char	   *detailmsg;
	Oid			trfid;

	parameterCount = parameterTypes->dim1;
	if (parameterCount < 0 || parameterCount > FUNC_MAX_ARGS)
		ereport(ERROR, (errcode(ERRCODE_TOO_MANY_ARGUMENTS), errmsg_plural("functions cannot have more than %d argument","functions cannot have more than %d arguments", FUNC_MAX_ARGS,FUNC_MAX_ARGS)));

	/* Deconstruct array inputs */
	if (allParameterTypes != PointerGetDatum(NULL)) {
		/* We expect the array to be a 1-D OID array; verify that. We don't
		 * need to use deconstruct_array() since the array data is just going
		 * to look like a C array of OID values.*/
		ArrayType  *allParamArray = (ArrayType *) DatumGetPointer(allParameterTypes);

		allParamCount = ARR_DIMS(allParamArray)[0];
		if (ARR_NDIM(allParamArray) != 1 || allParamCount <= 0 || ARR_HASNULL(allParamArray) || ARR_ELEMTYPE(allParamArray) != OIDOID)
			elog(ERROR, "allParameterTypes is not a 1-D Oid array");
		allParams = (Oid *) ARR_DATA_PTR(allParamArray);
		/* we assume caller got the contents right */
	} else {
		allParamCount = parameterCount;
		allParams = parameterTypes->values;
	}

	if (parameterModes != PointerGetDatum(NULL)) {
		/* We expect the array to be a 1-D CHAR array; verify that. We don't
		 * need to use deconstruct_array() since the array data is just going
		 * to look like a C array of char values.*/
		ArrayType  *modesArray = (ArrayType *) DatumGetPointer(parameterModes);

		if (ARR_NDIM(modesArray) != 1 || ARR_DIMS(modesArray)[0] != allParamCount || ARR_HASNULL(modesArray) || ARR_ELEMTYPE(modesArray) != CHAROID)
			elog(ERROR, "parameterModes is not a 1-D char array");
		paramModes = (char *) ARR_DATA_PTR(modesArray);
	}

	/* Do not allow polymorphic return type unless there is a polymorphic input argument that we can use to deduce the actual return type. */
	detailmsg = check_valid_polymorphic_signature(returnType, parameterTypes->values, parameterCount);
	if (detailmsg)
		ereport(ERROR,(errcode(ERRCODE_INVALID_FUNCTION_DEFINITION), errmsg("cannot determine result data type"), errdetail_internal("%s", detailmsg)));

	/* Also, do not allow return type INTERNAL unless at least one input argument is INTERNAL. */
	detailmsg = check_valid_internal_signature(returnType, parameterTypes->values, parameterCount);
	if (detailmsg)
		ereport(ERROR,(errcode(ERRCODE_INVALID_FUNCTION_DEFINITION),errmsg("unsafe use of pseudo-type \"internal\""),errdetail_internal("%s", detailmsg)));

	/* Apply the same tests to any OUT arguments.*/
	if (allParameterTypes != PointerGetDatum(NULL))
	{
		for (i = 0; i < allParamCount; i++)
		{
			if (paramModes == NULL || paramModes[i] == PROARGMODE_IN || paramModes[i] == PROARGMODE_VARIADIC)
				continue;		/* ignore input-only params */

			detailmsg = check_valid_polymorphic_signature(allParams[i], parameterTypes->values, parameterCount);
			if (detailmsg)
				ereport(ERROR,(errcode(ERRCODE_INVALID_FUNCTION_DEFINITION), errmsg("cannot determine result data type"), errdetail_internal("%s", detailmsg)));
			
			detailmsg = check_valid_internal_signature(allParams[i], parameterTypes->values, parameterCount);
			if (detailmsg)
				ereport(ERROR,(errcode(ERRCODE_INVALID_FUNCTION_DEFINITION), errmsg("unsafe use of pseudo-type \"internal\""), errdetail_internal("%s", detailmsg)));
		}
	}

	/* Identify variadic argument type, if any */
	if (paramModes != NULL)
	{
		/* Only the last input parameter can be variadic; if it is, save its
		 * element type.  Errors here are just elog since caller should have checked this already. */
		for (i = 0; i < allParamCount; i++)
		{
			switch (paramModes[i])
			{
				case PROARGMODE_IN:
				case PROARGMODE_INOUT:
					if (OidIsValid(variadicType))
						elog(ERROR, "variadic parameter must be last");
					break;
				case PROARGMODE_OUT:
				case PROARGMODE_TABLE:
					break;/* okay */
				case PROARGMODE_VARIADIC:
					if (OidIsValid(variadicType))
						elog(ERROR, "variadic parameter must be last");
					switch (allParams[i])
					{
						case ANYOID:
							variadicType = ANYOID;
							break;
						case ANYARRAYOID:
							variadicType = ANYELEMENTOID;
							break;
						case ANYCOMPATIBLEARRAYOID:
							variadicType = ANYCOMPATIBLEOID;
							break;
						default:
							variadicType = get_element_type(allParams[i]);
							if (!OidIsValid(variadicType))
								elog(ERROR, "variadic parameter is not an array");
							break;
					}
					break;
				default:
					elog(ERROR, "invalid parameter mode '%c'", paramModes[i]);
					break;
			}
		}
	}

	// 插入到系统表pg_proc中
	/* All seems OK; prepare the data to be inserted into pg_proc.*/
	for (i = 0; i < Natts_pg_proc; ++i)
	{
		nulls[i] = false;
		values[i] = (Datum) 0;
		replaces[i] = true;
	}

	namestrcpy(&procname, procedureName);
	values[Anum_pg_proc_proname - 1] = NameGetDatum(&procname);
	values[Anum_pg_proc_pronamespace - 1] = ObjectIdGetDatum(procNamespace);
	values[Anum_pg_proc_proowner - 1] = ObjectIdGetDatum(proowner);
	values[Anum_pg_proc_prolang - 1] = ObjectIdGetDatum(languageObjectId);
	values[Anum_pg_proc_procost - 1] = Float4GetDatum(procost);
	values[Anum_pg_proc_prorows - 1] = Float4GetDatum(prorows);
	values[Anum_pg_proc_provariadic - 1] = ObjectIdGetDatum(variadicType);
	values[Anum_pg_proc_prosupport - 1] = ObjectIdGetDatum(prosupport);
	values[Anum_pg_proc_prokind - 1] = CharGetDatum(prokind);
	values[Anum_pg_proc_prosecdef - 1] = BoolGetDatum(security_definer);
	values[Anum_pg_proc_proleakproof - 1] = BoolGetDatum(isLeakProof);
	values[Anum_pg_proc_proisstrict - 1] = BoolGetDatum(isStrict);
	values[Anum_pg_proc_proretset - 1] = BoolGetDatum(returnsSet);
	values[Anum_pg_proc_provolatile - 1] = CharGetDatum(volatility);
	values[Anum_pg_proc_proparallel - 1] = CharGetDatum(parallel);
	values[Anum_pg_proc_pronargs - 1] = UInt16GetDatum(parameterCount);
	values[Anum_pg_proc_pronargdefaults - 1] = UInt16GetDatum(list_length(parameterDefaults));
	values[Anum_pg_proc_prorettype - 1] = ObjectIdGetDatum(returnType);
	values[Anum_pg_proc_proargtypes - 1] = PointerGetDatum(parameterTypes);
	if (allParameterTypes != PointerGetDatum(NULL))
		values[Anum_pg_proc_proallargtypes - 1] = allParameterTypes;
	else
		nulls[Anum_pg_proc_proallargtypes - 1] = true;
	if (parameterModes != PointerGetDatum(NULL))
		values[Anum_pg_proc_proargmodes - 1] = parameterModes;
	else
		nulls[Anum_pg_proc_proargmodes - 1] = true;
	if (parameterNames != PointerGetDatum(NULL))
		values[Anum_pg_proc_proargnames - 1] = parameterNames;
	else
		nulls[Anum_pg_proc_proargnames - 1] = true;
	if (parameterDefaults != NIL)
		values[Anum_pg_proc_proargdefaults - 1] = CStringGetTextDatum(nodeToString(parameterDefaults));
	else
		nulls[Anum_pg_proc_proargdefaults - 1] = true;
	if (trftypes != PointerGetDatum(NULL))
		values[Anum_pg_proc_protrftypes - 1] = trftypes;
	else
		nulls[Anum_pg_proc_protrftypes - 1] = true;
	values[Anum_pg_proc_prosrc - 1] = CStringGetTextDatum(prosrc);
	if (probin)
		values[Anum_pg_proc_probin - 1] = CStringGetTextDatum(probin);
	else
		nulls[Anum_pg_proc_probin - 1] = true;
	if (proconfig != PointerGetDatum(NULL))
		values[Anum_pg_proc_proconfig - 1] = proconfig;
	else
		nulls[Anum_pg_proc_proconfig - 1] = true;
	/* proacl will be determined later */

	rel = table_open(ProcedureRelationId, RowExclusiveLock);
	tupDesc = RelationGetDescr(rel);

	/* Check for pre-existing definition */
	oldtup = SearchSysCache3(PROCNAMEARGSNSP,PointerGetDatum(procedureName),PointerGetDatum(parameterTypes), ObjectIdGetDatum(procNamespace));

	if (HeapTupleIsValid(oldtup))
	{
		/* There is one; okay to replace it? */
		Form_pg_proc oldproc = (Form_pg_proc) GETSTRUCT(oldtup);
		Datum		proargnames;
		bool		isnull;
		const char *dropcmd;

		if (!replace)
			ereport(ERROR,(errcode(ERRCODE_DUPLICATE_FUNCTION), errmsg("function \"%s\" already exists with same argument types",procedureName)));
		if (!pg_proc_ownercheck(oldproc->oid, proowner))
			aclcheck_error(ACLCHECK_NOT_OWNER, OBJECT_FUNCTION, procedureName);

		/* Not okay to change routine kind */
		if (oldproc->prokind != prokind)
			ereport(ERROR,(errcode(ERRCODE_WRONG_OBJECT_TYPE),errmsg("cannot change routine kind"),
					 (oldproc->prokind == PROKIND_AGGREGATE ? errdetail("\"%s\" is an aggregate function.", procedureName) :
					  oldproc->prokind == PROKIND_FUNCTION ? errdetail("\"%s\" is a function.", procedureName) : oldproc->prokind == PROKIND_PROCEDURE ? errdetail("\"%s\" is a procedure.", procedureName) :
					  oldproc->prokind == PROKIND_WINDOW ? errdetail("\"%s\" is a window function.", procedureName) : 0)));

		dropcmd = (prokind == PROKIND_PROCEDURE ? "DROP PROCEDURE" : prokind == PROKIND_AGGREGATE ? "DROP AGGREGATE" : "DROP FUNCTION");

		/* Not okay to change the return type of the existing proc, since
		 * existing rules, views, etc may depend on the return type. */
		if (returnType != oldproc->prorettype || returnsSet != oldproc->proretset)
			ereport(ERROR, (errcode(ERRCODE_INVALID_FUNCTION_DEFINITION),prokind == PROKIND_PROCEDURE? errmsg("cannot change whether a procedure has output parameters") : errmsg("cannot change return type of existing function"),
			/* translator: first %s is DROP FUNCTION, DROP PROCEDURE, or DROP AGGREGATE */
			 errhint("Use %s %s first.",dropcmd,format_procedure(oldproc->oid))));

		/* If it returns RECORD, check for possible change of record type implied by OUT parameters*/
		if (returnType == RECORDOID)
		{
			TupleDesc	olddesc;
			TupleDesc	newdesc;

			olddesc = build_function_result_tupdesc_t(oldtup);
			newdesc = build_function_result_tupdesc_d(prokind,allParameterTypes, parameterModes, parameterNames);
			if (olddesc == NULL && newdesc == NULL)
				 /* ok, both are runtime-defined RECORDs */ ;
			else if (olddesc == NULL || newdesc == NULL ||!equalTupleDescs(olddesc, newdesc))
				ereport(ERROR,
						(errcode(ERRCODE_INVALID_FUNCTION_DEFINITION),
						 errmsg("cannot change return type of existing function"),
						 errdetail("Row type defined by OUT parameters is different."),
				/* translator: first %s is DROP FUNCTION or DROP PROCEDURE */
						 errhint("Use %s %s first.",dropcmd, format_procedure(oldproc->oid))));
		}

		/* If there were any named input parameters, check to make sure the
		 * names have not been changed, as this could break existing calls. We
		 * allow adding names to formerly unnamed parameters, though.*/
		proargnames = SysCacheGetAttr(PROCNAMEARGSNSP, oldtup, Anum_pg_proc_proargnames,  &isnull);
		if (!isnull)
		{
			Datum		proargmodes;
			char	  **old_arg_names;
			char	  **new_arg_names;
			int			n_old_arg_names;
			int			n_new_arg_names;
			int			j;

			proargmodes = SysCacheGetAttr(PROCNAMEARGSNSP, oldtup, Anum_pg_proc_proargmodes,&isnull);
			if (isnull)
				proargmodes = PointerGetDatum(NULL);	/* just to be sure */

			n_old_arg_names = get_func_input_arg_names(proargnames, proargmodes, &old_arg_names);
			n_new_arg_names = get_func_input_arg_names(parameterNames, parameterModes, &new_arg_names);
			for (j = 0; j < n_old_arg_names; j++)
			{
				if (old_arg_names[j] == NULL)
					continue;
				if (j >= n_new_arg_names || new_arg_names[j] == NULL || strcmp(old_arg_names[j], new_arg_names[j]) != 0)
					ereport(ERROR, (errcode(ERRCODE_INVALID_FUNCTION_DEFINITION),
							 errmsg("cannot change name of input parameter \"%s\"", old_arg_names[j]),
					/* translator: first %s is DROP FUNCTION or DROP PROCEDURE */
							 errhint("Use %s %s first.", dropcmd, format_procedure(oldproc->oid))));
			}
		}

		/* If there are existing defaults, check compatibility: redefinition
		 * must not remove any defaults nor change their types. */
		if (oldproc->pronargdefaults != 0)
		{
			Datum		proargdefaults;
			List	   *oldDefaults;
			ListCell   *oldlc, *newlc;

			if (list_length(parameterDefaults) < oldproc->pronargdefaults)
				ereport(ERROR, (errcode(ERRCODE_INVALID_FUNCTION_DEFINITION), errmsg("cannot remove parameter defaults from existing function"),
				/* translator: first %s is DROP FUNCTION or DROP PROCEDURE */
						 errhint("Use %s %s first.", dropcmd, format_procedure(oldproc->oid))));

			proargdefaults = SysCacheGetAttr(PROCNAMEARGSNSP, oldtup, Anum_pg_proc_proargdefaults, &isnull);
			oldDefaults = castNode(List, stringToNode(TextDatumGetCString(proargdefaults)));

			/* new list can have more defaults than old, advance over 'em */
			newlc = list_nth_cell(parameterDefaults, list_length(parameterDefaults) - oldproc->pronargdefaults);
			foreach(oldlc, oldDefaults)
			{
				Node	   *oldDef = (Node *) lfirst(oldlc);
				Node	   *newDef = (Node *) lfirst(newlc);

				if (exprType(oldDef) != exprType(newDef))
					ereport(ERROR,(errcode(ERRCODE_INVALID_FUNCTION_DEFINITION), errmsg("cannot change data type of existing parameter default value"),
					/* translator: first %s is DROP FUNCTION or DROP PROCEDURE */
							 errhint("Use %s %s first.", dropcmd, format_procedure(oldproc->oid))));
				newlc = lnext(parameterDefaults, newlc);
			}
		}

		/* Do not change existing oid, ownership or permissions, either.  Note
		 * dependency-update code below has to agree with this decision.*/
		replaces[Anum_pg_proc_oid - 1] = false;
		replaces[Anum_pg_proc_proowner - 1] = false;
		replaces[Anum_pg_proc_proacl - 1] = false;

		/* Okay, do it... */
		tup = heap_modify_tuple(oldtup, tupDesc, values, nulls, replaces);
		CatalogTupleUpdate(rel, &tup->t_self, tup);

		ReleaseSysCache(oldtup);
		is_update = true;
	} else {
		/* Creating a new procedure */
		Oid			newOid;

		/* First, get default permissions and set up proacl */
		proacl = get_user_default_acl(OBJECT_FUNCTION, proowner, procNamespace);
		if (proacl != NULL)
			values[Anum_pg_proc_proacl - 1] = PointerGetDatum(proacl);
		else
			nulls[Anum_pg_proc_proacl - 1] = true;

		newOid = GetNewOidWithIndex(rel, ProcedureOidIndexId, Anum_pg_proc_oid);
		values[Anum_pg_proc_oid - 1] = ObjectIdGetDatum(newOid);
		tup = heap_form_tuple(tupDesc, values, nulls);
		CatalogTupleInsert(rel, tup);
		is_update = false;
	}

	retval = ((Form_pg_proc) GETSTRUCT(tup))->oid;

	// 新增依赖，插入系统表pg_depend。
	/* Create dependencies for the new function.  If we are updating an
	 * existing function, first delete any existing pg_depend entries.
	 * (However, since we are not changing ownership or permissions, the
	 * shared dependencies do *not* need to change, and we leave them alone.) */
	if (is_update)
		deleteDependencyRecordsFor(ProcedureRelationId, retval, true);

	myself.classId = ProcedureRelationId;
	myself.objectId = retval;
	myself.objectSubId = 0;

	/* dependency on namespace */
	referenced.classId = NamespaceRelationId;
	referenced.objectId = procNamespace;
	referenced.objectSubId = 0;
	recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);

	/* dependency on implementation language */
	referenced.classId = LanguageRelationId;
	referenced.objectId = languageObjectId;
	referenced.objectSubId = 0;
	recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);

	/* dependency on return type */
	referenced.classId = TypeRelationId;
	referenced.objectId = returnType;
	referenced.objectSubId = 0;
	recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);

	/* dependency on transform used by return type, if any */
	if ((trfid = get_transform_oid(returnType, languageObjectId, true)))
	{
		referenced.classId = TransformRelationId;
		referenced.objectId = trfid;
		referenced.objectSubId = 0;
		recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);
	}

	/* dependency on parameter types */
	for (i = 0; i < allParamCount; i++)
	{
		referenced.classId = TypeRelationId;
		referenced.objectId = allParams[i];
		referenced.objectSubId = 0;
		recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);

		/* dependency on transform used by parameter type, if any */
		if ((trfid = get_transform_oid(allParams[i], languageObjectId, true)))
		{
			referenced.classId = TransformRelationId;
			referenced.objectId = trfid;
			referenced.objectSubId = 0;
			recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);
		}
	}

	/* dependency on parameter default expressions */
	if (parameterDefaults)
		recordDependencyOnExpr(&myself, (Node *) parameterDefaults, NIL, DEPENDENCY_NORMAL);

	/* dependency on support function, if any */
	if (OidIsValid(prosupport))
	{
		referenced.classId = ProcedureRelationId;
		referenced.objectId = prosupport;
		referenced.objectSubId = 0;
		recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);
	}

	/* dependency on owner */
	if (!is_update)
		recordDependencyOnOwner(ProcedureRelationId, retval, proowner);

	/* dependency on any roles mentioned in ACL */
	if (!is_update)
		recordDependencyOnNewAcl(ProcedureRelationId, retval, 0, proowner, proacl);

	/* dependency on extension */
	recordDependencyOnCurrentExtension(&myself, is_update);
	heap_freetuple(tup);

	/* Post creation hook for new function */
	InvokeObjectPostCreateHook(ProcedureRelationId, retval, 0);
	table_close(rel, RowExclusiveLock);

	/* Verify function body */
	if (OidIsValid(languageValidator))
	{
		ArrayType  *set_items = NULL;
		int			save_nestlevel = 0;

		/* Advance command counter so new tuple can be seen by validator */
		CommandCounterIncrement();

		/* Set per-function configuration parameters so that the validation is
		 * done with the environment the function expects.  */
		if (check_function_bodies)
		{
			set_items = (ArrayType *) DatumGetPointer(proconfig);
			if (set_items)		/* Need a new GUC nesting level */
			{
				save_nestlevel = NewGUCNestLevel();
				ProcessGUCArray(set_items,(superuser() ? PGC_SUSET : PGC_USERSET),PGC_S_SESSION,GUC_ACTION_SAVE);
			}
		}

		OidFunctionCall1(languageValidator, ObjectIdGetDatum(retval));	// 这个地方需要注意，后面会分析一下，不同语言不同的处理
		if (set_items)
			AtEOXact_GUC(true, save_nestlevel);
	}

	return myself;
}
```
到这里，我们基本弄清了`CREATE FUNCTION`的逻辑，下面继续看一下对不同过程语言的处理逻辑。

#### 不同过程语言的处理
PG中有哪些过程语言呢？
```sql
postgres@postgres=# select * from pg_language;
  oid  | lanname  | lanowner | lanispl | lanpltrusted | lanplcallfoid | laninline | lanvalidator | lanacl 
-------+----------+----------+---------+--------------+---------------+-----------+--------------+--------
    12 | internal |       10 | f       | f            |             0 |         0 |         2246 | 
    13 | c        |       10 | f       | f            |             0 |         0 |         2247 | 
    14 | sql      |       10 | f       | t            |             0 |         0 |         2248 | 
 13573 | plpgsql  |       10 | t       | t            |         13570 |     13571 |        13572 | 
(4 rows)
```
对于sql，`OidFunctionCall1`会调用`fmgr_sql_validator`进行处理。其`fmgr_sql_validator`的实现细节这里不再详述，我们看一下pgpgsql过程语言的处理。

输入新的测试语句`create function btax(total int, OUT result int) as $$ begin result := total * 2; end; $$ language plpgsql;`，打印其调用栈：
```c++
plpgsql.so!do_compile(FunctionCallInfo fcinfo, HeapTuple procTup, PLpgSQL_function * function, PLpgSQL_func_hashkey * hashkey, _Bool forValidator) (\postgres\src\pl\plpgsql\src\pl_comp.c:320)
plpgsql.so!plpgsql_compile(FunctionCallInfo fcinfo, _Bool forValidator) (\postgres\src\pl\plpgsql\src\pl_comp.c:223)
plpgsql.so!plpgsql_validator(FunctionCallInfo fcinfo) (\postgres\src\pl\plpgsql\src\pl_handler.c:514)
FunctionCall1Coll(FmgrInfo * flinfo, Oid collation, Datum arg1) (\postgres\src\backend\utils\fmgr\fmgr.c:1142)
OidFunctionCall1Coll(Oid functionId, Oid collation, Datum arg1) (\postgres\src\backend\utils\fmgr\fmgr.c:1420)
ProcedureCreate(const char * procedureName, Oid procNamespace, _Bool replace, _Bool returnsSet, Oid returnType, Oid proowner, Oid languageObjectId, Oid languageValidator, const char * prosrc, const char * probin, char prokind, _Bool security_definer, _Bool isLeakProof, _Bool isStrict, char volatility, char parallel, oidvector * parameterTypes, Datum allParameterTypes, Datum parameterModes, Datum parameterNames, List * parameterDefaults, Datum trftypes, Datum proconfig, Oid prosupport, float4 procost, float4 prorows) (\postgres\src\backend\catalog\pg_proc.c:702)
CreateFunction(ParseState * pstate, CreateFunctionStmt * stmt) (\postgres\src\backend\commands\functioncmds.c:1152)
// ...... 前面的调用过程同最上面的调用栈，
```
不同之处在于`OidFunctionCall1Coll`的处理不同，在plpgsql过程语言中调用的是`plpgsql_validator`进行处理。


#### 调用函数
我们跟踪一下调用新创建的函数的过程：`select btax(2);`。
语法解析语义分析阶段，会调用`transformFuncCall`进行处理。其中最重要的是函数名、参数的处理。分别在`parse_func.c`、`parse_expr.c`中。
```c++
transformFuncCall(ParseState * pstate, FuncCall * fn) (postgres\src\backend\parser\parse_expr.c:1632)
transformExprRecurse(ParseState * pstate, Node * expr) (postgres\src\backend\parser\parse_expr.c:284)
transformExpr(ParseState * pstate, Node * expr, ParseExprKind exprKind) (postgres\src\backend\parser\parse_expr.c:173)
transformTargetEntry(ParseState * pstate, Node * node, Node * expr, ParseExprKind exprKind, char * colname, _Bool resjunk) (postgres\src\backend\parser\parse_target.c:122)
transformTargetList(ParseState * pstate, List * targetlist, ParseExprKind exprKind) (postgres\src\backend\parser\parse_target.c:210)
transformSelectStmt(ParseState * pstate, SelectStmt * stmt) (postgres\src\backend\parser\analyze.c:1275)
transformStmt(ParseState * pstate, Node * parseTree) (postgres\src\backend\parser\analyze.c:315)
transformOptionalSelectInto(ParseState * pstate, Node * parseTree) (postgres\src\backend\parser\analyze.c:253)
transformTopLevelStmt(ParseState * pstate, RawStmt * parseTree) (postgres\src\backend\parser\analyze.c:203)
parse_analyze(RawStmt * parseTree, const char * sourceText, Oid * paramTypes, int numParams, QueryEnvironment * queryEnv) (postgres\src\backend\parser\analyze.c:123)
pg_analyze_and_rewrite(RawStmt * parsetree, const char * query_string, Oid * paramTypes, int numParams, QueryEnvironment * queryEnv) (postgres\src\backend\tcop\postgres.c:700)
exec_simple_query(const char * query_string) (postgres\src\backend\tcop\postgres.c:1242)
```

在优化器阶段，调用`standard_planner`生成对应的执行计划`Result`：
```sql
postgres@postgres=# explain select btax(2);
                QUERY PLAN                
------------------------------------------
 Result  (cost=0.00..0.26 rows=1 width=4)
(1 row)
```
对应的调用栈如下：
```c++
standard_planner(Query * parse, const char * query_string, int cursorOptions, ParamListInfo boundParams) (postgres\src\backend\optimizer\plan\planner.c:511)
planner(Query * parse, const char * query_string, int cursorOptions, ParamListInfo boundParams) (postgres\src\backend\optimizer\plan\planner.c:283)
pg_plan_query(Query * querytree, const char * query_string, int cursorOptions, ParamListInfo boundParams) (postgres\src\backend\tcop\postgres.c:962)
pg_plan_queries(List * querytrees, const char * query_string, int cursorOptions, ParamListInfo boundParams) (postgres\src\backend\tcop\postgres.c:1053)
exec_simple_query(const char * query_string) (postgres\src\backend\tcop\postgres.c:1245)
```

生成了执行计划后，接下来执行执行计划，调用栈如下：
```c++
plpgsql.so!plpgsql_exec_function(PLpgSQL_function * func, FunctionCallInfo fcinfo, EState * simple_eval_estate, ResourceOwner simple_eval_resowner, _Bool atomic) (postgres\src\pl\plpgsql\src\pl_exec.c:726)
plpgsql.so!plpgsql_call_handler(FunctionCallInfo fcinfo) (postgres\src\pl\plpgsql\src\pl_handler.c:237)
ExecInterpExpr(ExprState * state, ExprContext * econtext, _Bool * isnull) (postgres\src\backend\executor\execExprInterp.c:682)
ExecInterpExprStillValid(ExprState * state, ExprContext * econtext, _Bool * isNull) (postgres\src\backend\executor\execExprInterp.c:1825)
ExecEvalExprSwitchContext(ExprState * state, ExprContext * econtext, _Bool * isNull) (postgres\src\include\executor\executor.h:323)
ExecProject(ProjectionInfo * projInfo) (postgres\src\include\executor\executor.h:357)
ExecResult(PlanState * pstate) (postgres\src\backend\executor\nodeResult.c:141)
ExecProcNodeFirst(PlanState * node) (postgres\src\backend\executor\execProcnode.c:450)
ExecProcNode(PlanState * node) (postgres\src\include\executor\executor.h:249)
ExecutePlan(EState * estate, PlanState * planstate, _Bool use_parallel_mode, CmdType operation, _Bool sendTuples, uint64 numberTuples, ScanDirection direction, DestReceiver * dest, _Bool execute_once) (postgres\src\backend\executor\execMain.c:1633)
standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (postgres\src\backend\executor\execMain.c:351)
ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count, _Bool execute_once) (postgres\src\backend\executor\execMain.c:295)
PortalRunSelect(Portal portal, _Bool forward, long count, DestReceiver * dest) (postgres\src\backend\tcop\pquery.c:916)
PortalRun(Portal portal, long count, _Bool isTopLevel, _Bool run_once, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (postgres\src\backend\tcop\pquery.c:760)
exec_simple_query(const char * query_string) (postgres\src\backend\tcop\postgres.c:1326)
```
可以看到，最终调用到了plpgsql中的`plpgsql_call_handler`中进行处理。其中调用`plpgsql_exec_function`返回函数执行的结果值。我们看一下其函数实现：
```c++
Datum plpgsql_call_handler(PG_FUNCTION_ARGS)
{
	bool		nonatomic;
	PLpgSQL_function *func;
	PLpgSQL_execstate *save_cur_estate;
	Datum		retval;
	int			rc;

	nonatomic = fcinfo->context && IsA(fcinfo->context, CallContext) && !castNode(CallContext, fcinfo->context)->atomic;

	/* Connect to SPI manager */
	if ((rc = SPI_connect_ext(nonatomic ? SPI_OPT_NONATOMIC : 0)) != SPI_OK_CONNECT)
		elog(ERROR, "SPI_connect failed: %s", SPI_result_code_string(rc));

	func = plpgsql_compile(fcinfo, false);	/* Find or compile the function */

	save_cur_estate = func->cur_estate;		/* Must save and restore prior value of cur_estate */
	func->use_count++;		/* Mark the function as busy, so it can't be deleted from under us */

	PG_TRY();
	{
		/* Determine if called as function or trigger and call appropriate subhandler */
		if (CALLED_AS_TRIGGER(fcinfo))
			retval = PointerGetDatum(plpgsql_exec_trigger(func, (TriggerData *) fcinfo->context));
		else if (CALLED_AS_EVENT_TRIGGER(fcinfo))
		{
			plpgsql_exec_event_trigger(func, (EventTriggerData *) fcinfo->context);
			retval = (Datum) 0;
		} else
			retval = plpgsql_exec_function(func, fcinfo, NULL, NULL, !nonatomic); // 调用执行函数
	}
	PG_FINALLY();
	{
		func->use_count--;		/* Decrement use-count, restore cur_estate */
		func->cur_estate = save_cur_estate;
	}
	PG_END_TRY();

	/* Disconnect from SPI manager */
	if ((rc = SPI_finish()) != SPI_OK_FINISH)
		elog(ERROR, "SPI_finish failed: %s", SPI_result_code_string(rc));

	return retval;
}
```

这里是函数的调用过程，后面会继续分析调用存储过程的调用过程。