### 특성정보(traits)
컴파일 도중 주어진 타입에 대한 정보를 얻을 수 있게 하는 객체 

관례적으로 구조체로 표현하는 것으로 굳어져 있으며, 구현체는 trait class라 부른다. 


http://www.fifi.org/doc/cpp-3.0/libstdc++/html_user/type__traits_8h-source.html  

<type_traits> 쪽 소스 보면 이해가 쉬움

템플릿을 활용해서 라이브러리 만들때, 개별 클래스별로 필요한 trait class 정의,  

traits class별로 메서드를 override해두고,  실제 템플렛 메서드에서는 이러한 override된 함수를 사용하면, client 코드에서 비교적 깔끔하게 나오는 듯. 

기존 std에서 사용하는  char_traits, numeric_limits도 참고.

적합한 케이스만 있다면, 이러한 방식으로 라이브러리 구축해놓고

autogen하고 같이 활용하기에도 좋아보임 
