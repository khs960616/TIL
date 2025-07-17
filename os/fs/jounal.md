https://www.kernel.org/doc/html/latest/filesystems/ext4/journal.html  (자세한 포맷팅등 내용이 필요한 경우 다시 보자)

1. System Crash 발생 시, **파일 시스템** 에 대한 메타데이터의 불일치를 방지하기 위해 저널을 사용한다.
2. 최대 10,240,00개의 블록이 파일 시스템 내부에 저널을 위한 공간으로 예약될 수 있다.
3. 성능상이 이유로 ext4는 default 설정으로는 파일 시스템에 메타데이터만 Journal을 통해 기록한다.
  - 디스크 마운트시 data=ordered (default)
  - data=journal (모든데이터, 메타데이터가 저널을 통해 기록), 
  - data=writeback 사용시 dirty blcok은 메타데이터가 저널을 통해 디스크에 기록되기전에 디스크로 플러시 되지 않음 

약어 jdb2 =  Journaling Block Device version 2 


하나의 page를 기준으로 몇개까지의 파일시스템 기준 블럭이 몇개 들어갈수있는지 체크. 
```c
int jbd2_journal_blocks_per_page(struct inode *inode)
{
	return 1 << (PAGE_SHIFT - inode->i_sb->s_blocksize_bits);
}
```

메타데이터 업데이트에 필요한 저널 블록을 계산하는 함수 정도로 이해하고 넘어가자. 
```
static int ext4_meta_trans_blocks(struct inode *inode, int lblocks,
				  int pextents)
{
	ext4_group_t groups, ngroups = ext4_get_groups_count(inode->i_sb);
	int gdpblocks;
	int idxblocks;
	int ret = 0;

	/*
	 * How many index blocks need to touch to map @lblocks logical blocks
	 * to @pextents physical extents?
	 */
	idxblocks = ext4_index_trans_blocks(inode, lblocks, pextents);

	ret = idxblocks;

	/*
	 * Now let's see how many group bitmaps and group descriptors need
	 * to account
	 */
	groups = idxblocks + pextents;
	gdpblocks = groups;
	if (groups > ngroups)
		groups = ngroups;
	if (groups > EXT4_SB(inode->i_sb)->s_gdb_count)
		gdpblocks = EXT4_SB(inode->i_sb)->s_gdb_count;

	/* bitmaps and block group descriptor blocks */
	ret += groups + gdpblocks;

	/* Blocks for super block, inode, quota and xattr blocks */
	ret += EXT4_META_TRANS_BLOCKS(inode->i_sb);

	return ret;
}

```

// 저널링에 필요한 블록 개수를 return하게 된다 
```c
int ext4_writepage_trans_blocks(struct inode *inode)
{
	int bpp = ext4_journal_blocks_per_page(inode); 
	int ret;

	ret = ext4_meta_trans_blocks(inode, bpp, bpp);

	/* Account for data blocks for journalled mode */
	if (ext4_should_journal_data(inode))  // 마운트 시 옵션에 따라 될수도, 안될수도있음 (default로는 x) 
		ret += bpp;
	return ret;
}
```



