# Exactly Once Semantics

## 정의
Exactly Once Semantics(EOS)는 중복 처리 없이 메시지를 한 번만 처리한 것처럼 보장하는 Kafka 처리 모델입니다.
Producer 트랜잭션과 idempotence, 소비-생산 경계 관리가 함께 필요합니다.

## 특징
- 단순 설정 하나로 끝나지 않고 처리 파이프라인 전반 설계가 필요합니다.
- Kafka Streams/트랜잭션 프로듀서에서 주로 적용합니다.
- 외부 DB까지 포함한 완전한 exactly-once는 추가 설계(아웃박스 등)가 필요합니다.

## 실제 사용 예시(사례)
- 집계 파이프라인에서 consume -> transform -> produce를 트랜잭션으로 묶어 중복 집계를 방지합니다.
- 외부 저장소 반영은 idempotency key로 중복 반영을 방지합니다.

## 보조 개념 정리
- Transactional ID: 트랜잭션 프로듀서 식별자
- Read Committed: 커밋된 메시지만 읽는 소비 모드
- Outbox Pattern: DB 트랜잭션과 이벤트 발행 간 정합성 패턴
