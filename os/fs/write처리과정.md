# Write 처리 과정 
VFS 레이어부터 차근차근 보자 

1) fd에 대응되는 file 객체를 가져오는 방법 -> 태스크 구조체부터 탐색해서 결국 fd테이블에서 가져옴 fastpath로직도 따로 있음
```c
static unsigned long __fget_light(unsigned int fd, fmode_t mask)
{
	struct files_struct *files = current->files;
	struct file *file;

	if (atomic_read(&files->count) == 1) {
		file = files_lookup_fd_raw(files, fd);
		if (!file || unlikely(file->f_mode & mask))
			return 0;
		return (unsigned long)file;
	} else {
		file = __fget(fd, mask, 1);
		if (!file)
			return 0;
		return FDPUT_FPUT | (unsigned long)file;
	}
}

static inline struct file *__fget(unsigned int fd, fmode_t mask,
				  unsigned int refs)
{
  /* task_struct (file_struct를 넘겨줌 */
	return __fget_files(current->files, fd, mask, refs);
}

static inline struct file *files_lookup_fd_raw(struct files_struct *files, unsigned int fd)
{
	struct fdtable *fdt = rcu_dereference_raw(files->fdt);

	if (fd < fdt->max_fds) {
		fd = array_index_nospec(fd, fdt->max_fds);
		return rcu_dereference_raw(fdt->fd[fd]);
	}
	return NULL;
}

```


2) pwrite 시작부분
```c
SYSCALL_DEFINE4(pwrite64, unsigned int, fd, const char __user *, buf,
			 size_t, count, loff_t, pos)
{
	return ksys_pwrite64(fd, buf, count, pos);
}

ssize_t ksys_pwrite64(unsigned int fd, const char __user *buf,
		      size_t count, loff_t pos)
{
	struct fd f;
	ssize_t ret = -EBADF;

	if (pos < 0)
		return -EINVAL;

	f = fdget(fd);
	if (f.file) {
		ret = -ESPIPE;
		if (f.file->f_mode & FMODE_PWRITE)  
      /* vfs 계층을 통해 시작된다.  */
			ret = vfs_write(f.file, buf, count, &pos);
		fdput(f);
	}

	return ret;
}
```


3) vfs_write

```c
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
	ssize_t ret; 
  // 파일 권한 관련 validation 
	if (!(file->f_mode & FMODE_WRITE))
		return -EBADF;
	if (!(file->f_mode & FMODE_CAN_WRITE))
		return -EINVAL;

  // 유저 레벨에서 넘겨준 buf가 정상적인지 체크 
	if (unlikely(!access_ok(buf, count)))
		return -EFAULT;

  // 읽기 or 쓰기에 대한 validtion 검사
  /*
   * 1. pos, count값 검증
   * 2. inode에 lock이 걸렸는지 체크
   * 3. Security 관련 (뭔지는 자세하게 안봐서 모르겠음, 필요하면 나중에 찾아보자)  모두 정상적이면 0을 반환한다. 
   */
	ret = rw_verify_area(WRITE, file, pos, count);
	if (ret)
		return ret;

  /* #define MAX_RW_COUNT (INT_MAX & PAGE_MASK)
   * PAGE_MASK	~(PAGE_SIZE - 1)  이므로 실제로 엄청 비상적인 사이즈로 
   * write 연산을 하지 않는 이상 왠만하면 유저 공간에서 준 size값 유지 
   */ 
	if (count > MAX_RW_COUNT)
		count =  MAX_RW_COUNT;
  
  // 파일시스템이 write도중 freeze되지 못하도록 세마포어값을 내려놓는다. 
	file_start_write(file);
  
	if (file->f_op->write)
		ret = file->f_op->write(file, buf, count, pos);  
	else if (file->f_op->write_iter)
		ret = new_sync_write(file, buf, count, pos); // 파일시스템 자체적으로 write함수는 구현안해놓고 writev류 함수만 구현해놨으면 buffer를 iov_iter로 말아서 처리함
	else
		ret = -EINVAL;
	if (ret > 0) {
		fsnotify_modify(file);
		add_wchar(current, ret);
	}
	inc_syscw(current);
	file_end_write(file);
	return ret;
}
```


```c
// write 시작전에 해당 파일(inode)이 레귤러파일이 맞으면, 속한 파일 시스템(슈퍼 블록)에 뭔가 처리를 하네? 

static inline void file_start_write(struct file *file)
{
	if (!S_ISREG(file_inode(file)->i_mode))
		return;
	sb_start_write(file_inode(file)->i_sb);
}

// 결국에 하는 일은 write도중에 freeze되는걸 방지하기 위해서 세마포어값을 올리고 내리는게 전부임 
static inline void __sb_end_write(struct super_block *sb, int level)
{
	percpu_up_read(sb->s_writers.rw_sem + level-1);
}

static inline void __sb_start_write(struct super_block *sb, int level)
{
	percpu_down_read(sb->s_writers.rw_sem + level - 1);
}
```


이러면.. 이제 또 개별 파일시스템별꺼로 봐야되는데 ext4기준으로 보자, 나중에 시간되면 VFS까지는 봐두면 좋긴할꺼같은데; 일단은 

```c
// 인터페이스에 write함수 구현이 없는걸 보면, 일단은 writev류 함수를 써서 처리됨 

const struct file_operations ext4_file_operations = {
	.llseek		= ext4_llseek,
	.read_iter	= ext4_file_read_iter,
	.write_iter	= ext4_file_write_iter,
	.iopoll		= iomap_dio_iopoll,
	.unlocked_ioctl = ext4_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= ext4_compat_ioctl,
#endif
	.mmap		= ext4_file_mmap,
	.mmap_supported_flags = MAP_SYNC,
	.open		= ext4_file_open,
	.release	= ext4_release_file,
	.fsync		= ext4_sync_file,
	.get_unmapped_area = thp_get_unmapped_area,
	.splice_read	= generic_file_splice_read,
	.splice_write	= iter_file_splice_write,
	.fallocate	= ext4_fallocate,
};

const struct inode_operations ext4_file_inode_operations = {
	.setattr	= ext4_setattr,
	.getattr	= ext4_file_getattr,
	.listxattr	= ext4_listxattr,
	.get_acl	= ext4_get_acl,
	.set_acl	= ext4_set_acl,
	.fiemap		= ext4_fiemap,
};
```

보기전에 먼저 (aio아니면 무조건 이거로 불린다, sync가 디스크에 실제 데이터가 동기화되는걸 의미하는게 아님, 유저레벨에서 커널 작업이 끝날때까지 기다리는지 여부 
```c
static ssize_t new_sync_write(struct file *filp, const char __user *buf, size_t len, loff_t *ppos)
{
	struct iovec iov = { .iov_base = (void __user *)buf, .iov_len = len };
	struct kiocb kiocb;
	struct iov_iter iter;
	ssize_t ret;
       
	init_sync_kiocb(&kiocb, filp);
	kiocb.ki_pos = (ppos ? *ppos : 0);
	iov_iter_init(&iter, WRITE, &iov, 1, len);
         
	ret = call_write_iter(filp, &kiocb, &iter);
	BUG_ON(ret == -EIOCBQUEUED);
	if (ret > 0 && ppos)
		*ppos = kiocb.ki_pos;
	return ret;
}

// 필요한 구조체들 초기화 
static inline void init_sync_kiocb(struct kiocb *kiocb, struct file *filp)
{
	*kiocb = (struct kiocb) {
		.ki_filp = filp,
		.ki_flags = iocb_flags(filp),
		.ki_hint = ki_hint_validate(file_write_hint(filp)),
		.ki_ioprio = get_current_ioprio(),
	};
}

void iov_iter_init(struct iov_iter *i, unsigned int direction,
			const struct iovec *iov, unsigned long nr_segs,
			size_t count)
{
	WARN_ON(direction & ~(READ | WRITE));
	direction &= READ | WRITE;

	/* It will get better.  Eventually... */
	if (uaccess_kernel()) {
		i->type = ITER_KVEC | direction;
		i->kvec = (struct kvec *)iov;
	} else {
		i->type = ITER_IOVEC | direction;
		i->iov = iov;
	}
	i->nr_segs = nr_segs;
	i->iov_offset = 0;
	i->count = count;
}

static inline ssize_t call_write_iter(struct file *file, struct kiocb *kio,
				      struct iov_iter *iter)
{
	return file->f_op->write_iter(kio, iter);
}
// 결국에 writev류를 호출해서 처리하게된다. 
```

이후 과정은 https://github.com/khs960616/TIL/blob/main/os/fs/ext4_write%20.md 를 참고!




버퍼를 초기화 하는 과정 

__block_write_begin 호출 (페이지 캐시에서 페이지를 가져온 이후 호출된다) 

```c
int __block_write_begin_int(struct page *page, loff_t pos, unsigned len,
		get_block_t *get_block, struct iomap *iomap)
{
	unsigned from = pos & (PAGE_SIZE - 1);           // 페이지에서 어디에 쓰려고하는지에 대한 offset
	unsigned to = from + len;                        // 어디까지 쓸껀지에 대한 정보
	struct inode *inode = page->mapping->host;       // 이 페이지가 어떤 파일에 대한 것인지에 대한 정보 
	unsigned block_start, block_end;
	sector_t block;
	int err = 0;
	unsigned blocksize, bbits;
	struct buffer_head *bh, *head, *wait[2], **wait_bh=wait;

	BUG_ON(!PageLocked(page));
	BUG_ON(from > PAGE_SIZE);
	BUG_ON(to > PAGE_SIZE);
	BUG_ON(from > to);

	head = create_page_buffers(page, inode, 0);
	blocksize = head->b_size;
	bbits = block_size_bits(blocksize);

	block = (sector_t)page->index << (PAGE_SHIFT - bbits);

	for(bh = head, block_start = 0; bh != head || !block_start;
	    block++, block_start=block_end, bh = bh->b_this_page) {
		block_end = block_start + blocksize;
		if (block_end <= from || block_start >= to) {
			if (PageUptodate(page)) {
				if (!buffer_uptodate(bh))
					set_buffer_uptodate(bh);
			}
			continue;
		}
		if (buffer_new(bh))
			clear_buffer_new(bh);
		if (!buffer_mapped(bh)) {
			WARN_ON(bh->b_size != blocksize);
			if (get_block) {
				err = get_block(inode, block, bh, 1);
				if (err)
					break;
			} else {
				iomap_to_bh(inode, block, bh, iomap);
			}

			if (buffer_new(bh)) {
				clean_bdev_bh_alias(bh);
				if (PageUptodate(page)) {
					clear_buffer_new(bh);
					set_buffer_uptodate(bh);
					mark_buffer_dirty(bh);
					continue;
				}
				if (block_end > to || block_start < from)
					zero_user_segments(page,
						to, block_end,
						block_start, from);
				continue;
			}
		}
		if (PageUptodate(page)) {
			if (!buffer_uptodate(bh))
				set_buffer_uptodate(bh);
			continue; 
		}
		if (!buffer_uptodate(bh) && !buffer_delay(bh) &&
		    !buffer_unwritten(bh) &&
		     (block_start < from || block_end > to)) {
			ll_rw_block(REQ_OP_READ, 0, 1, &bh);
			*wait_bh++=bh;
		}
	}
	/*
	 * If we issued read requests - let them complete.
	 */
	while(wait_bh > wait) {
		wait_on_buffer(*--wait_bh);
		if (!buffer_uptodate(*wait_bh))
			err = -EIO;
	}
	if (unlikely(err))
		page_zero_new_buffers(page, from, to);
	return err;
}
```


https://elixir.bootlin.com/linux/v5.11.6/source/include/linux/mm_types.h 

https://elixir.bootlin.com/linux/v5.11.6/source/include/linux/page-flags.h 필요하면 여길 참고하자 
```c
// PG_private,		/* If pagecache, has fs-private data */ 
// PG_private_2,		/* If pagecache, has fs aux data */

// 기본적으로 page내부에 union으로 여러 타입에 따라 메타데이터를 갈라놓는데  anon 영역이거나, 페이지캐시로 사용되는 캐시는 아래와 같다. 
struct {
    struct list_head lru;
    struct address_space *mapping;
    pgoff_t index;
    unsigned long private;   // 페이지 캐시 목적으로 사용되고 있다면, 여기에  buffer head의 주소가 저장되있다.
};
```

```c
static struct buffer_head *create_page_buffers(struct page *page, struct inode *inode, unsigned int b_state)
{
	BUG_ON(!PageLocked(page));

	if (!page_has_buffers(page))
		create_empty_buffers(page, 1 << READ_ONCE(inode->i_blkbits),
				     b_state);
	return page_buffers(page);
}


void create_empty_buffers(struct page *page,
			unsigned long blocksize, unsigned long b_state)
{
	struct buffer_head *bh, *head, *tail;

	head = alloc_page_buffers(page, blocksize, true);
	bh = head;
	do {
		bh->b_state |= b_state;
		tail = bh;
		bh = bh->b_this_page;
	} while (bh);
	tail->b_this_page = head;

	spin_lock(&page->mapping->private_lock);
	if (PageUptodate(page) || PageDirty(page)) {
		bh = head;
		do {
			if (PageDirty(page))
				set_buffer_dirty(bh);
			if (PageUptodate(page))
				set_buffer_uptodate(bh);
			bh = bh->b_this_page;
		} while (bh != head);
	}
	attach_page_private(page, head);
	spin_unlock(&page->mapping->private_lock);
}
```
