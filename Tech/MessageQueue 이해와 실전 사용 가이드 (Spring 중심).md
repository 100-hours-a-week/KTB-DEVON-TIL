# Message Queue 이해와 실전 사용 가이드 (Spring 중심, 공부 노트 버전)

이 문서는 "MQ를 써야 한다" 수준이 아니라, 실제로 운영에서 깨지는 지점을 기준으로 정리한 학습 노트다. 
나도 처음에는 "비동기로 보내면 빨라진다" 정도로 이해했는데, 실무에서는 속도보다 **실패를 어떻게 다루는지**가 핵심이었다. 
특히 아래 질문에 답할 수 있어야 MQ를 제대로 쓴다고 느꼈다.

1. 메시지가 중복으로 오면 내 서비스는 안전한가?
2. 소비자가 죽었을 때 어디까지 자동 복구되고, 어디서 사람이 개입해야 하는가?
3. 순서를 보장해야 하는 데이터는 어떤 키 전략으로 파티셔닝했는가?
4. 장애가 났을 때 "버그"인지 "설계상 의도"인지 바로 구분할 수 있는 지표가 있는가?

---

## 1. Message Queue를 왜 쓰는가 (정확한 목적)

MQ는 단순히 성능 최적화 도구가 아니다. 내가 이해한 핵심 목적은 아래 4가지다.

- 결합도 감소: 호출자가 처리자 상태를 몰라도 된다.
- 피크 흡수: 순간 트래픽 급증을 큐 적체로 흡수하고, 워커가 처리 가능한 속도로 소화한다.
- 장애 격리: 하류 서비스 장애가 상류 전체 장애로 바로 전파되지 않는다.
- 재처리 가능성: 실패 메시지를 추적하고 재실행할 수 있다.

동기 호출은 "지금 성공/실패"가 중요하고, MQ는 "결국 처리되었는가"가 더 중요하다. 
따라서 MQ를 도입하면 API 설계, 에러 모델, 모니터링, 운영 절차(runbook)까지 같이 바뀌어야 한다.

---

## 2. 꼭 알아야 하는 보장 모델 (그리고 오해)

### 2.1 전달 보장
- At most once: 중복은 적지만 유실 가능
- At least once: 유실은 줄지만 중복 가능
- Exactly once: 브로커 내부 일부 구간에서만 제한적으로 가능하고, 서비스 전체(end-to-end) 보장은 매우 어려움

### 2.2 실무에서의 현실적 결론
대부분 시스템은 아래 조합으로 간다.
1. 브로커는 at-least-once
2. 애플리케이션은 멱등 처리
3. 실패 메시지는 DLQ 격리 + 재처리 도구 제공

내가 처음 헷갈렸던 포인트는 "Kafka EOS면 다 끝난 것 아닌가?"였다. 
하지만 외부 DB 업데이트, 외부 API 호출이 섞이면 결국 중복/재시도 문제는 애플리케이션에서 다시 해결해야 했다.

---

## 3. RabbitMQ와 Kafka를 어떻게 구분해서 볼까

### 3.1 RabbitMQ 관점
- Exchange + Binding으로 라우팅 유연성 높음
- Ack/Nack, DLX, TTL, 우선순위 큐 등 작업 큐 운영 기능이 강함
- 다중 워커 기반 "작업 처리"에 강함

### 3.2 Kafka 관점
- 파티션 로그에 append, offset 기반 재처리
- 대용량 이벤트 스트림/분석 파이프라인에 강함
- consumer group을 통한 확장성과 replay가 장점

### 3.3 선택 기준 (내 기준표)
- "작업 분배 + 실패 라우팅" 중심: RabbitMQ
- "이벤트 로그 + 재생 + 장기 데이터 흐름" 중심: Kafka
- 둘 다 필요하면 혼용도 가능: 업무 이벤트는 Kafka, 단기 작업 큐는 RabbitMQ

---

## 4. 순서 보장과 확장성의 트레이드오프

이 부분이 실무에서 가장 자주 충돌한다.

- 순서를 강하게 보장하려면 병렬성이 줄어든다.
- 병렬성을 높이면 전역 순서는 포기해야 한다.

### 4.1 Kafka
- 순서는 "파티션 내부"만 보장
- 같은 aggregate(예: orderId)는 같은 파티션 키로 보내야 함
- 잘못된 키 전략이면 순서 버그가 간헐적으로 발생

### 4.2 RabbitMQ
- 단일 consumer일 때는 체감 순서가 비교적 명확
- 여러 consumer + requeue + prefetch가 섞이면 처리 완료 순서는 흔들릴 수 있음

**학습 체크 포인트**
- "순서 보장 범위"를 문서에 명시했는가? (전역/엔티티 단위/없음)
- 순서가 필요한 이벤트와 필요 없는 이벤트를 분리했는가?

---

## 5. 재시도 설계: 무한 재시도는 장애 증폭기다

처음에는 실패하면 재시도하면 된다고 생각했는데, 실제로는 독성 메시지(poison message) 때문에 큐가 마비되기 쉽다.

권장 패턴:
1. 즉시 재시도(짧게 1~3회)
2. 지수 백오프 재시도(예: 5s, 30s, 2m)
3. 한계 초과 시 DLQ 이동
4. DLQ는 원인 분류(데이터 오류/일시 장애/코드 버그) 후 재처리

핵심은 "같은 메시지를 같은 방식으로 무한 반복"하지 않는 것이다.

---

## 6. 멱등성(idempotency): MQ 실무의 생명줄

중복 수신은 예외가 아니라 정상 상황으로 봐야 한다.

### 6.1 멱등 키 전략
- eventId(전역 유니크 UUID)
- 또는 aggregateId + version

### 6.2 저장 전략 예시
- `processed_event` 테이블에 unique key(event_id)
- 소비 시 insert 시도
- 이미 존재하면 중복으로 판단하고 비즈니스 처리 skip

```sql
CREATE TABLE processed_event (
    event_id VARCHAR(64) PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL
);
```

이 패턴은 "한 번만 처리"를 기술적으로 강제할 수 있어 실무에서 효과가 컸다.

---

## 7. DB 트랜잭션과 메시지 발행 원자성: Outbox 패턴

가장 위험한 구간은 "DB는 성공했는데 메시지 발행 실패"(또는 반대)다.
이를 줄이기 위해 Outbox 패턴을 사용한다.

1. 비즈니스 데이터 + outbox 레코드를 같은 DB 트랜잭션으로 저장
2. 별도 퍼블리셔(스케줄러/CDC)가 outbox를 읽어 MQ로 발행
3. 발행 성공 시 outbox 상태를 SENT로 변경

### 7.1 Outbox를 더 안정적으로 운영하는 법
- 폴링 방식: 단순하지만 지연/중복 제어 필요
- CDC(Debezium 등): 변경 로그 기반으로 안정적 전송 가능

---

## 8. Spring 실전 예시 코드

아래 코드는 "돌아가기만 하는" 예시가 아니라, 장애 대응 포인트를 의식해서 작성했다.

### 8.1 RabbitMQ 설정 (DLQ + 재시도 기본)
```java
@Configuration
public class RabbitConfig {

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order.exchange");
    }

    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("order.created.queue")
                .withArgument("x-dead-letter-exchange", "order.dlx")
                .withArgument("x-dead-letter-routing-key", "order.created.dlq")
                .build();
    }

    @Bean
    public TopicExchange deadLetterExchange() {
        return new TopicExchange("order.dlx");
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable("order.created.dlq").build();
    }

    @Bean
    public Binding orderBinding() {
        return BindingBuilder.bind(orderQueue())
                .to(orderExchange())
                .with("order.created");
    }

    @Bean
    public Binding dlqBinding() {
        return BindingBuilder.bind(deadLetterQueue())
                .to(deadLetterExchange())
                .with("order.created.dlq");
    }
}
```

### 8.2 RabbitMQ Consumer (수동 ACK + 멱등 처리)
```java
@Component
@RequiredArgsConstructor
public class OrderCreatedConsumer {

    private final ProcessedEventRepository processedEventRepository;
    private final OrderDomainService orderDomainService;

    @RabbitListener(queues = "order.created.queue", ackMode = "MANUAL")
    public void consume(OrderCreatedEvent event,
                        Channel channel,
                        @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {
        try {
            if (processedEventRepository.existsById(event.eventId())) {
                channel.basicAck(tag, false);
                return;
            }

            orderDomainService.apply(event);
            processedEventRepository.save(new ProcessedEvent(event.eventId()));

            channel.basicAck(tag, false);
        } catch (RecoverableBusinessException ex) {
            // 일시 오류: 재큐잉 전략 선택 가능
            channel.basicNack(tag, false, true);
        } catch (Exception ex) {
            // 비가역 오류: DLQ로 격리
            channel.basicNack(tag, false, false);
        }
    }
}
```

### 8.3 Kafka Producer/Consumer (키 기반 순서 보장)
```java
@Service
@RequiredArgsConstructor
public class PaymentEventPublisher {

    private final KafkaTemplate<String, PaymentCompletedEvent> kafkaTemplate;

    public void publish(PaymentCompletedEvent event) {
        // 같은 orderId를 같은 파티션으로 보내 엔티티 단위 순서 보장
        kafkaTemplate.send("payment.completed", event.orderId(), event);
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class PaymentCompletedConsumer {

    private final ProcessedEventRepository processedEventRepository;
    private final PaymentService paymentService;

    @KafkaListener(topics = "payment.completed", groupId = "billing-service")
    public void consume(PaymentCompletedEvent event, Acknowledgment ack) {
        if (processedEventRepository.existsById(event.eventId())) {
            ack.acknowledge();
            return;
        }

        paymentService.apply(event);
        processedEventRepository.save(new ProcessedEvent(event.eventId()));

        ack.acknowledge();
    }
}
```

### 8.4 Spring Boot 설정 예시 (핵심만)
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 50
        retry:
          enabled: true
          max-attempts: 3
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      enable-auto-commit: false
      auto-offset-reset: earliest
      max-poll-records: 200
    producer:
      acks: all
      retries: 10
      properties:
        enable.idempotence: true
```

---

## 9. 성능/안정성 튜닝 포인트 (현업에서 자주 보는 문제)

### 9.1 RabbitMQ
- prefetch 과대 설정: 처리 지연 + 메모리 사용량 급증
- unacked 메시지 증가: 소비자 병목 신호
- queue depth 지속 증가: 처리량 부족 또는 장애 전조

### 9.2 Kafka
- consumer lag 증가: 소비 속도 < 유입 속도
- 리밸런싱 빈발: poll/처리 시간 설정 불균형 가능성
- 파티션 수 부족: 확장 한계

### 9.3 공통
- 큰 메시지 payload: 네트워크/직렬화 비용 증가
- 스키마 변경 무계획: 소비자 호환성 깨짐
- DLQ 모니터링 없음: 장애를 늦게 발견

---

## 10. 관측(Observability) 지표와 경보 기준 예시

내가 정리한 최소 운영 지표는 아래다.

1. 적체: queue depth 또는 consumer lag
2. 처리율: messages/sec in/out
3. 실패율: retry 비율, DLQ 유입량
4. 지연: enqueue-to-process latency
5. 건강도: broker CPU/memory/disk, connection/channel 수

경보 예시:
- lag가 10분 이상 지속 증가
- DLQ 유입량이 평시 대비 3배 이상 급증
- 처리 지연 P95가 SLO 초과 5분 지속

---

## 11. 보안과 거버넌스(놓치기 쉬운 포인트)

- 메시지에 PII(개인정보) 포함 여부를 먼저 분류
- 토픽/큐 ACL 최소 권한 적용
- 암호화(전송 TLS, 저장 암호화)와 키 관리 분리
- 감사 로그: 누가 어떤 메시지를 재처리했는지 추적 가능해야 함

---

## 12. 공부할 때 직접 해보면 좋은 실험

1. 동일 메시지를 2번 발행해서 멱등 처리 검증
2. Consumer 강제 종료 후 재기동하여 중복/유실 여부 확인
3. poison message 주입 후 retry -> DLQ 전환 확인
4. prefetch, max.poll.records 값을 바꿔 처리량/지연 곡선 비교
5. Outbox 적용 전/후로 정합성 실패 케이스 비교

이 실험을 해보면 "문서로만 이해"한 것과 실제 운영 감각의 차이가 크게 줄어든다.

---

## 13. 최종 요약 (내 기준)

MQ를 잘 쓴다는 것은 브로커를 잘 다루는 것이 아니라, **실패를 설계하는 것**이다.
- 중복은 발생한다고 가정하고 멱등으로 처리
- 재시도는 단계화하고 독성 메시지는 격리
- 순서 보장 범위를 명확히 제한
- 지표/경보/runbook으로 운영 가능하게 만든다

이 네 가지를 만족하면 MQ는 "비동기 기술"에서 "신뢰성 아키텍처"로 바뀐다.

---

## 공식 문서 (2026-02-17 기준)
- RabbitMQ Tutorials: https://www.rabbitmq.com/tutorials
- RabbitMQ Docs: https://www.rabbitmq.com/docs
- RabbitMQ Reliability Guide: https://www.rabbitmq.com/docs/reliability
- RabbitMQ Consumer Acks / Publisher Confirms: https://www.rabbitmq.com/docs/4.0/confirms
- RabbitMQ Dead Letter Exchanges: https://www.rabbitmq.com/docs/4.0/dlx
- Apache Kafka Documentation: https://kafka.apache.org/documentation/
- Apache Kafka 4.1 Docs Index: https://kafka.apache.org/41/
- Spring Boot Messaging: https://docs.spring.io/spring-boot/reference/messaging/index.html
- Spring Boot RabbitMQ: https://docs.spring.io/spring-boot/reference/messaging/amqp.html
- Spring Boot Kafka: https://docs.spring.io/spring-boot/reference/messaging/kafka.html
- Spring AMQP Reference: https://docs.spring.io/spring-amqp/reference/reference.html
- Spring for Apache Kafka Reference: https://docs.spring.io/spring-kafka/reference/
- Spring Framework JMS: https://docs.spring.io/spring-framework/reference/integration/jms.html
- Amazon SQS Developer Guide: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- Amazon SQS API Reference: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/Welcome.html

