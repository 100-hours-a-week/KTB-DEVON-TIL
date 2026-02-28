# [WebFlux 시리즈 2편] Reactor 스케줄러, 어떤 스레드에서 코드가 실행되는지 이해하기

> 비동기 코드를 작성하면서 "지금 이 코드가 어떤 스레드에서 돌고 있지?"를 모르면 언젠가 반드시 사고가 난다.

## 0. 이 글을 쓰게 된 계기

1편에서 리액티브 스트림즈를 공부하면서 이런 생각이 들었다.

> "subscribe()가 실행의 시작점이라는 건 알겠는데, 그러면 subscribe된 코드는 어떤 스레드에서 실행되는 거지?"

기존 Spring MVC에서는 이걸 크게 고민하지 않았다. 요청 하나에 스레드 하나가 할당되고, 그 스레드가 컨트롤러부터 DB 조회까지 전부 처리하니까.

WebFlux에서는 달랐다. 스트림 체인의 각 단계가 **다른 스레드에서 실행될 수 있다.** 어떤 작업을 어떤 스레드에 배치하느냐에 따라 성능이 크게 달라진다.

그래서 **스케줄러(Scheduler)** 라는 개념이 필요하다.

이 글은 **WebFlux 학습 시리즈 2편**이다.
- 1편: [리액티브 스트림즈와 Reactor, 표준과 구현체를 함께 이해하기](./리액티브%20스트림즈와%20Reactor,%20표준과%20구현체를%20함께%20이해하기.md)
- 3편: Reactor Context, 스레드가 바뀌어도 데이터를 전달하는 방법 (예정)

---

## 1. 스케줄러가 뭔지부터

**스케줄러(Scheduler)** 는 "이 작업을 어느 스레드(풀)에서 실행할지"를 제어하는 추상화다.

Reactor 없이 자바로 비동기 코드를 짜려면 직접 `ExecutorService`를 만들고, `submit()`으로 작업을 넘기는 코드를 작성해야 한다. 스케줄러는 이 과정을 리액티브 스트림 체인 안에서 선언적으로 표현할 수 있게 해준다.

```java
// Java 전통 방식
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> {
    // 어떤 작업
});

// Reactor 스케줄러 방식
Flux.range(1, 5)
    .publishOn(Schedulers.boundedElastic())
    .map(i -> doSomething(i))
    .subscribe();
```

선언적으로 "이 구간은 이 스케줄러에서 실행하라"고 명시할 수 있다.

---

## 2. 작업을 먼저 분류해야 스케줄러를 고를 수 있다

스케줄러 종류를 외우기 전에, **내 작업이 어떤 유형인지** 먼저 구분해야 한다.

### I/O-bound 작업 (입출력 중심)

CPU보다 **외부 대기 시간**이 긴 작업이다.

```
[I/O-bound 작업의 스레드 상태]

스레드 A: [DB 쿼리 전송] ─────── [대기 중... ] ─────── [결과 수신]
                                   ↑
                         이 시간 동안 스레드가 놀고 있다
```

- DB 쿼리 실행 및 결과 수신
- 외부 HTTP API 호출
- 파일 읽기/쓰기

스레드가 I/O 대기 중 놀고 있으므로, **많은 스레드를 동시에 운용**하는 게 유리하다. 한 스레드가 대기할 때 다른 스레드가 다른 요청을 처리하면 되니까.

### CPU-bound 작업 (연산 중심)

외부 대기 없이 **CPU를 계속 사용**하는 작업이다.

```
[CPU-bound 작업의 스레드 상태]

스레드 A: [계산 중...▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓]
스레드 B: [계산 중...▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓]
                     ↑
             CPU를 쉬지 않고 사용한다
```

- 암호화/복호화 연산
- 이미지/동영상 인코딩
- 복잡한 수학 연산

CPU를 쉬지 않고 쓰므로, 스레드가 CPU 코어 수보다 많아지면 **컨텍스트 스위칭 오버헤드**만 늘어난다. 코어 수에 맞춰 스레드를 고정하는 게 맞다.

---

## 3. 주요 스케줄러 종류

| 스케줄러 | 스레드 모델 | 적합한 작업 |
|---------|-----------|-----------|
| `Schedulers.boundedElastic()` | 동적 스레드 풀 (최대 CPU×10) | **I/O-bound** |
| `Schedulers.parallel()` | 고정 스레드 풀 (CPU 코어 수) | **CPU-bound** |
| `Schedulers.single()` | 단일 스레드 | 순서 보장이 필요한 직렬 작업 |
| `Schedulers.immediate()` | 현재 스레드 | 스레드 전환 없음 (테스트/디버깅) |
| `Schedulers.fromExecutorService(e)` | 커스텀 스레드 풀 | 기존 ExecutorService 연동 |

### boundedElastic – I/O 작업의 기본 선택

`elastic()`의 후속 버전이다. 스레드를 무한정 만들지 않고 **상한선**을 둔다 (기본: CPU 코어 수 × 10).

```java
// 외부 API 호출 같은 I/O 작업
Flux.range(1, 100)
    .flatMap(id ->
        Mono.fromCallable(() -> externalApiCall(id))
            .subscribeOn(Schedulers.boundedElastic())
    )
    .subscribe();
```

상한선이 없었던 `elastic()`은 트래픽이 몰리면 스레드가 무한 증가할 수 있었다.
`boundedElastic`은 그 문제를 해결하고 자원을 안전하게 관리한다.

### parallel – CPU 작업의 기본 선택

CPU 코어 수만큼 고정된 스레드 풀을 운용한다. 코어 수를 넘어서 스레드를 만들지 않는다.

```java
// 이미지 처리 같은 CPU 집약적 작업
Flux.fromIterable(images)
    .publishOn(Schedulers.parallel())
    .map(image -> processImage(image))
    .subscribe();
```

---

## 4. subscribeOn vs publishOn – 이게 핵심이다

스케줄러를 스트림 체인에 적용하는 연산자가 두 개 있다. 이 둘의 차이가 가장 중요하고 가장 헷갈리는 부분이다.

### subscribeOn – 소스부터 전체 업스트림의 스레드를 바꾼다

`subscribeOn`은 연산자 체인에서 어디에 위치하든, **데이터 소스가 실행되는 스레드**를 지정한다.

```java
Flux.range(1, 5)          // boundedElastic에서 실행
    .map(i -> i * 2)      // boundedElastic에서 실행
    .filter(i -> i > 4)   // boundedElastic에서 실행
    .subscribeOn(Schedulers.boundedElastic())  // 위치와 상관없이 소스에 영향
    .subscribe(System.out::println);
```

```
[흐름]
subscribe() 호출
  ↓
subscribeOn이 boundedElastic 스레드에서 구독 시작
  ↓
range(1, 5) → map → filter 모두 boundedElastic에서 실행
```

`subscribeOn`을 체인 중간에 두어도 결과는 같다. 항상 **소스의 실행 스레드**를 결정한다.

### publishOn – 해당 위치 이후 다운스트림만 바꾼다

`publishOn`은 위치가 중요하다. **이 연산자가 있는 지점부터** 아래 방향(다운스트림)의 스레드를 전환한다.

```java
Flux.range(1, 5)
    .map(i -> {
        System.out.println("stage1: " + Thread.currentThread().getName());
        return i * 2;
    })
    .publishOn(Schedulers.parallel())    // 이 지점부터 스레드 전환
    .map(i -> {
        System.out.println("stage2: " + Thread.currentThread().getName());
        return i + 1;
    })
    .subscribe(i -> System.out.println("subscribe: " + Thread.currentThread().getName()));
```

```
출력 예시:
stage1: main          ← publishOn 이전, 기존 스레드
stage1: main
stage2: parallel-1    ← publishOn 이후, parallel 스레드
stage2: parallel-1
subscribe: parallel-1
```

### 둘을 조합하면

실제 파이프라인에서는 두 연산자를 조합한다.

```java
Flux.range(1, 5)
    .subscribeOn(Schedulers.boundedElastic())  // 소스는 boundedElastic에서 실행
    .map(i -> fetchFromDb(i))                  // boundedElastic (I/O 작업)
    .publishOn(Schedulers.parallel())          // 이후부터 parallel로 전환
    .map(data -> processData(data))            // parallel (CPU 작업)
    .subscribe();
```

```
[구간별 실행 스레드]

range → fetchFromDb : boundedElastic (I/O 대기가 있는 구간)
          ↓ publishOn
processData → subscribe : parallel (CPU 연산 구간)
```

I/O 구간과 CPU 구간을 각각 최적화된 스레드 풀에 배치한 것이다.

---

## 5. 정리: subscribeOn vs publishOn 한 장으로

```
[subscribeOn]

소스 ─── map ─── filter ─── subscribeOn(A) ─── subscribe
  ↑                            ↑
  └─────────── A에서 실행 ──────┘
  (위치와 무관하게 소스부터 전체 업스트림에 영향)


[publishOn]

소스 ─── map1 ─── publishOn(B) ─── map2 ─── subscribe
           ↑              ↑           ↑
       기존 스레드    전환 지점      B에서 실행
                    (위치 이후 다운스트림에만 영향)
```

| 항목 | `subscribeOn` | `publishOn` |
|------|--------------|------------|
| 영향 범위 | 소스 포함 전체 업스트림 | 연산자 위치 이후 다운스트림 |
| 위치 중요성 | 위치 무관 (항상 소스에 영향) | 위치가 매우 중요 |
| 주요 용도 | 데이터 소스 실행 스레드 지정 | 중간 처리 단계 스레드 전환 |

---

## 6. 블로킹 코드가 왜 위험한가

WebFlux를 쓰면서 가장 주의해야 할 규칙이 있다.

> **이벤트 루프 스레드에서 블로킹 코드를 실행하면 안 된다.**

WebFlux는 Netty를 기반으로 동작하며, 적은 수의 **이벤트 루프 스레드**가 모든 요청을 처리한다.

```
[이벤트 루프 스레드가 블로킹되면]

이벤트 루프 스레드 (reactor-http-nio-1)
  → 요청 A 처리 시작
  → Thread.sleep(5000)  ← 5초 동안 이 스레드가 멈춤
  → 이 시간 동안 다른 요청도 처리 불가
```

이벤트 루프 스레드가 멈추면, 해당 스레드가 담당하는 **다른 모든 요청도 함께 지연**된다. MVC와 달리 스레드 수가 적기 때문에 피해가 훨씬 크다.

### 블로킹이 불가피한 경우

레거시 라이브러리나 JDBC처럼 블로킹 API만 존재하는 경우가 있다.
이때는 `Mono.fromCallable()`과 `subscribeOn(Schedulers.boundedElastic())`로 블로킹 작업을 별도 스레드 풀에 격리한다.

```java
// 블로킹 코드를 boundedElastic으로 격리
Mono.fromCallable(() -> jdbcRepository.findById(id))  // 블로킹 JDBC 호출
    .subscribeOn(Schedulers.boundedElastic())          // 이벤트 루프 보호
    .map(entity -> toDto(entity))
    .subscribe();
```

`Mono.fromCallable()`은 블로킹 코드를 Mono로 감싸주고, `subscribeOn`으로 해당 작업이 별도 스레드에서 실행되도록 격리한다. 이벤트 루프 스레드는 보호된다.

---

## 7. 스레드 이름으로 흐름을 추적하는 습관

스케줄러를 이해하는 가장 좋은 방법은 로그로 직접 확인하는 것이다.

```java
Flux.range(1, 3)
    .doOnNext(i -> System.out.println("[range] thread: " + Thread.currentThread().getName()))
    .subscribeOn(Schedulers.boundedElastic())
    .map(i -> i * 2)
    .doOnNext(i -> System.out.println("[map] thread: " + Thread.currentThread().getName()))
    .publishOn(Schedulers.parallel())
    .map(i -> i + 1)
    .doOnNext(i -> System.out.println("[final] thread: " + Thread.currentThread().getName()))
    .subscribe();
```

`doOnNext()`는 스트림 흐름을 방해하지 않으면서 각 단계에서 값을 관찰할 수 있는 연산자다.
스레드 이름을 찍어보면 각 단계가 어느 스레드에서 실행되는지 눈으로 확인할 수 있다.

Spring Security 디버깅에서 필터 로그를 켜는 것처럼,
WebFlux에서는 **스레드 이름 로그**를 켜두고 확인하는 습관이 도움이 된다.

---

## 마치며

스케줄러를 이해하고 나면, WebFlux 코드를 볼 때 "이 코드가 어느 스레드에서 실행될까?"를 자연스럽게 생각하게 된다.

요약하면:

1. **I/O 작업** → `boundedElastic`
2. **CPU 작업** → `parallel`
3. **소스 실행 스레드** → `subscribeOn`
4. **중간 처리 스레드 전환** → `publishOn`
5. **이벤트 루프 스레드에 블로킹 코드** → 절대 금지, `fromCallable` + `boundedElastic`으로 격리

이 원칙들을 머릿속에 두고 코드를 작성하면, 나중에 "왜 응답이 느리지?", "왜 타임아웃이 나지?" 같은 문제를 추적할 때 훨씬 빠르게 원인을 찾을 수 있다.

다음 글에서는 **[WebFlux 시리즈 3편] Reactor Context, 스레드가 바뀌어도 데이터를 전달하는 방법**을 다룬다. 스케줄러로 스레드가 바뀌는 상황에서, 요청 단위의 데이터(예: traceId, 사용자 정보)를 어떻게 전달하는지를 살펴본다.
