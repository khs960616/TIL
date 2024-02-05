```c++
#include <pthread.h>

int pthread_setcancelstate(int state, int *oldstate);
int pthread_setcanceltype(int type, int *oldtype);
```
하나의 쓰레드에서 실행중인 다른 쓰레드를 종료하기 위한 목적으로 사용된다

**state**
```
PTHREAD_CANCEL_ENABLE: 스레드 취소요청이 온 경우, 취소 지점까지 thread에서 동작을 진행하고, 종료한다.

PTHREAD_CANCEL_DISABLE: 취소 요청이와도 스레드가 종료되지 않는다. 
```

oldstate, oldtype -> 스레드에 대해서 setter를 동작하기 이전의 속상값을 받아오귀 위해 사용하며, 필요없는 경우 nullptr 넣어도 무관

**type**
```
PTHREAD_CANCEL_ASYNCHRONOUS: 취소 요청이 들어오는 경우 바로 종료한다.
PTHREAD_CANCEL_DEFERRED: 취소 지점을 벗어날때까지 기다린다.
```
