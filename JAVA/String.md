String vs StringBuffer/StringBuilder 클래스의 차이

1. String은 immutable의 속성을 가진다. (새로운 스트링을 만드는 경우 새로운 인스턴스를 만들게 된다.)
2. 변하지 않는 문자열을 자주 읽어들이는 경우 String을 사용하는 것이 좋다. 그러나 문자열의 추가, 수정, 삭제 등이 빈번하다면, 사용하지 않는 것이 좋다.

StringBuffer / StringBuilder는 mutable하다.

StringBuffer는 동기화 키워드를 제공하여 thread-safe하다.
StringBuilder는 동기화를 지원하지 않기 때문에 멀티쓰레드 환경에서 사용하는 것이 적합하지 않다. (단일 쓰레드에서의 성능은 더 뛰어나다.)

String의 경우, 리터럴을 그대로 대입하는 경우 Constance pool에 있는 객체를 활용하지만,
new String으로 객체를 새롭게 할당한다는 특성을 지닌다. 
