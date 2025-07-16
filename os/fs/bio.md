# Block I/O Layer

## 기본개념 정리

Block device:  Block(fixed-size chunk)에 대한 random access가 가능한 하드웨어 장치 \nCharacter device: 바이트 단위로 순차적인 접근을 하는 하드웨어 장치\n

Raw device: File System에 맞게 formatting되지 않은 block device (파일 시스템과 대비되는 개념) 

                     → page cache등의 layer를 안탄다고 생각하면 편하다.\n                     → 커널에 의해 bio layer를 타는건 동일함 


캐릭터 Device는 데이터를 스트림으로 다루기 때문에, 현재의 위치만 알면 되지만, 

block device는 어떤 위치로던 Random access가 가능해야되기 때문에 캐릭터 디바이스에 보다 비교적 복잡하고 또한 일반적으로 성능에 민감하다. 리눅스에서는 이를 핸들링하기 위해 서브시스템이 존재한다. 


## Sector와 Block

* **Sectors (섹터)**
  * 디바이스(하드웨어)에서 접근 가능한 가장 작은 물리적 단위
  * "hard sectors" 또는 "device blocks"라고도 부름
  * 보통 512바이트 크기이며, 물리적 장치의 최소 읽기/쓰기 단위
* **Blocks (블록)**
  * 파일시스템에서 정의한 논리적 최소 단위
  * "filesystem blocks" 또는 "I/O blocks"라고도 부름
  * 섹터 크기 이상의 크기를 가지며 보통 섹터 크기의 2^n 배수이고, 커널과 파일시스템이 I/O 단위로 처리하는 크기


섹터가 커널에서 중요한 이유는 모든 장치 I/O가 반드시 섹터 단위로 이루어지며, 그리고 커널에서 사용하는 상위 개념인 블록(block)은 이 섹터 위에서 정해진 것이기 때문이다. 


## Buffer

* read, write 연산을 진행할때 각 block은 인메모리에 저장되게 된다. 이를 Buffer라고 칭하게 된다. 
* 각 버퍼는 정확하게 하나의 block과 연관되어있다.


## Buffer Head 

각 버퍼가 위치한 메모리와, 해당 버퍼가 실제 디스크(디바이스)상에 어떤 위치를 뜻하는지 나타내는 \ndescriptor 역할을 하는 구조체 

```clike
struct buffer_head {
	unsigned long b_state;          // 버퍼의 상태(예: dirty인지 valid한 데이터를 가지고 있는지  등)를 나타내는 플래그 집합
	struct buffer_head *b_this_page; // 같은 페이지에 속한 버퍼들의 원형 리스트 내 연결 포인터 (페이지 내 다른 버퍼와 연결)
	struct page *b_page;            // 이 버퍼가 매핑된 메모리 페이지 포인터

	sector_t b_blocknr;             // 이 버퍼가 가리키는 디스크 상의 시작 블록 번호 (섹터가 아닌 논리적인 블럭)
	size_t b_size;                  // 이 버퍼가 커버하는 크기 (바이트 단위)
	char *b_data;                   // 이 페이지 내에서 실제 데이터가 저장된 위치를 가리키는 포인터

	struct block_device *b_bdev;    // 이 버퍼가 속한 블록 디바이스 (어느 디바이스의 블록인지)
	bh_end_io_t *b_end_io;          // I/O 완료 콜백 함수 포인터 (비동기 I/O 완료 시 호출)
	void *b_private;                // b_end_io에서 사용할 수 있도록 예약된 개인용 데이터 포인터

	struct list_head b_assoc_buffers; // 다른 매핑과 연관된 버퍼들의 리스트 헤드
	struct address_space *b_assoc_map; // 이 버퍼와 연관된 주소 공간 매핑 (파일 시스템의 매핑)

	atomic_t b_count;               // 버퍼의 사용 횟수 (참조 카운터)
	spinlock_t b_uptodate_lock;    // 페이지 내 첫 번째 버퍼에서 사용하며, 같은 페이지 내 다른 버퍼들의 I/O 완료 동기화를 위한 락
};
```

버퍼는 b_page라는 메모리에 위치하고 있으며, 

해당 페이지에서 b_data의 위치부터 b_size만큼이 실제로 인메모리에 맵핑된 주소를 뜻한다. 

또한 하드웨어적으로는   b_bdev라는 장치가 가르키는 디바이스에서 b_blocknr 위치를 나타낸다. 


## BIO

여러 개의 물리 메모리 상의 연속적이지 않은 버퍼들을 묶어서 하나의 I/O 작업으로 표현할 수 있게 만든 구조체

```clike
struct bio {
	struct bio		*bi_next;      // 요청 큐에서 다음 bio를 가리키는 링크
	struct block_device	*bi_bdev;     // 대상 블록 디바이스 포인터
	unsigned int		bi_opf;       // 요청 플래그 및 작업 유형 (REQ_OP)
	unsigned short		bi_flags;     // BIO_* 플래그
	unsigned short		bi_ioprio;    // I/O 우선순위
	unsigned short		bi_write_hint;// 쓰기 힌트 (예: 순차 쓰기 등)
	blk_status_t		bi_status;    // I/O 완료 상태
	atomic_t		__bi_remaining; // 완료 대기 중인 I/O 조각 수

	struct bvec_iter	bi_iter;      // bio_vec 리스트 내 현재 처리 위치 정보
	bio_end_io_t		*bi_end_io;   // I/O 완료 시 호출될 콜백 함수
	void			*bi_private;  // 드라이버/상위 레이어용 사용자 데이터 포인터

#ifdef CONFIG_BLK_CGROUP
	struct blkcg_gq		*bi_blkg;     // 블록 그룹 큐 관련 정보 (블록 cgroup)
	struct bio_issue	bi_issue;     // I/O issue 관련 데이터
#ifdef CONFIG_BLK_CGROUP_IOCOST
	u64			bi_iocost_cost; // I/O 비용(블록 cgroup IO cost)
#endif
#endif

#ifdef CONFIG_BLK_INLINE_ENCRYPTION
	struct bio_crypt_ctx	*bi_crypt_context; // 암호화 컨텍스트 정보
#endif

	union {
#if defined(CONFIG_BLK_DEV_INTEGRITY)
		struct bio_integrity_payload *bi_integrity; // 데이터 무결성 정보
#endif
	};

	unsigned short		bi_vcnt;      // 현재 bio_vec(메모리 조각) 개수
	unsigned short		bi_max_vecs;  // bio가 담을 수 있는 최대 bio_vec 개수
	atomic_t		__bi_cnt;     // bio 참조 카운터 (pin count)
	struct bio_vec		*bi_io_vec;   // 
	struct bio_set		*bi_pool;     // bio 할당 풀 포인터
	struct bio_vec		bi_inline_vecs[]; // 소량의 bio_vec을 인라인 저장 (메모리 최적화)
};
```


실제 주요한 필드들 

```clike
bi_io_vec: 실제 I/O 작업에 사용되는 버퍼 조각(bio_vec 구조체 배열)들의 리스트 포인터
bi_vcnt: bio_io_vec 배열 내 현재 포함된 버퍼 조각의 개수
bi_iter: 현재 처리 중인 버퍼의 위치 
```


```clike
/**
 * struct bio_vec - a contiguous range of physical memory addresses
 * @bv_page:   First page associated with the address range.
 * @bv_len:    Number of bytes in the address range.
 * @bv_offset: Start of the address range relative to the start of @bv_page.
 *
 * The following holds for a bvec if n * PAGE_SIZE < bv_offset + bv_len:
 *
 *   nth_page(@bv_page, n) == @bv_page + n
 *
 * This holds because page_is_mergeable() checks the above property.
 */
struct bio_vec {
	struct page	*bv_page;
	unsigned int	bv_len;
	unsigned int	bv_offset;
};
```


## Block Device Request Queue

```clike
struct request_queue {
	struct request		*last_merge;
	struct elevator_queue	*elevator;

	struct percpu_ref	q_usage_counter;

	struct blk_queue_stats	*stats;
	struct rq_qos		*rq_qos;

	const struct blk_mq_ops	*mq_ops;

	/* sw queues */
	struct blk_mq_ctx __percpu	*queue_ctx;

	unsigned int		queue_depth;

	/* hw dispatch queues */
	struct blk_mq_hw_ctx	**queue_hw_ctx;
	unsigned int		nr_hw_queues;

	struct backing_dev_info	*backing_dev_info;

	/*
	 * The queue owner gets to use this for whatever they like.
	 * ll_rw_blk doesn't touch it.
	 */
	void			*queuedata;

	/*
	 * various queue flags, see QUEUE_* below
	 */
	unsigned long		queue_flags;
	/*
	 * Number of contexts that have called blk_set_pm_only(). If this
	 * counter is above zero then only RQF_PM requests are processed.
	 */
	atomic_t		pm_only;

	/*
	 * ida allocated id for this queue.  Used to index queues from
	 * ioctx.
	 */
	int			id;

	/*
	 * queue needs bounce pages for pages above this limit
	 */
	gfp_t			bounce_gfp;

	spinlock_t		queue_lock;

	/*
	 * queue kobject
	 */
	struct kobject kobj;

	/*
	 * mq queue kobject
	 */
	struct kobject *mq_kobj;

#ifdef  CONFIG_BLK_DEV_INTEGRITY
	struct blk_integrity integrity;
#endif	/* CONFIG_BLK_DEV_INTEGRITY */

#ifdef CONFIG_PM
	struct device		*dev;
	enum rpm_status		rpm_status;
#endif

	/*
	 * queue settings
	 */
	unsigned long		nr_requests;	/* Max # of requests */

	unsigned int		dma_pad_mask;
	unsigned int		dma_alignment;

#ifdef CONFIG_BLK_INLINE_ENCRYPTION
	/* Inline crypto capabilities */
	struct blk_keyslot_manager *ksm;
#endif

	unsigned int		rq_timeout;
	int			poll_nsec;

	struct blk_stat_callback	*poll_cb;
	struct blk_rq_stat	poll_stat[BLK_MQ_POLL_STATS_BKTS];

	struct timer_list	timeout;
	struct work_struct	timeout_work;

	atomic_t		nr_active_requests_shared_sbitmap;

	struct list_head	icq_list;
#ifdef CONFIG_BLK_CGROUP
	DECLARE_BITMAP		(blkcg_pols, BLKCG_MAX_POLS);
	struct blkcg_gq		*root_blkg;
	struct list_head	blkg_list;
#endif

	struct queue_limits	limits;

	unsigned int		required_elevator_features;

#ifdef CONFIG_BLK_DEV_ZONED
	/*
	 * Zoned block device information for request dispatch control.
	 * nr_zones is the total number of zones of the device. This is always
	 * 0 for regular block devices. conv_zones_bitmap is a bitmap of nr_zones
	 * bits which indicates if a zone is conventional (bit set) or
	 * sequential (bit clear). seq_zones_wlock is a bitmap of nr_zones
	 * bits which indicates if a zone is write locked, that is, if a write
	 * request targeting the zone was dispatched. All three fields are
	 * initialized by the low level device driver (e.g. scsi/sd.c).
	 * Stacking drivers (device mappers) may or may not initialize
	 * these fields.
	 *
	 * Reads of this information must be protected with blk_queue_enter() /
	 * blk_queue_exit(). Modifying this information is only allowed while
	 * no requests are being processed. See also blk_mq_freeze_queue() and
	 * blk_mq_unfreeze_queue().
	 */
	unsigned int		nr_zones;
	unsigned long		*conv_zones_bitmap;
	unsigned long		*seq_zones_wlock;
	unsigned int		max_open_zones;
	unsigned int		max_active_zones;
#endif /* CONFIG_BLK_DEV_ZONED */

	/*
	 * sg stuff
	 */
	unsigned int		sg_timeout;
	unsigned int		sg_reserved_size;
	int			node;
	struct mutex		debugfs_mutex;
#ifdef CONFIG_BLK_DEV_IO_TRACE
	struct blk_trace __rcu	*blk_trace;
#endif
	/*
	 * for flush operations
	 */
	struct blk_flush_queue	*fq;

	struct list_head	requeue_list;
	spinlock_t		requeue_lock;
	struct delayed_work	requeue_work;

	struct mutex		sysfs_lock;
	struct mutex		sysfs_dir_lock;

	/*
	 * for reusing dead hctx instance in case of updating
	 * nr_hw_queues
	 */
	struct list_head	unused_hctx_list;
	spinlock_t		unused_hctx_lock;

	int			mq_freeze_depth;

#if defined(CONFIG_BLK_DEV_BSG)
	struct bsg_class_device bsg_dev;
#endif

#ifdef CONFIG_BLK_DEV_THROTTLING
	/* Throttle data */
	struct throtl_data *td;
#endif
	struct rcu_head		rcu_head;
	wait_queue_head_t	mq_freeze_wq;
	/*
	 * Protect concurrent access to q_usage_counter by
	 * percpu_ref_kill() and percpu_ref_reinit().
	 */
	struct mutex		mq_freeze_lock;

	struct blk_mq_tag_set	*tag_set;
	struct list_head	tag_set_list;
	struct bio_set		bio_split;

	struct dentry		*debugfs_dir;

#ifdef CONFIG_BLK_DEBUG_FS
	struct dentry		*sched_debugfs_dir;
	struct dentry		*rqos_debugfs_dir;
#endif

	bool			mq_sysfs_init_done;

	size_t			cmd_size;

#define BLK_MAX_WRITE_HINTS	5
	u64			write_hints[BLK_MAX_WRITE_HINTS];
};
```


요청 큐는 요청들을 담은 이중 연결 리스트(doubly linked list)와 관련 제어 정보를 포함합니다.\n요청은 파일 시스템과 같은 커널의 상위 계층 코드에 의해 큐에 추가됩니다.

요청 큐에 항목이 하나라도 있는 동안, 해당 큐에 연결된 블록 디바이스 드라이버는\n큐의 앞(head)에서 요청을 꺼내어 블록 디바이스에 제출합니다.\n요청 큐의 리스트에 있는 각 항목은 하나의 요청이며, 이는 `struct request` 타입입니다.

큐에 있는 개별 요청은 `<linux/blkdev.h>`에 정의된 `struct request`로 표현됩니다.\n각 요청은 하나 이상의 `bio` 구조체로 구성될 수 있는데,\n이유는 개별 요청이 연속적인 여러 디스크 블록에 대해 작동할 수 있기 때문입니다.

단, 디스크 상의 블록은 반드시 인접해야 하지만,\n메모리 상의 블록은 인접할 필요가 없습니다.\n각 `bio` 구조체는 여러 세그먼트(segments, 메모리 내 연속된 블록)를 묘사할 수 있고,\n하나의 요청은 여러 `bio` 구조체로 구성될 수 있습니다.


## I/O Scheduler 

Disk seek는 현대 컴퓨터에서 가장 느린 동작이다. 따라서 request를 커널이 만들자마자 매번 block device한테 다보내면 

성능이 떨어진다. 그러므로 커널은 I/O Scheduler라는 서브시스템에서 merging과 sorting을 해서 성능을 올리기 위한 노력을한다. 


→ 1) Request Queue에 만약, 새로 들어온 Request와 인접한 섹터에 대한 Operation이 있다면 두 Request를 Merge하여 하나의 요청으로 만든다. 


→ 2) 인접한 섹터에 대한 요청이 없다면, queue에 넣는다. 그런데 아무런 정렬없이 queue에 마구잡이로 넣으면, seek를 줄이기 어렵다. 따라서 Request Queue는  seek time을 줄이기 위해 섹터단위로 정렬되도록 request들을 정리한다.
