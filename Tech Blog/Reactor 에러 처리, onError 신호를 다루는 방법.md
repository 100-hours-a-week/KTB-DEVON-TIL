# [WebFlux 시리즈 4편] Reactor 에러 처리, onError 신호를 다루는 방법

> try-catch는 동기 세계의 도구다. 비동기 스트림에서 에러는 신호로 흐른다.

## 0. 이 글을 쓰게 된 계기

Reactor를 처음 배울 때 에러 처리가 가장 낯설었다.

MVC에서는 익숙했다.

```java
try {
    User user = userRepository.findById(id);
    return user;
} catch (Exception e) {
    return defaultUser();
}
```

`try-catch`로 감싸면 끝이다. 예외가 나면 `catch` 블록이 실행된다.

Reactor에서 같은 방식으로 하면?

```java
Mono.fromCallable(() -> userRepository.findById(id))
    .map(user -> {
        try {
            return process(user);  // 예외 발생 가능
        } catch (Exception e) {
            return defaultUser();  // 스트림 체인을 끊고 여기서 처리
        }
    });
```

이렇게도 할 수 있지만, 스트림 체인 안에 명령형 `try-catch`를 섞으면 코드가 지저분해진다. 무엇보다 **스트림 레벨의 에러**를 제대로 다루지 못한다.

Reactor에서 에러는 `onError` 신호로 흐른다. 그리고 이 신호를 처리하는 전용 연산자들이 있다.

이 글은 **WebFlux 학습 시리즈 4편**이다.
- 1편: 리액티브 스트림즈와 Reactor, 표준과 구현체를 함께 이해하기
- 2편: Reactor 스케줄러, 어떤 스레드에서 코드가 실행되는지 이해하기
- 3편: Reactor Context, 스레드가 바뀌어도 데이터를 전달하는 방법

---

## 1. 에러가 발생하면 스트림은 어떻게 되나

먼저 기본 동작을 이해해야 한다.

Reactor 스트림에서 예외가 발생하면:

```
정상 흐름:  onNext → onNext → onNext → onComplete
에러 흐름:  onNext → onNext → onError (이후 중단)
```

1. 예외가 캡처되어 스트림이 즉시 종료된다.
2. 이후의 `map`, `filter` 등 연산자는 **실행되지 않는다.**
3. `onError` 신호가 Subscriber까지 전파된다.
4. `onComplete`는 **호출되지 않는다.**

```java
Flux.just(1, 2, 0, 3)
    .map(i -> 10 / i)   // i=0일 때 ArithmeticException 발생
    .subscribe(
        value -> System.out.println("onNext: " + value),
        error -> System.out.println("onError: " + error.getMessage()),
        () -> System.out.println("onComplete")
    );
```

```
출력:
onNext: 10
onNext: 5
onError: / by zero   ← 여기서 종료, onComplete는 호출 안 됨
```

`0` 이후의 `3`은 처리되지 않는다. 에러가 발생한 순간 스트림이 종료되기 때문이다.

이 기본 동작을 바꾸는 게 에러 처리 연산자들의 역할이다.

---

## 2. 에러 처리 연산자 6가지

### 2-1. onErrorReturn – 기본값으로 대체하고 종료

에러가 발생하면 **미리 정한 기본값 하나**를 방출하고 스트림을 정상 종료한다.

```java
Flux.just(1, 2, 0, 3)
    .map(i -> 10 / i)
    .onErrorReturn(-1)
    .subscribe(System.out::println);
```

```
출력:
10
5
-1   ← 에러 대신 기본값, 이후 onComplete
```

`0` 이후의 `3`은 여전히 처리되지 않는다. 에러가 발생한 시점에 스트림이 종료되고 기본값만 대신 방출되는 구조다.

에러 타입을 지정해서 특정 예외에만 적용할 수도 있다.

```java
.onErrorReturn(ArithmeticException.class, -1)  // ArithmeticException일 때만 기본값
```

**언제 쓰나:** 에러가 발생해도 의미 있는 기본값을 내려줄 수 있을 때. 빈 목록, 기본 응답 등.

---

### 2-2. onErrorResume – 대체 스트림으로 이어간다

에러가 발생하면 **새로운 Publisher(Mono/Flux)** 로 전환해서 계속 방출한다.

```java
Flux.just(1, 2, 0, 3)
    .map(i -> 10 / i)
    .onErrorResume(e -> Flux.just(-4, -5))
    .subscribe(System.out::println);
```

```
출력:
10
5
-4   ← 에러 발생 후 대체 스트림으로 전환
-5
```

`onErrorReturn`과 달리 대체 스트림에서 여러 값을 방출할 수 있다. 에러 객체를 받아서 동적으로 대응할 수도 있다.

```java
.onErrorResume(e -> {
    if (e instanceof TimeoutException) {
        return Mono.just(cachedValue);  // 타임아웃 → 캐시 값 반환
    }
    return Mono.error(e);  // 다른 에러는 그대로 전파
})
```

**언제 쓰나:** 에러 종류에 따라 다른 방식으로 복구가 필요할 때. Fallback API 호출, 캐시 조회 등.

---

### 2-3. onErrorContinue – 에러 유발 요소만 건너뛴다

에러가 발생한 요소만 **스킵**하고 나머지 요소는 계속 처리한다. 스트림이 종료되지 않는다.

```java
Flux.just(1, 2, 0, 3)
    .map(i -> 10 / i)
    .onErrorContinue((error, value) -> {
        System.out.println("에러 발생한 값: " + value + ", 에러: " + error.getMessage());
    })
    .subscribe(System.out::println);
```

```
출력:
10
5
에러 발생한 값: 0, 에러: / by zero
3   ← 0이 스킵되고 3은 정상 처리
```

`(error, value)` BiConsumer를 통해 에러를 일으킨 원본 값(`0`)과 예외 객체에 접근할 수 있다. 이 시점에 로깅을 남기거나 별도 처리를 할 수 있다.

**주의:** `onErrorContinue`는 동작 방식이 다른 연산자들과 다르다. 에러를 다운스트림에서 잡는 게 아니라 **업스트림으로 에러를 회피**하는 방식이다. `flatMap` 내부 같은 특정 상황에서 예상과 다르게 동작할 수 있다. 복잡한 체인에서는 신중하게 써야 한다.

**언제 쓰나:** 일부 데이터에서 에러가 발생해도 나머지를 모두 처리해야 하는 배치성 작업.

---

### 2-4. onErrorMap – 예외를 다른 예외로 변환한다

에러를 **다른 타입의 예외로 바꿔서** 전파한다. 스트림이 종료되는 건 동일하다.

```java
Flux.just(1, 2, 0, 3)
    .map(i -> 10 / i)
    .onErrorMap(e -> new CustomException("계산 오류: " + e.getMessage(), e))
    .subscribe(
        System.out::println,
        error -> System.out.println("최종 에러: " + error.getClass().getSimpleName())
    );
```

```
출력:
10
5
최종 에러: CustomException
```

`try-catch`에서 `throw new CustomException(e)` 하는 것과 같은 역할이다.

**언제 쓰나:** 인프라 레벨 예외(JDBC, HTTP 클라이언트 예외 등)를 도메인 예외로 변환할 때. 예외 계층을 일관되게 유지하고 싶을 때.

---

### 2-5. onErrorStop – 추가 에러 처리를 막는다

에러가 발생하면 즉시 종료하고, **이후에 정의된 다른 에러 처리 연산자의 적용을 차단**한다.

```java
Flux.just(1, 2, 0, 3)
    .map(i -> 10 / i)
    .onErrorStop()         // 이 아래의 retry 등을 막는다
    .retry(2)
    .subscribe(System.out::println);
```

주로 `retry`와 조합해서 특정 조건에서 재시도를 막을 때 사용한다.

**언제 쓰나:** 특정 에러는 재시도해도 의미가 없는 경우. 예를 들어 인증 실패는 재시도가 필요 없다.

---

### 2-6. retry – 에러 발생 시 재구독한다

에러가 발생하면 **처음부터 다시 구독**한다.

```java
Flux.just(1, 2, 0, 3)
    .map(i -> 10 / i)
    .retry(1)  // 최대 1번 재시도
    .subscribe(
        System.out::println,
        error -> System.out.println("최종 에러: " + error.getMessage())
    );
```

```
출력:
10   ← 첫 번째 시도
5
10   ← 재시도 (처음부터 다시)
5
최종 에러: / by zero  ← 재시도도 실패
```

`retry`에서 가장 중요한 주의사항이다. **처음부터 다시 실행**되므로, 이미 처리된 `1`, `2`가 두 번 처리된다. 데이터가 DB에 저장되거나 외부 API를 호출하는 경우라면 **중복 처리**가 발생할 수 있다.

조건부 재시도와 지연 재시도도 지원한다.

```java
// 특정 예외에만 재시도
.retry(3, e -> e instanceof TimeoutException)

// 지연을 두고 재시도 (exponential backoff)
.retryWhen(Retry.backoff(3, Duration.ofSeconds(1)))
```

**언제 쓰나:** 일시적인 네트워크 장애, 타임아웃 같이 재시도하면 성공할 가능성이 있는 경우. 반드시 idempotent(멱등성)한 작업이어야 한다.

---

## 3. 각 연산자 한눈에 비교

| 연산자 | 에러 후 동작 | 스트림 종료 여부 | 사용 목적 |
|--------|------------|----------------|---------|
| `onErrorReturn` | 기본값 1개 방출 후 종료 | O (정상 종료) | 단순 기본값 대체 |
| `onErrorResume` | 대체 Publisher로 전환 | O (대체 스트림 완료 시) | 동적 Fallback |
| `onErrorContinue` | 에러 요소 스킵, 계속 진행 | X (계속 처리) | 부분 실패 허용 |
| `onErrorMap` | 다른 예외로 변환해 전파 | O (에러로 종료) | 예외 계층 변환 |
| `onErrorStop` | 즉시 종료, 추가 처리 차단 | O (에러로 종료) | 재시도 방지 |
| `retry` | 처음부터 재구독 | O (최종 실패 시) | 일시적 오류 재시도 |

---

## 4. try-catch와 대응 관계

처음에 이 연산자들이 낯설었던 건, `try-catch`와 연결해서 생각하지 않았기 때문이다.

```java
// try-catch 방식
try {
    String result = callApi();  // 실패 가능
    return result;
} catch (TimeoutException e) {
    return cachedValue;  // → onErrorResume
} catch (Exception e) {
    return "default";   // → onErrorReturn
}

// retry
for (int i = 0; i < 3; i++) {
    try {
        return callApi();
    } catch (Exception e) {
        if (i == 2) throw e;
    }
}
```

```java
// Reactor 방식
Mono.fromCallable(() -> callApi())
    .onErrorResume(TimeoutException.class, e -> Mono.just(cachedValue))
    .onErrorReturn("default")
    .retry(3)
```

동작은 같다. 표현 방식이 **명령형 → 선언형**으로 바뀐 것이다.

---

## 5. 에러 처리 연산자 위치가 중요하다

연산자들은 체인에서 **어디에 위치하느냐에 따라** 커버하는 범위가 달라진다.

```java
Flux.just(1, 2, 0, 3)
    .map(i -> 10 / i)          // 에러 발생 지점
    .onErrorReturn(-1)          // 위의 map 에러만 처리
    .map(i -> i * 2)           // 이 map은 이미 에러가 처리된 이후
    .subscribe(System.out::println);
```

```
출력:
20   (10 * 2)
10   (5 * 2)
-2   (-1 * 2, onErrorReturn이 -1을 방출한 후 정상 처리)
```

`onErrorReturn(-1)`은 그 위의 `map`에서 발생한 에러만 처리한다. 아래의 `map`에서 에러가 나면 처리하지 못한다.

---

## 6. 글로벌 에러 처리: Hooks

개별 스트림마다 에러 처리를 붙이는 것 외에, 애플리케이션 전체에 적용되는 **글로벌 에러 처리**도 있다.

`Hooks`를 이용해 처리되지 않은 에러에 대한 전역 동작을 정의한다.

```java
// 처리되지 않은 에러에 대한 전역 로깅
Hooks.onErrorDropped(error -> {
    log.error("처리되지 않은 에러: ", error);
});
```

Spring WebFlux에서는 `@RestControllerAdvice` + `ResponseEntityExceptionHandler`가 WebFlux 레벨의 에러를 HTTP 응답으로 변환하는 역할을 담당한다. 이것이 실질적인 글로벌 에러 처리 진입점이다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CustomException.class)
    public ResponseEntity<ErrorResponse> handleCustomException(CustomException e) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(e.getMessage()));
    }
}
```

---

## 마치며

처음에 Reactor 에러 처리가 낯설었던 이유는, 에러를 "발생시키고 잡는" 것이 아니라 **"신호로 흘려보내고 연산자로 제어하는"** 방식이기 때문이었다.

각 연산자의 역할을 `try-catch`와 대응해서 이해하면 훨씬 직관적이다.

- **기본값이 필요하면** → `onErrorReturn`
- **Fallback 로직이 필요하면** → `onErrorResume`
- **일부 실패는 무시하고 계속 진행하면** → `onErrorContinue`
- **예외 타입을 바꿔야 하면** → `onErrorMap`
- **재시도가 필요하면** → `retry` (단, idempotent 작업에만)

중요한 건 **왜 이 연산자인가**를 이해하는 것이다. 상황에 맞게 골라 쓸 수 있으면, 비동기 스트림의 에러 처리도 동기 코드만큼 자연스러워진다.

다음 글에서는 **[WebFlux 시리즈 5편] Spring WebFlux 개요, 왜 WebFlux를 쓰는가**를 다룬다. 지금까지 배운 Reactor 기반 위에서 WebFlux가 어떻게 동작하는지, MVC와 무엇이 다른지를 살펴본다.
