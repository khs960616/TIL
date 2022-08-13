### Enum

서로 관련이 있는 여러 개의 상수 집합을 정의할 때 사용하는 자료형

java에서 Enum은 하나의 완전한 클래스로 취급된다. 

Enum은 Method(static) 영역에 저장된다. 

Enum 사용의 장점
1. 매직넘버를 사용하는 것보다 코드가 의미하는 바가 명확해진다.
2. 잘못된 값을 사용함으로써 발생하는 위험성이 사라진다. 

---
Enum과 생성자 
enum의 각 열거형 상수에 추가 속성을 부여할 수 있다.

```java
public enum ExceptionType {
    NOT_FOUNT_RESOURCE("E001", "데이터가 존재하지 않습니다."),
    NOT_EXIST_ROOM("R001", "존재하지 않는 방입니다."),
    ALREADY_FULL_ROOM("R002", "이미 사용자가 가득찬 방입니다."),
    ALREADY_EXIT_ROOM("R003", "이미 종료된 방입니다.");
    private String code;
    private String message;

    private ExceptionType(String code, String message) {
        this.code = code;
        this.message = message;
    }

    public String getCode() {
        return this.code;
    }

    public String getMessage() {
        return this.message;
    }
}

```

추가 속성을 enum 클래스의 필드에 설정해주고, getter 메소드를 통해 속성을 가져다 쓸 수 있다.

enum의 경우 다른 클래스들과 달리 생성자의 접근제어자를 private으로 지정해야한다.
- 고정된 상수들의 집합으로써 런타임이 아닌 컴파일타임에 모든 값을 알고 있어야한다.
- 다른 패키지, 클래스에서 enum 타입에 접근해서 동적으로 어떤 값을 정해줄 수 없다. 
- 싱글톤을 일반화한다. 

---
Enum 메소드

static method
1. valueOf(String arg) : enum에서 string 값을 가져온다.
2. values() : enum의 요소들을 순서대로 enum 타입의 배열로 리턴한다. (카피본을 가져오는 것이므로 자주 호출하면 좋지 않다고 함)

```java
public class Test {
    enum EnumTest{
        TEST1,
        TEST2,
        TEST3
    }

    public static void main(String[] args) {
        EnumTest[] value1 = EnumTest.values();  
        EnumTest[] value2 = EnumTest.values(); 

        System.out.println(value1.hashCode()); // 13648335
        System.out.println(value2.hashCode()); // 312116338     매번 복사되서 새롭게 생성된다. 
    }
}

```

instance method
1. name() : 호출된 값의 이름을 String으로 리턴
2. ordinal : 해당 값이 enum에 정의된 순서를 리턴한다.
3. compareTo(E o) : enum과 지정된 객체의 순서를 비교한다. (작은 경우 음수, 같으면 0 크면 양의 정수 리턴)
4. equals(Object other) : 지정된 객체가 enum 정수와 같은 경우 true 리턴 


ordinal 메소드의 경우 잘 생각해서 사용해야한다.

단순히 enum 내부의 순서값을 int로 반환하기 때문에, 중간에 enum을 수정하다, 중간에 새로운 enum 요소가 끼어드는 경우

oridinal을 사용한 모든 로직이 망가질 가능성이 존재한다. 

---

경험상 oridinal을 사용하지 않더라도 enum의 순서값을 이용해 무언가 상태를 표시하는 경우 이런 위험성이 항상 존재하는 것 같다. 예전에 땅땅치킨 유지보수 할 때, 

배달 상태 관련된 값들을 enum으로 관리한 적이 있는데, 이 때 중간 중간 요구사항이 바뀌어서 배달관련 상태 코드를 추가해야 됬던 적이 있는데, 

이 경우에도 어플리케이션 로직을 다 일일히 수정해야했음.. 
