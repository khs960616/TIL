systemv  shm 생성 & shmat시

- SHM_HUGETLB에 따른 분기  (5.14 기준) 

메모리 사용량 관련 이슈 분석하다, nr_hugepages로 reserve해두고 hugetlb flag로 만든 shm의 경우 

task dump상에 남은 rss값에 shm값이 미포함되는 것 같아, 확인 및 메모차 남김

일반적으로 page fault 발생 시 do_fault에서 vma에 종속적인 vm op->fault를 호출하고

페이지 할당 성공 시 공통적으로 finish_fault를 호출하며 mm counter값을 올려주게 된다.

그러나  hugetlb에서는 별도 루틴으로 넘어가서  vma op를 사용하지도 않으며, mm counter를 올려주지 않는 것으로 보인다. 

```c
	if (shmflg & SHM_HUGETLB) {
		struct hstate *hs;
		size_t hugesize;

		hs = hstate_sizelog((shmflg >> SHM_HUGE_SHIFT) & SHM_HUGE_MASK);
		if (!hs) {
			error = -EINVAL;
			goto no_file;
		}
		hugesize = ALIGN(size, huge_page_size(hs));

		/* hugetlb_file_setup applies strict accounting */
		if (shmflg & SHM_NORESERVE)
			acctflag = VM_NORESERVE;
		file = hugetlb_file_setup(name, hugesize, acctflag,
				  &shp->mlock_ucounts, HUGETLB_SHMFS_INODE,
				(shmflg >> SHM_HUGE_SHIFT) & SHM_HUGE_MASK);
	} else {
		/*
		 * Do not allow no accounting for OVERCOMMIT_NEVER, even
		 * if it's asked for.
		 */
		if  ((shmflg & SHM_NORESERVE) &&
				sysctl_overcommit_memory != OVERCOMMIT_NEVER)
			acctflag = VM_NORESERVE;
		file = shmem_kernel_file_setup(name, size, acctflag);
	}
 
/*  hugetlb_file_setup을 보면 아래와 같으며 이러면 
		file = alloc_file_pseudo(inode, mnt, name, O_RDWR,
					&hugetlbfs_file_operations);
     
        alloc_file이 불리며 해당 함수 내부에서 fop를 
        hugetlbfs_file_operations로 만들게된다. 
        
        // SHM_HUGETLB가 아닌 케이스에는 
        file_operations shmem_file_operations;
*/ 
 

long do_shmat(int shmid, char __user *shmaddr, int shmflg,
	          ulong *raddr, unsigned long shmlba)
{
    unsigned long flags = MAP_SHARED;
    ... 
    
	base = get_file(shp->shm_file);
	shp->shm_nattch++;
	size = i_size_read(file_inode(base));
	ipc_unlock_object(&shp->shm_perm);
	rcu_read_unlock();

	err = -ENOMEM;
	sfd = kzalloc(sizeof(*sfd), GFP_KERNEL);
	if (!sfd) {
		fput(base);
		goto out_nattch;
	}
 
    // base의 huge 여부에 따라서 file의 op를 다르게 넣지만, 5.14 기준 동일해보임 
	file = alloc_file_clone(base, f_flags,
			  is_file_hugepages(base) ?
				&shm_file_operations_huge :
				&shm_file_operations);
    
    
	err = PTR_ERR(file);
	if (IS_ERR(file)) {
		kfree(sfd);
		fput(base);
		goto out_nattch;
	}

	sfd->id = shp->shm_perm.id;
	sfd->ns = get_ipc_ns(ns);
	sfd->file = base;
	sfd->vm_ops = NULL;
	file->private_data = sfd;
   ... 
   
    addr = do_mmap(file, addr, size, prot, flags, 0, &populate, NULL);
   ...
}

/*참고용
 static const struct file_operations shm_file_operations_huge = {
	.mmap		= shm_mmap,
	.fsync		= shm_fsync,
	.release	= shm_release,
	.get_unmapped_area	= shm_get_unmapped_area,
	.llseek		= noop_llseek,
	.fallocate	= shm_fallocate,
};

 
 static const struct file_operations shm_file_operations = {
	.mmap		= shm_mmap,
	.fsync		= shm_fsync,
	.release	= shm_release,
	.get_unmapped_area	= shm_get_unmapped_area,
	.llseek		= noop_llseek,
	.fallocate	= shm_fallocate,
};
 */


unsigned long do_mmap(struct file *file, unsigned long addr,
			unsigned long len, unsigned long prot,
			unsigned long flags, unsigned long pgoff,
			unsigned long *populate, struct list_head *uf)
{
    .. 중략 
	/* Do simple checking here so the lower-level routines won't have
	 * to. we assume access permissions have been handled by the open
	 * of the memory object, so we don't do any here.
	 */
	vm_flags = calc_vm_prot_bits(prot, pkey) | calc_vm_flag_bits(flags) |
			mm->def_flags | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;
   
   
   ....
   
   mmap_region(...);
   
}

unsigned long mmap_region(struct file *file, unsigned long addr,
		unsigned long len, vm_flags_t vm_flags, unsigned long pgoff,
		struct list_head *uf) {
  
   ...
   error = call_mmap(file, vma);
}


static int shm_mmap(struct file *file, struct vm_area_struct *vma)
{
	struct shm_file_data *sfd = shm_file_data(file);
	int ret;

	/*
	 * In case of remap_file_pages() emulation, the file can represent an
	 * IPC ID that was removed, and possibly even reused by another shm
	 * segment already.  Propagate this case as an error to caller.
	 */
	ret = __shm_open(vma);
	if (ret)
		return ret;
    // case에 따라 hugetlbfs_file_operations mmap 호출되며, 
                   
    // vma->vm_flags |= VM_HUGETLB | VM_DONTEXPAND;
	// vma->vm_ops = &hugetlb_vm_ops;   형태로 지정되게 된다.
	ret = call_mmap(sfd->file, vma);
	if (ret) {
		shm_close(vma);
		return ret;
	}
  
    // sfd의 vm_op는 
    // huge_tlb_vm_ops(SHM_HUGETLB) 또는 shmem_vm_ops 호출됨 
	sfd->vm_ops = vma->vm_ops;  
#ifdef CONFIG_MMU
	WARN_ON(!sfd->vm_ops->fault);
#endif
	vma->vm_ops = &shm_vm_ops;
	return 0;
}

// 이러면 sfd->vm_ops->fault 호출
static vm_fault_t shm_fault(struct vm_fault *vmf)
{
	struct file *file = vmf->vma->vm_file;
	struct shm_file_data *sfd = shm_file_data(file);

	return sfd->vm_ops->fault(vmf);
}

// 결론적으로 pagefault 발생 시, shmem_fault | hugetlb_vm_op_fault 를 타게됨 

// do_page_fault
    ->__handle_mm_fault->handle_pte_fault
    -> vma->vm_ops->fault(vmf) (shmem_fault 수행)
    -> finish_fault(vmf) 
// finish_fault rss증가치에 영향

// hugetlb_vm_op_fault의 경우 애초에 page fault 시점에 vma에 대한 op 수행을 하지않고
// flag 체크 후 별도 fault를 불러주기 때문에 호출될 수 없음 
static vm_fault_t hugetlb_vm_op_fault(struct vm_fault *vmf)
{
	BUG();
	return 0;
}

// do_fault 루틴에서 vma에 대한 flag 체크 후 아래로 전달
// 별도로 task관련 메모리 집계에 영향을 주지 않음 
ret = hugetlb_fault(vma->vm_mm, vma, address, flags);
```
