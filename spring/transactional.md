## @Transactional (전파 레벨, 격리 수준, Self-invocation)

### ```@Transactional```도 결국 AOP 프록시다
```@Transactional```이 붙은 메서드를 호출하면, 실제로는 프록시가 그 호출을 가로채서 "트랜잭션 시작 -> 원본 메서드 실행 -> 성공하면 커밋, 예외 터지면 롤백"을 대신 해준다. (프록시 동작 원리는 AOP 노트 참고)

```kotlin
@Service
class OrderService(private val repository: OrderRepository) {
    
    @Trasactional
    fun placeOrder(orderId: Long) {
        repository.save(orderId) // 이 사이에서 예외 발생 시 전체 롤백
        repository.updateStock(orderId)
    }
}
```

개념적으로는 프록시가 이렇게 감싸는 것과 같다.
```kotlin
fun placeOrder(orderId: Long) {
    transactionManager.begin()
    try {
        realService.placeOrder(orderId) // 진짜 메서드 호출
        transactionManager.commit()
    } catch (e: Exception) {
        transactionManager.rollback()
        throw e
    }
}
```

### 커밋/롤백 기준 - 어떤 예외일 때 롤백되는가
"예외가 터지면 롤백된다"고 했지만 정확히는 모든 예외가 롤백을 유발하는 건 아니다.
스프링의 기본 규칙은 이렇다.

#### Checked exception vs Unchecked exception
예외 클래스 계층 구조로 보면 이렇게 나뉜다.

```text
Throwable
├── Error                        (언체크 - OutOfMemoryError 등 심각한 시스템 오류)
└── Exception
    ├── RuntimeException         (언체크 - NullPointerException, IllegalStateException 등)
    └── IOException, SQLException 등  ← RuntimeException을 거치지 않고 Exception을 직접 상속 (체크 예외)
```

즉, "```Exception```을 상속한 체크드 익셉션"이란, ```RuntimeException```을 거치지 않고 ```Exception```을 바로 상속한 예외 클래스를 말한다(```IOException```, ```SQLException``` 등).
```RuntimeException```도 ```Exception```의 자식이지만 JVM/컴파일러가 별도로 "언체크"로 취급한다.

자바에서는 체크 예외를 메서드 시그니처에 ```throws```로 선언하거나 ```try-catch```로 처리하지 않으면 컴파일 에러가 나는 반면, 언체크 예외는 그런 강제가 없다.
코틀린에는 이 컴파일러 강제 규칙 자체가 없지만, JVM 위에서 동작하는 이상 클래스 상속 계층은 그대로 존재하고, 스프링의 ```@Transactional``` 롤백 규칙은 바로 이 상속 관계를 기준으로 판단하기 때문에 코틀린 코드에도 동일하게 적용된다.


| 예외 종류                               | 기본 동작        |
|-------------------------------------|--------------|
| RuntimeException (unchecked) / Error | 롤백됨          |
| Exception을 상속한 checked exception    | 롤백 안 됨 (커밋됨) |

```kotlin
@Transactional
fun placeOrder(orderId: Long) {
    repository.save(orderId)
    throw IllegalStateException("재고 부족") // unchecked -> 롤백됨
}

@Transactional
fun placeOrderChecked(orderId: Long) {
    repository.save(orderId)
    throw java.io.IOException("외부 API 실패") // checked -> 기본값으로는 롤백 안 되고 커밋됨!
}
```

체크 예외인데도 롤백시키고 싶다면 ```rollbackFor```로 명시해야 한다.
```kotlin
@Transactional(rollbackFor = [java.io.IOException::class])
fun placeOrderChecked(orderId: Long) {
    repository.save(orderId)
    throw java.io.IOException("외부 API 실패") // 이제는 롤백됨
}
```

반대로 unchecked exception인데 롤백시키고 싶지 않다면 ```noRollbackFor```를 쓴다.

이 기본 규칙은 "체크 예외는 복구 가능한 비즈니스 예외, 언체크 예외는 예상 못 한 심각한 오류"라는 자바의 관례를 따른 것인데, 실무에서는 체크 예외를 거의 안 쓰고 ```RuntimeException``` 계열만 쓰는 경우가 많아서 이 규칙 자체를 모르고 넘어가기 쉽다.
외부 API 연동처럼 체크 예외(```IOException``` 등)를 다루는 코드에 ```@Transactional```을 걸 때는 특히 주의해야 한다.


### Self-invocation 문제 - 왜 자기 자신을 호출하면 트랜잭션이 안 먹히나
```kotlin
@Service
class OrderService(private val repository: OrderRepository) {
    
    fun placeOrder(orderId: Long) {
        saveWithTransaction(orderId) // 문제 발생!
    }
    
    @Transactional
    fun saveWithTransaction(orderId: Long) {
        repository.save(orderId)
    }
}
```

```context.getBean(OrderService::class.java)```로 꺼낸 건 프록시지만, ```placeOrder()``` 안에서 ```saveWithTransaction()```을 호출하는 건 프록시를 거치지 않고 ```this.saveWithTransaction()```을 직접 호출하는 것이다.
자기 자신 내부에서 자기 메서드를 호출하는 거라 프록시의 "가로채기"가 일어날 기회 자체가 없다.
결과적으로 ```@Transactional```이 무시된다.

```text
클라이언트 -> [프록시] -> placeOrder() 호출 ✅(프록시를 거침)
                            ↓
                    내부에서 this.saveWithTransaction() 호출 ❌(프록시 안 거침, 그냥 메서드 호출)
```

해결 방법은 트랜잭션이 필요한 메서드를 다른 빈으로 분리해서, 외부에서 프록시를 거쳐 호출하도록 만드는 것이다.
```kotlin
@Service
class OrderService(
    private val transactionalOrderService: TransactionalOrderService, // 별도 빈 주입
) {
    fun placeOrder(orderId: Long) {
        transactionalOrderService.saveWithTransaction(orderId) // 프록시를 거쳐 호출됨
    }
}

@Service
class TransactionalOrderService(private val repository: OrderRepository) {
    @Transactional
    fun saveWithTransaction(orderId: Long) {
        repository.save(orderId)
    }
}
```

### 전파(Propagation) 레벨 - "이미 트랜잭션이 있을 때 어떻게 할까"
트랜잭션 메서드가 또 다른 트랜잭션 메서드를 호출할 때, 기존 트랜잭션을 어떻게 다룰지 정하는 옵션.
실무/면접에서 중요한건 아래 2개이다.

| 전파 레벨          | 동작                                       |
|----------------|------------------------------------------|
| REQUIRED (기본값) | 기존 트랜잭션이 있으면 합류, 없으면 새로 시작               |
| REQUIRES_NEW   | 기존 트랜잭션과 무관하게 항상 새 트랜잭션을 시작 (기존 건 잠시 보류) |

```kotlin
@Transactional // REQUIRED (기본값)
fun placeOrder(orderId: Long) {
    repository.save(orderId)
    logService.writeLog(orderId) // 같은 트랜잭션에 합류
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
fun writeLog(orderId: Long) {
    logRepository.save(orderId) // 독립된 트랜잭션 -> placeOrder가 롤백돼도 로그는 남음
}
```

```REQUIRES_NEW```가 자주 쓰이는 이유는, "주문 처리는 실패해도 로그는 반드시 남겨야 한다" 같은 요구사항 때문이다. 
같은 트랜잭션이면 ```placeOrder```가 롤백될 때 로그도 같이 롤백된다.

### 격리 수준(Isolation Level) - "동시에 여러 트랜잭션이 돌 때 서로 얼마나 간섭할까"
| 격리 수준          | 허용되는 현상                                       |
|----------------|-----------------------------------------------|
| READ_UNCOMMITED | Dirty Read까지 허용 (커밋 안 된 데이터도 읽힘)              |
| READ_COMMITED  | Dirty Read 방지, Non-Repeatable Read는 발생 가능     |
| REPEATABLE_READ | Non-Repeatable Read까지 방지, Phantom Read는 발생 가능 |
| SERIALIZABLE   | 전부 방지 (가장 엄격, 성능 저하 큼)                        |

- Dirty Read: 다른 트랜잭션이 아직 커밋 안 한 데이터를 읽어버림
- Non-Repeatable Read: 같은 트랜잭션 안에서 같은 행을 두 번 읽었는데 값이 다름 (중간에 다른 트랜잭션이 UPDATE 커밋)
- Phantom Read: 같은 조건으로 두 번 조회했는데 행의 개수가 다름 (중간에 다른 트랜잭션이 INSERT 커밋)

스프링/DB 기본값은 보통 READ_COMMITTED(MySQL은 ```REPEATABLE_READ```가 기본)이고, 특별한 이유 없으면 기본값을 그대로 쓰는 게 일반적이다.

### 예상 면접 질문
- ```@Transactional```은 내부적으로 어떻게 동작하나요? (AOP 프록시가 트랜잭션 시작/커밋/롤백을 감싸는 형태)
- Self-invocation 문제가 무엇이고 왜 발생하나요? (같은 클래스 내부에서 ```this```로 호출하면 프록시를 거치지 않기 때문)
- ```REQUIRED```와 ```REQUIRES_NEW```의 차이는 무엇인가요?
- Dirty Read, Non-Repeatable Read, Phantom Read의 차이를 설명해보세요.
- ```@Transactional```은 모든 예외에 대해 롤백되나요? (아니요 - 기본적으로 unchecked exception/```Error```만 롤백되고, checked exception은 커밋됨. ```rollbackFor```로 커스터마이징 가능)






