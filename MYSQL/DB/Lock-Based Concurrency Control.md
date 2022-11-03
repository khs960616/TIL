https://github.com/khs960616/TIL/blob/main/MYSQL/DB/Transaction%20Management.md 

데이터 베이스에서 한꺼번에 여러 트랜잭션을 인터리빙하며 처리하다보면 

Dirty Read, Unrepeatable Reads, Overwriting Uncommitted Data 등의 이상현상이 발생한다.

이러한 문제 없이, 직렬화 스케쥴을 만들어 내기 위해서 데이터 베이스는 어떤 방식으로 동시성을 컨트롤 할 수 있을까? 

---

### Lock-Based Concurrency Control

**Strict Two-phase Locking Protocol** : 두 가지 종류의 Lock을 사용하여 직렬화 스케쥴을 보장하는 방법이다. 
```
사용되는 Lock의 종류는 다음과 같다.
S(shared) Lock : 데이터베이스 object를 Read하기 위해 필요한 Lock
X(Exclusive) Lock : 데이터베이스 object에 대해 write하기 위해 필요한 Lock
```

- 각 트랜잭션은 Read, Write시 위 Lock들을 먼저 얻어야하며, Lock을 얻는 규칙은 다음과 같다.
> 1. 특정 트랜잭션이 오브젝트에 대한 S-lock을 가지고 있다면, 해당 오브젝트에 대해서 다른 트랜잭션들은 S-lock은 획득할 수 있으나 X-lock을 획득할 수 없다.
> 
> 2. 트랜잭션들이 끝나는 시점(commit, abort)에 트랜잭션이 가지고 있는 모든 lock을 풀어준다.
>
> 3. 특정 트랜잭션이 오브젝트에 대한 X-lock을 가지고 있다면, 해당 오브젝트에 대해서 다른 트랜잭션들은 어떠한 Lock도 획득할 수 없다.

---
