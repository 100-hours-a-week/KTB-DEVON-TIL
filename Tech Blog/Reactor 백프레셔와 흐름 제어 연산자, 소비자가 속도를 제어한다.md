# [WebFlux 시리즈 보너스] Reactor 백프레셔와 흐름 제어 연산자, 소비자가 속도를 제어한다

> 리액티브 프로그래밍에서 진짜 어려운 것은 데이터를 흘리는 것이 아니라, 속도를 맞추는 것이다.

---

## 0. 이 글을 쓰게 된 계기

1편에서 백프레셔를 처음 언급했을 때, 설명은 이해했다. "소비자가 처리할 수 있는 만큼만 요청한다." 맞는 말이다.

그런데 실제로 Reactor를 쓸 때 `Flux.range(1, 1000).subscribe(System.out::println)`을 쓰면 백프레셔가 어디 있는지 보이지 않는다. 그냥 다 흘러나온다.

백프레셔가 실제로 어떻게 동작하는지, 그리고 그걸 제어하는 연산자들이 어떤 역할을 하는지를 이 글에서 정리한다.

---

## 1. 왜 백프레셔가 필요한가

동기/블로킹 방식에서는 생산자와 소비자가 자연스럽게 속도를 맞춘다. 소비자 스레드가 데이터를 처리하는 동안 생산자 스레드가 블로킹되기 때문이다.

```
[동기 방식]
생산자: 데이터 생성 → 소비자에게 전달 → (소비자가 처리할 때까지 대기) → 다음 데이터 생성
소비자: 처리중...........완료 → 다음 요청
```

리액티브 방식에서는 이 자연스러운 제동이 없다. 논블로킹이기 때문에 생산자는 소비자가 처리 중이든 아니든 데이터를 계속 밀어낼 수 있다.

```
[리액티브 방식, 백프레셔 없이]
생산자: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10... (계속 밀어냄)
소비자: [1 처리중] [2 버퍼에 쌓임] [3 버퍼에 쌓임] ... → OOM
```

이게 백프레셔가 필요한 이유다. 소비자가 처리할 수 있는 만큼만 데이터가 흘러들어오도록 **소비자가 생산자를 제어**해야 한다.

---

## 2. 리액티브 스트림즈의 백프레셔 프로토콜

1편에서 봤던 인터페이스가 이 문제를 어떻게 해결하는지 다시 본다.

```
subscribe() 호출
  ↓
Publisher → onSubscribe(Subscription) → Subscriber
                                           ↓
                                   subscription.request(3)  ← "3개 주세요"
                                           ↓
Publisher → onNext(1), onNext(2), onNext(3) → Subscriber
                                           ↓
                                   subscription.request(2)  ← "2개 더 주세요"
```

**`Subscription.request(n)`이 핵심이다.** 소비자가 얼마나 받을 수 있는지를 직접 생산자에게 알려준다. 생산자는 요청받은 `n`개를 초과해 데이터를 보내지 않는다.

이것이 "Pull 방식의 백프레셔"다. 소비자가 당기는 만큼만 흐른다.

---

## 3. 기본 subscribe()는 백프레셔를 어떻게 처리하는가

단순 `subscribe()`를 쓰면 백프레셔 제어가 없는 것처럼 보인다.

```java
Flux.range(1, 1000)
    .subscribe(n -> System.out.println(n));
```

실제로는 Reactor가 내부적으로 `Long.MAX_VALUE`를 요청하는 것과 같다. "전부 다 주세요"를 선언한 것이다. 소비자가 충분히 빠르다고 가정하는 상황이다.

백프레셔를 직접 제어하려면 `BaseSubscriber`를 써야 한다.

---

## 4. BaseSubscriber: 수동으로 request(n) 제어

```java
Flux.range(1, 100)
    .subscribe(new BaseSubscriber<Integer>() {

        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            // 시작할 때 1개만 요청
            request(1);
        }

        @Override
        protected void hookOnNext(Integer value) {
            // 받은 데이터 처리
            System.out.println("처리: " + value + " | 스레드: " + Thread.currentThread().getName());

            // 처리가 오래 걸리는 작업 시뮬레이션
            try { Thread.sleep(100); } catch (InterruptedException e) {}

            // 처리가 끝나면 다음 1개 요청
            request(1);
        }
    });
```

`hookOnSubscribe`에서 `request(1)`로 시작하면, 생산자는 처음엔 1개만 보낸다. `hookOnNext`에서 처리를 완료하고 다시 `request(1)`을 호출하면 그때 다음 데이터가 온다.

**소비자가 처리 속도를 결정하는 구조**가 된 것이다.

동적으로 요청량을 조절하는 것도 가능하다.

```java
@Override
protected void hookOnNext(Integer value) {
    processData(value);

    // 시스템 상태에 따라 다음 요청량 조절
    if (isSystemOverloaded()) {
        request(1);   // 부하 높을 때는 천천히
    } else {
        request(10);  // 여유 있을 때는 빠르게
    }
}
```

---

## 5. onBackpressure 연산자: 핫 소스의 과압 처리

Cold Publisher(Flux.range 등)는 request(n)에 맞춰 생성 속도를 조절할 수 있다. 하지만 Hot Publisher(Flux.interval, 외부 이벤트 스트림 등)는 소비자와 무관하게 데이터를 계속 발행한다.

이런 경우 `onBackpressureXXX` 연산자로 과압 처리 전략을 선언한다.

```
Hot Publisher (Flux.interval)
    ↓
onBackpressure전략
    ↓
소비자 (처리 속도 느림)
```

### 5.1 버퍼링: 데이터를 임시 저장

```java
Flux.interval(Duration.ofMillis(10))
    .onBackpressureBuffer(100)  // 최대 100개까지 버퍼
    .subscribe(n -> {
        Thread.sleep(100); // 처리가 느림
        System.out.println(n);
    });
```

소비가 느려도 데이터를 버퍼에 보관해두었다가 차례로 전달한다. 버퍼가 꽉 차면 설정에 따라 에러를 발생시키거나, 새 데이터를 버리거나, 오래된 데이터를 버린다.

**언제 쓰는가**: 데이터 유실이 허용되지 않고, 처리 속도 차이가 일시적일 때.

### 5.2 드롭: 넘치면 버리기

```java
Flux.interval(Duration.ofMillis(10))
    .onBackpressureDrop(dropped -> log.warn("버려진 데이터: {}", dropped))
    .subscribe(n -> {
        Thread.sleep(100);
        System.out.println(n);
    });
```

소비자가 준비되지 않았을 때 들어오는 데이터를 즉시 폐기한다. 콜백으로 어떤 데이터가 버려졌는지 로깅할 수 있다.

**언제 쓰는가**: 실시간 메트릭, 모니터링처럼 모든 데이터가 아닌 최근 추세만 중요할 때.

### 5.3 최신값 유지: 가장 최근 것만

```java
Flux.interval(Duration.ofMillis(10))
    .onBackpressureLatest()
    .subscribe(n -> {
        Thread.sleep(100);
        System.out.println("최신: " + n);
    });
```

소비자가 처리할 준비가 됐을 때, 그 시점까지 쌓인 것 중 **가장 최근 값** 하나만 전달한다. 중간 값들은 버려진다.

**언제 쓰는가**: 주식 현재가, 실시간 대시보드처럼 최신 상태만 중요하고 이전 상태는 무의미할 때.

---

## 6. buffer 연산자: 데이터를 묶어 배치 처리

`buffer`는 백프레셔 전략이라기보다 **데이터 수집 연산자**다. 개별 요소를 `List`로 묶어서 전달한다.

```java
Flux.range(1, 20)
    .buffer(5)  // 5개씩 묶기
    .subscribe(list -> System.out.println("배치: " + list));

// 출력:
// 배치: [1, 2, 3, 4, 5]
// 배치: [6, 7, 8, 9, 10]
// 배치: [11, 12, 13, 14, 15]
// 배치: [16, 17, 18, 19, 20]
```

### 주요 변형

| 메서드 | 설명 |
|--------|------|
| `buffer(int size)` | 지정 개수만큼 쌓이면 방출 |
| `buffer(Duration timespan)` | 주어진 시간 동안 쌓인 것을 방출 |
| `bufferTimeout(int size, Duration timespan)` | 개수 또는 시간 중 먼저 충족된 조건으로 방출 |
| `bufferUntil(Predicate)` | 조건이 true가 될 때까지 모으다 방출 |

**buffer의 실전 활용**

```java
// DB 벌크 인서트: 100개씩 묶어 한 번에 저장
eventFlux
    .buffer(100)
    .flatMap(batch -> repository.saveAll(batch))
    .subscribe();

// 또는 1초마다 쌓인 이벤트를 한 번에 처리
eventFlux
    .bufferTimeout(100, Duration.ofSeconds(1))
    .flatMap(batch -> processEvents(batch))
    .subscribe();
```

개별 요소를 하나씩 DB에 저장하면 100번의 INSERT가 발생하지만, `buffer(100)`으로 묶으면 1번의 bulk INSERT로 처리할 수 있다.

**주의사항**: Hot Publisher에서 크기 기반 버퍼만 쓰면 마지막 버퍼가 꽉 차지 않아 방출되지 않을 수 있다. 이때는 `bufferTimeout`으로 시간 조건을 함께 걸어야 한다.

---

## 7. take 연산자: 조건이 충족되면 스트림 종료

`take`는 필요한 만큼만 받고 스트림을 닫는 연산자다. 백프레셔와는 다른 개념이지만, "얼마나 받을 것인가"를 제어한다는 점에서 흐름 제어의 맥락에 함께 놓인다.

```java
Flux.interval(Duration.ofMillis(500))
    .take(5)  // 5개만 받고 종료
    .subscribe(n -> System.out.println(n));

// 출력: 0, 1, 2, 3, 4 출력 후 종료
```

### 주요 변형

| 메서드 | 설명 |
|--------|------|
| `take(long n)` | 처음 `n`개만 통과시키고 종료 |
| `take(Duration timespan)` | 지정 시간 동안만 수신하고 종료 |
| `takeWhile(Predicate)` | 조건이 true인 동안 통과, false가 되면 종료 |
| `takeUntil(Predicate)` | 조건이 true가 될 때까지 통과, true인 순간 종료 |
| `takeUntilOther(Publisher)` | 다른 Publisher가 신호를 보내면 종료 |

**take의 실전 활용**

```java
// 10초 동안만 이벤트 수집
sensorFlux
    .take(Duration.ofSeconds(10))
    .collectList()
    .subscribe(events -> analyzeEvents(events));

// 특정 이벤트가 올 때까지 수신
eventFlux
    .takeUntil(event -> event.getType() == EventType.SHUTDOWN)
    .subscribe(event -> process(event));
```

---

## 8. buffer vs take 비교

| 구분 | buffer | take |
|------|--------|------|
| 목적 | 데이터를 수집해 배치 단위로 처리 | 필요한 양만 받고 스트림 종료 |
| 데이터 변환 | `T` → `List<T>` | `T` 그대로 |
| 스트림 종료 | 종료하지 않음 | 조건 충족 시 즉시 종료 |
| 비유 | 장바구니에 담았다가 계산대로 | 필요한 물건만 집고 나오기 |

---

## 9. limitRate: Reactor의 내부 prefetch 조절

`onBackpressure` 연산자 외에, Reactor 내부의 요청 전략을 조절하는 방법도 있다.

```java
Flux.range(1, 1000)
    .limitRate(10)  // 한 번에 최대 10개씩 요청
    .subscribe(n -> expensiveProcess(n));
```

기본적으로 Reactor는 업스트림에서 256개씩 prefetch해 오는 경향이 있다. `limitRate(10)`으로 설정하면 10개씩만 미리 가져오므로, 외부 API나 DB 같은 비싼 소스에 부하를 조절할 수 있다.

---

## 10. 아키텍처 관점: 시스템 전체의 백프레셔

애플리케이션 레이어의 백프레셔만으로는 충분하지 않을 때가 있다. 시스템 전체를 봐야 한다.

```
클라이언트
  ↓ (요청 수 제한: 연결 풀, 타임아웃)
WebFlux 애플리케이션
  ↓ (Reactor 백프레셔: request/onBackpressure)
메시지 브로커 (Kafka, RabbitMQ)
  ↓ (컨슈머 그룹, 파티션)
DB / 외부 API
```

각 레이어에 보호 장치가 있어야 한다.

- **Reactor 레이어**: `onBackpressureBuffer`, `limitRate`, `BaseSubscriber`
- **메시지 브로커**: 컨슈머 오프셋, 파티션 할당
- **외부 시스템 보호**: `retry`와 `retryBackoff` 조합, 서킷 브레이커

```java
webClient.get().uri("/external/api")
    .retrieve()
    .bodyToMono(Response.class)
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
        .maxBackoff(Duration.ofSeconds(10)))
    .onErrorResume(e -> Mono.just(fallbackResponse()));
```

실패 시 즉시 재시도하면 이미 부하가 높은 외부 시스템에 더 큰 압박을 준다. `retryBackoff`로 점진적으로 간격을 늘리면 재시도 자체가 또 다른 백프레셔 역할을 한다.

---

## 11. 흔한 실수

**1. 버퍼만 키워서 문제를 덮기**

```java
// 이렇게 하면 안 된다
.onBackpressureBuffer(Integer.MAX_VALUE)
```

당장은 에러가 안 나지만 메모리가 계속 쌓인다. 결국 OOM으로 터진다. 버퍼는 임시방편이지, 근본 해결책이 아니다.

**2. Hot Publisher를 Cold처럼 테스트**

개발 환경에서 데이터가 100개면 문제없다. 운영에서 초당 10만 건이 들어오면 터진다. Hot Publisher는 반드시 부하 조건에서 테스트해야 한다.

**3. 데이터 유실 허용 범위를 정의하지 않기**

`onBackpressureDrop`을 쓸 때 어떤 데이터를 잃어도 괜찮은지 명확히 정의해야 한다. 결제 이벤트는 절대 유실되면 안 되지만, 로그 메트릭은 일부 유실이 허용될 수 있다.

---

## 마치며

백프레셔는 리액티브 프로그래밍에서 가장 미묘한 개념이다. 처음에는 "소비자가 요청한 만큼만 보낸다"는 말을 이해해도, 실제 코드에서 그게 어디서 어떻게 작동하는지 보이지 않는다.

핵심은 **소비자가 주도권을 가진다**는 것이다. 전통적인 생산자-소비자 모델에서는 생산자가 밀어 넣었다면, 리액티브 스트림에서는 소비자가 당겨온다. `request(n)`이 그 당기는 힘이다.

`buffer`, `take`, `limitRate`, `onBackpressureXXX`는 이 흐름을 다양한 방식으로 제어하는 도구들이다. 어떤 도구를 언제 써야 하는지는 데이터 유실 허용 여부, 처리 속도 차이의 성격, 시스템 전체 아키텍처를 함께 고려해야 한다.

중요한 건 **왜 이렇게 설계되었는가**를 이해하는 것이다. 백프레셔가 왜 필요한지를 알면, 어떤 전략을 선택할지는 자연스럽게 따라온다.

---

## 참고자료

- [Project Reactor Reference - Backpressure](https://projectreactor.io/docs/core/release/reference/#reactiveStream.Subscriber.request)
- [Project Reactor Reference - BaseSubscriber](https://projectreactor.io/docs/core/release/reference/#_an_alternative_to_lambdas_basesubscriber)
- [Project Reactor Reference - Handling Errors - Operators](https://projectreactor.io/docs/core/release/reference/#which.errors)
- [Reactive Streams Specification](https://www.reactive-streams.org/)
