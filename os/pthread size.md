[posix thread attribute](https://github.com/khs960616/TIL/blob/main/os/APUE%20Thread.md)


```c
#include <pthread.h>

int pthread_atrr_getstack(const pthared_attr_t *restrict attr, void **trestrict stackaddr, size_t *restrict stacksize);
int pthread_atrr_setstack(pthared_attr_t *attr, void *stackaddr, size_t stack_size);

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
