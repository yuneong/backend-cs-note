## AOP (Aspect-Oriented Programming, 관점 지향 프로그래밍)

### AOP란?
로깅, 트랜잭션, 권한 체크처럼 **여러 모듈에 공통으로 걸쳐있는 관심사(횡단 관심사, Cross-cutting Concern)** 를 비즈니스 로직에서 분리해서 한 군데 모아두고, 필요한 곳에 "끼워 넣는" 프로그래밍 방식.

```kotlin
// AOP 미적용 - 로깅 코드가 비즈니스 로직에 섞여있고, 모든 Service 클래스마다 반복됨
class OrderService {
    fun placeOrder(orderId: Long) {
        val start = System.currentTimeMillis()
        println("주문 시작")
        try {
            repository.save(orderId)
        } finally {
            println("소요시간: ${System.currentTimeMillis() - start}")
        }
    }
}
```

```kotlin
// AOP 적용 - 로깅 관심사를 Aspect로 분리, OrderService는 비즈니스 로직만 남음
@Aspect
@Component
class LoggingAspect {
    
    @Around("execution(* com.example.service.*.*(...))") // 서비스 패키지의 모든 메서드에 적용
    fun logExecutionTime(joinPoint: ProceedingJoinPoint): Any? {
        val start = System.currentTimeMillis()
        val result = joinPoint.proceed() // 실제 원본 메서드 호출
        println("소요시간: ${System.currentTimeMillis() - start}")
        return result
    }
}

class OrderService {
    fun placeOrder(orderId: Long) {
        repository.save(orderId)
    }
}
```

### 어떻게 "끼워 넣는" 게 가능한가? - 프록시 기반 동작
스프링 AOP는 원본 객체 대신 그걸 감싸는 프록시 객체를 만들어서 컨테이너에 등록하는 방식으로 동작한다.

```text
클라이언트 → [프록시 객체] → 원본 OrderService 객체
                ↑
        여기서 로깅/트랜잭션 등 부가기능 실행 후 원본 메서드 호출  
```

컨테이너가 빈을 생성하는 과정에는 ```BeanPostProcessor```라는 후처리기가 개입할 수 있는데, AOP 담당 후처리기(```AnnotationAwareAspectJAutoProxyCreator```)가 여기서 개입한다.

```text
1. 컨테이너가 OrderService 원본 객체를 생성 (진짜 인스턴스)
2. AOP 후처리가 개입: "이 빈이 Pointcut에 해당하나?"
3. 해당하면 -> 원본 객체를 감싸는 프록시 객체를 새로 만듦
4. 컨테이너의 싱글톤 저장소에는 원본이 아니라 이 "프록시"가 저장됨
```

즉 ```context.getBean(OrderService::class.java)```를 호출하면 실제로 반환되는 건 진짜 ```OrderService```가 아니라 런타임에 새로 만들어진 프록시 객체다.

### 직접 만들어보는 JDK 동적 프록시
스프링 없이 표준 API만으로 똑같은 원리를 재현하면 이렇다.

```kotlin
import java.lang.reflect.InvocationHandler
import java.lang.reflect.Method
import java.lang.reflect.Proxy

interface OrderService {
    fun placeOrder(orderId: Long)
}

class OrderServiceImpl : OrderService {
    override fun placeOrder(orderId: Long) {
        println("주문 처리: $orderId")
    }
}

// "가로채는 놈" - 프록시가 메서드 호출을 받으면 이 invoke()로 위임됨
class LoggingHandler(private val target: Any) : InvocationHandler {
    override fun invoke(proxy: Any, method: Method, args: Array<out Any>?): Any? {
        println("[로그] ${method.name} 호출 시작")
        val result = method.invoke(target, *(args ?: emptyArray())) // 진짜 원본 메서드 호출
        println("[로그] ${method.name} 호출 종료")
        return result
    }
}

val realService = OrderServiceImpl()

// Proxy.newProxyInstance가 런타임에 OrderService 인터페이스를 구현한 클래스를 즉석에서 생성
val proxy = Proxy.newProxyInstance(
    OrderService::class.java.classLoader,
    arrayOf(OrderService::class.java),
    LoggingHandler(realService)
) as OrderService

proxy.placeOrder(1L)

// [로그] placeOrder 호출 시작
// 주문 처리: 1
// [로그] placeOrder 호출 종료
```

```proxy```는 ```OrderService``` 타입이지만 ```OrderServiceImpl```이 아니라, ```Proxy.newProxyInstance```가 런타임에 새로 만든 별개의 클래스 인스턴스다.
```proxy.placeOrder(1L)``` 을 호출하면 실제로는 ```LoggingHandler.invoke()``` 가 먼저 실행되고, 그 안에서 ```method.invoke(target, ...)``` 로 리플렉션을 통해 진짜 ```realService.placeOrder(1L)``` 을 대신 호출해준다.

스프링 AOP의 ```@Around``` + ```joinPoint.proceed()``` 가 바로 이 ```invoke()``` + ```method.invoke(target, ...)``` 에 대응한다.

### 프록시를 만드는 방법 2가지 - JDK 동적 프록시 vs CGLIB
| 방식         | 조건                                | 동작 방식                         |
|------------|-----------------------------------|-------------------------------|
| JDK 동적 프록시 | 대상 클래스가 인터페이스를 구현하고 있을 때 | 그 인터페이스를 구현한 프록시 클래스를 런타임에 생성 |
| CGLIB      | 인터페이스가 없을 때 (Spring Boot 2.0+ 기본 값) | 대상 클래스를 상속받은 자식 클래스를 런타임에 생성(바이트코드 조작) |

```kotlin
// 인터페이스가 있는 경우 -> JDK 동적 프록시로 OrderService 인터페이스를 구현한 프록시 생성 가능
interface OrderService { fun placeOrder(orderId: Long) }
class OrderServiceImpl : OrderService { override fun placeOrder(orderId: Long) { } }

// 인터페이스가 없는 경우 -> CGLIB로 OrderService를 상속한 프록시 생성
class OrderService { fun placeOrder(orderId: Long) { } }
```

CGLIB는 상속 기반이라 ```final``` 클래스나 ```final``` 메서드는 오버라이딩(프록시)이 불가능하다.
Kotlin 클래스와 메서드는 기본이 ```final```이라서, CGLIB 프록시 대상이 되려면 ```open```으로 열려있어야 한다.
```kotlin-spring``` 컴파일러 플러그인이 ```@Component```, ```@Service``` 등이 붙은 클래스를 자동으로 ```open```으로 만들어주기 때문에 실무에서는 크게 신경쓰지 않아도 되지만, 원리를 알아두면 "왜 이 클래스가 AOP가 안먹지?" 같은 문제를 디버깅할 때 도움이 된다.

### 용어 정리
- Aspect: 흩어진 관심사를 모듈화한 것 (```LoggingAspect``` 같은 클래스)
- Advice: 실제로 실행할 부가 기능 코드 (```@Around```, ```@Before```, ```@After``` 등)
- Pointcut: 어디에 적용할지 지정하는 표현식 (execution(* com.example.service.*.*(..)))
- JoinPoint: Advice가 적용될 수 있는 지점 (메서드 실행 시점)

### 예상 면접 질문
- AOP가 왜 필요한지, 어떤 문제를 해결하는지 설명해보세요.
- 스프링 AOP는 내부적으로 어떻게 동작하나요? (프록시를 만들어서 컨테이너에 등록 -> 메서드 호출을 가로채서 부가 기능 실행 후 원본 호출)
- JDK 동적 프록시와 CGLIB의 차이는 무엇인가요?
- Kotlin에서 CGLIB 프록시를 쓸 때 주의할 점은? (클래스/메서드가 기본 ```final```이라 ```open```으로 열어줘야 함)












