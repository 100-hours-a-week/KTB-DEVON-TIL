# [WebFlux 시리즈 5편] Spring WebFlux, 왜 필요하고 언제 써야 하는가

> WebFlux는 무조건 더 빠른 게 아니다. 제대로 써야 빠르다.

## 0. 이 글을 쓰게 된 계기

1편부터 4편까지 Reactor의 핵심 개념들을 쌓아왔다.

- 리액티브 스트림즈 표준과 Reactor (1편)
- 스케줄러와 스레드 제어 (2편)
- Context와 데이터 전파 (3편)
- 에러 처리 전략 (4편)

이제 이 개념들이 실제 웹 프레임워크에서 어떻게 동작하는지를 볼 차례다.

그런데 WebFlux를 왜 쓰는가? Spring MVC가 있는데?

"비동기·논블로킹이라서 더 빠르다"는 말은 들었다. 하지만 그게 구체적으로 무슨 의미인지, 어떤 상황에서 의미가 있는지를 제대로 이해하지 못한 채 공부를 시작했다.

이 글은 그 질문에 대한 답이다.

이 글은 **WebFlux 학습 시리즈 5편**이다.
- 1편: 리액티브 스트림즈와 Reactor, 표준과 구현체를 함께 이해하기
- 2편: Reactor 스케줄러, 어떤 스레드에서 코드가 실행되는지 이해하기
- 3편: Reactor Context, 스레드가 바뀌어도 데이터를 전달하는 방법
- 4편: Reactor 에러 처리, onError 신호를 다루는 방법
- 6편: WebFlux 요청/응답 처리 흐름 (예정)

---

## 1. Spring MVC의 스레드 모델부터 이해해야 한다

WebFlux가 왜 등장했는지를 알려면, 먼저 Spring MVC가 어떻게 동작하는지를 봐야 한다.

MVC는 **요청당 스레드 하나(Thread-per-request)** 모델이다.

```
[Spring MVC 요청 처리]

요청 1 → 스레드 A 할당 → 컨트롤러 → DB 쿼리 (대기 중...) → 응답 → 스레드 A 반환
요청 2 → 스레드 B 할당 → 컨트롤러 → DB 쿼리 (대기 중...) → 응답 → 스레드 B 반환
요청 3 → 스레드 C 할당 → ...
...
요청 201 → 스레드 없음 → 대기 (Tomcat 기본 최대 스레드: 200개)
```

스레드 풀이 고갈되면 새로운 요청은 스레드가 생길 때까지 기다려야 한다.

단순히 스레드 수가 모자란 게 아니다. 더 근본적인 문제가 있다.

**스레드가 DB 응답을 기다리는 동안 아무것도 안 한다.** 스레드가 I/O 대기 상태에 묶여 있는 동안 다른 요청을 처리할 수 있는 기회를 버리고 있다.

요청당 스레드가 필요하고, 스레드는 대부분의 시간을 대기하며 보내고, 대기 중에도 메모리는 점유된다. 동시 접속이 늘수록 이 낭비가 선형으로 증가한다.

---

## 2. WebFlux의 이벤트 루프 모델

WebFlux는 기반 서버로 **Netty**를 사용한다. Netty는 이벤트 루프(Event Loop) 기반이다.

```
[WebFlux + Netty 요청 처리]

이벤트 루프 스레드 (소수, CPU 코어 수 정도)

→ 요청 1 수신 → I/O 이벤트 등록 후 다음 작업으로
→ 요청 2 수신 → I/O 이벤트 등록 후 다음 작업으로
→ 요청 3 수신 → I/O 이벤트 등록 후 다음 작업으로
→ 요청 1의 DB 응답 도착 → 이어서 처리
→ 요청 3의 DB 응답 도착 → 이어서 처리
```

스레드가 I/O 완료를 기다리는 대신, 완료 이벤트가 오면 그때 이어서 처리한다. 스레드는 대기하지 않고 계속 다른 이벤트를 처리한다.

결과적으로 **소수의 스레드로 많은 동시 요청을 처리**할 수 있다.

이때 각 처리 단계를 연결하는 구조가 바로 **Reactor의 Mono/Flux 체인**이다. 1편에서 공부한 `subscribe()` → `onNext` → `onComplete` 흐름이 여기서 실제로 동작한다.

---

## 3. MVC vs WebFlux 비교

| 항목 | Spring MVC | Spring WebFlux |
|------|-----------|---------------|
| 동작 방식 | 동기·블로킹 | 비동기·논블로킹 |
| 스레드 모델 | 요청당 스레드 | 이벤트 루프, 소수 스레드 |
| 기반 서버 | Tomcat, Jetty | Netty, Undertow |
| 반환 타입 | `User`, `List<User>` | `Mono<User>`, `Flux<User>` |
| 코드 디버깅 | 한 스레드에서 추적 용이 | 체인 + 스레드 전환으로 추적 어려움 |
| 학습 곡선 | 낮음 | 높음 |
| 리소스 효율 | 동시성 증가 시 스레드 급증 | 적은 스레드로 높은 동시성 처리 |

중요한 건 둘이 완전히 분리된 세계가 아니라는 점이다.

- WebFlux에서도 `@RestController`, `@GetMapping` 같은 어노테이션 기반 컨트롤러를 그대로 쓸 수 있다.
- MVC에서도 `WebClient` 같은 리액티브 HTTP 클라이언트를 사용할 수 있다.

WebFlux는 "완전히 다른 것"이 아니라 **다른 실행 모델 위에서 동작하는 웹 레이어**다.

---

## 4. WebFlux가 빛나는 상황

WebFlux가 MVC보다 유리한 상황은 명확하다.

**"데이터는 작지만 발생 빈도가 매우 높은" 서비스**

### 실시간 채팅

수천 명의 사용자가 동시에 메시지를 주고받는다. 각 메시지는 작지만 연결 수가 많다.
MVC 모델로는 연결 수만큼 스레드가 필요하다. WebFlux는 소수 스레드로 모든 연결을 처리한다.

### 실시간 스트리밍

주식 시세, 코인 가격처럼 데이터가 지속적으로 흘러온다. SSE(Server-Sent Events)나 WebSocket으로 클라이언트에게 푸시한다.

### IoT 센서 데이터

수천 개의 센서가 초당 여러 번 데이터를 전송한다. 각 데이터는 작지만 동시 처리 수가 엄청나다.

### API 게이트웨이

여러 백엔드 서비스를 호출하고 조합해서 응답한다. 외부 API 호출 대기 시간이 길고, 모든 클라이언트 요청이 집중된다. 논블로킹으로 대기 시간을 효율적으로 활용할 수 있다.

---

## 5. WebFlux의 엔드포인트 구성 방식 두 가지

WebFlux는 엔드포인트를 정의하는 방식이 두 가지다.

### 어노테이션 기반 컨트롤러

MVC와 거의 동일하다. 반환 타입만 `Mono`/`Flux`로 바뀐다.

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @GetMapping
    public Flux<User> getAllUsers() {
        return userService.findAll();
    }
}
```

MVC에서 넘어올 때 가장 친숙한 방식이다. 진입 장벽이 낮다.

### 라우터 함수 (Router Functions)

함수형 스타일로 라우팅과 핸들러를 분리한다.

```java
@Configuration
public class UserRouter {

    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return RouterFunctions.route()
            .GET("/users/{id}", handler::getUser)
            .GET("/users", handler::getAllUsers)
            .POST("/users", handler::createUser)
            .build();
    }
}

@Component
public class UserHandler {

    public Mono<ServerResponse> getUser(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        return userService.findById(id)
            .flatMap(user -> ServerResponse.ok().bodyValue(user))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
}
```

라우팅 규칙과 비즈니스 로직이 분리된다. 람다 기반이라 선언적으로 읽힌다.

**두 방식 중 선택 기준:**
- 기존 MVC 팀이 빠르게 적응해야 한다 → 어노테이션 기반
- 함수형 스타일로 깔끔하게 구성하고 싶다 → 라우터 함수

---

## 6. WebFlux의 성능이 무조건 좋은 건 아니다

이 부분을 반드시 짚고 넘어가야 한다.

> WebFlux가 MVC보다 항상 빠른 게 아니다.

### 전체 스택이 논블로킹이어야 효과가 있다

WebFlux의 이점은 I/O 대기 동안 스레드를 해방하는 것에서 온다.
그런데 체인 중 **한 곳이라도 블로킹 코드**가 있으면 이점이 사라진다.

```
[블로킹 코드가 섞인 WebFlux]

이벤트 루프 스레드 → JDBC 쿼리 호출 → 스레드 블로킹 (대기 중...)
                                              ↑
                              이벤트 루프가 멈춰서 다른 요청도 처리 못 함
```

DB, 캐시, 외부 API 등 **모든 연동 지점이 논블로킹을 지원해야** 효과가 있다.

- DB: R2DBC (JDBC는 블로킹)
- Redis: Reactive Redis
- MongoDB: Reactive MongoDB
- HTTP 클라이언트: WebClient (RestTemplate은 블로킹)

만약 중간에 JDBC를 써야 한다면, 2편에서 설명한 것처럼 반드시 `boundedElastic` 스케줄러로 격리해야 한다.

### 블로킹 코드 혼용은 더 나쁜 결과를 낳는다

MVC에서는 스레드가 블로킹되면 해당 스레드만 멈춘다. 200개 스레드 중 1개가 멈춰도 나머지 199개가 동작한다.

WebFlux에서 이벤트 루프 스레드가 블로킹되면, 그 스레드가 담당하는 **모든 요청이 함께 멈춘다.** 스레드 수 자체가 적기 때문에 피해가 훨씬 크다.

블로킹 코드를 WebFlux에서 잘못 쓰면 MVC보다 성능이 **훨씬 나쁠 수 있다.**

---

## 7. 도입할 때 현실적으로 고려해야 할 것들

### 마이그레이션 비용

기존 MVC 코드를 WebFlux로 바꾸는 건 단순한 프레임워크 교체가 아니다. **코드 패러다임 전환**이다.

```java
// MVC 방식
public User getUser(Long id) {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new NotFoundException());
    return enrichUser(user);
}

// WebFlux 방식
public Mono<User> getUser(Long id) {
    return userRepository.findById(id)    // Mono<User>
        .switchIfEmpty(Mono.error(new NotFoundException()))
        .map(user -> enrichUser(user));
}
```

로직 자체는 같지만, 표현 방식이 완전히 다르다. 팀 전체가 이 패러다임에 익숙해지는 시간이 필요하다.

### 디버깅 난이도

동기 코드에서 스택 트레이스는 직관적이다.
```
UserController.getUser
  → UserService.findById
    → UserRepository.findById
      → SQL 실행
```

WebFlux에서는 스레드가 여러 번 바뀌고 체인이 복잡하면 스택 트레이스가 길고 파악하기 어렵다.

`Mono.checkpoint()`, `.log()` 같은 연산자를 활용해 체인 중간에 관찰 포인트를 심는 습관이 필요하다.

```java
userRepository.findById(id)
    .checkpoint("findById 완료")   // 이 시점의 스택 트레이스 캡처
    .map(user -> enrichUser(user))
    .log()                          // 모든 신호 로깅
```

---

## 8. 언제 MVC를 쓰고 언제 WebFlux를 써야 하나

이 질문이 핵심이다.

**WebFlux를 고려할 때:**
- 동시 연결 수가 매우 많고 I/O 대기가 주된 병목인 서비스
- 실시간 스트리밍, 채팅, IoT, API 게이트웨이
- 전체 스택을 논블로킹으로 구성할 수 있는 환경
- 팀이 리액티브 프로그래밍을 학습할 의지와 시간이 있을 때

**MVC를 유지하는 게 나을 때:**
- 일반적인 트래픽 수준의 CRUD 위주 서비스
- 연동 시스템 중 블로킹 API가 많아 논블로킹 스택 구성이 어려울 때
- 개발 생산성과 빠른 디버깅이 더 중요할 때
- 팀이 리액티브 학습에 투자할 여유가 없을 때

한 줄로 요약하면:

> "요청이 많고 대부분의 시간을 I/O 대기에 쓰는 서비스라면 WebFlux, 그 외에는 MVC가 현실적인 선택이다."

---

## 마치며

지금까지 Reactor의 핵심 개념들을 쌓고, 이 글에서 그것들이 왜 필요한지를 프레임워크 레벨에서 연결했다.

- **리액티브 스트림즈**: WebFlux의 데이터 흐름 표준
- **Reactor(Mono/Flux)**: 그 표준의 구현체
- **스케줄러**: 어떤 스레드에서 어떤 작업을 처리할지
- **Context**: 스레드가 바뀌어도 요청 데이터를 전달하는 방법
- **에러 처리**: 비동기 체인에서 장애를 선언적으로 처리하는 방법

이 모든 개념이 WebFlux에서 **하나의 요청을 처리하는 동안** 함께 동작한다.

WebFlux는 "쓰면 무조건 좋은 것"이 아니라, **상황을 이해하고 선택해야 하는 도구**다. 그 선택을 잘 하려면 지금까지 쌓아온 개념들이 필요하다.

다음 글에서는 **[WebFlux 시리즈 6편] WebFlux 요청과 응답 처리 흐름**을 다룬다. `ServerHttpRequest`, `ServerHttpResponse`, `ServerWebExchange`가 실제 HTTP 처리에서 어떻게 동작하는지를 살펴본다.
