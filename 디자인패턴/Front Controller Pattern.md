# Front Controller Pattern 

웹 어플리케이션을 설계할 때, 여러 코드(웹 요청을 처리하는 여러 컨트롤러)에서 중복되는 몇 가지 패턴의 목록을 리스트화 시킨 후 한 곳에서 처리하는 패턴

```
The front controller software design pattern is listed in several pattern catalogs 
and related to the design of web applications
```

프론트 컨트롤러는 웹 어플리케이션에서 반드시 사용되야만 하는 것은 아니지만, 

다양한 웹 요청에 대한 페이지 또는 데이터를 처리하는 객체를 탐색하는 코드를 요청마다 작성하는 것보다,

하나의 프론트 컨트롤러에서 여러 웹 요청에 대한 탐색 작업을 담당하도록 구현하면 유지 보수하기 더 좋으며, 탐색을 제어하기도 더 편하다. 

ex) 웹 요청에 대한 핸들링, 특정 요청에 대한 input의 필터링, 캐싱등 공통적인 여러 기능을 한 곳에 모아서 중복을 줄일 수 있다.



#### 참고자료 

[WikiPedia](https://en.wikipedia.org/wiki/Front_controller)
