# <center>buffer pool</center>

## 2. buf_pool_t

	/*
		buffer pool是InnoDB控制的一个用来缓存数据和索引的内存结构，经常访问的数据会被放到这个区域里
		面，其通过一个LRU变种算法实现。
		
		buffer pool可以优化的几个方面：
			1. 加大buffer pool的大小，buffer pool越大，mysql的行为越趋近于内存数据库；
			2. 64位系统可以管理很大的内存空间，可以将buffer pool分成多个空间，减少统一空间的竞争；
			3. 通过一些方法将常用的数据维持在内存中；？？
			4. 控制预取方式；
			5. 控制脏页刷盘时机，是否允许InnoDB自动调节刷盘频率；
			6. 优化刷盘行为；
			7. 让InnoDB保持buffer pool状态，避免冷启动问题；
		
		LRU算法实现：

		InnoDB buffer pool配置参数：
			1.innodb_buffer_pool_size
				buffer pool的大小（所有instance之和）
			2.innodb_buffer_pool_chunk_size ？
			3.innodb_buffer_pool_instances
				将buffer pool分成指定数量个区域，每一个区域都有独立的LRU链表和相关数据结构，减少
				读写操作对资源的竞争。此操作只有在innodb_buffer_pool_size >= 1G的情况下才会生效
				你所设置的innodb_buffer_pool_size将会在每个instance上均分；（1~64）
			4.innodb_old_blocks_pct
				old block sublist占整个LUR的近似百分比，取值范围是5~95，默认值为37（3/8）;
			5.innodb_old_blocks_time
				当一个页被插入old block sublist后，只有超过innodb_old_blocks_time ms后的访问
				才会导致其被移动到new sublist；
			6.innodb_read_ahead_threshold
				默认值为56，取值范围0~64，在InnoDB线性读取这个多页后，触发线性预取功能；读取整个extent；
			7.innodb_random_read_ahead
				取值ON、OFF，默认OFF；当一个extent中有13个page被访问是，触发随机预取；读取整个extent；
			8.innodb_adaptive_flushing
				是否允许InnoDB根据负载情况自动调节刷页频率；
			9.innodb_adaptive_flushing_lwm
				low water mark;代表redo log超过这个百分比将会刷页；
			10.innodb_flush_neighbors
				是否在刷页的同时也刷其他在the same extent的脏页；
			11.innodb_flushing_avg_loops
				InnoDB保持当前刷页状态的迭代次数；
			12.innodb_lru_scan_depth
				？？
			13.innodb_max_dirty_pages_pct
				如果脏页率达到该值，InnoDB将侵略性的刷脏页；
			14.innodb_max_dirty_pages_pct_lwm
				通过该值控制预刷脏页操作
			15.innodb_buffer_pool_filename
				包含 innodb_buffer_pool_dump_at_shutdown或者innodb_buffer_pool_dump_now
				产生的tablespce IDs和page IDs
			16.innodb_buffer_pool_dump_at_shutdown
				在mysql关闭时InnoDB是否保存innodb buffer pool的状态
			17.innodb_buffer_pool_load_at_startup
				和innodb_buffer_pool_dump_at_shutdown配合使用，在mysql启动时加载上次
				buffer pool的内容；
			18.innodb_buffer_pool_dump_now
				即时保存buffer pool中缓存了那些页；
			19.innodb_buffer_pool_load_now
				不等服务重启，即时装置一个页集；
			20.innodb_buffer_pool_dump_pct
				dump掉的buffer pool百分比；
			21.innodb_buffer_pool_load_abort
				阻断buffer pool load的触发器；一般与innodb_buffer_pool_load_at_startup和
				innodb_buffer_pool_load_now联合使用；
	*/

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