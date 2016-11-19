# <center>hash</center>

## 1. 

	typedef struct st_dynamic_array
	{
	 	uchar *buffer;
 		uint elements,max_element;
		uint alloc_increment;
		uint size_of_element;
	} DYNAMIC_ARRAY;
	//动态数组

	typedef struct st_hash {
  		size_t key_offset,key_length；
  		size_t blength;
  		ulong records;
  		uint flags;
  		DYNAMIC_ARRAY array;
  		my_hash_get_key get_key;
		//hash值计算函数
  		void (*free)(void *);
  		CHARSET_INFO *charset;
  		my_hash_function hash_function;
		//hash值计算函数
	} HASH;