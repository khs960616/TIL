### 요쳥 매핑

@RequestMapping 어노테이션의 여러 속성

```
params : 쿼리 스트링에 대한 조건을 지정할 수 있다. 

ex) 
@RequestMapping(value = "/test-params", params = "조건...")
다음과 같이 조건을 설정할 수 있다.
* params="field",   // field 라는 쿼리파라미터가 존재하는 요청에 대해서만 처리한다. 
* params="!field"   // field 라는 쿼리파라미터가 존재하지 않는 요청에 대해서만 처리한다. 
* params="field=1"  // field 라는 쿼리파라미터의 값이 1인 요청에 대해서만 처리한다. 
* params="field!=1"  /
* params = {"field1=2","field2=2"}


headers : 헤더에 대한 조건을 지정할 수 있다.

consumes : http request header에 Content-type에 따라 컨트롤러의 동작 유무를 결정할 수 있다.
 
produces : http response header에 Content-type을 지정한다. 이 때 클라이언트쪽에서 http header Accept로
받아들일 수 없는 타입을 서버에서 보내는 경우

클라이언트 쪽에서 응답을 제대로 받을 수 없다.
```

headers, consumes, produces 모두 params와 동일하게 여러 조건에 따라 컨트롤러의 호출 유무를 결정할 수 있다.

---
@PathVariable (경로변수)

@RequestMapping 류의 어노테이션과 함께 사용하면, 요청 url을 템플릿화 하여 사용할 수 있다. 
