https://github.com/torvalds/linux/blob/master/arch/x86/mm/tlb.c 
(cpu 따라 코드는 다 다르겠지만 일단 x86기준으로 좀 봐두자)

# Translation Lookaside Buffer
하드웨어적으로 지원하는 일종의 캐시 ((MMU)내부에 존재)  (MMU는 CPU내부에 있거나, 별도 칩으로 존재)
