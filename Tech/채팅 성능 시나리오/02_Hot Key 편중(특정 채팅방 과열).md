# _2 Hot Key 편중(특정 채팅방 과열)

## 문제 정의
인기 대화방 하나에 트래픽이 집중되면 Redis 단일 키가 과열되어 전체 지연을 끌어올린다.

## 재현 조건
- 특정 `conversationId`에 요청 50% 이상 집중
- 나머지 트래픽은 다수 룸으로 분산
- 피크 단계에서 동일 hot key 반복 조회

## 개선 전략
- hot key 분산: shard suffix(`:s1`, `:s2`) 적용
- 로컬 1차 캐시(Caffeine)로 Redis 호출 흡수
- fan-out 비동기화로 read/write 경합 축소

## 검증 지표
- key 단위 `command/sec`
- Redis CPU, `P95/P99`
- hot key 구간 에러/timeout 비율

## 포트폴리오 포인트
평균 지표는 정상인데 tail latency만 붕괴하는 패턴을 분석해낸 고급 운영 사례로 어필 가능하다.
