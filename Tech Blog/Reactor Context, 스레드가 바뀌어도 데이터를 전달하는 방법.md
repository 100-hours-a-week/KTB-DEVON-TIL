# [WebFlux 시리즈 3편] Reactor Context, 스레드가 바뀌어도 데이터를 전달하는 방법

> ThreadLocal은 스레드가 바뀌면 데이터를 잃는다. Context는 스레드가 아니라 스트림에 데이터를 붙인다.

## 0. 이 글을 쓰게 된 계기

2편에서 스케줄러를 공부하고 나서 새로운 문제가 생겼다.

스케줄러로 스레드를 전환할 수 있다는 건 알겠는데, 그러면 **요청 단위로 필요한 데이터는 어떻게 전달하지?**

MVC에서는 `ThreadLocal`을 많이 썼다. 요청 하나가 스레드 하나를 점유하니까, 스레드에 데이터를 묶어두면 어디서든 꺼내 쓸 수 있었다. Spring Security가 `SecurityContext`를 전달하는 방식도 내부적으로 `ThreadLocal` 기반이다.

WebFlux에서 이걸 그대로 쓰면?

```java
ThreadLocal<String> traceId = new ThreadLocal<>();
traceId.set("T-1001");

Flux.range(1, 3)
    .publishOn(Schedulers.boundedElastic())  // 스레드 전환
    .map(i -> {
        System.out.println(traceId.get());  // null! 스레드가 바뀌어서 데이터가 없다
        return i;
    })
    .subscribe();
```

스레드가 바뀌는 순간 `ThreadLocal`의 값은 사라진다.

이 문제를 해결하기 위해 Reactor가 제공하는 것이 **Context**다.

이 글은 **WebFlux 학습 시리즈 3편**이다.
- 1편: 리액티브 스트림즈와 Reactor, 표준과 구현체를 함께 이해하기
- 2편: Reactor 스케줄러, 어떤 스레드에서 코드가 실행되는지 이해하기
- 4편: Reactor 에러 처리, onError 신호를 다루는 방법 (예정)

---

## 1. ThreadLocal의 한계를 명확히 이해하자

먼저 왜 `ThreadLocal`이 안 되는지를 제대로 이해해야 한다.

### ThreadLocal의 작동 원리

`ThreadLocal`은 스레드마다 독립적인 저장소를 갖는 구조다.

```
[Thread-A] → ThreadLocal["traceId"] = "T-1001"
[Thread-B] → ThreadLocal["traceId"] = "T-2002"
[Thread-C] → ThreadLocal["traceId"] = null  ← 값을 넣은 적 없음
```

스레드 A에서 넣은 값은 스레드 A에서만 보인다. 스레드가 바뀌면 다른 저장소가 된다.

### WebFlux에서 스레드가 바뀌는 상황

MVC에서는 요청 하나가 하나의 스레드를 처음부터 끝까지 점유한다.
그래서 `ThreadLocal`에 넣은 값이 요청 처리 내내 살아있다.

WebFlux에서는 다르다.

```
요청 처리 흐름:

Netty 이벤트 루프 스레드  →  데이터 수신, 파이프라인 시작
         ↓
boundedElastic 스레드   →  DB 쿼리 (I/O 작업)
         ↓
parallel 스레드          →  데이터 가공 (CPU 작업)
         ↓
Netty 이벤트 루프 스레드  →  응답 전송
```

하나의 요청이 **여러 스레드를 거쳐서 처리**된다. 스레드가 바뀔 때마다 `ThreadLocal`의 데이터는 끊긴다.

---

## 2. Reactor Context: 스레드가 아닌 스트림에 데이터를 붙인다

Context는 발상의 전환이다.

- `ThreadLocal`: 데이터를 **스레드에** 붙인다.
- `Context`: 데이터를 **리액티브 시퀀스(스트림)에** 붙인다.

스트림은 스레드가 바뀌어도 동일한 시퀀스가 유지된다. 그래서 Context에 넣은 데이터는 스케줄러로 스레드가 전환되어도 항상 같은 워크플로우 안에서 접근할 수 있다.

```
[Reactor Context]

스트림 체인 ──────────────────────────────────────────
  Netty 스레드  →  boundedElastic  →  parallel  →  Netty
       ↑                  ↑               ↑
       └──────── Context["traceId"] = "T-1001" 가 여기 붙어서 이동 ────┘
```

스레드가 몇 번 바뀌든 Context는 스트림을 따라다닌다.

---

## 3. Context의 핵심 특징: 불변성

Context는 **불변 객체**다. 한 번 만들어진 Context의 내부 값은 변경되지 않는다.

값을 추가하거나 수정하면, 기존 Context를 수정하는 게 아니라 **새로운 Context 인스턴스**가 만들어진다.

```java
Context ctx1 = Context.of("key", "value1");
Context ctx2 = ctx1.put("key", "value2");  // ctx1은 그대로, ctx2가 새로 생성

// ctx1.get("key") → "value1"  (변경 안 됨)
// ctx2.get("key") → "value2"  (새로운 Context)
```

왜 불변으로 설계했을까?

1. **데이터 안전성**: 여러 연산자를 거치는 동안 Context가 의도치 않게 수정되지 않는다.
2. **동시성 버그 방지**: 여러 스레드가 동시에 같은 Context를 참조해도, 아무도 값을 바꿀 수 없으니 동시성 문제가 생기지 않는다.

---

## 4. Context 사용법: contextWrite와 deferContextual

Context를 다루는 핵심 메서드는 두 가지다.

### contextWrite – Context에 값을 넣는다

```java
Mono.just("hello")
    .contextWrite(Context.of("traceId", "T-1001"))
    // 또는
    .contextWrite(ctx -> ctx.put("traceId", "T-1001"))
```

- `Context.of("key", "value")`: 새 Context를 생성해 체인에 적용한다.
- `ctx -> ctx.put("key", "value")`: 기존 Context에 값을 추가한 새 Context를 만든다. 기존 키는 유지된다.

둘의 차이:

| 방식 | 동작 |
|------|------|
| `Context.of("k", "v")` | 기존 Context를 덮어씌운다 |
| `ctx -> ctx.put("k", "v")` | 기존 Context의 다른 키는 유지하고 해당 키만 추가/변경 |

기존 Context를 유지하면서 값을 추가할 때는 `put()`을 써야 한다.

### deferContextual – Context에서 값을 꺼낸다

```java
Mono.deferContextual(ctx -> {
    String traceId = ctx.getOrDefault("traceId", "no-trace");
    return Mono.just("traceId = " + traceId);
})
```

`deferContextual`은 실행 시점의 Context를 `ContextView`로 제공한다.
`get()`, `getOrDefault()` 등으로 값을 읽는다.

왜 `deferContextual`이 필요한가? `contextWrite`로 넣은 값을 체인 중간에서 읽으려면, 실행 시점에 Context를 바인딩하는 연산자가 필요하기 때문이다.

---

## 5. 가장 중요한 규칙: Context는 아래에서 위로 전파된다

이게 Context에서 가장 헷갈리는 부분이다.

**Context는 구독(subscribe) 시점에서 출발해, 체인의 아래에서 위로(downstream → upstream) 전파된다.**

```java
Mono.deferContextual(ctx -> Mono.just(ctx.get("key")))  // ← 위 (upstream)
    .contextWrite(Context.of("key", "first"))
    .contextWrite(Context.of("key", "second"))  // ← 아래 (downstream)
    .subscribe(System.out::println);

// 출력: second
```

구독이 시작되면 Context가 아래에서 위로 올라간다. 가장 아래의 `contextWrite`가 먼저 적용되고, 그 위로 전파된다.

같은 키가 여러 번 설정되면 **가장 아래쪽(나중에 선언된) `contextWrite`가 우선**한다.

```
[Context 전파 방향]

deferContextual (ctx.get "key" → "second")
        ↑  Context 아래에서 위로 전파
        │
contextWrite("key" = "first")  ← 위에 있으므로 나중에 적용됨
        ↑
        │
contextWrite("key" = "second") ← 아래에 있으므로 먼저 적용됨
        ↑
        │
subscribe() ← 시작점
```

데이터 흐름(onNext)은 위에서 아래로 흐르지만, Context는 정반대 방향이다.

### 실전에서 이 규칙을 어떻게 활용하나

```java
Mono.just("request")
    .flatMap(msg -> Mono.deferContextual(ctx -> {
        String traceId = ctx.getOrDefault("traceId", "no-trace");
        String userId = ctx.getOrDefault("userId", "anonymous");
        return Mono.just(msg + " [trace=" + traceId + ", user=" + userId + "]");
    }))
    .contextWrite(ctx -> ctx.put("userId", "U-100"))   // userId 추가
    .contextWrite(Context.of("traceId", "T-1001"))      // traceId 설정
    .subscribe(System.out::println);

// 출력: request [trace=T-1001, user=U-100]
```

두 개의 `contextWrite`가 있어도 서로 다른 키를 추가하므로 두 값이 모두 살아있다.
`ctx.put()`을 쓰면 기존 키들이 유지되기 때문이다.

---

## 6. 스레드가 바뀌어도 Context는 살아있다

2편에서 스케줄러로 스레드를 전환했을 때 `ThreadLocal`이 깨진다는 걸 봤다.
Context는 어떨까.

```java
Flux.range(1, 3)
    .flatMap(i -> Mono.deferContextual(ctx ->
        Mono.just("[thread=" + Thread.currentThread().getName()
                + ", traceId=" + ctx.getOrDefault("traceId", "none") + "] item=" + i)
    ))
    .publishOn(Schedulers.boundedElastic())  // 스레드 전환
    .contextWrite(Context.of("traceId", "T-1001"))
    .subscribe(System.out::println);
```

```
출력 예시:
[thread=boundedElastic-1, traceId=T-1001] item=1
[thread=boundedElastic-2, traceId=T-1001] item=2
[thread=boundedElastic-3, traceId=T-1001] item=3
```

스레드 이름이 바뀌어도 `traceId`는 동일하게 읽힌다. Context가 스트림을 따라다니기 때문이다.

---

## 7. 실전 패턴: WebFilter에서 Context 주입

Context의 가장 대표적인 실무 활용이 **traceId 전파**다.

HTTP 요청이 들어오면 `X-Trace-Id` 헤더를 읽거나 새로 생성해서 Context에 넣고,
이후 핸들러와 서비스에서 로깅에 활용한다.

```java
@Component
public class TraceIdFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String traceId = exchange.getRequest().getHeaders()
            .getFirst("X-Trace-Id");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }

        final String finalTraceId = traceId;
        return chain.filter(exchange)
            .contextWrite(ctx -> ctx.put("traceId", finalTraceId));  // Context에 주입
    }
}
```

```java
// 서비스 레이어에서 꺼내 사용
public Mono<String> processRequest(String data) {
    return Mono.deferContextual(ctx -> {
        String traceId = ctx.getOrDefault("traceId", "no-trace");
        log.info("[{}] Processing: {}", traceId, data);
        return Mono.just(data.toUpperCase());
    });
}
```

필터에서 넣은 Context가 서비스 레이어까지 스레드 전환 없이 전달된다.
파라미터로 traceId를 모든 메서드에 넘기지 않아도 된다.

---

## 8. Context와 ThreadLocal 비교 정리

| 항목 | ThreadLocal | Reactor Context |
|------|-------------|-----------------|
| 데이터 바인딩 대상 | 스레드 | 리액티브 시퀀스 |
| 스레드 전환 시 | 데이터 손실 | 데이터 유지 |
| 불변성 | X (가변) | O (불변, put 시 새 인스턴스) |
| 전파 방향 | 동일 스레드 내 | downstream → upstream |
| 접근 방법 | `threadLocal.get()` | `Mono.deferContextual()` |
| 주요 용도 | MVC 요청-스레드 바인딩 | WebFlux 요청-시퀀스 바인딩 |

---

## 마치며

Context를 이해하고 나면 WebFlux에서 요청 단위 데이터를 다루는 방식이 명확해진다.

핵심만 정리하면:

1. `ThreadLocal`은 스레드가 바뀌면 데이터를 잃는다 → WebFlux에서 신뢰할 수 없다.
2. Context는 **스트림에 데이터를 붙인다** → 스레드 전환과 무관하게 유지된다.
3. **Context는 불변**이다. `put()`은 새 인스턴스를 반환한다.
4. Context 전파는 **아래에서 위(downstream → upstream)** 방향이다.
5. 같은 키가 여러 `contextWrite`에 있으면 **가장 아래쪽이 우선**한다.

다음 글에서는 **[WebFlux 시리즈 4편] Reactor 에러 처리, onError 신호를 다루는 방법**을 다룬다. `try-catch` 대신 스트림 안에서 에러를 선언적으로 처리하는 방법을 살펴본다.
