## Bin
- 재사용 가능한 즉, Free Chunk들을 크기 단위로 관리(binning)하는 역할을 하는 자료 구조
-> pt malloc쪽 코드보면서 가져왔던 내용 같음.  douglea 구현마다 좀 다른건가 싶기도한데.. bin마다 handling하는 사이즈들도 구현에 따라 일부 조금씩 다른거 같긴하다. 
 

## Bin의 종류 

1. small bin (0x20 ~ 0x400)
```
chunk크기가 MIN_LARGE_SIZE(64bit system의 경우 1024byte, 32bit system 512byte)보다
작은 chunk들을 포함하는 62개의 small bin이 존재하며 각 bin마다
한 가지 크기의 chunk만 저장되기 때문에 자동으로 정렬이 되어 삽입 및 제거가 빠르다.

특징
1. FIFO(First In First Out)를 사용하며 double-liked list로 관리된다.
(먼저 해제된 chunk가, 먼저 할당에 다시 사용된다)

2. 서로 다른 크기의 62개의 small bin 존재한다.

```

2. large bin (0x400 이상의 Chunk들을 관리함)
```
정해진 고정된 크기로 Chunk들을 관리하지 않으며, 특정 단위로 범위로 끊어서 관리한다. 따라서 bin index에서,
여러 가지 사이즈의 Chunk가 존재할수 있으며, 사이즈 별로 정렬 되어있다. (따라서 메모리 할당과 반환 속도가 bin중에서 가장 느리다)

특징
1. FIFO(First In First Out)를 사용하며 double-liked list로 관리

2. 63개의 bin을 사용하며 특정 범위 단위로 관리

3. 하나의 bin에 다양한 크기의 chunk들을 보관 (

4. bin내에서 크기 별로 정렬되어 할당의 효율성 향상

largebin[0] ~ largebin[31] (32개) : 0x40(64) bytes씩 증가
largebin[32] ~ largebin[47] (16개) : 0x200(512) bytes씩 증가
largebin[48] ~ largebin[55] (8개) : 0x1000(4,096) bytes씩 증가
largebin[56] ~ largebin[59] (4개) : 0x8000(32,768) bytes씩 증가
largebin[60] ~ largebin[61] (2개) : 0x40000(262,144) bytes씩 증가
largebin[61] (1개) : 이외의 남은 크기
```

3. fast bin 
```
인접한 Chunk들끼리 병합하지 않으며, 단일 고정 크기의 Chunk들만 처리하므로 메모리 할당 / 해제가 가장 빠르다.

상한 값 : 128byte(64*8/4)
chunk size 종류 : 32, 48, 64, 80, 96, 112, 128, 144, 160, 176byte
추가로 64bit에서 일반적으로 32(0x20) ~ 128(0x80)까지 7개의 bin만 사용함

특징
1. LIFO(Last In First Out, stack과 동일한 방식)를 사용하며 single-linked list로 관리된다.
2. sinle-linked list로 관리되어 bk는 사용되지 않음
3. in use bit가 설정되지 않게 하여 병합되지 않는다.
4. fast bin에서 관리되는 chunk들은 완전히 해제, 병합되지 않으므로 주기적으로 consolidates을 거친다. 
```

4. Unsorted bin
```
chunk가 해제되는 경우,
unsorted bin에 먼저 각 chunk들을 넣은 후, 적절한 크기의 chunk를 먼저 할당한다. (일종의 캐시 역할)
```

각 bin에 속하는 Chunk들은 크기순, 가장 오래된 순을 우선순위로 정렬된다. 

---
#### REF
(Data Binning이란 정의된 기준에 따라 각각의 개별적인 데이터값을 저장하는 특정한 bin(구간, interval) 또는 group으로 묶는 과정을 의미한다)
