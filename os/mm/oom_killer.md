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
해당 값은 -17~15사이의 값을 가질수 있으며, 낮은 값 일수록 우선순위에서 밀려난다.   << 2.6.36 이후버전부터는 아래 것을 참조.. 

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

/*
   file: 매핑할 파일을 가리키는 포인터, 익명 메모리 mapping인 경우 NULL로 들어온다.
   addr: 명시적으로 가상 메모리 주소를 넘겨준 경우에 사용되며, 넘겨주지 않은 경우 NULL로 들어오며, 커널레벨에서 알아서 virtual memory 상에 mmaping한다.
   len: 할당할 사이즈 
   prot: 권한 (PROT_READ, PROT_WRITE, PROT_EXEC, PROT_NONE) 등 호출 시 넘겨준 정보
   flags: MAP_SHARED, MAP_PRIVATE, MAP_FIXED 등 호출시 넘겨줬던 flag정보 
   vm_flags: 맵핑에 대한 가상메모리 정보라 함, 뭔지는 나중에 더 봐야알듯.. 
   pgoff: 주로 파일 맵핑하는 경우, 맵핑할 오프셋(파일에서의) 을 설정할때 사용됨
   populate: 맵핑하면서 페이지를 미리 할당할지 여부를 결정하는 인자 (MAP_POPULATE 와 연관된 녀석인듯 리눅스 2.6.23부터 지원 시작) 
   uf: 뭔지 모르겠음 
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

        // 크기를 0으로 주는 경우 오류 발생 (음수는?) 
	if (!len)
		return -EINVAL;

	/*
	 * Does the application expect PROT_READ to imply PROT_EXEC?
	 *
	 * (the exception is when the underlying filesystem is noexec
	 *  mounted, in which case we don't add PROT_EXEC.)
	 */
        // 읽기 권한 설정 && 읽기 권한 -> 실행권한으로 확장 가능하며 && file이 주어졌고, file이 올바른 경로로 주어진 경우 권한을 추가해준다.
	if ((prot & PROT_READ) && (current->personality & READ_IMPLIES_EXEC))
		if (!(file && path_noexec(&file->f_path)))
			prot |= PROT_EXEC;

        /*MAP_FIXED vs MAP_FIXED_NOREPLACE  
        MAP_FIXED: 주소를 넘겨준 경우 무조건 해당 virtual address상에 새롭게 맵핑한다. (기존 영역이 존재했다면 강제로 밀어버림)
        MAP_FIXED_NOREPLACE: 지정된 주소에 이미 매핑이 있으면 실패하고, ENOMEM을 반환한다. 
        */

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

redhet기준 (버전 몇이지... 분석전에좀 알려주면 좋겠다...) 

os message상에  oom killer가 동작했을때 프로세스들에 대한 정보가 다 떨어져있음.

다만, rss나 swap의 경우 shared memory도 포함하기 때문에 완벽하게 정확하게 사용량을 유추하긴 어렵긴하지만, 대략적으로 계산은 가능함 

(oom_badness의 경우 rss, swap, pgtable_bytes 다 찍혀있어서 계산하기 편함) 

2.6.36 버전까지는 또 다르네... 다시 봐야할듯 


```
메모리가 부족한 경우, 종료시키기 좋은 작업을 선택하여 죽인다.

선별의 기준 

이미 수행된 작업을 가능한 적게 잃는 것
많은 양의 메모리를 회수하는 것
과도한 메모리를 사용하지 않은 무고한 프로세스를 죽이지 않는 것
가능한 한 적은 수의 프로세스(이상적으로는 하나)를 종료하는 것
사용자가 종료되기를 기대하는 프로세스를 종료하려고 노력하는 것


1. 프로세스의 가상 메모리 크기를 점수 선별의 기준으로 잡는다.
2. PF_OOM_ORIGIN 플래그가 켜져있는 프로세스는 최우선으로 죽인다.  (ULONG_MAX 값 리턴)
3. 자식 프로세스가  독립된 메모리 공간을 가지고 있다면, 해당 vm의 절반값을   부모 프로세스의 점수에 추가 
4. cpu time과 runtime의 sqrt값만큼 기존 점수를 나눈다.   (오래 실행되거나, cpu를 많이 사용한 프로세스는 중요 프로세스로 간주)
5. 프로세스에 설정된 nice값 이 0이상이면, 점수를 2배로 올린다.   (우선순위가 낮은 프로세스니까 더 죽이려함)
6. CAP_SYS_ADMIN, CAP_SYS_RESOURCE 권한을 가진 프로세스인 경우 점수를 4로 나눔 
7. CAP_SYS_RAWIO 권한을 가진 프로세스면 점수를 또 4로 나눔
8. 프로세스가 접근 가능한 메모리 노드와 oom이 발생한 노드 사이에 겹치는 부분이 있다면 점수를 유지하고 겹치지 않는다면 8로 나눈다 . 
   (numa 기능 켰을때, memory bank(node)와 프로세스가 접근가능한 노드가 겹치는지 여부)
    (안켰다면 항상 true라서 8로 안나눔) 
9. oom_adj값으로 값을 보정  
```

아 == 또 4.19부터는 
~~oom_evaluate_task이거로 쓰는거같음 == 다시 코드 검토하고 내용 추가하자.~~
어쩌피 그 안에서 badness로 score 매기네, 더 디테일한 부분은 필요할때 더 보자.  


참고 (5.14 기준 dump_task 내용) 

```c
static int dump_task(struct task_struct *p, void *arg)
{
	struct oom_control *oc = arg;
	struct task_struct *task;

	if (oom_unkillable_task(p))
		return 0;

	/* p may not have freeable memory in nodemask */
	if (!is_memcg_oom(oc) && !oom_cpuset_eligible(p, oc))
		return 0;

	task = find_lock_task_mm(p);
	if (!task) {
		/*
		 * All of p's threads have already detached their mm's. There's
		 * no need to report them; they can't be oom killed anyway.
		 */
		return 0;
	}

	pr_info("[%7d] %5d %5d %8lu %8lu %8ld %8lu         %5hd %s\n",
		task->pid, from_kuid(&init_user_ns, task_uid(task)),
		task->tgid, task->mm->total_vm, get_mm_rss(task->mm),
		mm_pgtables_bytes(task->mm),
		get_mm_counter(task->mm, MM_SWAPENTS),
		task->signal->oom_score_adj, task->comm);
	task_unlock(task);

	return 0;
}
```
