# Transaction Management 

### Transactions (트랜잭션)
DBMS's abstract view of user program (a sequence of reads and writes)

사용자 프로그램 (데이터베이스 사용자 프로그램)에서의 DBMS 추상적인 뷰

(사용자 프로그램에서 특정 작업을 처리하기 위해 데이터베이스에 실행하는 일련의 DB read와 write들을 모든 작업 단위)


Concurrent execution of user programs is essential for good DMBS performance.
- 좋은 DMBS 성능을 위해서는 사용자 프로그램의 동시 실행이 필수적이다.
ex) 구매 프로그램, 예약 프로그램 등 

- DB를 사용하는 사용자 프로그램(트랜잭션)에서, 디스크 접근(I/O)는 자주 일어난다. 
- 그러나 디스크 접근은 실제 CPU에서 일어나는 연산 처리 속도보다 느리다. 
- 따라서 CPU Humming(keep busy)를 위해서는 이러한 트랜잭션들이 동시에 일어나는 것이 효율적이다.

사용자 프로그램들은 데이터베이스로부터 데이터를 가져와서, 그 데이터를 이용해 비지니스에 필요한 여러 로직들을 처리한다.
> 그러나 DBMS 입장에서 봤을 때 사용자 프로그램에서 일어나는 CPU 연산은 중요한 것이 아니다. 
> 
> DBMS에 입장에서는 disk i/o (사용자 프로그램이 데이터베이스에 read / write 하는 연산)이 중요하다. 

### 동시성 

동시성 : (여러개의 유저프로그램을 동시에 실행하는 것)

DBMS에서의 동시성은 여러 트랜잭션들의 action(read, write)들을 interleaves 방식으로 처리함으로써 달성한다. 

interleave의 동작 방식 : 실제로 프로세스가 병렬로 처리되는 것이 아니라 context switch 하며 처리하는 것 

트랜잭션들의 시작전, 데이터베이스가 일관성을 유지하고 있는 상태라면 각 트랜잭션이 종료된 후에도 일관성이 유지되어야한다. 

트랜잭션 관리에서는 크게 두 가지의 이슈가 있다
1. 여러 트랜잭션들이 동시에 수행되는 경우
2. 각 트랜잭션 실행 중 장애 발생 

각각의 side effect는 어떤 것이 있으며 해당 이슈를  어떻게 처리할 것인가에 대한 이슈이다. 

---
### Atomicity of Transactions 

DBMS입장에서 보았을때, 각 트랜잭션들은 일련의 i/o action(read, write)의 연속이다.

트랜잭션이 끝날 때는 두 가지 상태(commit, abort)중 한 가지 상태를 가진다.

DBMS는 각 트랜잭션에 대해서 원자성을 보장해줘야한다.

트랜잭션의 atomicity (원자성) : 트랜잭션에 속하는 모든 i/o actions은 마치 하나의 원자처럼 더 이상 쪼갤 수 없어야한다. 

즉 전체가 실행되서 정상적으로 모든 작업이 완료되거나, 모든 액션의 작업이 롤백되어야한다. (all or nothing) 

DBMS는 트랜잭션의 모든 action들을 log로 남긴다. log를 이용해서 트랜잭션이 abort되는 경우 i/o action에 대한 undo작업을 진행할 수 있다. 



---
### REF 

Database Management Systems 3ed, R. Ramakrishnan and J.Gehrke
