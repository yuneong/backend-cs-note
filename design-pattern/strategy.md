## Strategy 패턴

### 핵심 아이디어
알고리즘(로직)을 인터페이스로 뽑아내서, 런타임에 구현체를 교체할 수 있게 만드는 패턴.
"여러 가지 방식으로 할 수 있는 일"이 있을 때, 그 방식(전략)을 캡슐화해서 서로 바꿔 끼울 수 있게 만든다.

### 문제 상황 - Strategy 없이 짜면
```kotlin
class PaymentService {
    fun pay(method: String, amount: Int) {
        if (method == "CARD") {
            println("카드로 $amount 원 결제")
        } else if (method == "KAKAO_PAY") {
            println("카카오페이로 $amount 원 결제")
        } else if (method == "TOSS") {
            println("토스로 $amount 원 결제")
        }
        // 결제 수단이 추가될 때마다 이 메서드를 계속 고쳐야 함
    }
}
```

결제 수단이 늘어날 때마다 ```PaymentService```를 계속 수정해야 하고, 조건문이 끝없이 늘어난다. 
이건 **OCP(개방-폐쇄 원칙: 확장엔 열려있고 변경엔 닫혀있어야 한다)**를 위반하는 대표적인 사례다.

### Strategy 패턴 적용
```kotlin
// 1. 전략(알고리즘)을 인터페이스로 추출
interface PaymentStrategy {
    fun pay(amount: Int)
}

// 2. 각 전략을 별도 클래스로 구현
class CardPayment : PaymentStrategy {
    override fun pay(amount: Int) = println("카드로 $amount 원 결제")
}

class KakaoPayPayment : PaymentStrategy {
    override fun pay(amount: Int) = println("카카오페이로 $amount 원 결제")
}

// 3. Context는 전략을 "들고 있다가 위임"만 함 - 구체적인 방법은 모름
class PaymentService(private val strategy: PaymentStrategy) {
    fun pay(amount: Int) = strategy.pay(amount) // 어떤 전략인지 신경 안 씀
}

// 4. 사용
val service = PaymentService(KakaoPayPayment())
service.pay(10000)
```

새로운 결제 수단이 추가돼도 ```PaymentStrategy```를 구현한 클래스 하나만 새로 추가하면 되고, 기존 ```PaymentService``` 코드는 단 한 줄도 건드릴 필요가 없다.
이게 OCP를 지키는 방식이다.

### Spring에서는 DI와 결합해서 더 강력해진다
Spring은 같은 인터페이스를 구현한 빈이 여러 개면, 그 빈들을 ```Map```이나 ```List```로 한 번에 주입받을 수 있다.

```kotlin
@Component("CARD")
class CardPayment : PaymentStrategy {
    override fun pay(amount: Int) = println("카드로 $amount 원 결제")
}

@Component("KAKAO_PAY")
class KakaoPayPayment : PaymentStrategy {
    override fun pay(amount: Int) = println("카카오페이로 $amount 원 결제")
}

@Service
class PaymentService(
    private val strategies: Map<String, PaymentStrategy>, // 빈 이름 -> 구현체, 스프링이 자동으로 모아서 주입
) {
    fun pay(method: String, amount: Int) {
        val strategy = strategies[method] ?: throw IllegalArgumentException("지원하지 않는 결제 수단")
        strategy.pay(amount)
    }
}
```

이러면 ```if-else```나 ```when```문이 아예 사라지고, 새 결제 수단은 ```@Component```로 새 클래스 하나 추가하는 것만으로 자동 등록된다.
```PaymentService```는 전혀 수정할 필요가 없다.
이게 "Strategy 패턴 + DI"가 실무에서 가장 많이 쓰이는 조합이다.

### 예상 면접 질문
- Strategy 패턴이 해결하는 문제가 무엇인가요? (조건문 분기 폭증, OCP 위반)
- Strategy 패턴과 Spring DI를 어떻게 함께 쓸 수 있나요? (```Map<String, Interface>``` 주입으로 분기 없이 구현체 선택)
- Strategy 패턴을 적용하기 전/후로 OCP 관점에서 어떤 차이가 생기나요? 
  - 적용 전에는 기능을 추가할 때마다 기존 클래스를 수정해야 했지만, 적용 후에는 새 클래스를 추가하기만 하면 되고 기존 클래스는 손대지 않아도 된다
  - ```PaymentService```가 구체 클래스 대신 **추상화에 의존**하게 됐기 때문 (DIP(의존성 역전 원칙)과도 연결됨)


















