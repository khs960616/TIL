[posix thread attribute](https://github.com/khs960616/TIL/blob/main/os/APUE%20Thread.md)


```c
#include <pthread.h>

int pthread_atrr_getstack(const pthared_attr_t *restrict attr, void **trestrict stackaddr, size_t *restrict stacksize);
int pthread_atrr_setstack(pthared_attr_t *attr, void *stackaddr, size_t stack_size);

# 두 함수 모두 성공 시 0, 실패 시 ERRNO Return
```

-> 사이즈만 조절하고 싶다면, pthared_attr_xxxstacksize()류의 함수를 이용

-> 스레드 stack을 위한 가상 주소 공간이 모자란 경우, 위 pthread_atrr_xxxstack를 사용해서, 

Process는 사용 가능한 virtual address 크기가 고정되어 있고, 스택이 하나뿐이라 스택 크기가 문제가 되는 경우가 없다.

Thread의 경우에는, 모든 Tharead Stack이 동일한 Virtual Address를 공유해야하므로, 

1. Thread Stack의 누적 크기가 가용가능한 Virtual Address 크기를 넘을 위험성이 있는 프로그램 작성시, Thread의 기본 스택 크기를 줄일 필요가 있다.

2. Thread 내부에서 너무 큰 automatic variable를 할당한다거나, |
   함수 호출 연쇄가 깊어서, 스택 프레임이 많이 쌓이는 구조라면, 기본 스택 크기를 늘릴 필요가 있다. 
