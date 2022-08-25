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


---
### REF
인프런 - 스프링 MVC 2편
https://www.thymeleaf.org/
