# <center>存储引擎接口</center>

## 1、 简介：

	MySQL在存储引擎和优化器之间提供了一个抽象的接口层，使得无论使用哪种存储引擎，都有统一的表访问接口；

## 2、 分析

### 1）、handler

	handler实例是表描述符，一个表可能同时对应多个handler实例；

	class Sql_alloc{}
	// 该类没有成员，重写了new和delete方法，使得内存操作都是在MEM_ROOT上进行的，delete什么也不做，通过free_root来释放空间

	class handler::public Sql_alloc
	{
		TABLE_SHARE *table_share; 
		TABLE *table; //当前打开的表
		Table_flags cached_table_flags; 
		ha_rows estimation_rows_to_insert;
		handlerton *ht;
		uchar *ref; //指向当前行
		uchar *dup_ref; //指向复制的行
		ha_statistics stats; //统计信息
		
		/*
			MultiRangeRead相关变量
		*/
		
		ulonglong next_insert_id; // auto_increment column value
		ulonglong insert_id_for_cur_row;
		Discrete_interval auto_inc_interval_for_cur_row;
		uint auto_inc_intervals_count;
		PSI_table *m_psi;

	}

### 2)、handlerton

	handlerton是一个包含存储引擎回调函数指针的c结构体，随着存储引擎变得越来越复杂，一些功能很难通过handler提供的接口来完成，比如检查点、组提交等，于是MySQL提供了一个标准的回调函数集合来为存储引擎实现这些功能提供入口；每一个存储引擎对应一个handlerton，这是一个全局的结构体。

	struct handlerton {
		uint slot;
		//每一个存储引擎都在thd中有自己用来存放连接信息的的存储空间，指针为thd->ha_data[xxx_hton.slot]
		
	}