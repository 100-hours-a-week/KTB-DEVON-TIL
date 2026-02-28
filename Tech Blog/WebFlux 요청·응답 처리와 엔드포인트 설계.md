# [WebFlux 시리즈 6편] WebFlux 요청·응답 처리와 엔드포인트 설계

> HTTP를 다루는 두 가지 방식, 그리고 그 아래에 있는 진짜 인터페이스들

---

## 0. 이 글을 쓰게 된 계기

WebFlux를 처음 접했을 때, 가장 낯선 것은 어노테이션이 아니었다. `@GetMapping`, `@PostMapping`은 MVC에서 이미 쓰던 방식이라 금방 익숙해졌다.

진짜 혼란은 라우터 함수 스타일을 마주했을 때였다.

```java
RouterFunctions.route(GET("/hello"), request ->
    ServerResponse.ok().bodyValue("Hello WebFlux"))
```

이게 뭐지? 컨트롤러가 없어도 되는 건가? `ServerRequest`는 `HttpServletRequest`랑 뭐가 다른 건가?

이 글은 WebFlux에서 HTTP 요청과 응답을 다루는 두 가지 방식과, 그 아래에 깔린 핵심 인터페이스들을 정리한다.

---

## 1. 두 가지 엔드포인트 구성 방식

WebFlux는 API 엔드포인트를 두 가지 방식으로 구성할 수 있다.

**어노테이션 기반 컨트롤러**

```java
@RestController
public class HelloController {

    @GetMapping("/hello/{name}")
    public Mono<String> hello(@PathVariable String name) {
        return Mono.just("Hello, " + name);
    }
}
```

**라우터 함수 방식**

```java
@Configuration
public class HelloRouter {

    @Bean
    public RouterFunction<ServerResponse> route(HelloHandler handler) {
        return RouterFunctions
            .route(GET("/hello/{name}"), handler::hello);
    }
}

@Component
public class HelloHandler {

    public Mono<ServerResponse> hello(ServerRequest request) {
        String name = request.pathVariable("name");
        return ServerResponse.ok().bodyValue("Hello, " + name);
    }
}
```

겉으로 보면 단순히 스타일의 차이처럼 보인다. 하지만 내부 구조는 꽤 다르다.

어노테이션 방식은 MVC와 유사한 방식으로 동작하며, Spring이 파라미터 바인딩을 대신 처리해 준다. 라우터 함수 방식은 요청 처리 로직을 직접 함수로 표현하며, `ServerRequest`를 통해 HTTP 레이어에 더 가깝게 접근한다.

---

## 2. HTTP 요청·응답을 다루는 핵심 인터페이스

WebFlux에는 HTTP 요청과 응답을 직접 다루는 세 가지 인터페이스가 있다.

```
ServerWebExchange
├── ServerHttpRequest  (요청 정보 조회)
└── ServerHttpResponse (응답 구성)
```

이 인터페이스들은 라우터 함수 방식에서 자연스럽게 마주하게 된다. 어노테이션 방식에서도 `WebFilter` 같은 곳에서 직접 접근하는 경우가 있다.

### 2.1 ServerHttpRequest

클라이언트가 보낸 HTTP 요청의 모든 정보에 접근하는 인터페이스다.

| 기능 | 메서드 | 설명 |
|------|--------|------|
| 헤더 조회 | `getHeaders().getFirst("key")` | 특정 헤더 값 조회. 헤더는 복수 값 가능이므로 `getFirst`로 첫 값을 꺼낸다 |
| 쿼리 파라미터 | `getQueryParams().getFirst("key")` | URL `?key=value`에서 값 조회 |
| 쿠키 조회 | `getCookies().getFirst("key")` | 요청에 포함된 쿠키 값 조회 |
| 요청 바디 | `getBody()` | `Flux<DataBuffer>`로 반환. 비동기 데이터 스트림이다 |

`ServerHttpRequest`는 **"들어온 요청을 읽어들이는 창구"** 역할을 한다.

### 2.2 ServerHttpResponse

서버가 클라이언트에게 보낼 HTTP 응답을 구성하는 인터페이스다.

| 기능 | 메서드 | 설명 |
|------|--------|------|
| 헤더 설정 | `getHeaders().add("key", "value")` | 응답 헤더에 값 추가 |
| 쿠키 추가 | `addCookie(ResponseCookie.from(...).build())` | 응답에 쿠키 추가 |
| 상태 코드 | `setStatusCode(HttpStatus.OK)` | HTTP 상태 코드 설정 |
| 응답 바디 | `writeWith(Publisher<DataBuffer>)` | `Flux<DataBuffer>` 같은 Publisher로 비동기 스트림 전송 |

`ServerHttpResponse`는 **"무엇을 어떻게 돌려줄지 결정하는 출력 창구"** 역할을 한다.

### 2.3 ServerWebExchange

단일 HTTP 요청-응답 사이클 전체를 캡슐화하는 컨테이너 인터페이스다.

```java
ServerWebExchange exchange; // 요청과 응답 모두 포함

ServerHttpRequest request = exchange.getRequest();
ServerHttpResponse response = exchange.getResponse();

// 요청 헤더 조회 → 응답 구성
String userAgent = request.getHeaders().getFirst("User-Agent");
response.getHeaders().add("X-Processed-By", "WebFlux");
```

`ServerWebExchange`가 중요한 이유는, WebFlux의 필터 체인, 핸들러 체인 등 **모든 처리 단계에서 공유되는 컨텍스트 객체**이기 때문이다.

MVC의 `DispatcherServlet`이 `HttpServletRequest`와 `HttpServletResponse`를 들고 다니는 것처럼, WebFlux에서는 `ServerWebExchange` 하나가 요청-응답 사이클 전체를 담는다.

추가로 **속성(Attribute) 공유** 기능도 제공한다.

```java
// 필터에서 데이터 저장
exchange.getAttributes().put("userId", "devon");

// 핸들러에서 꺼내기
String userId = (String) exchange.getAttributes().get("userId");
```

---

## 3. 어노테이션 기반 컨트롤러: 파라미터 처리 방법

어노테이션 방식은 MVC와 거의 같다. 차이는 반환 타입이 `Mono<T>` 또는 `Flux<T>`라는 점이다.

### 3.1 경로 변수

```java
@GetMapping("/hello/{name}")
public Mono<String> hello(@PathVariable String name) {
    return Mono.just("Hello, " + name);
}
```

### 3.2 쿼리 파라미터

```java
@GetMapping("/search")
public Flux<Item> search(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size
) {
    return itemService.findAll(page, size);
}
```

`defaultValue`로 기본값을 설정하고, `required = false`로 필수 여부를 조정할 수 있다.

### 3.3 요청 헤더

```java
@GetMapping("/trace")
public Mono<String> trace(
    @RequestHeader("X-Request-Id") String requestId
) {
    return Mono.just("Processing request: " + requestId);
}
```

### 3.4 요청 본문

```java
@PostMapping("/users")
public Mono<User> createUser(@RequestBody Mono<User> userMono) {
    return userMono.flatMap(user -> userService.save(user));
}
```

WebFlux에서는 `@RequestBody`로 받는 타입을 `Mono<User>`로 선언하는 게 일반적이다. 이렇게 하면 요청 바디 역직렬화 자체도 리액티브 파이프라인 안에 포함된다.

---

## 4. 라우터 함수 방식: 파라미터 처리 방법

라우터 함수에서는 `ServerRequest` 객체를 통해 모든 파라미터에 접근한다.

### 4.1 경로 변수

```java
public Mono<ServerResponse> hello(ServerRequest request) {
    String name = request.pathVariable("name");
    return ServerResponse.ok().bodyValue("Hello, " + name);
}
```

### 4.2 쿼리 파라미터

```java
public Mono<ServerResponse> search(ServerRequest request) {
    String keyword = request.queryParam("q").orElse("*");
    int page = Integer.parseInt(request.queryParam("page").orElse("0"));
    return ServerResponse.ok().bodyValue(searchService.search(keyword, page));
}
```

라우터 함수에서는 쿼리 파라미터가 `Optional<String>`으로 반환된다. `.orElse("default")`로 기본값을 처리하는 것이 자연스럽다.

### 4.3 요청 헤더

```java
public Mono<ServerResponse> trace(ServerRequest request) {
    String requestId = request.headers().firstHeader("X-Request-Id");
    return ServerResponse.ok().bodyValue("TraceId: " + requestId);
}
```

### 4.4 요청 본문

```java
public Mono<ServerResponse> createUser(ServerRequest request) {
    return request.bodyToMono(User.class)
        .flatMap(user -> userService.save(user))
        .flatMap(saved -> ServerResponse.created(URI.create("/users/" + saved.getId()))
            .bodyValue(saved));
}
```

`request.bodyToMono(User.class)`로 요청 바디를 `Mono<User>`로 변환하고, `flatMap`으로 이후 처리 체인을 이어간다. 라우터 함수에서는 **비동기 체인 구성이 코드 구조 자체에 명확히 드러난다.**

---

## 5. 두 방식 비교

| 항목 | 어노테이션 방식 | 라우터 함수 방식 |
|------|----------------|-----------------|
| 스타일 | Spring MVC와 유사 | 함수형 DSL |
| 라우팅 정의 | 어노테이션으로 분산 | 코드로 중앙 집중 |
| 파라미터 접근 | 어노테이션 바인딩 (`@PathVariable` 등) | `ServerRequest` API 직접 호출 |
| HTTP 레이어 제어 | 프레임워크가 추상화 | 더 직접적인 접근 가능 |
| MVC 경험 활용 | 바로 적용 가능 | 추가 학습 필요 |
| 테스트 용이성 | 상대적으로 익숙한 방식 | 함수 단위 테스트가 더 자연스러움 |

**어노테이션 방식이 유리한 경우**
- MVC 경험이 있고, 전환 비용을 낮추고 싶을 때
- 팀원 대부분이 기존 스타일에 익숙할 때

**라우터 함수 방식이 유리한 경우**
- 라우팅 규칙을 코드로 명시적으로 관리하고 싶을 때
- HTTP 레이어를 더 세밀하게 제어해야 할 때
- 함수 단위로 테스트 코드를 작성하고 싶을 때

두 방식은 한 프로젝트 안에서 공존 가능하다. 기존 어노테이션 방식을 유지하면서 특정 모듈만 라우터 함수로 구성하는 것도 선택지다.

---

## 6. WebFlux 엔드포인트의 핵심: 반환 타입

어노테이션 방식이든 라우터 함수 방식이든, WebFlux 엔드포인트에서 가장 중요한 것은 **반환 타입**이다.

```java
// MVC
public String hello() { return "Hello"; }

// WebFlux
public Mono<String> hello() { return Mono.just("Hello"); }
```

단순히 `Mono`로 감싼 것처럼 보이지만, 이는 단순한 래퍼가 아니다.

`Mono<String>`을 반환한다는 것은 "이 처리는 비동기로 완료될 것이며, 완료되면 그 결과를 스트림으로 흘려보내겠다"는 선언이다. WebFlux 런타임은 이 `Mono`를 구독하고, 값이 준비되면 HTTP 응답으로 변환한다.

이 덕분에 반환 도중 스레드가 블로킹되지 않고, Netty 이벤트 루프는 다음 요청을 계속 받을 수 있다.

---

## 마치며

WebFlux의 엔드포인트 구성 방식을 이해하고 나면, 이전에 배웠던 것들이 다시 연결된다.

- `Mono.deferContextual`로 traceId를 꺼내 로그에 찍는 것
- `subscribeOn(boundedElastic())`으로 블로킹 작업을 격리하는 것
- `onErrorResume`으로 외부 API 실패를 처리하는 것

이 모든 것이 결국 HTTP 핸들러 안에서 일어나는 일이다. **엔드포인트는 파이프라인의 시작점**이고, 여기서 올바르게 설계되어야 이후 처리 흐름이 안정적으로 이어진다.

다음 글에서는 실제 코드 예제들을 통해 지금까지 배운 개념들이 어떻게 조합되는지 살펴본다.

---

## 참고자료

- [Spring WebFlux Reference - Annotated Controllers](https://docs.spring.io/spring-framework/reference/web/webflux/controller.html)
- [Spring WebFlux Reference - Functional Endpoints](https://docs.spring.io/spring-framework/reference/web/webflux-functional.html)
- [Spring WebFlux Reference - ServerWebExchange](https://docs.spring.io/spring-framework/reference/web/webflux/reactive-spring.html)
