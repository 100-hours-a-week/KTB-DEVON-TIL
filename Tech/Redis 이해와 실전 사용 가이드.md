# Redis 이해와 실전 사용 가이드 (학습 노트, Spring 중심)

> 이 문서는 "Redis를 캐시 정도로만 아는 상태"에서 "운영 중 장애를 줄일 수 있는 설계 감각"까지 가기 위해 정리한 학습 노트다. 단순 명령어 나열이 아니라, 왜 이런 선택을 하는지(원리)와 Spring에서 어떻게 구현하는지(실전)를 같이 본다.

## 1. Redis를 처음 이해할 때 내가 먼저 잡은 관점

처음에는 Redis를 "메모리 DB라서 빠른 저장소" 정도로 이해했는데, 실무에서 중요한 건 속도 자체보다 아래 세 가지였다.

1. Redis는 **자료구조 서버**다. (String/Hash/List/ZSet/Stream이 모델링 핵심)
2. Redis는 **단일 명령의 비용 편차**가 매우 크다. (`GET`은 가볍지만, 큰 키 전체 조회는 위험)
3. Redis는 **운영 옵션**(TTL, eviction, persistence, replication)에 따라 성격이 크게 바뀐다.

즉, "빠른 캐시"가 아니라 "자료구조 + 원자 연산 + 운영 정책"의 조합으로 보는 것이 맞다.

## 2. 알면 실수 줄어드는 내부 동작

### 2.1 만료(Expire)는 즉시 삭제가 아니다
- Redis 만료는 크게 두 경로로 처리된다.
- Passive expiration: 키를 읽을 때 만료 여부를 확인하고 제거
- Active expiration: 백그라운드 샘플링으로 만료 키를 주기적으로 정리

이 의미는 명확하다. TTL이 지났다고 해서 모든 키가 정확히 그 시각에 사라지는 것은 아니다. 만료 직후 짧은 구간에서 메모리에 남아 있을 수 있다. 실무에서 "정확한 시각 동기화"가 필요하면 애플리케이션 레벨 검증까지 고려해야 한다.

### 2.2 eviction은 완전한 LRU가 아니라 근사(approximation)
Redis의 LRU/LFU는 엄밀한 전체 정렬 기반이 아니라 샘플링/근사 전략이다. 그래서 트래픽 패턴이 비정상(핫키 쏠림)일 때 기대와 다른 축출이 나올 수 있다. 정책 선택 전에 `maxmemory-policy`, hit ratio, `evicted_keys`를 같이 봐야 한다.

### 2.3 Big Key가 지연 스파이크를 만든다
Hash/List/ZSet 한 개 키가 과도하게 커지면 단건 명령도 무거워진다. 특히 전체 조회 계열(`HGETALL`, `LRANGE 0 -1`, 큰 `ZRANGE`)은 P99를 크게 악화시킨다. 운영 전부터 "키 하나의 최대 크기"를 제한하는 설계가 필요하다.

### 2.4 Redis 6+는 I/O 스레드를 쓰지만, 명령 실행은 본질적으로 단일 흐름
네트워크 I/O 최적화가 있어도 무거운 명령이 이벤트 루프를 오래 점유하면 전체 지연이 오른다. 그래서 "명령 복잡도"는 여전히 핵심이다.

## 3. 자료구조 선택: 명령이 아니라 접근 패턴으로 결정

- String: 토큰, 단일 JSON, 카운터
- Hash: 객체 필드 묶음 (부분 업데이트/부분 조회가 잦을 때)
- List: 최근 N개 로그/채팅
- Set: 중복 없는 멤버십
- ZSet: 점수 정렬, 랭킹, 지연 큐
- Stream: 소비자 그룹 기반 이벤트 처리

내가 헷갈렸던 포인트는 "객체니까 Hash"라는 고정관념이었다. 실제로는 읽기 패턴이 더 중요하다.
- 전체 객체 조회가 대부분이면 String(JSON)이 단순할 때가 많다.
- 필드 부분 변경/부분 조회가 많으면 Hash가 유리하다.
- 시간순 정렬이 핵심이면 Hash+정렬보다 ZSet이 훨씬 안전하다.

## 4. 운영에서 잘 터지는 주제

### 4.1 캐시 스탬피드(동시 만료 폭주)
동일 키가 동시에 만료되면 DB로 트래픽이 급격히 몰린다. 완화 방법은 보통 아래를 같이 쓴다.
- TTL jitter: 기본 TTL + 랜덤 오프셋
- 요청 병합(single flight): 동일 키 재계산 1회로 제한
- 소프트 TTL: 오래된 값을 잠깐 허용하면서 백그라운드 갱신

### 4.2 핫키 문제
트래픽이 하나의 키에 집중되면 Redis가 아니라 그 키가 병목이 된다. 완화는
- 키 샤딩(`key:0..N`)
- 로컬 캐시(Caffeine) + Redis 2계층
- 읽기 복제본 분산

### 4.3 영속성 옵션의 현실적인 해석
- RDB는 빠른 복구에 강하지만 스냅샷 간격 사이의 데이터는 유실 가능
- AOF `everysec`는 보통 1초 내외 유실 가능성을 받아들이는 옵션
- 장애 허용도가 낮은 도메인은 `appendfsync always/everysec`의 비용-안전 트레이드오프를 사전에 합의해야 한다

### 4.4 복제/장애조치에서 놓치기 쉬운 점
Redis replication은 비동기다. 즉, 마스터 응답 이후 복제본 반영 전 장애가 나면 데이터 손실이 생길 수 있다. 강한 내구성이 필요하면 `WAIT` 같은 동기화 보조 전략을 검토한다.

### 4.5 Cluster 제약
Cluster에서는 멀티키 명령이 같은 hash slot에 있어야 안전하다. 그래서 키에 해시태그(`{user:123}:cart`, `{user:123}:coupon`)를 넣어 슬롯을 고정하는 설계를 자주 쓴다.

## 5. Spring에서의 실전 구현 포인트

### 5.1 CacheManager에 TTL/직렬화/Null 정책을 명시
```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory cf) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .disableCachingNullValues()
                .entryTtl(Duration.ofMinutes(10));

        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
                "productDetail", defaultConfig.entryTtl(Duration.ofMinutes(5)),
                "hotTimeline", defaultConfig.entryTtl(Duration.ofSeconds(30))
        );

        return RedisCacheManager.builder(cf)
                .cacheDefaults(defaultConfig)
                .withInitialCacheConfigurations(cacheConfigs)
                .build();
    }
}
```

핵심은 "캐시별 TTL 분리"다. 모든 데이터를 10분으로 고정하면 히트율과 신선도가 동시에 나빠진다.

### 5.2 캐시 스탬피드 완화 예시 (간단한 single-flight)
```java
@Service
@RequiredArgsConstructor
public class ProductFacade {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductRepository productRepository;

    public ProductDetail getProduct(Long productId) {
        String key = "product:detail:" + productId;
        Object cached = redisTemplate.opsForValue().get(key);
        if (cached instanceof ProductDetail detail) {
            return detail;
        }

        String lockKey = "lock:" + key;
        String token = UUID.randomUUID().toString();
        Boolean locked = redisTemplate.opsForValue().setIfAbsent(lockKey, token, Duration.ofSeconds(3));

        if (Boolean.TRUE.equals(locked)) {
            try {
                ProductDetail loaded = productRepository.findDetail(productId)
                        .orElseThrow(() -> new IllegalArgumentException("상품이 존재하지 않습니다."));

                long ttlSeconds = 240 + ThreadLocalRandom.current().nextLong(0, 60); // jitter
                redisTemplate.opsForValue().set(key, loaded, Duration.ofSeconds(ttlSeconds));
                return loaded;
            } finally {
                releaseLockSafely(lockKey, token);
            }
        }

        // 다른 스레드가 재생성 중이면 짧게 대기 후 재조회
        try {
            Thread.sleep(40L);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        Object retried = redisTemplate.opsForValue().get(key);
        if (retried instanceof ProductDetail detail) {
            return detail;
        }

        return productRepository.findDetail(productId)
                .orElseThrow(() -> new IllegalArgumentException("상품이 존재하지 않습니다."));
    }

    private void releaseLockSafely(String lockKey, String token) {
        String lua = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "return redis.call('del', KEYS[1]) else return 0 end";
        redisTemplate.execute(new DefaultRedisScript<>(lua, Long.class), List.of(lockKey), token);
    }
}
```

주의: 분산 락은 "락 획득"보다 "락 해제 안전성"이 더 중요하다. 단순 `DEL`은 다른 요청의 락을 지울 수 있다.

### 5.3 Stream 소비자 그룹 (이벤트 처리)
```java
@Bean
public Subscription streamSubscription(RedisConnectionFactory cf) {
    StreamMessageListenerContainerOptions<String, MapRecord<String, String, String>> options =
            StreamMessageListenerContainerOptions.builder()
                    .pollTimeout(Duration.ofSeconds(1))
                    .build();

    StreamMessageListenerContainer<String, MapRecord<String, String, String>> container =
            StreamMessageListenerContainer.create(cf, options);

    container.receiveAutoAck(
            Consumer.from("chat-group", "consumer-1"),
            StreamOffset.create("chat-events", ReadOffset.lastConsumed()),
            message -> {
                // 메시지 처리
            }
    );
    container.start();
    return container;
}
```

실무 포인트: Stream은 PEL(Pending Entries List) 관리가 핵심이다. 장애 복구 시 `XAUTOCLAIM` 전략이 없으면 유실이 아니라 "미처리 누적"이 발생한다.

## 6. 초급에서 중급으로 올라갈 때 꼭 보는 체크리스트

1. 키 네이밍에 도메인/범위/버전이 포함되어 있는가
2. 모든 캐시에 TTL이 있는가
3. 키별 최대 크기(Big Key 기준)를 정해두었는가
4. eviction 정책이 트래픽 패턴과 맞는가 (`allkeys-lru`, `allkeys-lfu`, `volatile-ttl`)
5. 장애 시 데이터 유실 허용 범위를 문서화했는가
6. 모니터링 지표(`used_memory`, `mem_fragmentation_ratio`, `keyspace_hits/misses`, `evicted_keys`, `rejected_connections`)를 대시보드로 보고 있는가

## 7. 자주 나오는 오해 정리

- 오해 1: Redis는 무조건 빠르다.
- 정리: 명령/키 크기/네트워크/직렬화 비용에 따라 느려질 수 있다.

- 오해 2: TTL 지나면 즉시 사라진다.
- 정리: 만료는 lazy + active 샘플링 조합이다.

- 오해 3: 캐시 정합성은 나중 문제다.
- 정리: 정합성 모델(Cache Aside, Write Through, Write Behind)을 초기에 고르지 않으면 운영 중 버그가 폭발한다.

- 오해 4: 분산 락을 걸었으니 안전하다.
- 정리: 락 타임아웃, 안전 해제, 재시도 정책, fencing token까지 고려해야 진짜 안전하다.

## 8. 내가 추천하는 학습 순서

1. Redis CLI로 자료구조별 명령과 시간복잡도 체감
2. Spring `RedisTemplate` + `RedisCacheManager`로 캐시 정책 코드화
3. 스탬피드/핫키/Big Key를 로컬 부하 테스트로 재현
4. replication/cluster 제약과 장애 시나리오 문서화
5. Stream 소비자 복구, 분산 락 안전 해제, 지표 알람까지 확장

---

## 공식 문서 (심화 학습 순서)
- Redis Data Types: https://redis.io/docs/latest/develop/data-types/
- Redis Commands + Complexity: https://redis.io/docs/latest/commands/
- Redis Key Eviction: https://redis.io/docs/latest/develop/reference/eviction/
- Redis Expiration: https://redis.io/docs/latest/commands/expire/
- Redis Persistence (RDB/AOF): https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Redis Replication: https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
- Redis Cluster Specification: https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/
- Redis Distributed Locks (Redlock 포함): https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
- Redis Latency troubleshooting: https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- Spring Data Redis Reference: https://docs.spring.io/spring-data/redis/reference/redis.html
- Spring Boot Caching: https://docs.spring.io/spring-boot/reference/io/caching.html

이 문서를 반복해서 읽을 때는 "명령"보다 "운영 질문"으로 읽는 것이 좋았다.
- 이 데이터는 유실 허용 가능한가?
- TTL/무효화의 기준은 무엇인가?
- 장애 시 어떤 정합성까지 보장해야 하는가?

이 세 질문에 답할 수 있으면, Redis를 단순 캐시가 아니라 서비스 아키텍처 구성요소로 다룰 수 있다.
