## Asynchronous I/O
I/O Request를 보낸 프로세스가 IO작업이 완료될 때까지 Block되지 않고
CPU를 점유한 상태로 다음 작업을 진행하며, 차후에 I/O 요청에 대한 상태를 확인하는 기법

Linux 커널단에서는 fs/aio.c에 실제 커널에서 작동하는 코드들이 정의되어 있으며, 

User space 인터페이스는 https://github.com/spotify/linux/blob/master/include/linux/aio_abi.h 을 참고한다.


## 라이브러리
1. libaio: 커널의 도움을 받아 user space에서 aio를 활용할 수 있도록 구현된 aio 라이브러리 (system call 사용)
2. librt: 차후 사용할 일 있으면 별도 추가 조사 필요 (일단은 커널의 도움없이 user space에서 aio를 구현한 라이브러리)
3. io_uring: 리눅스 커널 5.1 이후로 도입된 비동기 I/O처리 모델

---

### AIO Context 
```C
struct kioctx {
    struct percpu_ref	users;             
    atomic_t		dead;              // 컨텍스트가 종료되었는지 여부를 나타내는 카운터

    struct percpu_ref	reqs;              // 요청 (kiocb)에 대한 percpu 참조

    unsigned long		user_id;           // 사용자 ID

    struct __percpu kioctx_cpu *cpu;      // percpu 데이터를 가리키는 포인터

    unsigned		req_batch;         // 요청 배치 크기
    unsigned		max_reqs;          // 최대 요청 수

    unsigned		nr_events;         // 이벤트의 수 (struct io_event)
    
    unsigned long		mmap_base;         // mmap을 통한 접근을 위한 베이스 주소
    unsigned long		mmap_size;         // mmap을 통한 접근을 위한 크기

    struct page		**ring_pages;      // AIO 링버퍼 페이지의 배열
    long			nr_pages;          // 페이지의 수

    struct rcu_work		free_rwork;        // RCU를 사용한 workqueue에서 free_ioctx()를 수행하기 위한 work

    struct ctx_rq_wait	*rq_wait;          // 모든 진행 중인 요청이 완료되었을 때 신호를 주는 구조체

    struct {                               // 요청 가능한 슬롯의 수를 나타내는 구조체
        atomic_t	reqs_available;  // ??
    } ____cacheline_aligned_in_smp;

    struct {                               // 활성상태인 요청을 관리하는 구조체
        spinlock_t	ctx_lock;          // 요청 리스트에 대한 락
        struct list_head active_reqs;  // 리스트 헤더
    } ____cacheline_aligned_in_smp;

    struct {                               // 링버퍼에 대한 동기화를 제공하는 구조체
        struct mutex	ring_lock;        // 링버퍼에 대한 락
        wait_queue_head_t wait;          // 대기 큐 헤드
    } ____cacheline_aligned_in_smp;

    struct {                               // 완료된 이벤트의 상태를 관리하는 구조체
        unsigned	tail;               // 링버퍼에서 처리되지 않은 이벤트의 tail 위치
        unsigned	completed_events;   // 완료된 이벤트의 수
        spinlock_t	completion_lock;   // 완료된 이벤트에 대한 락
    } ____cacheline_aligned_in_smp;

    struct page		*internal_pages[AIO_RING_PAGES];  // AIO 링버퍼 내부 페이지 배열
    struct file		*aio_ring_file;                  // AIO 링버퍼를 나타내는 파일

    unsigned		id;                // 컨텍스트의 고유 ID
};
```

### libaio 
```
- Linux에서 비동기적으로 입출력 작업을 수행하기 위한 라이브러리 (#include <libaio.h>)
- 디스크 I/O 또는 네트워크 I/O와 같이 시간이 오래 걸리는 작업에 대한 성능 향상을 위해 사용된다.
```
```C
extern int io_setup(int maxevents, io_context_t *ctxp);
extern int io_destroy(io_context_t ctx);
extern int io_submit(io_context_t ctx, long nr, struct iocb *ios[]);
extern int io_cancel(io_context_t ctx, struct iocb *iocb, struct io_event *evt);
extern int io_getevents(io_context_t ctx_id, long min_nr, long nr, struct io_event *events, struct timespec *timeout);
```

헬퍼 함수들 
```C
static inline void io_prep_pread(struct iocb *iocb, int fd, void *buf, size_t count, long long offset)
static inline void io_prep_pwrite(struct iocb *iocb, int fd, void *buf, size_t count, long long offset)
static inline void io_prep_preadv(struct iocb *iocb, int fd, const struct iovec *iov, int iovcnt, long long offset)
static inline void io_prep_pwritev(struct iocb *iocb, int fd, const struct iovec *iov, int iovcnt, long long offset)
static inline void io_prep_poll(struct iocb *iocb, int fd, int events)
static inline void io_prep_fsync(struct iocb *iocb, int fd)
static inline void io_prep_fdsync(struct iocb *iocb, int fd)
static inline int io_poll(io_context_t ctx, struct iocb *iocb, io_callback_t cb, int fd, int events)
static inline int io_fsync(io_context_t ctx, struct iocb *iocb, io_callback_t cb, int fd)
static inline int io_fdsync(io_context_t ctx, struct iocb *iocb, io_callback_t cb, int fd)
static inline void io_set_eventfd(struct iocb *iocb, int eventfd);
```


기본적인 흐름

1. user space에서 aio context를 초기화 요청을 하면, 커널 모드에서 aio context를 생성하고, 생성된 context에 대한 id를 return해준다.
   
2. user space에서 i/o request는 iocb(io control block) 구조체로 구성하여, aio context에 submit 한다.(submit이 성공하면 성공한 갯수만큼 return해준다)

3. aio context는 submit된 io 작업들을 비동기적으로 처리하며, io처리 결과를 event로 생성해서 aio_ring에 저장해서 가지고 있는다.

4. user space에서 i/o request에 대한 결과를 확인하고자 할 때, getevents 함수를 호출하여 i/o 수행결과를 확인한다. 
