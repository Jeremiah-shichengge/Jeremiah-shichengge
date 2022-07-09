DAOS 源码解析之 daos_array
 
尚先生的博客 于 2022-05-27 14:35:29 发布 29  收藏
分类专栏： DAOS 文章标签： 云计算
版权
 DAOS专栏收录该内容
6 篇文章0 订阅
订阅专栏
【摘要】 DAOS (Distributed Asynchronous Object Storage) 是一个开源的对象存储系统，专为大规模分布式非易失性内存设计，利用了 SCM 和 NVMe 等的下一代 NVM 技术。 DAOS 同时在硬件之上提供了键值存储接口，提供了诸如事务性非阻塞 I/O、具有自我修复的高级数据保护、端到端数据完整性、细粒度数据控制和弹性存储的高级数据保护，从而优化性能并降低成本。
本文以 Release 1.1.4 版本为标准，解析 include/daos_array.h 中的具体实现。include/daos_array.h 包含了 DAOS 与数组相关的 API ，提供 DAOS 对象数据模型上的一维数组实现。
daos_range_t
结构图 daos_range_t 表示连续记录的区间：
1.	typedef struct {
2.		// 区间中第一条记录的索引
3.		daos_off_t		rg_idx;
4.		// 区间中记录的数量
5.		daos_size_t		rg_len;
6.	} daos_range_t;
daos_array_iod_t
结构体 daos_array_iod_t 表示要访问的 DAOS 数组对象区间的 IO 描述符：
1.	typedef struct {
2.		// arr_rgs 中数据项的数量
3.		daos_size_t		arr_nr;
4.		// 数组区间：每个区间定义一个起始索引和一个区间长度。
5.		daos_range_t	*arr_rgs;
6.	  
7.		// 只读情况：
8.		// 返回从最大的 dkey (distribution key) 中 short fetch 的记录数。
9.		// 这有助于检查 short read。
10.		// 如果该值不为零，则可以进行 short read，并应使用 
11.		// daos_array_get_size() 函数与正在读取的索引进行比较。
12.		daos_size_t		arr_nr_short_read;
13.		daos_size_t		arr_nr_read;
14.	} daos_array_iod_t;
daos_array_generate_id
daos_array_generate_id 函数是一个辅助函数，通过编码对象地址空间的 DAOS 保留位（高 32 位）来生成 DAOS 对象 ID。
参数：
•	old [in, out]： 
•	[in]：设置了低 96 位且在容器内唯一的对象 ID。
•	[out]：完全填充的 DAOS 对象标识符，低 96 位不变，编码 DAOS 保留位（高32位）。
•	cid [in]：类标识符。
•	add_attr [in]：指示用户操作的标志 
•	true：元数据应存储在对象中
•	false：维护数组单元格和块大小。
•	args [in]：预留参数。
返回值：
•	始终为 0。
1.	static inline int
2.	daos_array_generate_id(daos_obj_id_t *oid, daos_oclass_id_t cid, 
3.	                       bool add_attr, uint32_t args)
4.	{
5.		static daos_ofeat_t feat;
6.		uint64_t            hdr;
7.	 
8.		// DAOS_OF_DKEY_UINT64：
9.		// 		未进行散列和数字排序的 DKEY key
10.		// 		key 按客户端的字节顺序接受，DAOS 负责操作正确
11.		// DAOS_OF_KV_FLAT  预留：1-level flat KV store
12.		feat = DAOS_OF_DKEY_UINT64 | DAOS_OF_KV_FLAT;
13.		
14.		// 判断用户操作标志
15.		if (add_attr)
16.			// DAOS_OF_ARRAY  预留：多维数组
17.			feat = feat | DAOS_OF_ARRAY;
18.	 
19.		// TODO：此处应增加检查，如果用户修改了 DAOS 保留位，则返回 error
20.		// OID_FMT_INTR_BITS：DAOS 保留位 (32 bits)
21.		// 		版本 (OID_FMT_VER_BITS, 4 bits)
22.		//		特征 (OID_FMT_FEAT_BITS, 16 bits)
23.		//		类 ID (OID_FMT_CLASS_BITS, 12 bits)
24.		oid->hi &= (1ULL << OID_FMT_INTR_BITS) - 1;
25.	  
26.		// OID_FMT_VER：当前对象 ID 的版本
27.		// OID_FMT_VER_SHIFT：对象 ID 中版本位的相对位移
28.		hdr = ((uint64_t)OID_FMT_VER << OID_FMT_VER_SHIFT);
29.		// OID_FMT_FEAT_SHIFT：对象 ID 中特征位的相对位移
30.		hdr |= ((uint64_t)feat << OID_FMT_FEAT_SHIFT);
31.		// OID_FMT_CLASS_SHIFT：对象 ID 中类 ID 位的相对位移
32.		hdr |= ((uint64_t)cid << OID_FMT_CLASS_SHIFT);
33.		oid->hi |= hdr;
34.	 
35.		return 0;
36.	}
结构体 daos_obj_id_t 表示对象 ID，共 128 位。有关 daos_obj_id_t 的更多操作细节将在 include/daos_obj.h 中解析。
1.	typedef struct {
2.		// 高 32 位是 DAOS 的保留位
3.		// 低 96 位由用户提供，并假定在容器中是唯一的
4.		uint64_t	lo;
5.		uint64_t	hi;
6.	} daos_obj_id_t;
daos_oclass_id_t 类型用来表示对象类 ID：
typedef uint16_t	daos_oclass_id_t;
daos_ofeat_t 类型用来表示对象的特征位：
typedef uint16_t	daos_ofeat_t;
daos_array_create
daos_array_create 函数用于创建数组对象。这将打开一个 DAOS KV 对象并添加元数据来定义单元大小和块大小。进一步访问 KV 对象需要通过句柄，这需要使用元数据存储数组元素。
数组的元数据存储在 DKEY 0 (distribution key) 的特定 AKEY (attribute key) 下。这意味着该对象是一个通用数组对象，它的元数据在 DAOS 对象中被跟踪。oid 中的特征位必须设置为
DAOS_OF_DKEY_UINT64 | DAOS_OF_KV_FLAT | DAOS_OF_ARRAY
如果特征位没有设置 DAOS_OF_ARRAY，则需要由用户来负责记住数组元数据，因为 DAOS 不会存储这些元数据，并且不应该调用此 API，因为该 API 不会向数组对象写入任何内容。在这种情况下，可以使用 daos_array_open_with_attrs() 获取数组对象，并通过数组 API 进行访问。
元数据只是 KV 对象中的项，这意味着任何用户都可以打开该对象并覆盖该元数据。用户可以重新创建数组，这不会破坏现有的原始数据，只会覆盖元数据。但是，更改元数据将导致未定义的访问问题（在这种情况下，我们可以通过读取元数据来检查对象是否存在，从而发现错误，但这会增加额外的开销）。
参数：
•	coh [in]：打开的容器句柄。
•	oid [in]：对象 ID，要求 dkey 的特征位必须设置为 DAOS_OF_KV_FLAT | DAOS_OF_DKEY_UINT64 |DAOS_OF_ARRAY。
•	th [in]：事务句柄。
•	cell_size [in]：数组的单元大小。
•	chunk_size [in]：在移动到其它 dkey 前每个 dkey 下存储的连续记录的数量。
•	oh [out]：返回的打开的数组对象句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的容器句柄。
•	-DER_INVAL：无效的参数。
•	-DER_NO_PERM：没有访问权限。
•	-DER_UNREACH：无法访问网络。
1.	int
2.	daos_array_create(daos_handle_t coh, daos_obj_id_t oid, daos_handle_t th,
3.			  daos_size_t cell_size, daos_size_t chunk_size,
4.			  daos_handle_t *oh, daos_event_t *ev)
5.	{
6.		daos_array_create_t *args;
7.		tse_task_t *         task;
8.		int                  rc;
9.	 
10.		// 创建新任务 dc_array_create，并将其与输入事件 ev 关联
11.		// 如果事件 ev 为 NULL，则将获取私有事件
12.		rc = dc_task_create(dc_array_create, NULL, ev, &task);
13.		if (rc)
14.			// dc_task_create 成功返回 0，失败返回负数
15.			return rc;
16.	 
17.		// 从 task 中获取参数
18.		args             = dc_task_get_args(task);
19.		args->coh        = coh;
20.		args->oid        = oid;
21.		args->th         = th;
22.		args->cell_size  = cell_size;
23.		args->chunk_size = chunk_size;
24.		args->oh         = oh;
25.	 
26.		// 调度创建的任务 task
27.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
28.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
29.		//
30.		// 第二个参数 instant 为 true，表示任务将立即执行
31.		return dc_task_schedule(task, true);
32.	}
daos_size_t 是 64 位无符号整数：
typedef uint64_t	daos_size_t;
结构体 daos_array_create_t 表示数组创建参数：
1.	typedef struct {
2.		//打开的容器句柄
3.		daos_handle_t	coh;
4.		// 数组 ID
5.		daos_obj_id_t	oid;
6.		// 打开的事务句柄
7.		daos_handle_t	th;
8.		// 数组的单元大小
9.		daos_size_t		cell_size;
10.		// 1 个 dkey 下存储的记录数量
11.		daos_size_t		chunk_size;
12.		// 返回的打开的数组句柄
13.		daos_handle_t	*oh;
14.	} daos_array_create_t;
daos_array_open
daos_array_open 函数打开一个数组对象。如果该数组之前没有被创建过（不存在数组元数据），该操作会失败。
参数：
•	coh [in]：打开的容器句柄。
•	oid [in]：对象 ID，要求 dkey 的特征位必须设置为 DAOS_OF_KV_FLAT | DAOS_OF_DKEY_UINT64 |DAOS_OF_ARRAY。
•	th [in]：事务句柄。
•	mode [in]：打开模式：DAOS_OO_RO 或 DAOS_OO_RW。
•	cell_size [out]：数组的单元大小。
•	chunk_size [out]：在移动到其它 dkey 前每个 dkey 下存储的连续记录的数量。
•	oh [out]：返回的打开的数组对象句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的容器句柄。
•	-DER_INVAL：无效的参数。
•	-DER_NO_PERM：没有访问权限。
•	-DER_NONEXIST：无法找到对象。
•	-DER_UNREACH：无法访问网络。
1.	int
2.	daos_array_open(daos_handle_t coh, daos_obj_id_t oid, daos_handle_t th,
3.			    unsigned int mode, daos_size_t *cell_size,
4.			    daos_size_t *chunk_size, daos_handle_t *oh,
5.			    daos_event_t *ev)
6.	{
7.		daos_array_open_t *args;
8.		tse_task_t *       task;
9.		int                rc;
10.	 
11.		// 创建新任务 dc_array_open，并将其与输入事件 ev 关联
12.		// 如果事件 ev 为 NULL，则将获取私有事件
13.		rc = dc_task_create(dc_array_open, NULL, ev, &task);
14.		if (rc)
15.			// dc_task_create 成功返回 0，失败返回负数
16.			return rc;
17.	 
18.		// 初始状态下数组的单元大小和块大小均为 0
19.		*cell_size  = 0;
20.		*chunk_size = 0;
21.	 
22.		// 从 task 中获取参数
23.		args                 = dc_task_get_args(task);
24.		args->coh            = coh;
25.		args->oid            = oid;
26.		args->th             = th;
27.		args->mode           = mode;
28.		args->open_with_attr = 0;
29.		args->cell_size      = cell_size;
30.		args->chunk_size     = chunk_size;
31.		args->oh             = oh;
32.	 
33.		// 调度创建的任务 task
34.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
35.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
36.		//
37.		// 第二个参数 instant 为 true，表示任务将立即执行
38.		return dc_task_schedule(task, true);
39.	}
结构体 daos_array_open_t 表示数组打开参数：
1.	typedef struct {
2.		// 打开的容器句柄
3.		daos_handle_t	coh;
4.		// 数组 ID
5.		daos_obj_id_t	oid;
6.		// 打开的事务句柄
7.		daos_handle_t	th;
8.		// 打开模式
9.		unsigned int	mode;
10.		// 标记单元大小和块大小是否由用户提供
11.		// 如果为 1，表示由用户提供，否则为 0
12.		unsigned int	open_with_attr;
13.		// 数组的单元大小
14.		daos_size_t		*cell_size;
15.		// 1 个 dkey 下存储的记录数量
16.		daos_size_t		*chunk_size;
17.		// 返回的打开的数组句柄
18.		daos_handle_t	*oh;
19.	} daos_array_open_t;
daos_array_open_with_attr
daos_array_open_with_attr 函数打开具有用户指定属性的数组对象。如果对象不存在，这与调用 daos_array_create 的结果相同，只是对象中没有任何更新，该 API 只是向用户返回一个打开的数组句柄。如果以前使用不同的单元大小和块大小访问数组，再次访问它将导致数组数据损坏。
参数：
•	coh [in]：打开的容器句柄。
•	oid [in]：对象 ID，要求 dkey 的特征位必须设置为 DAOS_OF_KV_FLAT | DAOS_OF_DKEY_UINT64 |DAOS_OF_ARRAY。
•	th [in]：事务句柄。
•	mode [in]：打开模式：DAOS_OO_RO 或 DAOS_OO_RW。
•	cell_size [in]：数组的单元大小。
•	chunk_size [in]：在移动到其它 dkey 前每个 dkey 下存储的连续记录的数量。
•	oh [out]：返回的打开的数组对象句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的容器句柄。
•	-DER_INVAL：无效的参数。
•	-DER_NO_PERM：没有访问权限。
1.	int
2.	daos_array_open_with_attr(daos_handle_t coh, daos_obj_id_t oid,
3.				  daos_handle_t th, unsigned int mode,
4.				  daos_size_t cell_size, daos_size_t chunk_size,
5.				  daos_handle_t *oh, daos_event_t *ev)
6.	{
7.		daos_array_open_t *args;
8.		tse_task_t *       task;
9.		int                rc;
10.	 
11.		// 创建新任务 dc_array_open，并将其与输入事件 ev 关联
12.		// 如果事件 ev 为 NULL，则将获取私有事件
13.		rc = dc_task_create(dc_array_open, NULL, ev, &task);
14.		if (rc)
15.			// dc_task_create 成功返回 0，失败返回负数
16.			return rc;
17.	 
18.		// 从 task 中获取参数
19.		args                 = dc_task_get_args(task);
20.		args->coh            = coh;
21.		args->oid            = oid;
22.		args->th             = th;
23.		args->mode           = mode;
24.		// 单元大小和块大小由用户提供
25.		args->open_with_attr = 1;
26.		args->cell_size      = &cell_size;
27.		args->chunk_size     = &chunk_size;
28.		args->oh             = oh;
29.	 
30.		// 调度创建的任务 task
31.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
32.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
33.		//
34.		// 第二个参数 instant 为 true，表示任务将立即执行
35.		return dc_task_schedule(task, true);
36.	}
daos_array_local2global
daos_array_local2global 函数将本地数组句柄转换为可与对等进程共享的全局表示数据。
如果 glob->iov_buf 被设置为NULL，则通过 glob->iov_buf_len 返回全局句柄的实际大小。
此功能不涉及任何通信，也不阻塞。
参数：
•	oh [in]：将被共享的有效的本地数组对象的打开句柄。
•	glob [out]：指向 IO vector 缓冲区的指针，用于存储句柄信息。
返回值，在非阻塞模式下：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_INVAL：无效的参数。
•	-DER_NO_HDL：数组句柄不存在。
•	-DER_TRUNC：glob 中的缓冲区过小，要求更大的缓冲区。在这种情况下，要求的缓冲区大学会被写入 glob->iov_buf_len。
1.	int
2.	daos_array_local2global(daos_handle_t oh, d_iov_t *glob)
3.	{
4.		return dc_array_local2global(oh, glob);
5.	}
daos_array_global2local
daos_array_global2local 函数为全局表示数据创建本地数组打开句柄。
必须使用 daos_array_close() 关闭此句柄。
参数：
•	oh [in]：将被共享的有效的本地数组对象的打开句柄。
•	glob [in]：要提取的集合句柄的全局（共享）表示。
•	mode [in]：改变对象的打开模式（可选）。传递 0 将继承全剧表示的打开模式。
•	oh [out]：返回的本地数组打开句柄。
返回值，在非阻塞模式下：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_INVAL：无效的参数。
•	-DER_NO_HDL：数组句柄不存在。
1.	int
2.	daos_array_global2local(daos_handle_t coh, d_iov_t glob, unsigned int mode,
3.				daos_handle_t *oh)
4.	{
5.		return dc_array_global2local(coh, glob, mode, oh);
6.	}
daos_array_close
daos_array_close 函数关闭一个打开的数组对象。
参数：
•	oh [in]：打开的数组对象句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的打开句柄。
1.	int
2.	daos_array_close(daos_handle_t oh, daos_event_t *ev)
3.	{
4.		daos_array_close_t *args;
5.		tse_task_t *        task;
6.		int                 rc;
7.	 
8.		// 创建新任务 dc_array_close，并将其与输入事件 ev 关联
9.		// 如果事件 ev 为 NULL，则将获取私有事件
10.		rc = dc_task_create(dc_array_close, NULL, ev, &task);
11.		if (rc)
12.			// dc_task_create 成功返回 0，失败返回负数
13.			return rc;
14.	 
15.		// 从 task 中获取参数
16.		args     = dc_task_get_args(task);
17.		args->oh = oh;
18.	 
19.		// 调度创建的任务 task
20.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
21.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
22.		//
23.		// 第二个参数 instant 为 true，表示任务将立即执行
24.		return dc_task_schedule(task, true);
25.	}
结构体 daos_array_close_t 表示数组关闭参数：
1.	typedef struct {
2.		// 打开的数组句柄
3.		daos_handle_t		oh;
4.	} daos_array_close_t;
daos_array_read
daos_array_read 函数从数组对象中读取数据。
参数：
•	oh [in]：打开的数组对象句柄。
•	th [in]：事务句柄。
•	iod [in]：要读取的数组区间的 IO 描述符。
•	sgl [in]：存储数组数据的分散/聚集列表（sgl）。缓冲区大小不必与单个区间大小匹配，只要总大小匹配即可。用户需要分配缓冲区并设置每个缓冲区的长度。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的打开句柄。
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
•	-DER_REC2BIG：记录太大，无法放入输出缓冲区。
1.	int
2.	daos_array_read(daos_handle_t oh, daos_handle_t th, daos_array_iod_t *iod,
3.			d_sg_list_t *sgl, daos_event_t *ev)
4.	{
5.		daos_array_io_t *args;
6.		tse_task_t *     task;
7.		int              rc;
8.	 
9.		// 创建新任务 dc_array_read，并将其与输入事件 ev 关联
10.		// 如果事件 ev 为 NULL，则将获取私有事件
11.		rc = dc_task_create(dc_array_read, NULL, ev, &task);
12.		if (rc)
13.			// dc_task_create 成功返回 0，失败返回负数
14.			return rc;
15.	 
16.		// 从 task 中获取参数
17.		args      = dc_task_get_args(task);
18.		args->oh  = oh;
19.		args->th  = th;
20.		args->iod = iod;
21.		args->sgl = sgl;
22.	 
23.		// 调度创建的任务 task
24.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
25.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
26.		//
27.		// 第二个参数 instant 为 true，表示任务将立即执行
28.		return dc_task_schedule(task, true);
29.	}
结构体 d_sg_list_t 表示内存缓冲区的分散/聚集列表：
1.	typedef struct {
2.		// I/O vector 总数
3.		uint32_t	sg_nr;
4.		// 已使用的 I/O vector 数量
5.		uint32_t	sg_nr_out;
6.		// 存储 I/O vector 的数组
7.		d_iov_t		*sg_iovs;
8.	} d_sg_list_t;
I/O vector 是与 readv 和 wirtev 操作相关的结构体。readv 和 writev 函数用于在一次原子操作中读、写多个非连续缓冲区。有时也将这两个函数称为分散读 (scatter read) 和聚集写 (gather write)。
结构体 daos_array_io_t 表示数组读/写参数：
1.	typedef struct {
2.		// 打开的数组句柄
3.		daos_handle_t		oh;
4.		// 打开的事务句柄
5.		daos_handle_t		th;
6.		// 数组 IO 描述符
7.		daos_array_iod_t	*iod;
8.		// 内存描述符
9.		d_sg_list_t			*sgl;
10.	} daos_array_io_t;
adaos_array_write
daos_array_write 函数向数组对象写入数据。
参数：
•	oh [in]：打开的数组对象句柄。
•	th [in]：事务句柄。
•	iod [in]：要写入的数组区间的 IO 描述符。
•	sgl [in]：存储数组数据的分散/聚集列表（sgl）。缓冲区大小不必与单个区间大小匹配，只要总大小匹配即可。用户需要分配缓冲区并设置每个缓冲区的长度。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的打开句柄。
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
•	-DER_REC2BIG：记录太大，无法放入输出缓冲区。
1.	int
2.	daos_array_write(daos_handle_t oh, daos_handle_t th, daos_array_iod_t *iod,
3.			 d_sg_list_t *sgl, daos_event_t *ev)
4.	{
5.		daos_array_io_t *args;
6.		tse_task_t *     task;
7.		int              rc;
8.	 
9.		// 创建新任务 dc_array_write，并将其与输入事件 ev 关联
10.		// 如果事件 ev 为 NULL，则将获取私有事件
11.		rc = dc_task_create(dc_array_write, NULL, ev, &task);
12.		if (rc)
13.			// dc_task_create 成功返回 0，失败返回负数
14.			return rc;
15.	 
16.		// 从 task 中获取参数
17.		args      = dc_task_get_args(task);
18.		args->oh  = oh;
19.		args->th  = th;
20.		args->iod = iod;
21.		args->sgl = sgl;
22.	 
23.		// 调度创建的任务 task
24.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
25.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
26.		//
27.		// 第二个参数 instant 为 true，表示任务将立即执行
28.		return dc_task_schedule(task, true);
29.	}
daos_array_get_size
daos_array_get_size 函数查询数组对象中的记录数量。
参数：
•	oh [in]：打开的数组对象句柄。
•	th [in]：事务句柄。
•	size [out]：返回的数组大小（记录的数量）。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的打开句柄。
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
1.	int
2.	daos_array_get_size(daos_handle_t oh, daos_handle_t th, daos_size_t *size,
3.			    daos_event_t *ev)
4.	{
5.		daos_array_get_size_t *args;
6.		tse_task_t *           task;
7.		int                    rc;
8.	 
9.		// 创建新任务 dc_array_get_size，并将其与输入事件 ev 关联
10.		// 如果事件 ev 为 NULL，则将获取私有事件
11.		rc = dc_task_create(dc_array_get_size, NULL, ev, &task);
12.		if (rc)
13.			// dc_task_create 成功返回 0，失败返回负数
14.			return rc;
15.	 
16.		// 从 task 中获取参数
17.		args       = dc_task_get_args(task);
18.		args->oh   = oh;
19.		args->th   = th;
20.		args->size = size;
21.	  
22.		// 调度创建的任务 task
23.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
24.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
25.		//
26.		// 第二个参数 instant 为 true，表示任务将立即执行
27.		return dc_task_schedule(task, true);
28.	}
daos_size_t 类型是 64 位无符号整数：
typedef uint64_t	daos_size_t;
结构体 daos_array_get_size_t 表示获取数组大小的参数：
1.	typedef struct {
2.		// 打开的数组句柄
3.		daos_handle_t	oh;
4.		// 打开的事务句柄
5.		daos_handle_t	th;
6.		// 返回的数组大小（记录数）
7.		daos_size_t		*size;
8.	} daos_array_get_size_t;
daos_array_set_size
daos_array_set_size 函数在记录中设置数组大小（缩短）。
如果缩小数组，我们将按要求大小对 dkey/记录进行打孔 (punch) （“punch” 原意为打孔，这里的 punch 操作我个人理解是类似于截断，在原大小范围内制造一个空洞，清除所有数据，然后将空洞的前后拼接在一起）。
如果扩大数组，我们将以相应的大小插入一条记录。这并不等同于分配。
参数：
•	oh [in]：打开的数组对象句柄。
•	th [in]：事务句柄。
•	size [out]：返回的数组大小（记录的数量）。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的打开句柄。
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
1.	int
2.	daos_array_set_size(daos_handle_t oh, daos_handle_t th, daos_size_t size,
3.			    daos_event_t *ev)
4.	{
5.		daos_array_set_size_t *args;
6.		tse_task_t *           task;
7.		int                    rc;
8.	 
9.		// 创建新任务 dc_array_set_size，并将其与输入事件 ev 关联
10.		// 如果事件 ev 为 NULL，则将获取私有事件
11.		rc = dc_task_create(dc_array_set_size, NULL, ev, &task);
12.		if (rc)
13.			// dc_task_create 成功返回 0，失败返回负数
14.			return rc;
15.	 
16.		// 从 task 中获取参数
17.		args       = dc_task_get_args(task);
18.		args->oh   = oh;
19.		args->th   = th;
20.		args->size = size;
21.	 
22.		// 调度创建的任务 task
23.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
24.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
25.		//
26.		// 第二个参数 instant 为 true，表示任务将立即执行
27.		return dc_task_schedule(task, true);
28.	}
结构体 daos_array_set_size_t 表示数组设置大小的参数：
1.	typedef struct {
2.		// 打开的数组句柄
3.		daos_handle_t	oh;
4.		// 打开的事务句柄
5.		daos_handle_t	th;
6.		// 数组需要缩短的大小
7.		daos_size_t		size;
8.	} daos_array_set_size_t;
daos_array_destroy
daos_array_destroy 函数通过打孔数组对象中的所有数据（键），包括与数组关联的元数据，销毁数组对象。
daos_obj_punch() 将在接下来调用。在执行完该操作后仍然需要通过调用 daos_array_close() 来关闭打开的数组句柄，但是使用该句柄或其他打开的数组句柄进行的任何访问都将失败。销毁将在不考虑任何打开的句柄的情况下执行，因此用户有责任确保在执行销毁操作之前不再有对数组的访问。
参数：
•	oh [in]：打开的数组对象句柄。
•	th [in]：事务句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的打开句柄。
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
1.	int
2.	daos_array_destroy(daos_handle_t oh, daos_handle_t th, daos_event_t *ev)
3.	{
4.		daos_array_destroy_t *args;
5.		tse_task_t *          task;
6.		int                   rc;
7.	 
8.		// 创建新任务 dc_array_destroy，并将其与输入事件 ev 关联
9.		// 如果事件 ev 为 NULL，则将获取私有事件
10.		rc = dc_task_create(dc_array_destroy, NULL, ev, &task);
11.		if (rc)
12.			// dc_task_create 成功返回 0，失败返回负数
13.			return rc;
14.	 
15.		// 从 task 中获取参数
16.		args     = dc_task_get_args(task);
17.		args->oh = oh;
18.		args->th = th;
19.	 
20.		// 调度创建的任务 task
21.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
22.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
23.		//
24.		// 第二个参数 instant 为 true，表示任务将立即执行
25.		return dc_task_schedule(task, true);
26.	}
结构体 daos_array_destroy_t 表示数组销毁参数：
1.	typedef struct {
2.		// 打开的数组句柄
3.		daos_handle_t	oh;
4.		// 打开的事务句柄
5.		daos_handle_t	th;
6.	} daos_array_destroy_t;
daos_array_punch
daos_array_punch 函数在 IO 描述符 (iod) 中指定的数组区间中打孔。
参数：
•	oh [in]：打开的数组对象句柄。
•	th [in]：事务句柄。
•	iod [in]：要打孔的数组区间的 IO 描述符。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的打开句柄。
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
1.	int
2.	daos_array_punch(daos_handle_t oh, daos_handle_t th,
3.			 daos_array_iod_t *iod, daos_event_t *ev)
4.	{
5.		daos_array_io_t *args;
6.		tse_task_t *     task;
7.		int              rc;
8.	 
9.		// 创建新任务 dc_array_punch，并将其与输入事件 ev 关联
10.		// 如果事件 ev 为 NULL，则将获取私有事件
11.		rc = dc_task_create(dc_array_punch, NULL, ev, &task);
12.		if (rc)
13.			// dc_task_create 成功返回 0，失败返回负数
14.			return rc;
15.	 
16.		// 从 task 中获取参数
17.		args      = dc_task_get_args(task);
18.		args->oh  = oh;
19.		args->th  = th;
20.		args->iod = iod;
21.		args->sgl = NULL;
22.	 
23.		// 调度创建的任务 task
24.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
25.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
26.		//
27.		// 第二个参数 instant 为 true，表示任务将立即执行
28.		return dc_task_schedule(task, true);
29.	}
daos_array_get_attr
daos_array_get_attr 函数从打开的句柄中检索数组单元大小和块大小。
参数：
•	oh [in]：打开的数组句柄。
•	chunk_size [out]：数组的块大小。
•	cell_size [out]：数组的单元大小。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，写入 0。
•	如果失败： 
•	-DER_NO_HDL：无效的打开句柄。
•	-DER_INVAL：无效的参数。
1.	int
2.	daos_array_get_attr(daos_handle_t oh, daos_size_t *chunk_size,
3.			    daos_size_t *cell_size)
4.	{
5.		return dc_array_get_attr(oh, chunk_size, cell_size);
6.	}
转自华为云社区
Emai: debugzhang@163.com
DAOS: GitHub - daos-stack/daos: DAOS Storage Stack (client libraries, storage engine, control plane)
 
DAOS 源码解析之 daos_pool
尚先生的博客 于 2022-05-27 14:11:55 发布 38  收藏 1
分类专栏： DAOS 文章标签： 云计算
版权
 DAOS专栏收录该内容
6 篇文章0 订阅
订阅专栏
【摘要】 DAOS (Distributed Asynchronous Object Storage) 是一个开源的对象存储系统，专为大规模分布式非易失性内存设计，利用了 SCM 和 NVMe 等的下一代 NVM 技术。 DAOS 同时在硬件之上提供了键值存储接口，提供了诸如事务性非阻塞 I/O、具有自我修复的高级数据保护、端到端数据完整性、细粒度数据控制和弹性存储的高级数据保护，从而优化性能并降低成本。
本文以 Release 1.1.4 版本为标准，解析 include/daos_pool.h 中的具体实现。include/daos_pool.h 包含了 DAOS Stoarge Pool 相关的类型和函数。有关 Pool 的具体信息可参考 DAOS 分布式异步对象存储｜存储模型 DAOS Pool 一节和 DAOS 分布式异步对象存储｜Pool。
Storage Target
Target 的类型有 4 种：
1.	typedef enum {
2.		// 未知
3.		DAOS_TP_UNKNOWN,
4.		// 机械硬盘
5.		DAOS_TP_HDD,
6.		// 闪存
7.		DAOS_TP_SSD,
8.		// 持久内存
9.		DAOS_TP_PM,
10.		// 易失性内存
11.		DAOS_TP_VM,
12.	} daos_target_type_t;
Target 当前的状态有 6 种：
1.	typedef enum {
2.		// 未知
3.		DAOS_TS_UNKNOWN,
4.		// 不可用
5.		DAOS_TS_DOWN_OUT,
6.		// 不可用，可能需要重建
7.		DAOS_TS_DOWN,
8.		// 启动
9.		DAOS_TS_UP,
10.		// 启动并运行
11.		DAOS_TS_UP_IN,
12.		// Pool 映射改变导致的中间状态
13.		DAOS_TS_NEW,
14.		// 正在被清空
15.		DAOS_TS_DRAIN,
16.	} daos_target_state_t;
结构体daos_target_perf_t 用于描述 Target 的性能：
1.	typedef struct {
2.		// TODO: 存储/网络带宽、延迟等
3.		int	foo;
4.	} daos_target_perf_t;
结构体 daos_space 表示 Pool Target 的空间使用情况：
1.	struct daos_space {
2.		// 全部空间（字节）
3.		uint64_t s_total[DAOS_MEDIA_MAX];
4.		// 空闲空间（字节）
5.		uint64_t s_free[DAOS_MEDIA_MAX];
6.	};
其中，DAOS_MEDIA_MAX 表示存储空间介质的数量，一共有两种：
1.	enum {
2.		DAOS_MEDIA_SCM = 0,
3.		DAOS_MEDIA_NVME,
4.		DAOS_MEDIA_MAX
5.	};
即 s_total[DAOS_MEDIA_SCM] 和 s_free[DAOS_MEDIA_SCM] 表示 SCM (Storage-Class Memory) 的使用信息，s_total[DAOS_MEDIA_NVME] 和 s_free[DAOS_MEDIA_NVME] 表示 NVMe (Non-Volatile Memory express) 的使用信息。
结构体 daos_target_info_t 表示 Target 的信息：
1.	typedef struct {
2.		// 类型
3.		daos_target_type_t	ta_type;
4.		// 状态
5.		daos_target_state_t	ta_state;
6.		// 性能
7.		daos_target_perf_t	ta_perf;
8.		// 空间使用情况
9.		struct daos_space	ta_space;
10.	} daos_target_info_t;
Storage Pool
结构体 daos_pool_space 表示 Pool 的空间使用情况：
1.	struct daos_pool_space {
2.		// 所有活动的 Target 的聚合空间
3.		struct daos_space	ps_space;
4.		// 所有 Target 中的最大可用空间（字节）
5.		uint64_t			ps_free_min[DAOS_MEDIA_MAX];
6.		// 所有 Target 中的最小可用空间（字节）
7.		uint64_t			ps_free_max[DAOS_MEDIA_MAX];
8.		// Target 平均可用空间（字节）
9.		uint64_t			ps_free_mean[DAOS_MEDIA_MAX];
10.		// Target(VOS, Versioning Object Store) 数量
11.		uint32_t			ps_ntargets;
12.		uint32_t			ps_padding;
13.	};
结构体 daos_rebuild_status 表示重建状态：
1.	struct daos_rebuild_status {
2.		// Pool 映射在重建过程中的版本或上一个完成重建的版本
3.		uint32_t	rs_version;
4.		// 重建的时间（秒）
5.		uint32_t	rs_seconds;
6.		// 重建错误的错误码
7.		int32_t		rs_errno;
8.	 
9.		// 重建是否完成
10.		// 该字段只有在 rs_version 非 0 时有效
11.		int32_t		rs_done;
12.	 
13.		// 重建状态的填充
14.		int32_t		rs_padding32;
15.	 
16.		// 失败的 rank
17.		int32_t		rs_fail_rank;
18.	 
19.		// 要重建的对象总数，它不为 0 并且在重建过程中增加
20.		// 当 rs_done = 1 时，它将不再更改，并且应等于 rs_obj_nr
21.		// 使用 rs_toberb_obj_nr 和 rs_obj_nr，用户可以知道重建的进度
22.		uint64_t	rs_toberb_obj_nr;
23.		// 重建的对象数量，该字段非 0 当且仅当 rs_done = 1
24.		uint64_t	rs_obj_nr;
25.		// 重建的记录数量，该字段非 0 当且仅当 rs_done = 1
26.		uint64_t	rs_rec_nr;
27.	 
28.		// 重建的空间开销
29.		uint64_t	rs_size;
30.	};
daos_pool_info_bit 表示 Pool 信息查询位：
1.	enum daos_pool_info_bit {
2.		// 如果为真，查询 Pool 的空间使用情况
3.		DPI_SPACE			= 1ULL << 0,
4.		// 如果为真，查询重建状态
5.		DPI_REBUILD_STATUS 	= 1ULL << 1,
6.		// 查询所有的可选信息
7.		DPI_ALL				= -1,
8.	};
结构体 daos_pool_info_t 表示 Pool 的信息：
1.	typedef struct {
2.		// UUID
3.		uuid_t		pi_uuid;
4.		// Target 数量
5.		uint32_t					pi_ntargets;
6.		// Node 数量
7.		uint32_t					pi_nnodes;
8.		// 不活跃的 Target 数量
9.		uint32_t					pi_ndisabled;
10.		// 最新的 Pool 映射版本
11.		uint32_t					pi_map_ver;
12.		// 当前的 Raft Leader
13.		uint32_t					pi_leader;
14.		// Pool 信息查询位，其值为枚举类型 daos_pool_info_bit
15.		uint64_t					pi_bits;
16.		// 空间使用情况
17.		struct daos_pool_space		pi_space;
18.		// 重建状态
19.		struct daos_rebuild_status	pi_rebuild_st;
20.	} daos_pool_info_t;
对于每个 daos_pool_query() 调用，将始终查询基本 Pool 信息，如从 pi_uuid 到 pi_leader 的字段。但是 pi_space 和 pi_rebuild_st 是基于 pi_bits 的可选查询字段。
结构体 daos_pool_cont_info 表示 Pool Container 的信息：
1.	struct daos_pool_cont_info {
2.		// UUID
3.		uuid_t	pci_uuid;
4.	};
daos_pool_connect
daos_pool_connect 函数连接到由 UUID uuid 标识的 DAOS Pool。
成功执行后，poh 返回 Pool 句柄，info 返回最新的 Pool 信息。
参数：
•	uuid [in]：标识 Pool 的 UUID。
•	grp [in]：管理 Pool 的 DAOS 服务器的进程集合名称。
•	flags [in]：由 DAOS_PC_ 位表示的连接模式。
•	poh [out]：返回的打开句柄。
•	info [in, out]：可选参数，返回的 Pool 信息，参考枚举类型 daos_pool_info_bit。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，返回 0。
•	如果失败，返回 
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
•	-DER_NO_PERM：没有访问权限。
•	-DER_NONEXIST：Pool 不存在。
1.	int
2.	daos_pool_connect(const uuid_t uuid, const char *grp,
3.			  unsigned int flags,
4.			  daos_handle_t *poh, daos_pool_info_t *info, daos_event_t *ev)
5.	{
6.		daos_pool_connect_t *args;
7.		tse_task_t *         task;
8.		int                  rc;
9.	 
10.		// 判断 *args 大小是否与 daos_pool_connect_t 的预期大小相等
11.		DAOS_API_ARG_ASSERT(*args, POOL_CONNECT);
12.		if (!daos_uuid_valid(uuid))
13.			// UUID 无效
14.			return -DER_INVAL;
15.	 
16.		// 创建新任务 dc_pool_connect，并将其与输入事件 ev 关联
17.		// 如果事件 ev 为 NULL，则将获取私有事件
18.		rc = dc_task_create(dc_pool_connect, NULL, ev, &task);
19.		if (rc)
20.			// dc_task_create 成功返回 0，失败返回负数
21.			return rc;
22.	 
23.		// 从 task 中获取参数
24.		args        = dc_task_get_args(task);
25.		args->grp   = grp;
26.		args->flags = flags;
27.		args->poh   = poh;
28.		args->info  = info;
29.		uuid_copy((unsigned char *)args->uuid, uuid);
30.	 
31.		// 调度创建的任务 task
32.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
33.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
34.		//
35.		// 第二个参数 instant 为 true，表示任务将立即执行
36.		return dc_task_schedule(task, true);
37.	}
Pool 的连接模式有三种，由 DAOS_PC_ 位表示：
1.	// 以只读模式连接 Pool
2.	#define DAOS_PC_RO		(1U << 0)
3.	// 以读写模式连接 Pool
4.	#define DAOS_PC_RW		(1U << 1)
5.	// 以独占读写模式连接到 Pool
6.	// 如果当前存在独占 Pool 句柄，则不允许与 DSM_PC_RW 模式的连接。
7.	#define DAOS_PC_EX		(1U << 2)
8.	 
9.	// 表示连接模式的位个数
10.	#define DAOS_PC_NBITS	3
11.	// 连接模式位掩码
12.	#define DAOS_PC_MASK	((1U << DAOS_PC_NBITS) - 1)
结构体 daos_pool_connect_t 表示 Pool 连接参数：
1.	typedef struct {
2.		// Pool 的 UUID
3.		uuid_t				uuid;
4.		// 管理 Pool 的 DAOS 服务器的进程集合名称。
5.		const char			*grp;
6.		// 由 DAOS_PC_ 位表示的连接模式
7.		unsigned int		flags;
8.		// 返回的打开句柄
9.		daos_handle_t		*poh;
10.		// 可选，返回的 Pool 信息
11.		daos_pool_info_t	*info;
12.	} daos_pool_connect_t;
daos_pool_disconnect
daos_pool_disconnect 函数断开 DAOS Pool 的连接。它应该撤销该 Pool 的所有打开的 Container 句柄。
参数：
•	poh [in]：连接到 Pool 的句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，返回 0。
•	如果失败，返回 
•	-DER_UNREACH：无法访问网络。
•	-DER_NO_HDL：Pool 句柄无效。
1.	int
2.	daos_pool_disconnect(daos_handle_t poh, daos_event_t *ev)
3.	{
4.		daos_pool_disconnect_t *args;
5.		tse_task_t *            task;
6.		int                     rc;
7.	 
8.		// 判断 *args 大小是否与 daos_pool_disconnect_t 的预期大小相等
9.		DAOS_API_ARG_ASSERT(*args, POOL_DISCONNECT);
10.		// 创建新任务 dc_pool_disconnect，并将其与输入事件 ev 关联
11.		// 如果事件 ev 为 NULL，则将获取私有事件
12.		rc = dc_task_create(dc_pool_disconnect, NULL, ev, &task);
13.		if (rc)
14.			// dc_task_create 成功返回 0，失败返回负数
15.			return rc;
16.	 
17.		// 从 task 中获取参数
18.		args      = dc_task_get_args(task);
19.		args->poh = poh;
20.	 
21.		// 调度创建的任务 task
22.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
23.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
24.		//
25.		// 第二个参数 instant 为 true，表示任务将立即执行
26.		return dc_task_schedule(task, true);
27.	}
结构体 daos_pool_disconnect_t 表示断开 Pool 连接到参数：
1.	typedef struct {
2.		// 打开的 Pool 句柄
3.		daos_handle_t	poh;
4.	} daos_pool_disconnect_t;
daos_pool_local2global
daos_pool_local2global 函数将本地 Pool 连接转换为可与对等进程共享的全局表示数据。
如果 glob->iov_buf 设置为 NULL，则通过 glob->iov_buf_len 返回全局句柄的实际大小。
此功能不涉及任何通信，也不阻塞。
参数：
•	poh [in]：要共享的打开的 Pool 连接句柄。
•	glob [out]：指向 IO vector 缓冲区的指针，用于存储句柄信息。
返回值，在非阻塞模式下：
•	如果成功，返回 0。
•	如果失败： 
•	-DER_INVAL：无效的参数。
•	-DER_NO_HDL：Pool 句柄无效。
•	-DER_TRUNC：glob 中的缓冲区过小，要求更大的缓冲区。在这种情况下，要求的缓冲区大学会被写入 glob->iov_buf_len。
1.	int
2.	daos_pool_local2global(daos_handle_t poh, d_iov_t *glob)
3.	{
4.		return dc_pool_local2global(poh, glob);
5.	}
daos_pool_global2local
daos_pool_global2local 函数为全局表示数据创建本地 Pool 连接。
参数：
•	glob [in]：要提取的集合句柄的全局（共享）表示。
•	poh [out]：返回的本地 Pool 连接句柄。
返回值，在非阻塞模式下：
•	如果成功，返回 0。
•	如果失败，返回 
•	-DER_INVAL：无效的参数。
1.	int
2.	daos_pool_global2local(d_iov_t glob, daos_handle_t *poh)
3.	{
4.		return dc_pool_global2local(glob, poh);
5.	}
daos_pool_query
daos_pool_query 函数查询 Pool 信息。用户应至少提供 info 和 tgts 中的一个作为输出缓冲区。
参数：
•	poh [in]：Pool 连接句柄。
•	tgts [out]：可选，返回的 Pool 中的 Target。
•	info [in, out]：可选，返回的 Pool 信息，参考枚举类型 daos_pool_info_bit。
•	pool_prop [out]：可选，返回的 Pool 属性。 
•	如果为空，则不需要查询属性。
•	如果 pool_prop 非空，但其 dpp_entries 为空，则将查询所有 Pool 属性，DAOS 在内部分配所需的缓冲区，并将指针分配给 dpp_entries。
•	如果 pool_prop 的 dpp_nr > 0 且 dpp_entries 非空，则会查询特定的 dpe_type 属性，DAOS 会在内部为 dpe_str 或 dpe_val_ptr 分配所需的缓冲区，如果具有立即数的 dpe_type 则会直接将其分配给 dpe_val。
•	用户可以通过调用 daos_prop_free() 释放关联的缓冲区。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下：
•	如果成功，返回 0。
•	如果失败： 
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
•	-DER_NO_HDL：Pool 句柄无效。
1.	int
2.	daos_pool_query(daos_handle_t poh, d_rank_list_t *tgts, daos_pool_info_t *info,
3.			daos_prop_t *pool_prop, daos_event_t *ev)
4.	{
5.		daos_pool_query_t *args;
6.		tse_task_t *       task;
7.		int                rc;
8.	 
9.		// 判断 *args 大小是否与 daos_pool_query_t 的预期大小相等
10.		DAOS_API_ARG_ASSERT(*args, POOL_QUERY);
11.	 
12.		if (pool_prop != NULL && !daos_prop_valid(pool_prop, true, false)) {
13.			// 无效输入
14.			D_ERROR("invalid pool_prop parameter.\n");
15.			return -DER_INVAL;
16.		}
17.	 
18.		// 创建新任务 dc_pool_query，并将其与输入事件 ev 关联
19.		// 如果事件 ev 为 NULL，则将获取私有事件
20.		rc = dc_task_create(dc_pool_query, NULL, ev, &task);
21.		if (rc)
22.			// dc_task_create 成功返回 0，失败返回负数
23.			return rc;
24.	 
25.		// 从 task 中获取参数
26.		args       = dc_task_get_args(task);
27.		args->poh  = poh;
28.		args->tgts = tgts;
29.		args->info = info;
30.		args->prop = pool_prop;
31.	 
32.		// 调度创建的任务 task
33.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
34.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
35.		//
36.		// 第二个参数 instant 为 true，表示任务将立即执行
37.		return dc_task_schedule(task, true);
38.	}
结构体 daos_prop_t 表示 DAOS
Pool 或 Container 的属性：
1.	typedef struct {
2.		// 项的数量
3.		uint32_t				dpp_nr;
4.		// 保留供将来使用，现在用于 64 位对齐
5.		uint32_t				dpp_reserv;
6.		// 属性项数组
7.		struct daos_prop_entry	*dpp_entries;
8.	} daos_prop_t;
DAOS Pool 的属性类型包括：
1.	enum daos_pool_props {
2.		// 在 (DAOS_PROP_PO_MIN, DAOS_PROP_PO_MAX) 范围内有效
3.		DAOS_PROP_PO_MIN = 0,
4.	    
5.		// 标签：用户与 Pool 关联的字符串
6.		// default = ""
7.		DAOS_PROP_PO_LABEL,
8.	    
9.		// ACL：Pool 的访问控制列表
10.		// 详细说明用户和组访问权限的访问控制项的有序列表。
11.		// 期望的主体类型：Owner, User(s), Group(s), Everyone
12.		DAOS_PROP_PO_ACL,
13.	    
14.		// 保留空间比例：每个 Target 上为重建目的保留的空间量。
15.		// default = 0%.
16.		DAOS_PROP_PO_SPACE_RB,
17.	    
18.		// 自动/手动 自我修复
19.		// default = auto
20.		// 自动/手动 排除
21.		// 自动/手动 重建
22.		DAOS_PROP_PO_SELF_HEAL,
23.	    
24.		// 空间回收策略 = time|batched|snapshot
25.		// default = snapshot
26.		// time：时间间隔
27.		// batched：commits
28.		// snapshot：快照创建
29.		DAOS_PROP_PO_RECLAIM,
30.	    
31.		// 充当 Pool 所有者的用户
32.		// 格式：user@[domain]
33.		DAOS_PROP_PO_OWNER,
34.	    
35.		// 充当 Pool 所有者的组
36.		// 格式：group@[domain]
37.		DAOS_PROP_PO_OWNER_GROUP,
38.	    
39.		// Pool 的 svc rank list
40.		DAOS_PROP_PO_SVC_LIST,
41.	    
42.		DAOS_PROP_PO_MAX,
43.	};
44.	 
45.	// Pool 属性类型数量
46.	#define DAOS_PROP_PO_NUM	(DAOS_PROP_PO_MAX - DAOS_PROP_PO_MIN - 1)
结构体 daos_pool_query_t 表示 Pool 查询的参数：
1.	typedef struct {
2.		// 打开的 Pool 句柄
3.		daos_handle_t		poh;
4.		// 可选，返回的 Pool 中的 Target
5.		d_rank_list_t		*tgts;
6.		// 可选，返回的 Pool 信息
7.		daos_pool_info_t	*info;
8.		// 可选，返回的 Pool 属性
9.		daos_prop_t			*prop;
10.	} daos_pool_query_t;
daos_pool_query_target
daos_pool_query_target 函数在 DAOS Pool 中查询 Target 信息。
参数：
•	poh [in]：Pool 连接句柄。
•	tgt [in]：要查询的单个 Target 的索引。
•	rank [in]：要查询的 Target 索引的排名。
•	info [out]：返回的有关 tgt 的信息。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下：
•	如果成功，返回 0。
•	如果失败： 
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
•	-DER_NO_HDL：Pool 句柄无效。
•	-DER_NONEXIST：指定 Target 上没有 Pool。
1.	int
2.	daos_pool_query_target(daos_handle_t poh, uint32_t tgt, d_rank_t rank,
3.			       daos_target_info_t *info, daos_event_t *ev)
4.	{
5.		daos_pool_query_target_t *args;
6.		tse_task_t *              task;
7.		int                       rc;
8.	 
9.		// 判断 *args 大小是否与 daos_pool_query_target_t 的预期大小相等
10.		DAOS_API_ARG_ASSERT(*args, POOL_QUERY_INFO);
11.	 
12.		// 创建新任务 dc_pool_query_target，并将其与输入事件 ev 关联
13.		// 如果事件 ev 为 NULL，则将获取私有事件
14.		rc = dc_task_create(dc_pool_query_target, NULL, ev, &task);
15.		if (rc)
16.			// dc_task_create 成功返回 0，失败返回负数
17.			return rc;
18.	 
19.		// 从 task 中获取参数
20.		args          = dc_task_get_args(task);
21.		args->poh     = poh;
22.		args->tgt_idx = tgt_idx;
23.		args->rank    = rank;
24.		args->info    = info;
25.	 
26.		// 调度创建的任务 task
27.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
28.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
29.		//
30.		// 第二个参数 instant 为 true，表示任务将立即执行
31.		return dc_task_schedule(task, true);
32.	}
结构体 daos_pool_query_target_t 表示 Pool 的 Target 查询参数：
1.	typedef struct {
2.		// 打开的 Pool 句柄
3.		daos_handle_t		poh;
4.		// 要查询的单个 Target
5.		uint32_t			tgt_idx;
6.		// 要查询的 Target 的等级
7.		d_rank_t			rank;
8.		// 返回的 Target 信息
9.		daos_target_info_t	*info;
10.	} daos_pool_query_target_t;
daos_pool_list_attr
daos_pool_list_attr 函数列出所有用户定义的 Pool 属性的名称。
参数：
•	poh [in]：Pool 句柄。
•	buffer [out]：包含所有属性名的串联的缓冲区，每个属性名以空字符结尾。不执行截断，只返回全名。允许为 NULL，在这种情况下，只检索聚合大小。
•	size [in, out]： 
•	[in]：缓冲区大小。
•	[out]：所有属性名（不包括终止的空字符）的聚合大小，忽略实际缓冲区大小。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
1.	int
2.	daos_pool_list_attr(daos_handle_t poh, char *buffer, size_t *size,
3.			    daos_event_t *ev)
4.	{
5.		daos_pool_list_attr_t *args;
6.		tse_task_t *           task;
7.		int                    rc;
8.	 
9.		// 判断 *args 大小是否与 daos_pool_list_attr_t 的预期大小相等
10.		DAOS_API_ARG_ASSERT(*args, POOL_LIST_ATTR);
11.	 
12.		// 创建新任务 dc_pool_list_attr，并将其与输入事件 ev 关联
13.		// 如果事件 ev 为 NULL，则将获取私有事件
14.		rc = dc_task_create(dc_pool_list_attr, NULL, ev, &task);
15.		if (rc)
16.			// dc_task_create 成功返回 0，失败返回负数
17.			return rc;
18.	 
19.		// 从 task 中获取参数
20.		args       = dc_task_get_args(task);
21.		args->poh  = poh;
22.		args->buf  = buf;
23.		args->size = size;
24.	 
25.		// 调度创建的任务 task
26.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
27.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
28.		//
29.		// 第二个参数 instant 为 true，表示任务将立即执行
30.		return dc_task_schedule(task, true);
31.	}
结构体 daos_pool_list_attr_t 表示 Pool 列出属性参数：
1.	typedef struct {
2.		// 打开的 Pool 句柄
3.		daos_handle_t	poh;
4.		// 包含所有属性名的串联的缓冲区
5.		char			*buf;
6.		// [in]：缓冲区大小
7.		// [out]：所有属性名（不包括终止的空字符）的聚合大小
8.		size_t			*size;
9.	} daos_pool_list_attr_t;
daos_pool_get_attr
daos_pool_get_attr 函数获取用户定义的 Pool 属性值列表。
参数：
•	poh [in]：Pool 句柄。
•	n [in]：属性的数量。
•	names [in]：存储以空字符结尾的属性名的 n 个数组。
•	buffer [out]：存储属性值的 n 个缓冲区的数组。大于相应缓冲区大小的属性值将被截断。允许为 NULL，将被视为与零长度缓冲区相同，在这种情况下，只检索属性值的大小。
•	sizes [in, out]： 
•	[in]：存储缓冲区大小的 n 个数组。
•	[out]：存储属性值实际大小的 n 个数组，忽略实际缓冲区大小。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
1.	int
2.	daos_pool_get_attr(daos_handle_t poh, int n, char const *const names[],
3.			   void *const buffers[], size_t sizes[], daos_event_t *ev)
4.	{
5.		daos_pool_get_attr_t *args;
6.		tse_task_t *          task;
7.		int                   rc;
8.	 
9.		// 判断 *args 大小是否与 daos_pool_get_attr_t 的预期大小相等
10.		DAOS_API_ARG_ASSERT(*args, POOL_GET_ATTR);
11.	 
12.		// 创建新任务 dc_pool_get_attr，并将其与输入事件 ev 关联
13.		// 如果事件 ev 为 NULL，则将获取私有事件
14.		rc = dc_task_create(dc_pool_get_attr, NULL, ev, &task);
15.		if (rc)
16.			// dc_task_create 成功返回 0，失败返回负数
17.			return rc;
18.	 
19.		// 从 task 中获取参数
20.		args         = dc_task_get_args(task);
21.		args->poh    = poh;
22.		args->n      = n;
23.		args->names  = names;
24.		args->values = values;
25.		args->sizes  = sizes;
26.	 
27.		// 调度创建的任务 task
28.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
29.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
30.		//
31.		// 第二个参数 instant 为 true，表示任务将立即执行
32.		return dc_task_schedule(task, true);
33.	}
结构体 daos_pool_get_attr_t 表示 Pool 获取属性的参数：
1.	typedef struct {
2.		// 打开的 Pool 句柄
3.		daos_handle_t		poh;
4.		// 属性数量
5.		int					n;
6.		// 存储 n 个以空字符结尾的属性名的
7.		char const *const	*names;
8.		// 存储 n 个属性值的缓冲区
9.		void *const			*values;
10.		// [in]：存储 n 个缓冲区大小
11.		// [out]：存储  n 个属性值实际大小
12.		size_t				*sizes;
13.	} daos_pool_get_attr_t;
daos_pool_set_attr
daos_pool_set_attr 函数创建或更新用户定义的 Pool 属性值列表。
参数：
•	poh [in]：Pool 句柄。
•	n [in]：属性的数量。
•	names [in]：存储以空字符结尾的属性名的 n 个数组。
•	values [in]：存储属性值的 n 个数组。
•	sizes [in]：存储相应属性值大小的 n 个元素的数组。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
1.	int
2.	daos_pool_set_attr(daos_handle_t poh, int n, char const *const names[],
3.			   void const *const values[], size_t const sizes[],
4.			   daos_event_t *ev)
5.	{
6.		daos_pool_set_attr_t *args;
7.		tse_task_t *          task;
8.		int                   rc;
9.	 
10.		// 判断 *args 大小是否与 daos_pool_set_attr_t 的预期大小相等
11.		DAOS_API_ARG_ASSERT(*args, POOL_SET_ATTR);
12.	 
13.		// 创建新任务 dc_pool_set_attr，并将其与输入事件 ev 关联
14.		// 如果事件 ev 为 NULL，则将获取私有事件
15.		rc = dc_task_create(dc_pool_set_attr, NULL, ev, &task);
16.		if (rc)
17.			// dc_task_create 成功返回 0，失败返回负数
18.			return rc;
19.	 
20.		// 从 task 中获取参数
21.		args         = dc_task_get_args(task);
22.		args->poh    = poh;
23.		args->n      = n;
24.		args->names  = names;
25.		args->values = values;
26.		args->sizes  = sizes;
27.	 
28.		// 调度创建的任务 task
29.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
30.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
31.		//
32.		// 第二个参数 instant 为 true，表示任务将立即执行
33.		return dc_task_schedule(task, true);
34.	}
结构体 daos_pool_set_attr_t 表示 Pool 设置属性的参数：
1.	typedef struct {
2.		// 打开的 Pool 句柄
3.		daos_handle_t		poh;
4.		// 属性的数量
5.		int					n;
6.		// 存储 n 个以空字符结尾的属性名
7.		char const *const	*names;
8.		// 存储 n 个属性值
9.		void const *const	*values;
10.		// 存储 n 个相应属性值的大小。
11.		size_t const		*sizes;
12.	} daos_pool_set_attr_t;
daos_pool_del_attr
daos_pool_del_attr 函数删除用户定义的 Pool 属性值列表。
参数：
•	poh [in]：Pool 句柄。
•	n [in]：属性的数量。
•	names [in]：存储以空字符结尾的属性名的 n 个数组。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值，在非阻塞模式下会被写入 ev::ev_error：
•	如果成功，返回 0。
•	如果失败，返回 
•	-DER_INVAL：无效的参数。
•	-DER_UNREACH：无法访问网络。
•	-DER_NO_PERM：没有访问权限。
•	-DER_NO_HDL：无效的 Container 句柄。
•	-DER_NOMEM：内存不足。
1.	int
2.	daos_pool_del_attr(daos_handle_t poh, int n, char const *const names[],
3.			   daos_event_t *ev)
4.	{
5.		daos_pool_del_attr_t *args;
6.		tse_task_t *          task;
7.		int                   rc;
8.	 
9.		// 判断 *args 大小是否与 daos_pool_del_attr_t 的预期大小相等
10.		DAOS_API_ARG_ASSERT(*args, POOL_DEL_ATTR);
11.	 
12.		// 创建新任务 dc_pool_del_attr，并将其与输入事件 ev 关联
13.		// 如果事件 ev 为 NULL，则将获取私有事件
14.		rc = dc_task_create(dc_pool_del_attr, NULL, ev, &task);
15.		if (rc)
16.			// dc_task_create 成功返回 0，失败返回负数
17.			return rc;
18.	 
19.		// 从 task 中获取参数
20.		args        = dc_task_get_args(task);
21.		args->poh   = poh;
22.		args->n     = n;
23.		args->names = names;
24.	 
25.		// 调度创建的任务 task
26.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
27.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
28.		//
29.		// 第二个参数 instant 为 true，表示任务将立即执行
30.		return dc_task_schedule(task, true);
31.	}
结构体 daos_pool_del_attr_t 表示 Pool 删除属性的参数：
1.	typedef struct {
2.		// 打开的 Pool 句柄
3.		daos_handle_t		poh;
4.		// 属性的数量
5.		int					n;
6.		// 存储 n 个以空字符结尾的属性名
7.		char const *const	*names;
8.	} daos_pool_del_attr_t;
daos_pool_list_cont
daos_pool_list_cont 函数列出 Pool 的 Container。
参数：
•	poh [in]：Pool 连接句柄。
•	ncont [in, out]： 
•	[in]：以元素为单位的 cbuf 长度。
•	[out]：Pool 中的 Container 数量。
•	cbuf [out]：存储 Container 结构的数组。允许为 NULL，在这种情况下只会讲 Container 的数量写入 ncont 返回。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值：
•	如果成功，返回 0。
•	如果失败，返回 
•	-DER_INVAL：无效的参数。
•	-DER_TRUNC：cbuf 没有足够的空间存储 ncont 个元素。
1.	int
2.	daos_pool_list_cont(daos_handle_t poh, daos_size_t *ncont,
3.			    struct daos_pool_cont_info *cbuf, daos_event_t *ev)
4.	{
5.		daos_pool_list_cont_t *args;
6.		tse_task_t *           task;
7.		int                    rc;
8.	 
9.		// 判断 *args 大小是否与 daos_pool_list_cont_t 的预期大小相等
10.		DAOS_API_ARG_ASSERT(*args, POOL_LIST_CONT);
11.	 
12.		if (ncont == NULL) {
13.			// 无效输入
14.			D_ERROR("ncont must be non-NULL\n");
15.			return -DER_INVAL;
16.		}
17.	 
18.		// 创建新任务 dc_pool_list_cont，并将其与输入事件 ev 关联
19.		// 如果事件 ev 为 NULL，则将获取私有事件
20.		rc = dc_task_create(dc_pool_list_cont, NULL, ev, &task);
21.		if (rc)
22.			// dc_task_create 成功返回 0，失败返回负数
23.			return rc;
24.	 
25.		// 从 task 中获取参数
26.		args           = dc_task_get_args(task);
27.		args->poh      = poh;
28.		args->ncont    = ncont;
29.		args->cont_buf = cbuf;
30.	 
31.		// 调度创建的任务 task
32.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
33.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
34.		//
35.		// 第二个参数 instant 为 true，表示任务将立即执行
36.		return dc_task_schedule(task, true);
37.	

 
DAOS 源码解析之 daos_api
 
尚先生的博客 于 2022-05-27 13:57:58 发布 26  收藏
分类专栏： DAOS 文章标签： 云计算
版权
 DAOS专栏收录该内容
6 篇文章0 订阅
订阅专栏
【摘要】 DAOS (Distributed Asynchronous Object Storage) 是一个开源的对象存储系统，专为大规模分布式非易失性内存设计，利用了 SCM 和 NVMe 等的下一代 NVM 技术。 DAOS 同时在硬件之上提供了键值存储接口，提供了诸如事务性非阻塞 I/O、具有自我修复的高级数据保护、端到端数据完整性、细粒度数据控制和弹性存储的高级数据保护，从而优化性能并降低成本。
本文以 Release 1.1.4 版本为标准，解析 include/daos_api.h 中的具体实现。include/daos_api.h 包含了 DAOS 系统主要的 API 接口。
daos_rank_list_parse
daos_rank_list_parse 函数从带有分隔符参数的字符串生成服务器标识符列表。
该函数是 daos_pool_connect 的辅助函数，主要用于生成其所需的服务器标识符列表。
参数：
•	str [in]：用于生成服务器标识符列表的带有分隔符参数的字符串。
•	sep [in]：分隔符，例如在 dmg 中使用 : 作为分隔符。
返回值：
•	分配的标识符列表，用户需要负责调用 d_rank_list_free 释放列表。
1.	d_rank_list_t *daos_rank_list_parse(const char *str, const char *sep)
2.	{
3.		d_rank_t *     buf;
4.		int            cap   = 8;
5.		d_rank_list_t *ranks = NULL;
6.		char *         s, *s_saved;
7.		char *         p;
8.		int            n = 0;
9.	 
10.		// 分配空间给 buf 
11.		D_ALLOC_ARRAY(buf, cap);
12.		if (buf == NULL)
13.			// 给 buf 分配空间失败，不用释放 s_saved 和 buf，直接返回空指针
14.			D_GOTO(out, ranks = NULL);
15.		
16.		// 将 str 指向的字符串复制到字符串指针 s_saved 上去
17.		// s_saved 没被初始化，在复制时会给这个指针分配空间
18.		D_STRNDUP(s_saved, str, strlen(str));
19.		s = s_saved;
20.		if (s == NULL)
21.			// 给 s_saved 分配空间失败，不用释放 buf，返回空指针
22.			D_GOTO(out_buf, ranks = NULL);
23.	 
24.		// 用 seq 分割字符串
25.		while ((s = strtok_r(s, sep, &p)) != NULL) {
26.			// 当前缓冲区已满，扩展缓冲区至原来的两倍，并将数据复制过去
27.			if (n == cap) {
28.				d_rank_t *buf_new;
29.				int       cap_new;
30.	 
31.				cap_new = cap * 2;
32.				D_ALLOC_ARRAY(buf_new, cap_new);
33.				if (buf_new == NULL)
34.					D_GOTO(out_s, ranks = NULL);
35.				memcpy(buf_new, buf, sizeof(*buf_new) * n);
36.				D_FREE(buf);
37.				buf = buf_new;
38.				cap = cap_new;
39.			}
40.			// 存储标识符
41.			buf[n] = atoi(s);
42.			n++;
43.			s = NULL;
44.		}
45.	 
46.		if (n > 0) {
47.			// 为 ranks 分配空间，数组长度为 n
48.			ranks = daos_rank_list_alloc(n);
49.			if (ranks == NULL)
50.				// 给 ranks 分配空间失败，释放 s_saved 和 buf，返回空指针
51.				D_GOTO(out_s, ranks = NULL);
52.			// 将缓冲区里的标识符复制到 ranks 中
53.			memcpy(ranks->rl_ranks, buf, sizeof(*buf) * n);
54.		}
55.	out_s:
56.		// 释放 s_saved 的空间
57.		D_FREE(s_saved);
58.	out_buf:
59.		// 释放 buf 的空间
60.		D_FREE(buf);
61.	out:
62.		return ranks;
63.	}
结构体 d_rank_list_t 封装一个数组，用于存储服务器标识符：
1.	typedef struct {
2.		// 数组，用于存储服务器标识
3.		d_rank_t	*rl_ranks;
4.		// 数组长度
5.		uint32_t	rl_nr;
6.	} d_rank_list_t;
d_rank_t 表示服务器标识符：
typedef uint32_t d_rank_t;
daos_tx_open
daos_tx_open 函数在容器句柄上打开一个事务。该事务句柄可以用于容器中需要事务提交的 IOs。
参数：
•	coh [in]：容器句柄。
•	th [out]：返回的事务句柄。
•	flags [in]：事务标志（例如 DAOS_TF_RDONLY）。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值：
•	如果成功，返回 0。
•	如果失败，返回负数。
1.	int
2.	daos_tx_open(daos_handle_t coh, daos_handle_t *th, uint64_t flags,
3.		     daos_event_t *ev)
4.	{
5.		daos_tx_open_t *args;
6.		tse_task_t *    task;
7.		int             rc;
8.	 
9.		// 判断 *args 大小是否与 daos_tx_open_t 的预期大小相等
10.		DAOS_API_ARG_ASSERT(*args, TX_OPEN);
11.	  
12.		// 创建新任务 dc_tx_open，并将其与输入事件 ev 关联
13.		// 如果事件 ev 为 NULL，则将获取私有事件
14.		rc = dc_task_create(dc_tx_open, NULL, ev, &task);
15.		if (rc)
16.			// dc_task_create 成功返回 0，失败返回负数
17.			return rc;
18.	 
19.		// 从 task 中获取参数
20.		args        = dc_task_get_args(task);
21.		args->coh   = coh;
22.		args->th    = th;
23.		args->flags = flags;
24.	 
25.		// 调度创建的任务 task
26.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
27.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
28.		//
29.		// 第二个参数 instant 为 true，表示任务将立即执行
30.		return dc_task_schedule(task, true);
31.	}
结构体 daos_handle_t 表示各种 DAOS 组件（如容器、对象等）的通用句柄：
1.	typedef struct {
2.		uint64_t	cookie;
3.	} daos_handle_t;
daos_tx_open 函数的 flags 为下列两种类型：
1.	enum {
2.		// 只读事务
3.		DAOS_TF_RDONLY		= (1 << 0),
4.		// 当存在与事务相关的修改时，不要复制调用方的数据缓冲区
5.		// 在事务 daos_tx_commit 操作完成前，缓冲区必须保持不变
6.		// 
7.		// 无论此标志如何，始终复制键的缓冲区
8.		// 它们可以在相应的操作完成后释放或调整用途
9.		DAOS_TF_ZERO_COPY	= (1 << 1),
10.	};
结构体 daos_event_t 表示事件或事件队列：
1.	typedef struct daos_event {
2.		int			ev_error;
3.	  
4.		// 该结构体仅限内部使用，禁止更改
5.		struct {
6.			uint64_t space[19];
7.		}			ev_private;
8.	  
9.		// 仅限 debug 模式时使用
10.		uint64_t	ev_debug;
11.	} daos_event_t;
结构体 daos_tx_open_t 表示事务打开参数：
1.	typedef struct {
2.		// 打开的容器句柄
3.		daos_handle_t	coh;
4.		// 返回的打开的事务句柄
5.		daos_handle_t	*th;
6.		// 事务标志
7.		uint64_t		flags;
8.	} daos_tx_open_t;
结构体 tse_sched_t 用来跟踪调度程序下的所有任务：
1.	typedef struct {
2.		int		ds_result;
3.	 
4.		// 与调度程序关联的用户数据（例如 completion cb）
5.		void	*ds_udata;
6.	 
7.		// daos 的内部调度计划
8.		struct {
9.			uint64_t	ds_space[48];
10.		}		ds_private;
11.	} tse_sched_t;
daos_tx_commit
daos_tx_commit 函数用于提交事务。
如果操作成功，则事务句柄不能再用于任何新的 IO 操作。
如果返回 -DER_TX_RESTART，调用方需要使用相同的事务句柄调用 daos_tx_restart 函数重新启动该事务，执行此事务的调用方代码，然后再次调用 daos_tx_commit。
参数：
•	th [in]：要提交的事务句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值：
•	如果成功，返回 0。
•	如果失败，返回负数。可能的错误值为： 
•	-DER_NO_HDL：无效的事务句柄
•	-DER_INVAL：无效的参数
•	-DER_TX_RESTART：事物需要重启（例如，由于冲突）
1.	int
2.	daos_tx_commit(daos_handle_t th, daos_event_t *ev)
3.	{
4.		daos_tx_commit_t *args;
5.		tse_task_t *      task;
6.		int               rc;
7.	 
8.		// 判断 *args 大小是否与 daos_tx_commit_t 的预期大小相等
9.		DAOS_API_ARG_ASSERT(*args, TX_COMMIT);
10.	  
11.		// 创建新任务 daos_tx_commit，并将其与输入事件 ev 关联
12.		// 如果事件 ev 为 NULL，则将获取私有事件
13.		rc = dc_task_create(dc_tx_commit, NULL, ev, &task);
14.		if (rc)
15.			// dc_task_create 成功返回 0，失败返回负数
16.			return rc;
17.	 
18.		// 从 task 中获取参数
19.		args        = dc_task_get_args(task);
20.		args->th    = th;
21.		args->flags = 0;
22.	 
23.		// 调度创建的任务 task
24.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
25.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
26.		//
27.		// 第二个参数 instant 为 true，表示任务将立即执行
28.		return dc_task_schedule(task, true);
29.	}
结构体 daos_tx_commit_t 表示事务提交参数：
1.	typedef struct {
2.		// 打开的事务句柄
3.		daos_handle_t	th;
4.		// 事务标志，控制提交行为，例如重试
5.		uint32_t		flags;
6.	} daos_tx_commit_t;
daos_tx_open_snap
daos_tx_open_snap 函数从快照创建只读事务。
该函数不会创建快照，但只有一个读取事务会被 daos_cont_create_snap 函数创建的快照中读取出来。
如果用户传递的 epoch 代表的不是快照，或者快照已被删除，那么读取该事务可能会获得未定义的结果。
参数：
•	coh [in]：容器句柄。
•	epoch [in]：要读取的快照的 epoch。
•	th [out]：返回的只读事务句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值：
•	如果成功，返回 0。
•	如果失败，返回负数。
1.	int
2.	daos_tx_open_snap(daos_handle_t coh, daos_epoch_t epoch, daos_handle_t *th,
3.			  daos_event_t *ev)
4.	{
5.		daos_tx_open_snap_t *args;
6.		tse_task_t *         task;
7.		int                  rc;
8.	 
9.		// 判断 *args 大小是否与 daos_tx_open_snap_t 的预期大小相等
10.		DAOS_API_ARG_ASSERT(*args, TX_OPEN_SNAP);
11.	  
12.		// 创建新任务 dc_tx_open_snap，并将其与输入事件 ev 关联
13.		// 如果事件 ev 为 NULL，则将获取私有事件
14.		rc = dc_task_create(dc_tx_open_snap, NULL, ev, &task);
15.		if (rc)
16.			// dc_task_create 成功返回 0，失败返回负数
17.			return rc;
18.	 
19.		// 从 task 中获取参数
20.		args        = dc_task_get_args(task);
21.		args->coh   = coh;
22.		args->epoch = epoch;
23.		args->th    = th;
24.	 
25.		// 调度创建的任务 task
26.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
27.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
28.		//
29.		// 第二个参数 instant 为 true，表示任务将立即执行
30.		return dc_task_schedule(task, true);
31.	}
结构体 daos_tx_open_snap_t 表示事务快照打开参数：
1.	typedef struct {
2.		// 打开的容器句柄
3.		daos_handle_t	coh;
4.		// 要从中读取事务的持久快照的 epoch
5.		daos_epoch_t	epoch;
6.		// 返回的打开的事务句柄
7.		daos_handle_t	*th;
8.	} daos_tx_open_snap_t;
daos_tx_abort
daos_tx_abort 函数会中止对事务的所有修改，事务句柄将不能用于任何新的 IO 操作。
参数：
•	th [in]：要终止的事务句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值：
•	如果成功，返回 0。
•	如果失败，返回负数。
1.	int
2.	daos_tx_abort(daos_handle_t th, daos_event_t *ev)
3.	{
4.		daos_tx_abort_t *args;
5.		tse_task_t *     task;
6.		int              rc;
7.	 
8.		// 判断 *args 大小是否与 daos_tx_abort_t 的预期大小相等
9.		DAOS_API_ARG_ASSERT(*args, TX_ABORT);
10.	  
11.		// 创建新任务 dc_tx_open_snap，并将其与输入事件 ev 关联
12.		// 如果事件 ev 为 NULL，则将获取私有事件
13.		rc = dc_task_create(dc_tx_abort, NULL, ev, &task);
14.		if (rc)
15.			// dc_task_create 成功返回 0，失败返回负数
16.			return rc;
17.		
18.		// 从 task 中获取参数
19.		args     = dc_task_get_args(task);
20.		args->th = th;
21.	 
22.		// 调度创建的任务 task
23.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
24.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
25.		//
26.		// 第二个参数 instant 为 true，表示任务将立即执行
27.		return dc_task_schedule(task, true);
28.	}
结构体 daos_tx_abort_t 表示事务中断参数：
1.	typedef struct {
2.		// 打开的事务句柄
3.		daos_handle_t	th;
4.	} daos_tx_abort_t;
daos_tx_close
daos_tx_close 函数用于关闭事务句柄。
这是一个本地操作，不需要 RPC 参与。
参数：
•	th [in]：要释放的事务句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值：
•	如果成功，返回 0。
•	如果失败，返回负数。
1.	int
2.	daos_tx_close(daos_handle_t th, daos_event_t *ev)
3.	{
4.		daos_tx_close_t *args;
5.		tse_task_t *     task;
6.		int              rc;
7.	 
8.		// 判断 *args 大小是否与 daos_tx_close_t 的预期大小相等
9.		DAOS_API_ARG_ASSERT(*args, TX_CLOSE);
10.	  
11.		// 创建新任务 dc_tx_open_snap，并将其与输入事件 ev 关联
12.		// 如果事件 ev 为 NULL，则将获取私有事件
13.		rc = dc_task_create(dc_tx_close, NULL, ev, &task);
14.		if (rc)
15.			// dc_task_create 成功返回 0，失败返回负数
16.			return rc;
17.	 
18.		// 从 task 中获取参数
19.		args     = dc_task_get_args(task);
20.		args->th = th;
21.	 
22.		// 调度创建的任务 task
23.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
24.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
25.		//
26.		// 第二个参数 instant 为 true，表示任务将立即执行
27.		return dc_task_schedule(task, true);
28.	}
结构体 daos_tx_close_t 表示事务关闭参数：
1.	typedef struct {
2.		// 打开的事务句柄
3.		daos_handle_t	th;
4.	} daos_tx_close_t;
daos_tx_restart
daos_tx_restart 函数用于遇到 -DER_TX_RESTART 错误后重新启动事务，将删除通过事务句柄发出的所有 IOs 操作。
重新启动的事务能否观察到此事务最初打开后提交的任何冲突修改是未定义的。如果调用者重试事务有其它的目的，则应打开新的事务。
这是一个本地操作，不涉及RPC。
参数：
•	th [in]：要重启的事务句柄。
•	ev [in]：结束事件，该参数是可选的，可以为 NULL。当该参数为 NULL 时，该函数在阻塞模式下运行。
返回值：
•	如果成功，返回 0。
•	如果失败，返回负数。
1.	int
2.	daos_tx_restart(daos_handle_t th, daos_event_t *ev)
3.	{
4.		daos_tx_restart_t *args;
5.		tse_task_t *       task;
6.		int                rc;
7.	 
8.		// 判断 *args 大小是否与 daos_tx_restart_t 的预期大小相等
9.		DAOS_API_ARG_ASSERT(*args, TX_RESTART);
10.	  
11.		// 创建新任务 dc_tx_open_snap，并将其与输入事件 ev 关联
12.		// 如果事件 ev 为 NULL，则将获取私有事件
13.		rc = dc_task_create(dc_tx_restart, NULL, ev, &task);
14.		if (rc)
15.			// dc_task_create 成功返回 0，失败返回负数
16.			return rc;
17.	 
18.		// 从 task 中获取参数
19.		args     = dc_task_get_args(task);
20.		args->th = th;
21.	 
22.		// 调度创建的任务 task
23.		// 如果该任务的关联事件是私有事件，则此函数将等待任务完成
24.		// 否则它将立即返回，并通过测试事件或在 EQ 上轮询找到其完成情况
25.		//
26.		// 第二个参数 instant 为 true，表示任务将立即执行
27.		return dc_task_schedule(task, true);
28.	}
结构体 daos_tx_close_t 表示事务关闭参数：
1.	typedef struct {
2.		// 打开的事务句柄
3.		daos_handle_t	th;
4.	} daos_tx_close_t;
daos_tx_hdl2epoch
daos_tx_hdl2epoch 函数返回与事务句柄关联的 epoch。
一个 epoch 在事务开始时可能不可用，需要当事务成功提交后才可用。
此函数为当前系统的特定实现，它只能用于测试和调试目的。
参数：
•	th [in]：事务句柄。
•	epoch [out]：需要返回的 epoch 值
返回值：
•	如果成功，返回 0。
•	如果失败，返回负数。 
•	-DER_UNINIT 表示当前 epoch 不可用。
1.	int
2.	daos_tx_hdl2epoch(daos_handle_t th, daos_epoch_t *epoch)
3.	{
4.		return dc_tx_hdl2epoch(th, epoch);
5.	}
daos_epoch_t 为 64 位无符号整数，表示 epoch：
typedef uint64_t	daos_epoch_t;
daos_anchor_init
daos_anchor_init 函数用于初始化一个迭代器锚。
参数：
•	anchor [in]：待初始化的迭代器锚。
•	opts [in]：初始化选项（保留）。
1.	static inline int
2.	daos_anchor_init(daos_anchor_t *anchor, unsigned int opts)
3.	{
4.		// 初始化
5.		*anchor = DAOS_ANCHOR_INIT;
6.		return 0;
7.	}
DAOS_ANCHOR_INIT 是初始化宏函数，将 daos_anchor_t 的 da_type 置为 DAOS_ANCHOR_TYPE_ZERO：
#define DAOS_ANCHOR_INIT	((daos_anchor_t){DAOS_ANCHOR_TYPE_ZERO})
结构体 daos_anchor_t 用于迭代过程中的锚定：
1.	#define DAOS_ANCHOR_BUF_MAX	104
2.	typedef struct {
3.		// daos_anchor_type_t 类型
4.		uint16_t	da_type;
5.		uint16_t	da_shard;
6.		// daos_anchor_flags 枚举量
7.		uint32_t	da_flags;
8.		// 记录 EC 编解码对象的每个分片的偏移量
9.		uint64_t	da_sub_anchors;
10.		uint8_t		da_buf[DAOS_ANCHOR_BUF_MAX];
11.	} daos_anchor_t;
da_type 包括如下类型：
1.	typedef enum {
2.		DAOS_ANCHOR_TYPE_ZERO	= 0,
3.		DAOS_ANCHOR_TYPE_HKEY	= 1,
4.		DAOS_ANCHOR_TYPE_KEY	= 2,
5.		DAOS_ANCHOR_TYPE_EOF	= 3,
6.	} daos_anchor_type_t;
daos_anchor_fini
daos_anchor_fini 函数用于结束迭代，释放迭代过程中分配的空间：
1.	static inline void
2.	daos_anchor_fini(daos_anchor_t *anchor)
3.	{
4.		// 当前为空，未来可能会添加释放空间的功能
5.	}
daos_anchor_is_eof
daos_anchor_is_eof 函数用于迭代 anchor 时判断结束符：
1.	static inline bool
2.	daos_anchor_is_eof(daos_anchor_t *anchor)
3.	{
4.		return anchor->da_type == DAOS_ANCHOR_TYPE_EOF;
5.	}
本文转自华为云社区：
Emai: debugzhang@163.com
DAOS: GitHub - daos-stack/daos: DAOS Storage Stack (client libraries, storage engine, control plane)

