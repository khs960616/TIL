**Thymeleaf is a modern server-side Java template engine for both web and standalone environments.**

### 타임리프 특징
1. 서버사이드 HTML 렌더링 : 백엔드 서버에서 HTML을 동적으로 렌더링 하는 용도로 사용한다. 
2. 네츄럴 템플릿 : 순수 HTML을 유지하기 때문에 웹 브라우저에서 파일을 직접 열어도 내용 확인이 가능하다. 또한 서버를 통해 뷰 템플릿을 거치면 동적으로 변경된 결과를 확인할 수 있다.
3. 스프링 통합 지원 
---
### 타임리프 기능

타임리프 사용 선언 : \<html xmlns:th="http//www.thymeleaf.org"\>

---
#### escape
escape :  <, > 등 HTML에서 사용되는 특수문자가 존재하는 경우 이를 %lt, %gt등의 HTML 엔티티로 변경하는 것 

---
#### 일반 텍스트 출력 방법 (escape 자동 지원)

th:text=${attribute명}   // 해당 태그의 text를 변수의 value로 변경한다. 

[[${attribute명}]] : 태그 내부에서 직접 텍스트를 사용하는 경우 이를 사용한다.

---

#### 일반 텍스트 출력 방법 (unescape)
th:utext,   [(${attribute명})]

---
#### thymeleaf 기본 객체 

스프링과 연동해서 사용시 다음의 몇가지 객체 접근 방식을 활용할 수 있다. 

- ${#request}   // HttpServlertRequest
- ${#response}  // HttpServlertResponse
- ${#session}   // HTTPSession
- ${$servletContext}
- ${#locale}
- ${param} // 쿼리 파라미터 접근 
- ${#locale}

${@beanName}으로 스프링 빈에 접근 또한 가능하다. 

---
#### thymeleaf 유틸리티 객체 
https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html#expression-utility- objects

https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html#appendix-b-expression- utility-objects

---
#### Spring Expression Languag
The Spring Expression Language (SpEL for short) is a powerful expression language that supports querying and manipulating an object graph at runtime.

#{SpEL 표현식}의 형태로 사용하며 SpEL 표현식의 값을 evaluation 한다.

${someobject.property} 특정 속성값에 대한 참조는 다음과 같이 표현한다. 

사용 시 참고 사항 
https://atoz-develop.tistory.com/entry/Spring-SpEL-Spring-Expression-Language
https://docs.spring.io/spring-framework/docs/3.0.x/reference/expressions.html

---
### Elvis 연산자 
${expression} ?: "대체할 표현식"

데이터가 null 인 경우 ?: 뒤에 오는 표현식으로 대체한다. 또한 

?: _ 처럼 (_)를 사용하는 경우 no-operation을 의미하며 데이터가 null인 경우 thymeleaf 태그를 무효화한다. 

---
### attribute 추가

th:attrappend="추가할속성 = ' value'"    (html element에 특정 속성에 값을 추가한다.)

th:classappend="'추가할 클래스'" (html element에 특정 클래스 값을 추가한다. 이 경우 알아서 띄어쓰기를 해준다.)

---
### th:each
th:each="각 element를 의미하는 변수명 각 element에 대한 반복 상태를 의미하는 변수명 :${반복의 대상이 되는 컬렉션}" 형태로 사용이 가능하다. 

첫 번째 위치하는 변수명은 실제 컬렉션에 들어있는 element이며, 

두 번째 위치하는 변수명은 (생략시 첫 번쨰 변수명 +stat으로 자동으로 생성) 반복 상태를 의미한다. 

index, count, size, even, odd, first , lass, current 등 현재 객체가 컬렉션에서 어디에 위치하는지, 짝수번째 index인지 홀수번쨰 index인지,

어떠한 객체인지 등의 정보를 추가로 알려준다.

### th:if, th:unless, th:switcvh
```
th:if="${expression}"

th:unless="${expression}"

th:switch="${expression}"
th:case = "value"
th:case = "value"
th:case = "value"
```

th:if : expression의 값이 true가 아니라면 해당 태그는 렌더링 되지 않는다. (unless는 false인 경우에만 렌더링함, if문의 반대) 

th:swtich-case : th:switch로 지정해준 value와 일치하는 case만 렌더링한다.

---
### <th:block> </th:block>
타입 리프에서 제공하는 태그이며 렌더링시 제거된다. 여러 html 태그에 걸쳐서 th:each를 적용해야 되는 경우 해당 block을 적용하면 쉽게 처리가 가능하다. 

---
### Fragment
```html
<htmlTag th:template = "fragment이름 (param1, param2....)">  
...
</htmlTag>
```
형태로 특정 html 태그를 fragment로 지정하고, 이름과 parameter를 부여하면 thymeleaf로 렌더링 되는 다른 html 파일에서 

해당 fragment가 존재하는 html 파일 경로와, fragment의 이름을 통해 재활용할 수 있다. 

th:insert="~{경로 :: fragment 이름}" 

(사용된 태그의 하위로 element를 insert한다. 단 사용한 태그 하위의 기존에 자식 노드가 존재했다면, 해당 자식 노드들은 다 사라지는 것으로 보임)

th:replace="~{경로 :: fragment 이름}" (사용된 태그를 fragment로 대체시킨다.)

경로가 단순한 경우 "~{경로 :: fragment 이름}"없이 그냥 "/경로 :: fragment이름" 형태의 단순 표현식으로 경로와 Fragment 이름을 명시할 수 있음

---
### 템플릿 레이아웃 

사용할 레이아웃을 fragment로 먼저 생성해놓는다. 이때 레이아웃마다 값을 변화시키거나 추가하고자하는 값이 있다면 parameter로 받아서 페이지별로 다르게 렌더링한다. 

~{::htmlTag} 형태로 현재 페이지내의 dom element를 직접 넘길 수 있다. 만약 해당 html tag가 여러 개라면, 여러 개가 하나의 변수로 다 같이 넘어가게 된다.  

fragment에 필요한 몇 가지 요소를 parameter로 넘겨서 페이지 마다 조금씩 다르게 사용하는 좀 더 확장된 개념이라고 강의에서는 설명한다 

(근데 뭐 그냥 의미적으로 크게 나눠서 사용할 이유는 굳이 있나 싶음. 그냥 ~{::tag} 형태로 현재 html에 존재하는 태그를 넘길 수 있다는 것 정도로 알아 두면 될 듯 하다.)

( th:replace로 실제 레이아웃으로 대체되는 부분 하위에 존재하는 태그만 넘어갈 줄 알았는데, 실제 레이아웃을 불러다 쓰는 html 파일에 있는 태그 다 넘기니 주의 할 것. 

---
### REF
인프런 - 스프링 MVC 2편
https://www.thymeleaf.org/
