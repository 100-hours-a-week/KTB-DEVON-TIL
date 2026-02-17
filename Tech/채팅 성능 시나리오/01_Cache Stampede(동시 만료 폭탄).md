# _1 Cache Stampede(동시 만료 폭탄)

## 문제 정의
동일 TTL로 키를 발급하면 특정 시점에 다수 키가 동시에 만료되어, Redis miss가 급증하고 DB/DynamoDB read burst가 발생한다.

## 재현 조건
- 동일 TTL(예: 300초)로 대량 키 생성
- 만료 시점 직전에 VU를 집중시켜 조회 트래픽 인가
- 워크로드: latest 조회 70%, history 조회 20%, write 10%

## 개선 전략
- TTL jitter: TTL에 랜덤 분산 추가
- single-flight: 동일 키 miss 동시 요청을 1회 조회로 수렴
- stale-while-revalidate: 짧은 stale 허용으로 급격한 miss 완화

## 검증 지표
- `DB read burst` 최대치
- `P99` 및 timeout 비율
- miss storm 구간의 `keyspace_misses`

## 포트폴리오 포인트
평시 정상인데 피크에서만 무너지는 실전형 장애를 재현하고, tail latency 안정화로 운영 품질을 개선한 사례를 보여줄 수 있다.
