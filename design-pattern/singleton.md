## Singleton 패턴

### GoF Singleton 패턴이란?
클래스의 인스턴스를 오직 하나만 생성되로록 보장하는 디자인 패턴.

핵심은 이렇다.
1. 생성자를 외부에서 호출 못 하게 막는다 (```private```)
2. 클래스 내부에서 유일한 인스턴스를 직접 들고 있는다
3. 그 인스턴스에 접근하는 전역적인 방법(보통 static 메서드)을 제공한다

```java
// Java - 고전적인 Singleton 구현
public class DatabaseConnection {
    private static final DatabaseConnection INSTANCE = new DatabaseConnection();

    private DatabaseConnection() { } // 생성자를 private으로 막아서 외부에서 new 못 하게 함
    
    public static DatabaseConnection getInstance() {
        return INSTANCE; // 항상 같은 인스턴스 반환
    }
}

DatabaseConnection.getInstance(); // 어디서든 이걸로만 접근, 항상 동일 객체
```

### Kotlin에서는 ```object``` 키워드로 훨씬 간단하게
Kotiln은 언어 차원에서 Singleton 패턴을 문법으로 지원한다.

```kotlin
object DatabaseConnection {
    fun connect() { println("연결") }
}

DatabaseConnection.connect() // 클래스이자 동시에 유일한 인스턴스
```

```object```로 선언하면 컴파일러가 알아서 private 생성자 + static 인스턴스 + thread-safe한 초기화(JVM 클래스 로딩 매커니즘을 이용)까지 다 처리해준다.
그래서 Kotlin 백엔드에서는 "Singleton 패턴을 직접 구현해봐라"보다는, 이 원리를 설명할 줄 아는게 더 중요하다.

### 왜 필요한가 (의도)
- 공유 자원(커넥션 풀, 설정값, 캐시 등)을 앱 전체에서 하나만 유지하고 싶을 때
- 여러 인스턴스가 생기면 상태 불일치, 불필요한 메모리/리소스 낭비가 생길 수 있는 경우

### 멀티스레드 환경에서의 함정 (면접 단골)
고전적인 Lazy 초기화 방식(필요할 때 처음 생성)은 멀티스레드 환경에서 동시에 두 스레드가 ```getInstance()```를 호출하면 인스턴스가 두 개 생길 수 있는 문제가 있다.

```java
// 위험한 Lazy Singleton (스레드 안전하지 않음)
public class DatabaseConnection {
    private static DatabaseConnection instance;
    
    public static DatabaseConnection getInstance() {
        if (instance == null) { // 스레드 A, B가 동시에 여기 도달 가능
            instance = new DatabaseConnection(); // 둘 다 새로 생성 -> 인스턴스 2개!
        }
        return instance;
    }
}
```

해결책은 ```synchronized```, Double-Checked Locking, 혹은 애초에 **클래스 로딩 시점에 즉시 생성(Eager Initialization)**하는 방식이다.
Kotlin의 ```object```는 내부적으로 JVM의 클래스 초기화가 스레드 세이프하다는 보장을 이용해서 이 문제를 깔끔하게 피해간다.

### 중요한 함정 - "Spring Bean Singleton Scope"와는 다른 개념이다
면접에서 자주 나오는 헷갈림 포인트.
Bean 생명주기 노트에서 다룬 "스프링 빈의 싱글톤 스코프"와 디자인 패턴의 Singleton은 범위(scope)가 다르다.

|            | GoF Singleton 패턴                    | Spring Bean Singleton Scope                                       |
|------------|-------------------------------------|-------------------------------------------------------------------|
| 보장 범위      | JVM 클래스 로더(사실상 애플리케이션) 전체에서 인스턴스 1개 | 하나의 스프링 컨테이너(ApplicationContext) 안에서 빈 1개                         |
| 강제 방법      | 생성자를 private으로 막아서 언어/클래스 차원에서 강제   | 그냥 평범한 클래스 - 누구나 new로 여러 개 만들 수 있음. 컨테이너가 "관리상" 하나만 만들어서 재활용하는 것뿐 |
| 컨테이너가 2개면? | 여전히 인스턴스 1개 (클래스 자체가 막아놓음)          | 컨테이너가 2개면 빈도 2개 생김 (각 컨테이너마다 하나씩)                                 |

```kotlin
@Service // private 생성자가 아님 - 그냥 평범한 클래스
class OrderService {
    fun placeOrder() { }
}

val a = OrderService() // 스프링 빈이 아닌, 그냥 new로 만들면 당연히 별개의 인스턴스가 생김
val b = OrderService() 

println(a === b) // false -> 스프링이 "관리"해줄 때만 싱글톤이 보장되는 거지, 클래스 자체가 싱글톤을 강제하는 게 아님
```

즉 Spring의 싱글톤 스코프는 GoF Singleton 패턴을 "흉내"내는 것에 가깝고, 실제로 언어/클래스 차원에서 강제하는 진짜 Singleton 패턴은 아니다.

### 예상 면접 질문
- Singleton 패턴이 해결하려는 문제는 무엇인가요?
- Kotlin에서 Singleton 패턴을 어떻게 구현하나요? (```object``` 키워드)
- 멀티스레드 환경에서 Lazy Singleton 구현 시 어떤 문제가 생길 수 있나요?
- Spring Bean의 싱글톤 스코프와 GoF Singleton 패턴의 차이는 무엇인가요?




