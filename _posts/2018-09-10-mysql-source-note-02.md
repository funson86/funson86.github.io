---
title: Mysql源码分析2 - 并发控制和锁
categories:
 - 源码分析 
tags:
 - mysql源码分析
---



### 读写锁
mysql的并发通过锁来进行并发控制，锁的种类分为2种

- 共享锁(shared lock)也叫读锁(read lock)
- 排他锁(exclusive lock)也叫写锁(write lock)

读锁是共享的，多个客户可以在同一时间同时读取同一个资源互不干扰

写锁是排他的，一个写锁会阻塞其他的写锁和读锁

### 锁的粒度有3种

锁粒度即加锁的锁定范围，不同锁粒度的开销也不同，mysql每种存储引擎可以实现自己的锁策略和粒度。

- 全局锁 对整个数据库实例加锁，命令为Flush tables with read lock (FTWRL)，一般用在整个库做逻辑备份，需要把每个表select出来存成文本
- 表锁(table lock) 对整张表加锁，开销较小，写锁会阻塞其他读写操作，读锁可以共享。表级别锁有两种，一种表锁，一种为元数据锁(MDL lock, meta data lock)
- 行锁(row lock) 可以最大程度地支持并发处理，同时也带来最大的锁开销。行锁只在存储引擎中实现，服务器层不实现。


### 表锁
表级别锁有两种，一种表锁，一种为元数据锁(MDL lock, meta data lock)

- 表锁

表锁的语法是 lock tables ... read/write。与 FTWRL 类似，可以用 unlock tables 主动释放 锁，也可以在客户端断开的时候自动释放。

- 元数据锁MDL锁

5.5版本后引入的，不需要显示的使用，在访问表时自动加上。

### 行锁
行锁只在存储引擎中实现，服务器层不实现。InnoDB和XtraDB实现了行锁

### InnoDB的行锁模式及加锁方法
[MySQL－ InnoDB锁机制](https://www.cnblogs.com/aipiaoborensheng/p/5767459.html)

InnoDB实现了以下两种类型的行锁。

- 共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。
- 排他锁（X)：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。

另外，为了允许行锁和表锁共存，实现多粒度锁机制，InnoDB还有两种内部使用的意向锁（Intention Locks），这两种意向锁都是表锁。

- 意向共享锁（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的IS锁。
- 意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的IX锁。

```
# InnoDB行锁模式兼容性列表
         | Type of active   |
Request  |   scoped lock    |
type     | IS(*)  IX   S  X |
---------+------------------+
IS       |  +      +   +  + |
IX       |  +      +   -  - |
S        |  +      -   +  - |
X        |  +      -   -  - |
----------------------------- 
+号表示请求的锁可以满足。
-号表示请求的锁无法满足需要等待。
如果一个事务请求的锁模式与当前的锁兼容，InnoDB就将请求的锁授予该事务；反之，如果两者不兼容，该事务就要等待锁释放。
意向锁是InnoDB自动加的，不需用户干预。
```

事务可以通过以下语句显示给记录集加共享锁或排他锁。

- 共享锁（S）：SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE。
- 排他锁（X)：SELECT * FROM table_name WHERE ... FOR UPDATE。

用SELECT ... IN SHARE MODE获得共享锁，主要用在需要数据依存关系时来确认某行记录是否存在，并确保没有人对这个记录进行UPDATE或者DELETE操作。但是如果当前事 务也需要对该记录进行更新操作，则很有可能造成死锁，对于锁定行记录后需要进行更新操作的应用，应该使用SELECT... FOR UPDATE方式获得排他锁。

InnoDB行锁是通过给索引上的索引项加锁来实现的，InnoDB这种行锁实现特点意味着：只有通过索引条件检索 数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁！ 



## mysql源码分析

### <a name="2.01">全局锁Flush tables with read lock</a>
```
// sql/sql_yacc.yy
flush:
          FLUSH_SYM opt_no_write_to_binlog
          {
            LEX *lex=Lex;
            lex->sql_command= SQLCOM_FLUSH;
            lex->type= 0;
            lex->no_write_to_binlog= $2;
          }
          flush_options
          {}
        ;

flush_options:
          table_or_tables
          {
            Lex->type|= REFRESH_TABLES;
            /*
              Set type of metadata and table locks for
              FLUSH TABLES table_list [WITH READ LOCK].
            */
            YYPS->m_lock_type= TL_READ_NO_INSERT;
            YYPS->m_mdl_type= MDL_SHARED_HIGH_PRIO;
          }
          opt_table_list {}
          opt_flush_lock {}
        | flush_options_list
        ;

opt_flush_lock:
          /* empty */ {}
        | WITH READ_SYM LOCK_SYM
          {
            TABLE_LIST *tables= Lex->query_tables;
            Lex->type|= REFRESH_READ_LOCK;
            for (; tables; tables= tables->next_global)
            {
              tables->mdl_request.set_type(MDL_SHARED_NO_WRITE);
              tables->required_type= FRMTYPE_TABLE; /* Don't try to flush views. */
              tables->open_type= OT_BASE_ONLY;      /* Ignore temporary tables. */
            }
          }

//sql/sql_parse.cc
  case SQLCOM_FLUSH:
  {
      if (flush_tables_with_read_lock(thd, all_tables))
        goto error;

    if (!reload_acl_and_cache(thd, lex->type, first_table, &write_to_binlog))
    {
    }

  }
  
bool flush_tables_with_read_lock(THD *thd, TABLE_LIST *all_tables)
{
  if (lock_table_names(thd, all_tables, NULL,
                       thd->variables.lock_wait_timeout,
                       MYSQL_OPEN_SKIP_SCOPED_MDL_LOCK))
    goto error;
}

lock_table_names(THD *thd,
                 TABLE_LIST *tables_start, TABLE_LIST *tables_end,
                 ulong lock_wait_timeout, uint flags)
{
  for (table= tables_start; table && table != tables_end;
       table= table->next_global)
  {
  
  }
}

reload_acl_and_cache
{
  if (options & (REFRESH_TABLES | REFRESH_READ_LOCK))
  {
    if ((options & REFRESH_READ_LOCK) && thd)
    {
      ...
      if (thd->global_read_lock.lock_global_read_lock(thd))//当未指定表的时候，加全局锁
        return 1;
      ...
      if (thd->global_read_lock.make_global_read_lock_block_commit(thd))//当未指定表的时候，加COMMIT锁
}

  
```

### <a name="2.01">表锁lock tables ... read/write</a>
```
//sql/sql_parse.cc
  case SQLCOM_LOCK_TABLES:
    
    res= lock_tables_open_and_lock_tables(thd, all_tables);
    
static bool lock_tables_open_and_lock_tables(THD *thd, TABLE_LIST *tables)
{
  open_tables(thd, &tables, &counter, 0, &lock_tables_prelocking_strategy);
    for (table= tables; table; table= table->next_global)
    if (!table->placeholder() && table->table->s->tmp_table)
      table->table->reginfo.lock_type= TL_WRITE;

  if (lock_tables(thd, tables, counter, 0) ||
      thd->locked_tables_list.init_locked_tables(thd))
    goto err;

  thd->in_lock_tables= 0;
}
```

### MDL锁

参考 [MySQL - 常用SQL语句的MDL加锁源码分析](http://xnerv.wang/analysis-of-mdl-source-code-for-common-sql-statements)

```
class MDL_key
{
//锁粒度 锁的范围
  enum enum_mdl_namespace { GLOBAL=0, //用于global read lock，例如FLUSH TABLES WITH READ LOCK。
                            SCHEMA, // 用于保护tablespace/schema。
                            TABLE, // 用于保护tablespace/schema。
                            FUNCTION, //用于保护function/procedure/trigger/event。
                            PROCEDURE, // 用于保护function/procedure/trigger/event。
                            TRIGGER, //用于保护function/procedure/trigger/event。
                            EVENT, // 用于保护function/procedure/trigger/event。
                            COMMIT, // 主要用于global read lock后，阻塞事务提交。
                            /* This should be the last ! */
                            NAMESPACE_END };

  const char *db_name() const { return m_ptr + 1; }
  const char *name() const { return m_ptr + m_db_name_length + 2; }
}
                            
                            
enum enum_mdl_type {
  MDL_INTENTION_EXCLUSIVE= 0, //意向排他锁，锁定一个范围，用在GLOBAL/SCHEMA/COMMIT粒度。
  MDL_SHARED, // 用在只访问元数据信息，不访问数据。例如CREATE TABLE t LIKE t1;
  MDL_SHARED_HIGH_PRIO, // 也是用于只访问元数据信息，但是优先级比排他锁高，用于访问information_schema的表。例如：select * from information_schema.tables;
  MDL_SHARED_READ, // 访问表结构并且读表数据，例如：SELECT * FROM t1; LOCK TABLE t1 READ LOCAL;
  MDL_SHARED_WRITE, // 访问表结构且写表数据， 例如：INSERT/DELETE/UPDATE t1 … ;SELECT * FROM t1 FOR UPDATE;LOCK TALE t1 WRITE
  MDL_SHARED_UPGRADABLE, 可升级锁，允许并发update/read表数据。持有该锁可以同时读取表metadata和表数据，但不能修改数据。可以升级到SNW、SNR、X锁。用在alter table的第一阶段，使alter table的时候不阻塞DML，防止其他DDL。
  MDL_SHARED_NO_WRITE, // 持有该锁可以读取表metadata和表数据，同时阻塞所有的表数据修改操作，允许读。可以升级到X锁。用在ALTER TABLE第一阶段，拷贝原始表数据到新表，允许读但不允许更新。
  MDL_SHARED_NO_READ_WRITE,// 持有该锁可以读取表metadata和表数据，同时阻塞所有的表数据修改操作，允许读。可以升级到X锁。用在ALTER TABLE第一阶段，拷贝原始表数据到新表，允许读但不允许更新。
  MDL_EXCLUSIVE, // 排他锁，持有该锁连接可以修改表结构和表数据，使用在CREATE/DROP/RENAME/ALTER TABLE 语句。
  /* This should be the last !!! */
  MDL_TYPE_END};
  
enum enum_mdl_duration {
  MDL_STATEMENT= 0, //语句中持有，语句执行完释放
  MDL_TRANSACTION, //事务中持有，事务完成后释放
  MDL_EXPLICIT, //需要显式地调用MDL_context::release_lock()释放
  /* This should be the last ! */
  MDL_DURATION_END };

```

### 间隙锁 - REPEATABLE READ级别才有

MYSQL数据库间隙锁(https://blog.csdn.net/sinat_27143551/article/details/81736330)
当我们用范围条件而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的已有数据记录的索引项加锁；对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP)”，InnoDB也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁（Next-Key锁）。


举例来说，假如user表中只有101条记录，其empid的值分别是 1,2,...,100,101，下面的SQL：

select * from  user where user_id > 100 for update;
是一个范围条件的检索，InnoDB不仅会对符合条件的user_id值为101的记录加锁，也会对user_id大于101（这些记录并不存在）的“间隙”加锁。

InnoDB使用间隙锁的目的，一方面是为了防止幻读，以满足相关隔离级别的要求，对于上面的例子，要是不使用间隙锁，如果其他事务插入了user_id大于100的任何记录，那么本事务如果再次执行上述语句，就会发生幻读；另外一方面，是为了满足其恢复和复制的需要


### 悲观锁 & 乐观锁
悲观锁与乐观锁是两种常见的资源并发锁设计思路，也是并发编程中一个非常基础的概念。

- 悲观锁的特点是先获取锁，再进行业务操作，即“悲观”的认为获取锁是非常有可能失败的，因此要先确保获取锁成功再进行业务操作。

通常所说的“一锁二查三更新”即指的是使用悲观锁。通常来讲在数据库上的悲观锁需要数据库本身提供支持，即通过常用的select … for update操作来实现悲观锁。当数据库执行select for update时会获取被select中的数据行的行锁，因此其他并发执行的select for update如果试图选中同一行则会发生排斥（需要等待行锁被释放），因此达到锁的效果。select for update获取的行锁会在当前事务结束时自动释放，因此必须在事务中使用。

- 乐观锁的特点先进行业务操作，不到万不得已不去拿锁。即“乐观”的认为拿锁多半是会成功的，因此在进行完业务操作需要实际更新数据的最后一步再去拿一下锁就好。

乐观锁在数据库上的实现完全是逻辑的，不需要数据库提供特殊的支持。一般的做法是在需要锁的数据上增加一个版本号，或者时间戳，然后按照如下方式实现：

```
1. SELECT data AS old_data, version AS old_version FROM …;
2. 根据获取的数据进行业务操作，得到new_data和new_version
3. UPDATE SET data = new_data, version = new_version WHERE version = old_version
if (updated row > 0) {
    // 乐观锁获取成功，操作完成
} else {
    // 乐观锁获取失败，回滚并重试
}
```
乐观锁是否在事务中其实都是无所谓的，其底层机制是这样：在数据库内部update同一行的时候是不允许并发的，即数据库每次执行一条update语句时会获取被update行的写锁，直到这一行被成功更新后才释放。因此在业务操作进行前获取需要锁的数据的当前版本号，然后实际更新数据时再次对比版本号确认与之前获取的相同，并更新版本号，即可确认这之间没有发生并发的修改。如果更新失败即可认为老版本的数据已经被并发修改掉而不存在了，此时认为获取锁失败，需要回滚整个业务操作并可根据需要重试整个过程。


总结：

- 乐观锁在不发生取锁失败的情况下开销比悲观锁小，但是一旦发生失败回滚开销则比较大，因此适合用在取锁失败概率比较小的场景，可以提升系统并发性能

- 乐观锁还适用于一些比较特殊的场景，例如在业务操作过程中无法和数据库保持连接等悲观锁无法适用的地方
