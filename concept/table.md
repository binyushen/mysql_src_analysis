# <center>表相关概念</center>

## 1. Field

	class Field {
		uchar *ptr; //一条记录中该域的位置
		uchar *null_ptr; //一条记录中NULL bit存放的位置，如果该域NOT NULL，该指针为NULL
		TABLE *table; //指向表的指针 ?
		TABLE *orig_table; // ?
		const char	**table_name, *field_name;
		LEX_STRING	comment;
		key_map key_start; //本域开始的主键
		key_map part_of_key; //包含本域的所有主键
		key_map part_of_key_not_clustered; //包含本域的所有非聚集键
		key_map part_of_sortkey; //包含本域且只用来排序的键
		
		uint32	field_length; //本域字段长度
		uint32	flags; // ?
		uint16  field_index; //在域数组中的索引号
		uchar null_bit; // ?
		bool is_created_from_null_item; // 
	}

## 1. TABLE_SHARE

	不同表相关对象间共享的结构，同一个表只对应一个TABLE_SHARE；

	struct TABLE_SHARE{
		TABLE_CATEGORY table_category;
		//枚举类，描述表的类别，包括TABLE_UNKNOWN_CATEGORY	、TABLE_CATEGORY_TEMPORARY、TABLE_CATEGORY_USER、TABLE_CATEGORY_SYSTEM、TABLE_CATEGORY_INFORMATION、TABLE_CATEGORY_LOG、TABLE_CATEGORY_PERFORMANCE、TABLE_CATEGORY_RPL_INFO

		HASH name_hash;
		//字段名哈希表
		MEM_ROOT mem_root;
		TYPELIB keynames; //主键名
		TYPELIB fieldnames; //域名
		TYPELIB *intervals; // ?
		mysql_mutex_t LOCK_ha_data; //保护ha_data
		TABLE_SHARE *next, **prev; //unused TABLE_SHARE链表指针
		Table_cache_element **cache_element;
		//表缓存数组指针 ?
		Field **field;
		Field **found_next_number_field;
		KEY  *key_info; //主键信息
		uint	*blob_field; //在Field数组中所有blobs字段的索引
		uchar	*default_values; // 默认值
		LEX_STRING comment; //表注释
		const CHARSET_INFO *table_charset; //表字符集
		MY_BITMAP all_set; // ?
		LEX_STRING table_cache_key; // 从table_cache或者thread‘s temporary tables中寻找表的key
		//格式"database_name\0table_name\0" + optional part for temporary tables
		LEX_STRING db; 
		LEX_STRING table_name; 
		LEX_STRING path; // .frm文件路径
		LEX_STRING normalized_path; // 
		LEX_STRING connect_string;
		key_map keys_in_use;
		key_map keys_for_keyread;
		ha_rows min_rows, max_rows;
		ulong   avg_row_length;
		ulong   version;
		ulong   mysql_version;
		ulong   reclength;
		

		uint ref_count; // 该TABLE_SHARE被引用次数
		uint key_block_size;
		uint stats_sample_pages;
		

	}
