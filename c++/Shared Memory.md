# Shared Memory 

프로세스 시작과 실행 중 커널로 부터 할당받은 메모리 공간은, 해당 공간을 요청한 프로세스만이 접근 가능하도록 되어있다. 
프로세스간의 데이터 공유를 위한 IPC 방법은 Shared Memory는, 커널에서 여러 프로세스에서 접근하여 사용할 수 있는 메모리 공간을 만들고,
각 프로세스에서 해당 공유 메모리에 attach, detach할 수 있는 방법을 제공한다. 

해당 MD문서에서는 POSIX에서 기본적으로 제공하는 Shared Memory 관련된 함수들의 사용 방법과, 사용하는 자료구조에 대해 알아보고,
Boost.Interprocess docs를 통해 제공되는 여러 Shared Memory 관련된 유틸들을 살펴보고자 한다. 

## POSIX
- 공유 메모리는 프로세스로부터 먼저 생성되며, 만들어진 공유 메모리는 커널에 의해 관리된다.
- 한번 만들어진 공유 메모리 공간은, 직접 삭제를 하거나, 시스템을 재부팅하는 경우에만 삭제된다.

```c++
#include<sys/shm.h>
#include<sys/types.h>
```

```c++
int shmget(key_t key, int size, int shmflg)
void *shmat(int shmid, const void *shmaddr, int shmflg)
int shmdt(const void *shaddr)
int shmctl(int shmid, int cmd, struct shmid_ds *buf)
```

```c++
// 공유 메모리 공간 관리를 위한 자료 구조
struct shmid_ds {
  struct ipc_perm shm_perm; // 퍼미션
  int shm_segsz; //  shared memory 크기 
  time_t shm_atime;  // 마지막 attach한 시간 
  time_t shm_dtime; // 마지막 detach 시간
  time_t shm_ctime // 마지막 변경 시간
  unsigned short shm_cpid; // 생성한 프로세스의 pid
  unsinged short shm_lpid; // 마지막으로 작동한 프로세스의 pid
  short shm_nattach; // 현재 접근한 프로세스의 수
}
```

### shmget
커널에 공유 메모리 공간을 요청하기 위한 system call 함수, 새로운 공유 메모리 영역을 생성하거나, 기존에 만들어진 공유 메모리 영역을 참조할 때 사용한다. 
```
key = 공유메모리별로, 가지는 고유값
size = 공유 메모리의 최소 크기, 새로운 공유 메모리 생성시 크기를 명시해야하며, 이미 존재하는 공유메모리에 접근하는 것이라면 0으로 명시해야한다. 
shmflg  = 공유 메모리 접근 권한 및 생성 방식을 명시하기 위한 flag이다. 
```

### shmat
생성한 공유메모리 공간에 attach할때 사용하는 함수이다.
```
1) attach하고자하는 공유 메모리 식별자
2) shmaddr: 메모리가 붙을 주소를 명시하기 위해 사용하며, 해당 값을 0을 주면 커널이 attach할 주소를 명시한다.
3) 공유 메모리를 읽기 모드, 읽기/쓰기 모드, 전용 중 어떤 것으로 사용할 지 결정하는 인자.
```

### shmdt
프로세스가 더 이상 해당 공유메모리 공간을 사용하지 않을 경우 프로세스와 공유 메모리 공간을 분리하기 위해 사용한다.

### shmctl
shmid_ds 구조체를 관리하며, 해당 공유 메모리에 대한 owner, permission등을 변경하거나, 
공유 메모리 삭제 또는 lock 설정등의 작업을 수행한다.

### 기타 참고 
ipcs -l,   ipcs -m   → cli에서 sharemd memory segement관련된 정보 확인 용도


## Boost.Interprocess (Shared Memory)



## REF
https://www.boost.org/doc/libs/1_67_0/doc/html/interprocess.html
