# <center>bin log</center>

## Log_event_header

在同一版的MySQL中，binlog的头部一般拥有相同的格式和长度，每一种类型的event指定不同长度的post-header。

~~~
class Log_event_header {

	struct timeval when; //四字节无符号整形，该查询开始执行的时间戳
	Log_event_type  type_code; //一字节枚举类型，该event的类型
	unsigned int unmasked_server_id; //四字节无符号整形， server ID
	size_t data_written; //四字节无符号整形，整个event的长度，Common-Header+Post-Header+Body
	unsigned long long log_pos; //下一个event在binlog中的位置，在relay log中，该值为master binlog中下一个event的位置
	uint16_t flags;

	//上边所述总共有19B，为common header

	explicit Log_event_header(Log_event_type type_code_arg= ENUM_END_EVENT)；
	Log_event_header(const char* buf, uint16_t binlog_version); //在IO线程和SQL线程中使用
	~Log_event_header() {}
}
~~~

## Log_event_footer

在当前MySQL版本中，该类只包含一个用来指定进行校验的算法

~~~
class class Log_event_footer {
	enum_binlog_checksum_alg checksum_alg; //计算校验和所采用的算法描述符
}
~~~

## Log_event

Log_event是二进制日志中事件的顶层抽象类

~~~
class Log_event {

	bool is_valid_param; // ?
	char *temp_buf; //read_log_event使用的临时缓冲区
	ulong exec_time; //在master上该query执行的时间
	uint32 server_id; //master的server ID
	ulong rbr_exec_mode; // ?
	enum_event_cache_type event_cache_type; //定义缓冲类型，即在刷盘前，这个event会存在哪里
	enum_event_logging_type event_logging_type; //定义刷盘的时机
	ha_checksum crc; //校验和占位符
	ulong mts_group_idx; //rli->gaq数组的索引
	binary_log::Log_event_header *common_header; //包含event的功用头部变量
	binary_log::Log_event_footer *common_footer; //指定校验算法
	Relay_log_info *worker; // 将该event关联到一个worker
	ulonglong future_event_relay_log_pos; // ?
	THD* thd;
	db_worker_hash_entry *mts_assigned_partitions[MAX_DBS_IN_EVENT_MTS]; //

	//ifndef ----MYSQL_CLIENT----
	/*
	
	*/
	Log_event::Log_event(THD* thd_arg, uint16 flags_arg,
	                     enum_event_cache_type cache_type_arg,
	                     enum_event_logging_type logging_type_arg,
	                     Log_event_header *header, Log_event_footer *footer)
	  : is_valid_param(false), temp_buf(0), exec_time(0),
	    event_cache_type(cache_type_arg), event_logging_type(logging_type_arg),
	    crc(0), common_header(header), common_footer(footer), thd(thd_arg)
	/*
	
	*/
	Log_event(Log_event_header* header, Log_event_footer *footer,
                     enum_event_cache_type cache_type_arg,
                     enum_event_logging_type logging_type_arg)
  	: is_valid_param(false), temp_buf(0), exec_time(0), event_cache_type(cache_type_arg),
   	event_logging_type(logging_type_arg), crc(0), common_header(header),
   	common_footer(footer), thd(0) { ... }

	//#else
	Log_event::Log_event(Log_event_header *header,
	                     Log_event_footer *footer)
	  : is_valid_param(false), temp_buf(0), exec_time(0),
	    event_cache_type(EVENT_INVALID_CACHE),
	    event_logging_type(EVENT_INVALID_LOGGING),
	    crc(0), common_header(header), common_footer(footer)
}
~~~

## IO_CACHE

文件缓冲读取的结构体

~~~

typedef struct st_io_cache {
	my_off_t pos_in_file; //uchar *buffer第一个字节对应的文件偏移位置
	my_off_t end_of_file; //?
	uchar	*read_pos; //当前读取的位置指针
	uchar  *read_end; //有消读取的边界，不包含该位置
	uchar  *buffer; //读缓冲区
	uchar  *request_pos; // ?
	uchar  *write_buffer; //写缓冲区
	uchar *append_read_pos; //在SEQ_READ_APPEND模式下指向write buffer中当前读的位置，
	uchar *write_pos; //指向当前写的位置
	uchar *write_end; //指向当前写的边界，不包含该位置
	uchar  **current_pos, **current_end; //为使用my_b_tell()等函数方便引入的两个变量
	mysql_mutex_t append_buffer_lock; //SEQ_READ_APPEND模式下拷贝append buffer到read buffer需要互斥
	IO_CACHE_SHARE *share; //多线程读取同一个文件需要同步的变量
	int (*read_function)(struct st_io_cache *,uchar *,size_t); //
	int (*write_function)(struct st_io_cache *,const uchar *,size_t);
	enum cache_type type; //读写缓冲区类型
	IO_CACHE_CALLBACK pre_read; //
	IO_CACHE_CALLBACK post_read;
	IO_CACHE_CALLBACK pre_close;
	ulong disk_writes; //强制使用磁盘计数
	void* arg; //pre/post_read使用
	char *file_name;
	char *dir,*prefix;
	File file; 
	PSI_file_key file_key;
	int	seek_not_done,error; //my_b_seek()执行是否成功
	size_t	buffer_length; //
	size_t  read_length;
	myf	myflags;
	my_bool alloced_buffer; //1 通过init_io_cache得到， 0 用户提供
} IO_CACHE;
~~~


## Binlog_sender

dump线程的实现，根据请求，返回event

~~~
class Binlog_sender {

	Binlog_sender(THD *thd, const char *start_file, my_off_t start_pos,
                Gtid_set *exclude_gtids, uint32 flag);
	

	THD *m_thd;
	String& m_packet; //缓冲区
	
	const char *m_start_file; //请求要开始的文件名
	my_off_t m_start_pos; //请求要开始的文件位置

	/*对于COM_BINLOG_DUMP_GTID，这里包含一个GTID SET，在该集合中的event被忽略，不发送出去*/
	Gtid_set *m_exclude_gtid;
	bool m_using_gtid_protocol;
	bool m_check_previous_gtid_event;
	bool m_gtid_clear_fd_created_flag;

	LOG_INFO m_linfo; //当前正在读取的binlog
	
	/*主从分别使用的校验算法*/
	binary_log::enum_binlog_checksum_alg m_event_checksum_alg;
	binary_log::enum_binlog_checksum_alg m_slave_checksum_alg;

	/*心跳相关*/
	ulonglong m_heartbeat_period;
	time_t m_last_event_sent_ts;

	bool m_wait_new_events; //是否等待新的event，在mysqlbinlog中，读完最后一个event后就会理解停止

	Diagnostics_area m_diag_area; //语句执行诊断区
	char m_errmsg_buf[MYSQL_ERRMSG_SIZE];
	const char *m_errmsg;
	int m_errno;

	/*最近读取的上一个事件的相关信息，用于出错处理*/
	char m_last_file_buf[FN_REFLEN];
	const char *m_last_file;
	my_off_t m_last_pos;

	ushort m_half_buffer_size_req_counter; //？ 当buffer需要调整是可以被评估
	size_t m_new_shrink_size; //每次buffer调整，更新该值
	
	const static uint32 PACKET_MAX_SIZE;
	const static ushort PACKET_SHRINK_COUNTER_THRESHOLD;
	const static uint32 PACKET_MIN_SIZE;
	const static float PACKET_GROW_FACTOR;
	const static float PACKET_SHRINK_FACTOR;
	
	uint32 m_flag;
	bool m_observe_transmission;
	bool m_transmit_started;
}

~~~