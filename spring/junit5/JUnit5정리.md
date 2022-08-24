[JUnit5 User Guide](https://junit.org/junit5/docs/current/user-guide/)


1. JUnit 테스트 클래스는, 각 테스트 메소드를 수행할 때마다 새로운 오브젝트로 만들어진다.
  - 따라서 별도의 설정을 하지 않는 한, 각 테스트 메소드별로 서로 영향을 미치지 않는다.
  - JUnit5에서부터는 @TestInstance(LifeCycle.PER_CLASS) 어노테이션을 활용하여 클래스 단위의 라이프 사이클을 가지도록 할 수 있다. (때때로 동일 환경에서 여러 테스트 진행 시 사용)


2. Application Context의 경우, 테스트 과정에서 한번 만들어진 동일한 Object가 재사용되어진다.
  - @DirtiesContext : 클래스 레벨, 메서드 레벨에 적용 가능한 어노테이션이며 해당 어노테이션과 property를 사용하여 기존에 사용하던 Context를 close하고 새로운 Context로 
    사용이 가능하다. 
  - https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/annotation/DirtiesContext.MethodMode.html
  - https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/annotation/DirtiesContext.ClassMode.html

---
### 테스트 인스턴스의 라이프 사이클

@TestInstance(LifeCycle.PER_CLASS) : 해당 클래스내에 존재하는 모든 테스트 메서드를 동일한 인스턴스를 이용하여 테스트 한다.
@TestInstance(LifeCycle.PER_METHOD) : 각 테스트 메서드별로 새로운 인스턴스를 생성하여 사용한다.


삽질한 코드 .. 
```java 
import org.springframework.stereotype.Component;
import java.util.*;

@Component
public class ItemRepository {
    private static final Map<Long, Item> repository = new HashMap<>();
    private static long sequence = 0L;

    public Item saveItem(Item item) {
        item.setId(sequence);
        repository.put(sequence++, item);
        return item;
    }

    public void deleteItems(Long id) {
        if(repository.getOrDefault(id, null) == null) return;
        repository.remove(id);
    }

    public void updateItems(Item item) {
        repository.put(item.getId(), item);
    }

    public Item getItem(Long id){
        return repository.get(id);
    }

    public List<Item> getItems() {
        return new ArrayList<>(repository.values());
    }

    public void clear(){
        repository.clear();
        sequence = 0L;
    }

    public int getSize() {
        return repository.size();
    }
}

```

```
package inflearn.itemservice.itemtest;

import inflearn.itemservice.basic.item.Item;
import inflearn.itemservice.basic.item.ItemRepository;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;

@SpringBootTest
public class ItemRepositoryTest {
    private final ItemRepository repository = new ItemRepository();

    @AfterEach
    void DB_초기화() {
        repository.clear();
    }

    @Test
    void 사용자_추가() {
        Item item1 = Item.builder()
                .price(1000L)
                .name("상품1")
                .quantity(20L)
                .build();

        Item item2 = Item.builder()
                .price(2000L)
                .name("상품2")
                .quantity(30L)
                .build();

        Item item3 = Item.builder()
                .price(3000L)
                .name("상품3")
                .quantity(40L)
                .build();
        repository.saveItem(item1);
        repository.saveItem(item2);
        repository.saveItem(item3);

        assertEquals(item1.getId(), 0);
        assertEquals(item2.getId(), 1);
        assertEquals(item3.getId(), 2);
        assertEquals(repository.getSize(), 3);
    }

    @Test
    void 사용자_삭제() {
        Item dummy1 = Item.builder()
                .name("DUMMY1")
                .price(2000L)
                .quantity(20L)
                .build();
        repository.saveItem(dummy1);
        assertEquals(repository.getSize(), 1);

        repository.deleteItems(0L);
        assertEquals(repository.getSize(), 0);
    }


    @Test
    void 사용자_조회() {
        Item dummy1 = Item.builder()
                .name("DUMMY1")
                .price(2000L)
                .quantity(20L)
                .build();
        repository.saveItem(dummy1);
        assertEquals(repository.getSize(), 1);

        Item getItem = repository.getItem(0L);
        assertEquals(dummy1, getItem);
    }

    @Test
    void 사용자_전체조회() {
        Item item1 = Item.builder()
                .price(1000L)
                .name("상품1")
                .quantity(20L)
                .build();

        Item item2 = Item.builder()
                .price(2000L)
                .name("상품2")
                .quantity(30L)
                .build();

        Item item3 = Item.builder()
                .price(3000L)
                .name("상품3")
                .quantity(40L)
                .build();
        repository.saveItem(item1);
        repository.saveItem(item2);
        repository.saveItem(item3);

        List<Item> items = repository.getItems();
        assertEquals(items.size(), 3);
    }

    @Test
    void 사용자_업데이트() {
        Item item1 = Item.builder()
                .price(1000L)
                .name("상품1")
                .quantity(20L)
                .build();
        repository.saveItem(item1);

        item1.setName("변경 상품명");
        item1.setQuantity(1L);
        item1.setPrice(2L);
        repository.updateItems(item1);

        Item updateInstance = repository.getItem(0L);
        assertEquals(updateInstance.getName(), "변경 상품명");
        assertEquals(updateInstance.getQuantity(), 1L);
        assertEquals(updateInstance.getPrice(), 2L);
    }
}
```
@TestInstance(LifeCycle.PER_METHOD) 테스트 클래스 인스턴스는 새롭게 만들어진다해도, 

해당 클래스 인스턴스에서 테스트중인 코드 내부에 스태틱 멤버가 있고, 그 스태틱 멤버를 사용한 로직이 존재하는 경우 사용에 주의하자.

위 테스트 코드의 경우 인스턴스 멤버 변수인 private final ItemRepository repository = new ItemRepository(); 는 테스트 메서드마다 새롭게 만들어진다.

그러나 ItemRepository자체는  private static final Map<Long, Item> repository = new HashMap<>();, private static long sequence = 0L;

다음 두 가지의 클래스 멤버변수를 가지고 있기 때문에, @AfterEach 또는 @BeforeEach로 매번 초기화 해주지 않는다면, 각 테스트 메서드끼리 서로 영향을 주게 된다. 

---
@BeforeAll :
```
Denotes that the annotated method should be executed before all @Test, @RepeatedTest, @ParameterizedTest, and @TestFactory methods in the current class; analogous to JUnit 4’s @BeforeClass. Such methods are inherited – unless they are hidden, overridden, or superseded, (i.e., replaced based on signature only, irrespective of Java’s visibility rules) – and must be static unless the "per-class" test instance lifecycle is used.
```

@AfterAll : 

```
Denotes that the annotated method should be executed after all @Test, @RepeatedTest, @ParameterizedTest, and @TestFactory methods in the current class; analogous to JUnit 4’s @AfterClass. Such methods are inherited – unless they are hidden, overridden, or superseded, (i.e., replaced based on signature only, irrespective of Java’s visibility rules) – and must be static unless the "per-class" test instance lifecycle is used.
```

현재 클래스에 모든 Test 메서드가 실행되기 전, 종료된 후 1번 실행될 메서드에 사용하는  어노테이션들이다. 

@BeforeEach와 @AfterEach는 각 테스트 메소드의 실행 전후로 1번 실행되는 메서드이다. 

---
