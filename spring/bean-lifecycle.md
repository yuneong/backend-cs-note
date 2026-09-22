## Bean 생명주기 (Bean Lifecycle)

### Bean 생명주기란?
스프링 컨테이너가 빈을 생성하고 초기화한 뒤, 사용이 끝나면 소멸시키기까지의 전체 과정.

```text
컨테이너 생성 -> 빈 생성(생성자 호출) -> 의존관계 주입 -> 초기화 콜백 -> 사용 -> 소멸 전 콜백 -> 컨테이너 종료
```

생성자 호출과 의존관계 주입은 별개의 단계다. 생성자 주입이면 생성자 호출 시점에 의존관계까지 한 번에 끝나지만, 필드/수정자 주입이면 객체가 먼저 생성된 뒤 스프링이 필드/setter로 의존성을 나중에 채워 넣는다.

### 초기화/소멸 콜백을 지정하는 3가지 방법
| 방법                                               | 특징                                          |
|--------------------------------------------------|---------------------------------------------|
| @PostConstruct / @PreDestroy                     | 스프링 공식 권장, 자바 표준 어노테이션 (jakarta.annotation) |
| InitializingBean / DisposableBean 인터페이스 구현       | 스프링 인터페이스에 코드가 강하게 결합됨, 비권장                 |
| @Bean(initMethod = "...", destroyMethod = "...") | 내가 수정할 수 없는 외부 라이브러리 클래스에 콜백을 지정할 때 사용      |

#### 예시 1) ```@PostConstruct``` / ```@PreDestroy``` (권장)
```kotlin
@Component
class DatabaseConnector {
    
    @PostConstruct
    fun connect() {
        println("DB 연결 시작") // 의존성 주입이 끝난 뒤 호출 보장 <- 초기화 콜백
    }
    
    @PreDestroy
    fun disconnect() {
        println("DB 연결 종료") // 컨테이너 종료 직전 호출
    }
}
```

#### 예시 2) 외부 라이브러리 클래스에 콜백 지정
```kotlin
// 내가 직접 어노테이션을 붙일 수 없는 외부 라이브러리 클래스라고 가정
class ExternalResource { // 라이브러리 안에 이미 이렇게 정의되어 있음 (수정 불가 상태)
    fun init() = println("초기화")
    fun close() = println("종료")
}

@Configuration
class AppConfig {
    
    @Bean(initMethod = "init", destroyMethod = "close")
    fun externalResource(): ExternalResource = ExternalResource()
}
```

### 왜 생성자보다 ```@PostConstruct```가 더 안전한가? (면접 단골 질문)
- 생성자 주입을 쓴다면 생성자가 끝난 시점에 이미 모든 의존성이 채워져 있어서 생성자 안에서 초기화 로직을 실행해도 대부분 안전하다.
- 하지만 필드/수정자 주입은 객체 생성(생성자 호출) 이후에 스프링이 의존성을 채워 넣기 때문에, 생성자 안에서 아직 주입되지 않은 필드를 사용하면 ```NullPointException```이 발생할 수 있다.
- ```@PostConstruct```는 주입 방식과 무관하게 "의존관게 주입이 모두 끝난 뒤 호출됨"이 스프링에 의해 보장되므로, 의존성을 사용하는 초기화 로직은 생성자보다 ```@PostConstruct```에 두는 게 일관되게 안전하다.

```kotlin
@Component
class OrderService {
    
    @Autowired
    private lateinit var repository: OrderRepository
    
    init {
        // 위험: 필드 주입은 이 시점에 repository가 아직 null일 수 있음
    }
    
    @PostConstuct
    fun setup() {
        // 안전: 이 시점에는 repository 주입이 보장됨
        repository.warmUpCache()
    }
}
```

### 싱글톤 vs 프로토타입 - 소멸 콜백 차이
기본적으로 스프링 빈은 아무 설정을 안 하면 전부 싱글톤이다. 프로토타입으로 만들고 싶으면 ```@Scope```를 명시적으로 붙여야 한다.

```kotlin
// 싱글톤 빈 (기본값, @Scope 생략 가능)
@Component
class OrderService(
    private val repository: OrderRepository,
)

// 프로토타입 빈 (명시적으로 지정해야 함)
@Component
@Scope("prototype")
class ReportGenerator {
    var buffer: StringBuilder = StringBuilder() // 요청마다 상태를 새로 가져야 하는 경우
}
```

- 싱글톤 빈: 컨터에너 안에 인스턴스가 딱 하나만 존재하고, ```context.getBean(OrderService::class.java)```를 몇 번 호출해도 같은 객체가 반환된다. 컨테이너가 생성부터 소멸까지 생명주기 전체를 관리한다. ```ApplicationContext.close()```가 호출되면(또는 정상 종료 시 등록된 shutdown hook에 의해) ```@PreDestroy```콜백이 호출된다.
- 프로토타입 빈: ```@getBean()```을 호출할 때마다(또는 다른 빈에 주입될 때마다) 매번 새 인스턴스가 생성된다. 컨테이너는 빈을 생성해서 넘겨준 뒤로는 관리하지 않는다. 즉 소멸 콜백을 스프링이 호출해주지 않는다. 리소스 정리가 필요하면 개발자가 직접 해줘야 한다.

```kotlin
val a = context.getBean(ReportGenerator::class.java)
val b = context.getBean(ReportGenerator::class.java)
println(a === b) // 싱글톤이면 true, 프로토타입이면 false
```

### 예상 면접 질문
- Bean 생명주기 콜백을 지정하는 방법 3가지를 설명해보세요.
- ```@PostConstruct```가 생성자보다 안전한 이유는?
- 프로토타입 스코프 빈도 소멸 콜백이 호출되나요? (아니오 - 컨테이너가 생성 이후 관리를 넘기기 때문에 직접 관리해야함)
- ```InitializingBean``` / ```DisposableBean```보다 ```@PostConstruct``` / ```@PreDestroy```를 권장하는 이유는? (인터페이스 구현 방식은 스프링 API에 코드가 종속되어 결합도가 높아짐)





















