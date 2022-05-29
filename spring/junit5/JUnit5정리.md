[JUnit5 User Guide](https://junit.org/junit5/docs/current/user-guide/)


1. JUnit 테스트 클래스는, 각 테스트 메소드를 수행할 때마다 새로운 오브젝트로 만들어진다.
  - 따라서 별도의 설정을 하지 않는 한, 각 테스트 메소드별로 서로 영향을 미치지 않는다.
  - JUnit5에서부터는 @TestInstance(LifeCycle.PER_CLASS) 어노테이션을 활용하여 클래스 단위의 라이프 사이클을 가지도록 할 수 있다. (때때로 동일 환경에서 여러 테스트 진행 시 사용)


2. Application Context의 경우, 테스트 과정에서 한번 만들어진 동일한 Object가 재사용되어진다.
  - @DirtiesContext : 클래스 레벨, 메서드 레벨에 적용 가능한 어노테이션이며 해당 어노테이션과 property를 사용하여 기존에 사용하던 Context를 close하고 새로운 Context로 
    사용이 가능하다. 
  - https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/annotation/DirtiesContext.MethodMode.html
  - https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/annotation/DirtiesContext.ClassMode.html
