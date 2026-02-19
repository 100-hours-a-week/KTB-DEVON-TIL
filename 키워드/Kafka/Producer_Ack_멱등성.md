# Producer Ack와 멱등성

## 정의
Producer Ack는 브로커가 메시지 저장을 확인하는 수준이고, 멱등성(idempotent producer)은 재시도 시 중복 저장을 줄이는 기능입니다.
내구성과 지연 시간의 균형을 맞추는 핵심 설정입니다.

## 특징
- `acks=0`: 가장 빠르지만 유실 위험이 큽니다.
- `acks=1`: 리더 저장 확인, 일부 장애에서 유실 가능성이 남습니다.
- `acks=all`: ISR 복제 확인으로 내구성이 가장 높습니다.
- 멱등성 활성화 시 네트워크 재시도 중 중복 전송 위험이 감소합니다.

## 실제 사용 예시(사례)
- 결제/주문 이벤트는 `acks=all`, `enable.idempotence=true`로 설정합니다.
- 채팅 이벤트는 지연 민감도에 따라 `acks=1`을 선택하기도 합니다.

## 보조 개념 정리
- ISR: In-Sync Replica 집합
- Retry: 전송 실패 시 재시도 횟수
- Delivery Semantics: at-most-once, at-least-once, exactly-once 보장 수준
