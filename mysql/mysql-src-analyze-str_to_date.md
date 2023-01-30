### MySQL源码分析 —— str_to_date函数
我们进行源码分析，分析一下Mysql中内置函数的调用流程以及`str_to_date`函数的实现。

#### 主流程
```c++
handle_connection
--> do_command // 处理每个请求
    --> dispatch_command
        --> mysql_parse // 解析
            --> lex_start
            --> parse_sql
                --> MYSQLparse
                    --> yyparse  // bison， 语法解析
        --> mysql_rewrite_query
        --> mysql_execute_command
            --> select_precheck
            --> execute_sqlcom_select
                --> handle_query
                    --> select->prepare
                        --> setup_fields  //  Check that all given fields exists and fill struct with current data
                            --> item->fix_fields
                                --> Item_func::fix_fields
                                    --> Item_func_str_to_date::fix_length_and_dec()  // 确定返回类型
                                        --> Item_func_str_to_date::fix_from_format
                    --> select->optimize
                        --> JOIN::optimize()
                    --> select->join->exec();
                        --> query_result->send_data
                            --> thd->send_result_set_row
                                --> item->send
                                    --> Item_temporal_hybrid_func::get_date
                                        --> Item_func_str_to_date::val_datetime   // 调用str_to_date类
                                            --> extract_date_time
```


#### 语法解析层面
我们先看一下在语法解析层面，mysql是如何表示`SELECT STR_TO_DATE('2022-05-26 11:30:00','%Y-%m-%d');`的。最重要的部分是`PTI_function_call_generic_ident_sys`，
```c++
/*
  Regular function calls.
  The function name is *not* a token, and therefore is guaranteed to not
  introduce side effects to the language in general.
  MAINTAINER:
  All the new functions implemented for new features should fit into
  this category. The place to implement the function itself is
  in sql/item_create.cc
*/
function_call_generic:
          IDENT_sys '(' opt_udf_expr_list ')'
          {
            $$= NEW_PTN PTI_function_call_generic_ident_sys(@1, $1, $3);
          }
        | ident '.' ident '(' opt_expr_list ')'
          {
            $$= NEW_PTN PTI_function_call_generic_2d(@$, $1, $3, $5);
          }
        ;
```
通过`PTI_function_call_generic_ident_sys`来表示这个`str_to_date`的函数调用。下面我们看一下整体的情况。
```c++
/*
  Indentation of grammar rules:

rule: <-- starts at col 1
          rule1a rule1b rule1c <-- starts at col 11
          { <-- starts at col 11
            code <-- starts at col 13, indentation is 2 spaces
          }
        | rule2a rule2b
          {
            code
          }
        ; <-- on a line by itself, starts at col 9

  Also, please do not use any <TAB>, but spaces.
  Having a uniform indentation in this file helps
  code reviews, patches, merges, and make maintenance easier.
  Tip: grep [[:cntrl:]] sql_yacc.yy
  Thanks.
*/

query:
          END_OF_INPUT
          {
            THD *thd= YYTHD;
            if (!thd->bootstrap &&
                !thd->m_parser_state->has_comment())
            {
              my_message(ER_EMPTY_QUERY, ER(ER_EMPTY_QUERY), MYF(0));
              MYSQL_YYABORT;
            }
            thd->lex->sql_command= SQLCOM_EMPTY_QUERY;
            YYLIP->found_semicolon= NULL;
          }
        | verb_clause
          {
            Lex_input_stream *lip = YYLIP;

            if (YYTHD->get_protocol()->has_client_capability(CLIENT_MULTI_QUERIES) &&
                lip->multi_statements &&
                ! lip->eof())
            {
              /*
                We found a well formed query, and multi queries are allowed:
                - force the parser to stop after the ';'
                - mark the start of the next query for the next invocation
                  of the parser.
              */
              lip->next_state= MY_LEX_END;
              lip->found_semicolon= lip->get_ptr();
            }
            else
            {
              /* Single query, terminated. */
              lip->found_semicolon= NULL;
            }
          }
          ';'
          opt_end_of_input
        | verb_clause END_OF_INPUT
          {
            /* Single query, not terminated. */
            YYLIP->found_semicolon= NULL;
          }
        ;

opt_end_of_input:
          /* empty */
        | END_OF_INPUT
        ;
verb_clause:
          statement
        | begin
        ;
/* Verb clauses, except begin */
statement:
        | select                { CONTEXTUALIZE($1); }  // 调用
select:
          select_init
          {
            $$= NEW_PTN PT_select($1, SQLCOM_SELECT);
          }
        ;

/* Need first branch for subselects. */
select_init:
          SELECT_SYM select_part2 opt_union_clause
          {
            $$= NEW_PTN PT_select_init2($1, $2, $3);
          }
        | '(' select_paren ')' union_opt
          {
            $$= NEW_PTN PT_select_init_parenthesis($2, $4);
          }
        ;
select_part2:
          select_options_and_item_list
select_options_and_item_list:
          {
            /*
              TODO: remove this semantic action (currently this removal
              adds shift/reduce conflict)
            */
          }
          select_options select_item_list
          {
            $$= NEW_PTN PT_select_options_and_item_list($2, $3);
          }
        ;
select_item_list:
          select_item_list ',' select_item
          {
            if ($1 == NULL || $1->push_back($3))
              MYSQL_YYABORT;
            $$= $1;
          }
        | select_item
          {
            $$= NEW_PTN PT_select_item_list;
            if ($$ == NULL || $$->push_back($1))
              MYSQL_YYABORT;
          }
select_item:
          table_wild { $$= $1; }
        | expr select_alias
          {
            $$= NEW_PTN PTI_expr_with_alias(@$, $1, @1.cpp, $2);
          }
        ;
/* all possible expressions */
expr:
        | bool_pri
        ;
bool_pri:
        | predicate
        ;
predicate:
        | bit_expr
        ;
bit_expr:
        | simple_expr
        ;
simple_expr:
          simple_ident
        | function_call_keyword
        | function_call_nonkeyword
        | function_call_generic
/*
  Regular function calls.
  The function name is *not* a token, and therefore is guaranteed to not
  introduce side effects to the language in general.
  MAINTAINER:
  All the new functions implemented for new features should fit into
  this category. The place to implement the function itself is
  in sql/item_create.cc
*/
function_call_generic:
          IDENT_sys '(' opt_udf_expr_list ')'
          {
            $$= NEW_PTN PTI_function_call_generic_ident_sys(@1, $1, $3);
          }
        | ident '.' ident '(' opt_expr_list ')'
          {
            $$= NEW_PTN PTI_function_call_generic_2d(@$, $1, $3, $5);
          }
        ;
udf_expr_list:
          udf_expr
          {
            $$= NEW_PTN PT_item_list;
            if ($$ == NULL || $$->push_back($1))
              MYSQL_YYABORT;
          }
udf_expr:
          expr select_alias
          {
            $$= NEW_PTN PTI_udf_expr(@$, $1, $2, @1.cpp);
          }
        ;
select_alias:
          /* empty */ { $$=null_lex_str;}
        | AS ident { $$=$2; }
        | AS TEXT_STRING_sys { $$=$2; }
        | ident { $$=$1; }
        | TEXT_STRING_sys { $$=$1; }
        ;        
literal:
          text_literal { $$= $1; }
text_literal:
          TEXT_STRING
          {
            $$= NEW_PTN PTI_text_literal_text_string(@$,
                YYTHD->m_parser_state->m_lip.text_string_is_7bit(), $1);
          }

IDENT_sys:
          IDENT { $$= $1; }
        | IDENT_QUOTED
          {
            THD *thd= YYTHD;

            if (thd->charset_is_system_charset)
            {
              const CHARSET_INFO *cs= system_charset_info;
              int dummy_error;
              size_t wlen= cs->cset->well_formed_len(cs, $1.str,
                                                     $1.str+$1.length,
                                                     $1.length, &dummy_error);
              if (wlen < $1.length)
              {
                ErrConvString err($1.str, $1.length, &my_charset_bin);
                my_error(ER_INVALID_CHARACTER_STRING, MYF(0), cs->csname, err.ptr());
                MYSQL_YYABORT;
              }
              $$= $1;
            }
            else
            {
              if (thd->convert_string(&$$, system_charset_info,
                                  $1.str, $1.length, thd->charset()))
                MYSQL_YYABORT;
            }
          }
        ;

```

解析层面的表示，下面这个类比较重要：
```c++
class PTI_function_call_generic_ident_sys : public Parse_tree_item
{
  typedef Parse_tree_item super;

  LEX_STRING ident;   // 函数名
  PT_item_list *opt_udf_expr_list;   // 函数参数列表，参数表达式

  udf_func *udf;

public:
  PTI_function_call_generic_ident_sys(const POS &pos,
                                      const LEX_STRING &ident_arg,
                                      PT_item_list *opt_udf_expr_list_arg)
  : super(pos), ident(ident_arg),
    opt_udf_expr_list(opt_udf_expr_list_arg)
  {}

  virtual bool itemize(Parse_context *pc, Item **res)
  {
    if (super::itemize(pc, res))
      return true;

    THD *thd= pc->thd;
#ifdef HAVE_DLOPEN
    udf= 0;
    if (using_udf_functions &&
        (udf= find_udf(ident.str, ident.length)) &&
        udf->type == UDFTYPE_AGGREGATE)
    {
      pc->select->in_sum_expr++;
    }
#endif

    if (sp_check_name(&ident))
      return true;

    /*
      Implementation note:
      names are resolved with the following order:
      - MySQL native functions,
      - User Defined Functions,
      - Stored Functions (assuming the current <use> database)

      This will be revised with WL#2128 (SQL PATH)
    */
    Create_func *builder= find_native_function_builder(thd, ident);
    if (builder)
      *res= builder->create_func(thd, ident, opt_udf_expr_list);
    else
    {
#ifdef HAVE_DLOPEN
      if (udf)
      {
        if (udf->type == UDFTYPE_AGGREGATE)
        {
          pc->select->in_sum_expr--;
        }

        *res= Create_udf_func::s_singleton.create(thd, udf, opt_udf_expr_list);
      }
      else
#endif
      {
        builder= find_qualified_function_builder(thd);
        assert(builder);
        *res= builder->create_func(thd, ident, opt_udf_expr_list);
      }
    }
    return *res == NULL || (*res)->itemize(pc, res);
  }
};
```
最终会调用`builder->create_func`构建`Item_func_str_to_date`类，调用栈如下：
```c++
Item_func_str_to_date::Item_func_str_to_date(Item_func_str_to_date * const this, const POS & pos, Item * a, Item * b) (mysql-server\sql\item_timefunc.h:1720)
Create_func_str_to_date::create(Create_func_str_to_date * const this, THD * thd, Item * arg1, Item * arg2) (mysql-server\sql\item_create.cc:7193)
Create_func_arg2::create_func(Create_func_arg2 * const this, THD * thd, LEX_STRING name, PT_item_list * item_list) (mysql-server\sql\item_create.cc:4360)
PTI_function_call_generic_ident_sys::itemize(PTI_function_call_generic_ident_sys * const this, Parse_context * pc, Item ** res) (mysql-server\sql\parse_tree_items.h:332)
PTI_expr_with_alias::itemize(PTI_expr_with_alias * const this, Parse_context * pc, Item ** res) (mysql-server\sql\parse_tree_items.cc:186)
PT_item_list::contextualize(PT_item_list * const this, Parse_context * pc) (mysql-server\sql\parse_tree_helpers.h:77)
PT_select_item_list::contextualize(PT_select_item_list * const this, Parse_context * pc) (mysql-server\sql\parse_tree_nodes.h:191)
PT_select_options_and_item_list::contextualize(PT_select_options_and_item_list * const this, Parse_context * pc) (mysql-server\sql\parse_tree_nodes.h:2185)
PT_select_part2::contextualize(PT_select_part2 * const this, Parse_context * pc) (mysql-server\sql\parse_tree_nodes.h:2255)
PT_select_init2::contextualize(PT_select_init2 * const this, Parse_context * pc) (mysql-server\sql\parse_tree_nodes.h:2380)
PT_select::contextualize(PT_select * const this, Parse_context * pc) (mysql-server\sql\parse_tree_nodes.h:2417)
MYSQLparse(THD * YYTHD) (mysql-server\sql\sql_yacc.yy:1707)
parse_sql(THD * thd, Parser_state * parser_state, Object_creation_ctx * creation_ctx) (mysql-server\sql\sql_parse.cc:7146)
mysql_parse(THD * thd, Parser_state * parser_state) (mysql-server\sql\sql_parse.cc:5469)
dispatch_command(THD * thd, const COM_DATA * com_data, enum_server_command command) (mysql-server\sql\sql_parse.cc:1492)
do_command(THD * thd) (mysql-server\sql\sql_parse.cc:1031)
handle_connection(void * arg) (mysql-server\sql\conn_handler\connection_handler_per_thread.cc:313)
pfs_spawn_thread(void * arg) (mysql-server\storage\perfschema\pfs.cc:2197)
libpthread.so.0!start_thread (Unknown Source:0)
libc.so.6!clone (Unknown Source:0)
```
`Item_func_str_to_date`类定义如下：
```c++
// 函数str_to_date 类定义
class Item_func_str_to_date :public Item_temporal_hybrid_func
{
  timestamp_type cached_timestamp_type;
  bool const_item;
  void fix_from_format(const char *format, size_t length);  // 根据format字符串确定返回数据类型
protected:
  bool val_datetime(MYSQL_TIME *ltime, my_time_flags_t fuzzy_date);
public:
  Item_func_str_to_date(const POS &pos, Item *a, Item *b)
    :Item_temporal_hybrid_func(pos, a, b), const_item(false)
  {}
  const char *func_name() const { return "str_to_date"; }
  void fix_length_and_dec();  
};
```
创建函数的类如下（工厂模式）
```c++
class Create_func_str_to_date : public Create_func_arg2
{
public:
  virtual Item *create(THD *thd, Item *arg1, Item *arg2);

  static Create_func_str_to_date s_singleton;

protected:
  Create_func_str_to_date() {}
  virtual ~Create_func_str_to_date() {}
};

Item* Create_func_str_to_date::create(THD *thd, Item *arg1, Item *arg2)
{
  return new (thd->mem_root) Item_func_str_to_date(POS(), arg1, arg2);
}
```
#### 核心实现
创建对应的函数处理类后，我们看一下其关键类方法，在提前日期时间之前，会根据format参数判定返回的实际类型，比如可以是`date`类型或者是`datetime`类型。重点是分析成员函数`fix_from_format`。
```c++
void Item_func_str_to_date::fix_length_and_dec()
{
  maybe_null= 1;
  cached_field_type= MYSQL_TYPE_DATETIME;
  cached_timestamp_type= MYSQL_TIMESTAMP_DATETIME;
  fix_length_and_dec_and_charset_datetime(MAX_DATETIME_WIDTH,
                                          DATETIME_MAX_DECIMALS);
  sql_mode= current_thd->variables.sql_mode & (MODE_NO_ZERO_DATE |
                                               MODE_NO_ZERO_IN_DATE |
                                               MODE_INVALID_DATES);
  if ((const_item= args[1]->const_item()))
  {
    char format_buff[64];
    String format_str(format_buff, sizeof(format_buff), &my_charset_bin);
    String *format= args[1]->val_str(&format_str);
    if (!args[1]->null_value)
      fix_from_format(format->ptr(), format->length());
  }
}

/**
  Set type of datetime value (DATE/TIME/...) which will be produced
  according to format string.

  @param format   format string
  @param length   length of format string

  @note
    We don't process day format's characters('D', 'd', 'e') because day
    may be a member of all date/time types.

  @note
    Format specifiers supported by this function should be in sync with
    specifiers supported by extract_date_time() function.
*/
void Item_func_str_to_date::fix_from_format(const char *format, size_t length)
{
  const char *time_part_frms= "HISThiklrs";
  const char *date_part_frms= "MVUXYWabcjmvuxyw";
  bool date_part_used= 0, time_part_used= 0, frac_second_used= 0;
  const char *val= format;
  const char *end= format + length;

  for (; val != end; val++)
  {
    if (*val == '%' && val + 1 != end)
    {
      val++;
      if (*val == 'f')
        frac_second_used= time_part_used= 1;
      else if (!time_part_used && strchr(time_part_frms, *val))
	time_part_used= 1;
      else if (!date_part_used && strchr(date_part_frms, *val))
	date_part_used= 1;
      if (date_part_used && frac_second_used)
      {
        /*
          frac_second_used implies time_part_used, and thus we already
          have all types of date-time components and can end our search.
        */
        cached_timestamp_type= MYSQL_TIMESTAMP_DATETIME;
        cached_field_type= MYSQL_TYPE_DATETIME; 
        fix_length_and_dec_and_charset_datetime(MAX_DATETIME_WIDTH,
                                                DATETIME_MAX_DECIMALS);
        return;
      }
    }
  }

  /* We don't have all three types of date-time components */
  if (frac_second_used) /* TIME with microseconds */
  {
    cached_timestamp_type= MYSQL_TIMESTAMP_TIME;
    cached_field_type= MYSQL_TYPE_TIME;
    fix_length_and_dec_and_charset_datetime(MAX_TIME_FULL_WIDTH,
                                            DATETIME_MAX_DECIMALS);
  }
  else if (time_part_used)
  {
    if (date_part_used) /* DATETIME, no microseconds */
    {
      cached_timestamp_type= MYSQL_TIMESTAMP_DATETIME;
      cached_field_type= MYSQL_TYPE_DATETIME; 
      fix_length_and_dec_and_charset_datetime(MAX_DATETIME_WIDTH, 0);
    }
    else /* TIME, no microseconds */
    {
      cached_timestamp_type= MYSQL_TIMESTAMP_TIME;
      cached_field_type= MYSQL_TYPE_TIME;
      fix_length_and_dec_and_charset_datetime(MAX_TIME_WIDTH, 0);
    }
  }
  else /* DATE */
  {
    cached_timestamp_type= MYSQL_TIMESTAMP_DATE;
    cached_field_type= MYSQL_TYPE_DATE; 
    fix_length_and_dec_and_charset_datetime(MAX_DATE_WIDTH, 0);
  }
}
```
函数调用栈如下：
```c++
Item_func_str_to_date::fix_from_format(Item_func_str_to_date * const this, const char * format, size_t length) (mysql-server\sql\item_timefunc.cc:3283)
Item_func_str_to_date::fix_length_and_dec(Item_func_str_to_date * const this) (mysql-server\sql\item_timefunc.cc:3355)
Item_func::fix_fields(Item_func * const this, THD * thd, Item ** ref) (mysql-server\sql\item_func.cc:253)
Item_str_func::fix_fields(Item_str_func * const this, THD * thd, Item ** ref) (mysql-server\sql\item_strfunc.cc:115)
setup_fields(THD * thd, Ref_ptr_array ref_pointer_array, List<Item> & fields, ulong want_privilege, List<Item> * sum_func_list, bool allow_sum_func, bool column_update) (mysql-server\sql\sql_base.cc:9138)
st_select_lex::prepare(st_select_lex * const this, THD * thd) (mysql-server\sql\sql_resolver.cc:197)
handle_query(THD * thd, LEX * lex, Query_result * result, ulonglong added_options, ulonglong removed_options) (mysql-server\sql\sql_select.cc:139)
execute_sqlcom_select(THD * thd, TABLE_LIST * all_tables) (mysql-server\sql\sql_parse.cc:5156)
mysql_execute_command(THD * thd, bool first_level) (mysql-server\sql\sql_parse.cc:2829)
mysql_parse(THD * thd, Parser_state * parser_state) (mysql-server\sql\sql_parse.cc:5589)
dispatch_command(THD * thd, const COM_DATA * com_data, enum_server_command command) (mysql-server\sql\sql_parse.cc:1492)
do_command(THD * thd) (mysql-server\sql\sql_parse.cc:1031)
handle_connection(void * arg) 
```


下面这个函数会根据输入的format参数提取日期时间等信息。
```c++
bool Item_temporal_hybrid_func::get_date(MYSQL_TIME *ltime,
                                         my_time_flags_t fuzzy_date)
{
  MYSQL_TIME tm;
  if (val_datetime(&tm, fuzzy_date))    // 根据format提取日期时间
  {
      assert(null_value == true);
      return true;
  }
  if (cached_field_type == MYSQL_TYPE_TIME ||
      tm.time_type == MYSQL_TIMESTAMP_TIME)
    time_to_datetime(current_thd, &tm, ltime);   
  else
    *ltime= tm;
  return false;
}
```
函数调用栈如下：
```c++
extract_date_time(Date_time_format * format, const char * val, size_t length, MYSQL_TIME * l_time, timestamp_type cached_timestamp_type, const char ** sub_pattern_end, const char * date_time_type) (mysql-server\sql\item_timefunc.cc:175)
Item_func_str_to_date::val_datetime(Item_func_str_to_date * const this, MYSQL_TIME * ltime, my_time_flags_t fuzzy_date) (mysql-server\sql\item_timefunc.cc:3385)
Item_temporal_hybrid_func::get_date(Item_temporal_hybrid_func * const this, MYSQL_TIME * ltime, my_time_flags_t fuzzy_date) (mysql-server\sql\item_timefunc.cc:906)
Item::send(Item * const this, Protocol * protocol, String * buffer) (mysql-server\sql\item.cc:7603)
THD::send_result_set_row(THD * const this, List<Item> * row_items) (mysql-server\sql\sql_class.cc:4747)
Query_result_send::send_data(Query_result_send * const this, List<Item> & items) (mysql-server\sql\sql_class.cc:2740)
JOIN::exec(JOIN * const this) (mysql-server\sql\sql_executor.cc:165)
handle_query(THD * thd, LEX * lex, Query_result * result, ulonglong added_options, ulonglong removed_options) (mysql-server\sql\sql_select.cc:191)
execute_sqlcom_select(THD * thd, TABLE_LIST * all_tables) (mysql-server\sql\sql_parse.cc:5156)
mysql_execute_command(THD * thd, bool first_level) (mysql-server\sql\sql_parse.cc:2829)
mysql_parse(THD * thd, Parser_state * parser_state) (mysql-server\sql\sql_parse.cc:5589)
dispatch_command(THD * thd, const COM_DATA * com_data, enum_server_command command) (mysql-server\sql\sql_parse.cc:1492)
do_command(THD * thd) (mysql-server\sql\sql_parse.cc:1031)
handle_connection(void * arg) (mysql-server\sql\conn_handler\connection_handler_per_thread.cc:313)
pfs_spawn_thread(void * arg) (mysql-server\storage\perfschema\pfs.cc:2197)
libpthread.so.0!start_thread (Unknown Source:0)
libc.so.6!clone (Unknown Source:0)
```
后面会根据类型调用`my_date_to_str`或者`my_datetime_to_str`将日期时间信息转为字符串然后返回字符串信息。
