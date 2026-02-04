https://ko.wikipedia.org/wiki/Procfs   (procfs내용부터 보고 오기) 

https://man7.org/linux/man-pages/man5/proc_pid_smaps.5.html 

/proc/<pid>/smaps는 하나의 프로세스가 보유한 각 VMA(Virtual Memory Area) 별 메모리 사용량을 상세하게 보여준다. 

같은 정보를 요약해서 보여주는 도구가 pmap(1)이 있으며, 

smaps는 그 원천 데이터에 가깝다. 

https://elixir.bootlin.com/linux/v5.14/source/fs/proc/task_mmu.c   

```c
const struct file_operations proc_pid_smaps_operations = {
	.open		= pid_smaps_open,
	.read		= seq_read,
	.llseek		= seq_lseek,
	.release	= proc_map_release,
};
```


cat /proc/[pid]/smaps

-> open("/proc/[pid]/smaps", O_RDONLY) 
* /proc : procfs로 마운트된 가상 파일시스템
* pid : 숫자 디렉토리 → “PID 디렉토리”
* smaps : 해당 pid에 대한 smaps 파일

fs/proc/base.c 쪽 코드를 참고 
```c
struct dentry *proc_pid_lookup(struct dentry *dentry, unsigned int flags)
{
	struct task_struct *task;
	unsigned tgid;
	struct proc_fs_info *fs_info;
	struct pid_namespace *ns;
	struct dentry *result = ERR_PTR(-ENOENT);

	tgid = name_to_int(&dentry->d_name);
	if (tgid == ~0U)
		goto out;

	fs_info = proc_sb_info(dentry->d_sb);
	ns = fs_info->pid_ns;
	rcu_read_lock();
	task = find_task_by_pid_ns(tgid, ns);
	if (task)
		get_task_struct(task);
	rcu_read_unlock();
	if (!task)
		goto out;

	/* Limit procfs to only ptraceable tasks */
	if (fs_info->hide_pid == HIDEPID_NOT_PTRACEABLE) {
		if (!has_pid_permissions(fs_info, task, HIDEPID_NO_ACCESS))
			goto out_put_task;
	}

	result = proc_pid_instantiate(dentry, task, NULL);
out_put_task:
	put_task_struct(task);
out:
	return result;
}
```


seq_op (task_mmu.c 에 있음) 
```c
static const struct seq_operations proc_pid_maps_op = {
	.start	= m_start,
	.next	= m_next,
	.stop	= m_stop,
	.show	= show_map
};

// 결국에 타고타고 들어가다보면 이거임 
static int show_map(struct seq_file *m, void *v)
{
	show_map_vma(m, v);
	return 0;
}
```


// 아래 녀석들은 참고만하고 결론적으로는  show_map_vma 보면됨 
```c
static int pid_smaps_open(struct inode *inode, struct file *file)
{
	return do_maps_open(inode, file, &proc_pid_smaps_op);
}

static int do_maps_open(struct inode *inode, struct file *file,
			const struct seq_operations *ops)
{
	return proc_maps_open(inode, file, ops,
				sizeof(struct proc_maps_private));
}

static int proc_maps_open(struct inode *inode, struct file *file,
			const struct seq_operations *ops, int psize)
{
	struct proc_maps_private *priv = __seq_open_private(file, ops, psize);

	if (!priv)
		return -ENOMEM;

	priv->inode = inode;
	priv->mm = proc_mem_open(inode, PTRACE_MODE_READ);
	if (IS_ERR(priv->mm)) {
		int err = PTR_ERR(priv->mm);

		seq_release_private(inode, file);
		return err;
	}

	return 0;
}
```


```c
static void
show_map_vma(struct seq_file *m, struct vm_area_struct *vma)
{
	struct mm_struct *mm = vma->vm_mm;
	struct file *file = vma->vm_file;
	vm_flags_t flags = vma->vm_flags;
	unsigned long ino = 0;
	unsigned long long pgoff = 0;
	unsigned long start, end;
	dev_t dev = 0;
	const char *name = NULL;

	if (file) {
		struct inode *inode = file_inode(vma->vm_file);
		dev = inode->i_sb->s_dev;
		ino = inode->i_ino;
		pgoff = ((loff_t)vma->vm_pgoff) << PAGE_SHIFT;
	}

	start = vma->vm_start;
	end = vma->vm_end;
	show_vma_header_prefix(m, start, end, flags, pgoff, dev, ino);

	/*
	 * Print the dentry name for named mappings, and a
	 * special [heap] marker for the heap:
	 */
	if (file) {
		seq_pad(m, ' ');
		seq_file_path(m, file, "\n");
		goto done;
	}

	if (vma->vm_ops && vma->vm_ops->name) {
		name = vma->vm_ops->name(vma);
		if (name)
			goto done;
	}

	name = arch_vma_name(vma);
	if (!name) {
		if (!mm) {
			name = "[vdso]";
			goto done;
		}

    /*  heap으로 마킹되는 조건 !  */
		if (vma->vm_start <= mm->brk &&
		    vma->vm_end >= mm->start_brk) {
			name = "[heap]";
			goto done;
		}

		if (is_stack(vma))
			name = "[stack]";
	}

done:
	if (name) {
		seq_pad(m, ' ');
		seq_puts(m, name);
	}
	seq_putc(m, '\n');
}
```

---

mm->start_brk    프로세스의 brk 힙 시작 주소입니다. 한 번 정해지면 프로세스 수명 동안 변하지 않음.

mm->brk  현재 brk 힙의 끝 주소, brk()/sbrk()에 의해 앞뒤로 이동 가능 


---

PSS(Proportional Set Size): 공유 메모리를 여러 프로세스가 나눠 가질 때 실제 해당 프로세스가 차지하는 메모리를 계산 
