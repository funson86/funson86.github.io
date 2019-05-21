---
title: Mysql源码分析2 - 事务
categories:
 - 源码分析 
tags:
 - mysql源码分析
---

## mysql事务分析

### 事务ACID特性
事务是由一组SQL语句组成的逻辑处理单元，事务具有以下4个属性，通常简称为事务的ACID属性。

- 原子性（Atomicity）：事务是一个原子操作单元，其对数据的修改，要么全都执行，要么全都不执行。
- 一致性（Consistent）：在事务开始和完成时，数据都必须保持一致状态。这意味着所有相关的数据规则都必须应用于事务的修改，以保持数据的完整性；事务结束时，所有的内部数据结构（如B树索引或双向链表）也都必须是正确的。
- 隔离性（Isolation）：数据库系统提供一定的隔离机制，保证事务在不受外部并发操作影响的“独立”环境执行。这意味着事务处理过程中的中间状态对外部是不可见的，反之亦然。
- 持久性（Durable）：事务完成之后，它对于数据的修改是永久性的，即使出现系统故障也能够保持。

### 并发事务存在的问题

当有多个事务并发处理，会带来一些问题：

- 更新丢失（Lost Update）：当两个或多个事务选择同一行，然后基于最初选定的值更新该行时，由于每个事务都不知道其他事务的存在，就会发生丢失更新问题－－最后的更 新覆盖了由其他事务所做的更新。例如，两个编辑人员制作了同一文档的电子副本。每个编辑人员独立地更改其副本，然后保存更改后的副本，这样就覆盖了原始文 档。最后保存其更改副本的编辑人员覆盖另一个编辑人员所做的更改。如果在一个编辑人员完成并提交事务之前，另一个编辑人员不能访问同一文件，则可避免此问 题。
- 脏读（Dirty Reads）：一个事务正在对一条记录做修改，在这个事务完成并提交前，这条记录的数据就处于不一致状态；这时，另一个事务也来读取同一条记录，如果不加 控制，第二个事务读取了这些“脏”数据，并据此做进一步的处理，就会产生未提交的数据依赖关系。这种现象被形象地叫做"脏读"。
- 不可重复读（Non-Repeatable Reads）：一个事务在读取某些数据后的某个时间，再次读取以前读过的数据，却发现其读出的数据已经发生了改变、或某些记录已经被删除了！这种现象就叫做“不可重复读”。
- 幻读（Phantom Reads）：一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据，这种现象就称为“幻读”。

### 事务隔离级别
表级别锁有两种，一种表锁，一种为元数据锁(MDL lock, meta data lock)

- READ UNCOMMITTED 读未提交

其他事务未提交的数据也能读取，实际应用中很少使用。


- READ COMMITTED 读已提交

事务直到提交之前，所做的任何修改对其他事务不可见的。


- REPEATABLE READ 可重复读 (MYSQL默认级别)
解决不可重复读问题，多次读取同一条记录得到相同的值。

但是没有解决幻读问题（某个事务读取某个范围的记录时，另外一个事务插入一条数据，前事务再次读取范围记录时产生幻行）

InnoDB和XtraDB存储引擎通过MVCC和间隙锁解决幻读问题。


- SERIALIZABLE 串行化
最高级别，强制事务串行执行，可以避免并发事务的所有问题包括幻读。会在读取的每一行数据加锁，可能导致大量的超时和锁争用。很少使用



### InnoDB多版本并发控制MVCC
通过在每条记录后面保存两个隐藏的列来实现

- 一个保存行创建时间（系统版本号） 每开始一个新的事务，系统版本号自动增加
- -个保存行删除时间（系统版本号，即该版本号之前有效）


通过这两个字段，在REPEATABLE READ隔离级别下，MVCC具体操作过程：

- INSERT 为新插入的每一行保存当前系统版本号为行版本号
- DELETE 为删除的每一行保存当前系统版本号为行删除标识
- UPDATE 执行上述2个动作，新增加一条记录保存当前系统版本号为行版本号  保存当前系统版本号到原数据的行删除标识
- SELECT 根据两个条件检查每行数据 1.系统版本号小于当前事务版本号  2.行删除标识要么未定义，要么大于当前版本号

优点：大多数读操作不用加锁

缺点：每行记录需要额外的存储空间

参考：

[InnoDB MVCC实现原理及源码解析](https://blog.csdn.net/yanzongshuai/article/details/79949332)

[InnoDB存储引擎MVCC实现原理](https://zhuanlan.zhihu.com/p/29532524)


### InnoDB三种日志：

- Bin Log 服务层日志，用来进行数据恢复、数据库复制，主从架构是master往slave传binlog实现的
- Redo Log 数据在物理层面的修改，mysql使用大量缓存，修改操作直接修改内存，而不是直接修改磁盘。事务中不断产生redolog，在事务提交的时候一次修改保存到磁盘中。当数据库或者主机失效重启时，会根据redolog进行恢复。
- Undo Log 数据修改时候记录undolog，根据undolog回溯摸个特定的版本，实现MVCC

read_view_t主要包括3个数据成员{low_limit_id,up_limit_id,trx_ids}。

- low_limit_id：表示创建read view时，当前事务活跃读写链表最大的事务ID，即最近创建的除自身外最大的事务ID
- up_limit_id：表示创建read view时，当前事务活跃读写链表最小的事务ID。
- trx_ids：创建read view时，活跃事务链表里所有事务ID

对于RR隔离级别，则SQL语句结束后不会删除read_view，从而下一个SQL语句时，使用上次申请的，这样保证事务中的read view都一样，从而实现可重复读的隔离级别。
对于可见性判断，分配聚集索引和二级索引。

- 聚集索引：记录的DATA_TRX_ID < view->up_limit_id：在创建read view时，修改该记录的事务已提交，该记录可见 DATA_TRX_ID 位于（view->up_limit_id，view->low_limit_id）：需要在活跃读写事务数组查找trx_id是否存在，如果存在，记录对于当前read view是不可见的。
- 由于InnoDB的二级索引只保存page最后更新的trx_id，当利用二级索引进行查询的时候，如果page的trx_id小于view->up_limit_id，可以直接判断page的所有记录对于当前view是可见的，否则需要回clustered索引进行判断。


```
//read0read.h
struct read_view_t{
	ulint		type;	/*!< VIEW_NORMAL, VIEW_HIGH_GRANULARITY */
	undo_no_t	undo_no;/*!< 0 or if type is
				VIEW_HIGH_GRANULARITY
				transaction undo_no when this high-granularity
				consistent read view was created */
	trx_id_t	low_limit_no;
				/*!< The view does not need to see the undo
				logs for transactions whose transaction number
				is strictly smaller (<) than this value: they
				can be removed in purge if not needed by other
				views */
	trx_id_t	low_limit_id;
				/*!< The read should not see any transaction
				with trx id >= this value. In other words,
				this is the "high water mark". */
	trx_id_t	up_limit_id;
				/*!< The read should see all trx ids which
				are strictly smaller (<) than this value.
				In other words,
				this is the "low water mark". */
	ulint		n_trx_ids;
				/*!< Number of cells in the trx_ids array */
	trx_id_t*	trx_ids;/*!< Additional trx ids which the read should
				not see: typically, these are the read-write
				active transactions at the time when the read
				is serialized, except the reading transaction
				itself; the trx ids in this array are in a
				descending order. These trx_ids should be
				between the "low" and "high" water marks,
				that is, up_limit_id and low_limit_id. */
	trx_id_t	creator_trx_id;
				/*!< trx id of creating transaction, or
				0 used in purge */
	UT_LIST_NODE_T(read_view_t) view_list;
				/*!< List of read views in trx_sys */
};

bool
lock_clust_rec_cons_read_sees(
/*==========================*/
	const rec_t*	rec,	/*!< in: user record which should be read or
				passed over by a read cursor */
	dict_index_t*	index,	/*!< in: clustered index */
	const ulint*	offsets,/*!< in: rec_get_offsets(rec, index) */
	read_view_t*	view)	/*!< in: consistent read view */
{
	trx_id_t	trx_id;

	ut_ad(dict_index_is_clust(index));
	ut_ad(page_rec_is_user_rec(rec));
	ut_ad(rec_offs_validate(rec, index, offsets));

	/* NOTE that we call this function while holding the search
	system latch. */

	trx_id = row_get_rec_trx_id(rec, index, offsets);

	return(read_view_sees_trx_id(view, trx_id));
}

bool
read_view_sees_trx_id(
/*==================*/
	const read_view_t*	view,	/*!< in: read view */
	trx_id_t		trx_id)	/*!< in: trx id */
{
	if (trx_id < view->up_limit_id) {//小于最小trx_id，可见

		return(true);
	} else if (trx_id >= view->low_limit_id) {//大于最大trx_id，不可见

		return(false);
	} else { // 用二分法查询是否在trx_ids里面，如果在，则不可见
		ulint	lower = 0;
		ulint	upper = view->n_trx_ids - 1;

		ut_a(view->n_trx_ids > 0);

		do {
			ulint		mid	= (lower + upper) >> 1;
			trx_id_t	mid_id	= view->trx_ids[mid];

			if (mid_id == trx_id) {
				return(FALSE);
			} else if (mid_id < trx_id) {
				if (mid > 0) {
					upper = mid - 1;
				} else {
					break;
				}
			} else {
				lower = mid + 1;
			}
		} while (lower <= upper);
	}

	return(true);
}
```


