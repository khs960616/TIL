APUE11, APUE12 

스레드 생성 & 스레드 속성등 일부 시스템콜과 기억해두어야할 것을 잠시 메모하기 위해 작성. 

## Thread 
다중 스레드를 사용 고려 

1. 비동기적인 작업 코드 단순화가 가능한 경우 
   복잡한 작업을 각 종류의 사건마다 개별적인 스레드를 배정, 각 스레드에서는 동기적으로 이를 처리함으로써 구조를 단순화 시킬 필요가 있는 경우

2. 공유자원 접근 편의성을 고려해야할 때
   (프로세스로 구성 시, FD & Memory 공유해서 사용하고 싶다면, OS여러 기능 활용해야되지만, Thread로 작업 구성 시, 자원 공유가 비교적 간단)

3. Throughput 증가를 위해 
   (단, 이 경우 각 Thread의 작업이 interleaved하게 처리될 수 있는 경우에만 의미가 있다.)

4. 사용자 입력 * 출력 처리와 그 외의 부분을 분리함으로써 반응시간 개선이 필요한 경우

---

## Execution Context

하나의 스레드는, 프로세스 내부에서 스레드ID, 레즈스터값, 스택, 스케쥴링 우선순위 & 스케쥴링 방식, errno 등의 정보로 구성되있으며,
이는 Execution Context를 나타내기 충분한 정보로 구성된다.

---

```c
#include <pthread.h>

int pthread_equal(pthread_t tid1, pthread_t tid2); 
# return value는, 두 스레드가 동일하지 않은 경우 0, 동일한 경우 0이 아닌 값
```

구현에 따라 인터페이스가 pthread_t 타입을 구조체로 두는 것을 허용하므로, 
동일 Thread인지 여부 판단을 위해서는 위 함수 사용해야한다.

(2017-2022 pthreadtypes.h 기준으로는 unsigned long int로 정의되어있다.) 
(구현 시 참고하며, 타입 정의만 보고 정수형 비교로 사용하는 코드들은, 의도적으로 쓴 것인지 파악 후, 아니라면 제거)

```c
#include <pthread.h>

pthread_t pthread_self(void); # 호출한 스레드의 스레드 ID를 return
```
구현에 따라 thread id가 필요한 경우, 위 함수를 호출해서 사용.


## pthread 생성

```c
#include <pthread.h>

int pthread_create(pthread_t *restrict tidp, const pthread_attr_t *restrict attr, void* *(start_rtn)(void*), void *resrcit arg); # 성공 시 0, 실패 시 errno
```

호출 성공 시 tidp에, 현재 생성된 pthread의 pthread_t값 저장, attr은 pthread 생성 시 여러 특성을 설정하기 위해 사용된다. (NULL)로 지정 시 default 속성으로 설정됨

뒤에 두가지 인자는 pthread 생성 시 thread가 실행을 시작할 함수의 주소와, 그 함수에 넣어줄 인자이다. 

(start_rtn == start routine)


## Thread Limit 

Thread를 구성할 때 참고해야되는 값들 (getconf로 조회할때는 pthread대신 _SC_THRAED_)를 줘서 조회해볼수있다. (os에 따라 조회 안될수도?)

PTHREAD_DESRUCTOR_ITERATIONS : 스레드 종료 시, 해당 스레드 제거를 시도하는 최대 횟수 (필요할때 좀더 조사)  
PTHREAD_KEYS_MAX: 하나의 프로세스가 생성할 수 있는 키들의 최대 갯수   (필요할때 좀더 조사)  
PTHRAED_STACK_MIN : 스레드 스택의 최소크기 (바이트 단위)
PTHRED_THREADS_MAX: (하나의 프로세스가 생성 가능한 스레드의 최대갯수)


## Thread Attr
