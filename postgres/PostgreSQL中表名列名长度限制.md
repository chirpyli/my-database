
### PostgreSQL中表名，列名的长度限制
在PostgreSQL中，表名，列名长度是有限制的，不能任由客户端定义无限长的名称，这个长度限制被限制在63字符长度，超过的部分会被truncate掉，只保留63个字符长度。我们看一下其源码是怎么实现的。

如果是你自己实现这个限制，你会怎么做呢？ 肯定是先想好限制的长度，然后定义一个常量，在某个地方进行判断，超过这个常量的长度就进行截断处理。是的，这个思路没有错，PostgreSQL是在`src/include/pg_config_manual.h`中定义了标识符的最大命名长度64，因为包含一个`\0`，所以，实际上的最大长度是63。
```c
// Maximum length for identifiers (e.g. table names, column names, function names).  Names actually are limited to one less byte than this,
// because the length must include a trailing zero byte.
#define NAMEDATALEN 64
```

下一个问题就是，在哪里做这个判断呢？ 对于表名，列名，肯定是在定义阶段进行就要进行合法性检查，比如`create table tablename(column type, ......)`，也就是说，数据库在收到客户端发送来的建表语句的时候进行处理。我们想一下，收到SQL语句后数据库是怎么处理的，先进行词法语法分析，优化，执行......。我们后面根据这个流程，分析一下是在哪里进行的处理（一般，命名的合法性检查都是在前期就要做的，所以能够想到肯定是下词法语法分析阶段处理的，后面会详细说明处理过程）。因为表名，列名都是在SQL语句中定义，SQL语句由客户端发起，所以我们先看一下相关的处理流程，起点是`src/backend/main/main.c`。 接收到一个新的客户端请求，fork一个子进程单独处理该客户端的请求。
```c++
// Any Postgres server process begins execution here.
main(int argc, char *argv[])
    // Postmaster main entry point
--> PostmasterMain(argc, argv);
    // Main idle loop of postmaster ,New connection pending on any of our sockets? If so, fork a child process to deal with it.
    --> ServerLoop();
        // BackendStartup -- start backend process
        --> BackendStartup(port);
        	// BackendRun -- set up the backend's argument list and invoke PostgresMain()
		    --> BackendRun(port);
                // postgres main loop -- all backends, interactive or otherwise start here
                --> PostgresMain(ac, av, port->database_name, port->user_name);
                	// queries loop here.
	                --> for (;;)        // 在这里不断接收客户端的请求，处理
                        // tell dest that we are ready for a new query
                    	--> ReadyForQuery(whereToSendOutput);       // 执行完毕后，输出结果返回给客户端
                        // Execute a "simple Query" protocol message.
                        --> exec_simple_query(const char *query_string)
                            --> pg_parse_query(query_string);
```

### 源码分析
在处理客户端发起的SQL语句的时候，需要先进行词法语法分析。我们看一下`raw_parser()`的源码：
```c++
// raw_parser	Given a query in string form, do lexical and grammatical analysis.
List *raw_parser(const char *str) {
	core_yyscan_t yyscanner;
	base_yy_extra_type yyextra;
	int			yyresult;

	/* initialize the flex scanner */
	yyscanner = scanner_init(str, &yyextra.core_yy_extra, &ScanKeywords, ScanKeywordTokens);  // 词法分析器

	/* base_yylex() only needs this much initialization */
	yyextra.have_lookahead = false;

	/* initialize the bison parser */
	parser_init(&yyextra);

	/* Parse! */
	yyresult = base_yyparse(yyscanner);

	/* Clean up (release memory) */
	scanner_finish(yyscanner);

	if (yyresult)				/* error */
		return NIL;

	return yyextra.parsetree;
}
```
在调用`base_yyparse`时，会调用`yylex`，也就是下面的`base_yylex`，
```c++
int yyparse (core_yyscan_t yyscanner) { 
// 忽略部分代码...

  /* YYCHAR is either YYEMPTY or YYEOF or a valid lookahead symbol.  */
  if (yychar == YYEMPTY) {
      YYDPRINTF ((stderr, "Reading a token: "));
      yychar = yylex (&yylval, &yylloc, yyscanner);         // 调用yylex
    }

  if (yychar <= YYEOF) {
      yychar = yytoken = YYEOF;
      YYDPRINTF ((stderr, "Now at end of input.\n"));
    } else {
      yytoken = YYTRANSLATE (yychar);
      YY_SYMBOL_PRINT ("Next token is", yytoken, &yylval, &yylloc);
    }

// 忽略部分代码...
}

#define yylex           base_yylex      // 调用base_yylex

// 这里会调用`core_yylex`，在scan.c文件中，由scan.l生成.
// Intermediate filter between parser and core lexer (core_yylex in scan.l).
int base_yylex(YYSTYPE *lvalp, YYLTYPE *llocp, core_yyscan_t yyscanner) {
	base_yy_extra_type *yyextra = pg_yyget_extra(yyscanner);
	int			cur_token;
	int			next_token;
	int			cur_token_length;
	YYLTYPE		cur_yylloc;

	/* Get next token --- we might already have it */
	if (yyextra->have_lookahead) 
	{
		cur_token = yyextra->lookahead_token;
		lvalp->core_yystype = yyextra->lookahead_yylval;
		*llocp = yyextra->lookahead_yylloc;
		*(yyextra->lookahead_end) = yyextra->lookahead_hold_char;
		yyextra->have_lookahead = false;
	} else
		cur_token = core_yylex(&(lvalp->core_yystype), llocp, yyscanner);

	/* Check for special handling of PARTITION keyword. (see OptFirstPartitionSpec rule in the grammar)*/
	if (yyextra->tail_partition_magic)
	{
		if (cur_token == PARTITION)
		{
			yyextra->tail_partition_magic = false;
			return PARTITION_TAIL;
		}
	}

	/* If this token isn't one that requires lookahead, just return it.  If it
	 * does, determine the token length.  (We could get that via strlen(), but
	 * since we have such a small set of possibilities, hardwiring seems
	 * feasible and more efficient --- at least for the fixed-length cases.)*/
	switch (cur_token)
	{
		case NOT:
			cur_token_length = 3;
			break;
		case NULLS_P:
			cur_token_length = 5;
			break;
		case WITH:
			cur_token_length = 4;
			break;
		case UIDENT:
		case USCONST:
			cur_token_length = strlen(yyextra->core_yy_extra.scanbuf + *llocp);
			break;
		default:
			return cur_token;
	}

	/* Identify end+1 of current token.  core_yylex() has temporarily stored a
	 * '\0' here, and will undo that when we call it again.  We need to redo
	 * it to fully revert the lookahead call for error reporting purposes. */
	yyextra->lookahead_end = yyextra->core_yy_extra.scanbuf +
		*llocp + cur_token_length;
	Assert(*(yyextra->lookahead_end) == '\0');

	/* Save and restore *llocp around the call.  It might look like we could
	 * avoid this by just passing &lookahead_yylloc to core_yylex(), but that
	 * does not work because flex actually holds onto the last-passed pointer
	 * internally, and will use that for error reporting.  We need any error
	 * reports to point to the current token, not the next one. */
	cur_yylloc = *llocp;

	/* Get next token, saving outputs into lookahead variables */
	next_token = core_yylex(&(yyextra->lookahead_yylval), llocp, yyscanner);
	yyextra->lookahead_token = next_token;
	yyextra->lookahead_yylloc = *llocp;

	*llocp = cur_yylloc;

	/* Now revert the un-truncation of the current token */
	yyextra->lookahead_hold_char = *(yyextra->lookahead_end);
	*(yyextra->lookahead_end) = '\0';

	yyextra->have_lookahead = true;

	/* Replace cur_token if needed, based on lookahead */
	switch (cur_token)
	{
		case NOT:
			/* Replace NOT by NOT_LA if it's followed by BETWEEN, IN, etc */
			switch (next_token)
			{
				case BETWEEN:
				case IN_P:
				case LIKE:
				case ILIKE:
				case SIMILAR:
					cur_token = NOT_LA;
					break;
			}
			break;

		case NULLS_P:
			/* Replace NULLS_P by NULLS_LA if it's followed by FIRST or LAST */
			switch (next_token)
			{
				case FIRST_P:
				case LAST_P:
					cur_token = NULLS_LA;
					break;
			}
			break;

		case WITH:
			/* Replace WITH by WITH_LA if it's followed by TIME or ORDINALITY */
			switch (next_token)
			{
				case TIME:
				case ORDINALITY:
					cur_token = WITH_LA;
					break;
			}
			break;

		case UIDENT:
		case USCONST:
			/* Look ahead for UESCAPE */
			if (next_token == UESCAPE)
			{
				/* Yup, so get third token, which had better be SCONST */
				const char *escstr;

				/* Again save and restore *llocp */
				cur_yylloc = *llocp;

				/* Un-truncate current token so errors point to third token */
				*(yyextra->lookahead_end) = yyextra->lookahead_hold_char;

				/* Get third token */
				next_token = core_yylex(&(yyextra->lookahead_yylval), llocp, yyscanner);

				/* If we throw error here, it will point to third token */
				if (next_token != SCONST)
					scanner_yyerror("UESCAPE must be followed by a simple string literal", yyscanner);

				escstr = yyextra->lookahead_yylval.str;
				if (strlen(escstr) != 1 || !check_uescapechar(escstr[0]))
					scanner_yyerror("invalid Unicode escape character", yyscanner);

				/* Now restore *llocp; errors will point to first token */
				*llocp = cur_yylloc;

				/* Apply Unicode conversion */
				lvalp->core_yystype.str = str_udeescape(lvalp->core_yystype.str, escstr[0], *llocp, yyscanner);

				/* We don't need to revert the un-truncation of UESCAPE.  What
				 * we do want to do is clear have_lookahead, thereby consuming all three tokens. */
				yyextra->have_lookahead = false;
			}
			else
			{
				/* No UESCAPE, so convert using default escape character */
				lvalp->core_yystype.str = str_udeescape(lvalp->core_yystype.str, '\\', *llocp, yyscanner);
			}

			if (cur_token == UIDENT)
			{
				/* It's an identifier, so truncate as appropriate */
				truncate_identifier(lvalp->core_yystype.str, strlen(lvalp->core_yystype.str), true);
				cur_token = IDENT;
			} else if (cur_token == USCONST)
			{
				cur_token = SCONST;
			}
			break;
	}

	return cur_token;
}
```
在进行词法分析的时候，会调用如下代码：
```c++
// 通过这个函数处理前面规则段定义
extern int core_yylex(YYSTYPE * yylval_param,YYLTYPE * yylloc_param ,yyscan_t yyscanner);

#define YY_DECL int core_yylex \
               (YYSTYPE * yylval_param, YYLTYPE * yylloc_param , yyscan_t yyscanner)


/** The main scanner function which does all the work. */
YY_DECL
{
	register yy_state_type yy_current_state;
	register char *yy_cp, *yy_bp;
	register int yy_act;
    struct yyguts_t * yyg = (struct yyguts_t*)yyscanner;

#line 416 "scan.l"


#line 5414 "scan.c"

    yylval = yylval_param;

    yylloc = yylloc_param;

	if ( !yyg->yy_init )
		{
		yyg->yy_init = 1;

#ifdef YY_USER_INIT
		YY_USER_INIT;
#endif

		if ( ! yyg->yy_start )
			yyg->yy_start = 1;	/* first start state */

		if ( ! yyin )
			yyin = stdin;

		if ( ! yyout )
			yyout = stdout;

		if ( ! YY_CURRENT_BUFFER ) {
			core_yyensure_buffer_stack (yyscanner);
			YY_CURRENT_BUFFER_LVALUE =
				core_yy_create_buffer(yyin,YY_BUF_SIZE ,yyscanner);
		}

		core_yy_load_buffer_state(yyscanner );
		}

	while ( 1 )		/* loops until end-of-file is reached */
		{
		yy_cp = yyg->yy_c_buf_p;

		/* Support of yytext. */
		*yy_cp = yyg->yy_hold_char;

		/* yy_bp points to the position in yy_ch_buf of the start of the current run. */
		yy_bp = yy_cp;

		yy_current_state = yy_start_state_list[yyg->yy_start];
yy_match:
		{
		register yyconst struct yy_trans_info *yy_trans_info;

		register YY_CHAR yy_c;

		for ( yy_c = YY_SC_TO_UI(*yy_cp);
		      (yy_trans_info = &yy_current_state[(unsigned int) yy_c])->
		yy_verify == yy_c;
		      yy_c = YY_SC_TO_UI(*++yy_cp) )
			yy_current_state += yy_trans_info->yy_nxt;
		}

yy_find_action:
		yy_act = yy_current_state[-1].yy_nxt;

		YY_DO_BEFORE_ACTION;

do_action:	/* This label is used only to access EOF actions. */

		switch ( yy_act )
	{ /* beginning of action switch */
case 1:
/* rule 1 can match eol */
YY_RULE_SETUP
#line 418 "scan.l"
{
					/* ignore */
				}
	YY_BREAK
case 2:
// 中间其他规则省略......
case 62:
YY_RULE_SETUP
#line 1012 "scan.l"
{
					int			kwnum;
					char	   *ident;

					SET_YYLLOC();

					/* Is it a keyword? */
					kwnum = ScanKeywordLookup(yytext,
											  yyextra->keywordlist);   //see if a given word is a keyword
					if (kwnum >= 0)
					{
						yylval->keyword = GetScanKeyword(kwnum,
														 yyextra->keywordlist);
						return yyextra->keyword_tokens[kwnum];
					}

					/* No.  Convert the identifier to lower case, and truncate if necessary. */
					ident = downcase_truncate_identifier(yytext, yyleng, true);     // 在这个地方，如果超出了63字符长度，会将其截断
					yylval->str = ident;
					return IDENT;
				}
	YY_BREAK

	default:
		YY_FATAL_ERROR(
			"fatal flex scanner internal error--no action found" );
	} /* end of action switch */
		} /* end of scanning one token */
} /* end of core_yylex */


char* downcase_truncate_identifier(const char *ident, int len, bool warn) {
	return downcase_identifier(ident, len, warn, true);
}

// a workhorse for downcase_truncate_identifier
char *downcase_identifier(const char *ident, int len, bool warn, bool truncate) {
	char	   *result;
	int			i;
	bool		enc_is_single_byte;

	result = palloc(len + 1);
	enc_is_single_byte = pg_database_encoding_max_length() == 1;

	/* SQL99 specifies Unicode-aware case normalization, which we don't yet
	 * have the infrastructure for.  Instead we use tolower() to provide a
	 * locale-aware translation.  However, there are some locales where this
	 * is not right either (eg, Turkish may do strange things with 'i' and
	 * 'I').  Our current compromise is to use tolower() for characters with
	 * the high bit set, as long as they aren't part of a multi-byte
	 * character, and use an ASCII-only downcasing for 7-bit characters. */
	for (i = 0; i < len; i++) {
		unsigned char ch = (unsigned char) ident[i];

		if (ch >= 'A' && ch <= 'Z')
			ch += 'a' - 'A';
		else if (enc_is_single_byte && IS_HIGHBIT_SET(ch) && isupper(ch))
			ch = tolower(ch);
		result[i] = (char) ch;
	}
	result[i] = '\0';

	if (i >= NAMEDATALEN && truncate)           // 如果超出了NAMEDATALEN的长度，则截断
		truncate_identifier(result, i, warn);

	return result;
}

/* truncate_identifier() --- truncate an identifier to NAMEDATALEN-1 bytes.
 * The given string is modified in-place, if necessary.  A warning is issued if requested.
 * We require the caller to pass in the string length since this saves a strlen() call in some common usages. */
void truncate_identifier(char *ident, int len, bool warn) {
	if (len >= NAMEDATALEN)
	{
		len = pg_mbcliplen(ident, len, NAMEDATALEN - 1);
		if (warn)
		{
			/* We avoid using %.*s here because it can misbehave if the data
			 * is not valid in what libc thinks is the prevailing encoding. */
			char		buf[NAMEDATALEN];

			memcpy(buf, ident, len);
			buf[len] = '\0';
			ereport(NOTICE, (errcode(ERRCODE_NAME_TOO_LONG), errmsg("identifier \"%s\" will be truncated to \"%s\"", ident, buf)));
		}
		ident[len] = '\0';
	}
}
```
总结一下，识别表名，列名肯定是在词法分析阶段去做的，那么判断其长度是否需要截断也一定是在这个处理过程中实现的。