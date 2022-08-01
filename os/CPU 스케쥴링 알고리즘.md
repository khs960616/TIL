### CPU Scheduler
스케쥴링의 대상 : ready queue에 존재하는 (또는 이동한 프로세스들)

---
## First Come First Served
정의 : CPU에 먼저 도착하는 순서대로 프로세스를 할당해주는 방식 
선점형/비선점형 : 비선점형

장점 : FIFO 방식의 Queue를 이용한 구현이 쉽다.
단점 : convoy효과 (실행시간이 오래걸리는 프로세스가, 만일 실행시간이 적게 걸리는 프로세스보다 먼저 들어온 경우, CPU 효율성이 낮아지는 단점이 있다.
(Average waiting time이 길어진다.)

---

### Shortest-Job-First 
정의 : CPU burst time이 짧은 프로세스에게 먼저 CPU를 할당한다.
선점형/비선점형 : 비선점형 

장점 : Average waiting time이 적어짐을 어느정도 보장한다.
단점 : starvation (사용 시간이 긴 프로세스의 경우, 지속적으로 ready queue에 사용시간이 짧은 프로세스가 들어오는 경우 영원히 CPU를 할당받지 못한다.)
      특정 프로세스가 얼마나 CPU를 사용할 지 모르는 경우 해당 알고리즘을 적용하기가 어렵다.(단지 추정할 뿐이다.)
      
---
### Shortest Remaining Time First
정의 : Ready queue에 존재하는 프로세스중 남은 cpu burst time이 가장 짧은 프로세스에게 CPU를 할당한다.
     (따라서 새로운 프로세스가 Ready queue로 들어오는 경우 스케쥴링이 새롭게 이루어진다.)
선점/비선점형 : 선점형
단점 : SJF와 같다.

---
### Round Robin 
정의 : 각 프로세스에게 동일한 크기의 할당시간(Time quantum)을 부여하며, quantum을 모두 소모한 프로세스는 다시 ready queue에 맨 뒤로 가서 대기하게 된다.
선점/비선점형 : 선점형
장점 : starvation 문제가 해결된다.
단점 : (SJF보다 turnaround time이 일반적으로 더 길다.)
      적절한 Time quantum 선점이 어렵다. (퀀텀이 길면 길수록 First Come First serve와 유사하며 짧을 수록, 컨텍스트 스위칭으로 인핸 오버헤드가 커진다.)

---
### Priority Scheduling 
정의 : 우선순위가 가장 높은 프로세스에게 CPU를 할당하는 스케쥴링이다.
선점/비선점형 : 선점형, 비선점형 모두 가능 (구현에 따라 다르다)
기아 문제가 존재할 수 있으며 aging 기법등으로 이를 일부 해결할 수 있다. 
(큐에 오래 대기하는 프로세스들의 우선순위를 점진적으로 증가시키는 방식)


