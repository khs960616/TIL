## OOM Killer
- 시스템 메모리가 부족해지는 상황에서 강제로 프로세스를 종료시켜 메모리를 확보시키는 커널의 기능
- 시스템 메모리가 임계 수준 이하로 떨어지는 경우 동작한다.


## oom_badness 및 score 설정 방법 

이는 프로세스의 메모리 사용량, 프로세스 우선순위등을 데몬 내부적으로 사용하여 계산한다.
oom_badness() 함수를 통해 게산하며 0~1000사이 값으로 계산되며 OOM Score가 높을수록 우선순위가 높아진다. 

```c
# mm = memory management 
# RSS(Resident Set Size) = 실제 메모리에 적재된 페이지 수 × 페이지 크기(보통 4KB)
	points = get_mm_rss(p->mm) + get_mm_counter(p->mm, MM_SWAPENTS) +
		mm_pgtables_bytes(p->mm) / PAGE_SIZE;
	task_unlock(p);

	/* Normalize to oom_score_adj units */
	adj *= totalpages / 1000;
	points += adj;

	/*
	 * Never return 0 for an eligible task regardless of the root bonus and
	 * oom_score_adj (oom_score_adj can't be OOM_SCORE_ADJ_MIN here).
	 */
	return points > 0 ? points : 1;
```

```c
# mm_rss는 아래의 값들의 합이다. 
# mm_count는 mm구조체에 존재하는 값들을 atomic하게 읽어오는 function임 
# atomic_long_read(&mm->rss_stat.count[member]);

	return get_mm_counter(mm, MM_FILEPAGES) +
		get_mm_counter(mm, MM_ANONPAGES) +
		get_mm_counter(mm, MM_SHMEMPAGES);
```



oom_badness 메소드의 순위 선정 방법 (코드 분석전 자료 조사) 
1. 특정 프로세스를 죽임으로써, 최소한 양의 프로세스만 잃어야한다.
2. 많은 메모리를 회수할 수 있어야한다.
3. leak이 발생되지 않는 프로세스는 선택하지 않는다.
4. 사용자가 지정한 프로세스순으로 죽인다.

- 모든 프로세스는  /proc/pid/oom_score 에 해달 프로세스의 oom_score가 기록 되어 있다.

## [우선순위 조작방법]
/proc/pid/oom_adj 값을 조정함으로써 OOM Killer의 순위 설정을 조작할 수 있다.
해당 값은 -17~15사이의 값을 가질수 있으며, 낮은 값 일수록 우선순위에서 밀려난다.

/proc/pid/oom_score_adj 값을 조정함으로써 OOM Killer의 순위 설정을 조작할 수 있다.
해당 값은 -1000~1000사이의 값을 가질수 있으며, 낮은 값 일수록 우선순위에서 밀려난다.

## [로그 확인 방법]
OOM Killer가 동작한 경우 /var/log/messages 경로에 oom 관련 메시지가 기록된다.




## [관련 자료]

https://www.kernel.org/doc/gorman/html/understand/understand016.html 
https://elixir.bootlin.com/linux/v5.6/source/mm/oom_kill.c#L199 



---

## mm type 관련 메모

MM_FILEPAGES 
→ 파일을 mmap으로 메모리 매핑해놨을때, 해당 파일의 데이터를 메모리에 로드하면 
file-backed 페이지로 분류된다. (실행중인 바이너리, 공유 라이브러리등)
(/dev/zero로 맵핑하는 경우 여기로 잡히는 듯?)


MM_ANONPAGES
→ malloc()등을 통해 확보한 메모리처럼, 파일과 관련이 없는 메모리.
→ 스왑 아웃되지 않고 현재 물리 메모리에 상주하고 있는 익명 페이지 
→ (mmap써도 파일하고 맵핑하는게 아니라, anonymous 주면 여기로 잡힘) 


MM_SHMEMPAGES
→  SYSV, POSIX 방식으로 생성된 Shared Memory로써, 현재 물리 메모리에 올라가 있는 페이지 수


MM_SWAPENTS
→  swap out된 모든 페이지 개수 (shared memory 구별없음) 

NR_MM_COUNTERS → 이건 그냥 enum 마지막을 뜻하는 녀석


## do mmap 

```c 
/*
 * The caller must write-lock current->mm->mmap_lock.
 */
unsigned long do_mmap(struct file *file, unsigned long addr,
			unsigned long len, unsigned long prot,
			unsigned long flags, vm_flags_t vm_flags,
			unsigned long pgoff, unsigned long *populate,
			struct list_head *uf)
{
	struct mm_struct *mm = current->mm;
	int pkey = 0;

	*populate = 0;

	if (!len)
		return -EINVAL;

	/*
	 * Does the application expect PROT_READ to imply PROT_EXEC?
	 *
	 * (the exception is when the underlying filesystem is noexec
	 *  mounted, in which case we don't add PROT_EXEC.)
	 */
	if ((prot & PROT_READ) && (current->personality & READ_IMPLIES_EXEC))
		if (!(file && path_noexec(&file->f_path)))
			prot |= PROT_EXEC;

	/* force arch specific MAP_FIXED handling in get_unmapped_area */
	if (flags & MAP_FIXED_NOREPLACE)
		flags |= MAP_FIXED;

	if (!(flags & MAP_FIXED))
		addr = round_hint_to_min(addr);

	/* Careful about overflows.. */
	len = PAGE_ALIGN(len);
	if (!len)
		return -ENOMEM;

	/* offset overflow? */
	if ((pgoff + (len >> PAGE_SHIFT)) < pgoff)
		return -EOVERFLOW;

	/* Too many mappings? */
	if (mm->map_count > sysctl_max_map_count)
		return -ENOMEM;

	/*
	 * addr is returned from get_unmapped_area,
	 * There are two cases:
	 * 1> MAP_FIXED == false
	 *	unallocated memory, no need to check sealing.
	 * 1> MAP_FIXED == true
	 *	sealing is checked inside mmap_region when
	 *	do_vmi_munmap is called.
	 */

	if (prot == PROT_EXEC) {
		pkey = execute_only_pkey(mm);
		if (pkey < 0)
			pkey = 0;
	}

	/* Do simple checking here so the lower-level routines won't have
	 * to. we assume access permissions have been handled by the open
	 * of the memory object, so we don't do any here.
	 */
	vm_flags |= calc_vm_prot_bits(prot, pkey) | calc_vm_flag_bits(file, flags) |
			mm->def_flags | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;

	/* Obtain the address to map to. we verify (or select) it and ensure
	 * that it represents a valid section of the address space.
	 */
	addr = __get_unmapped_area(file, addr, len, pgoff, flags, vm_flags);
	if (IS_ERR_VALUE(addr))
		return addr;

	if (flags & MAP_FIXED_NOREPLACE) {
		if (find_vma_intersection(mm, addr, addr + len))
			return -EEXIST;
	}

	if (flags & MAP_LOCKED)
		if (!can_do_mlock())
			return -EPERM;

	if (!mlock_future_ok(mm, vm_flags, len))
		return -EAGAIN;

	if (file) {
		struct inode *inode = file_inode(file);
		unsigned long flags_mask;

		if (!file_mmap_ok(file, inode, pgoff, len))
			return -EOVERFLOW;

		flags_mask = LEGACY_MAP_MASK;
		if (file->f_op->fop_flags & FOP_MMAP_SYNC)
			flags_mask |= MAP_SYNC;

		switch (flags & MAP_TYPE) {
		case MAP_SHARED:
			/*
			 * Force use of MAP_SHARED_VALIDATE with non-legacy
			 * flags. E.g. MAP_SYNC is dangerous to use with
			 * MAP_SHARED as you don't know which consistency model
			 * you will get. We silently ignore unsupported flags
			 * with MAP_SHARED to preserve backward compatibility.
			 */
			flags &= LEGACY_MAP_MASK;
			fallthrough;
		case MAP_SHARED_VALIDATE:
			if (flags & ~flags_mask)
				return -EOPNOTSUPP;
			if (prot & PROT_WRITE) {
				if (!(file->f_mode & FMODE_WRITE))
					return -EACCES;
				if (IS_SWAPFILE(file->f_mapping->host))
					return -ETXTBSY;
			}

			/*
			 * Make sure we don't allow writing to an append-only
			 * file..
			 */
			if (IS_APPEND(inode) && (file->f_mode & FMODE_WRITE))
				return -EACCES;

			vm_flags |= VM_SHARED | VM_MAYSHARE;
			if (!(file->f_mode & FMODE_WRITE))
				vm_flags &= ~(VM_MAYWRITE | VM_SHARED);
			fallthrough;
		case MAP_PRIVATE:
			if (!(file->f_mode & FMODE_READ))
				return -EACCES;
			if (path_noexec(&file->f_path)) {
				if (vm_flags & VM_EXEC)
					return -EPERM;
				vm_flags &= ~VM_MAYEXEC;
			}

			if (!file->f_op->mmap)
				return -ENODEV;
			if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))
				return -EINVAL;
			break;

		default:
			return -EINVAL;
		}
	} else {
		switch (flags & MAP_TYPE) {
		case MAP_SHARED:
			if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))
				return -EINVAL;
			/*
			 * Ignore pgoff.
			 */
			pgoff = 0;
			vm_flags |= VM_SHARED | VM_MAYSHARE;
			break;
		case MAP_DROPPABLE:
			if (VM_DROPPABLE == VM_NONE)
				return -ENOTSUPP;
			/*
			 * A locked or stack area makes no sense to be droppable.
			 *
			 * Also, since droppable pages can just go away at any time
			 * it makes no sense to copy them on fork or dump them.
			 *
			 * And don't attempt to combine with hugetlb for now.
			 */
			if (flags & (MAP_LOCKED | MAP_HUGETLB))
			        return -EINVAL;
			if (vm_flags & (VM_GROWSDOWN | VM_GROWSUP))
			        return -EINVAL;

			vm_flags |= VM_DROPPABLE;

			/*
			 * If the pages can be dropped, then it doesn't make
			 * sense to reserve them.
			 */
			vm_flags |= VM_NORESERVE;

			/*
			 * Likewise, they're volatile enough that they
			 * shouldn't survive forks or coredumps.
			 */
			vm_flags |= VM_WIPEONFORK | VM_DONTDUMP;
			fallthrough;
		case MAP_PRIVATE:
			/*
			 * Set pgoff according to addr for anon_vma.
			 */
			pgoff = addr >> PAGE_SHIFT;
			break;
		default:
			return -EINVAL;
		}
	}

	/*
	 * Set 'VM_NORESERVE' if we should not account for the
	 * memory use of this mapping.
	 */
	if (flags & MAP_NORESERVE) {
		/* We honor MAP_NORESERVE if allowed to overcommit */
		if (sysctl_overcommit_memory != OVERCOMMIT_NEVER)
			vm_flags |= VM_NORESERVE;

		/* hugetlb applies strict overcommit unless MAP_NORESERVE */
		if (file && is_file_hugepages(file))
			vm_flags |= VM_NORESERVE;
	}

	addr = mmap_region(file, addr, len, vm_flags, pgoff, uf);
	if (!IS_ERR_VALUE(addr) &&
	    ((vm_flags & VM_LOCKED) ||
	     (flags & (MAP_POPULATE | MAP_NONBLOCK)) == MAP_POPULATE))
		*populate = len;
	return addr;
}
```


