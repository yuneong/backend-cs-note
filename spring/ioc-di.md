## IoC (Inversion of Control) / DI (Dependency Injection)

### IoC란?
객체의 생성과 생명주기 관리를 개발자가 아닌 프레임워크(스프링 컨테이너)가 대신 담당하는 원칙.

일반적인 프로그래밍에서는 개발자가 직접 ```new```로 객체를 생성하고 의존 관계를 맺지만, IoC를 적용하면 그 제어권이 프레임워크로 역전(Inversion)된다.

```kotlin
// Ioc 미적용 - 개발자가 직접 제어
class OrderService {
    private val repository = OrderRepository() // 직접 생성, 강한 결합
}

// Ioc 적용 - 컨테이너가 생성해서 주입
class OrderService(
    private val repository: OrderRepository // 컨테이너가 넣어줌
)
```

### DI란?
IoC를 구현하는 대표적인 방법. 객체가 필요로 하는 의존성을 외부(컨테이너)에서 주입해주는 패턴.

```kotlin
@Service
class OrderService(
    private val repository: OrderRepository, // 컨테이너가 OrderRepository 빈을 찾아 주입
) {
    fun placeOrder(orderId: Long) = repository.findById(orderId)
}

@Repository
class OrderRepository {
    fun findById(id: Long): Order? = TODO()
}
```
```OrderService```는 ```OrderRepository```를 어떻게 생성하는지 전혀 몰라도 된다. 컨테이너가 ```@Repository```로 등록된 빈을 찾아서 생성자에 넣어준다.

### DI 방식 3가지
| 방식 | 특징 |
|---|---|
| 생성자 주입 | 스프링 공식 권장 |
| 필드 주입 | 테스트/불변성 취약 |
| 수정자(Setter) 주입 | 선택적 의존성에만 제한적 사용 |

```kotlin
// 1. 생성자 주입 (권장)
@Service
class OrderService(
    private val repository: OrderRepository,
)

// 2. 필드 주입
@Service
class OrderService {
    @Autowired
    private lateinit var repository: OrderRepository
}

// 3. 수정자(Setter) 주입
@Service
class OrderService {
    private lateinit var repository: OrderRepository

    @Autowired
    fun setRepository(repository: OrderRepository) {
        this.repository = repository
    }
}
```

### 왜 생성자 주입을 권장하는가?
1. 불변성(Immutability) 보장 - kotlin ```val``` / Java ```final``` 로 선언 가능, 런타임에 의존성이 바뀔 위험 제거
2. 순환 참조를 컴파일/기동 시점에 발견
3. 필수 의존성 누락을 컴파일 타임에 감지 - 생성자 파라미터가 없으면 컴파일 자체가 안 됨
4. 테스트 용이성 - 스프링 컨테이너 없이 단위 테스트 가능
5. Spring 4.3+부터는 생성자가 하나뿐이면 ```@Autowired``` 생략 가능

> Kotlin 백엔드라면 보통 ```class```의 primary constructor에 ```private val```로 선언하는 게 사실상 표준 패턴이다.

#### 예시1) 순환 참조 감지 시점 차이
```kotlin
// A -> B, B -> A 순환 참조 상황

// 필드 주입: 앱은 정상적으로 뜨지만, A를 실제로 사용하는 순간 StackOverflowError 위험
@Service
class AService {
    @Autowired
    private lateinit var bService: BService
}
@Service
class BService {
    @Autowired
    private lateinit var aService: AService
}

// 생성자 주입: 컨테이너가 A를 만들려면 B가, B를 만들려면 A가 필요해서
// 애플리케이션 기동 시점에 BeanCurrentlyInCreationException 발생 -> 즉시 문제 인지 가능
@Service
class AService(private val bService: BService)
@Service
class BService(private val aService: AService)
```

#### 예시2) 테스트 용이성
```kotlin
// 생성자 주입이면 스프링 컨테이너를 띄우지 않고도 순수 단위 테스트 가능
class OrderServiceTest {
    
    private val repository = mockk<OrderRepository>()
    private val orderService = OrderService(repository) // new로 직접 생성 + mock 주입
    
    @Test
    fun `주문 조회 시 repository를 호출한다`() {
        every { repository.findById(1L) } returns Order(id = 1L)
        
        orderService.placeOrder(1L)
        
        verify { repository.findById(1L) }
    }
}
```
필드 주입(```lateinit var```)이었다면 리플렉션으로 강제로 값을 넣어주거나 ```@SpringBootTest```로 컨테이너를 띄워야 해서 테스트가 무거워진다.

### 컨테이너 관점
- ```BeanFactory```: DI의 기본 컨테이너, 지연 로딩(요청 시점에 Bean 생성)
- ```ApplicationContext```: ```BeanFactory```를 확장, 컨테이너 구동 시점에 싱글톤 Bean을 미리 생성(eager loading), 이벤트/국제화 등 부가 기능 제공 - 실무에서는 거의 이걸 사용

```kotlin
// ApplicationContext는 기동 시점에 OrderService 싱글톤 빈을 미리 생성해둔다.
val context = AnnotationConfigApplicationContext(AppConfig::class.java)
val orderService = context.getBean(OrderService::class.java) // 이미 생성되어 있던 빈을 반환
```

### 예상 면접 질문
- IoC와 DI의 차이는? (IoC는 원칙/개념, DI는 그 구현 방법 중 하나)
- 생성자 주입이 필드 주입보다 나은 이유 3가지 이상 말해보세요
- 순환 참조가 발생하면 어떻게 해결하나요? (설계 재검토가 우선, 불가피하면 ```@Lazy``` 또는 setter 주입으로 우회)
```kotlin
// 불가피하게 순환 참조를 우회해야 할 때: @Lazy로 프록시를 주입해서 실제 사용 시점까지 생성을 미룸
@Service
class AService(
    @Lazy private val bService: BService,
)
```






















