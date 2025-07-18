## ftrace

```
su -s
cd /sys/kernel/debug/tracing

1) 추적하고 싶은 함수들 설정
echo xxx > set_graph_function

2) 특정 PID만 추적
echo [PID] > set_ftrace_pid  

3) 트레이싱 시작 or 종료  (0 또는 1 입력)
echo 0 or 1  > tracing_on
```

## perf

sudo perf trace -p <PID> > trace.log 2>&1  (유저레벨에서 호출한 system call들 수행하는데 얼마나 걸린지 간단하게 체크)

## blktrace

실제 bio요청이 어떻게 처리되고 있는지 필요할 때 활용, blkparse iowatcher와 함께 텍스트나 시각화한 자료형태로 뽑기 좋다. 
