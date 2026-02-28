# JPA Batch Insert 성능 함정과 해결 전략

> Devon의 saveAll vs save 루프, IDENTITY 전략의 배치 비활성화, merge SELECT 문제 정리

## 들어가며

JPA로 대량 데이터를 저장할 때 "그냥 saveAll() 쓰면 되지 않나?" 라고 생각하기 쉽다.
나도 처음에는 그렇게 생각했다.

그런데 실제로 쿼리 로그를 켜고 보면 예상과 전혀 다른 일이 벌어진다.
INSERT가 한 번에 묶이지 않고 N번 나가거나, 심지어 INSERT마다 SELECT가 선행되는 상황이 발생한다.

이 문서는 JPA 대량 저장 시 흔히 마주치는 세 가지 성능 함정과 각각의 해결 방법을 정리한 것이다.

- **트랜잭션 낭비**: `save()`를 루프로 호출할 때의 N번 커밋 문제
- **배치 비활성화**: `IDENTITY` 전략이 JDBC 배치를 원천 차단하는 문제
- **불필요한 SELECT**: ID가 있는 엔티티를 `save()`할 때 `merge()`가 호출되는 문제

---

## 1. save() 루프 vs saveAll() — 트랜잭션의 차이

### 무엇이 다른가

`saveAll()`의 구현체인 `SimpleJpaRepository`를 직접 보면 이해가 빠르다.

```java
// SimpleJpaRepository.java

@Transactional  // 여기
public <S extends T> List<S> saveAll(Iterable<S> entities) {
    List<S> result = new ArrayList<>();
    for (S entity : entities) {
        result.add(save(entity));  // 내부에서 save() 호출
    }
    return result;
}

@Transactional  // 여기도
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
        return entity;
    } else {
        return em.merge(entity);
    }
}
```

`saveAll()`과 `save()` 모두 `@Transactional`이 붙어 있다.
차이는 **트랜잭션 경계**다.

### 비(非)트랜잭션 컨텍스트에서의 차이

서비스 메서드에 `@Transactional`이 없는 상태에서 호출한다고 가정한다.

```java
// 나쁜 예 — N개의 트랜잭션, N번의 커밋
for (Post post : posts) {
    postRepository.save(post);  // 매번 새 트랜잭션 시작 → 커밋 → 종료
}

// 좋은 예 — 1개의 트랜잭션, 1번의 커밋
postRepository.saveAll(posts);  // saveAll 자체가 단일 트랜잭션으로 감쌈
```

`save()` 루프는 100개의 엔티티를 저장할 때 100번의 트랜잭션 시작과 커밋이 발생한다.
트랜잭션 커밋 자체가 비싼 연산이기 때문에 규모가 커질수록 차이가 극명해진다.

### @Transactional로 감싸면 동일한가?

```java
@Transactional  // 외부 트랜잭션 존재
public void saveAll(List<Post> posts) {
    for (Post post : posts) {
        postRepository.save(post);  // 기존 트랜잭션에 참여
    }
}
```

이 경우 `save()` 루프와 `saveAll()` 모두 단일 트랜잭션으로 동작한다.
**트랜잭션 관점에서는 동일**해진다.

다만 `saveAll()`이 의도를 더 명확하게 전달하기 때문에, 나는 항상 `saveAll()`을 사용한다.

---

## 2. IDENTITY 전략과 JDBC 배치 비활성화

### 진짜 문제의 시작

`saveAll()`로 바꾸고, `hibernate.jdbc.batch_size=100`도 설정했다.
그런데 여전히 INSERT가 100개씩 묶이지 않고 1개씩 나간다.

원인은 `GenerationType.IDENTITY` 전략이다.

```java
@Entity
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 여기가 문제
    private Long id;
}
```

### IDENTITY가 배치를 막는 이유

`IDENTITY` 전략은 DB의 `AUTO_INCREMENT`에 의존한다.
INSERT가 실행된 후에야 DB가 ID를 생성한다.

Hibernate는 엔티티 객체에 생성된 ID를 즉시 채워야 한다.
그래서 INSERT를 하나 실행하고 → `getGeneratedKeys()`로 ID를 즉시 가져온다.

이 구조 때문에 **INSERT를 모아서 한 번에 보낼 수 없다**.
`hibernate.jdbc.batch_size`를 아무리 크게 설정해도 IDENTITY 전략에서는 무효다.

```
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 100  # IDENTITY 전략에서는 INSERT에 효과 없음
```

### 해결책 1: SEQUENCE 전략으로 전환

`SEQUENCE` 전략은 INSERT 전에 시퀀스에서 ID를 미리 가져온다.
Hibernate가 ID를 미리 알고 있으므로 INSERT를 묶을 수 있다.

```java
@Entity
@SequenceGenerator(
    name = "post_seq",
    sequenceName = "post_sequence",
    allocationSize = 50  // 시퀀스를 50개 단위로 미리 할당
)
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "post_seq")
    private Long id;
}
```

`allocationSize = 50`은 DB 시퀀스 호출 횟수를 1/50으로 줄여준다.
50개의 ID를 미리 메모리에 예약하고, 소진되면 다음 블록을 가져온다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true   # 같은 엔티티 타입끼리 묶어서 배치
        order_updates: true
```

`order_inserts: true`는 여러 엔티티 타입이 섞인 경우 같은 타입끼리 모아서 배치 효율을 높인다.

### 해결책 2: JdbcTemplate 직접 사용

진짜 대량 INSERT가 필요하다면 JPA를 우회하는 것이 가장 확실하다.

```java
@Repository
@RequiredArgsConstructor
public class PostBulkRepository {

    private final JdbcTemplate jdbcTemplate;

    public void bulkInsert(List<Post> posts) {
        String sql = "INSERT INTO post (title, content, created_at) VALUES (?, ?, ?)";

        jdbcTemplate.batchUpdate(sql, posts, posts.size(), (ps, post) -> {
            ps.setString(1, post.getTitle());
            ps.setString(2, post.getContent());
            ps.setTimestamp(3, Timestamp.valueOf(post.getCreatedAt()));
        });
    }
}
```

IDENTITY 전략을 유지하면서도 진짜 배치 INSERT를 사용할 수 있다.
단, JPA 1차 캐시와 동기화되지 않으므로 벌크 연산 후에는 `em.clear()`를 고려해야 한다.

---

## 3. ID가 null이 아닐 때 — merge()의 SELECT 문제

### 상황

ID를 직접 할당하는 경우가 있다.
UUID 전략이나 외부 시스템에서 ID를 받아서 저장하는 경우다.

```java
@Entity
public class Post {
    @Id
    private String id;  // UUID, 직접 할당
}

// 서비스에서
Post post = new Post();
post.setId(UUID.randomUUID().toString());
postRepository.save(post);  // 이미 ID가 있다
```

이때 `save()` 내부에서 어떤 일이 벌어지는지 다시 보면:

```java
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        em.persist(entity);   // 새 엔티티 → INSERT
        return entity;
    } else {
        return em.merge(entity);  // 기존 엔티티? → SELECT 후 판단
    }
}
```

`isNew()` 기본 구현은 **ID가 null이면 새 엔티티, null이 아니면 기존 엔티티**로 판단한다.

ID를 직접 할당했기 때문에 `isNew()`는 `false`를 반환한다.
그 결과 `em.merge()`가 호출되고, Hibernate는 DB에 해당 ID가 존재하는지 SELECT를 먼저 보낸다.

100개를 저장하려 했는데 100번의 SELECT가 먼저 발생한다.

### 해결책: Persistable 구현

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Post implements Persistable<String> {

    @Id
    private String id;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @Override
    public String getId() {
        return id;
    }

    @Override
    public boolean isNew() {
        return createdAt == null;  // 처음 저장 시에는 createdAt이 null
    }
}
```

`Persistable<ID>`를 구현하면 `isNew()` 판단 로직을 직접 제어할 수 있다.
`@CreatedDate`는 `persist()` 이전까지 null이므로, 이를 새 엔티티 여부의 기준으로 삼는다.

`@PostPersist`, `@PostLoad` 어노테이션으로 상태를 관리하는 방법도 있다.

```java
@Entity
public class Post implements Persistable<String> {

    @Id
    private String id;

    @Transient
    private boolean isNew = true;

    @Override
    public String getId() {
        return id;
    }

    @Override
    public boolean isNew() {
        return isNew;
    }

    @PostPersist
    @PostLoad
    void markNotNew() {
        this.isNew = false;
    }
}
```

다만 `@Transient` 필드는 직렬화/역직렬화 시 초기화되므로 캐시나 세션과 함께 사용할 때 주의가 필요하다.

---

## 4. 성능 비교 요약

| 상황 | 발생 쿼리 | 원인 |
|------|-----------|------|
| `save()` 루프 (외부 트랜잭션 없음) | INSERT × N + 커밋 × N | 트랜잭션 N개 생성 |
| `saveAll()` (IDENTITY 전략) | INSERT × N (배치 없음) | IDENTITY가 배치 비활성화 |
| `saveAll()` (SEQUENCE + batch_size) | INSERT 배치 | 정상 동작 |
| `save()` (ID 직접 할당) | SELECT × N + INSERT × N | merge() 호출 |
| `save()` + `Persistable` 구현 | INSERT × N | isNew() 제어 |
| `JdbcTemplate.batchUpdate()` | INSERT 배치 | JDBC 직접 제어 |

---

## 5. 실무 관점의 주의사항

### 배치 INSERT 후 1차 캐시 동기화

`JdbcTemplate`으로 배치 INSERT를 하면 JPA 1차 캐시에는 해당 엔티티가 없다.
이후에 같은 엔티티를 `findById()`로 조회하면 캐시 미스가 발생해 DB에서 읽는다.
대량 저장 후 즉시 조회가 필요하다면 `em.clear()` 또는 별도 캐시 전략을 고려한다.

### SEQUENCE allocationSize와 ID 공백

`allocationSize = 50`으로 설정하면 서버 재시작 시 현재 블록의 나머지 ID가 사용되지 않고 낭비된다.
ID에 공백이 생기는 것이 문제가 되는 시스템(감사 추적, 순번 보장)에서는 주의가 필요하다.
대부분의 경우 ID 공백은 문제가 되지 않으므로 allocationSize를 크게 잡는 것이 유리하다.

### 트랜잭션 경계와 flush 시점

```java
@Transactional
public void saveAll(List<Post> posts) {
    for (int i = 0; i < posts.size(); i++) {
        postRepository.save(posts.get(i));
        if (i % 100 == 0) {
            em.flush();    // 주기적으로 플러시
            em.clear();    // 1차 캐시 비워서 메모리 절약
        }
    }
}
```

수만 건을 단일 트랜잭션으로 처리할 때는 1차 캐시가 메모리를 과다하게 사용할 수 있다.
주기적인 `flush() + clear()`로 메모리를 관리한다.

---

## 핵심 원칙 정리

### save() vs saveAll()
`saveAll()`은 단일 트랜잭션을 보장한다. 외부 `@Transactional`이 없다면 `save()` 루프는 N번 커밋을 발생시킨다.

### IDENTITY는 배치를 막는다
`GenerationType.IDENTITY`는 DB가 INSERT 후 ID를 생성하므로 Hibernate가 INSERT를 묶을 수 없다. 진짜 배치가 필요하다면 `SEQUENCE` 전략 + `allocationSize`를 사용하거나 `JdbcTemplate`으로 우회한다.

### ID가 있으면 merge()가 SELECT를 부른다
ID를 직접 할당하는 엔티티를 `save()`하면 `merge()`가 호출되어 SELECT가 선행된다. `Persistable<ID>`를 구현해서 `isNew()` 로직을 직접 제어한다.

---

## 마치며

JPA의 편리함 뒤에는 이런 함정들이 숨어 있다.
`saveAll()` 한 줄이면 충분하다고 생각했지만, 실제로는 전략 선택부터 트랜잭션 경계, ID 생성 방식까지 모두 맞물려 있다.

중요한 건 쿼리 로그를 직접 확인하는 습관이다.
`spring.jpa.show-sql=true`와 `hibernate.format_sql=true`를 켜고, 기대한 쿼리가 실제로 나가는지 눈으로 확인해야 한다.

코드가 맞다고 느껴도, 로그가 진실을 말해준다.

---

## 참고자료

- [Spring Data JPA - SimpleJpaRepository 소스](https://github.com/spring-projects/spring-data-jpa/blob/main/spring-data-jpa/src/main/java/org/springframework/data/jpa/repository/support/SimpleJpaRepository.java)
- [Hibernate ORM - Batch Processing](https://docs.jboss.org/hibernate/orm/6.0/userguide/html_single/Hibernate_User_Guide.html#batch)
- [Spring Data - Persistable interface](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Persistable.html)
