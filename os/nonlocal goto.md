## REF 
[setjmp](https://www.csl.mtu.edu/cs4411.ck/www/NOTES/non-local-goto/goto.html) 단순 코드 분기볼정도로는 정리가 나름 잘 되있음 

setjmp, longjmp 둘다 일단 결국엔 machine depedency있는 __setjmp, __longjmp 호출함

https://codebrowser.dev/glibc/glibc/sysdeps/x86_64/__longjmp.S.html   예시 삼아... 하나보면 

```
ENTRY(__longjmp)
	/* Restore registers.  */
	mov (JB_RSP*8)(%rdi),%R8_LP     -> 스택포인터 복구 
	mov (JB_RBP*8)(%rdi),%R9_LP     -> 베이스 포인터 복구 
	mov (JB_PC*8)(%rdi),%RDX_LP     -> PC 복구 
   
   ... 생략 (나머지 애들은 32bit machine에서 추가적으로 해줘야되는 것들이나, 주소 디코딩 (mangling 되있는 경우), 
   Shadow Stack(주소 반환값 보호를 위한거라는데 얘는 뭔지 잘모르겠네.. 나중에 필요하면 별도로 조사해서 내용 추가) 
  
  /* Set return value for setjmp.  */
	mov %esi, %eax             -> val로 받아온 값 설정 
	mov %R8_LP,%RSP_LP
	movq %r9,%rbp
	LIBC_PROBE (longjmp_target, 3,
		    LP_SIZE@%RDI_LP, -4@%eax, LP_SIZE@%RDX_LP)
	jmpq *%rdx
```

이런식으로 분기탐 
