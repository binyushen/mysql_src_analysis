# <center>memory pool</center>

## 1. mem_pool_t

	struct mem_pool_t {
		byte* buf; //内存指针
		ulint size; //内存池大小
		ulint reserved; //现在被分配出去内存的大小
		ib_mutex_t mutex; //用于此结构互斥访问
		UT_LIST_BASE_NODE_T(mem_area_t) free_list[64]; //空闲空间链表
	}
	//采用伙伴算法实现的内存池

## 2. buf_pool_t

	struct buf_pool_t {
		/*一般域*/
		ib_mutex_t	mutex; //本结构实例的互斥访问变量
		ib_mutex_t	zip_mutex; //保护压缩页
		ulint		instance_no; //本buffer pool instance的数组索引
		ulint		old_pool_size; // ?
		ulint		curr_pool_size;	// 
		ulint		LRU_old_ratio; //LRU中old blocks数量
		ulint		n_chunks;
		buf_chunk_t*	chunks;
		ulint		curr_size; // ?
		hash_table_t*	page_hash;
		hash_table_t*	zip_hash;
		ulint		n_pend_reads; // 读操作积累的数量
		ulint		n_pend_unzip; // 
		time_t		last_printout_time;
		buf_buddy_stat_t buddy_stat[BUF_BUDDY_SIZES_MAX + 1];
		buf_pool_stat_t	stat; // 统计信息
		buf_pool_stat_t	old_stat; 
		

		/*刷页算法所用域*/
		ib_mutex_t	flush_list_mutex; //保护flush_list
		const buf_page_t*	flush_list_hp;
		UT_LIST_BASE_NODE_T(buf_page_t) flush_list;
		ibool		init_flush[BUF_FLUSH_N_TYPES];
		ulint		n_flush[BUF_FLUSH_N_TYPES];
		os_event_t	no_flush[BUF_FLUSH_N_TYPES];



		/*LRU替换相关域*/


	}