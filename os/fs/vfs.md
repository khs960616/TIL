# Vitrual File System

## 일반적인 파일 시스템과 관련된 인터페이스와 동작 

파일 시스템은 ==특정한 **계층 구조**로 데이터를 저장==하며, **==파일==**==과 **디렉토리** 그리고 관련된 **control Information**==을 가진다. 

일반적으로 파일 시스템이 수행하는 Operation은 생성, 삭제, 마운팅이다.


### File

* 파일은 순서가 있는 바이트 문자열이다. 
* 각 파일에는 시스템 / 유저가 구별할 수 있도록 식별자가 존재한다.
* File에 대한 일반적인 Operation은 Read, Write, Create, Delete 


### Directory entries (dentries)

* 서로 연관이 있는 파일들은 디렉토리로 모여 관리된다.
* 디렉토리는 서브 디렉토리를 포함할 수 있으며, 이러한 방식으로 디렉토리는 중첩되서 Path를 형성할 수 있으며, Path를 구성하고 있는 구성요소들을 Directory Entry라 부른다. 

  ```clike
  /home/wolfman/butter  라는 경로가 존재할 때,
  
  root directory(/)
  home  (directory)
  wolfman (directory)
  buffer  (file)
  
  path를 구성하는 모든 구성요소가 dentries이다. 
  ```
* Unix에서 디렉토리는 디렉토리 내부의 파일들의 목록을 나열하는 일반 파일일 뿐임. 
* 따라서 VFS관점에서는 디렉토리도 그저 파일이기 때문에 일반 file에 대한 Operation을 동일하게 수행할 수 있다. 


### Index Node (inode)

* Unix 시스템에서는, 파일 그 자체와 파일에 대한 부가 정보 (접근 권한, 크기, 소유자 생성시간 등)을 분리하여 관리한다.
* 파일의 메타데이터는 inode라는 별도의 데이터 구조에 저장되게 된다.


### Super block (file system meta data)

* 파일에 대한 메타정보와, 파일시스템에 대한 ==control information==은 super block에 저장되어있다.

  
:::info
  Control Information은 Super Block에서 모든 파일에 대한 컨트롤 정보를 가지고 있는 것으로 보임

  :::
* Super block은 파일 시스템에 대한 모든 정보를 저장하고 있는 자료 구조이다.
  * 각 개별 파일에 대한 정보뿐 아니라, 파일 시스템에 대한 메타 정보도 포함


### Mount points

Unix에서 파일 시스템은 namespace라는 전역 계층 구조체에 Mount되며, 이러한 특성 때문에 마운트된 각 파일 시스템들은 

사용자가 보기에는 하나의 트리 구조로 보이게 된다.

(Window는 이와 다르게, C드라이브, D드라이브 등 장치나 파티션에 따라 네임스페이스를 구별한다.)

```clike
Unix에서는 아래와 같이 파일 시스템이 달라도 결국에는 / 아래에 존재하는 하나의 시스템처럼 다루게됨 

Filesystem      Size  Used Avail Use% Mounted on
overlay         916G  357G  513G  42% /
tmpfs            64M     0   64M   0% /dev
shm             8.0G     0  8.0G   0% /dev/shm
/dev/nvme0n1p2  916G  357G  513G  42% /home
tmpfs            24G  9.2M   24G   1% /run
efivarfs        256K  125K  127K  50% /sys/firmware/efi/efivars
```


Unix에서는 파일 시스템에서 나타내는 Inode, directory, file, super block등은 실제 물리 매체에 대응되어 저장된다. 

이러한 개념이 없는 파일 시스템이라 하더라도, 실제 linux에서 이를 import해서 사용하게끔 동작하려면 마치 있는것 처럼 동작하게 만들어야한다.  → (VFS가 기대하는 형태로 데이터를 가공하는데 시간이 걸릴 수 있다)


## **VFS** (Virtual File System)

### 정의

* VFS란 유저에게 파일시스템과 연관된 인터페이스를 제공하는 리눅스의 서브 시스템이다.
* VFS는 open, read, write 와 같은 시스템콜들이 특정 파일 시스템, 머신과 관계없이 일관되게 동작할 수 있도록 \nglue 역할을 한다.

| User Interface (App I/O Request) | → | VFS Layer | → | file system method | → | Physical Media | 


### VFS Object & Data Structures 개요 


VFS는 객체지향 방식으로 설계되어있으며 다음의 4가지 주요한 object type이 존재한다. 

* **Superblock 객체**
  * 특정 마운트된 파일시스템을 나타낸다.
* **Inode 객체**
  * 특정 파일을 나타낸다.
* **Dentry 객체** (Directory Entry)
  * path 의 한 구성 요소(예: `/home/user/file.txt`에서 `home`, `user`, `file.txt`)를 나타낸다.
* **File 객체**
  * 프로세스와 연관된 open 파일을 나타낸다.

    \

  \

각 주요 Object Type들은 호출 메서드가 인터페이스로 정의되어있다.

인터페이스가 포함하는 함수들은 <https://elixir.bootlin.com/linux/v5.11.22/source/include/linux/fs.h#L1933> 를 참조 

(실제 구현은 fs를 봐야한다)

```clike
super_operations
ex) write_inode() and sync_fs()

inode_operations
ex) create() and link()

dentry_operations
ex) d_compare() and d_delete()

file_operations
ex) read() and write()
```

fs헤더를 보면, generic method들도 존재하는데, 기본 기능만으로 충분하다면 인터페이스 생성 시 \n해당 메서드들을 사용해도 무방하다.

\nVFS의 경우 이 외에도 많은 struct들이 존재하며 아래와 같은 구조체들 또한 차후 필요시 참고한다.

```clike
file_system_type: 파일시스템의 종류와 기능을 나타낸다.

vfsmount: 마운트 지점의 위치, 마운트 flag등을 가진다. 

fs_struct, file : 프로세스와 연관된 fs와 file을 나타내기 위해  사용한다.
```


### Super Block

```clike
struct super_block {
	struct list_head	s_list;		/* 모든 Super Block을 연결하는 List */
	dev_t			s_dev;		/* file system을 구별하는 identifier */
	unsigned char		s_blocksize_bits;  /* 해당 파일 시스템이 사용하는 block size(bits) */
	unsigned long		s_blocksize;      /* 해당 파일 시스템이 사용하는 block sizeb(bytes) */
	loff_t			s_maxbytes;	/* 파일 시스템에서 정의하는 최대 파일 사이즈 */
	struct file_system_type	*s_type;   /* 파일 시스템 타입 */
	const struct super_operations	*s_op;  /* 파일 시스템별로 구현해야되는 super block op */
	const struct dquot_operations	*dq_op; /*디스크 쿼터(disk quota) 관련 작업을 처리하는 함수들의 집합*/
	const struct quotactl_ops	*s_qcop; /*디스크 쿼터(disk quota) 관련 시스템 콜 인터페이스 */
   /*NFS(Network File System)와 같은 네트워크 파일 공유 시에 파일시스템 데이터를 (export) 관련 작업들을 처리하는 함수 집합*/
	const struct export_operations *s_export_op; 
	unsigned long		s_flags;    /* mount flags */
	unsigned long		s_iflags;	/* internal SB_I_* flags */
	unsigned long		s_magic; /*파일 시스템 매직 넘버 */
	struct dentry		*s_root; /*directory mount point */
	struct rw_semaphore	s_umount; /*unmount할때 잡아야되는 세마포어*/
 	int			s_count; /* super block에 대한 reference count */
	atomic_t		s_active; /* active ref count */
#ifdef CONFIG_SECURITY
	void                    *s_security; /* 보안관련 모듈이라는데 안볼거임 */
#endif
	const struct xattr_handler **s_xattr;/* 파일에 대한 확장 속성을 붙힐 수 있는 핸들러 */
#ifdef CONFIG_FS_ENCRYPTION
	const struct fscrypt_operations	*s_cop;
	struct key		*s_master_keys; /* master crypto keys in use */
#endif
#ifdef CONFIG_FS_VERITY
	const struct fsverity_operations *s_vop;
#endif
#ifdef CONFIG_UNICODE
	struct unicode_map *s_encoding;
	__u16 s_encoding_flags;
#endif
	struct hlist_bl_head	s_roots;	/* alternate root dentries for NFS */
	struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */
	struct block_device	*s_bdev;
	struct backing_dev_info *s_bdi;
	struct mtd_info		*s_mtd;
	struct hlist_node	s_instances;
	unsigned int		s_quota_types;	/* Bitmask of supported quota types */
	struct quota_info	s_dquot;	/* Diskquota specific options */

	struct sb_writers	s_writers;

	/*
	 * Keep s_fs_info, s_time_gran, s_fsnotify_mask, and
	 * s_fsnotify_marks together for cache efficiency. They are frequently
	 * accessed and rarely modified.
	 */
	void			*s_fs_info;	/* Filesystem private info */

	/* Granularity of c/m/atime in ns (cannot be worse than a second) */
	u32			s_time_gran;
	/* Time limits for c/m/atime in seconds */
	time64_t		   s_time_min;
	time64_t		   s_time_max;
#ifdef CONFIG_FSNOTIFY
	__u32			s_fsnotify_mask;
	struct fsnotify_mark_connector __rcu	*s_fsnotify_marks;
#endif

	char			s_id[32];	/* Informational name */
	uuid_t			s_uuid;		/* UUID */

	unsigned int		s_max_links;
	fmode_t			s_mode;

	/*
	 * The next field is for VFS *only*. No filesystems have any business
	 * even looking at it. You had been warned.
	 */
	struct mutex s_vfs_rename_mutex;	/* Kludge */

	/*
	 * Filesystem subtype.  If non-empty the filesystem type field
	 * in /proc/mounts will be "type.subtype"
	 */
	const char *s_subtype;

	const struct dentry_operations *s_d_op; /* default d_op for dentries */

	/*
	 * Saved pool identifier for cleancache (-1 means none)
	 */
	int cleancache_poolid;

	struct shrinker s_shrink;	/* per-sb shrinker handle */

	/* Number of inodes with nlink == 0 but still referenced */
	atomic_long_t s_remove_count;

	/* Pending fsnotify inode refs */
	atomic_long_t s_fsnotify_inode_refs;

	/* Being remounted read-only */
	int s_readonly_remount;

	/* per-sb errseq_t for reporting writeback errors via syncfs */
	errseq_t s_wb_err;

	/* AIO completions deferred from interrupt context */
	struct workqueue_struct *s_dio_done_wq;
	struct hlist_head s_pins;

	/*
	 * Owning user namespace and default context in which to
	 * interpret filesystem uids, gids, quotas, device nodes,
	 * xattrs and security labels.
	 */
	struct user_namespace *s_user_ns;

	/*
	 * The list_lru structure is essentially just a pointer to a table
	 * of per-node lru lists, each of which has its own spinlock.
	 * There is no need to put them into separate cachelines.
	 */
	struct list_lru		s_dentry_lru;
	struct list_lru		s_inode_lru;
	struct rcu_head		rcu;
	struct work_struct	destroy_work;

	struct mutex		s_sync_lock;	/* sync serialisation lock */

	/*
	 * Indicates how deep in a filesystem stack this SB is
	 */
	int s_stack_depth;

	/* s_inode_list_lock protects s_inodes */
	spinlock_t		s_inode_list_lock ____cacheline_aligned_in_smp;
	struct list_head	s_inodes;	/* all inodes */
                         /* 모든 inode들  */

	spinlock_t		s_inode_wblist_lock;
	struct list_head	s_inodes_wb;	/* writeback inodes */
} __randomize_layout;
```


### SuperBlock Operation

모든 operation이 필수인 것은 아니며, NULL일 수 있다. 

각 operation은 모두 VFS계층에서 호출되는데 사용하고, 만약 구현되어있지 않으면 generic  function을 호출하거나, 아무런 일도 일어나지 않도록 구현되어있다. 

```clike
struct super_operations {
    /* 주어진 super node아래에 새로운 inode object를 생성, 초기화하는 메서드  */
   	struct inode *(*alloc_inode)(struct super_block *sb);
    /*주어진 inode들 destory하는 메서드  */
	void (*destroy_inode)(struct inode *);
	void (*free_inode)(struct inode *);
    
    /* 
       메모리상의 inode가 수정됬을때, vfs에 의해 호출되고, ext3, ext4등의 파일 시스템에서
       저널링등에 사용된다. 
     */
   	void (*dirty_inode) (struct inode *, int flags);
    
    /* 
      주어진 inode들 disk에 wirte한다.  
      disk에 모두 쓰여질때까지 wait할껀지, 이 write가 어떤 operation에 의해 수행된건지
      (reclaim ,kupdate, background)등의 정보를 담는다. 
    */
	int (*write_inode) (struct inode *, struct writeback_control *wbc);
    
    /*
       주어진 inode의 refcount가 0이 될때 주로 호출된다.
       일반적인 파일 시스템은 이 함수가 구현이 안되어있을 수 있고, 구현이 안된경우에 
       vfs쪽에서 inode를 제거한다. (evict_inode)
    */
	int (*drop_inode) (struct inode *);
 
	void (*evict_inode) (struct inode *);
 
    /*
      주어진 super block을 release할때 사용한다. (unmount등으로 인해)
      s_lock을 잡은살태에서만 호출이 가능 
    */
	void (*put_super) (struct super_block *);
 
    /*
       파일 시스템의 메타데이터를 디스크에 동기화하며, wait할지 여부는 인자로 결정된다. 
    */
	int (*sync_fs)(struct super_block *sb, int wait);
 
 
    /* 
       커널 옛날 버전의 write_super, write_super_lockfs등과 같은 메서드인거 같음 
       디스크에 어떤 작업을할때 메모리값들 변경안되게 막기 위한 락 메커니즘인거같음 
     */
	int (*freeze_super) (struct super_block *);
	int (*freeze_fs) (struct super_block *);
	int (*thaw_super) (struct super_block *);
    
    /* 아마도 unlockfs를 대체하는거같다. */
	int (*unfreeze_fs) (struct super_block *);
    /* vfs레이어에서 호출되어 파일시스템에 대한 통계정보를 얻는데 사용됨 */
	int (*statfs) (struct dentry *, struct kstatfs *);
 
    /* 새로운 mount option으로 fs가 remount될때 호출됨   */
	int (*remount_fs) (struct super_block *, int *, char *);
 
    /* NFS등에서 사용되는 거같은데, 마운드 작업을 중단하기 위해 호출하는 것을 보임 */
	void (*umount_begin) (struct super_block *);

	int (*show_options)(struct seq_file *, struct dentry *);
	int (*show_devname)(struct seq_file *, struct dentry *);
	int (*show_path)(struct seq_file *, struct dentry *);
	int (*show_stats)(struct seq_file *, struct dentry *);
#ifdef CONFIG_QUOTA
	ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
	ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);
	struct dquot **(*get_dquots)(struct inode *);
#endif
	int (*bdev_try_to_free_page)(struct super_block*, struct page*, gfp_t);
	long (*nr_cached_objects)(struct super_block *,
				  struct shrink_control *);
	long (*free_cached_objects)(struct super_block *,
				    struct shrink_control *);
};
```


### Inode

inode 구조체는 커널에서 파일이나 디렉토리를 다루기 위한 모든 정보를 포함한다. 

(unix 스타일의 파일 시스템이 아닌 경우, inode가 없는 경우도 있어, 이 경우 linux에서 호환되려면 file에 대한 정보들을 활용해서 인메모리에서 inode 정보를 채워야 호환 가능) 


실제 inode가 가르키는 데이터의 위치 (디스크와 맵핑된) 것은 file system 별로, inode 구조체를 임베딩해서 구현한 구조체들을 봐야한다.  (예시:  ext4_inode_info → vfs_inode 임베딩, 실제 fs 구현에 따른 추가적인 정보들도 더 포함)  

```clike
struct inode {
	umode_t			i_mode;          /*  파일에 대한 접근권한  */
	unsigned short		i_opflags;
	kuid_t			i_uid;           /* user id of owner   */
	kgid_t			i_gid;           /* group id of owner  */  
	unsigned int		i_flags;     /* filesystem flags   */ 

#ifdef CONFIG_FS_POSIX_ACL
	struct posix_acl	*i_acl;
	struct posix_acl	*i_default_acl;
#endif

	const struct inode_operations	*i_op; /* inode op table */ 
	struct super_block	*i_sb;             /* 이 inode가 속한 super block ptr */
	struct address_space	*i_mapping;    /* 이 inode와 관련된 페이지 캐시      */

#ifdef CONFIG_SECURITY
	void			*i_security;
#endif

	/* Stat data, not accessed from path walking */
	unsigned long		i_ino;            /* inode 번호 (파일 시스템 내에서 고유한 식별자임에 유의) */ 
	/*
	 * Filesystems may only read i_nlink directly.  They shall use the
	 * following functions for modification:
	 *
	 *    (set|clear|inc|drop)_nlink
	 *    inode_(inc|dec)_link_count
	 */
	union {
		const unsigned int i_nlink; 
		unsigned int __i_nlink;          
	};
	dev_t			i_rdev;             /* 장치의 번호          */
	loff_t			i_size;             /* 전체 파일 크기 (byte단위, block 단위로 절삭한 크기일 수 있음) */
	struct timespec64	i_atime;        /* 마지막 접근 시간     */
	struct timespec64	i_mtime;        /* 마지막 수정 시간     */
	struct timespec64	i_ctime;        /* 마지막 메타데이터 변경 시간  */
	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
	unsigned short          i_bytes;    /* block 단위로 절삭하고 남은 크기 */
	u8			i_blkbits;              /* 논리적 블록 사이즈 나타낸다, 12이면 1블록당 2^12 */
	u8			i_write_hint;           /* 쓰기 힌트 플래그 */ 
	blkcnt_t		i_blocks;           /* 물리적으로 디스크 블록 몇개를 할당받았는지 개수(512byte단위) */

#ifdef __NEED_I_SIZE_ORDERED
	seqcount_t		i_size_seqcount;
#endif

	/* Misc */
	unsigned long		i_state;         /* 상태플래그 */
	struct rw_semaphore	i_rwsem;         /* 읽기 / 쓰기 관련 세마포어 */

	unsigned long		dirtied_when;	/* jiffies of first dirtying */
	unsigned long		dirtied_time_when; /* 더티 상태 진입 시간 */

	struct hlist_node	i_hash;     /* inode 해시 테이블 노드 */
	struct list_head	i_io_list;	/* backing dev IO list */
#ifdef CONFIG_CGROUP_WRITEBACK
	struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

	/* foreign inode detection, see wbc_detach_inode() */
	int			i_wb_frn_winner;
	u16			i_wb_frn_avg_time;
	u16			i_wb_frn_history;
#endif
	struct list_head	i_lru;		/* inode LRU list */
	struct list_head	i_sb_list;
	struct list_head	i_wb_list;	/* backing dev writeback list */
	union {
		struct hlist_head	i_dentry;
		struct rcu_head		i_rcu;
	};
	atomic64_t		i_version;
	atomic64_t		i_sequence; /* see futex */
	atomic_t		i_count;    /* 이 inode에 대한 전체 참조 카운트 (프로세스 or 파일, 디렉토리)*/
	atomic_t		i_dio_count; /* 이 inode에 대해 진행중인 direct io 개수 */
	atomic_t		i_writecount; /* 이 inode에 대해 현재 쓰기를 진행중인 수*/
#if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
	atomic_t		i_readcount; /* struct files open RO */
#endif
	union {
		const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
		void (*free_inode)(struct inode *);
	};
	struct file_lock_context	*i_flctx;
	struct address_space	i_data;
	struct list_head	i_devices;
	union {
		struct pipe_inode_info	*i_pipe;
		struct cdev		*i_cdev;
		char			*i_link;
		unsigned		i_dir_seq;
	};

	__u32			i_generation;

#ifdef CONFIG_FSNOTIFY
	__u32			i_fsnotify_mask; /* all events this inode cares about */
	struct fsnotify_mark_connector __rcu	*i_fsnotify_marks;
#endif

#ifdef CONFIG_FS_ENCRYPTION
	struct fscrypt_info	*i_crypt_info;
#endif

#ifdef CONFIG_FS_VERITY
	struct fsverity_info	*i_verity_info;
#endif

	void			*i_private; /* fs or device private pointer */
} __randomize_layout;
```

#### I_STATE

```clike
I_DIRTY_SYNC : 동기식(writeback) 더티 상태
I_DIRTY_DATASYNC : 데이터만 동기식 더티 상태

I_DIRTY_PAGES : 페이지 캐시에 더티 페이지가 있음
I_FREEING : inode가 해제되는 중임

I_NEW : 새로 생성된 inode
I_WILL_FREE : inode가 곧 삭제될 예정
I_REFERENCED : inode가 참조됨 (캐시 관리용)
```


### Inode Operation

```clike
struct inode_operations {
    // 디렉토리에서 이름에 해당하는 엔트리를 찾아 반환 (파일 또는 디렉토리 검색에 사용됨) 
	struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
 
    // 경로 탐색시, 심볼링 링크가 가르키는 실제 경로에 대한 문자열을 얻어온다.
	const char * (*get_link) (struct dentry *, struct inode *, struct delayed_call *);
 
    // inode에 대한 권한 검사를 수행한다. 
	int (*permission) (struct user_namespace *, struct inode *, int);
	struct posix_acl * (*get_acl)(struct inode *, int);

    // 유저레벨에서 시스템콜을 통해, 심볼릭 링크가 가르키는 실제 경로를 유저 버퍼에 복사해준다. 
	int (*readlink) (struct dentry *, char __user *,int);

    // 파일 생성 
	int (*create) (struct user_namespace *, struct inode *,struct dentry *,
		       umode_t, bool);
         
    // 하드 링크 생성 
	int (*link) (struct dentry *,struct inode *,struct dentry *);
    // 파일 링크 제거 (만약 호출해서, inode에 대한 i_nlink가 0이되면 inode와 데이터블록이 삭제된다) 
	int (*unlink) (struct inode *,struct dentry *);
    // 심볼릭 링크 생성 
	int (*symlink) (struct user_namespace *, struct inode *,struct dentry *,
			const char *);
     // 디렉토리 생성 
	int (*mkdir) (struct user_namespace *, struct inode *,struct dentry *,
		      umode_t);
     // 디렉토리 삭제 
	int (*rmdir) (struct inode *,struct dentry *);
    // 특수파일 생성
	int (*mknod) (struct user_namespace *, struct inode *,struct dentry *,
		      umode_t,dev_t);
	int (*rename) (struct user_namespace *, struct inode *, struct dentry *,
			struct inode *, struct dentry *, unsigned int);
	int (*setattr) (struct user_namespace *, struct dentry *,
			struct iattr *);
	int (*getattr) (struct user_namespace *, const struct path *,
			struct kstat *, u32, unsigned int);
	ssize_t (*listxattr) (struct dentry *, char *, size_t);
	int (*fiemap)(struct inode *, struct fiemap_extent_info *, u64 start,
		      u64 len);
	int (*update_time)(struct inode *, struct timespec64 *, int);
	int (*atomic_open)(struct inode *, struct dentry *,
			   struct file *, unsigned open_flag,
			   umode_t create_mode);
	int (*tmpfile) (struct user_namespace *, struct inode *,
			struct dentry *, umode_t);
	int (*set_acl)(struct user_namespace *, struct inode *,
		       struct posix_acl *, int);
} ____cacheline_aligned;
```


### Dentry 

dentry는 linux/dcache.h에 정의되어있음

→ 실질적으로 pathname lookup 같은 작업을 수행할때, 사용하기 위한 계층이라 생각하면 될 거 같음 

```clike
struct dentry {
	/* RCU lookup touched fields */
	unsigned int d_flags;		/* protected by d_lock */
	seqcount_spinlock_t d_seq;	/* per dentry seqlock */
	struct hlist_bl_node d_hash;	/* lookup hash list */
	struct dentry *d_parent;	/* parent directory */
	struct qstr d_name;         /* 이름(문자열) 길이 포함 */
	struct inode *d_inode;		/* 이 dentry가 가르키는 inode */ 
	unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */

	/* Ref lookup also touches following */
	struct lockref d_lockref;	/* per-dentry lock and refcount */
	const struct dentry_operations *d_op;
	struct super_block *d_sb;	/* The root of the dentry tree */
	unsigned long d_time;		/* used by d_revalidate */
	void *d_fsdata;			/* fs-specific data */

	union {
		struct list_head d_lru;		/* LRU list */
		wait_queue_head_t *d_wait;	/* in-lookup ones only */
	};
	struct list_head d_child;	/* child of parent list */
	struct list_head d_subdirs;	/* our children */
	/*
	 * d_alias and d_rcu can share memory
	 */
	union {
		struct hlist_node d_alias;	/* inode alias list */
		struct hlist_bl_node d_in_lookup_hash;	/* only for in-lookup ones */
	 	struct rcu_head d_rcu;
	} d_u;
} __randomize_layout;
```


### dentry operation

```clike
struct dentry_operations {
    // dentry가 유효한지 여부를 검사 (1) 
	int (*d_revalidate)(struct dentry *, unsigned int);
    // dentry가 유효한지 여부를 검사 (2) 
	int (*d_weak_revalidate)(struct dentry *, unsigned int);
 
    // dentry의 이름 기반 해시값을 계산
	int (*d_hash)(const struct dentry *, struct qstr *);
    // dentry와 주어진 char가 동일한지 비교하는 함수, 주로 경로 탐색시 활용
	int (*d_compare)(const struct dentry *,
			unsigned int, const char *, const struct qstr *);
    // dentry를 캐시에서 삭제할지 여부를 결정
	int (*d_delete)(const struct dentry *);
    // dentry 초기화시 호출
	int (*d_init)(struct dentry *);
    // dentry가 참조 카운트가 0이 되서 해제되기전 호출됨
	void (*d_release)(struct dentry *);
 
    // dentry 트리에서 이를 제거할때 호출
	void (*d_prune)(struct dentry *);
    // dentry와 연관된 inode를 해제할때 호출 (inode ref count 감소 등)
	void (*d_iput)(struct dentry *, struct inode *);
	char *(*d_dname)(struct dentry *, char *, int);
	struct vfsmount *(*d_automount)(struct path *);
	int (*d_manage)(const struct path *, bool);
	struct dentry *(*d_real)(struct dentry *, const struct inode *);
} ____cacheline_aligned;
```


### dcache

매번 path 이름을 dentry로 변환해서 탐색하고, 모든 작업 내용을 버리면 효율적이지 못하다. 따라서 커널은 이 객체들을 dentry cache에 캐싱해놓고 사용한다. 

* List of Used
* LRU List
* path를 빠르게 dentry객체로 변환할 수 있는 해시 테이블과 해시함수 

3가지 요소로 구성된다. 


---

### File

프로세스가 실제로 open한 파일을 메모리 상에서 나타내는 객체.

동일한 inode에 대해서도 여러 file 객체가 존재할 수 있으며, file 객체는 프로세스 입장에서 본 열린 파일 객체이지, 

실제 disk상에 존재하는 데이터와 연관은 없다. 

```clike
struct file {
	union {
		struct llist_node	fu_llist;
		struct rcu_head 	fu_rcuhead;
	} f_u;
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	/*
	 * Protects f_ep, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	enum rw_hint		f_write_hint;
	atomic_long_t		f_count;
	unsigned int 		f_flags;
	fmode_t			f_mode;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct hlist_head	*f_ep;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
	errseq_t		f_wb_err;
	errseq_t		f_sb_err; /* for syncfs */
} __randomize_layout
```


### File Operation

```clike
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    /* read, write의 vector 버전에 대응 (writev, readv) */
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
    /* 비동기 io 완료 여부  */ 
	int (*iopoll)(struct kiocb *kiocb, bool spin);
 
	int (*iterate) (struct file *, struct dir_context *);
	int (*iterate_shared) (struct file *, struct dir_context *);
	__poll_t (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	unsigned long mmap_supported_flags;
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
 
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
	ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
			loff_t, size_t, unsigned int);
	loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
				   struct file *file_out, loff_t pos_out,
				   loff_t len, unsigned int remap_flags);
	int (*fadvise)(struct file *, loff_t, loff_t, int);
} __randomize_layout;
```


### 기타 연관된 구조체들 

프로세스가 Open한 파일들에 대한 정보들을 담는 구조체 

```clike

/*
 * Open file table structure
 */
struct files_struct {
  /*
   * read mostly part
   */
	atomic_t count; /* file struct에 대한 count값이며 clone되면 증가한다. */
	bool resize_in_progress;  /* fd 테이블을 증가시킬때 사용하는 flag 변수 */
	wait_queue_head_t resize_wait; 

	struct fdtable __rcu *fdt;    /* 현재 유효한 fd 테이블을 가르키는 포인터이며, 초기에는 fdtab을 가르킨다 */
	struct fdtable fdtab;         /* 작은수의 파일을쓰면, 초기에는 해당 fd 테이블을 쓰다가, 
                                   * 더 많은 fd을 열기시작하면 동적으로 늘려놓게 된다.  
                                   */
  /*
   * written part on a separate cache line in SMP
   */
	spinlock_t file_lock ____cacheline_aligned_in_smp;
	unsigned int next_fd;
	unsigned long close_on_exec_init[1];
	unsigned long open_fds_init[1];
	unsigned long full_fds_bits_init[1];
    /*열려있는 fd array이며, 초기에는 이 배열을 쓰다가 만약 fd를 더 많이 필요로 하면 동적으로 할당한 배열을 쓰게됨*/
	struct file __rcu * fd_array[NR_OPEN_DEFAULT]; 
};
```


프로세스와 연관된 파일 시스템에 대한 정보를 담는 구조체 

```clike
struct fs_struct {
  int users;    // 이 구조체를 참조하고 있는 프로세스 수를 나타냄 (아마 task?)  
  rwlock_t lock; // 구조체 내 필드(특히 root, pwd)의 동기화를 위해 사용되는 스핀락
  int umask;     // 파일 생성 시 사용되는 umask 값
  int in_exec;   // 현재 프로세스가 execve() 중인지 여부를 
  struct path root; // 해당 프로세스의 루트 디렉토리 chroot() 호출 시 변경될 수 있으며, 일반적으로는 /를 나타낸다.
  struct path pwd;  // 해당 프로세스의 현재 작업 디렉토리
};
```


프로세스마다 가지고 있는 시스템에 마운트된 파일 시스템에 대한 뷰이다. (개별 프로세스마다 기준이 다를수 있다.)

```clike
struct mnt_namespace {
	struct ns_common	ns;   // namespace별로 가지고 있는 정보이며, id, reference count등이 있다.
	struct mount *	root;     // 이 namespace 상에서 '/'가 어디를 가르키는지 나타낸다. 
                              // chroot등을 이용한 프로세스는 변경될 수 있음 
	/*
	 * Traversal and modification of .list is protected by either
	 * - taking namespace_sem for write, OR
	 * - taking namespace_sem for read AND taking .ns_lock.
	 */
	struct list_head	list; // (마운트 테이블) 
                             //    - 내부적으로 이 네임스페이스에서 보는 마운트된 파일 시스템들이 연결되있음
	spinlock_t		ns_lock;
	struct user_namespace	*user_ns;
	struct ucounts		*ucounts;        // 이 네임스페이스에서의 리소스 사용량 계수 
	u64			seq;	/* Sequence number to prevent loops */
	wait_queue_head_t poll;
	u64 event;
	unsigned int		mounts; /* # of mounts in the namespace */
	unsigned int		pending_mounts;
} __randomize_layout;
```


**==fs_struct, file_struct는 일반적으로 프로세스마다 고유==**하게 가지지만, 만약 clone 호출시 flag를 이용하면 서로 다른 프로세스에서 공유해서 볼수도 있다. 


**==namespace 구조체는 기본적으로는 모든 프로세스가 동일한 네임스페이스 구조체를 참조==**한다. \n(따라서 일반적으로는 바라보는 mount table이 같기 때문에, file system에 대한 계층 구조도 동일하게 인식한다)


CLONE_NEWNS를 이용해서 clone한 경우에만, mnt_namespace 구조체를 copy해서 바라보게 된다.
