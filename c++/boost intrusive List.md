## Intrusive List

```
Instrusive Data Structure는 컨테이너의 존재하는 item의 data부분만 선언하는 것이 아닌, 실제 llink, rlink 정보도 item이 포함하도록 하는 자료구조 구현이다.

non-instrusive Data Structure의 경우, data부분은 컨테이너 내부의 노드로 복사하고, 실제 link정보는 컨테이너 내부에서 관리한다.

따라서, copy하는 비용이 추가적으로 발생한다. (다만, 프리미티브 타입을 직접 넣을 수 있는 장점이 있음)

boost:intrusive::list -> Instrusive한 list를 제공한다.
```

## Hook
```
hook: 실제 link정보를 나타내며, boost를 사용한 intrusvie list에 넣을 멤버는 list_base_hook를 가져야한다. 

(list_member_hook으로 실제 멤버에서 어떤 것이 hook으로 사용되고 있는지 알 수 있다.)

이 때 hook에는 다양한 link mode들이 존재한다.
```

## link mode

intrusive 자료구조에서 사용되는 link 모드는 3가지가 지원된다. 
```
safe_link: default로 설정되는 link모드이다.  (필요시 더 찾아볼것)

normal_link: (필요시 더 찾아볼것)

auto_unlink: non-constant time container에서만 사용이 가능하나, 다음의 두 가지 feature를 추가적으로  활용할 수 있다.
1. hook의 소멸자가 호출될때, 만약 해당 hook이 컨테이너에 속해있다면, 컨테이너에서 해당 노드를 제거한다.

2. unlink()함수를 명시적으로 호출할 수 있기 때문에, 컨테이너에 대한 정보 없이, 컨테이너에서 링크를 끊을 수 있다. 
```

---
#### 추가정보

constant time container란, 컨테이너의 크기를, 항상 상수시간에 얻을 수 있는 컨테이너이다. 
이를 위해서, 컨테이너에서 컨테이너 사이즈를 hold하고 있을 수 있는 member를 추가로 가지며 (메모리 더 먹음), 매 insert, delete 연산시에 해당 사이즈를 조절한다.

-> 따라서 auto_unlink의 경우에는, 컨테이너를 통해서 링크를 끊는 것이 아니므로 constant time container와 호환이 불가능함.
