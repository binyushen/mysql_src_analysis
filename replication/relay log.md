# <center>relay log</center>

## Error
错误报告所使用的结构
~~~
class Error {
	uint32 number; //错误码
	char message[MAX_SLAVE_ERRMSG]; //出错信息
	char timestamp[16]; //string格式的出错时间
	time_t skr; //时间戳格式的出错时间

	
}
~~~

## Slave_reporting_capability
通过继承该类，使相关类具有slave reporting的功能
~~~
class Slave_reporting_capability {
	public:
	mutable mysql_mutex_t err_lock; //在show slave status是互斥访问m_last_error

	//通过线程名创建该类，初始化key_mutex_slave_reporting_capability_err_lock
	Slave_reporting_capability(char const *thread_name);
	//in-错误级别，错误码，错误信息，如果是错误信息，会先写到Last_Error中，然后写出去
	virtual void report(loglevel level, int err_code, const char *msg, ...) const 
		MY_ATTRIBUTE((format(printf, 4, 5))); 
	void clear_error()； //清楚错误信息
	//检查当前（thd中的）错误(有多个)是否有temporary nature，silent出口代表该错误是否应该静默，
	//返回0位没有，返回1为有
	int has_temporary_error(THD *thd, uint error_arg= 0, bool* silent= 0) const;
	Error const& last_error() const；
	bool is_error() const；//last_error中是否有错误
	//对于多源复制来说，本应给每一个channel引入错误信息记录，引入一个channel_str来进行标识
	virtual const char* get_for_channel_str(bool upper_case) const = 0;
	virtual ~Slave_reporting_capability()= 0;
	
	protected:
	mutable Error m_last_error; //IO线程或者SQL线程产生的最近一个错误

	//report的真正实现
	virtual void do_report(loglevel level, int err_code,
                 const char *msg, va_list v_args) const
	private:
	char const *const m_thread_name;
	char channel_str[100];
}
~~~

## Rpl_info_values
~~~
	public:
	String *value;
~~~

## Rpl_info_handler

~~~
class Rpl_info_handler
{
	public:
	//在repository写出前存放的位置，或者读取后存放的位置
	Rpl_info_values *field_values;

	protected：
	int ninfo; //在repository中需要存的字段数
	int cursor; //读写开始的位置
	bool prv_error; //标记在处理某个域信息是是否出现错误
	//跟踪fsyncing之前的event数量，--sync-master-info and --sync-relay-log-info
	//标记在fsyncing之前允许处理的event数
	uint sync_counter; 
	uint sync_period; //多少个event之后需要fsync
	
}
~~~

## Rpl_info

~~~
class Rpl_info : public Slave_reporting_capability {
	public:
	/*获取锁的顺序应固定，以避免死锁
	run_lock, data_lock, relay_log.LOCK_log, relay_log.LOCK_index
    run_lock, sleep_lock
    run_lock, info_thd_lock

	info_thd_lock用来保护info_thd上的操作
	在读info_thd前，至少应该获得info_thd_lock、run_lock之一
	在写info_thd前，应该同时获得info_thd_lock、run_lock
	*/
	mysql_mutex_t data_lock, run_lock, sleep_lock, info_thd_lock;
	/*start_cond - SQL线程启动时发出该信号量
	stop_cond - SQL线程停止时发出该信号量
	data_cond - 当被data_lock锁住的数据改变时
	sleep_cond - 当SQL线程被killed时
	*/
	mysql_cond_t data_cond, start_cond, stop_cond, sleep_cond;

	THD *info_thd;
	bool inited;
	volatile bool abort_slave;
	volatile uint slave_running;
	volatile ulong slave_run_id;

	Atomic_int32 is_stopping; //true，该线程正在运行， false，该线程停止

	protect:
	Rpl_info_handler *handler; //指向repository handler
	uint internal_id; //info entry用于多源复制（MTR）
	char channel[CHANNEL_NAME_LENGTH+1]; 


	private:
}
~~~

## MYSQL、st_mysql

~~~

~~~

## Master_info
复制过程中IO线程的抽象  
该类包含：  
1. 连接master的信息；  
2. 从master上读取的当前binlog名  
3. 从master上读取的当前binlog偏移  
4. 其他控制变量  

如果data目录下有master.info文件，该类会根据master.info文件进行初始化;  
如果没有该文件，这个类的各字段通过master-*操作进行设置；  
通过mi_init_info()初始化；  

master.info文件格式为：
log_name  
log_pos  
master_host  
master_user  
master_pass  
master_port  
master_connect_retry  

通过flush_info()写出master.info，当前代码中每次read和queue数据时会调用该函数；  

~~~
class Master_info : public Rpl_info
{
	public:
	char host[HOSTNAME_LENGTH + 1];

	my_bool ssl; //如果为真，connection使用ssl连接
	char ssl_ca[FN_REFLEN], ssl_capath[FN_REFLEN], ssl_cert[FN_REFLEN];
	char ssl_cipher[FN_REFLEN], ssl_key[FN_REFLEN], tls_version[FN_REFLEN];
	char ssl_crl[FN_REFLEN], ssl_crlpath[FN_REFLEN];
	my_bool ssl_verify_server_cert;

	MYSQL* mysql;
	uint32 file_id; //3.23用来加载文件 ？
	Relay_log_info *rli;
	uint port;
	uint connect_retry;
	long clock_diff_with_master; //主从之间时钟相差的秒数
	/*从落后于主的时机计算方法：
	clock_of_slave - last_timestamp_executed_by_SQL_thread - clock_diff_with_master
	*/
	float heartbeat_period; //和change master或master.info交互的周期
	ulonglong received_heartbeats; //接收heartbeat events计数
	time_t last_heartbeat;
	Server_ids *ignore_server_ids;
	ulong master_id;
	/*初始设置成novalue，然后从master上查询@@global.binlog_checksum值进行设置，
	一旦收到FD事件后，这个校验算法就会被失效
	*/
	binary_log::enum_binlog_checksum_alg checksum_alg_before_fd;
	ulong retry_count;
	char master_uuid[UUID_LENGTH+1];
	char bind_addr[HOSTNAME_LENGTH+1]; // ？
	/*用来保存“for channel <channel_name>”*/
	char for_channel_str[CHANNEL_NAME_LENGTH+15];
	char for_channel_uppercase_str[CHANNEL_NAME_LENGTH+15];
	//由于一个事务可能对于多个event，需要一个解析器来判定事务的边界	
	Transaction_boundary_parser transaction_parser; 

	protect:

	private:
	bool start_user_configured; //如果为真，start slave时user/password已被指定
	char user[USERNAME_LENGTH + 1];
	char password[MAX_PASSWORD_LENGTH + 1];
	char start_user[USERNAME_LENGTH + 1]; //start slave时指定的用户名
	char start_password[MAX_PASSWORD_LENGTH + 1];  //start slave时指定的密码
	char start_plugin_auth[FN_REFLEN + 1]; //start slave时指定的插件
	char start_plugin_dir[FN_REFLEN + 1]; //start slave时指定的插件目录

	/*IO线程从master收到的Format_description_log_event，使用方法：
	1. IO线程启动时创建，IO线程停止时销毁
	2. IO线程接收到Format_description_log_event时进行更新
	3. IO线程使用它对接收到的event进行反序列化
	4. 每次rotate时写入新的relay log
	5. 每次flush logs时，写入新的relay log
	*/
	Format_description_log_event *mi_description_event;
	bool auto_position;
	/*上一个queue的GTID，可能并不是当前事务的最后一个event，
	当处理完最后一个event后，会被加入Retrieved_Gtid_Set
	*/
	Gtid last_gtid_queued;
	/*用来同步一个复制channel的管理命令，以下每个操作的过程中必须加锁
	1. START SLAVE;
	2. STOP SLAVE;
	3. CHANGE MASTER;
	4. RESET SLAVE;
	5. end_slave() (when mysqld stops)).
	*/
	Checkable_rwlock *m_channel_lock;
	//这个channel的引用，只有为0时才能被删除
	Atomic_int32 references;
}
~~~

## Relay_log_info--rli

~~~
class Relay_log_info : public Rpl_info {

}
~~~
