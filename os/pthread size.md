[posix thread attribute](https://github.com/khs960616/TIL/blob/main/os/APUE%20Thread.md)


```c
#include <pthread.h>

int pthread_attr_getstack(const pthared_attr_t *restrict attr, void **trestrict stackaddr, size_t *restrict stacksize);
int pthread_attr_setstack(pthared_attr_t *attr, void *stackaddr, size_t stack_size);

# 두 함수 모두 성공 시 0, 실패 시 ERRNO Return
/*
stack size는 PTHREAD_STACK_MIN 16384byte보다 항상 커야하며, 
시스템에 따라 stack addr, stack addr + stacksize가 aligned이 안맞는 상태면 
에러가 발생할 수 있다. 
stacksize도 page의 배수로 설정이 권장된다.

해당 속성을 이용해서, 스레드 여러개 만들때, pthread_create에서 동일한 값을 반복해서 사용되지 않도록 주의해야한다.
(thread들이 동일한 메모리 영역으로 스택 활용을 시도하게 되므로)

해당 함수 코드보면, 그냥 union으로 도있고 (pthared_attr_t)
 stack사이즈 검증(PTHREAD_STACK_MIN보다 큰지만 검사), stack 시작주소 결정 (grow down인 경우, 스택시작주소를 스택 + stacksize한거로 잡네?)
*/
```

-> 사이즈만 조절하고 싶다면, pthared_attr_xxxstacksize()류의 함수를 이용

-> 스레드 stack을 위한 가상 주소 공간이 모자란 경우, 위 pthread_atrr_xxxstack를 사용해서, 

Process는 사용 가능한 virtual address 크기가 고정되어 있고, 스택이 하나뿐이라 스택 크기가 문제가 되는 경우가 없다.

Thread의 경우에는, 모든 Tharead Stack이 동일한 Virtual Address를 공유해야하므로, 

1. Thread Stack의 누적 크기가 가용가능한 Virtual Address 크기를 넘을 위험성이 있는 프로그램 작성시, Thread의 기본 스택 크기를 줄일 필요가 있다.

2. Thread 내부에서 너무 큰 automatic variable를 할당한다거나,
   함수 호출 연쇄가 깊어서, 스택 프레임이 많이 쌓이는 구조라면, 기본 스택 크기를 늘릴 필요가 있다. 

---

pthread_atrr_setstack함수 사용시, application이 스택 할당에 대한 책임을 가지게 된다. 

해당 함수 사용 이전에, pthread_attr_setguardsize 등으로 가드 사이즈등을 지정한 경우 무시되게된다. 

(따라서 스택 오버플로우 가능성을 처리할 필요가 있다면, application level에서 가드 영역도 알아서 만들어줘야한다.)

---

arm64
```
struct thread_info {
	unsigned long		flags;		/* low level flags */
#ifdef CONFIG_ARM64_SW_TTBR0_PAN
	u64			ttbr0;		/* saved TTBR0_EL1 */
#endif
	union {
		u64		preempt_count;	/* 0 => preemptible, <0 => bug */
		struct {
#ifdef CONFIG_CPU_BIG_ENDIAN
			u32	need_resched;
			u32	count;
#else
			u32	count;
			u32	need_resched;
#endif
		} preempt;
	};
#ifdef CONFIG_SHADOW_CALL_STACK
	void			*scs_base;
	void			*scs_sp;
#endif
	u32			cpu;
};
```

x86
```
struct thread_info {
	unsigned long		flags;		/* low level flags */
	unsigned long		syscall_work;	/* SYSCALL_WORK_ flags */
	u32			status;		/* thread synchronous flags */
#ifdef CONFIG_SMP
	u32			cpu;		/* current CPU */
#endif
};

```

--

아.. 결국 posix에서 이거 어떻게 동작하려는지 보려면

pthread_create -> create_thread -> allocatestack.c 이쪽 코드롤 봐야된다.. 


https://codebrowser.dev/glibc/glibc/nptl/allocatestack.c.html#allocate_stack

-> 별다른 코드는 없다.. 다만 create시에 특성 객체에 stack 시작 주소 넘겼는지, 안넘겼는지에 따라 코드 분기가 달라진다. 

시작 주소를 준 경우에는 mmap이나, guard page설정등의 작업을 일절하지 않는다. (사용자 코드에서 알아서 하는 것을 전제로함)

이 내부에서도 
```
tls_static_size_for_stack 

/* Minimal stack size after allocating thread descriptor and guard size.  */
#define MINIMAL_REST_STACK 2048
```
아래 두가지값으로 stack size 검증한번 더 함. 



pthread (thread descriptor)객체가 thread의 stack공간에 위치할곳을, pthread struct 만큼 0으로 초기화 해놓는다.
(이때 call쪽에서 달아둔 pdp포인터에, 이 tcb의 위치를 저장해서 pthread_create등의 함수에서 해당 위치를 알수있게해준다.)
(tcb에 stackblock, stack size등은 allocated stack안에서 설정해준다) 

// TLS_TCB_AT_TP 켜져있는 경우, pthread->header.tcb에  tls를 위해 pd객체 자체를 달아놓는데,
// TLS_DTV_AT_TP인 경우 애초에 tcbhead_t 자체를 두지 않고 별도 header 객체를 가진다. 

// tls쪽 코드 봤을땐 두개를 완전하게 동일한 개념으로 보면 안될듯하다. 

여기쪽 코드가 grow down, up인 경우에 따라 코드 분기가 갈리고, 

grow down인 경우 시작 주소 넣어주면, 알아서 stackaddr이 스택의 시작점이 되게끔 (메모리상에서의 시작점이 아닌, 자라나는 방향 기준으로의 시작점)

#if _STACK_GROWS_DOWN
  iattr->stackaddr = (char *) stackaddr + stacksize;
#else 

attr설정하는 클래스들 코드쪽에 있어서, 매우 헷갈리니 주의해서 보자. 

추가로 aix에서는 mmap한 주소 넘기면 터진다 (aix 자체가 mmap한 메모리랑 io관련하여 이슈가 있는듯하고, 
stack 공간 잡아서 넘길땐 malloc해서 얻은 공간을 넘기는 식으로 되어있다. 따라서 이 공간은 mincore로 추적도 안되고 매우 짜증나게 되있음)
