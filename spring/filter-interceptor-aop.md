## Filter vs Interceptor vs AOP
셋 다 "본래 로직 앞 뒤에 뭔가를 끼워 넣는다"는 점은 같다.
차이는 어느 계층에서, 무엇을 대상으로 끼어드는가다.

### 전체 흐름
```text
클라이언트 요청
   ↓
[Filter]                  ← 서블릿 컨테이너 레벨 (스프링 알기 전)
   ↓
DispatcherServlet
   ↓
[Interceptor.preHandle]   ← 스프링 MVC 레벨 (컨트롤러 실행 직전)
   ↓
Controller
   ↓
[AOP 프록시]                ← 메서드 레벨 (Service 메서드 호출을 감쌈)
   ↓
Service
   ↓
[Interceptor.postHandle / afterCompletion]
   ↓
Filter (응답 나갈 때)
   ↓
클라이언트 응답
```

### 1) Filter - 서블릿 스펙 자체의 기능
스프링이 아니라 Servlet 컨테이너(톰캣 등)가 제공하는 기능.
```DispatcherServlet```보다 더 앞단에서 동작한다.

```kotlin
class LoggingFilter : Filter {
    override fun doFilter(
        request: ServletRequest,
        response: ServletResponse,
        chain: FilterChain
    ) {
        println("요청 시작: ${(request as HttpServletRequest).requestURI}") // 1. 여기서 실행 (요청이 들어올 때)
        chain.doFilter(request, response) // 다음 필터 or 서블릿으로 넘김
        // ↑ 이 한 줄 안에서 DispatcherServlet, Interceptor, Controller, Service 등
        // 나머지 전체 파이프라인이 다 실행되고 끝난 뒤에야 다음 줄로 넘어감
        println("응답 완료") // 2. 여기서 실행 (응답이 나갈 때)
    }
}
```

- 다루는 대상: ```ServletRequest``` / ```ServletResponse``` (HTTP 요청/응답 자체)
- 컨트롤러가 뭔지, 어떤 메서드가 호출될지 전혀 모름
- 적합한 용도: 인코딩 설정, CORS, XSS 필터링, 요청 로깅처럼 "어떤 컨트롤러인지와 무관하게" 모든 요청에 걸어야 하는 것
- 스프링 예외 처리(```@ControllerAdvice```)보다 바깥에 있어서, Filter에서 발생한 예외는 ```@ControllerAdvice```가 못 잡는다.

### 2) Interceptor - 스프링 MVC 자체의 기능
```DispatcherServlet```이 컨트롤러를 호출하기 직전/직후에 끼어드는, 스프링 MVC 안에 있는 기능.

```kotlin
class AuthInterceptor : HandlerInterceptor {
    override fun preHandle(
        request: HttpServletRequest, 
        response: HttpServletResponse,
        handler: Any): Boolean {
        // handler를 통해 "어떤 컨트롤러의 어떤  메서드가 호출될지" 알 수 있음
        if (handler is HandlerMethod) {
            println("호출될 메서드: ${handler.method.name}")
        }
        return true // false 반환 시 컨트롤러 실행 자체를 막음
    }
}
```

- 다루는 대상: ```HandlerMethod``` (호출될 컨트롤러/메서드 정보까지 접근 가능)
- Filter보다 더 세분화 된 시점 제공: ```preHandle```(컨트롤러 실행 전) / ```postHandle```(컨트롤러 실행 후, 뷰 렌더링 전) / ```afterCompletion```(뷰 렌더링까지 끝난 후)
- 적합한 용도: 로그인 여부 체크, 컨트롤러 실행 여부를 조건부로 막고 싶을 때(```preHandle```이 ```false```를 반환하면 컨트롤러 자체가 실행 안 됨)
- 스프링 컨텍스트 안에 있어서 ```@ControllerAdvice```로 예외 처리 가능

### Filter도 컨트롤러 실행을 막을 수 있다
"Interceptor가 컨트롤러 실행을 막는다"고 했지만, 사실 Filter도 똑같이 막을 수 있다. 
```chain.doFilter()```를 호출하지 않고 그냥 응답을 써버리면, 요청이 아예 ```DispatcherServlet```까지도 못 간다.

```kotlin
class IpBlockFilter : Filter {
    override fun doFilter(
        request: ServletRequest,
        response: ServletResponse,
        chain: FilterChain
    ) {
        if (isBlacklisted(request as HttpServletRequest)) {
            (response as HttpServletResponse).status = 403
            return // chain.doFilter()를 안 부르니 컨트롤러는 커녕 DispatcherServlet도 못 감
        }
        chain.doFilter(request, response)
    }
}
```

그럼 둘 다 막을 수 있는데 왜 나눠 쓰는가? - Filter는 "어떤 컨트롤러/메서드가 호출될지"를 전혀 모르지만, Interceptor는 ```HandlerMethod```를 통해 정확히 어떤 메서드가 호출될지 알 수 있기 때문이다.
그래서 판단 기준에 "어떤 엔드포인트인지"가 필요한 로직은 Interceptor에서 처리하는 게 자연스럽다.
대표적인 예가 커스텀 어노테이션 기반 권한 체크다.

```kotlin
@Target(AnnotationTarget.FUNCTION)
annotation class LoginRequired

class OrderController {
    @LoginRequired // 이 메서드는 로그인이 필요함을 "선언"해둠
    fun getMyOrders(): List<Order> { ... }
    
    fun getPublicNotice(): Notice { ... } // 이 메서드는 로그인 불필요
}

class AuthInterceptor : HandlerInterceptor {
    override fun preHandle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any): Boolean {
            if (handler is HandlerMethod && handler.hasMethodAnnotation(LoginRequired::class.java)) {
                // 호출될 메서드에 @LoginRequired가 붙어있는지 확인 가능 -> Filter는 이걸 모름
                if (!isLoggedIn(request)) {
                    response.status = 401
                    return false // 이 메서드만 막고, @LoginRequired 없는 메서드는 통과시킴
                }
            }
        return true
    }
}
```

정리하면:
- **Filter**: "요청 전체에 공통으로 적용되는, 어떤 엔드포인트인지와 무관한" 차단 로직 (IP 차단, Rate Limit, 전역 인증 토큰 형식 검증)
- **Interceptor**: "이 특정 컨트롤러/메서드에만 적용되는" 세밀한 차단 로직 (특정 엔드포인트만 로그인 필요, 특정 API만 관리자 권한 필요 등)

즉 둘 다 "막을 수 있다"는 능력은 같지만, 막을지 말지를 결정하는 데 필요한 정보의 세밀함이 다르기 때문에 역할을 나눠 쓴다.



### 3) AOP - 메서드 레벨, HTTP와 무관
Filter/Interceptor는 HTTP 요청이 있어야만 동작하는데, AOP는 HTTP 요청과 완전히 무관하게 스프링 빈의 어떤 메서드든 대상으로 삼을 수 있다.
Controller뿐 아니라 Service, Repository 계층 메서드에도 걸 수 있다.

- 다루는 대상: ```JoinPoint```(메서드 실행 시점) - 메서드 이름, 파라미터, 반환값까지 세밀하게 접근 가능
- 적합한 용도: 트랜잭션(```@Transactional```도 사실 AOP로 구현됨), 특정 서비스 메서드의 실행 시간 측정, 도메인 로직에 걸친 공통 처리

자세한 프록시 동작 원리는 AOP 노트 참고.

### 한눈에 비교
| | Filter | Interceptor | AOP |
|---|---|---|---|
| 관리 주체 | 서블릿 컨테이너 | 스프링 MVC | 스프링 컨테이너(프록시) |
| 동작 위치 | DispatcherServlet 이전/이후 | DispatcherServlet ~ Controller 사이 | 메서드 호출 지점 (계층 무관) |
| 접근 가능 정보 | ServletRequest/Response | HandlerMethod (컨트롤러 정보) | JoinPoint (메서드/파라미터/반환값) |
| 대표 용도 | 인코딩, CORS, XSS 방지 | 인증/인가, 컨트롤러 실행 여부 제어 | 트랜잭션, 로깅, 도메인 공통 로직 |

### 예상 면접 질문
- Filter와 Interceptor의 차이는 무엇인가요? (관리 주체 다름 - 서블릿 컨테이너 vs 스프링 MVC, 그래서 접근 가능한 정보와 예외 처리 방식도 다름)
- Interceptor와 AOP의 차이는 무엇인가요? (Interceptor는 HTTP 요청에 국한되고 컨트롤러 앞뒤로만 동작하지만, AOP는 계층과 무관하게 모든 스프링 빈의 메서드에 적용 가능)
- 인증/인가 로직은 Filter, Interceptor, AOP 중 어디에 두는게 적절한가요? (일반적으로 Interceptor 또는 Filter - 컨트롤러 실행 자체를 막아야 하므로, 다만 세부 인가 로직은 AOP로 메서드 단위 권한 체크를 하기도 함)
- Filter도 컨트롤러 실행을 막을 수 있는데, 왜 Interceptor를 따로 쓰나요? (Filter는 어떤 컨트롤러/메서드가 호출될지 모르는 반면, Interceptor는 ```HandlerMethod```로 호출될 메서드 정보를 알 수 있어서 엔드포인트별 세밀한 차단이 가능함)



















