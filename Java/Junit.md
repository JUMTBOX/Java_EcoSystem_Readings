## Junit 사용법
- 의존성 확인
```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```
- 이 ```spring-boot-starter-test```안에 ```JUnit 5(Jupiter)```, ```Mockito```, ```AssertJ``` 가 포함되어 있다.

### 기본 문법

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*; // 기본 assertions

class CalculatorTest {
    @Test
    void 더하기_기본() {
        // given
        int a = 1;
        int b = 2;

        // when
        int result = a + b;

        // then
        assertEquals(3, result); // (기댓값, 실제값)
    }
}
```
- ```@Test```: 이 어노테이션이 붙은 메서드가 테스트 메서드이다.
- **Assertions (검증)**
  - ```assertEquals(expected, actual)```
  - ```assertTrue(condition)```
  - ```assertThrow(예외클래스, () -> 코드)```
- **공통 준비/정리 메서드**
  - ```@BeforeEach```: 각 테스트 실행 **직전**에 실행 
  - ```@AfterEach```: 각 테스트 실행 **직후**에 실행 

```java
import org.junit.jupiter.api.*;

class CalculatorTest {

    @BeforeEach
    void setUp() {
        // 각 테스트 전에 공통 준비
    }

    @AfterEach
    void tearDown() {
        // 각 테스트 후에 정리
    }

    @Test
    void 케이스1() { /*...*/ }

    @Test
    void 케이스2() { /*...*/ }
}
```

### 서비스 테스트 방식 2가지
1. 순수 단위 테스트 (스프링 컨테이너 X)
2. Spring 통합 테스트 (```@SpringBootTest```로 컨텍스트 로딩) - 추후 기술 예정...


**순수 단위 테스트**
- JPA 레포지토리를 전부 mock으로 대체하고 ```서비스 로직만 검증```

```java
// 테스트 대상 서비스
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final MemberRepository memberRepository;
    
    public Order createOrder(Long memberId, int amount) {
        Member member = memberRepository.findById(memberId)
                .orElseThrow(() -> new IllegalArgumentException("회원 없음"));

        Order order = new Order(member, amount);
        return orderRepository.save(order);
    }
}
```

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.BDDMockito.given;
import static org.mockito.ArgumentMatchers.any;

@ExtendWith(MockitoExtension.class) // <--- JUnit 5에서 Mockito 확장을 붙이는 방법
class OrderServiceTest {

    @Mock // <--- JPA Repository 인터페이스를 테스트에서 가짜로 만들어 준다.
    OrderRepository orderRepository;

    @Mock
    MemberRepository memberRepository;

    @InjectMocks               // <--- Mock들이 자동으로 생성자에 주입된 상태로 서비스 인스턴스를 만들어 준다. 
    OrderService orderService; // 위 두 Mock이 주입된 상태로 생성됨

    @Test
    void createOrder_정상생성() {
        // given
        Member member = new Member(1L, "홍길동");
        given(memberRepository.findById(1L))
                .willReturn(Optional.of(member));
        given(orderRepository.save(any(Order.class)))
                .willAnswer(invocation -> invocation.getArgument(0)); // 그냥 그대로 반환

        // when
        Order order = orderService.createOrder(1L, 10000);

        // then
        assertThat(order.getMember().getId()).isEqualTo(1L);
        assertThat(order.getAmount()).isEqualTo(10000);
    }
}
```
**컬렉션이나 배열의 내용까지 재귀적으로 비교하려면?**
- ```Arrays.deepEquals```
- AssertJ 의 ```usingRecursiveComparison```
- JUnit5 의 ```assertIterableEquals```


## ⚠️주의할 점..⚠️
- 검증용 메서드 ```Assertions.assertEquals```는 ```int```, ```long```, ```boolean```, ```char```, ```byte```, ```short```, ```float```, ```double``` 용 메서드가 따로 존재한다.
<br/> 이들은 결론적으로 일반적인 값 비교```==```를 수행한다고 한다.
- 그러나 참조 타입을 검사할 때는 전적으로 ```Object.equals```에 의존한다고 한다.
- 이 말은 곧 ```String```, ```Integer``` 등과 같은 ```Object.equals```를 오버라이드한 JDK 클래스들은 자신들이 재정의(=오버라이드)한 비교 방식을 따른다는 이야기
- 반대로 우리 MP에서 사용하는 대부분의 검증이 필요한 값들은 개발자들이 직접 구성한 도메인 클래스.. 즉, ```Object.equals```를 별도로 오버라이드 하지 않은 클래스들이다.
- 별도의 재정의 한 ```Object.equals```가 없기 때문에 ```Assertions.assertEquals```에서 원본 ```Object.equals```메서드가 호출 되어질 것이고.. 
  <br/> 원본 ```Object.equals```메서드 내부를 살펴본다면 하기와 같이 작성되어 있는데
```java
    public boolean equals(Object obj) {
        return (this == obj);
    }
```
- 객체끼리의 ```==``` 비교 수행시 참조 주소가 같은지만을 비교하기 때문에 비교하는 두 객체의 구성이 같아도 다른 인스턴스라면 다르다고 판단되어 테스트가 ```failed``` 되므로 이를 주의해야한다. 
  <br/> 여담으로 JS에서 객체끼리의 ```==, ===```비교도 동일하다.