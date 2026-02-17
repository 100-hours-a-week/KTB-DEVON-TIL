# _7 메모리 정책 미스(eviction thrash)

## 문제 정의
TTL/eviction 정책이 데이터 온도와 맞지 않으면 캐시 churn이 심해지고 hit ratio가 하락한다.

## 재현 조건
- hot/cold 키 구분 없이 동일 TTL 적용
- maxmemory 임계 근처에서 지속 트래픽 인가
- eviction 이벤트 증가 구간 관찰

## 개선 전략
- hot/cold key 공간 분리
- latest/history TTL 차등 적용
- `allkeys-lfu` 검토 및 `maxmemory` 튜닝

## 검증 지표
- `evicted_keys`
- hit ratio
- miss penalty(latency, DB fallback)

## 포트폴리오 포인트
성능 최적화와 비용 최적화를 동시에 다룬 사례로 확장성 있는 운영 감각을 보여줄 수 있다.
