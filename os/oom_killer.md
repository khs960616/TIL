## OOM Killer
- 시스템 메모리가 부족해지는 상황에서 강제로 프로세스를 종료시켜 메모리를 확보시키는 커널의 기능
- 시스템 메모리가 임계 수준 이하로 떨어지는 경우 동작한다.


## oom_badness 및 score 설정 방법 

이는 프로세스의 메모리 사용량, 프로세스 우선순위등을 데몬 내부적으로 사용하여 계산한다.
oom_badness() 함수를 통해 게산하며 0~1000사이 값으로 계산되며 OOM Score가 높을수록 우선순위가 높아진다. 

oom_badness 메소드의 순위 선정 방법 (코드 분석전 자료 조사) 
1. 특정 프로세스를 죽임으로써, 최소한 양의 프로세스만 잃어야한다.
2. 많은 메모리를 회수할 수 있어야한다.
3. leak이 발생되지 않는 프로세스는 선택하지 않는다.
4. 사용자가 지정한 프로세스순으로 죽인다.

- 모든 프로세스는  /proc/pid/oom_score 에 해달 프로세스의 oom_score가 기록 되어 있다.

## [우선순위 조작방법]
/proc/pid/oom_adj 값을 조정함으로써 OOM Killer의 순위 설정을 조작할 수 있다.
해당 값은 -17~15사이의 값을 가질수 있으며, 낮은 값 일수록 우선순위에서 밀려난다.

/proc/pid/oom_score_adj 값을 조정함으로써 OOM Killer의 순위 설정을 조작할 수 있다.
해당 값은 -1000~1000사이의 값을 가질수 있으며, 낮은 값 일수록 우선순위에서 밀려난다.

## [로그 확인 방법]
OOM Killer가 동작한 경우 /var/log/messages 경로에 oom 관련 메시지가 기록된다.




## [관련 자료]

https://www.kernel.org/doc/gorman/html/understand/understand016.html 
https://elixir.bootlin.com/linux/v5.6/source/mm/oom_kill.c#L199 
