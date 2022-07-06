Java Lambda

람다 : 프로그래밍 언어에서 사용되는 개념으로 익명함수(Anomymous Function)을 지칭하는 용어, 메서드를 하나의 expression으로 표현한 것이다.

자바에서 람다식은 일급 객체이다. (lambda expression은 함수의 매개변수로 전달될 수 있으며, 메서드의 결과로 반환될 수도 있다.

람다식을 통해 메서드를 마치 변수처럼 다루는 것이 가능해진다. 

```
Lambdas give us the ability to encapsulate a single unit of code block and pass on to another code.

It can also be considered as anonymous function 

which doesn’t have any function name but has list of parameters, function body, returns result and even throws exceptions
```

Lambda Expression은 세가지 요소로 구성된다.

1. (단수, 혹은 복수 개의) parameters
2. Function Body
3. Arrow(->) : parameter들과 Function Body를 구별해주는 Seperator

~~~java
// 다음의 메서드와 각 람다식은 동일한 역할을 수행한다.
int max(int a, int b) {
  return a > b ? a : b;
}

// 람다식의 function body에 return 문이 존재하는 경우 {}를 생략할 수 없다.
(int a, int b) -> { return a > b? a : b; }

// 반환값이 있는 메서드를 람다로 바꾸는 경우, return문 대신 expression으로 대체가 가능하다. (이 떄 expression)뒤에는 ;를 생략한다.
(int a, int b) -> a > b ? a : b

// 람다식에 선언된 매개변수의 타입 추론이 가능한 경우 생략이 가능하다. (대다수의 경우 생략이 가능하다.)
(a, b) -> a > b ? a : b
~~~

~~~
Parameter types are optional.
If you have single parameter then both parameter type and parenthesis are optional.
If you have multiple parameters, then they should be enclosed with in parenthesis.
For multiple statements in function body should be enclosed with in courly braces.
If lambda body exnclosed inside courly braces then return keyward is required in case your behavior returns value.
~~~



### REF

[자바 람다](https://java-8-tips.readthedocs.io/en/stable/lambdas.html)

[람다 대수](https://ko.wikipedia.org/wiki/%EB%9E%8C%EB%8B%A4_%EB%8C%80%EC%88%98)

Java의정석 3.0 Chapter 14
