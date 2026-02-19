# Spring Kafka 기본 흐름

## 정의
Spring Kafka는 Kafka Producer/Consumer를 Spring 애플리케이션에서 선언적으로 구성하고 운영할 수 있게 해주는 프레임워크입니다.
`KafkaTemplate`와 `@KafkaListener`가 핵심 진입점입니다.

## 특징
- Producer는 `KafkaTemplate`로 메시지 발행을 단순화합니다.
- Consumer는 `@KafkaListener`로 구독/처리를 선언합니다.
- 에러 핸들러, 재시도, DLT 구성을 프레임워크 레벨에서 지원합니다.

## 실제 사용 예시(사례)
- 주문 생성 시 `order-created` 이벤트를 KafkaTemplate으로 발행합니다.
- 소비 실패 메시지는 DLT로 보내 후속 복구 워크플로우를 분리합니다.

## 보조 개념 정리
- DLT: 처리 실패 메시지를 격리 저장하는 토픽
- 컨테이너 팩토리: 리스너 동작 정책(역직렬화, 에러 핸들러) 구성 단위
- 배치 리스너: 여러 레코드를 묶어 처리하는 소비 방식
