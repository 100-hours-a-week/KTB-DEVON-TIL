# _4 Network RTT/커맨드 왕복 과다

## 문제 정의
요청 1건에 Redis 명령을 과도하게 호출하면 데이터 구조가 적절해도 네트워크 왕복이 병목이 된다.

## 재현 조건
- 요청 처리 중 `GET -> HGETALL -> EXPIRE` 다단 호출
- API 서버와 Redis 간 RTT가 작은 환경/큰 환경 모두 측정

## 개선 전략
- `pipeline`으로 명령 묶음 전송
- `MGET/HMGET`로 다건 조회 일괄화
- Lua 스크립트로 read-modify-write 원자화

## 검증 지표
- 앱-Redis RTT
- Redis `ops/sec` 대비 처리량
- 요청당 Redis 명령 수

## 포트폴리오 포인트
자료구조가 아니라 통신 패턴이 병목임을 분리해낸 사례로, 시스템 관점의 성능 분석 역량을 강조할 수 있다.
