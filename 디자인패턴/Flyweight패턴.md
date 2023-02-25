## Flyweight Pattern

* 사전적 정의 : 동일하거나 유사한 객체들 사이에 가능한 많은 데이터를 서로 공유하도록하여 메모리 사용량을 최소화하는
소프트웨어 디자인 패턴

--- 
## 플라이 웨이트 패턴의 구조 

Flyweight Factory : 실제 공유될 객체를 생성, 공유해주는 역할을 수행하는 클래스

Flyweight : 공유에 사용될 객체들을 나타내는 인터페이스 

ConcreteFlyweight : Flyweight 인터페이스를 구현한, 실제 Factory에 의해 공유되어 사용될 클래스

ConcreteFlyweight형태로 만들어진 객체를 Factory를 통해 생성을 제한하며, 다른 객체에 공유해서 메모리를 절약할 수 있는 방식이다.

---
### 실제 적용된 사례

- Java String Constant Pool

-  Wrapper Class 들의 valueOf 메서드들도 적용되었다고 적혀있는 글들이 있었는데 그냥 이건 일부 범위의 수를 캐싱해놓고 사용하는 것으로 보는게 맞지 않을까? 
-  특정 구간(-128~127의 경우는 그냥 매번 새로운 객체 만들어서 뱉음)

```java

    // 
   public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }

   public static BigInteger valueOf(long val) {
        // If -MAX_CONSTANT < val < MAX_CONSTANT, return stashed constant
        if (val == 0)
            return ZERO;
        if (val > 0 && val <= MAX_CONSTANT)
            return posConst[(int) val];
        else if (val < 0 && val >= -MAX_CONSTANT)
            return negConst[(int) -val];

        return new BigInteger(val);
    }
    
    // 실수형의 경우는 캐싱 별도로 안하는 듯 보임 
    public static Double valueOf(double d) {
        return new Double(d);
    }
    
    public static Float valueOf(float f) {
        return new Float(f);
    }
    
    // https://stackoverflow.com/questions/8561710/why-does-the-double-valueof-javadoc-say-it-caches-values-when-it-doesnt 
   
```

---
### REF
https://stackoverflow.com/questions/11189155/is-javas-string-intern-a-flyweight 

https://ko.wikipedia.org/wiki/%ED%94%8C%EB%9D%BC%EC%9D%B4%EC%9B%A8%EC%9D%B4%ED%8A%B8_%ED%8C%A8%ED%84%B4


--- 
패턴에 대해서 찾아보며, 해당 패턴을 고려해서 사용할 경우는 
```
프로그램에서 사용되는 특정 클래스가, 너무 많은 인스턴스를 생성해서 메모리를 많이 차지하는 경우 
  1. 객체의 속성을 분석해서 "한번 정해지면 불변하는 속성" 이 있는지를 관찰한다.
  2. 해당 속성을 많은 객체에서 공유해서 사용되고 있는지를 판단한다. 
  3. 만일 그렇다면 해당 속성들을 ConcreteFlyweight 형태로 만들 수 있는지를 판단해보고, 적합하다면 패턴을 적용한 후 다른 사이드 이펙트가 없는지 관찰 후 사용한다. 
```
정도로 이해했다. 

또한 싱글톤 패턴과의 차이는 객체의 생성을 클래스 자체에서 하는지, 팩토리에서 제어하도록 구성하는지 정도의 차이가 있겠으나, 결국 내가 이해한 바로는

두 패턴 다 공유하는 상태가 없고, 메모리를 절약해야되는 환경에서 고려해볼 수 있는 패턴이라고 생각이 든다.

---

https://lee1535.tistory.com/106 

https://velog.io/@hoit_98/%EB%94%94%EC%9E%90%EC%9D%B8-%ED%8C%A8%ED%84%B4-Flyweight-%ED%8C%A8%ED%84%B4

- 대략 이러한 형태로 나무, Circle등 예제로 하는 한국 블로그 글들이 많은데, 읽다가 혼란이 와서 정리함.

해당 예제들의 시작점은 java Swing에서 그래픽을 이용해서 화면에 뭔가를 그리는 데 사용되는 코드들의 형태로 보인다. 

코드만 봤을 때 Flyweight 객체가 단순히 나무와 원을 의미하는 것이라면 자주 변경되는 상태인 (y, x) 위치 정보를 가지고 있으므로 Flyweight 패턴이라고 부를수 없을 것 같다. 

해당 코드에서 각 객체들은 "어떠한 UI를 그리기 위한 설계도의 역할을 하는 객체"라고 생각하면 Flyweight 패턴을 적용한 것이라고 볼 수 있을 것 같다.

(Swing에서 실제 UI로써 그려지는 오브젝트들은 해당 객체를 참고해서 Swing의 메서드를 통해 각기 다른 오브젝트가 생성될 것이므로)



https://refactoring.guru/design-patterns/flyweight 해당 사이트에 있는 예제들이 패턴을 이해하는데 큰 도움을 줌 
