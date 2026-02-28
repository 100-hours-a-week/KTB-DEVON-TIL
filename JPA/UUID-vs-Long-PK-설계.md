# UUID vs Long PK 설계 전략

> Devon의 PK 타입 선택 기준과 인덱스 관점의 성능 분석

## 들어가며

PK를 UUID로 할지 Long으로 할지는 단순한 타입 선택이 아니다.
저장 구조, 인덱스 성능, 분산 환경 대응 능력이 모두 달라진다.

실무에서는 "UUID가 보안에 좋다"는 이유만으로 UUID를 선택하는 경우를 종종 본다.
그런데 인덱스 관점을 이해하고 나면 이 선택이 얼마나 큰 성능 차이를 만드는지 알게 된다.

주요 비교 포인트:
- **저장 구조**: 물리적 크기와 클러스터드 인덱스와의 관계
- **B-Tree 인덱스**: 순차 vs 랜덤 삽입이 인덱스에 미치는 영향
- **세컨더리 인덱스**: PK 크기가 모든 인덱스에 미치는 파급 효과
- **UUID v7**: 두 방식의 장점을 모두 취하는 현실적 대안

---

## 1. 기본 비교

| 항목 | Long (BIGINT) | UUID v4 (VARCHAR 36) | UUID (BINARY 16) |
|------|--------------|---------------------|-----------------|
| 저장 크기 | 8 bytes | 36 bytes | 16 bytes |
| 생성 위치 | DB (AUTO_INCREMENT) | 애플리케이션 | 애플리케이션 |
| 순차성 | 단조 증가 | 완전 랜덤 | 완전 랜덤 |
| 전역 유일성 | DB 범위 내 | 전역 유일 | 전역 유일 |
| 충돌 가능성 | 없음 (DB 보장) | 이론상 있음 (현실적으로 무시) | 이론상 있음 |

---

## 2. 클러스터드 인덱스와 물리적 저장 구조

### InnoDB의 클러스터드 인덱스

MySQL InnoDB는 **PK 자체가 클러스터드 인덱스**다.
테이블 데이터가 PK 순서로 물리적으로 정렬되어 저장된다.

```
[클러스터드 인덱스 = 실제 데이터 파일]

PK=1 → { id:1, title:"foo", content:"..." }
PK=2 → { id:2, title:"bar", content:"..." }
PK=3 → { id:3, title:"baz", content:"..." }
```

이 구조 때문에 PK의 순차성이 삽입 성능에 직접적인 영향을 미친다.

### Long (순차 삽입)

```
Before:  [1] [2] [3] [4] [5]

INSERT id=6:
After:   [1] [2] [3] [4] [5] [6]  ← 항상 마지막 페이지에 추가
```

새 레코드는 항상 인덱스의 가장 오른쪽 리프 노드에 추가된다.
**페이지 분할이 거의 발생하지 않는다.** 마지막 페이지가 꽉 찰 때만 분할한다.

### UUID v4 (랜덤 삽입)

```
Before:  [1aef...] [3bc2...] [7d4a...] [f2e1...]
                       ↑
INSERT id=4c88...: 중간 어딘가에 삽입해야 함
After:   [1aef...] [3bc2...] [4c88...] [7d4a...] [f2e1...]
                   ← 페이지 분할 발생 →
```

UUID v4는 완전 랜덤이기 때문에 새 레코드가 인덱스의 임의 위치에 삽입된다.
해당 위치의 페이지가 꽉 찼다면 **페이지 분할(Page Split)**이 발생한다.

---

## 3. 페이지 분할의 비용

### 페이지 분할이란

B-Tree의 노드(InnoDB 기본 16KB 페이지)가 꽉 찬 상태에서 새 데이터를 삽입해야 할 때, 페이지를 둘로 나누는 작업이다.

```
분할 전 (100% 사용):
[A][B][C][D][E][F][G][H]  (꽉 참)

C와 D 사이에 X 삽입 시 분할:
[A][B][C][X]   +   [D][E][F][G][H]  (각각 약 50% 사용)
```

### 분할의 비용

1. **쓰기 증폭**: 1번 INSERT가 2개 페이지 쓰기를 유발
2. **페이지 단편화**: 분할 후 각 페이지가 50% 수준으로 채워짐 → 같은 데이터를 저장하는 데 더 많은 페이지 필요
3. **읽기 성능 저하**: 데이터가 여러 페이지에 흩어져 순차 읽기가 랜덤 I/O로 변환
4. **잠금 경합**: 페이지 분할 중 해당 영역에 잠금 발생

### 실제 페이지 사용률 차이

장기간 운영 시 대략적인 페이지 사용률 차이다.

| PK 타입 | 페이지 사용률 | 동일 데이터 저장에 필요한 페이지 수 |
|---------|-------------|-------------------------------|
| Long (순차) | ~95% | 기준 (1x) |
| UUID v4 (랜덤) | ~50-70% | 1.4~2x |

페이지 수가 늘어나면 캐시(InnoDB Buffer Pool) 효율이 떨어지고, 디스크 I/O가 증가한다.

---

## 4. 세컨더리 인덱스에 미치는 영향

### InnoDB 세컨더리 인덱스의 구조

InnoDB에서 모든 세컨더리 인덱스는 실제 데이터 행을 직접 가리키지 않는다.
대신 **PK 값을 저장**하고, PK를 통해 클러스터드 인덱스를 한 번 더 조회한다.

```
세컨더리 인덱스 (title 컬럼):
[title="bar"]  → PK=2   ← PK 값을 저장
[title="baz"]  → PK=3
[title="foo"]  → PK=1

조회 시: title 인덱스로 PK를 찾고 → PK로 클러스터드 인덱스에서 실제 행을 조회
```

### PK 크기의 파급 효과

PK가 크면 모든 세컨더리 인덱스 엔트리도 그만큼 커진다.

```
// Long PK (8 bytes)
세컨더리 인덱스 1개 엔트리:  [컬럼값 + 8 bytes]

// UUID VARCHAR(36) PK (36 bytes)
세컨더리 인덱스 1개 엔트리:  [컬럼값 + 36 bytes]  → 4.5배 증가
```

테이블에 세컨더리 인덱스가 5개 있다면, PK 크기 차이가 5배로 증폭된다.
인덱스가 메모리(Buffer Pool)에 올라가는 비율이 줄어들고, 캐시 효율이 낮아진다.

### 조회 시 추가 I/O

```java
// title로 조회하는 쿼리
SELECT * FROM post WHERE title = 'foo';

// Long PK: title 인덱스 → PK(8B) 조회 → 클러스터드 인덱스 탐색
// UUID PK: title 인덱스 → PK(36B) 조회 → 클러스터드 인덱스 탐색 (랜덤 위치)
```

UUID는 클러스터드 인덱스에서도 랜덤 위치를 탐색하기 때문에 캐시 히트율이 낮다.

---

## 5. UUID v7 — 현실적인 대안

### UUID v4의 문제를 해결한 시간 정렬 UUID

UUID v7은 타임스탬프를 앞에 배치해서 **단조 증가하는 UUID**다.

```
UUID v4 (완전 랜덤):
550e8400-e29b-41d4-a716-446655440000
^^^^^^^^ 랜덤

UUID v7 (시간 정렬):
018e9b3d-4f2a-7000-8000-000000000001
^^^^^^^^ 밀리초 타임스탬프 (단조 증가)
```

같은 밀리초 내에 생성된 UUID도 순서가 보장된다.

### UUID v7의 인덱스 동작

```
순차적으로 증가하는 UUID v7:
018e9b3d-0001 → [데이터]
018e9b3d-0002 → [데이터]  ← 항상 뒤에 추가
018e9b3d-0003 → [데이터]
```

Long처럼 항상 마지막 리프 노드에 추가되므로 페이지 분할이 최소화된다.

### Java에서 UUID v7 사용

Java 표준 라이브러리(Java 21까지)에는 UUID v7이 없다.
외부 라이브러리를 사용한다.

```gradle
implementation 'com.github.f4b6a3:uuid-creator:5.3.7'
```

```java
import com.github.f4b6a3.uuid.UuidCreator;

@Entity
public class Post {

    @Id
    private UUID id;

    @PrePersist
    public void generateId() {
        if (id == null) {
            id = UuidCreator.getTimeOrderedEpoch();  // UUID v7
        }
    }
}
```

---

## 6. UUID를 써야 하는 상황

UUID의 인덱스 비용을 감수할 만한 상황이 있다.

### 분산 환경에서의 ID 생성

여러 서버 또는 여러 DB 인스턴스가 ID를 독립적으로 생성해야 할 때다.

```
서버 A: INSERT id=? → DB 시퀀스 없이 UUID 생성 → 충돌 없이 저장
서버 B: INSERT id=? → DB 시퀀스 없이 UUID 생성 → 충돌 없이 저장

// Long SEQUENCE/IDENTITY는 단일 DB에 의존
// DB 분리(샤딩, 멀티 마스터) 시 충돌 가능성 있음
```

### DB INSERT 전에 ID가 필요한 경우

이벤트 소싱, 사가 패턴 등에서 INSERT 전에 ID를 애플리케이션에서 미리 알아야 할 때다.

```java
// 주문 생성 전에 orderId를 먼저 생성하고
// 여러 시스템에 전파한 뒤 실제 저장
String orderId = UUID.randomUUID().toString();
eventBus.publish(new OrderCreatedEvent(orderId, ...));
orderRepository.save(new Order(orderId, ...));
```

Long IDENTITY는 INSERT 후에야 ID를 알 수 있어서 이 패턴이 불가능하다.

### 순차 ID 노출이 보안 이슈인 경우

```
// Long: URL에서 순차 노출
GET /api/posts/1
GET /api/posts/2   ← 다음 리소스 쉽게 추측 가능

// UUID: 추측 불가
GET /api/posts/550e8400-e29b-41d4-a716-446655440000
```

단, 이것만을 위해 UUID를 사용한다면 ULID나 Hashids 같은 대안도 고려할 수 있다.

---

## 7. UUID를 사용할 때 BINARY(16) 권장

UUID를 VARCHAR(36)으로 저장하면 36 bytes, BINARY(16)으로 저장하면 16 bytes다.
인덱스 크기와 비교 연산 속도 모두 BINARY(16)이 유리하다.

```java
@Entity
public class Post {

    @Id
    @Column(columnDefinition = "BINARY(16)")
    private UUID id;

    @PrePersist
    public void generateId() {
        if (id == null) {
            id = UuidCreator.getTimeOrderedEpoch();
        }
    }
}
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        type:
          preferred_uuid_jdbc_type: BINARY
```

---

## 선택 기준

```
단일 DB, 높은 쓰기 성능이 중요한가?
  → Long (BIGINT IDENTITY/SEQUENCE)

분산 시스템, 또는 INSERT 전 ID가 필요한가?
  → UUID v7 (시간 정렬) + BINARY(16)

UUID v4는 언제 쓰나?
  → 성능보다 전역 고유성과 비예측성이 더 중요한 경우
  → 쓰기 빈도가 낮아 인덱스 단편화가 문제되지 않는 경우
```

---

## 핵심 원칙 정리

### Long의 강점
순차 삽입으로 B-Tree 페이지 분할 최소화, 8 bytes로 세컨더리 인덱스 크기 최소화. 단일 DB 환경에서 성능 최우선이면 Long이 맞다.

### UUID v4의 약점
랜덤 삽입 → 빈번한 페이지 분할 → 단편화 → I/O 증가. 세컨더리 인덱스도 36 bytes PK를 저장해 전체적으로 인덱스가 비대해진다.

### UUID v7의 포지션
시간 정렬로 Long에 가까운 인덱스 성능 + 전역 고유성 보장. 분산 환경에서 UUID가 필요하다면 v4 대신 v7을 선택한다.

### PK 크기의 파급
PK는 모든 세컨더리 인덱스에 포함된다. PK 크기 차이(8 vs 36 bytes)는 인덱스 수만큼 곱해진다.

---

## 마치며

UUID를 쓸 것인가 Long을 쓸 것인가는 결국 **시스템 특성의 문제**다.

단일 DB, 높은 쓰기 부하 → Long이 압도적으로 유리하다.
분산 환경, 사전 ID 생성 필요 → UUID v7 + BINARY(16)을 선택한다.

중요한 건 UUID를 아무 생각 없이 기본값으로 선택하지 않는 것이다.
인덱스 단편화와 세컨더리 인덱스 비대화는 데이터가 쌓일수록 천천히 성능을 갉아먹는다.

선택할 때 항상 "이 시스템이 분산 ID 생성을 필요로 하는가"를 먼저 물어보자.

---

## 참고자료

- [MySQL InnoDB - Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)
- [RFC 9562 - UUID Version 7](https://www.rfc-editor.org/rfc/rfc9562#name-uuid-version-7)
- [uuid-creator GitHub](https://github.com/f4b6a3/uuid-creator)
