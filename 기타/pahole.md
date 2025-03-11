## pahole [Packing Hole]

C/C++ 구조체의 메모리 레이아웃을 분석하여, 구조체 내 필드들 사이에 발생하는 padding 또는 "hole"을 확인하는 데 사용된다.

분석을 위해서는, 빌드 시 디버깅 심볼을 포함한 상태에서 빌드된 바이너리가 있어야한다. 

```
sudo apt-get install dwarves
```

install 후 사용 가능하다. 옵션들은 https://linux.die.net/man/1/pahole에서 참고해서 사용
