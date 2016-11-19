# <center>gdb innoDB初始化过程</center>

	plugin_initialize(struct st_plugin_int *plugin) <--innobase的plugin传入
		ha_initialize_handlerton(struct st_plugin_init *plugin)
		//通过plugin->plugin->type在plugin_type_initialize数组中寻找到ha_initialize_handlerton函数
		//入口进行全局handlerton初始化
			//分配handlerton的空间，创建实例
			innobase_init(void *p) 
			//p指向handlerton
			//通过plugin->plugin->init寻找到innobase_init函数进行innodb的handlerton的初始化
				//handlerton各回调函数指针初始化
				//innobase各运行参数初始化 innobase_XXXXX -> srv_XXXXX
				innobase_start_or_create_for_mysql();
				//启动innobase
					srv_boot(); //boots InnoDB server
						srv_normalize_init_values(); //规范化初始参数
						srv_general_init(); //初始化同步使用的变量，内存系统，和线程本地存储空间
							ut_mem_init(); // ?
							recv_sys_var_init(); //重置恢复系统变量
							os_sync_init(); //初始化全局事件和 ？
							sync_init(); //初始化同步互斥变量结构
							mem_init(ulint size); //初始化内存系统 <- 内存池大小
								//如果srv_use_sys_malloc被设置成TRUE，mem_comm_pool不会被用来做内存
								//分配，但这里需要创建一个假的mem_comm_pool，因为一些统计和调试代码需
								//要他被初始化
								mem_comm_pool = mem_pool_create(ulint size); // <- 1
							que_init(); // ?
							row_mysql_init(); // ?
						srv_init(); //服务器初始化
					buf_pool_init(ulint total_size, ulint n_instances);
					//创建buffer pool <- 128M 1
						//创建一个有n_instances个buf_pool_t*的数组，逐个初始化
						buf_pool_init_instance(buf_pool_t* buf_pool, ulint buf_pool_size, ulint instance_no); 
						//初始化buffer pool instance <- * 134217728 0
							buf_chunk_init(buf_pool_t *buf_pool, buf_chunk_t *chunk, ulint mem_size); 
							//初始化buffer chunk
						buf_pool_set_sizes(); //在改变大小后，设置buffer pool大小
						buf_LRU_old_ratio_update();
						btr_search_sys_create(); // ?
					fsp_init(); // Initializes the fsp system
					log_init(void); //Initializes the log

					/*启动IO线程 io_handler_thread(void *) 默认10个，和配置参数有关*/

					recv_sys_create(); //创建恢复系统
					recv_sys_init(); //初始化恢复系统
					open_or_create_data_files(); //创建或者打开数据库文件  创建12M的ibdata1文件
					create_log_files(); //创建日志文件，默认两个48M的ib_logfile1、ib_logfile101
					fil_open_log_and_system_tablespace_files(); //打开日志文件和系统表空间文件，直到数据库关闭
					srv_undo_tablespaces_init(); //打开指定个数的undo表空间
					dict_stats_thread_init(); //初始化dict_stats_thread()需要的全局变量
					trx_sys_file_format_init(); //Initializes the tablespace tag system.
					trx_sys_create(); //Creates the trx_sys instance and initializes ib_bh and mutex. 
					
					//创建线程lock_wait_timeout_thread
					//创建线程srv_error_monitor_thread
					//创建线程srv_monitor_thread

					//创建线程srv_master_thread
					
					//创建线程srv_purge_coordinator_thread

					//创建线程buf_flush_page_cleaner_thread

					//创建线程buf_dump_thread
				
					//创建线程dict_stats_thread

					//创建线程fts_optimize_thread

					

					