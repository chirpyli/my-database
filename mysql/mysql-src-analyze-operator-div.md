### MySQL源码分析——除法操作符
因为之前的文章中没有分析过mysql的源码，所以这里先分析一下mysql处理每个客户端连接请求的过程。需要注意的是，Mysql服务端启动后，监听端口，当有新的客户端连接请求时，是分配一个线程处理这个请求，而在Postgres中，是fork一个进程进行处理。需要注意这一点不同。
#### mysql处理客户端请求
MySQL主流程如下：
```c++
main   // main.c  mysqld入口
--> mysqld_main
    --> mysqld_socket_acceptor->connection_event_loop() // 等待客户端连接请求
        --> poll  // 监听socket
    --> mgr->process_new_connection(channel_info);  // 处理每个新的连接请求
        --> add_connection
            // 创建一个线程，线程函数为handle_connection
            --> mysql_thread_create(key_thread_one_connection, &id, &connection_attrib, handle_connection, (void*) channel_info);
                --> pthread_create // 创建一个新的线程
```
处理客户端连接请求的调用栈如下：
```c++
libc.so.6!poll (Unknown Source:0)
Mysqld_socket_listener::listen_for_connection_event(Mysqld_socket_listener * const this) (mysql-server\sql\conn_handler\socket_connection.cc:859)
Connection_acceptor<Mysqld_socket_listener>::connection_event_loop(Connection_acceptor<Mysqld_socket_listener> * const this) (mysql-server\sql\conn_handler\connection_acceptor.h:73)
mysqld_main(int argc, char ** argv) (mysql-server\sql\mysqld.cc:5156)
main(int argc, char ** argv) (mysql-server\sql\main.cc:32)
```
mysql与postgres不同，mysql是每个客户端连接对应一个线程进行处理，而postgres是每个连接一个进程。
```c++
template <typename Listener> class Connection_acceptor
{
  Listener *m_listener;

public:
  Connection_acceptor(Listener *listener)
  : m_listener(listener)
  { }

  /**
    Connection acceptor loop to accept connections from clients.
  */
  void connection_event_loop()
  {
    Connection_handler_manager *mgr= Connection_handler_manager::get_instance();
    while (!abort_loop)
    {
      Channel_info *channel_info= m_listener->listen_for_connection_event();
      if (channel_info != NULL)
        mgr->process_new_connection(channel_info);  // 处理每个新的客户端连接请求
    }
  }

  // ...
};
```

#### 处理每个连接请求handle_connection
mysql接收到一个新的客户端请求后，创建一个新的线程进行处理，线程函数为`handle_connection`，我们进入这个线程中进行分析。
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
                    --> select->optimize
                        --> JOIN::optimize()
                    --> select->join->exec();
                        --> query_result->send_data
                            --> thd->send_result_set_row
                                --> item->send
                                    --> val_str
```

除法表达式的语法解析为：
```sql

bit_expr:
        | bit_expr '/' bit_expr %prec '/'       -- 除法 / 
          {
            $$= NEW_PTN Item_func_div(@$, $1,$3);
          }
        | bit_expr DIV_SYM bit_expr %prec DIV_SYM  -- 整型除法 div
          {
            $$= NEW_PTN Item_func_int_div(@$, $1,$3);
          }
        | simple_expr
        ;
simple_expr:
          literal
        | param_marker { $$= $1; }
literal:
          text_literal { $$= $1; }
        | NUM_literal
        | temporal_literal
        | NULL_SYM
          {
            Lex_input_stream *lip= YYLIP;
            /*
              For the digest computation, in this context only,
              NULL is considered a literal, hence reduced to '?'
              REDUCE:
                TOK_GENERIC_VALUE := NULL_SYM
            */
            lip->reduce_digest_token(TOK_GENERIC_VALUE, NULL_SYM);
            $$= NEW_PTN Item_null(@$);
            lip->next_state= MY_LEX_OPERATOR_OR_IDENT;
          }
NUM_literal:
          NUM
          {
            $$= NEW_PTN PTI_num_literal_num(@$, $1);
          }
```

```c++
class Item_func_div :public Item_num_op
{
public:
  uint prec_increment;
  Item_func_div(const POS &pos, Item *a,Item *b) :Item_num_op(pos, a,b) {}
  longlong int_op() { assert(0); return 0; }
  double real_op();
  my_decimal *decimal_op(my_decimal *);
  const char *func_name() const { return "/"; }
  void fix_length_and_dec();
  void result_precision();
};

/* Base class for operations like '+', '-', '*' */
class Item_num_op :public Item_func_numhybrid
{
 public:
  Item_num_op(Item *a,Item *b) :Item_func_numhybrid(a, b) {}
  Item_num_op(const POS &pos, Item *a,Item *b) :Item_func_numhybrid(pos, a, b)
  {}

  virtual void result_precision()= 0;

  virtual inline void print(String *str, enum_query_type query_type)
  {
    print_op(str, query_type);
  }

  void find_num_type();
  String *str_op(String *str) { assert(0); return 0; }
  bool date_op(MYSQL_TIME *ltime, my_time_flags_t fuzzydate)
  { assert(0); return 0; }
  bool time_op(MYSQL_TIME *ltime)
  { assert(0); return 0; }
};

class Item_func_numhybrid: public Item_func
{
protected:
  Item_result hybrid_type;
public:

  Item_func_numhybrid(const POS &pos, Item *a,Item *b)
    :Item_func(pos, a,b), hybrid_type(REAL_RESULT)
  { collation.set_numeric(); }
};
```
mysql中的除法函数为:
```c++
my_decimal *Item_func_div::decimal_op(my_decimal *decimal_value)
{
  my_decimal value1, *val1;
  my_decimal value2, *val2;
  int err;
  // 将数据类型转为decimal型
  val1= args[0]->val_decimal(&value1);
  if ((null_value= args[0]->null_value))
    return 0;
  // 将数据类型转为decimal型
  val2= args[1]->val_decimal(&value2);
  if ((null_value= args[1]->null_value))
    return 0;
  if ((err= check_decimal_overflow(my_decimal_div(E_DEC_FATAL_ERROR &
                                                  ~E_DEC_OVERFLOW &
                                                  ~E_DEC_DIV_ZERO,
                                                  decimal_value,
                                                  val1, val2,
                                                  prec_increment))) > 3)
  {
    if (err == E_DEC_DIV_ZERO)
      signal_divide_by_null();
    null_value= 1;
    return 0;
  }
  return decimal_value;
}
// 将值longlong型 Item_int::value转换为浮点数
my_decimal *Item_int::val_decimal(my_decimal *decimal_value)
{
  int2my_decimal(E_DEC_FATAL_ERROR, value, unsigned_flag, decimal_value);
  return decimal_value;
}
// int转换为decimal型
inline int int2my_decimal(uint mask, longlong i, my_bool unsigned_flag, my_decimal *d)
{
  return d->check_result(mask, (unsigned_flag ?
                                ulonglong2decimal((ulonglong)i, d) :
                                longlong2decimal(i, d)));
}
// 类型转换,longlong 转换为 decimal
int longlong2decimal(longlong from, decimal_t *to)
{
  if ((to->sign= from < 0))
    return ull2dec(-from, to);
  return ull2dec(from, to);
}

// mysql中int型数字的表示
class Item_int :public Item_num
{
  typedef Item_num super;
public:
  longlong value;
  Item_int(int32 i,uint length= MY_INT32_NUM_DECIMAL_DIGITS)
    :value((longlong) i)
    { max_length=length; fixed= 1; }
  Item_int(const POS &pos, int32 i,uint length= MY_INT32_NUM_DECIMAL_DIGITS)
    :super(pos), value((longlong) i)
  { max_length=length; fixed= 1; }

  Item_int(longlong i,uint length= MY_INT64_NUM_DECIMAL_DIGITS)
    :value(i)
    { max_length=length; fixed= 1; }
  Item_int(ulonglong i, uint length= MY_INT64_NUM_DECIMAL_DIGITS)
    :value((longlong)i)
    { max_length=length; fixed= 1; unsigned_flag= 1; }
  Item_int(Item_int *item_arg)
  {
    value= item_arg->value;
    item_name= item_arg->item_name;
    max_length= item_arg->max_length;
    fixed= 1;
  }

  Item_int(const Name_string &name_arg, longlong i, uint length) :value(i)
  {
    max_length= length;
    item_name= name_arg;
    fixed= 1;
  }
  Item_int(const POS &pos, const Name_string &name_arg, longlong i, uint length)
    :super(pos), value(i)
  {
    max_length= length;
    item_name= name_arg;
    fixed= 1;
  }

  Item_int(const char *str_arg, uint length)
  { init(str_arg, length); }
  Item_int(const POS &pos, const char *str_arg, uint length) : super(pos)
  { init(str_arg, length); }

private:
  void init(const char *str_arg, uint length);

protected:
  type_conversion_status save_in_field_inner(Field *field, bool no_conversions);

public:
  enum Type type() const { return INT_ITEM; }
  enum Item_result result_type () const { return INT_RESULT; }
  enum_field_types field_type() const { return MYSQL_TYPE_LONGLONG; }
  longlong val_int() { assert(fixed == 1); return value; }
  double val_real() { assert(fixed == 1); return (double) value; }
  my_decimal *val_decimal(my_decimal *);
  String *val_str(String*);
  bool get_date(MYSQL_TIME *ltime, my_time_flags_t fuzzydate)
  {
    return get_date_from_int(ltime, fuzzydate);
  }
  bool get_time(MYSQL_TIME *ltime)
  {
    return get_time_from_int(ltime);
  }
  bool basic_const_item() const { return 1; }
  Item *clone_item() { return new Item_int(this); }
  virtual void print(String *str, enum_query_type query_type);
  Item_num *neg() { value= -value; return this; }
  uint decimal_precision() const
  { return (uint)(max_length - MY_TEST(value < 0)); }
  bool eq(const Item *, bool binary_cmp) const;
  bool check_partition_func_processor(uchar *bool_arg) { return false;}
  bool check_gcol_func_processor(uchar *int_arg) { return false;}
};
```


具体的，除法执行的函数是`Item_func_div::decimal_op`函数，执行除法的调用栈如下：
```c++
Item_func_div::decimal_op(Item_func_div * const this, my_decimal * decimal_value) (mysql-server\sql\item_func.cc:2157)
Item_func_numhybrid::val_str(Item_func_numhybrid * const this, String * str) (mysql-server\sql\item_func.cc:1355)
Item::send(Item * const this, Protocol * protocol, String * buffer) (mysql-server\sql\item.cc:7541)
THD::send_result_set_row(THD * const this, List<Item> * row_items) (mysql-server\sql\sql_class.cc:4747)
Query_result_send::send_data(Query_result_send * const this, List<Item> & items) (mysql-server\sql\sql_class.cc:2740)
JOIN::exec(JOIN * const this) (mysql-server\sql\sql_executor.cc:165)
handle_query(THD * thd, LEX * lex, Query_result * result, ulonglong added_options, ulonglong removed_options) (mysql-server\sql\sql_select.cc:191)
execute_sqlcom_select(THD * thd, TABLE_LIST * all_tables) (mysql-server\sql\sql_parse.cc:5156)
mysql_execute_command(THD * thd, bool first_level) (mysql-server\sql\sql_parse.cc:2829)
mysql_parse(THD * thd, Parser_state * parser_state) (mysql-server\sql\sql_parse.cc:5589)
dispatch_command(THD * thd, const COM_DATA * com_data, enum_server_command command) (mysql-server\sql\sql_parse.cc:1492)
do_command(THD * thd) (mysql-server\sql\sql_parse.cc:1031)
handle_connection(void * arg)
```