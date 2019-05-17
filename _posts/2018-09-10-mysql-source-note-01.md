---
title: Mysql学习笔记分析
categories:
 - 源码分析 mysql
tags:
 - 源码分析 mysql
---

## mysql架构
和其他软件类似，mysql使用网络端口方式提供服务，服务端大体分成三层，

客户端 | 客户端
--------- | -------------
连接/线程处理 | 连接/线程处理
查询缓存 | 解析器
存储引擎 | 存储引擎

### 1. 连接/线程处理 
和大多数客户端/服务器有类似架构，比如连接处理，授权认证，安全等。<a href="#1.01">源码分析</a>

- 用户连接成功后，就会读取对应的权限，此时如果修改用户权限，已经生效的连接不会使用新的权限配置 <a href="#1.03">源码分析</a>
- 客户端太长时间没动静，连接器会主动断开，由参数wait_timeout控制，默认8小时
- 尽量使用长连接，但是内存会涨得很快，因为mysql在执行过程中使用的内存在连接的时候才释放，长期累积可能导致内存太大。一种方案是定期断开长连接再重连；一种方案是5.7之后执行mysql_reset_connection来初始化连接资源，此过程不需要重连和重新权限验证。

### 2. 查询缓存
mysql在执行查询请求后，先去查询缓存中看看是不是执行过这条语句。之前执行的语句和结果会以key-value缓存在内存中，key是查询语句，value是结果。

- 查询缓存很容易失效，只要有一个表更新，所有和该表有关的查询缓存都会清空
- 适合更新频率很低的表，系统配置表适合查询缓存
- 将参数query_cache_type设置成DEMAND，默认语句不适用查询缓存，需要显式地指定，select SQL_CACHE * from user where id = 100;


### 3.分析器
分析器分为词法分析、优化以及内置函数和所有跨存储引擎功能如存储过程、触发器、视图等。

- 词法分析 判断用户的语句是否存在语法错误
- 优化器 多索引时决定使用哪个索引，多表关联时决定连接顺序


### 4.执行器
执行器执行语句

- 执行前会再次判断用户权限
- 调用引擎接口取第一行和下一行，到最后一行返回结果给客户端
- 可以在慢查询日志中看到rows_examined表示语句执行中扫描了多少行


## mysql源码分析

### <a name="1.03">mysql启动流程</a>

- 启动文件在sql/mysqld.cc文件中
- myssql_init() 调用mysys/my_init.c文件函数，初始化内部系统库
- load_defaults() 读取my.cnf配置
- logger.init_base(); 初始化日志
- init_common_variables(); 初始化公共变量 包含init_thread_environment() mysql_init_variables()
- my_init_signals 初始化信号
- user_info= check_user(mysqld_user) 检测用户是否有权限启动
- init_server_components() 初始化内部组件 MDL_map::init()MDL锁初始化  Query_cache::init()初始化查询缓存 randominit()随机数初始化 init_slave_list()如果主从复制初始化从库
- network_init() 初始化网络，创建socket，监听端口
- start_signal_handler() 创建pid文件
- mysql_rm_tmp_tables() 删除临时表  acl_init(opt_noacl) 初始化数据库权限 my_tz_init((THD *)0, default_tz_name, opt_bootstrap) 初始化时区
- init_status_vars() 初始化all_status_vars变量
- create_shutdown_thread() windows下创建关闭线程
- start_handle_manager() 创建管理线程 执行handle_manager(void *arg MY_ATTRIBUTE((unused)))函数
- handle_connections_sockets() 处理连接socket，创建新的线程处理新的socket连接

### <a name="1.04"> handle_connections_sockets解析 </a>

基本流程如下

```
  FD_SET(mysql_socket_getfd(ip_sock), &clientFDs);
  while (!abort_loop)
  {
    readFDs=clientFDs;

    retval= select((int) max_used_connection,&readFDs,0,0,0);

    thd= new THD;
    create_new_thread(thd);
    MYSQL_CALLBACK(thread_scheduler, add_connection, (thd));
// add_connection 对应 handle_connection_in_main_thread 或 create_thread_to_handle_connection
  }
    
void create_thread_to_handle_connection(THD *thd)
{
mysql_thread_create(key_thread_one_connection, &thd->real_id, &connection_attrib, handle_one_connection, (void*) thd)
}

pthread_handler_t handle_one_connection(void *arg)
{
  THD *thd= (THD*) arg;

  mysql_thread_set_psi_id(thd->thread_id);

  do_handle_one_connection(thd);
  return 0;
}

void do_handle_one_connection(THD *thd_arg)
{
  for (;;)
  {
    while (thd_is_connection_alive(thd))
    {
      mysql_audit_release(thd);
      if (do_command(thd))
  break;
    }
    end_connection(thd);
  }
}
```

处理具体的命令do_command()

```
enum enum_server_command
{
  COM_SLEEP, COM_QUIT, COM_INIT_DB, COM_QUERY, COM_FIELD_LIST,
  COM_CREATE_DB, COM_DROP_DB, COM_REFRESH, COM_SHUTDOWN, COM_STATISTICS,
  COM_PROCESS_INFO, COM_CONNECT, COM_PROCESS_KILL, COM_DEBUG, COM_PING,
  COM_TIME, COM_DELAYED_INSERT, COM_CHANGE_USER, COM_BINLOG_DUMP,
  COM_TABLE_DUMP, COM_CONNECT_OUT, COM_REGISTER_SLAVE,
  COM_STMT_PREPARE, COM_STMT_EXECUTE, COM_STMT_SEND_LONG_DATA, COM_STMT_CLOSE,
  COM_STMT_RESET, COM_SET_OPTION, COM_STMT_FETCH, COM_DAEMON,
  COM_BINLOG_DUMP_GTID,
  /* don't forget to update const char *command_name[] in sql_parse.cc */

  /* Must be last */
  COM_END
};

bool do_command(THD *thd)
{
  NET *net= &thd->net;
  packet= (char*) net->read_pos;
  command= (enum enum_server_command) (uchar) packet[0];
  return_value= dispatch_command(command, thd, packet+1, (uint) (packet_length-1));
}

bool dispatch_command(enum enum_server_command command, THD *thd,
		      char* packet, uint packet_length)
{
  switch (command) {
  case COM_INIT_DB:
  {
    LEX_STRING tmp;
    status_var_increment(thd->status_var.com_stat[SQLCOM_CHANGE_DB]);
    thd->convert_string(&tmp, system_charset_info,
			packet, packet_length, thd->charset());
    if (!mysql_change_db(thd, &tmp, FALSE))
    {
      general_log_write(thd, command, thd->db, thd->db_length);
      my_ok(thd);
    }
    break;
  }
  case COM_REGISTER_SLAVE:
  case COM_CHANGE_USER:
  case COM_STMT_EXECUTE:
  case COM_STMT_FETCH:
  ...
  case COM_QUERY:
  {
    mysql_parse(thd, thd->query(), thd->query_length(), &parser_state); //解析查询sql语句
  }
  ...
  case COM_QUIT:
  case COM_REFRESH:
  ...
  case COM_SET_OPTION:
  case COM_SLEEP:
  case COM_CONNECT:				// Impossible here
  case COM_TIME:				// Impossible from client
  case COM_DELAYED_INSERT:
  case COM_END:
  default:
    my_message(ER_UNKNOWN_COM_ERROR, ER(ER_UNKNOWN_COM_ERROR), MYF(0));
    break;
  }
}

//解析查询sql语句
void mysql_parse(THD *thd, char *rawbuf, uint length, Parser_state *parser_state)
{
  lex_start(thd);
  mysql_reset_thd_for_next_command(thd);
  if (query_cache_send_result_to_client(thd, rawbuf, length) <= 0) //先从查询缓存中读取
  {
    error= mysql_execute_command(thd);
  }
  else
  {
    thd->m_statement_psi= MYSQL_REFINE_STATEMENT(thd->m_statement_psi, sql_statement_info[SQLCOM_SELECT].m_key);
    if (!opt_log_raw)
      general_log_write(thd, COM_QUERY, thd->query(), thd->query_length());
    parser_state->m_lip.found_semicolon= NULL;
  }
}


//执行具体的sql语句
enum enum_sql_command {
  SQLCOM_SELECT, SQLCOM_CREATE_TABLE, ...  SQLCOM_ALTER_USER,
  SQLCOM_END
};
int mysql_execute_command(THD *thd)
{
  LEX  *lex= thd->lex;
  switch (lex->sql_command) {

  case SQLCOM_SHOW_STATUS:
  case SQLCOM_SHOW_PROFILE:
  case SQLCOM_SELECT:
  {
    if ((res= select_precheck(thd, lex, all_tables, first_table)))
      break;

    res= execute_sqlcom_select(thd, all_tables);
  }
case SQLCOM_PREPARE:
  {
    mysql_sql_stmt_prepare(thd);
    break;
  }
  case SQLCOM_EXECUTE:
  {
    mysql_sql_stmt_execute(thd);
    break;
  }
  case SQLCOM_CREATE_TABLE:
  {
    res= mysql_create_table(thd, create_table, &create_info, &alter_info);
  }
  case SQLCOM_INSERT:
  {
    res= open_temporary_tables(thd, all_tables);
    insert_precheck(thd, all_tables);
    MYSQL_INSERT_START(thd->query());
    res= mysql_insert(thd, all_tables, lex->field_list, lex->many_values,
		      lex->update_list, lex->value_list,
                      lex->duplicates, lex->ignore);
    MYSQL_INSERT_DONE(res, (ulong) thd->get_row_count_func());
  }
}

//具体查询函数execute_sqlcom_select解析
static bool execute_sqlcom_select(THD *thd, TABLE_LIST *all_tables)
{
  select_result *result= lex->result;
  open_normal_and_derived_tables(thd, all_tables, 0));
  res= handle_select(thd, result, 0);
  return res;
}
bool open_normal_and_derived_tables(THD *thd, TABLE_LIST *tables, uint flags)
{
  open_tables(thd, &tables, &thd->lex->table_count, flags,&prelocking_strategy);
  mysql_handle_derived(thd->lex, &mysql_derived_prepare))
}
bool handle_select(THD *thd, select_result *result,ulong setup_tables_done_option)
{
    res= mysql_select(thd,
		      select_lex->table_list.first,
		      select_lex->with_wild, select_lex->item_list,
		      select_lex->where,
		      &select_lex->order_list,
		      &select_lex->group_list,
		      select_lex->having,
		      select_lex->options | thd->variables.option_bits |
                      setup_tables_done_option,
		      result, unit, select_lex);
}

bool mysql_select(THD *thd,
             TABLE_LIST *tables, uint wild_num, List<Item> &fields,
             Item *conds, SQL_I_List<ORDER> *order, SQL_I_List<ORDER> *group,
             Item *having, ulonglong select_options,
             select_result *result, SELECT_LEX_UNIT *unit,
             SELECT_LEX *select_lex)
{
  mysql_execute_select(thd, select_lex, free_join)
}
static bool
mysql_execute_select(THD *thd, SELECT_LEX *select_lex, bool free_join)
{
  JOIN* join= select_lex->join;
  join->exec();
}
void JOIN::exec()
{
  result->send_result_set_metadata(*fields,
                                   Protocol::SEND_NUM_ROWS | Protocol::SEND_EOF);
  error= do_select(this);
}
static int do_select(JOIN *join)
{
}

```











