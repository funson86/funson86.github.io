---
title: Mysql源码分析4 - 索引
categories:
 - 源码分析 
tags:
 - mysql源码分析
---


### 索引类型
索引是存储引擎用于快速找到记录的一种数据结构

- BTREE索引（B+树） 默认索引，使用B+树记录数据，每一个叶子节点到根距离相同，索引数据存储在叶子几点上（即叶子节点记录数据的地址或主键ID）
- 哈希索引 使用哈希表实现，智能精确匹配的数据查询才有效，不能进行范围搜索排序等，只有Memory引擎显式的支持该索引
- 全文索引 查找文本中的关键词，MYISAM支持，5.6之后的INNODB也支持，不是使用where去查，而是使用MATCH (COL) AGAINST (SEARCH)
- 空间索引 用作地理数据存储，并不完善

### B-TREE索引
B-TREE索引

- 单列索引

1. 普通索引 INDEX index_name(col1)
2. 唯一索引 索引列的值必须唯一，可以为空值 UNIQUE index index_name(col1)
3. 主键索引 唯一且不能为空值 PRIMARY KEY (`id`)

- 组合索引(最左前缀) 多列组合在一起的索引 index index_username_email(username, email)

### 聚集索引和非聚集索引
这两个不是实际的索引，而是一种存储方式。

- 聚集索引 根据主键值按照顺序紧凑的存储在一起，这样在索引中存储的是ID值，当有新的主键插入时，不需要更新索引中的键值。INNODB使用该索引，如果没有非空唯一列，INNODB会自动生成一个隐藏的唯一列。
- 非聚集索引 如Myisam，数据按照先后顺序存储，索引中存储的是数据的地址，当有新的数据插入的时候，需要计算索引中指向索引的指针。

### 高性能索引策略


索引基本数据结构

```
typedef struct st_key {
  /** Tot length of key */
  uint	key_length;
  /** dupp key and pack flags */
  ulong flags;
  /** dupp key and pack flags for actual key parts */
  ulong actual_flags;
  /** How many key_parts */
  uint  user_defined_key_parts;
  /** How many key_parts including hidden parts */
  uint  actual_key_parts;
  /**
     Key parts allocated for primary key parts extension but
     not used due to some reasons(no primary key, duplicated key parts)
  */
  uint  unused_key_parts;
  /** Should normally be = key_parts */
  uint	usable_key_parts;
  uint  block_size;
  enum  ha_key_alg algorithm;
  /**
    Note that parser is used when the table is opened for use, and
    parser_name is used when the table is being created.
  */
  union
  {
    /** Fulltext [pre]parser */
    plugin_ref parser;
    /** Fulltext [pre]parser name */
    LEX_STRING *parser_name;
  };
  KEY_PART_INFO *key_part;
  /** Name of key */
  char	*name;
  /**
    Array of AVG(#records with the same field value) for 1st ... Nth key part.
    0 means 'not known'.
    For temporary heap tables this member is NULL.
  */
  ulong *rec_per_key;
  union {
    int  bdb_return_if_eq;
  } handler;
  TABLE *table;
  LEX_STRING comment;
} KEY;

class KEY_PART_INFO {	/* Info about a key part */
public:
  Field *field;
  uint	offset;				/* offset in record (from 0) */
  uint	null_offset;			/* Offset to null_bit in record */
  /* Length of key part in bytes, excluding NULL flag and length bytes */
  uint16 length;
  /*
    Number of bytes required to store the keypart value. This may be
    different from the "length" field as it also counts
     - possible NULL-flag byte (see HA_KEY_NULL_LENGTH)
     - possible HA_KEY_BLOB_LENGTH bytes needed to store actual value length.
  */
  uint16 store_length;
  uint16 key_type;
  uint16 fieldnr;			/* Fieldnum in UNIREG */
  uint16 key_part_flag;			/* 0 or HA_REVERSE_SORT */
  uint8 type;
  uint8 null_bit;			/* Position to null_bit */
  void init_from_field(Field *fld);     /** Fill data from given field */
  void init_flags();                    /** Set key_part_flag from field */
};
```

插入索引源代码摘要

```
//storage/innobase/row/row0ins.cc
row_ins(
/*====*/
	ins_node_t*	node,	/*!< in: row insert node */
	que_thr_t*	thr)	/*!< in: query thread */
{
  row_ins_alloc_row_id_step(node);//插入数据到磁盘
  while (node->index != NULL) {//遍历索引，插入相关索引记录
    err = row_ins_index_entry_step(node, thr);
    node->index = dict_table_get_next_index(node->index);
  }
}

dberr_t
row_ins_index_entry_step(
/*=====================*/
	ins_node_t*	node,	/*!< in: row insert node */
	que_thr_t*	thr)	/*!< in: query thread */
{
  row_ins_index_entry_set_vals(node->index, node->entry, node->row);
  err = row_ins_index_entry(node->index, node->entry, thr);
}

row_ins_index_entry() 
{
	if (dict_index_is_clust(index)) {//聚集索引
		return(row_ins_clust_index_entry(index, entry, thr, 0));
	} else {//非聚集索引
		return(row_ins_sec_index_entry(index, entry, thr));
	}
}

row_ins_clust_index_entry() 
{
	err = row_ins_clust_index_entry_low(
		0, BTR_MODIFY_LEAF, index, n_uniq, entry, n_ext, thr);
}

row_ins_clust_index_entry_low()
{
  btr_cur_search_to_nth_level(index, 0, entry, PAGE_CUR_LE, mode,
				    &cursor, 0, __FILE__, __LINE__, &mtr);

  err = btr_cur_optimistic_insert(
				flags, &cursor, &offsets, &offsets_heap,
				entry, &insert_rec, &big_rec,
				n_ext, thr, &mtr);

}

```


