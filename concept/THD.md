# <center>THD</center>

## 1. THD

	/*
	线程描述符，主要包括用来处理请求的线程信息；每一个请求对应一个处理线程，每一个线程有一个线程描述符；
	除此之外，MySQL还有很多系统线程；
	
	*/

	//NODE:bootstrap模式下创建THD，但并不创建线程

	class THD: public MDL_context_owner, public Statement, public Open_tables_state {
		MDL_context mdl_context; //元数据锁上下文信息 context of the owner of medadata locks
		Relay_log_info* rli_fake; //用来在服务器上执行base64 coded binlog event
		
		
	}