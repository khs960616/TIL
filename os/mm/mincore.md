```c
SYSCALL_DEFINE3(mincore, unsigned long, start, size_t, len,
		unsigned char __user *, vec)
{
	long retval;
	unsigned long pages;
	unsigned char *tmp;

	start = untagged_addr(start);

	/* Check the start address: needs to be page-aligned.. */
	if (start & ~PAGE_MASK)
		return -EINVAL;

	/* ..and we need to be passed a valid user-space range */
	if (!access_ok((void __user *) start, len))
		return -ENOMEM;

	/* This also avoids any overflows on PAGE_ALIGN */
	pages = len >> PAGE_SHIFT;
	pages += (offset_in_page(len)) != 0;

	if (!access_ok(vec, pages))
		return -EFAULT;

	tmp = (void *) __get_free_page(GFP_USER);
	if (!tmp)
		return -EAGAIN;

	retval = 0;
	while (pages) {
		/*
		 * Do at most PAGE_SIZE entries per iteration, due to
		 * the temporary buffer size.
		 */
		mmap_read_lock(current->mm);
		retval = do_mincore(start, min(pages, PAGE_SIZE), tmp);
		mmap_read_unlock(current->mm);

		if (retval <= 0)
			break;
		if (copy_to_user(vec, tmp, retval)) {
			retval = -EFAULT;
			break;
		}
		pages -= retval;
		vec += retval;
		start += retval << PAGE_SHIFT;
		retval = 0;
	}
	free_page((unsigned long) tmp);
	return retval;
}
```
