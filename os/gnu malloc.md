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

### chunk 구조 (일부 구현 레벨 내용 참고용)

- in use chunk 

실제 유저 공간에서 바라보는 데이터 외에도 청크의 사이즈 정보와 일부 메타정보를 나타내는 Flag를 가진다. 

| size    | LSB(3) - (A|M|P)| 

-> A: 메인 아레나에 소속된 chunk인지 여부 (0이면 main 아레나 소속), M: chunk자체가 특정 heap에 속한 것이 아니라, mmap으로 할당된 것인지 여부, P: 이전 chunk가 사용중인지 여부 


- free chunk

fwd, bck (아마 기억상 bin에 달려있을때 다른 free chunk들과 연결해놓기 위한 포인터)

(prev_size: free상태일때는 payload공간으로 나가는 맨 마지막쪽 (다른 chunk와 인접한 위치)에 자기 자신의 chunk size를 써둔다)

fd_nextsize / bk_nextsize  (large bin에는 크기 순서로 정렬 안되있어서, 자기 자신보다 큰, 원소 주소 빨리 찾으려고 달아두는듯)

(mchunk_ptr (실제 유저 공간에 나가 있는 chunk가 아니라 prev size를 포함한 시작주소,  glibc쪽은 일단 유저 공간에서 바라보는 payload외에도 size나 flag정보는 chunk라고 보네?)

(arena 관련된 구조체는 초기에 확보한 heap 공간(메모리 덩어리) 맨 앞에다가 메타 포함시켜서씀)

--

### bin

https://github.com/khs960616/TIL/blob/main/os/heap(bin).md 여기 예전에 써놨었네.. 


### thread local cache

각 스레드는 자신이 마지막으로 사용했던 arena가 무엇인지 기억하는 스레드 로컬 변수를 갖는다. 

각 스레드는 그 arena를 사용해야 할 때 해당 arena가 이미 사용 중이면, arena가 비게 될 때까지 기다리며 block된다.

스레드가 이전에 어떤 arena도 사용해본 적이 없다면, 사용되지 않는 arena를 재사용하려고 시도하거나,  새 arena를 만들거나, 전역 리스트에서 다음 arena를 고르는 방식으로 진행한다. 

각 스레드는 tcache라고 불리는 스레드별 캐시를 가지고 있어, arena에 대한 Lock 없이 얻을 수 있는 작은 chunk 들이 존재한다. 

chunk들은 fastbin과 비슷하게 단일 연결 리스트들의 배열 형태로 저장되지만, 링크가 chunk 헤더가 아니라 payload(사용자 영역)를 가리킨다는 점이 다르다. 

각 bin은 하나의 크기 chunk만 담으므로, 이 배열은 chunk 크기별로 인덱싱된다. 

fastbin과 달리 tcache는 각 bin에 들어갈 수 있는 chunk 개수에 제한이 있으며(tcache_count), 

어떤 요청 크기에 대해 해당 tcache bin이 비어 있으면 그 다음으로 더 큰 크기의 chunk를 사용하지 않는다.

대신 정상적인 malloc 루틴으로 폴백하게 된다.  (tcache 먼저 접근해보고 안되면 malloc 루틴으로 가는듯)



---

### 코드 검토 

#### 구조체 
```c
/*
   ----------- Internal state representation and initialization -----------
 */

/*
   have_fastchunks indicates that there are probably some fastbin chunks.
   It is set true on entering a chunk into any fastbin, and cleared early in
   malloc_consolidate.  The value is approximate since it may be set when there
   are no fastbin chunks, or it may be clear even if there are fastbin chunks
   available.  Given it's sole purpose is to reduce number of redundant calls to
   malloc_consolidate, it does not affect correctness.  As a result we can safely
   use relaxed atomic accesses.
 */


struct malloc_state
{
  // 아래나 보호용 mutex
  __libc_lock_define (, mutex);		

  /* Flags (formerly in max_fast).  */
  int flags;

  /* Set if the fastbin chunks contain recently inserted free blocks.  */
  /* Note this is a bool but not all targets support atomics on booleans.  */
  int have_fastchunks;

  /* Fastbins */
  mfastbinptr fastbinsY[NFASTBINS];

  // top chunk
  mchunkptr top; 

  // dv청크 같은건가? 최근에 request때 쓰고 남은 chunk   * The remainder from the most recent split of a small request */
  mchunkptr last_remainder;

  // smallbin + large bin 합쳐서 chunk들 관리하는 배열
  mchunkptr bins[NBINS * 2 - 2];

  // bin이 비어있는지 여부를 빨리 체크하기 위한 bitmap
  unsigned int binmap[BINMAPSIZE];

  /* Linked list */				// 아레나끼리도 링크드 리스트로 연결되있는듯 
  struct malloc_state *next;

  /* Linked list for free arenas.  Access to this field is serialized
     by free_list_lock in arena.c.  */ 
  struct malloc_state *next_free;        // 어떤 스레드에도 붙어 있지 않은 경우, free_list에 달아두기 위해 사용하는 link 역할을 하게 된다. (아래 참고) 

  /* Number of threads attached to this arena.  0 if the arena is on
     the free list.  Access to this field is serialized by
     free_list_lock in arena.c.  */
  INTERNAL_SIZE_T attached_threads;  // 이 아레나에 붙어있는 스레드 개수

  /* Memory allocated from the system in this arena.  */
  INTERNAL_SIZE_T system_mem;   // OS로부터 직접 확보한 메모리 총량
  INTERNAL_SIZE_T max_system_mem; // 이전에 확보한 메모리 총량  (malloc info에서 얘네 찍어주는거였네;)
};
```

```c
/* Arena free list.  free_list_lock synchronizes access to the
   free_list variable below, and the next_free and attached_threads
   members of struct malloc_state objects.  No other locks must be
   acquired after free_list_lock has been acquired.  */

__libc_lock_define_initialized (static, free_list_lock);
#if IS_IN (libc)
static size_t narenas = 1;
#endif
static mstate free_list;
```

// 아레나 초기화 함수
```c

/*
64비트 glibc → SIZE_SZ = 8
32비트 glibc → SIZE_SZ = 4
*/
#define DEFAULT_MXFAST     (64 * SIZE_SZ / 4) 

static void
malloc_init_state (mstate av)
{
  int i;
  mbinptr bin;

  /* Establish circular links for normal bins */
  for (i = 1; i < NBINS; ++i)    // 비어있으면 그냥 bin 객체 깡통으로 fd, bk에 bin 주소를 붙혀버림 
    {
      bin = bin_at (av, i);
      bin->fd = bin->bk = bin;
    }

#if MORECORE_CONTIGUOUS
  if (av != &main_arena)	
#endif
  set_noncontiguous (av);		//메인 아레나가 아닌 경우에는 flag마킹 (메인 아레나만 brk로 breakpoint늘려가며 사용하고, 다른 아레나들은 mmap 기반으로 쓰니까 뭐)
  if (av == &main_arena)
    set_max_fast (DEFAULT_MXFAST);   // fastbin으로 취급할 최대 chunk 크기 한계 설정, default_mxfast 자체가 64bit에서는 128이므로, 요청 크기 약 120바이트 이하까지는 fastbin으로 처리되게됨 
  atomic_store_relaxed (&av->have_fastchunks, false);

  av->top = initial_top (av);  // top chunk init은 아래서 좀 더 자세히 ! 
}
```

// 새로운 아레나 생성 함수 
```c
static mstate
_int_new_arena (size_t size)
{
  mstate a;
  heap_info *h;
  char *ptr;
  unsigned long misalign;

  h = new_heap (size + (sizeof (*h) + sizeof (*a) + MALLOC_ALIGNMENT),
                mp_.top_pad);
  if (!h)
    {
      /* Maybe size is too large to fit in a single heap.  So, just try
         to create a minimally-sized arena and let _int_malloc() attempt
         to deal with the large request via mmap_chunk().  */
      h = new_heap (sizeof (*h) + sizeof (*a) + MALLOC_ALIGNMENT, mp_.top_pad);
      if (!h)
        return 0;
    }
  a = h->ar_ptr = (mstate) (h + 1);
  malloc_init_state (a);
  a->attached_threads = 1;
  /*a->next = NULL;*/
  a->system_mem = a->max_system_mem = h->size;

  /* Set up the top chunk, with proper alignment. */
  ptr = (char *) (a + 1);
  misalign = (unsigned long) chunk2mem (ptr) & MALLOC_ALIGN_MASK;
  if (misalign > 0)
    ptr += MALLOC_ALIGNMENT - misalign;
  top (a) = (mchunkptr) ptr;
  set_head (top (a), (((char *) h + h->size) - ptr) | PREV_INUSE);

  LIBC_PROBE (memory_arena_new, 2, a, size);
  mstate replaced_arena = thread_arena;
  thread_arena = a;
  __libc_lock_init (a->mutex);

  __libc_lock_lock (list_lock);

  /* Add the new arena to the global list.  */
  a->next = main_arena.next;
  /* FIXME: The barrier is an attempt to synchronize with read access
     in reused_arena, which does not acquire list_lock while
     traversing the list.  */
  atomic_write_barrier ();
  main_arena.next = a;

  __libc_lock_unlock (list_lock);

  __libc_lock_lock (free_list_lock);
  detach_arena (replaced_arena);
  __libc_lock_unlock (free_list_lock);

  /* Lock this arena.  NB: Another thread may have been attached to
     this arena because the arena is now accessible from the
     main_arena.next list and could have been picked by reused_arena.
     This can only happen for the last arena created (before the arena
     limit is reached).  At this point, some arena has to be attached
     to two threads.  We could acquire the arena lock before list_lock
     to make it less likely that reused_arena picks this new arena,
     but this could result in a deadlock with
     __malloc_fork_lock_parent.  */

  __libc_lock_lock (a->mutex);

  return a;
}
```

```
/*
   Top

   사용 가능한 메모리의 끝(brk)에 인접해 있는 가장 위쪽(top-most)의 사용 가능한 청크는 특별하게 취급된다.
   이 청크는 어떤 bin에도 포함되지 않으며, 다른 사용 가능한 청크가 전혀 없을 때에만 사용된다.
   또한 이 청크가 매우 큰 경우에는 시스템으로 반환된다, 사이즈 제한은 (M_TRIM_THRESHOLD)을 참고.

    top은 초기에는 크기가 0인 자기 자신의 bin을 가리키도록 되어 있기 때문에,
    top chunk가 존재하는지 여부를 체크하는 별도 분기 처리 없이 첫 번째 malloc 요청 시 반드시 힙 확장이 일어나도록 강제할 수 있다. 

    그러나 systemalloc (malloc 내부에서 실제 os로부터 메모리를 받아오는 경우)에서는
    현재 top/힙 상태가 초기 상태인지, 정상 heap 상태인지”를 정확히 구분해서 힙 레이아웃을 맞춰야하기 떄문에, 
    따라서 initial_top 매크로로 초기 상태인지 체크한다. 
 
    malloc_check.c쪽   top_check 쓰이는 코드들 보는게 이해가 빠를듯.. 
 */

/* Conveniently, the unsorted bin can be used as dummy top on first call */
#define initial_top(M)              (unsorted_chunks (M))
```



#### 로직 
결국에 모든 malloc요청은 
__libc_malloc 통해서 들어오는 거 같고, 해당 함수 내부에서 만약에 malloc_initialized 변수가 설정되어있지 않다면 

ptmalloc_init를 호출해서 초기화한다.  (arena.c 에 위치함)

```c
/* Already initialized? */
static bool __malloc_initialized = false;

// thread areana 변수 
static __thread mstate thread_arena attribute_tls_model_ie;

static void
ptmalloc_init (void)
{
  if (__malloc_initialized)
    return;

  __malloc_initialized = true;

#if USE_TCACHE
  tcache_key_initialize ();
#endif


// USE_MTAG 관련된 건 일단, arm 아키텍처인 경우 아니면 딱히 고려 안해도될듯? 
// GLIBC_TUNABLES=glibc.mem.tagging=1 

#ifdef USE_MTAG
  if ((TUNABLE_GET_FULL (glibc, mem, tagging, int32_t, NULL) & 1) != 0)
    {
      /* If the tunable says that we should be using tagged memory
	 and that morecore does not support tagged regions, then
	 disable it.  */
      if (__MTAG_SBRK_UNTAGGED)  // 해당 아키텍처에서 sbrk 영역이 태그를 지원하지 않는 케이스
	__always_fail_morecore = true;

      mtag_enabled = true;
      mtag_mmap_flags = __MTAG_MMAP_FLAGS;
    }
#endif

// dlopen 등으로 libc 여는 등 희안한 형태로 쓰는거 아니면 여기 걸릴 일도 없어 보임 

#if defined SHARED && IS_IN (libc)
  /* In case this libc copy is in a non-default namespace, never use
     brk.  Likewise if dlopened from statically linked program.  The
     generic sbrk implementation also enforces this, but it is not
     used on Hurd.  */
  if (!__libc_initial)
    __always_fail_morecore = true;
#endif

  // main process의 tls변수에 main areana 설정 
  thread_arena = &main_arena;

  malloc_init_state (&main_arena);

#if HAVE_TUNABLES
  TUNABLE_GET (top_pad, size_t, TUNABLE_CALLBACK (set_top_pad));
  TUNABLE_GET (perturb, int32_t, TUNABLE_CALLBACK (set_perturb_byte));
  TUNABLE_GET (mmap_threshold, size_t, TUNABLE_CALLBACK (set_mmap_threshold));
  TUNABLE_GET (trim_threshold, size_t, TUNABLE_CALLBACK (set_trim_threshold));
  TUNABLE_GET (mmap_max, int32_t, TUNABLE_CALLBACK (set_mmaps_max));
  TUNABLE_GET (arena_max, size_t, TUNABLE_CALLBACK (set_arena_max));
  TUNABLE_GET (arena_test, size_t, TUNABLE_CALLBACK (set_arena_test));
# if USE_TCACHE
  TUNABLE_GET (tcache_max, size_t, TUNABLE_CALLBACK (set_tcache_max));
  TUNABLE_GET (tcache_count, size_t, TUNABLE_CALLBACK (set_tcache_count));
  TUNABLE_GET (tcache_unsorted_limit, size_t,
	       TUNABLE_CALLBACK (set_tcache_unsorted_limit));
# endif
  TUNABLE_GET (mxfast, size_t, TUNABLE_CALLBACK (set_mxfast));
#else
  if (__glibc_likely (_environ != NULL))
    {
      char **runp = _environ;
      char *envline;

      while (__builtin_expect ((envline = next_env_entry (&runp)) != NULL,
                               0))
        {
          size_t len = strcspn (envline, "=");

          if (envline[len] != '=')
            /* This is a "MALLOC_" variable at the end of the string
               without a '=' character.  Ignore it since otherwise we
               will access invalid memory below.  */
            continue;

          switch (len)
            {
            case 8:
              if (!__builtin_expect (__libc_enable_secure, 0))
                {
                  if (memcmp (envline, "TOP_PAD_", 8) == 0)
                    __libc_mallopt (M_TOP_PAD, atoi (&envline[9]));
                  else if (memcmp (envline, "PERTURB_", 8) == 0)
                    __libc_mallopt (M_PERTURB, atoi (&envline[9]));
                }
              break;
            case 9:
              if (!__builtin_expect (__libc_enable_secure, 0))
                {
                  if (memcmp (envline, "MMAP_MAX_", 9) == 0)
                    __libc_mallopt (M_MMAP_MAX, atoi (&envline[10]));
                  else if (memcmp (envline, "ARENA_MAX", 9) == 0)
                    __libc_mallopt (M_ARENA_MAX, atoi (&envline[10]));
                }
              break;
            case 10:
              if (!__builtin_expect (__libc_enable_secure, 0))
                {
                  if (memcmp (envline, "ARENA_TEST", 10) == 0)
                    __libc_mallopt (M_ARENA_TEST, atoi (&envline[11]));
                }
              break;
            case 15:
              if (!__builtin_expect (__libc_enable_secure, 0))
                {
                  if (memcmp (envline, "TRIM_THRESHOLD_", 15) == 0)
                    __libc_mallopt (M_TRIM_THRESHOLD, atoi (&envline[16]));
                  else if (memcmp (envline, "MMAP_THRESHOLD_", 15) == 0)
                    __libc_mallopt (M_MMAP_THRESHOLD, atoi (&envline[16]));
                }
              break;
            default:
              break;
            }
        }
    }
#endif
}

```

```c
// main arena는 조기에 전역변수로 아래값들 설정함 (나머지는 0) 
static struct malloc_state main_arena =   
{
  .mutex = _LIBC_LOCK_INITIALIZER,
  .next = &main_arena,
  .attached_threads = 1
};
```

```c

// tls변수에 arena 설정 시, mutex 획득까지 기다리기, 없다면 arena_get2 호출해서 tls
#define arena_get(ptr, size) do { \
      ptr = thread_arena;						         \
      arena_lock (ptr, size);						      \
  } while (0)

#define arena_lock(ptr, size) do {					      \
      if (ptr)								      \
        __libc_lock_lock (ptr->mutex);					      \
      else								      \
        ptr = arena_get2 ((size), NULL);				      \
  } while (0)
```

```c
static mstate
arena_get2 (size_t size, mstate avoid_arena)
{
  mstate a;

  static size_t narenas_limit;

  a = get_free_list ();     // free_list (attach된 스레드가 없는 arena가 있는지 체크해서, 있다면 바로 해당 arena 리턴 
  if (a == NULL)
    {
      /* Nothing immediately available, so generate a new arena.  */
      if (narenas_limit == 0)  
        {
          if (mp_.arena_max != 0)   // MALLOC_ARENA_MAX 값을 유저가 따로 설정해뒀으면 그 값을 그대로 사용, 
            narenas_limit = mp_.arena_max;
          else if (narenas > mp_.arena_test)  // 현재 아레나 개수가 arena_test를 넘기면, 이 시점에는  CPU 개수 기반하여 limit을 처버림
            {
              int n = __get_nprocs ();
              /*  참고로 64bit 머신 기준으로는 cpu 개수 * 8 이 기본적인 상한값
			     #define NARENAS_FROM_NCORES(n) ((n) * (sizeof (long) == 4 ? 2 : 8))
              */
              if (n >= 1)
                narenas_limit = NARENAS_FROM_NCORES (n);
              else
                /* We have no information about the system.  Assume two
                   cores.  */
                narenas_limit = NARENAS_FROM_NCORES (2);
            }
        }
    repeat:;
      size_t n = narenas;
      /* NB: the following depends on the fact that (size_t)0 - 1 is a
         very large number and that the underflow is OK.  If arena_max
         is set the value of arena_test is irrelevant.  If arena_test
         is set but narenas is not yet larger or equal to arena_test
         narenas_limit is 0.  There is no possibility for narenas to
         be too big for the test to always fail since there is not
         enough address space to create that many arenas.  */

      /* 새로운 아레나를 생성할 수 있는지 체크 */
      if (__glibc_unlikely (n <= narenas_limit - 1))
        {
           /* CAS로 arena 개수 증가를 시도해보는데, 실패했다면 재시도 때림  */
          if (catomic_compare_and_exchange_bool_acq (&narenas, n + 1, n))
            goto repeat;

          // 아레나 개수 늘리는데 성공했다면 new_arena

          a = _int_new_arena (size);    // 이 안에서 새로 아레나 만들면서 tls 변수에 arena 걸어둠 
	  if (__glibc_unlikely (a == NULL))
            catomic_decrement (&narenas); 
        }
      else
        a = reused_arena (avoid_arena);  // 마찬가지로 여기서도 아레나 tls변수에 걸어둠 
                                         // reused_arena는 전역변수 next_to_use (areana)를 기준으로 try lock으로 arena들 잡아보면서 attach해봄
                                         // 안잡히면 그냥 next_to_use에 대한 blocking lock을 잡고 선정한다 (next_to_use가 avoid arena라면 그 다음거)
                                         // 선정되면 next_to_use값은 현재 선정된거 다음으로 옮김 
    }
  return a;
}
```



```c
#if IS_IN (libc)
void *
__libc_malloc (size_t bytes)
{
  mstate ar_ptr;
  void *victim;

  _Static_assert (PTRDIFF_MAX <= SIZE_MAX / 2,
                  "PTRDIFF_MAX is not more than half of SIZE_MAX");

  // 초기화 되지 않았으면 ptmalloc_init(); 
  if (!__malloc_initialized)
    ptmalloc_init ();
#if USE_TCACHE
  /* int_free also calls request2size, be careful to not pad twice.  */
  size_t tbytes;

/*
 요청온 사이즈 bytes가  define PTRDIFF_MAX		(9223372036854775807L) 보다 큰,
 말도 안되는 요청인 경우 false, 그 외의 경우에는 tbytes에

#define request2size(req)                                         \
  (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)  ?             \
   MINSIZE :                                                      \
   ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)

로 값을 계산해서 실제 할당할 메모리를 걸어두게 된다. (메타 포함된 chunk 사이즈) 

*/
  if (!checked_request2size (bytes, &tbytes))
    {
      __set_errno (ENOMEM);
      return NULL;
    }

/* ================ [case1]  tcache에서 chunk를 가져오는 경우 ================   */
  size_t tc_idx = csize2tidx (tbytes);  // tcache부터 탐색을 위한 idx 계산 

  MAYBE_INIT_TCACHE ();

  DIAG_PUSH_NEEDS_COMMENT;
  if (tc_idx < mp_.tcache_bins
      && tcache
      && tcache->counts[tc_idx] > 0)
    {
      victim = tcache_get (tc_idx);
      return tag_new_usable (victim);
    }
  DIAG_POP_NEEDS_COMMENT;
#endif
/* ===================================================================   */

  if (SINGLE_THREAD_P)
    {
      victim = tag_new_usable (_int_malloc (&main_arena, bytes));
      assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
	      &main_arena == arena_for_chunk (mem2chunk (victim)));
      return victim;
    }

  arena_get (ar_ptr, bytes);  /

  victim = _int_malloc (ar_ptr, bytes); 
  /* Retry with another arena only if we were able to find a usable arena
     before.  */
  if (!victim && ar_ptr != NULL)
    {
      LIBC_PROBE (memory_malloc_retry, 1, bytes);
      ar_ptr = arena_get_retry (ar_ptr, bytes);
      victim = _int_malloc (ar_ptr, bytes);
    }

  if (ar_ptr != NULL)
    __libc_lock_unlock (ar_ptr->mutex);

  victim = tag_new_usable (victim);

  assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
          ar_ptr == arena_for_chunk (mem2chunk (victim)));
  return victim;
}
libc_hidden_def (__libc_malloc)
```


```c
/* The value of tcache_key does not really have to be a cryptographically
   secure random number.  It only needs to be arbitrary enough so that it does
   not collide with values present in applications.  If a collision does happen
   consistently enough, it could cause a degradation in performance since the
   entire list is checked to check if the block indeed has been freed the
   second time.  The odds of this happening are exceedingly low though, about 1
   in 2^wordsize.  There is probably a higher chance of the performance
   degradation being due to a double free where the first free happened in a
   different thread; that's a case this check does not cover.  */

// 결국에 double free 방지하기 위한 일종의 key값을 free 시점에 페이로드에 박아놓고 사용하는 것 같음 (일단 메모리 할당 로직에서 예외나 디버깅 로직은 필요하면 다시 보자) 
static void
tcache_key_initialize (void)
{
  if (__getrandom (&tcache_key, sizeof(tcache_key), GRND_NONBLOCK)
      != sizeof (tcache_key))
    {
      tcache_key = random_bits ();
#if __WORDSIZE == 64
      tcache_key = (tcache_key << 32) | random_bits ();
#endif
    }
}
```





