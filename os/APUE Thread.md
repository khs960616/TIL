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
pthread 인터페이스는 스레드 객체를 세밀하게 조율할 수 있는 수단을 제공한다. 
각 객체에 연관된 여러 attribute들이 그러한 수단이다. 
이러한 attribute는 thread말고도, 여러 객체들이 존재하며, 각 객체들은 대체로 아래와 같은 패턴을 따른다. 

1. 하나의 특성 객체가 여러 개의 특성을 대표할 수 있다. 
2. 특성들을 기본값으로 설정하는 초기화 함수가 존재한다.
3. 특성객체를 파괴하는 또 다른 함수가 존재하며, 초기화 함수가 특성 객체와 관련해서 자원을 할당했다면, 소멸자에서 이것을 해제한다.
4. 각 특성마다 특성 객체로부터 특성값을 조회하는 함수가 존재한다. (return값은 인수를 통해 return되며, 실제 return value는 성공시0, 실패시 errno)
5. 각 특성마다 특성의 값을 설정하는 함수가 존재하며, 이 경우 호출자는 그 값을 value방식의 인수를 통해 지정한다.



```c
#include <pthread.h>

int pthared_attr_init(pthread_attr_t *attr);     # 구현에서 지원하는 thread의 모든 속성의 기본값이 atrr 객체에 설정된다. 
int pthared_attr_destory(pthared_attr_t *attr);  # 특성 객체를 유효하지 않은 값으로 초기화 시켜, 재사용시 오류가 나게끔한다. 
```

스레드 속성
detachstate : 탈착된 스레드의 특성
guardsize : 스레드 스택 끝의 보호 버퍼 크기(바이트)
stackaddr : 스레드 스택의 최하위 주소 
stacksize : 스레드 스택의 최소 크기


---

여기서부터는 apue에서 다루는 내용은 아니고, glibc pthread create에 달려있는 주석보면서 정리한 것임.. 차후 참고하자.

---

#### struct pthread 참고

https://codebrowser.dev/glibc/glibc/nptl/descr.h.html#pthread 

pthread관련 함수들에서 pthread의 owner가 누구인지 아는 것은 중요하다. 
pthread_create, pthread_join, pthread_detach 및 기타 pthread 구조체를 조작하는 함수들의 정확한 동작을 결정하는 데 도움이된다.
pthread 구조체의 owner는, 구조체와 관련한 자원을 해제해야되는 책임이 있다.


pthread_create함수 호출 시, 해당 함수를 호출하는 스레드를 creating thread라 부르며, 
초기에는 creating thread가 pthread 구조체의 owner가 된다. 
(생성되는 스레드는 초기화 과정에서 owner와 협력하며 해당 구조체를 볼 수 있다.)

이 pthread 구조체의 owner는 다음 4가지 케이스에서 옮겨질 수 있다.

1. pthread_create로 새롭게 생성되는 thread가 joinable상태로 시작하는 경우 (pthread_create 호출이 완료되고, 유효한 pthread_t를 return하는 경우)
   -> pthread의 소유권이 프로세스단위로 풀린다. (모든 스레드에서 사용 가능)

2.  새 스레드가 detached상태로 시작하게 되면, pthread소유권은 해당 스레드에게 넘어간다. 

3. pthread_detach를 호출하는 경우 (실행 중인 스레드로 동적 소유권 이전)

4. pthread_join을 호출하는 경우 (pthread_join을 호출한 스레드가 소유권 획득)

---


