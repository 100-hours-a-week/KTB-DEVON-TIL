# [WebFlux 시리즈 7편] WebFlux 실전 패턴, 코드로 완성하기

> 이론은 충분히 쌓았다. 이제 실제 코드로 연결한다.

---

## 0. 이 글을 쓰게 된 계기

1편부터 6편까지 리액티브 스트림즈, Reactor, 스케줄러, Context, 에러 처리, 엔드포인트 구성 방식을 차례로 정리했다.

그런데 각각을 따로 공부할 때는 이해가 됐는데, 실제로 코드를 쓸 때는 다시 막막했다.

> "TraceId를 넘기려면 Context를 쓰면 된다고 했는데, 그걸 WebFilter에서 어떻게 연결하지?"
> "SSE는 Flux로 반환하면 된다고 했는데, 실제 엔드포인트는 어떻게 생겼지?"
> "WebClient로 외부 API 호출할 때 에러 처리는 어디서 해야 하지?"

이 글은 그런 질문들에 대한 답이다. 앞에서 배운 개념들이 실제 코드에서 어떻게 조합되는지, 패턴별로 정리한다.

---

## 1. 기본: @RestController에서 Mono/Flux 반환

가장 먼저 할 것은 익숙한 어노테이션 방식으로 진입하는 것이다. 차이는 반환 타입이다.

```java
@RestController
public class HelloController {

    // 단일 값: Mono<T>
    @GetMapping("/hello")
    public Mono<String> hello() {
        return Mono.just("Hello WebFlux");
    }

    // 여러 값: Flux<T>
    @GetMapping("/numbers")
    public Flux<Integer> numbers() {
        return Flux.range(1, 10);
    }
}
```

`Mono.just(...)`, `Flux.range(...)`는 이미 만들어진 값을 리액티브 스트림으로 감싼다. 실제 비즈니스에서는 서비스 레이어가 `Mono`/`Flux`를 반환하고, 컨트롤러는 그걸 그대로 반환하면 된다.

**핵심**: WebFlux를 처음 도입할 때 기존 로직을 크게 바꾸지 않아도 된다. 반환 타입을 `Mono`/`Flux`로 바꾸는 것부터 시작한다.

---

## 2. 함수형 라우터: 핸들러 분리 패턴

어노테이션 방식에 익숙해졌다면, 라우터 함수 스타일도 써보는 게 좋다.

```java
// 라우팅 정의
@Configuration
public class GreetingRouter {

    @Bean
    public RouterFunction<ServerResponse> route(GreetingHandler handler) {
        return RouterFunctions.route()
            .GET("/functional/hello", handler::hello)
            .GET("/functional/user/{name}", handler::helloUser)
            .build();
    }
}

// 핸들러: 비즈니스 로직 담당
@Component
public class GreetingHandler {

    public Mono<ServerResponse> hello(ServerRequest request) {
        return ServerResponse.ok()
            .bodyValue("Hello Functional WebFlux!");
    }

    public Mono<ServerResponse> helloUser(ServerRequest request) {
        String name = request.pathVariable("name");
        return ServerResponse.ok()
            .bodyValue("Hello, " + name + "!");
    }
}
```

라우터 함수 방식의 장점은 **라우팅 규칙이 한 곳에 모인다**는 것이다. 어노테이션 방식에서는 각 컨트롤러에 분산된 `@GetMapping` 어노테이션을 파악해야 했다면, 여기서는 `route()` 빈 하나만 보면 전체 API 구조가 보인다.

---

## 3. SSE 스트리밍: Flux + interval

WebFlux가 기존 MVC와 가장 뚜렷하게 다른 영역은 **스트리밍**이다.

```java
@GetMapping(value = "/stream/numbers", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamNumbers() {
    return Flux.interval(Duration.ofSeconds(1))
        .map(tick -> "Event #" + tick);
}
```

`Flux.interval(Duration.ofSeconds(1))`은 1초마다 0, 1, 2, 3... 을 무한히 발행하는 스트림을 만든다. `produces = MediaType.TEXT_EVENT_STREAM_VALUE`를 설정하면 HTTP 응답이 한 번에 끝나지 않고 데이터가 생길 때마다 클라이언트로 전송된다.

클라이언트에서는 브라우저 탭에 직접 URL을 열어보면 숫자가 계속 내려오는 것을 볼 수 있다.

**관찰 포인트**: 기존 MVC에서 이 기능을 구현하려면 Long Polling이나 별도 라이브러리가 필요했다. WebFlux에서는 `Flux` 반환 타입 하나로 자연스럽게 처리된다.

---

## 4. WebClient: 논블로킹 외부 API 호출

외부 API를 호출할 때 `RestTemplate`을 쓰면 스레드가 블로킹된다. WebFlux 환경에서는 `WebClient`를 써야 파이프라인 전체가 논블로킹으로 유지된다.

```java
@Service
public class JokeService {

    private final WebClient webClient;

    public JokeService(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("https://api.chucknorris.io")
            .build();
    }

    public Mono<String> getRandomJoke() {
        return webClient.get()
            .uri("/jokes/random")
            .retrieve()
            .bodyToMono(JokeResponse.class)
            .map(JokeResponse::getValue)
            .onErrorResume(e -> Mono.just("Failed to fetch joke: " + e.getMessage()));
    }
}
```

```java
@GetMapping("/joke")
public Mono<String> joke(
    @RequestHeader(value = "Accept-Language", defaultValue = "en") String lang
) {
    return jokeService.getRandomJoke()
        .map(joke -> "[lang=" + lang + "] " + joke);
}
```

핵심 포인트:
- `WebClient.Builder`를 주입받아 사용하면 Spring Boot의 기본 설정(타임아웃, 코덱 등)을 자동으로 적용받는다
- `onErrorResume`으로 외부 API 실패 시 대체 응답을 정의한다
- 전체 파이프라인이 `Mono<String>` 형태로 연결되어 논블로킹이 유지된다

---

## 5. 서비스 레이어 분리: map과 flatMap

실제 애플리케이션에서는 컨트롤러가 서비스를 호출하고, 서비스가 `Mono`/`Flux`를 반환하는 구조다.

```java
@Service
public class HelloService {

    public Mono<String> getGreeting(String name) {
        // 실제로는 DB 조회나 외부 호출이 여기 들어간다
        return Mono.just("Hello, " + name + "!")
            .map(String::toUpperCase);
    }
}

@RestController
public class HelloController {

    private final HelloService helloService;

    @GetMapping("/hello/{name}")
    public Mono<String> hello(@PathVariable String name) {
        return helloService.getGreeting(name)
            .map(greeting -> ">> " + greeting); // 추가 가공
    }
}
```

**`map` vs `flatMap` 선택 기준**

```java
// map: 동기 변환 (T → R)
.map(user -> user.getName())

// flatMap: 비동기 변환 (T → Mono<R> 또는 Flux<R>)
.flatMap(user -> userRepository.findDetailById(user.getId()))
```

`flatMap` 안에서 또 다른 `Mono`/`Flux`를 반환해야 한다면 `flatMap`을 써야 한다. `map` 안에서 `Mono`를 반환하면 `Mono<Mono<T>>`가 되어 버린다.

---

## 6. TraceId + WebFilter + Context: 요청 추적 패턴

실무에서 자주 쓰는 패턴이다. 3편에서 Context를 배웠는데, 그게 실제로 어떻게 쓰이는지가 바로 이 패턴이다.

**흐름**

```
클라이언트 요청
  ↓
WebFilter: X-Trace-Id 헤더 확인 → 없으면 UUID 생성
  ↓
contextWrite(ctx -> ctx.put("traceId", traceId))
  ↓
핸들러: deferContextual로 traceId 꺼내 로그 출력
  ↓
응답 헤더에 X-Trace-Id 추가
```

```java
@Component
public class TraceIdFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String traceId = exchange.getRequest()
            .getHeaders()
            .getFirst("X-Trace-Id");

        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }

        final String finalTraceId = traceId;

        // 응답 헤더에 추가
        exchange.getResponse().getHeaders().add("X-Trace-Id", finalTraceId);

        // Reactor Context에 저장하여 하위 체인에서 접근 가능하게
        return chain.filter(exchange)
            .contextWrite(ctx -> ctx.put("traceId", finalTraceId));
    }
}
```

```java
@GetMapping("/time")
public Mono<String> currentTime() {
    return Mono.deferContextual(ctx -> {
        String traceId = ctx.getOrDefault("traceId", "unknown");
        log.info("[traceId={}] Processing request", traceId);
        return Mono.just("Current time: " + Instant.now());
    });
}
```

**왜 ThreadLocal을 못 쓰는가?**

WebFlux에서는 요청 처리 도중 스레드가 바뀐다. `ThreadLocal`에 traceId를 저장하면, 스레드가 바뀐 뒤에는 값이 사라진다. Reactor Context는 스트림에 바인딩되므로 스레드가 바뀌어도 값이 전파된다.

---

## 7. Sinks: 실시간 브로드캐스트

가장 인상적인 패턴 중 하나다. `Sinks.Many`를 사용하면 POST로 들어온 메시지를 실시간으로 SSE를 구독 중인 모든 클라이언트에게 전달할 수 있다.

```java
@Service
public class ChatService {

    // 다수 구독자에게 브로드캐스트, 백프레셔 버퍼 포함
    private final Sinks.Many<ChatMessage> sink =
        Sinks.many().multicast().onBackpressureBuffer();

    public void publish(String content) {
        ChatMessage message = new ChatMessage(content, Instant.now());
        sink.tryEmitNext(message);
    }

    public Flux<ChatMessage> subscribe() {
        return sink.asFlux();
    }
}
```

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatService chatService;

    // 메시지 발행
    @PostMapping("/messages")
    public Mono<ResponseEntity<Void>> sendMessage(@RequestBody ChatMessageRequest request) {
        chatService.publish(request.getContent());
        return Mono.just(ResponseEntity.status(HttpStatus.CREATED).build());
    }

    // SSE 구독
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ChatMessage> stream() {
        return chatService.subscribe();
    }
}
```

`Sinks.many().multicast().onBackpressureBuffer()`는 여러 구독자에게 같은 메시지를 브로드캐스트하며, 소비가 느린 구독자를 위해 버퍼를 두는 방식이다.

SSE 클라이언트가 `/api/chat/stream`을 열어두면, 이후 `/api/chat/messages`로 POST가 들어올 때마다 모든 구독자에게 메시지가 전달된다.

---

## 8. 조합: 실전에서 보이는 패턴

실제 코드에서는 위에서 본 패턴들이 조합된다.

```java
@GetMapping("/orders/{orderId}")
public Mono<OrderResponse> getOrder(
    @PathVariable String orderId,
    @RequestHeader(value = "X-User-Id", required = false) String userId
) {
    return Mono.deferContextual(ctx -> {
            String traceId = ctx.getOrDefault("traceId", "unknown");
            log.info("[traceId={}] Fetching order {} for user {}", traceId, orderId, userId);
            return orderService.findById(orderId);
        })
        .flatMap(order -> enrichmentService.addDetails(order)) // 추가 정보 조회
        .onErrorResume(OrderNotFoundException.class, e ->
            Mono.error(new ResponseStatusException(HttpStatus.NOT_FOUND, e.getMessage())))
        .onErrorResume(e -> {
            log.error("Unexpected error fetching order {}", orderId, e);
            return Mono.error(new ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR));
        });
}
```

여기서 사용된 패턴:
- `deferContextual`: Context에서 traceId 꺼내 로그 출력
- `flatMap`: 다른 서비스 비동기 호출로 데이터 보완
- `onErrorResume`: 특정 예외 타입별 에러 처리
- 전체 파이프라인이 논블로킹으로 이어짐

---

## 9. 디버깅: 스레드 이름 찍기

WebFlux 코드를 처음 작성할 때, 스레드 흐름이 어떻게 되는지 확인하는 습관이 중요하다.

```java
@GetMapping("/debug")
public Mono<String> debug() {
    return Mono.just("start")
        .doOnNext(v -> log.info("1. thread: {}", Thread.currentThread().getName()))
        .flatMap(v -> externalService.call()
            .doOnNext(r -> log.info("2. thread: {}", Thread.currentThread().getName())))
        .map(result -> "done: " + result)
        .doOnNext(v -> log.info("3. thread: {}", Thread.currentThread().getName()));
}
```

`doOnNext`로 각 단계에서 스레드 이름을 찍으면, 어디서 스레드가 바뀌는지 눈으로 확인할 수 있다. 블로킹 코드가 이벤트 루프 스레드에서 실행되고 있는지 여부도 이렇게 파악할 수 있다.

---

## 마치며

WebFlux 시리즈를 7편까지 쓰면서, 결국 핵심은 하나라는 생각이 든다.

**"데이터가 언제, 어떤 스레드에서, 어떻게 흐르는지 이해하는 것."**

리액티브 스트림즈는 그 흐름의 약속이었고, Reactor는 그 약속을 구현한 도구였으며, 스케줄러는 어느 스레드에서 실행할지를, Context는 그 흐름을 따라 데이터를 어떻게 전달할지를, 에러 처리는 흐름이 끊겼을 때 어떻게 복구할지를 다뤘다.

WebFlux를 처음 쓸 때는 낯설고 복잡하게 느껴진다. 하지만 각 개념이 왜 그렇게 설계되었는지를 이해하고 나면, 선언적인 코드 체인이 오히려 의도를 더 명확하게 드러낸다는 걸 알게 된다.

중요한 건 **왜 이렇게 동작하는가**를 이해하는 것이다. 그 이해 위에서 작성한 코드가 실제로 논블로킹하게, 그리고 안전하게 동작한다.

---

## 참고자료

- [Spring WebFlux Reference - WebClient](https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html)
- [Project Reactor Reference - Sinks](https://projectreactor.io/docs/core/release/reference/#sinks)
- [Spring WebFlux Reference - WebFilter](https://docs.spring.io/spring-framework/reference/web/webflux/reactive-spring.html#webflux-web-handler-api)
- [Project Reactor Reference - Context](https://projectreactor.io/docs/core/release/reference/#context)
