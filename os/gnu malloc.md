## https://sourceware.org/glibc/manual/2.34/html_mono/libc.html#Memory

GNU C Library의 malloc 구현은 ptmalloc(pthreads malloc)에서 파생되었으며, 
ptmalloc은 dlmalloc(Doug Lea malloc)에서 파생되었다. 

이 malloc은 할당 크기와 사용자가 제어할 수 있는 특정 파라미터에 따라 두 가지 서로 다른 방식으로 메모리를 할당할 수 있다.

가장 일반적인 방식은 하나의 큰 연속 메모리 영역에서 메모리의 일부(청크라고 불림)를 할당하고, 
사용 불가능한 청크 형태의 낭비를 줄이기 위해 이러한 영역들을 관리하여 사용을 최적화하는 것이다. 

전통적으로 시스템 힙은 하나의 큰 메모리 영역으로 설정되었지만, 
GNU C Library의 malloc 구현은 멀티스레드 애플리케이션에서의 사용을 최적화하기 위해 이러한 영역을 여러 개 유지한다. 이러한 각 영역은 내부적으로 arena라고 불린다.


다른 버전들과 달리, GNU C Library의 malloc은 큰 크기이든 작은 크기이든 청크 크기를
2의 거듭제곱으로 올림하지 않는다. 인접한 청크들은 크기와 무관하게 free 시점에 병합될 수 있다. 
이로 인해 이 구현은 단편화로 인한 높은 메모리 낭비를 일반적으로 발생시키지 않으면서도, 
모든 종류의 할당 패턴에 적합하다. 여러 개의 arena가 존재함으로써, 
여러 스레드가 각각 분리된 arena에서 동시에 메모리를 할당할 수 있어 성능이 향상된다.


메모리 할당의 또 다른 방식은 페이지 크기보다 훨씬 큰 매우 큰 블록에 대한 경우이다. 
이러한 요청은 mmap(익명 매핑 또는 /dev/zero를 통한 매핑; Memory-mapped I/O 참조)을 사용하여 할당된다. 

이 방식의 큰 장점은 해당 청크들이 free될 때 즉시 시스템에 반환된다는 점이다. 
따라서 큰 청크가 더 작은 청크들 사이에 “(lock)” 상태가 되어, free를 호출한 이후에도 메모리를 낭비하는 상황이 생기지 않는다.  



## https://sourceware.org/glibc/wiki/MallocInternals 


### 용어 정리 

1. Arena

하나 이상의 스레드 간에 공유되는 구조체로, 

하나 이상의 힙에 대한 참조와 해당 힙안에 존재하는 “free” 상태의 청크들에 대한 연결 리스트들을 포함한다.

각 arena에 할당된 스레드들은 해당 arena의 free 리스트로부터 메모리를 할당받는다.

2. Heap
   
할당을 위해 청크들로 분할되는 연속적인 메모리 영역, 각 힙은 정확히 하나의 arena에 속하게 된다.

3. Chunk

작은 범위의 연속된 메모리로써, 실제 유저 어플리케이션에 전달되는 wrapper (할당, 해제, 인접 청크들과 병합이 가능한 단위) 

각 청크는 하나의 힙 안에 존재하며 하나의 arena에 속한다. 

4. Memory

어플리케이션 address space의 일부 

---

chunk 구조 

- in use chunk 

실제 유저 공간에서 바라보는 데이터 외에도 청크의 사이즈 정보와 일부 메타정보를 나타내는 Flag를 가진다. 

| size    | LSB(3) - (A|M|P)| 

-> A: 메인 아레나에 소속된 chunk인지 여부 (0이면 main 아레나 소속), M: chunk자체가 특정 heap에 속한 것이 아니라, mmap으로 할당된 것인지 여부, P: 이전 chunk가 사용중인지 여부 


- free chunk

fwd, bck (아마 기억상 bin에 달려있을때 다른 free chunk들과 연결해놓기 위한 포인터)

(prev_size: free상태일때는 payload공간으로 나가는 맨 마지막쪽 (다른 chunk와 인접한 위치)에 자기 자신의 chunk size를 써둔다)

fd_nextsize / bk_nextsize  (large bin에는 크기 순서로 정렬 안되있어서, 자기 자신보다 큰, 원소 주소 빨리 찾으려고 달아두는듯)


(mchunk_ptr (실제 유저 공간에 나가 있는 chunk가 아니라 prev size를 포함한 시작주소,  glibc쪽은 일단 유저 공간에서 바라보는 payload외에도 size나 flag정보는 chunk라고 보네?)





