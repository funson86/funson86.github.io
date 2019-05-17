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
- if (mysql_rm_tmp_tables() || acl_init(opt_noacl) || my_tz_init((THD *)0, default_tz_name, opt_bootstrap))  删除临时表 初始化数据库权限 初始化时区
- init_status_vars() 初始化all_status_vars变量
- create_shutdown_thread() windows下创建关闭线程
- start_handle_manager() 创建管理线程 执行handle_manager(void *arg MY_ATTRIBUTE((unused)))函数
- handle_connections_sockets() 处理连接socket
















