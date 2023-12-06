# Mysql 行的大小限制

```c
constexpr const int MAX_FIELD_VARCHARLENGTH{65535};
```

## sql 变长测试

### 测试一

```sql
CREATE TABLE t (a VARCHAR(10000), b VARCHAR(10000), c VARCHAR(10000), d VARCHAR(10000),  
e VARCHAR(10000), f VARCHAR(10000), g VARCHAR(6000)) ENGINE=InnoDB CHARACTER SET latin1;
```

该表的长度是 10000 * 6 + 6000 = 66000，遇到的报错是

```
Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs
```

### 测试二

```sql
CREATE  TABLE t (a VARCHAR(10000), b VARCHAR(10000), c VARCHAR(10000), 
d VARCHAR(10000), e VARCHAR(10000), f VARCHAR(10000), g TEXT(6000)) 
ENGINE=InnoDB CHARACTER  SET latin1; 
```

该表的最后字段改为了 TEXT，则可以正常创建

### 测试三

```sql
CREATE  TABLE t1 (c1 VARCHAR(32765)  NOT  NULL, c2 VARCHAR(32766)  NOT  NULL)  
ENGINE  = InnoDB CHARACTER  SET latin1;  
```

实际长度是 32765 + 32766 = 65531，因为 varchar 需要两字节用来存储长度，所以实际就是 65535。可以正常创建

### 测试四

```sql
CREATE  TABLE t2 (c1 VARCHAR(65535)  NOT  NULL)  ENGINE  = InnoDB CHARACTER  SET latin1; 
```

由于 varchar 额外占用了两字节，所以 t2 失败。

### 测试五

```sql
CREATE  TABLE ut1 (c1 VARCHAR(21845)  NOT  NULL)  ENGINE  = InnoDB CHARACTER  SET utf8;
```

由于 utf8 字符占用 3个字节，所以 21845 * 3 = 65535，所以也会失败。utf8mb4 就不试了，需要 * 4。

```sql
CREATE  TABLE t1 (c1 VARCHAR(21844)  NOT  NULL)  ENGINE  = InnoDB CHARACTER  SET utf8;
```

可以正常创建成功


## 单行 8126 测试

### 测试一

```sql
CREATE TABLE t4 (
       c1 CHAR(255),c2 CHAR(255),c3 CHAR(255),
       c4 CHAR(255),c5 CHAR(255),c6 CHAR(255),
       c7 CHAR(255),c8 CHAR(255),c9 CHAR(255),
       c10 CHAR(255),c11 CHAR(255),c12 CHAR(255),
       c13 CHAR(255),c14 CHAR(255),c15 CHAR(255),
       c16 CHAR(255),c17 CHAR(255),c18 CHAR(255),
       c19 CHAR(255),c20 CHAR(255),c21 CHAR(255),
       c22 CHAR(255),c23 CHAR(255),c24 CHAR(255),
       c25 CHAR(255),c26 CHAR(255),c27 CHAR(255),
       c28 CHAR(255),c29 CHAR(255),c30 CHAR(255),
       c31 CHAR(255),c32 CHAR(255),c33 CHAR(255)
       ) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET latin1;
```

在不存在变长字段的情况下，长度限制是 8126。 255 * 33 = 8145

```
Row size too large (> 8126). Changing some columns to TEXT or BLOB may help. In current row format, BLOB prefix of 0 bytes is stored inline.
```

### 测试二

```sql
CREATE TABLE t5 (
       c1 VARCHAR(255),c2 VARCHAR(255),c3 VARCHAR(255),
       c4 VARCHAR(255),c5 VARCHAR(255),c6 VARCHAR(255),
       c7 VARCHAR(255),c8 VARCHAR(255),c9 VARCHAR(255),
       c10 VARCHAR(255),c11 VARCHAR(255),c12 VARCHAR(255),
       c13 VARCHAR(255),c14 VARCHAR(255),c15 VARCHAR(255),
       c16 VARCHAR(255),c17 VARCHAR(255),c18 VARCHAR(255),
       c19 VARCHAR(255),c20 VARCHAR(255),c21 VARCHAR(255),
       c22 VARCHAR(255),c23 VARCHAR(255),c24 VARCHAR(255),
       c25 VARCHAR(255),c26 VARCHAR(255),c27 VARCHAR(255),
       c28 VARCHAR(255),c29 VARCHAR(255),c30 VARCHAR(255),
       c31 VARCHAR(255),c32 VARCHAR(255),c33 VARCHAR(255)
       ) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET latin1;
```

这种是正常的

## 源码分析

以下是产生报错的位置和原因，其中 page_rec_max 是限制，rec_max_size 是当前值
```c
// dict0dict.cc
bool dict_index_validate_max_rec_size(const dict_table_t *table,  
		const dict_index_t *index, bool strict,  
		const size_t page_rec_max,  
		const size_t page_ptr_max,  
		size_t &rec_max_size) {
	// rec_max_size 默认是 int 最大值
	// 此时 rec_max_size 的值是 5
	rec_max_size = comp ? REC_N_NEW_EXTRA_BYTES : REC_N_OLD_EXTRA_BYTES;

	rec_max_size += UT_BITS_IN_BYTES(index->n_nullable);
	
	for (size_t i = 0; i < index->n_fields; i++) {
		const dict_field_t *field = index->get_field(i);
		get_field_max_size(table, index, field, rec_max_size);
		// rec_max_size 现在是 13 
		// page_rec_max 的值是 8126
		if (rec_max_size >= page_rec_max) {
			return (true);
		}
	}
	return (false);
}
```

```c
// dict0dict.cc

void get_field_max_size(const dict_table_t *table, const dict_index_t *index,  
		const dict_field_t *field, size_t &rec_max_size) {
	ulint field_max_size;
	// 获取了字段的固定长度
	field_max_size = col->get_fixed_size(comp);	
	if (field_max_size && field->fixed_len != 0) {
		field_ext_max_size = 0;
		rec_max_size += field_max_size;
		return;
	}
}
```

### page_rec_max 的产生

```c
// dict0dict.cc

static bool dict_index_too_big_for_tree(const dict_table_t *table,  
		const dict_index_t *new_index,  
		bool strict) {

	size_t page_rec_max;
	size_t page_ptr_max;
	get_permissible_max_size(table, new_index, page_rec_max, page_ptr_max);
	
	// 以下是在计算字段的总大小，并判断是否超限
	size_t rec_max_size;  
	bool res = dict_index_validate_max_rec_size(  
				table, new_index, strict, page_rec_max, page_ptr_max, rec_max_size);
	return (res);
}
```

```c
void get_permissible_max_size(const dict_table_t *table,  
		const dict_index_t *index, size_t &page_rec_max,  
		size_t &page_ptr_max) {

	// srv_page_size 的值是 16384， UNIV_PAGE_SIZE_MAX 的值是 65535，不相等
	page_rec_max = srv_page_size == UNIV_PAGE_SIZE_MAX  
					? REC_MAX_DATA_SIZE - 1  : page_get_free_space_of_empty(comp) / 2;  
	page_ptr_max = page_rec_max;
}
```

```c

static inline ulint page_get_free_space_of_empty( bool comp)
{
	if (comp) {
		// UNIV_PAGE_SIZE 对应的是 srv_page_size，也就是 16384
		// 最终的值是 8126 * 2
		return ((ulint)(UNIV_PAGE_SIZE - PAGE_NEW_SUPREMUM_END - PAGE_DIR -  
					2 * PAGE_DIR_SLOT_SIZE));
	}
}
```

### 关于 innodb 的 page size

```c
// srv0srv.cc

ulong srv_page_size = UNIV_PAGE_SIZE_DEF;
```

```c
// univ.i

/** Default Page Size Shift (power of 2) */  
constexpr uint32_t UNIV_PAGE_SIZE_SHIFT_DEF = 14;

/** Default page size for InnoDB tablespaces. */  
constexpr uint32_t UNIV_PAGE_SIZE_DEF = 1 << UNIV_PAGE_SIZE_SHIFT_DEF;
```

## 小结

mysql 中 row 的最大长度限制是 64kb，但不同的引擎会有所不同。在 innodb 中，且不存在可变字段的情况下，长度限制为 page 大小的一半。

- innodb 的 page size 是 16kb
- 由于 innodb 限制了一页至少存储两行数据，所以一行数据的大小限制就是 8kb，也就是 8126