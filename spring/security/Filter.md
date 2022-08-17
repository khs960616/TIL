### Servelt Filter

(Filter) 필터 : 필터링 하는 객체 

(서블릿 요청 또는  정적 자원에 대한 클라이언트의 request와,  서버쪽에서 클라이언트로 보내는 reponse에 대해 필터링 작업을 수행한다.)

다음과 같은 상황에 필터를 활용할 수 있다.
```
1) Authentication Filters  
2) Logging and Auditing Filters
3) Image conversion Filters 
4) Data compression Filters 
5) Encryption Filters 
6) Tokenizing Filters 
7) Filters that trigger resource access events 
8) XSL/T filters 
9) Mime-type chain Filter  
```
---
### FilterConfig

필터가 초기화 되는 시점에 init() 매서드의 매개변수로 전달되는 객체이다. 

웹 어플리케이션에 대한 설정값 정보, ServletContext에 대한 참조를 가지고 있으므로, 필터 초기화 시 해당 정보들을 활용할 수 있다. 

---
# Filter 인터페이스

``` java
public void init(FilterConfig filterConfig) throws ServletException {}
// 웹 컨테이너가 필터를 만든 후, 해당 메서드에 적혀있는 초기화 작업을 진행한다. filterConfig에 존재하는 여러 값들을 초기화 과정에 활용할 수 있다.
```

``` java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws java.io.IOException, ServletException {}
// request와 response에 대한 작업을 처리한 후 필터 체인을 따라 각 요청들을 이동시킨다.
// 만일 필터체인을 통해 다음 필터로 값을 넘겨주는 경우, 현재 필터에서 가공된 request와 response객체를 넘겨주게 된다.
```

``` java
public void destroy() {}
// 웹 컨테이너에서 필터가 삭제될 때 호출된다.
```


---

[오라클 독스](https://docs.oracle.com/javaee/6/api/javax/servlet/Filter.html)
[참고 블로그](https://javacan.tistory.com/entry/58)
