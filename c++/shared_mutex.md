## Shared Mutex

https://gcc.gnu.org/onlinedocs/gcc-7.1.0/libstdc++/api/a01589_source.html

stdc++에서 제공하며, lock을 exclusive하게 잡을 수도 있고, 공유가 가능한 상태로도 잡을 수 있도록 지원하는 mutex이다.

RAII 지원 클래스들하고 같이 썼을때, unique_lock, lock_guard로 잡으면 lock으로 잡고, shared_lock으로 잡으면 shared_lock으로 잡는다. 

---
정리 목적 

1) mutex 자체가 왜 lock 매커니즘 방법에서 무거운 편인가
-> 실제로 atomic한 instruction들이 내부적으로 많이 수행되야된다고 봤는데 mutex쪽 구현보면서 (_base쪽) low레벨에서 성능좀 생각해보자. 

2) shared_mutex는 shard 상태를 어떤 방식으로 관리하는지, c++버전 올라가면서 shared_lock -> exclusive lock으로 boost처럼 lock을 잡은 상태에서
lock level을 올릴 수 있는 매커니즘을 자체적으로 지원하는지 찾아보고 정리 
