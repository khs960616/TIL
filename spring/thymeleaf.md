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
#### Spring Expression Languag
The Spring Expression Language (SpEL for short) is a powerful expression language that supports querying and manipulating an object graph at runtime.

#{SpEL 표현식}의 형태로 사용하며 SpEL 표현식의 값을 evaluation 한다.

${someobject.property} 특정 속성값에 대한 참조는 다음과 같이 표현한다. 

사용 시 참고 사항 
https://atoz-develop.tistory.com/entry/Spring-SpEL-Spring-Expression-Language
https://docs.spring.io/spring-framework/docs/3.0.x/reference/expressions.html

---
### REF
인프런 - 스프링 MVC 2편
https://www.thymeleaf.org/
