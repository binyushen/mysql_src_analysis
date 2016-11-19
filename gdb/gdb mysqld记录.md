# <center>gdb mysqld记录</center>

	main()
		mysqld_main()
			my_init() //对my_sys和pthreads库进行初始化 --mysys/my_init.c
				init_global_errs() //初始化全局错误信息，现在该过程为空，全局错误信息在静态变量中初始化 --mysys/errors.c
				my_thread_global_init() //初始化线程环境 --mysys/my_thr_init.c
					//初始化my_fast_mutexattr锁，允许一个线程抢占另外线程的锁，以减少上下文切换，但可能造成某些线程饥饿
					//初始化my_errorcheck_mutexattr锁
					//key_THR_LOCK_malloc THR_LOCK_malloc     --my_init.c
					//key_THR_LOCK_open THR_LOCK_open       --my_init.c
					//key_THR_LOCK_charset THR_LOCK_charset    --my_init.c
					//key_THR_LOCK_threads THR_LOCK_threads    --my_init.c
					my_thread_init() //初始化mysys和debug使用的内存 --mysys/my_thr_init.c
						//分配一个st_my_thread_var类型的内存   
						//锁为THR_LOCK_threads 锁thread_id、THR_thread_count
					//key_THR_LOCK_lock THR_LOCK_lock
					//key_THR_LOCK_myisam THR_LOCK_myisam
					//key_THR_LOCK_myisam_mmap THR_LOCK_myisam_mmap
					//key_THR_LOCK_heap THR_LOCK_heap
					//key_THR_LOCK_net THR_LOCK_net
					//key_THR_COND_threads THR_COND_threads
				safe_mutex_global_init() //初始化锁THR_LOCK_mutex
				load_defaults() //非线程安全
					my_load_defaults() //读配置文件，并将配置文件中的选项放在argv已有选项前
						init_alloc_root() 
						//准备MEM_ROOT以备使用，设置以后内存分配chunk的大小，如果指定，可分配第一个block，这里没有分配
						init_default_directories() 
						//在MEM_ROOT上创建字符串数组，为默认配置文件搜索路径的顺序 /etc/ /etc/mysql/ $basedir/etc "" ~/
						my_search_option_files()
						//处理argc、argv中的选项，顺序寻找配置文件并处理
							get_defaults_options() //从命令行中获得--defaults-file、--defaults-extra-file的参数
							search_default_file_with_ext() //存在--defaults-file，读该文件
			//设置默认编码utf-8
			init_sql_statement_names() //初始化SQL命令的关键字，存入sql_statement_names --mysqld.cc
			sys_var_init() //初始化系统变量hash表system_variable_hash
			init_pfs_instrument_array() //初始化一个动态数组pfs_instr_config_array来存放pfs_instrument的配置信息
			// --storage/perfschema/pfs_server.cc
			handle_early_options() //处理需要提前初始化的选项 --sql/mysqld.cc
			adjust_related_options() //设置相关选项 --sql/mysqld.cc
				adjust_open_files_limit() //获得打开文件数的限制 --sql/mysqld.cc
				adjust_max_connections() //设置最大连接数 --sql/mysqld.cc
				adjust_table_cache_size() //设置table cache大小 --sql/mysqld.cc
				adjust_table_def_size() //? --sql/mysqld.cc
			//设置pfs_parm的m_table_definition_cache、m_table_open_cache、m_max_connections、
			//m_open_files_limit
			initialize_performance_schema() //初始化performance schema存储引擎
			// --storage/perfschema/pfs_server.cc
			//得到pfs的操作接口
			init_server_psi_keys() //注册服务器需要的所有pfs组件 --sql/mysqld.cc
			my_thread_global_reinit() //某些组件（pfs）就绪后需要重新初始化 --mysys/my_thr_init.c
			//重新初始化在my_thread_global_init()中初始化的锁，destroy+create
			init_error_log_mutex() // --sql/mysqld.cc
			mysql_audit_initialize() //初始化审计组件，用于对连接和数据库操作进行审计 --sql/sql_audit.cc
			logger.init_base() // ?
			init_common_variables()
				my_decimal_set_zero()
				tzset()
				rpl_filter= new Rpl_filter
				binlog_filter= new Rpl_filter
				init_thread_environment() //Initialize MySQL global variables to default values
				ignore_db_dirs_init()
				default_storage_engine
				init_default_auth_plugin()
				add_status_vars() //Adds an array of SHOW_VAR entries to the output of SHOW STATUS
				set_server_version()
				//Initialize large page size
				init_errmessage() //从文件中读取error message
				init_client_errs()
				mysql_client_plugin_init()
				lex_init()
				item_create_init()
				item_init()
				my_regex_init()
			my_init_signals()
			init_server_components()
				mdl_init() //Initialize the metadata locking subsystem
				table_def_init()
				hostname_cache_init()
				query_cache_init()
				xid_cache_init()
				delegates_init()
				gtid_server_init()
				plugin_init()
				initialize_storage_engine()
				ft_init_stopwords()
				init_max_user_conn()
				init_update_queries()
			init_server_auto_options()
			network_init()
			start_signal_handler() //Creates pidfile
			mysql_rm_tmp_tables() // --sql/sql_base.cc
			acl_init(bool dont_read_acl_tables) // --sql/sql_acl.cc
			//Initialize structures responsible for user/db-level privilege 
			//checking and load privilege information for them from tables in the 'mysql' database.
			grant_init() // --sql/sql_acl.cc
			servers_init(bool dont_read_servers_table) // --sql/sql_server.cc
			//Initialize structures responsible for servers used in federated
			//server scheme information for them from the server
			//table in the 'mysql' database.
			udf_init() // --sql/sql_udf.cc
			//Read all predeclared functions from mysql.func and accept all that can be used.
			init_status_vars();
			binlog_unsafe_map_init() // --sql/sql_lex.cc
			set_slave_skip_errors(char** slave_skip_errors_ptr) // --sql/rpl_slave.cc
			init_slave() // --sql/rpl_slave.cc
			//Initialize slave structures
			initialize_performance_schema_acl() // --storage/perfschema/pfs_engine_schema.cc
			check_performance_schema() // --storage/perfschema/pfs_check.cc
			//Check that the performance schema tables have the expected structure
			initialize_information_schema_acl() // --sql/sql_show.cc
			execute_ddl_log_recovery() // --sql/sql_table.cc
			Events::init() //？
			create_shutdown_thread() //入口handle_shutdown
			start_handle_manager() //入口handle_manager --sql/sql_manager.cc
			handle_connections_sockets() 
			//Handle new connections and spawn new process to handle them
				create_new_thread(THD *thd)
				//Create new thread to handle incoming connection
					create_thread_to_handle_connection()
					//线程入口handle_one_connection
			Service.Stop()
			clean_up(1)
			mysqld_exit(0)