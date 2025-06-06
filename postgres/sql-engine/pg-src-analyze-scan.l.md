### PostgreSQL中scan.l源码分析
`scan.l` 是 PostgreSQL SQL 解析器的词法分析器源码，基于 Flex（lex）工具生成。
`scan.l` 是 PostgreSQL SQL 解析的第一步，负责将输入 SQL 文本分割成 token，处理字符串、数字、关键字、注释、转义等各种 SQL 语法细节，并为后续的语法分析（Bison 生成的 `gram.c`）提供输入。
```c++

// 定义部分， %top{ ... }，%{ ... %}，内容都是C代码，并将代码照搬到生成的C文件中，%top{ ... }是将其中的代码拷贝到scan.c文件中的顶部
%top{
/*-------------------------------------------------------------------------
 *
 * scan.l
 *	  lexical scanner for PostgreSQL
 *
 * NOTE NOTE NOTE:
 *
 * The rules in this file must be kept in sync with src/fe_utils/psqlscan.l
 * and src/interfaces/ecpg/preproc/pgc.l!
 *
 * The rules are designed so that the scanner never has to backtrack,
 * in the sense that there is always a rule that can match the input
 * consumed so far (the rule action may internally throw back some input
 * with yyless(), however).  As explained in the flex manual, this makes
 * for a useful speed increase --- several percent faster when measuring
 * raw parsing (Flex + Bison).  The extra complexity is mostly in the rules
 * for handling float numbers and continued string literals.  If you change
 * the lexical rules, verify that you haven't broken the no-backtrack
 * property by running flex with the "-b" option and checking that the
 * resulting "lex.backup" file says that no backing up is needed.  (As of
 * Postgres 9.2, this check is made automatically by the Makefile.)
 *
 *
 * Portions Copyright (c) 1996-2020, PostgreSQL Global Development Group
 * Portions Copyright (c) 1994, Regents of the University of California
 *
 * IDENTIFICATION
 *	  src/backend/parser/scan.l
 *
 *-------------------------------------------------------------------------
 */
#include "postgres.h"

#include <ctype.h>
#include <unistd.h>

#include "common/string.h"
#include "parser/gramparse.h"
#include "parser/parser.h"		/* only needed for GUC variables */
#include "parser/scansup.h"
#include "mb/pg_wchar.h"
}

%{

/* LCOV_EXCL_START */

/* Avoid exit() on fatal scanner errors (a bit ugly -- see yy_fatal_error) */
#undef fprintf
#define fprintf(file, fmt, msg)  fprintf_to_ereport(fmt, msg)

static void
fprintf_to_ereport(const char *fmt, const char *msg)
{
	ereport(ERROR, (errmsg_internal("%s", msg)));
}

/*
 * GUC variables.  This is a DIRECT violation of the warning given at the
 * head of gram.y, ie flex/bison code must not depend on any GUC variables;
 * as such, changing their values can induce very unintuitive behavior.
 * But we shall have to live with it until we can remove these variables.
 */
int			backslash_quote = BACKSLASH_QUOTE_SAFE_ENCODING;
bool		escape_string_warning = true;
bool		standard_conforming_strings = true;

/*
 * Constant data exported from this file.  This array maps from the
 * zero-based keyword numbers returned by ScanKeywordLookup to the
 * Bison token numbers needed by gram.y.  This is exported because
 * callers need to pass it to scanner_init, if they are using the
 * standard keyword list ScanKeywords.
 */
#define PG_KEYWORD(kwname, value, category) value,

const uint16 ScanKeywordTokens[] = {
#include "parser/kwlist.h"
};

#undef PG_KEYWORD

/*
 * Set the type of YYSTYPE.
 */
#define YYSTYPE core_YYSTYPE

/*
 * Set the type of yyextra.  All state variables used by the scanner should
 * be in yyextra, *not* statically allocated.
 */
#define YY_EXTRA_TYPE core_yy_extra_type *

/*
 * Each call to yylex must set yylloc to the location of the found token
 * (expressed as a byte offset from the start of the input text).
 * When we parse a token that requires multiple lexer rules to process,
 * this should be done in the first such rule, else yylloc will point
 * into the middle of the token.
 */
#define SET_YYLLOC()  (*(yylloc) = yytext - yyextra->scanbuf)

/*
 * Advance yylloc by the given number of bytes.
 */
#define ADVANCE_YYLLOC(delta)  ( *(yylloc) += (delta) )

/*
 * Sometimes, we do want yylloc to point into the middle of a token; this is
 * useful for instance to throw an error about an escape sequence within a
 * string literal.  But if we find no error there, we want to revert yylloc
 * to the token start, so that that's the location reported to the parser.
 * Use PUSH_YYLLOC/POP_YYLLOC to save/restore yylloc around such code.
 * (Currently the implied "stack" is just one location, but someday we might
 * need to nest these.)
 */
#define PUSH_YYLLOC()	(yyextra->save_yylloc = *(yylloc))
#define POP_YYLLOC()	(*(yylloc) = yyextra->save_yylloc)

#define startlit()	( yyextra->literallen = 0 )
static void addlit(char *ytext, int yleng, core_yyscan_t yyscanner);
static void addlitchar(unsigned char ychar, core_yyscan_t yyscanner);
static char *litbufdup(core_yyscan_t yyscanner);
static unsigned char unescape_single_char(unsigned char c, core_yyscan_t yyscanner);
static int	process_integer_literal(const char *token, YYSTYPE *lval);
static void addunicode(pg_wchar c, yyscan_t yyscanner);

#define yyerror(msg)  scanner_yyerror(msg, yyscanner)

#define lexer_errposition()  scanner_errposition(*(yylloc), yyscanner)

static void check_string_escape_warning(unsigned char ychar, core_yyscan_t yyscanner);
static void check_escape_warning(core_yyscan_t yyscanner);

/*
 * Work around a bug in flex 2.5.35: it emits a couple of functions that
 * it forgets to emit declarations for.  Since we use -Wmissing-prototypes,
 * this would cause warnings.  Providing our own declarations should be
 * harmless even when the bug gets fixed.
 */
extern int	core_yyget_column(yyscan_t yyscanner);
extern void core_yyset_column(int column_no, yyscan_t yyscanner);

%}

%option reentrant       // %option reentrant: flex能够生成一个可重入的扫描器。通过定义%option reentrant（与-R选项等价）来实现可重入。所生成的扫描器在一个或多个控制线程中不仅可移植，而且安全性好。可重入扫描器通常应用于多线程应用程序。任何一个线程都可以在不考虑与其他线程同步的情况下创建并执行一个可重入的flex扫描器。默认情况下，flex生成一个不可重入的扫描器。本项目为了实现多线程，因而在定义段指定%option reentrant.

%option bison-bridge  // flex和bison联合使用的选项，由于flex和bison部分内容声明不同，使用此选项后可以相互调用。
%option bison-locations  //此模式同上面参数同时使用，如果做了此声明，yylex 将被声明为int yylex (YYSTYPE* lvalp, YYLTYPE* llocp, yyscan_t scaninfo);加入了location参数.
%option 8bit        // 识别其输入中8位字符的扫描器。
%option never-interactive   // flex 生成的扫描器从不认为输入是交互的（不会调用isatty()
%option nodefault       // 所有没有被匹配的输入都拷贝到yyoutflex允许在添加%option nodefault，使它不要添加默认的规则这样输入无法被给定的规则完全匹配时，词法分析器可以报告一个错误。
%option noinput     // 不使用默认的input函数
%option nounput     // 添加Flex默认的C函数，比如yy_scan_buffer，yy_scan_bytes，yy_scan_string
%option noyywrap    // 不使用yywrap()，当lex读取到文件末尾时，会调用yywrap(), 目的是，当有另外一个输入文件时，yywrap可以调整yyin的值并且返回0来重新开始词法分析。如果是真正的文件末尾，那么就返回1来完成分析。
%option noyyalloc   // 不使用flex自带的函数yyalloc，yyrealloc，yyfree（内存的申请，重申请以及释放）
%option noyyrealloc
%option noyyfree
%option warn        // 开启所有警告
%option prefix="core_yy"    // 以将原来的yylex等函数 变成core_yylex.这样可以在一个程序中建立多个词法分析器。

/*
 * OK, here is a short description of lex/flex rules behavior.
 * The longest pattern which matches an input string is always chosen.
 * For equal-length patterns, the first occurring in the rules list is chosen.
 * INITIAL is the starting state, to which all non-conditional rules apply.
 * Exclusive states change parsing rules while the state is active.  When in
 * an exclusive state, only those rules defined for that state apply.
 *
 * We use exclusive states for quoted strings, extended comments,
 * and to eliminate parsing troubles for numeric strings.
 * Exclusive states:
 *  <xb> bit string literal
 *  <xc> extended C-style comments
 *  <xd> delimited identifiers (double-quoted identifiers)
 *  <xh> hexadecimal numeric string
 *  <xq> standard quoted strings
 *  <xqs> quote stop (detect continued strings)
 *  <xe> extended quoted strings (support backslash escape sequences)
 *  <xdolq> $foo$ quoted strings
 *  <xui> quoted identifier with Unicode escapes
 *  <xus> quoted string with Unicode escapes
 *  <xeu> Unicode surrogate pair in extended quoted string
 *
 * Remember to add an <<EOF>> case whenever you add a new exclusive state!
 * The default one is probably not the right thing.
 */
// 状态定义, 最开始是INITIAL状态
%x xb           // 位字符串
%x xc           // 扩展的C-style注释
%x xd           // 分隔标识符(双引号标识符)
%x xh           // 十六进制数字字符串
%x xq           // 标准的带引号的字符串
%x xqs
%x xe           // 扩展的带引号的字符串(支持反斜杠转义序列)
%x xdolq
%x xui          // 带Unicode转义的带引号的标识符
%x xus          // 带Unicode转义的带引号的字符串
%x xeu

/*
 * In order to make the world safe for Windows and Mac clients as well as
 * Unix ones, we accept either \n or \r as a newline.  A DOS-style \r\n
 * sequence will be seen as two successive newlines, but that doesn't cause
 * any problems.  Comments that start with -- and extend to the next
 * newline are treated as equivalent to a single whitespace character.
 *
 * NOTE a fine point: if there is no newline following --, we will absorb
 * everything to the end of the input as a comment.  This is correct.  Older
 * versions of Postgres failed to recognize -- as a comment if the input
 * did not end with a newline.
 *
 * XXX perhaps \f (formfeed) should be treated as a newline as well?
 *
 * XXX if you change the set of whitespace characters, fix scanner_isspace()
 * to agree.
 */

// 为正则表达式命名，后面可以用名字代替后面的正则表达式

// //\t -->Tab键,\n -->换行,\t -->回车,\f -->换页
space			[ \t\n\r\f]     // tab键/换行/回车/换页
horiz_space		[ \t\f]         // 空格/tab键/换页
newline			[\n\r]          // 换行/回车
non_newline		[^\n\r]

comment			("--"{non_newline}*)        // //单行注释

whitespace		({space}+|{comment})        // //空白字符(1个或以上空格或者注释均视为whitespace)

/*
 * SQL requires at least one newline in the whitespace separating
 * string literals that are to be concatenated.  Silly, but who are we
 * to argue?  Note that {whitespace_with_newline} should not have * after
 * it, whereas {whitespace} should generally have a * after it...
 */

special_whitespace		({space}+|{comment}{newline})
horiz_whitespace		({horiz_space}|{comment})
whitespace_with_newline	({horiz_whitespace}*{newline}{special_whitespace}*)

quote			'
/* If we see {quote} then {quotecontinue}, the quoted string continues */
quotecontinue	{whitespace_with_newline}{quote}

/*
 * {quotecontinuefail} is needed to avoid lexer backup when we fail to match
 * {quotecontinue}.  It might seem that this could just be {whitespace}*,
 * but if there's a dash after {whitespace_with_newline}, it must be consumed
 * to see if there's another dash --- which would start a {comment} and thus
 * allow continuation of the {quotecontinue} token.
 */
quotecontinuefail	{whitespace}*"-"?

/* Bit string
 * It is tempting to scan the string for only those characters
 * which are allowed. However, this leads to silently swallowed
 * characters if illegal characters are included in the string.
 * For example, if xbinside is [01] then B'ABCD' is interpreted
 * as a zero-length string, and the ABCD' is lost!
 * Better to pass the string forward and let the input routines
 * validate the contents.
 */
xbstart			[bB]{quote}
xbinside		[^']*

/* Hexadecimal number */
xhstart			[xX]{quote}
xhinside		[^']*

/* National character */
xnstart			[nN]{quote}

/* Quoted string that allows backslash escapes */
xestart			[eE]{quote}
xeinside		[^\\']+
xeescape		[\\][^0-7]
xeoctesc		[\\][0-7]{1,3}
xehexesc		[\\]x[0-9A-Fa-f]{1,2}
xeunicode		[\\](u[0-9A-Fa-f]{4}|U[0-9A-Fa-f]{8})
xeunicodefail	[\\](u[0-9A-Fa-f]{0,3}|U[0-9A-Fa-f]{0,7})

/* Extended quote
 * xqdouble implements embedded quote, ''''
 */
xqstart			{quote}
xqdouble		{quote}{quote}
xqinside		[^']+

/* $foo$ style quotes ("dollar quoting")
 * The quoted string starts with $foo$ where "foo" is an optional string
 * in the form of an identifier, except that it may not contain "$",
 * and extends to the first occurrence of an identical string.
 * There is *no* processing of the quoted text.
 *
 * {dolqfailed} is an error rule to avoid scanner backup when {dolqdelim}
 * fails to match its trailing "$".
 */
dolq_start		[A-Za-z\200-\377_]
dolq_cont		[A-Za-z\200-\377_0-9]
dolqdelim		\$({dolq_start}{dolq_cont}*)?\$
dolqfailed		\${dolq_start}{dolq_cont}*
dolqinside		[^$]+

/* Double quote
 * Allows embedded spaces and other special characters into identifiers.
 */
dquote			\"                      // 双引号
xdstart			{dquote}
xdstop			{dquote}
xddouble		{dquote}{dquote}
xdinside		[^"]+

/* Quoted identifier with Unicode escapes */
xuistart		[uU]&{dquote}

/* Quoted string with Unicode escapes */
xusstart		[uU]&{quote}

/* error rule to avoid backup */
xufailed		[uU]&


/* C-style comments
 *
 * The "extended comment" syntax closely resembles allowable operator syntax.
 * The tricky part here is to get lex to recognize a string starting with
 * slash-star as a comment, when interpreting it as an operator would produce
 * a longer match --- remember lex will prefer a longer match!  Also, if we
 * have something like plus-slash-star, lex will think this is a 3-character
 * operator whereas we want to see it as a + operator and a comment start.
 * The solution is two-fold:
 * 1. append {op_chars}* to xcstart so that it matches as much text as
 *    {operator} would. Then the tie-breaker (first matching rule of same
 *    length) ensures xcstart wins.  We put back the extra stuff with yyless()
 *    in case it contains a star-slash that should terminate the comment.
 * 2. In the operator rule, check for slash-star within the operator, and
 *    if found throw it back with yyless().  This handles the plus-slash-star
 *    problem.
 * Dash-dash comments have similar interactions with the operator rule.
 */
"扩展注释"语法与允许的操作符语法非常相似。这里比较棘手的部分是让词法分析器可以识别以斜杠加星号开头的字符串作为注释,
因为在认为星号为操作符时可能会产生更长的匹配 -- 记住:词法分析器倾向于更长的匹配.同时,如果存在形如"+/*"这样的字符串,
词法分析器会认为这是3元操作符,但其实我们希望把它视作一个加号操作符和注释的开始.
解决方案如下:
  1.追加{op_chars}*到xcstart中,以便它可以匹配尽可能多的文本(与{operator}一样).然后,tie-breaker(相同长度首次匹配原则)确保xcstart会首先匹配.
  我们用yyless()放进去了一些额外的东西,以防它包含一个星号和斜杠(即:"*/"),这会终止注释
  2.在操作符规则中,检查操作符中的反斜杠+星号,如发现则返回给yyless().这可以处理"+/*"这个问题"--"注释与操作符规则有类型的交互方式.

xcstart			\/\*{op_chars}*
xcstop			\*+\/
xcinside		[^*/]+

digit			[0-9]           // 数字:0-9
ident_start		[A-Za-z\200-\377_]          // 标识符开始:英文字母/80-FF字符/下划线
ident_cont		[A-Za-z\200-\377_0-9\$]     // 标识符:ident_start外加数字

identifier		{ident_start}{ident_cont}*      // 标识符

/* Assorted special-case operators and operator-like tokens */
typecast		"::"
dot_dot			\.\.
colon_equals	":="

/*
 * These operator-like tokens (unlike the above ones) also match the {operator}
 * rule, which means that they might be overridden by a longer match if they
 * are followed by a comment start or a + or - character. Accordingly, if you
 * add to this list, you must also add corresponding code to the {operator}
 * block to return the correct token in such cases. (This is not needed in
 * psqlscan.l since the token value is ignored there.)
 */
equals_greater	"=>"
less_equals		"<="
greater_equals	">="
less_greater	"<>"
not_equals		"!="

/*
 * "self" is the set of chars that should be returned as single-character
 * tokens.  "op_chars" is the set of chars that can make up "Op" tokens,
 * which can be one or more characters long (but if a single-char token
 * appears in the "self" set, it is not to be returned as an Op).  Note
 * that the sets overlap, but each has some chars that are not in the other.
 *
 * If you change either set, adjust the character lists appearing in the
 * rule for "operator"!
 */
self			[,()\[\].;\:\+\-\*\/\%\^\<\>\=]
op_chars		[\~\!\@\#\^\&\|\`\?\+\-\*\/\%\<\>\=]
operator		{op_chars}+

/* we no longer allow unary minus in numbers.
 * instead we pass it separately to parser. there it gets
 * coerced via doNegate() -- Leon aug 20 1999
 *
 * {decimalfail} is used because we would like "1..10" to lex as 1, dot_dot, 10.
 *
 * {realfail1} and {realfail2} are added to prevent the need for scanner
 * backup when the {real} rule fails to match completely.
 */

integer			{digit}+
decimal			(({digit}*\.{digit}+)|({digit}+\.{digit}*))
decimalfail		{digit}+\.\.
real			({integer}|{decimal})[Ee][-+]?{digit}+
realfail1		({integer}|{decimal})[Ee]
realfail2		({integer}|{decimal})[Ee][-+]

param			\${integer}

other			.

/*
 * Dollar quoted strings are totally opaque, and no escaping is done on them.
 * Other quoted strings must allow some special characters such as single-quote
 *  and newline.
 * Embedded single-quotes are implemented both in the SQL standard
 *  style of two adjacent single quotes "''" and in the Postgres/Java style
 *  of escaped-quote "\'".
 * Other embedded escaped characters are matched explicitly and the leading
 *  backslash is dropped from the string.
 * Note that xcstart must appear before operator, as explained above!
 *  Also whitespace (comment) must appear before operator.
 */

%%
// 规则部分
/*
可以看到下面有很多BEGIN(...)的代码部分， BEGIN, Flex 初始状态为 INITIAL, BEGIN (start stat) 意思是开始进入一个新的状态. 进入这个状态后，只有定义了对应状态的规则才会被匹配。例如：上面调用BEGIN(xc)之后，即匹配xcstart之后，进入xc状态，所以只有<xc>{xcstart} { ... }等开头带<xc>的代理器才会被匹配。其他的任何规则都不会被匹配。

yyextra， 此变量是用户传入的参数数据

yylval，yylex(在Bison yyparse 函数中调用)返回值可以理解分为两部分，一个是在规则中的return 值，此为返回的token, 另一个是与之一一对应的yylval。 yylval 的类型为YYSTYPE, 可以根据用户需要重新定义
*/

{whitespace}	{
					/* ignore */
				}

{xcstart}		{
					/* Set location in case of syntax error in comment */
					SET_YYLLOC();
					yyextra->xcdepth = 0;
					BEGIN(xc);
					/* Put back any characters past slash-star; see above */
					yyless(2);
				}

<xc>{
{xcstart}		{
					(yyextra->xcdepth)++;
					/* Put back any characters past slash-star; see above */
					yyless(2);
				}

{xcstop}		{
					if (yyextra->xcdepth <= 0)
						BEGIN(INITIAL);
					else
						(yyextra->xcdepth)--;
				}

{xcinside}		{
					/* ignore */
				}

{op_chars}		{
					/* ignore */
				}

\*+				{
					/* ignore */
				}

<<EOF>>			{
					yyerror("unterminated /* comment");
				}
} /* <xc> */

{xbstart}		{
					/* Binary bit type.
					 * At some point we should simply pass the string
					 * forward to the parser and label it there.
					 * In the meantime, place a leading "b" on the string
					 * to mark it for the input routine as a binary string.
					 */
					SET_YYLLOC();
					BEGIN(xb);
					startlit();
					addlitchar('b', yyscanner);
				}
<xh>{xhinside}	|
<xb>{xbinside}	{
					addlit(yytext, yyleng, yyscanner);
				}
<xb><<EOF>>		{ yyerror("unterminated bit string literal"); }

{xhstart}		{
					/* Hexadecimal bit type.
					 * At some point we should simply pass the string
					 * forward to the parser and label it there.
					 * In the meantime, place a leading "x" on the string
					 * to mark it for the input routine as a hex string.
					 */
					SET_YYLLOC();
					BEGIN(xh);
					startlit();
					addlitchar('x', yyscanner);
				}
<xh><<EOF>>		{ yyerror("unterminated hexadecimal string literal"); }

{xnstart}		{
					/* National character.
					 * We will pass this along as a normal character string,
					 * but preceded with an internally-generated "NCHAR".
					 */
					int		kwnum;

					SET_YYLLOC();
					yyless(1);	/* eat only 'n' this time */

					kwnum = ScanKeywordLookup("nchar",
											  yyextra->keywordlist);
					if (kwnum >= 0)
					{
						yylval->keyword = GetScanKeyword(kwnum,
														 yyextra->keywordlist);
						return yyextra->keyword_tokens[kwnum];
					}
					else
					{
						/* If NCHAR isn't a keyword, just return "n" */
						yylval->str = pstrdup("n");
						return IDENT;
					}
				}

{xqstart}		{
					yyextra->warn_on_first_escape = true;
					yyextra->saw_non_ascii = false;
					SET_YYLLOC();
					if (yyextra->standard_conforming_strings)
						BEGIN(xq);
					else
						BEGIN(xe);
					startlit();
				}
{xestart}		{
					yyextra->warn_on_first_escape = false;
					yyextra->saw_non_ascii = false;
					SET_YYLLOC();
					BEGIN(xe);
					startlit();
				}
{xusstart}		{
					SET_YYLLOC();
					if (!yyextra->standard_conforming_strings)
						ereport(ERROR,
								(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
								 errmsg("unsafe use of string constant with Unicode escapes"),
								 errdetail("String constants with Unicode escapes cannot be used when standard_conforming_strings is off."),
								 lexer_errposition()));
					BEGIN(xus);
					startlit();
				}

<xb,xh,xq,xe,xus>{quote} {
					/*
					 * When we are scanning a quoted string and see an end
					 * quote, we must look ahead for a possible continuation.
					 * If we don't see one, we know the end quote was in fact
					 * the end of the string.  To reduce the lexer table size,
					 * we use a single "xqs" state to do the lookahead for all
					 * types of strings.
					 */
					yyextra->state_before_str_stop = YYSTATE;
					BEGIN(xqs);
				}
<xqs>{quotecontinue} {
					/*
					 * Found a quote continuation, so return to the in-quote
					 * state and continue scanning the literal.  Nothing is
					 * added to the literal's contents.
					 */
					BEGIN(yyextra->state_before_str_stop);
				}
<xqs>{quotecontinuefail} |
<xqs>{other} |
<xqs><<EOF>>	{
					/*
					 * Failed to see a quote continuation.  Throw back
					 * everything after the end quote, and handle the string
					 * according to the state we were in previously.
					 */
					yyless(0);
					BEGIN(INITIAL);

					switch (yyextra->state_before_str_stop)
					{
						case xb:
							yylval->str = litbufdup(yyscanner);
							return BCONST;
						case xh:
							yylval->str = litbufdup(yyscanner);
							return XCONST;
						case xq:
						case xe:
							/*
							 * Check that the data remains valid, if it might
							 * have been made invalid by unescaping any chars.
							 */
							if (yyextra->saw_non_ascii)
								pg_verifymbstr(yyextra->literalbuf,
											   yyextra->literallen,
											   false);
							yylval->str = litbufdup(yyscanner);
							return SCONST;
						case xus:
							yylval->str = litbufdup(yyscanner);
							return USCONST;
						default:
							yyerror("unhandled previous state in xqs");
					}
				}

<xq,xe,xus>{xqdouble} {
					addlitchar('\'', yyscanner);
				}
<xq,xus>{xqinside}  {
					addlit(yytext, yyleng, yyscanner);
				}
<xe>{xeinside}  {
					addlit(yytext, yyleng, yyscanner);
				}
<xe>{xeunicode} {
					pg_wchar	c = strtoul(yytext + 2, NULL, 16);

					/*
					 * For consistency with other productions, issue any
					 * escape warning with cursor pointing to start of string.
					 * We might want to change that, someday.
					 */
					check_escape_warning(yyscanner);

					/* Remember start of overall string token ... */
					PUSH_YYLLOC();
					/* ... and set the error cursor to point at this esc seq */
					SET_YYLLOC();

					if (is_utf16_surrogate_first(c))
					{
						yyextra->utf16_first_part = c;
						BEGIN(xeu);
					}
					else if (is_utf16_surrogate_second(c))
						yyerror("invalid Unicode surrogate pair");
					else
						addunicode(c, yyscanner);

					/* Restore yylloc to be start of string token */
					POP_YYLLOC();
				}
<xeu>{xeunicode} {
					pg_wchar	c = strtoul(yytext + 2, NULL, 16);

					/* Remember start of overall string token ... */
					PUSH_YYLLOC();
					/* ... and set the error cursor to point at this esc seq */
					SET_YYLLOC();

					if (!is_utf16_surrogate_second(c))
						yyerror("invalid Unicode surrogate pair");

					c = surrogate_pair_to_codepoint(yyextra->utf16_first_part, c);

					addunicode(c, yyscanner);

					/* Restore yylloc to be start of string token */
					POP_YYLLOC();

					BEGIN(xe);
				}
<xeu>. |
<xeu>\n |
<xeu><<EOF>>	{
					/* Set the error cursor to point at missing esc seq */
					SET_YYLLOC();
					yyerror("invalid Unicode surrogate pair");
				}
<xe,xeu>{xeunicodefail}	{
					/* Set the error cursor to point at malformed esc seq */
					SET_YYLLOC();
					ereport(ERROR,
							(errcode(ERRCODE_INVALID_ESCAPE_SEQUENCE),
							 errmsg("invalid Unicode escape"),
							 errhint("Unicode escapes must be \\uXXXX or \\UXXXXXXXX."),
							 lexer_errposition()));
				}
<xe>{xeescape}  {
					if (yytext[1] == '\'')
					{
						if (yyextra->backslash_quote == BACKSLASH_QUOTE_OFF ||
							(yyextra->backslash_quote == BACKSLASH_QUOTE_SAFE_ENCODING &&
							 PG_ENCODING_IS_CLIENT_ONLY(pg_get_client_encoding())))
							ereport(ERROR,
									(errcode(ERRCODE_NONSTANDARD_USE_OF_ESCAPE_CHARACTER),
									 errmsg("unsafe use of \\' in a string literal"),
									 errhint("Use '' to write quotes in strings. \\' is insecure in client-only encodings."),
									 lexer_errposition()));
					}
					check_string_escape_warning(yytext[1], yyscanner);
					addlitchar(unescape_single_char(yytext[1], yyscanner),
							   yyscanner);
				}
<xe>{xeoctesc}  {
					unsigned char c = strtoul(yytext + 1, NULL, 8);

					check_escape_warning(yyscanner);
					addlitchar(c, yyscanner);
					if (c == '\0' || IS_HIGHBIT_SET(c))
						yyextra->saw_non_ascii = true;
				}
<xe>{xehexesc}  {
					unsigned char c = strtoul(yytext + 2, NULL, 16);

					check_escape_warning(yyscanner);
					addlitchar(c, yyscanner);
					if (c == '\0' || IS_HIGHBIT_SET(c))
						yyextra->saw_non_ascii = true;
				}
<xe>.			{
					/* This is only needed for \ just before EOF */
					addlitchar(yytext[0], yyscanner);
				}
<xq,xe,xus><<EOF>>		{ yyerror("unterminated quoted string"); }

{dolqdelim}		{
					SET_YYLLOC();
					yyextra->dolqstart = pstrdup(yytext);
					BEGIN(xdolq);
					startlit();
				}
{dolqfailed}	{
					SET_YYLLOC();
					/* throw back all but the initial "$" */
					yyless(1);
					/* and treat it as {other} */
					return yytext[0];
				}
<xdolq>{dolqdelim} {
					if (strcmp(yytext, yyextra->dolqstart) == 0)
					{
						pfree(yyextra->dolqstart);
						yyextra->dolqstart = NULL;
						BEGIN(INITIAL);
						yylval->str = litbufdup(yyscanner);
						return SCONST;
					}
					else
					{
						/*
						 * When we fail to match $...$ to dolqstart, transfer
						 * the $... part to the output, but put back the final
						 * $ for rescanning.  Consider $delim$...$junk$delim$
						 */
						addlit(yytext, yyleng - 1, yyscanner);
						yyless(yyleng - 1);
					}
				}
<xdolq>{dolqinside} {
					addlit(yytext, yyleng, yyscanner);
				}
<xdolq>{dolqfailed} {
					addlit(yytext, yyleng, yyscanner);
				}
<xdolq>.		{
					/* This is only needed for $ inside the quoted text */
					addlitchar(yytext[0], yyscanner);
				}
<xdolq><<EOF>>	{ yyerror("unterminated dollar-quoted string"); }

{xdstart}		{
					SET_YYLLOC();
					BEGIN(xd);
					startlit();
				}
{xuistart}		{
					SET_YYLLOC();
					BEGIN(xui);
					startlit();
				}
<xd>{xdstop}	{
					char	   *ident;

					BEGIN(INITIAL);
					if (yyextra->literallen == 0)
						yyerror("zero-length delimited identifier");
					ident = litbufdup(yyscanner);
					if (yyextra->literallen >= NAMEDATALEN)
						truncate_identifier(ident, yyextra->literallen, true);
					yylval->str = ident;
					return IDENT;
				}
<xui>{dquote}	{
					BEGIN(INITIAL);
					if (yyextra->literallen == 0)
						yyerror("zero-length delimited identifier");
					/* can't truncate till after we de-escape the ident */
					yylval->str = litbufdup(yyscanner);
					return UIDENT;
				}
<xd,xui>{xddouble}	{
					addlitchar('"', yyscanner);
				}
<xd,xui>{xdinside}	{
					addlit(yytext, yyleng, yyscanner);
				}
<xd,xui><<EOF>>		{ yyerror("unterminated quoted identifier"); }

{xufailed}	{
					char	   *ident;

					SET_YYLLOC();
					/* throw back all but the initial u/U */
					yyless(1);
					/* and treat it as {identifier} */
					ident = downcase_truncate_identifier(yytext, yyleng, true);
					yylval->str = ident;
					return IDENT;
				}

{typecast}		{
					SET_YYLLOC();
					return TYPECAST;
				}

{dot_dot}		{
					SET_YYLLOC();
					return DOT_DOT;
				}

{colon_equals}	{
					SET_YYLLOC();
					return COLON_EQUALS;
				}

{equals_greater} {
					SET_YYLLOC();
					return EQUALS_GREATER;
				}

{less_equals}	{
					SET_YYLLOC();
					return LESS_EQUALS;
				}

{greater_equals} {
					SET_YYLLOC();
					return GREATER_EQUALS;
				}

{less_greater}	{
					/* We accept both "<>" and "!=" as meaning NOT_EQUALS */
					SET_YYLLOC();
					return NOT_EQUALS;
				}

{not_equals}	{
					/* We accept both "<>" and "!=" as meaning NOT_EQUALS */
					SET_YYLLOC();
					return NOT_EQUALS;
				}

{self}			{
					SET_YYLLOC();
					return yytext[0];
				}

{operator}		{
					/*
					 * Check for embedded slash-star or dash-dash; those
					 * are comment starts, so operator must stop there.
					 * Note that slash-star or dash-dash at the first
					 * character will match a prior rule, not this one.
					 */
					int			nchars = yyleng;
					char	   *slashstar = strstr(yytext, "/*");
					char	   *dashdash = strstr(yytext, "--");

					if (slashstar && dashdash)
					{
						/* if both appear, take the first one */
						if (slashstar > dashdash)
							slashstar = dashdash;
					}
					else if (!slashstar)
						slashstar = dashdash;
					if (slashstar)
						nchars = slashstar - yytext;

					/*
					 * For SQL compatibility, '+' and '-' cannot be the
					 * last char of a multi-char operator unless the operator
					 * contains chars that are not in SQL operators.
					 * The idea is to lex '=-' as two operators, but not
					 * to forbid operator names like '?-' that could not be
					 * sequences of SQL operators.
					 */
					if (nchars > 1 &&
						(yytext[nchars - 1] == '+' ||
						 yytext[nchars - 1] == '-'))
					{
						int			ic;

						for (ic = nchars - 2; ic >= 0; ic--)
						{
							char c = yytext[ic];
							if (c == '~' || c == '!' || c == '@' ||
								c == '#' || c == '^' || c == '&' ||
								c == '|' || c == '`' || c == '?' ||
								c == '%')
								break;
						}
						if (ic < 0)
						{
							/*
							 * didn't find a qualifying character, so remove
							 * all trailing [+-]
							 */
							do {
								nchars--;
							} while (nchars > 1 &&
								 (yytext[nchars - 1] == '+' ||
								  yytext[nchars - 1] == '-'));
						}
					}

					SET_YYLLOC();

					if (nchars < yyleng)
					{
						/* Strip the unwanted chars from the token */
						yyless(nchars);
						/*
						 * If what we have left is only one char, and it's
						 * one of the characters matching "self", then
						 * return it as a character token the same way
						 * that the "self" rule would have.
						 */
						if (nchars == 1 &&
							strchr(",()[].;:+-*/%^<>=", yytext[0]))
							return yytext[0];
						/*
						 * Likewise, if what we have left is two chars, and
						 * those match the tokens ">=", "<=", "=>", "<>" or
						 * "!=", then we must return the appropriate token
						 * rather than the generic Op.
						 */
						if (nchars == 2)
						{
							if (yytext[0] == '=' && yytext[1] == '>')
								return EQUALS_GREATER;
							if (yytext[0] == '>' && yytext[1] == '=')
								return GREATER_EQUALS;
							if (yytext[0] == '<' && yytext[1] == '=')
								return LESS_EQUALS;
							if (yytext[0] == '<' && yytext[1] == '>')
								return NOT_EQUALS;
							if (yytext[0] == '!' && yytext[1] == '=')
								return NOT_EQUALS;
						}
					}

					/*
					 * Complain if operator is too long.  Unlike the case
					 * for identifiers, we make this an error not a notice-
					 * and-truncate, because the odds are we are looking at
					 * a syntactic mistake anyway.
					 */
					if (nchars >= NAMEDATALEN)
						yyerror("operator too long");

					yylval->str = pstrdup(yytext);
					return Op;
				}

{param}			{
					SET_YYLLOC();
					yylval->ival = atol(yytext + 1);
					return PARAM;
				}

{integer}		{
					SET_YYLLOC();
					return process_integer_literal(yytext, yylval);
				}
{decimal}		{
					SET_YYLLOC();
					yylval->str = pstrdup(yytext);
					return FCONST;
				}
{decimalfail}	{
					/* throw back the .., and treat as integer */
					yyless(yyleng - 2);
					SET_YYLLOC();
					return process_integer_literal(yytext, yylval);
				}
{real}			{
					SET_YYLLOC();
					yylval->str = pstrdup(yytext);
					return FCONST;
				}
{realfail1}		{
					/*
					 * throw back the [Ee], and figure out whether what
					 * remains is an {integer} or {decimal}.
					 */
					yyless(yyleng - 1);
					SET_YYLLOC();
					return process_integer_literal(yytext, yylval);
				}
{realfail2}		{
					/* throw back the [Ee][+-], and proceed as above */
					yyless(yyleng - 2);
					SET_YYLLOC();
					return process_integer_literal(yytext, yylval);
				}


{identifier}	{
					int			kwnum;
					char	   *ident;

					SET_YYLLOC();

					/* Is it a keyword? */
					kwnum = ScanKeywordLookup(yytext,
											  yyextra->keywordlist);
					if (kwnum >= 0)
					{
						yylval->keyword = GetScanKeyword(kwnum,
														 yyextra->keywordlist);
						return yyextra->keyword_tokens[kwnum];
					}

					/*
					 * No.  Convert the identifier to lower case, and truncate
					 * if necessary.
					 */
					ident = downcase_truncate_identifier(yytext, yyleng, true);
					yylval->str = ident;
					return IDENT;
				}

{other}			{
					SET_YYLLOC();
					return yytext[0];
				}

<<EOF>>			{
					SET_YYLLOC();
					yyterminate();
				}

%%

/* LCOV_EXCL_STOP */

/*
 * Arrange access to yyextra for subroutines of the main yylex() function.
 * We expect each subroutine to have a yyscanner parameter.  Rather than
 * use the yyget_xxx functions, which might or might not get inlined by the
 * compiler, we cheat just a bit and cast yyscanner to the right type.
 */
#undef yyextra
#define yyextra  (((struct yyguts_t *) yyscanner)->yyextra_r)

/* Likewise for a couple of other things we need. */
#undef yylloc
#define yylloc	(((struct yyguts_t *) yyscanner)->yylloc_r)
#undef yyleng
#define yyleng	(((struct yyguts_t *) yyscanner)->yyleng_r)


/*
 * scanner_errposition
 *		Report a lexer or grammar error cursor position, if possible.
 *
 * This is expected to be used within an ereport() call, or via an error
 * callback such as setup_scanner_errposition_callback().  The return value
 * is a dummy (always 0, in fact).
 *
 * Note that this can only be used for messages emitted during raw parsing
 * (essentially, scan.l, parser.c, and gram.y), since it requires the
 * yyscanner struct to still be available.
 */
int
scanner_errposition(int location, core_yyscan_t yyscanner)
{
	int			pos;

	if (location < 0)
		return 0;				/* no-op if location is unknown */

	/* Convert byte offset to character number */
	pos = pg_mbstrlen_with_len(yyextra->scanbuf, location) + 1;
	/* And pass it to the ereport mechanism */
	return errposition(pos);
}

/*
 * Error context callback for inserting scanner error location.
 *
 * Note that this will be called for *any* error occurring while the
 * callback is installed.  We avoid inserting an irrelevant error location
 * if the error is a query cancel --- are there any other important cases?
 */
static void
scb_error_callback(void *arg)
{
	ScannerCallbackState *scbstate = (ScannerCallbackState *) arg;

	if (geterrcode() != ERRCODE_QUERY_CANCELED)
		(void) scanner_errposition(scbstate->location, scbstate->yyscanner);
}

/*
 * setup_scanner_errposition_callback
 *		Arrange for non-scanner errors to report an error position
 *
 * Sometimes the scanner calls functions that aren't part of the scanner
 * subsystem and can't reasonably be passed the yyscanner pointer; yet
 * we would like any errors thrown in those functions to be tagged with an
 * error location.  Use this function to set up an error context stack
 * entry that will accomplish that.  Usage pattern:
 *
 *		declare a local variable "ScannerCallbackState scbstate"
 *		...
 *		setup_scanner_errposition_callback(&scbstate, yyscanner, location);
 *		call function that might throw error;
 *		cancel_scanner_errposition_callback(&scbstate);
 */
void setup_scanner_errposition_callback(ScannerCallbackState *scbstate, core_yyscan_t yyscanner, int location) {
	/* Setup error traceback support for ereport() */
	scbstate->yyscanner = yyscanner;
	scbstate->location = location;
	scbstate->errcallback.callback = scb_error_callback;
	scbstate->errcallback.arg = (void *) scbstate;
	scbstate->errcallback.previous = error_context_stack;
	error_context_stack = &scbstate->errcallback;
}

/*
 * Cancel a previously-set-up errposition callback.
 */
void cancel_scanner_errposition_callback(ScannerCallbackState *scbstate) {
	/* Pop the error context stack */
	error_context_stack = scbstate->errcallback.previous;
}

/*
 * scanner_yyerror
 *		Report a lexer or grammar error.
 *
 * The message's cursor position is whatever YYLLOC was last set to,
 * ie, the start of the current token if called within yylex(), or the
 * most recently lexed token if called from the grammar.
 * This is OK for syntax error messages from the Bison parser, because Bison
 * parsers report error as soon as the first unparsable token is reached.
 * Beware of using yyerror for other purposes, as the cursor position might
 * be misleading!
 */
void scanner_yyerror(const char *message, core_yyscan_t yyscanner) {
	const char *loc = yyextra->scanbuf + *yylloc;

	if (*loc == YY_END_OF_BUFFER_CHAR)
	{
		ereport(ERROR,
				(errcode(ERRCODE_SYNTAX_ERROR),
		/* translator: %s is typically the translation of "syntax error" */
				 errmsg("%s at end of input", _(message)),
				 lexer_errposition()));
	}
	else
	{
		ereport(ERROR,
				(errcode(ERRCODE_SYNTAX_ERROR),
		/* translator: first %s is typically the translation of "syntax error" */
				 errmsg("%s at or near \"%s\"", _(message), loc),
				 lexer_errposition()));
	}
}

// Called before any actual parsing is done 初始化扫描器,在实际解析完成前调用，参数中要输入关键字信息，规则中{identifier} 需要，
core_yyscan_t scanner_init(const char *str, core_yy_extra_type *yyext, const ScanKeywordList *keywordlist, const uint16 *keyword_tokens) {
	Size		slen = strlen(str);
	yyscan_t	scanner;

	if (yylex_init(&scanner) != 0)
		elog(ERROR, "yylex_init() failed: %m");

	core_yyset_extra(yyext, scanner);

	yyext->keywordlist = keywordlist;
	yyext->keyword_tokens = keyword_tokens;

	yyext->backslash_quote = backslash_quote;
	yyext->escape_string_warning = escape_string_warning;
	yyext->standard_conforming_strings = standard_conforming_strings;

	/*
	 * Make a scan buffer with special termination needed by flex.
	 */
	yyext->scanbuf = (char *) palloc(slen + 2);
	yyext->scanbuflen = slen;
	memcpy(yyext->scanbuf, str, slen);
	yyext->scanbuf[slen] = yyext->scanbuf[slen + 1] = YY_END_OF_BUFFER_CHAR;
	yy_scan_buffer(yyext->scanbuf, slen + 2, scanner);

	/* initialize literal buffer to a reasonable but expansible size */
	yyext->literalalloc = 1024;
	yyext->literalbuf = (char *) palloc(yyext->literalalloc);
	yyext->literallen = 0;

	return scanner;
}

/*
 * Called after parsing is done to clean up after scanner_init()
 */
void scanner_finish(core_yyscan_t yyscanner) {
	/*
	 * We don't bother to call yylex_destroy(), because all it would do is
	 * pfree a small amount of control storage.  It's cheaper to leak the
	 * storage until the parsing context is destroyed.  The amount of space
	 * involved is usually negligible compared to the output parse tree
	 * anyway.
	 *
	 * We do bother to pfree the scanbuf and literal buffer, but only if they
	 * represent a nontrivial amount of space.  The 8K cutoff is arbitrary.
	 */
	if (yyextra->scanbuflen >= 8192)
		pfree(yyextra->scanbuf);
	if (yyextra->literalalloc >= 8192)
		pfree(yyextra->literalbuf);
}


static void addlit(char *ytext, int yleng, core_yyscan_t yyscanner) {
	/* enlarge buffer if needed */
	if ((yyextra->literallen + yleng) >= yyextra->literalalloc)
	{
		do
		{
			yyextra->literalalloc *= 2;
		} while ((yyextra->literallen + yleng) >= yyextra->literalalloc);
		yyextra->literalbuf = (char *) repalloc(yyextra->literalbuf,
												yyextra->literalalloc);
	}
	/* append new data */
	memcpy(yyextra->literalbuf + yyextra->literallen, ytext, yleng);
	yyextra->literallen += yleng;
}


static void addlitchar(unsigned char ychar, core_yyscan_t yyscanner) {
	/* enlarge buffer if needed */
	if ((yyextra->literallen + 1) >= yyextra->literalalloc) {
		yyextra->literalalloc *= 2;
		yyextra->literalbuf = (char *) repalloc(yyextra->literalbuf,
												yyextra->literalalloc);
	}
	/* append new data */
	yyextra->literalbuf[yyextra->literallen] = ychar;
	yyextra->literallen += 1;
}


/*
 * Create a palloc'd copy of literalbuf, adding a trailing null.
 */
static char * litbufdup(core_yyscan_t yyscanner) {
	int			llen = yyextra->literallen;
	char	   *new;

	new = palloc(llen + 1);
	memcpy(new, yyextra->literalbuf, llen);
	new[llen] = '\0';
	return new;
}

/*
 * Process {integer}.  Note this will also do the right thing with {decimal},
 * ie digits and a decimal point.
 */
static int process_integer_literal(const char *token, YYSTYPE *lval) {
	int			val;
	char	   *endptr;

	errno = 0;
	val = strtoint(token, &endptr, 10);
	if (*endptr != '\0' || errno == ERANGE)
	{
		/* integer too large (or contains decimal pt), treat it as a float */
		lval->str = pstrdup(token);
		return FCONST;
	}
	lval->ival = val;
	return ICONST;
}

static void addunicode(pg_wchar c, core_yyscan_t yyscanner) {
	ScannerCallbackState scbstate;
	char		buf[MAX_UNICODE_EQUIVALENT_STRING + 1];

	if (!is_valid_unicode_codepoint(c))
		yyerror("invalid Unicode escape value");

	/*
	 * We expect that pg_unicode_to_server() will complain about any
	 * unconvertible code point, so we don't have to set saw_non_ascii.
	 */
	setup_scanner_errposition_callback(&scbstate, yyscanner, *(yylloc));
	pg_unicode_to_server(c, (unsigned char *) buf);
	cancel_scanner_errposition_callback(&scbstate);
	addlit(buf, strlen(buf), yyscanner);
}

static unsigned char unescape_single_char(unsigned char c, core_yyscan_t yyscanner) {
	switch (c) {
		case 'b':
			return '\b';
		case 'f':
			return '\f';
		case 'n':
			return '\n';
		case 'r':
			return '\r';
		case 't':
			return '\t';
		default:
			/* check for backslash followed by non-7-bit-ASCII */
			if (c == '\0' || IS_HIGHBIT_SET(c))
				yyextra->saw_non_ascii = true;

			return c;
	}
}

static void check_string_escape_warning(unsigned char ychar, core_yyscan_t yyscanner) {
	if (ychar == '\'') {
		if (yyextra->warn_on_first_escape && yyextra->escape_string_warning)
			ereport(WARNING,
					(errcode(ERRCODE_NONSTANDARD_USE_OF_ESCAPE_CHARACTER),
					 errmsg("nonstandard use of \\' in a string literal"),
					 errhint("Use '' to write quotes in strings, or use the escape string syntax (E'...')."),
					 lexer_errposition()));
		yyextra->warn_on_first_escape = false;	/* warn only once per string */
	}
	else if (ychar == '\\')
	{
		if (yyextra->warn_on_first_escape && yyextra->escape_string_warning)
			ereport(WARNING,
					(errcode(ERRCODE_NONSTANDARD_USE_OF_ESCAPE_CHARACTER),
					 errmsg("nonstandard use of \\\\ in a string literal"),
					 errhint("Use the escape string syntax for backslashes, e.g., E'\\\\'."),
					 lexer_errposition()));
		yyextra->warn_on_first_escape = false;	/* warn only once per string */
	}
	else
		check_escape_warning(yyscanner);
}

static void check_escape_warning(core_yyscan_t yyscanner) {
	if (yyextra->warn_on_first_escape && yyextra->escape_string_warning)
		ereport(WARNING,
				(errcode(ERRCODE_NONSTANDARD_USE_OF_ESCAPE_CHARACTER),
				 errmsg("nonstandard use of escape in a string literal"),
		errhint("Use the escape string syntax for escapes, e.g., E'\\r\\n'."),
				 lexer_errposition()));
	yyextra->warn_on_first_escape = false;		/* warn only once per string */
}

/*
 * Interface functions to make flex use palloc() instead of malloc().
 * It'd be better to make these static, but flex insists otherwise.
 */
void * core_yyalloc(yy_size_t bytes, core_yyscan_t yyscanner) {
	return palloc(bytes);
}

void *
core_yyrealloc(void *ptr, yy_size_t bytes, core_yyscan_t yyscanner)
{
	if (ptr)
		return repalloc(ptr, bytes);
	else
		return palloc(bytes);
}

void
core_yyfree(void *ptr, core_yyscan_t yyscanner)
{
	if (ptr)
		pfree(ptr);
}

```


### 关键代码分析

```c++
ident_start		[A-Za-z\200-\377_]
ident_cont		[A-Za-z\200-\377_0-9\$]

identifier		{ident_start}{ident_cont}*

%%

// ... 
{identifier}	{
					int			kwnum;
					char	   *ident;

					SET_YYLLOC();

					/* Is it a keyword? */
					kwnum = ScanKeywordLookup(yytext,
											  yyextra->keywordlist);
					if (kwnum >= 0)
					{
						yylval->keyword = GetScanKeyword(kwnum,
														 yyextra->keywordlist);
						return yyextra->keyword_tokens[kwnum];
					}

					/*
					 * No.  Convert the identifier to lower case, and truncate
					 * if necessary.
					 */
					ident = downcase_truncate_identifier(yytext, yyleng, true);
					yylval->str = ident;
					return IDENT;
				}
%%

```

`scan.l`最重要的是中间的规则段，也就是通过正则表达式识别出标识符或者数字，符号等，然后后面定义识别出来后需要做的事情。其中很关键的是识别出`identifier`。整个词法分析阶段，核心过程就是通过正则表达式，定义若干规则，进行相关的处理工作。

`scan.l`编译后生成对应的`scan.c`，其核心函数如下：
```c++
// 通过这个函数处理前面规则段定义
extern int core_yylex(YYSTYPE * yylval_param,YYLTYPE * yylloc_param ,yyscan_t yyscanner);

#define YY_DECL int core_yylex(YYSTYPE * yylval_param, YYLTYPE * yylloc_param , yyscan_t yyscanner)


// The main scanner function which does all the work.
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

		/* yy_bp points to the position in yy_ch_buf of the start of
		 * the current run.
		 */
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

					/* Is it a keyword?  查看是否是关键字，关键字的定义在kwlist.h中定义，由src/tools/gen_keywordlist.pl生成对于的文件*/ 
					kwnum = ScanKeywordLookup(yytext, yyextra->keywordlist);   //see if a given word is a keyword
					if (kwnum >= 0) {
						yylval->keyword = GetScanKeyword(kwnum, yyextra->keywordlist);
						return yyextra->keyword_tokens[kwnum];
					}

					/*
					 * No.  Convert the identifier to lower case, and truncate
					 * if necessary.
					 */
					ident = downcase_truncate_identifier(yytext, yyleng, true);
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


```



---

参考
- [Flex介绍](https://zhuanlan.zhihu.com/p/89473441)
- [Lexical Analysis With Flex, for Flex 2.6.3](http://www.cs.virginia.edu/~cr4bd/flex-manual/index.html#SEC_Contents)
- [PostgreSQL 源码解读（168）- 查询#88(PG中的词法定义：scanner.l)#1](http://blog.itpub.net/6906/viewspace-2641424/)
- [PostgreSQL 源码解读（169）- 查询#89(PG中的词法定义：scanner.l)#2](http://blog.itpub.net/6906/viewspace-2641503/)
- [PostgreSQL 源码解读（170）- 查询#90(PG中的词法定义：scanner.l)#3](http://blog.itpub.net/6906/viewspace-2641703/)
- [PostgreSQL 源码解读（171）- 查询#91(PG中的词法定义：scanner.l)#4](http://blog.itpub.net/6906/viewspace-2641788/)
- [PostgreSQL源码学习：词法和语法分析](https://pgfans.cn/a?id=599)