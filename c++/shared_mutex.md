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


__gthread_mutex_lock  (mutex)

pthread_rwlock_xxlock (shared mutex)  xx는 wr, rd 로 read, write lock 구분해서 사용 


## Shared Mutex쪽 lock 관련 함수 

-> 일단 posix 표준으로 보이는 소스 코드 기준으로.. 근데 shared_mutex 자체가 posix 표준은 아닌데.. 그냥 glibc 기준으로 봐야할까??
-> 근데 뭐.. operation들만 어떤식으로 쓰이면 봐도 되니까,... 아래꺼로 보고 내용 모자라면 보는거로.. 

```c
int pthread_rwlock_rdlock (pthread_rwlock_t * rwlock)
{
  int result;
  pthread_rwlock_t rwl;

  if (rwlock == NULL || *rwlock == NULL)  // parameter validation 
    {
      return EINVAL;
    }

  /*
   * We do a quick check to see if we need to do more work
   * to initialise a static rwlock. We check
   * again inside the guarded section of __ptw32_rwlock_check_need_init()
   * to avoid race conditions.
   */
  if (*rwlock == PTHREAD_RWLOCK_INITIALIZER)
    {
      result = __ptw32_rwlock_check_need_init (rwlock);

      if (result != 0 && result != EBUSY)
	{
	  return result;
	}
    }

  rwl = *rwlock;

  if (rwl->nMagic !=  __PTW32_RWLOCK_MAGIC)
    {
      return EINVAL;
    }

  if ((result = pthread_mutex_lock (&(rwl->mtxExclusiveAccess))) != 0)
    {
      return result;
    }

  if (++rwl->nSharedAccessCount == INT_MAX)
    {
      if ((result =
	   pthread_mutex_lock (&(rwl->mtxSharedAccessCompleted))) != 0)
	{
	  (void) pthread_mutex_unlock (&(rwl->mtxExclusiveAccess));
	  return result;
	}

      rwl->nSharedAccessCount -= rwl->nCompletedSharedAccessCount;
      rwl->nCompletedSharedAccessCount = 0;

      if ((result =
	   pthread_mutex_unlock (&(rwl->mtxSharedAccessCompleted))) != 0)
	{
	  (void) pthread_mutex_unlock (&(rwl->mtxExclusiveAccess));
	  return result;
	}
    }

  return (pthread_mutex_unlock (&(rwl->mtxExclusiveAccess)));
}
```

### lock을 잡으려고 했을때, rwlock 자체가 초기화 되었는지 체크하는 함수
```c
__ptw32_rwlock_check_need_init (pthread_rwlock_t * rwlock)
{
  int result = 0;
  __ptw32_mcs_local_node_t node;

  /*
   * The following guarded test is specifically for statically
   * initialised rwlocks (via PTHREAD_RWLOCK_INITIALIZER).
   */
  __ptw32_mcs_lock_acquire(&__ptw32_rwlock_test_init_lock, &node);

  /*
   * We got here possibly under race
   * conditions. Check again inside the critical section
   * and only initialise if the rwlock is valid (not been destroyed).
   * If a static rwlock has been destroyed, the application can
   * re-initialise it only by calling pthread_rwlock_init()
   * explicitly.
   */
  if (*rwlock == PTHREAD_RWLOCK_INITIALIZER)
    {
      result = pthread_rwlock_init (rwlock, NULL);
    }
  else if (*rwlock == NULL)
    {
      /*
       * The rwlock has been destroyed while we were waiting to
       * initialise it, so the operation that caused the
       * auto-initialisation should fail.
       */
      result = EINVAL;
    }

  __ptw32_mcs_lock_release(&node);

  return result;
}
``` 

### rw lock init
해당 함수는 기본적으로 pthread_mutex_init로 lock 초기화하고, 실패시 여러 errno 반환 등을 하니까 기록으로 안남겨도 될듯, 실제 mutex 초기화 코드는 아래와 같음

```c
int
pthread_mutex_init (pthread_mutex_t * mutex, const pthread_mutexattr_t * attr)
{
  int result = 0;
  pthread_mutex_t mx;

  if (mutex == NULL)
    {
      return EINVAL;
    }

  if (attr != NULL && *attr != NULL)
    {
      if ((*attr)->pshared == PTHREAD_PROCESS_SHARED)
        {
          /*
           * Creating mutex that can be shared between
           * processes.
           */
#if _POSIX_THREAD_PROCESS_SHARED >= 0

          /*
           * Not implemented yet.
           */

#error ERROR [__FILE__, line __LINE__]: Process shared mutexes are not supported yet.

#else

          return ENOSYS;

#endif /* _POSIX_THREAD_PROCESS_SHARED */
        }
    }

  mx = (pthread_mutex_t) calloc (1, sizeof (*mx));

  if (mx == NULL)
    {
      result = ENOMEM;
    }
  else
    {
      mx->lock_idx = 0;
      mx->recursive_count = 0;
      mx->robustNode = NULL;
      if (attr == NULL || *attr == NULL)
        {
          mx->kind = PTHREAD_MUTEX_DEFAULT;
        }
      else
        {
          mx->kind = (*attr)->kind;
          if ((*attr)->robustness == PTHREAD_MUTEX_ROBUST)
            {
              /*
               * Use the negative range to represent robust types.
               * Replaces a memory fetch with a register negate and incr
               * in pthread_mutex_lock etc.
               *
               * Map 0,1,..,n to -1,-2,..,(-n)-1
               */
              mx->kind = -mx->kind - 1;

              mx->robustNode = (__ptw32_robust_node_t*) malloc(sizeof(__ptw32_robust_node_t));
              if (NULL == mx->robustNode)
        	{
        	  result = ENOMEM;
        	}
              else
        	{
        	  mx->robustNode->stateInconsistent =  __PTW32_ROBUST_CONSISTENT;
        	  mx->robustNode->mx = mx;
        	  mx->robustNode->next = NULL;
        	  mx->robustNode->prev = NULL;
        	}
            }
        }

      if (0 == result)
	{
	  mx->ownerThread.p = NULL;

	  mx->event = CreateEvent (NULL,  __PTW32_FALSE,    /* manual reset = No */
				    __PTW32_FALSE,           /* initial state = not signalled */
				   NULL);                 /* event name */

	  if (0 == mx->event)
	    {
	      result = ENOSPC;
	    }
	}
    }

  if (0 != result)
    {
      if (NULL != mx->robustNode)
	{
	  free (mx->robustNode);
	}
      free (mx);
      mx = NULL;
    }

  *mutex = mx;

  return (result);
}

```

---
https://github.com/BrianGladman/pthreads/blob/master/pthread_rwlock_rdlock.c // posix 표준? 
https://wariua.github.io/man-pages-ko/pthread_rwlock_rdlock%28%29/
