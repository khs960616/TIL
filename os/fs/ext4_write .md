https://elixir.bootlin.com/linux/v5.10.238/source/fs/ext4/file.c 

fs/ext4/inode.c

인터페이스 
```c
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

static const struct address_space_operations ext4_aops = {
	.readpage		= ext4_readpage,
	.readahead		= ext4_readahead,
	.writepage		= ext4_writepage,
	.writepages		= ext4_writepages,
	.write_begin		= ext4_write_begin,
	.write_end		= ext4_write_end,
	.set_page_dirty		= ext4_set_page_dirty,
	.bmap			= ext4_bmap,
	.invalidatepage		= ext4_invalidatepage,
	.releasepage		= ext4_releasepage,
	.direct_IO		= noop_direct_IO,
	.migratepage		= buffer_migrate_page,
	.is_partially_uptodate  = block_is_partially_uptodate,
	.error_remove_page	= generic_error_remove_page,
	.swap_activate		= ext4_iomap_swap_activate,
};
```

write 
```c
static ssize_t
ext4_file_write_iter(struct kiocb *iocb, struct iov_iter *from)
{       
        // iocb에 있는 file pointer에 대응되는 실제 inode 포인터를 가져온다. 
	struct inode *inode = file_inode(iocb->ki_filp);

        // 이 inode가 속한 파일시스템이 맛탱이가 간 경우 
	if (unlikely(ext4_forced_shutdown(EXT4_SB(inode->i_sb))))
		return -EIO;

#ifdef CONFIG_FS_DAX
	if (IS_DAX(inode))
		return ext4_dax_write_iter(iocb, from);
#endif
        // O_DIRECT 켜서 연 경우, 
	if (iocb->ki_flags & IOCB_DIRECT)
		return ext4_dio_write_iter(iocb, from);
	else // 아닌 경우는 buffer_io! 
		return ext4_buffered_write_iter(iocb, from);
}
```

dio는 나중에 필요할때 다시 보자 (O_DIRECT) 안킨거 먼저 

```c
static ssize_t ext4_buffered_write_iter(struct kiocb *iocb,
					struct iov_iter *from)
{
	ssize_t ret;
	struct inode *inode = file_inode(iocb->ki_filp);

	if (iocb->ki_flags & IOCB_NOWAIT)
		return -EOPNOTSUPP;

	inode_lock(inode); // inode에 대한 Lock! -> 멀티 스레드 써봤자 도움안됨 
	ret = ext4_write_checks(iocb, from); //validation이랑, iov_iter에 크기를 실제 write할수있는 바이트수만큼으로 조절 
	if (ret <= 0)
		goto out;
        
	current->backing_dev_info = inode_to_bdi(inode); //프로세스에다가 현재 접근하려고 하는 블록 디바이스 정보를 적어놓음 
	ret = generic_perform_write(iocb->ki_filp, from, iocb->ki_pos); 
	current->backing_dev_info = NULL;  // 이후에 NULL로 값을 잘 닦는다. 

out:
	inode_unlock(inode);
	if (likely(ret > 0)) {
		iocb->ki_pos += ret;
                // sync 옵션 걸어둔 경우 
		ret = generic_write_sync(iocb, ret);
	}

	return ret;
}
```

커널 5버전부턴 일단 mm/filemap.c에 정의

```c
ssize_t generic_perform_write(struct file *file,
				struct iov_iter *i, loff_t pos)
{
	struct address_space *mapping = file->f_mapping;
	const struct address_space_operations *a_ops = mapping->a_ops;
	long status = 0;
	ssize_t written = 0;
	unsigned int flags = 0;

	do {
		struct page *page;
		unsigned long offset;	/* Offset into pagecache page */
		unsigned long bytes;	/* Bytes to write to page */
		size_t copied;		/* Bytes copied from user */
		void *fsdata = NULL;

		offset = (pos & (PAGE_SIZE - 1));
		bytes = min_t(unsigned long, PAGE_SIZE - offset,
						iov_iter_count(i));

again:
		/*
		 * Bring in the user page that we will copy from _first_.
		 * Otherwise there's a nasty deadlock on copying from the
		 * same page as we're writing to, without it being marked
		 * up-to-date.
		 *
		 * Not only is this an optimisation, but it is also required
		 * to check that the address is actually valid, when atomic
		 * usercopies are used, below.
		 */
		if (unlikely(iov_iter_fault_in_readable(i, bytes))) {
			status = -EFAULT;
			break;
		}

		if (fatal_signal_pending(current)) {
			status = -EINTR;
			break;
		}

                // 여기가 페이지 캐시 확보하는 부분 
		status = a_ops->write_begin(file, mapping, pos, bytes, flags,
						&page, &fsdata);
		if (unlikely(status < 0))
			break;

		if (mapping_writably_mapped(mapping))
			flush_dcache_page(page);

                // user로부터 넘어온 buffer 데이터를 페이지에 copy한다. 
		copied = iov_iter_copy_from_user_atomic(page, i, offset, bytes);
		flush_dcache_page(page);

                // dirty 페이지 마크 
		status = a_ops->write_end(file, mapping, pos, bytes, copied,
						page, fsdata);
		if (unlikely(status < 0))
			break;
		copied = status;

		cond_resched();

		iov_iter_advance(i, copied);
		if (unlikely(copied == 0)) {
			/*
			 * If we were unable to copy any data at all, we must
			 * fall back to a single segment length write.
			 *
			 * If we didn't fallback here, we could livelock
			 * because not all segments in the iov can be copied at
			 * once without a pagefault.
			 */
			bytes = min_t(unsigned long, PAGE_SIZE - offset,
						iov_iter_single_seg_count(i));
			goto again;
		}
		pos += copied;
		written += copied;

		balance_dirty_pages_ratelimited(mapping);
	} while (iov_iter_count(i));

	return written ? written : status;
}
```

일단 page캐시쪽에 copy하는건 fs에 종속적이지 않으니 이거부터 메모 
```c
size_t iov_iter_copy_from_user_atomic(struct page *page,
		struct iov_iter *i, unsigned long offset, size_t bytes)
{
	char *kaddr = kmap_atomic(page), *p = kaddr + offset;
	if (unlikely(!page_copy_sane(page, offset, bytes))) {
		kunmap_atomic(kaddr);
		return 0;
	}
	if (unlikely(iov_iter_is_pipe(i) || iov_iter_is_discard(i))) {
		kunmap_atomic(kaddr);
		WARN_ON(1);
		return 0;
	}
	iterate_all_kinds(i, bytes, v,
		copyin((p += v.iov_len) - v.iov_len, v.iov_base, v.iov_len),
		memcpy_from_page((p += v.bv_len) - v.bv_len, v.bv_page,
				 v.bv_offset, v.bv_len),
		memcpy((p += v.iov_len) - v.iov_len, v.iov_base, v.iov_len)
	)
	kunmap_atomic(kaddr);
	return bytes;
}
```


각각의 operation도 fs별 inode 구현에 있는 것으로 보인다.

fs/ext4/inode.c 

```c
static const struct address_space_operations ext4_aops = {
	.readpage		= ext4_readpage,
	.readahead		= ext4_readahead,
	.writepage		= ext4_writepage,
	.writepages		= ext4_writepages,
	.write_begin		= ext4_write_begin,
	.write_end		= ext4_write_end,
	.set_page_dirty		= ext4_set_page_dirty,
	.bmap			= ext4_bmap,
	.invalidatepage		= ext4_invalidatepage,
	.releasepage		= ext4_releasepage,
	.direct_IO		= noop_direct_IO,
	.migratepage		= buffer_migrate_page,
	.is_partially_uptodate  = block_is_partially_uptodate,
	.error_remove_page	= generic_error_remove_page,
	.swap_activate		= ext4_iomap_swap_activate,
};


static int ext4_write_begin(struct file *file, struct address_space *mapping,
			    loff_t pos, unsigned len, unsigned flags,
			    struct page **pagep, void **fsdata)
{
	struct inode *inode = mapping->host;
	int ret, needed_blocks;
	handle_t *handle;
	int retries = 0;
	struct page *page;
	pgoff_t index;
	unsigned from, to;

	if (unlikely(ext4_forced_shutdown(EXT4_SB(inode->i_sb))))
		return -EIO;

	trace_ext4_write_begin(inode, pos, len, flags);
	/*
	 * Reserve one block more for addition to orphan list in case  
	 * we allocate blocks but write fails for some reason             저널링에 필요한 block 개수 == needed_blocks 
	 */
	needed_blocks = ext4_writepage_trans_blocks(inode) + 1;

        // 지금 쓰려는 위치가 파일내의 몇번째 페이지를 가르키는지
	index = pos >> PAGE_SHIFT;
        // 페이지 내에서의 offset (bytes)
	from = pos & (PAGE_SIZE - 1);
        // 어디까지 쓸려고하는건지 
	to = from + len;

        // inode의 상태 flag를 체크한다. 마운트시 결정한 inline_data옵션등에 따라, inode 자체에 파일 내용까지 같이 적는 경우에 여기로 타게된다. 
	if (ext4_test_inode_state(inode, EXT4_STATE_MAY_INLINE_DATA)) {
		ret = ext4_try_to_write_inline_data(mapping, inode, pos, len,
						    flags, pagep);
		if (ret < 0)
			return ret;
		if (ret == 1)
			return 0;
	}

	/*
	 * grab_cache_page_write_begin() can take a long time if the
	 * system is thrashing due to memory pressure, or if the page
	 * is being written back.  So grab it first before we start
	 * the transaction handle.  This also allows us to allocate
	 * the page (if needed) without using GFP_NOFS.
	 */
retry_grab:
	page = grab_cache_page_write_begin(mapping, index, flags);
	if (!page)
		return -ENOMEM;
	unlock_page(page);

retry_journal:
	handle = ext4_journal_start(inode, EXT4_HT_WRITE_PAGE, needed_blocks);
	if (IS_ERR(handle)) {
		put_page(page);
		return PTR_ERR(handle);
	}

	lock_page(page);
	if (page->mapping != mapping) {
		/* The page got truncated from under us */
		unlock_page(page);
		put_page(page);
		ext4_journal_stop(handle);
		goto retry_grab;
	}
	/* In case writeback began while the page was unlocked */
	wait_for_stable_page(page);

#ifdef CONFIG_FS_ENCRYPTION
	if (ext4_should_dioread_nolock(inode))
		ret = ext4_block_write_begin(page, pos, len,
					     ext4_get_block_unwritten);
	else
		ret = ext4_block_write_begin(page, pos, len,
					     ext4_get_block);
#else
	if (ext4_should_dioread_nolock(inode))
		ret = __block_write_begin(page, pos, len,
					  ext4_get_block_unwritten);
	else
		ret = __block_write_begin(page, pos, len, ext4_get_block);
#endif
	if (!ret && ext4_should_journal_data(inode)) {
		ret = ext4_walk_page_buffers(handle, page_buffers(page),
					     from, to, NULL,
					     do_journal_get_write_access);
	}

	if (ret) {
		bool extended = (pos + len > inode->i_size) &&
				!ext4_verity_in_progress(inode);

		unlock_page(page);
		/*
		 * __block_write_begin may have instantiated a few blocks
		 * outside i_size.  Trim these off again. Don't need
		 * i_size_read because we hold i_mutex.
		 *
		 * Add inode to orphan list in case we crash before
		 * truncate finishes
		 */
		if (extended && ext4_can_truncate(inode))
			ext4_orphan_add(handle, inode);

		ext4_journal_stop(handle);
		if (extended) {
			ext4_truncate_failed_write(inode);
			/*
			 * If truncate failed early the inode might
			 * still be on the orphan list; we need to
			 * make sure the inode is removed from the
			 * orphan list in that case.
			 */
			if (inode->i_nlink)
				ext4_orphan_del(NULL, inode);
		}

		if (ret == -ENOSPC &&
		    ext4_should_retry_alloc(inode->i_sb, &retries))
			goto retry_journal;
		put_page(page);
		return ret;
	}
	*pagep = page;
	return ret;
}


static int ext4_write_end(struct file *file,
			  struct address_space *mapping,
			  loff_t pos, unsigned len, unsigned copied,
			  struct page *page, void *fsdata)
{
	handle_t *handle = ext4_journal_current_handle();
	struct inode *inode = mapping->host;
	loff_t old_size = inode->i_size;
	int ret = 0, ret2;
	int i_size_changed = 0;
	int inline_data = ext4_has_inline_data(inode);
	bool verity = ext4_verity_in_progress(inode);

	trace_ext4_write_end(inode, pos, len, copied);
	if (inline_data &&
	    ext4_test_inode_state(inode, EXT4_STATE_MAY_INLINE_DATA)) {
		ret = ext4_write_inline_data_end(inode, pos, len,
						 copied, page);
		if (ret < 0) {
			unlock_page(page);
			put_page(page);
			goto errout;
		}
		copied = ret;
		ret = 0;
	} else
		copied = block_write_end(file, mapping, pos,
					 len, copied, page, fsdata);
	/*
	 * it's important to update i_size while still holding page lock:
	 * page writeout could otherwise come in and zero beyond i_size.
	 *
	 * If FS_IOC_ENABLE_VERITY is running on this inode, then Merkle tree
	 * blocks are being written past EOF, so skip the i_size update.
	 */
	if (!verity)
		i_size_changed = ext4_update_inode_size(inode, pos + copied);
	unlock_page(page);
	put_page(page);

	if (old_size < pos && !verity)
		pagecache_isize_extended(inode, old_size, pos);
	/*
	 * Don't mark the inode dirty under page lock. First, it unnecessarily
	 * makes the holding time of page lock longer. Second, it forces lock
	 * ordering of page lock and transaction start for journaling
	 * filesystems.
	 */
	if (i_size_changed || inline_data)
		ret = ext4_mark_inode_dirty(handle, inode);

errout:
	if (pos + len > inode->i_size && !verity && ext4_can_truncate(inode))
		/* if we have allocated more blocks and copied
		 * less. We will have blocks allocated outside
		 * inode->i_size. So truncate them
		 */
		ext4_orphan_add(handle, inode);

	ret2 = ext4_journal_stop(handle);
	if (!ret)
		ret = ret2;

	if (pos + len > inode->i_size && !verity) {
		ext4_truncate_failed_write(inode);
		/*
		 * If truncate failed early the inode might still be
		 * on the orphan list; we need to make sure the inode
		 * is removed from the orphan list in that case.
		 */
		if (inode->i_nlink)
			ext4_orphan_del(NULL, inode);
	}

	return ret ? ret : copied;
}

```

논리 블록을 물리 블록으로 매핑하는 함수 
```c
static int _ext4_get_block(struct inode *inode, sector_t iblock,
			   struct buffer_head *bh, int flags)
{
	struct ext4_map_blocks map;
	int ret = 0;

	if (ext4_has_inline_data(inode))
		return -ERANGE;

	map.m_lblk = iblock;
	map.m_len = bh->b_size >> inode->i_blkbits;

	ret = ext4_map_blocks(ext4_journal_current_handle(), inode, &map,
			      flags);
	if (ret > 0) {
		map_bh(bh, inode->i_sb, map.m_pblk);
		ext4_update_bh_state(bh, map.m_flags);
		bh->b_size = inode->i_sb->s_blocksize * map.m_len;
		ret = 0;
	} else if (ret == 0) {
		/* hole case, need to fill in bh->b_size */
		bh->b_size = inode->i_sb->s_blocksize * map.m_len;
	}
	return ret;
}
```

