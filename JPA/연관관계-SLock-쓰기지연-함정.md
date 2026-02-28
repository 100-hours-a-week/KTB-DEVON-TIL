# JPA 연관관계와 S Lock — 쓰기 지연의 함정

> Devon의 부모 엔티티 미저장 + InnoDB FK 검증 S Lock 문제 정리

## 들어가며

JPA의 쓰기 지연(Write-Behind)은 SQL 실행을 트랜잭션 커밋 시점까지 모아두는 편리한 기능이다.
그런데 연관관계가 있는 두 엔티티를 다룰 때, 이 편리함을 과신하면 예상치 못한 문제가 생긴다.

부모 엔티티를 먼저 명시적으로 저장하지 않고 자식 엔티티를 저장하려 할 때 발생하는 두 가지 문제다.

- **즉각적인 문제**: 부모 ID가 없으니 자식의 FK 컬럼이 null → `TransientPropertyValueException`
- **숨겨진 문제**: 올바르게 저장해도 InnoDB FK 검증 과정에서 부모 행에 **S Lock**이 걸리고, 이 락이 트랜잭션 내내 유지된다

---

## 1. JPA 쓰기 지연과 연관관계

### 쓰기 지연의 기본 동작

```java
@Transactional
public void createMemberAndOrder() {
    Member member = new Member("devon");
    // 이 시점: 1차 캐시에만 존재, DB에는 INSERT 안 됨

    Order order = new Order(member, "item");
    orderRepository.save(order); // flush 발생 시점
}
```

`orderRepository.save(order)` 호출 시 Hibernate가 flush를 수행한다.
flush 시점에 Hibernate는 `order`의 FK 컬럼에 `member.id`를 채워야 한다.

그런데 `member`는 아직 DB에 INSERT되지 않았으므로 `member.id = null`이다.

### 발생하는 예외

```
org.hibernate.TransientPropertyValueException:
object references an unsaved transient instance -
save the transient instance before flushing:
Order.member -> Member
```

Hibernate가 flush 직전에 이 상황을 감지하고 예외를 던진다.

### CascadeType.PERSIST가 있을 때

```java
@ManyToOne(fetch = FetchType.LAZY, cascade = CascadeType.PERSIST)
@JoinColumn(name = "member_id")
private Member member;
```

CASCADE.PERSIST가 있으면 예외는 발생하지 않는다.
Hibernate가 부모를 먼저 INSERT한 뒤 자식을 INSERT한다.

```
flush 순서:
1. INSERT member (성공, id=1 획득)
2. INSERT order (member_id=1, FK 검증 통과)
```

겉으로는 정상 동작하는 것처럼 보인다.
그런데 여기서 InnoDB 내부에서 벌어지는 일이 다음 문제의 원인이다.

---

## 2. InnoDB FK 검증과 S Lock

### InnoDB가 FK를 검증하는 방법

InnoDB는 자식 행을 INSERT할 때 FK 제약 조건을 검증하기 위해 **부모 행에 Shared Lock(S Lock)을 획득**한다.

```
INSERT order (member_id=1) 실행 시 InnoDB 내부:

1. member 테이블에서 id=1인 행을 탐색
2. 해당 행에 S Lock 획득 (FK 검증용)
3. 부모 행이 존재함을 확인
4. order INSERT 완료
5. S Lock → 트랜잭션 종료까지 유지 (REPEATABLE READ 기준)
```

이것은 MySQL 공식 문서에 명시된 InnoDB 동작이다.

> "If a FOREIGN KEY constraint is defined on a table, any insert, update, or delete that requires a constraint check sets shared record-level locks on the records that it looks at to check the constraint."

### S Lock의 지속 시간

MySQL 기본 격리 수준은 **REPEATABLE READ**다.
이 격리 수준에서 FK 검증으로 획득한 S Lock은 **트랜잭션이 커밋되거나 롤백될 때까지 유지**된다.

```
트랜잭션 시작
  │
  ├─ INSERT order (member_id=1)
  │    └─ S Lock on member(id=1) 획득 ← 여기서 획득
  │
  ├─ (이후 다른 작업들 ...)
  │
  ├─ 트랜잭션 커밋
  │    └─ S Lock 해제 ← 여기서야 해제
  │
트랜잭션 종료
```

---

## 3. S Lock이 만드는 동시성 문제

### S Lock과 X Lock의 호환성

| 요청 \ 보유 | S Lock | X Lock |
|------------|--------|--------|
| S Lock 요청 | 호환 (동시 허용) | 대기 |
| X Lock 요청 | 대기 | 대기 |

S Lock끼리는 서로 호환된다. 여러 트랜잭션이 동시에 같은 부모 행에 S Lock을 가질 수 있다.
하지만 X Lock과는 호환되지 않는다.

### 부모 행 UPDATE/DELETE 차단

```
TX-1 (주문 생성, 진행 중):
  INSERT order (member_id=1) → S Lock on member(id=1) 보유

TX-2 (회원 정보 수정, 동시 실행):
  UPDATE member SET name='new' WHERE id=1
  → X Lock 필요 → TX-1의 S Lock과 충돌 → TX-2 대기
```

TX-1이 트랜잭션을 길게 유지할수록, member(id=1)에 대한 UPDATE/DELETE가 그만큼 차단된다.

### 데드락 시나리오

두 트랜잭션이 서로 다른 부모를 INSERT하면서 교차 참조 자식을 INSERT할 때 데드락이 발생한다.

```
TX-1:
  1. INSERT member(id=1) → X Lock on member(id=1)
  2. INSERT order(member_id=2) → S Lock on member(id=2) 요청
     → TX-2가 X Lock 보유 중 → 대기

TX-2:
  1. INSERT member(id=2) → X Lock on member(id=2)
  2. INSERT order(member_id=1) → S Lock on member(id=1) 요청
     → TX-1이 X Lock 보유 중 → 대기

→ DEADLOCK
```

### 쓰기 지연이 이 문제를 악화시키는 이유

쓰기 지연을 사용하면 여러 INSERT가 flush 시점에 한꺼번에 나간다.
SQL 실행 순서가 코드 작성 순서와 다를 수 있고, 락 획득 시점을 예측하기 어려워진다.

명시적으로 부모를 먼저 저장하면:
- 부모 INSERT → X Lock 획득 → 즉시 커밋 가능한 상태
- 자식 INSERT → S Lock 획득

락 구간이 짧고 예측 가능하다.

쓰기 지연에 의존하면:
- 부모와 자식 INSERT가 같은 flush에 묶임
- S Lock이 트랜잭션 전체 구간에 걸림
- 다른 작업들이 락 범위 안에 포함될 가능성이 높아짐

---

## 4. 올바른 구현 방법

### 해결책 1: 부모를 먼저 명시적으로 save

```java
// 나쁜 예 — 쓰기 지연에 의존
@Transactional
public void createMemberAndOrder(OrderDto dto) {
    Member member = new Member(dto.memberName());
    // memberRepository.save() 호출 없음

    Order order = new Order(member, dto.item());
    orderRepository.save(order); // member.id = null → 예외
}

// 좋은 예 — 부모를 먼저 명시적으로 저장
@Transactional
public void createMemberAndOrder(OrderDto dto) {
    Member member = memberRepository.save(new Member(dto.memberName()));
    // IDENTITY 전략이면 즉시 INSERT 실행, id 획득

    Order order = new Order(member, dto.item());
    orderRepository.save(order); // member.id 존재 → FK 정상
}
```

### 해결책 2: 트랜잭션 범위를 최소화

S Lock은 트랜잭션이 길수록 오래 유지된다.
부모-자식 저장 후 다른 무거운 작업이 있다면, 저장 트랜잭션을 분리하는 것을 고려한다.

```java
// 나쁜 예 — S Lock이 오래 지속됨
@Transactional
public void createOrderAndSendEmail(OrderDto dto) {
    Member member = memberRepository.save(new Member(dto.memberName()));
    Order order = orderRepository.save(new Order(member, dto.item()));
    // 이 시점: member에 S Lock 걸려 있음

    emailService.sendWelcomeEmail(member.getEmail()); // 외부 API 호출 (수 초 소요 가능)
    // S Lock이 외부 API 완료까지 유지됨
}

// 좋은 예 — 저장과 외부 작업을 트랜잭션 분리
@Transactional
public OrderResult createOrder(OrderDto dto) {
    Member member = memberRepository.save(new Member(dto.memberName()));
    Order order = orderRepository.save(new Order(member, dto.item()));
    return new OrderResult(order.getId(), member.getEmail());
    // 트랜잭션 종료 → S Lock 해제
}

public void process(OrderDto dto) {
    OrderResult result = createOrder(dto); // 트랜잭션 종료됨
    emailService.sendWelcomeEmail(result.email()); // 락 없는 상태에서 외부 호출
}
```

### CASCADE를 남용하지 않는 이유

```java
// CASCADE.PERSIST: 명시적 save 없이도 동작하게 해주지만
@ManyToOne(cascade = CascadeType.PERSIST)
private Member member;

// 실제로는 Hibernate가 flush 순서를 결정
// → 개발자가 INSERT 순서를 통제하지 못함
// → 복잡한 연관관계에서 예상치 못한 순서로 INSERT 발생 가능
// → S Lock 획득 시점이 불명확해짐
```

부모-자식 저장은 CASCADE에 맡기지 않고, 명시적인 `save()` 호출 순서로 제어하는 것이 안전하다.

---

## 5. 전체 흐름 정리

```
[쓰기 지연에 의존할 때]

코드:
  new Member() → (flush 전까지 DB에 없음)
  new Order(member) → orderRepository.save()

flush 시점:
  └─ member.id = null
       └─ TransientPropertyValueException 발생
       └─ (CASCADE 있으면) Hibernate가 순서를 추정해서 실행

[명시적으로 부모 먼저 save할 때]

코드:
  memberRepository.save(member) → INSERT member → id=1 획득
  orderRepository.save(order)   → INSERT order(member_id=1)
                                    └─ InnoDB FK 검증
                                         └─ S Lock on member(id=1)

S Lock 유지 구간:
  트랜잭션 종료까지 유지 (REPEATABLE READ)

S Lock 영향:
  └─ 동기간 member UPDATE/DELETE → X Lock 필요 → 차단
  └─ 교차 참조 INSERT → 데드락 가능
```

---

## 핵심 원칙 정리

### 부모를 먼저 명시적으로 save한다
쓰기 지연은 편리하지만, 연관관계가 있는 엔티티의 저장 순서는 개발자가 명시적으로 통제해야 한다. Hibernate나 CASCADE에 순서를 맡기면 예외가 발생하거나 예측 불가한 동작이 생긴다.

### S Lock은 트랜잭션 종료까지 유지된다
REPEATABLE READ(MySQL 기본값)에서 FK 검증용 S Lock은 트랜잭션이 끝날 때까지 풀리지 않는다. 트랜잭션 내에 외부 API 호출, 무거운 연산이 있다면 Lock 지속 시간이 늘어난다.

### 트랜잭션 범위를 최소화한다
DB 저장 로직과 외부 호출(이메일, API)을 같은 트랜잭션에 묶지 않는다. 저장 트랜잭션을 먼저 커밋하고, 이후 외부 작업을 수행한다.

### CASCADE.PERSIST는 편의가 아닌 부채다
CASCADE는 개발자가 INSERT 순서를 직접 통제하지 못하게 만든다. 단순한 부모-자식 관계라면 명시적 save 순서가 훨씬 명확하고 안전하다.

---

## 마치며

JPA의 쓰기 지연은 SQL을 줄여주는 최적화 기능이지, 저장 순서를 자동으로 해결해주는 마법이 아니다.
연관관계가 있는 엔티티는 항상 부모를 먼저 save하는 것이 원칙이다.

그리고 자식 INSERT가 성공했다고 끝이 아니다.
InnoDB가 FK 검증을 위해 부모 행에 S Lock을 거는 순간부터, 그 트랜잭션이 살아있는 동안 부모 행은 부분적으로 잠긴 상태다.

쿼리 한 줄이 아니라 트랜잭션 전체의 락 흐름을 그려보는 습관이 중요한 이유다.

---

## 참고자료

- [MySQL InnoDB - Locks Set by Different SQL Statements](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)
- [MySQL InnoDB - Deadlocks in InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)
- [Hibernate ORM - Flushing](https://docs.jboss.org/hibernate/orm/6.0/userguide/html_single/Hibernate_User_Guide.html#flushing)
