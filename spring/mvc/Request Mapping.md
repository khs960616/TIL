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

---
어노테이션 기반의 스프링 컨트롤러는 다음과 같은 어노테이션들을 활용해 변수에 http request로 서버로 도착한 값들을 바인딩해서 사용할 수 있다. 

```
@RequestHeader("원하는 헤더명") : Request Header를 꺼내서 쓸 수 있다. 

전체 헤더가 모두 필요한 경우 @RequestHeader MultiValueMap<String, String> headers 으로 헤더 전체를 받을 수 있으며, 특정 헤더만 필요한 경우 

value값에 필요한 헤더명을 쓰면 된다.

@CookieValue(value = "원하는 쿠키명", required = 필수여부) 를 이용해 쿠키값을 빼서 사용할 수 있다. 
```

---

@ModelAttribute 타입 (변수명)


매개변수를 생성한 후, 해당 매개변수의 객체 타입의 프로퍼티에 대한 Setter를 호출해서 요청 파라미터의 값을 객체에 바인딩 해준다.

1. Setter가 없는 프로퍼티에 대해서는 Request에 값을 실어보내도 값을 세팅할 수 없다.
2. @RequestParam과 같이 쿼리스트링 또는 (x-www-form-urlencoded)로 보내는 값은 바인딩 할 수 있다.
3. RequestBody에 json 형태로 객체를 담아서 보내는 경우, 해당 어노테이션으로는 값을 바인딩 할 수 없다.

---
