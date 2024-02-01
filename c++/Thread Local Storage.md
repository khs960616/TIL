## Thread Local Storage
각 스레드별로 독립적으로 가지는 메모리 공간 (이하 tls로 줄임)
(data 영역은 하나의 프로세스 내에서 모두 접근이 가능하므로, Thread를 쓰는 코드에서, 스레드 내부에서만 전역적으로 쓰고자하는 값을 thread local storage를 이용한다)

tls 공간을 사용하기 위해서는 __declspec(thread) 또는 C++11 이상부터는 thread_local 변수를 활용한다.

thread_local로 선언한 변수는, 하나의 스레드를 scope로 하며, 스레드 내에서 접근 가능하다. (static등의 예약어와도 함께 사용 가능)


#### Ref

https://learn.microsoft.com/ko-kr/windows/win32/procthread/thread-local-storage?redirectedfrom=MSDN
