# 효율적인 JPA 조회 전략, N+1을 없애는 방법들

> Devon의 N+1 문제 해결과 성능 최적화 원칙

---

## 들어가며

JPA를 사용하면서 가장 흔하게 마주치는 문제는 **N+1 쿼리**다. 목록을 조회했을 뿐인데 수백, 수천 개의 쿼리가 발생하는 것을 보고 당황했던 경험이 누구나 있을 것이다.

이 글은 내가 실무에서 겪은 성능 이슈와 그 해결 과정을 정리한 것이다. 주요 니즈는:

- **N+1 문제 완전 제거**: 목록 조회 시 추가 쿼리가 발생하지 않도록
- **메모리 효율**: 필요한 데이터만 조회
- **명시적 의도**: 코드만 봐도 어떤 쿼리가 나갈지 예측 가능
- **일관된 패턴**: 팀원 누구나 따라할 수 있는 표준화된 방식

---

## 1. LAZY 로딩: 모든 것의 시작점

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "member_id", nullable = false)
private Member member;
```

**모든 연관관계는 LAZY 로딩으로 시작한다.**

### EAGER 로딩을 쓰지 않는 이유

많은 사람이 N+1 문제를 EAGER 로딩으로 해결하려 한다.

```java
// 나쁜 예
@ManyToOne(fetch = FetchType.EAGER)
private Member member;
```

이렇게 하면 Post를 조회할 때마다 **무조건** Member도 함께 조회된다. Member 정보가 필요 없는 상황에도 조인이 발생한다. 여러 연관관계가 EAGER면 불필요한 조인이 기하급수적으로 늘어난다.

**원칙**: 기본은 LAZY, 필요한 곳에서만 Fetch Join 또는 Constructor Projection으로 해결

---

## 2. 조회 전략의 양대 산맥

JPA 조회는 크게 두 가지 패턴으로 나뉜다. 각각 명확한 사용 시점이 있다.

### 2.1 Fetch Join — 단건 조회의 최적해

```java
@Query("SELECT p FROM Post p JOIN FETCH p.member WHERE p.id = :id")
Optional<Post> findByIdWithMember(@Param("id") Long id);
```

**사용 시점**: 하나의 Post와 그 Member를 조회할 때.

왜 사용하는가:
1. **완전한 엔티티**: 영속성 컨텍스트에서 관리되는 실제 엔티티
2. **변경 감지**: 엔티티를 수정하면 자동으로 UPDATE 쿼리
3. **쿼리 1개**: Post와 Member를 한 번에 조회

```java
// Before - N+1 발생
Post post = postRepository.findById(1L).get();
String nickname = post.getMember().getNickname();  // 추가 쿼리!

// After - Fetch Join
Post post = postRepository.findByIdWithMember(1L).get();
String nickname = post.getMember().getNickname();  // 쿼리 없음
```

**단점**: 컬렉션 Fetch Join은 페이징이 깨진다 (3절에서 설명).

### 2.2 Constructor Projection — 목록 조회의 정답

```java
@Override
public Page<PostQueryDto> findAllActiveWithMemberAsDto(Pageable pageable) {
    List<PostQueryDto> content = queryFactory
        .select(Projections.constructor(PostQueryDto.class,
            post.id, post.title, post.createdAt,
            post.viewsCount, post.likeCount, post.commentCount,
            member.id, member.nickname, member.email
        ))
        .from(post)
        .join(post.member, member)
        .where(isNotDeleted())
        .orderBy(getOrderSpecifiers(pageable))
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();

    JPAQuery<Long> countQuery = queryFactory
        .select(post.count())
        .from(post)
        .where(isNotDeleted());

    return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
}
```

**사용 시점**: 목록 조회, 검색, 페이징.

왜 사용하는가:

1. **필요한 필드만 SELECT**
   ```sql
   -- Fetch Join: 모든 컬럼
   SELECT p.*, m.* FROM post p JOIN member m ...

   -- Constructor Projection: 필요한 컬럼만
   SELECT p.id, p.title, ..., m.id, m.nickname FROM post p JOIN member m ...
   ```

2. **메모리 효율**: 100개 Post 조회 시 Fetch Join은 Post 객체 + Member 객체 = 200개, DTO는 100개.

3. **N+1 문제 없음**: JOIN으로 한 번에 모든 데이터를 포함.

4. **영속성 컨텍스트 관리 불필요**: DTO는 영속 상태가 아니므로 변경 감지 오버헤드 없음. 읽기 전용 조회에 최적.

### DTO 정의는 Record로

```java
public record PostQueryDto(
    Long postId,
    String title,
    Instant createdAt,
    Long viewsCount,
    Long likeCount,
    Long commentCount,
    Long memberId,
    String memberNickname,
    String memberEmail
) { }
```

Record를 사용하면 불변성이 보장되고, 자동으로 getter, equals, hashCode가 생성된다. 데이터 전달 객체라는 의도도 명확하게 드러난다.

---

## 3. Fetch Join의 함정

### 컬렉션 Fetch Join + Pagination 문제

```java
// 위험한 코드
@Query("SELECT p FROM Post p JOIN FETCH p.comments")
Page<Post> findAllWithComments(Pageable pageable);
```

이 코드는 실제 쿼리에서 `LIMIT`이 없다. DB가 아닌 애플리케이션 메모리에서 페이징이 이뤄진다. Hibernate가 경고를 띄우는 이유다.

```
WARN: HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
```

### 해결 방법 1: @BatchSize

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    @BatchSize(size = 100)
    private List<Comment> comments;
}
```

또는 전역 설정:
```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

Post 10개를 조회한 후 Comment에 접근하면, 100번의 쿼리가 아닌 `IN (1, 2, 3, ..., 10)` 쿼리 1개로 처리된다.

### 해결 방법 2: Constructor Projection (권장)

```java
.select(Projections.constructor(PostQueryDto.class,
    post.id,
    post.title,
    comment.count()  // 집계
))
.from(post)
.leftJoin(post.comments, comment)
.groupBy(post.id)
.offset(pageable.getOffset())
.limit(pageable.getPageSize())
.fetch();
```

정확한 페이징, 메모리 효율, N+1 없음.

---

## 4. QueryDSL: 동적 쿼리의 정답

JPQL은 동적 쿼리가 어렵다.

```java
// 검색 조건이 있을 수도, 없을 수도
String jpql = "SELECT p FROM Post p WHERE 1=1";
if (keyword != null) jpql += " AND p.title LIKE :keyword";
// 문자열 조작... 끔찍하다
```

QueryDSL은 타입 안전하고 읽기 쉽다.

```java
@Override
public Page<PostQueryDto> searchByTitleOrContent(String keyword, Pageable pageable) {
    BooleanExpression condition = isNotDeleted().and(keywordSearch(keyword));

    List<PostQueryDto> content = queryFactory
        .select(Projections.constructor(PostQueryDto.class, ...))
        .from(post)
        .join(post.member, member)
        .where(condition)
        .orderBy(getOrderSpecifiers(pageable))
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();

    JPAQuery<Long> countQuery = queryFactory
        .select(post.count())
        .from(post)
        .where(condition);

    return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
}
```

### BooleanExpression 분리의 힘

```java
private BooleanExpression isNotDeleted() {
    return post.isDeleted.eq(false);
}

private BooleanExpression titleContains(String keyword) {
    return keyword != null ? post.title.containsIgnoreCase(keyword) : null;
}

private BooleanExpression contentContains(String keyword) {
    return keyword != null ? post.content.containsIgnoreCase(keyword) : null;
}

// 조합
private BooleanExpression keywordSearch(String keyword) {
    return titleContains(keyword).or(contentContains(keyword));
}
```

조건을 메서드로 분리하면 재사용 가능하고, `null`을 반환하면 조건에서 자동으로 제외된다.

### Count 쿼리 분리 최적화

Count 쿼리에는 JOIN이 필요 없는 경우가 많다.

```sql
-- Content 쿼리
SELECT p.id, p.title, m.nickname
FROM post p JOIN member m ON p.member_id = m.id
WHERE p.is_deleted = false
ORDER BY p.created_at DESC
LIMIT 10 OFFSET 0;

-- Count 쿼리
SELECT COUNT(p.id)
FROM post p
WHERE p.is_deleted = false;
-- JOIN 없음, ORDER BY 없음
```

`PageableExecutionUtils.getPage()`를 쓰면 마지막 페이지에서 Count 쿼리 자체를 실행하지 않는 최적화도 적용된다.

---

## 5. N+1 해결 무기고

실무에서 마주친 N+1 문제와 해결 방법들을 정리했다.

### 5.1 Scalar Projection — 특정 값만 조회

```java
// 비효율 - Comment의 postId만 필요한데 Post를 전체 조회
Comment comment = commentRepository.findById(commentId).get();
Long postId = comment.getPost().getId();  // Post SELECT 쿼리 발생

// 개선 - FK만 가져오기
@Query("SELECT c.post.id FROM Comment c WHERE c.id = :commentId")
Optional<Long> findPostIdByCommentId(@Param("commentId") Long commentId);
```

### 5.2 getReferenceById() — Proxy 활용

```java
// 비효율 - Comment 생성 시 Post 전체가 필요한가?
Post post = postRepository.findById(postId).get();  // SELECT 쿼리
Comment comment = Comment.create(member, post, content);
commentRepository.save(comment);  // post_id만 필요한데...

// 개선 - Proxy 객체 (SELECT 없음)
validatePostExists(postId);
Post post = postRepository.getReferenceById(postId);
Comment comment = Comment.create(member, post, content);
commentRepository.save(comment);
```

| 메서드 | 반환 | 쿼리 | 사용 시점 |
|-------|------|------|----------|
| **findById** | 실제 엔티티 | SELECT 즉시 실행 | 엔티티 데이터가 필요할 때 |
| **getReferenceById** | Proxy 객체 | FK 접근 시에만 실행 | FK만 필요할 때 |

**주의**: Proxy이므로 실제 필드에 접근하면 쿼리가 나간다. FK 설정 용도로만 사용.

### 5.3 Bulk Update — 엔티티 없이 업데이트

```java
// 비효율 - 좋아요 카운터를 위해 Post를 조회?
Post post = postRepository.findById(postId).get();  // SELECT
post.incrementLikeCount();  // 변경 감지로 UPDATE
// 총 2개 쿼리

// 개선 - Bulk Update
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE Post p SET p.likeCount = p.likeCount + 1 WHERE p.id = :postId")
int incrementLikeCount(@Param("postId") Long postId);
```

`clearAutomatically = true`는 쿼리 실행 후 영속성 컨텍스트를 자동 초기화해서, Bulk Update 결과와 영속성 컨텍스트 캐시 간 불일치를 방지한다.

### 5.4 existsById vs count

```java
// 비효율
long count = postRepository.count();
if (count > 0) { ... }

// 개선
if (postRepository.existsById(postId)) { ... }
```

`existsById`는 첫 번째 매치에서 즉시 반환하므로 더 빠르다.

### 5.5 Batch Fetch Size — 최후의 방어선

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

```java
// Batch Size 없으면: Post 100개 → 100개 쿼리
SELECT * FROM member WHERE id = 1
SELECT * FROM member WHERE id = 2
...

// Batch Size 있으면: 1개 쿼리
SELECT * FROM member WHERE id IN (1, 2, 3, ..., 100)
```

Fetch Join을 깜빡했을 때의 최후 방어선이다. 코드 수정 없이 전역으로 적용된다. 반드시 설정해두자.

---

## 6. 조회 전략 선택 가이드

| 시나리오 | 전략 | 이유 |
|---------|------|------|
| **단건 + 연관 엔티티** | Fetch Join | 완전한 엔티티, 쿼리 1개 |
| **목록 + 연관 데이터** | Constructor Projection | 필요한 필드만, 메모리 효율 |
| **동적 검색** | QueryDSL + BooleanExpression | 조건 조합, 타입 안전 |
| **카운터 증감** | Bulk Update | Entity 로딩 불필요 |
| **FK만 필요** | getReferenceById() | Proxy, 쿼리 없음 |
| **존재 확인** | existsById | count보다 빠름 |
| **특정 값만** | Scalar Projection | Entity 로딩 불필요 |
| **컬렉션 조회** | @BatchSize 또는 별도 쿼리 | 페이징 안전 |

---

## 7. 안티패턴과 해결

### ❌ EAGER 로딩으로 N+1 해결 시도

```java
@ManyToOne(fetch = FetchType.EAGER)
private Member member;
```

필요 없을 때도 항상 조인이 발생하고, 제어가 불가능하다.
**해결**: LAZY + 명시적 Fetch Join 또는 Constructor Projection

### ❌ 컬렉션 Fetch Join + Pagination

```java
@Query("SELECT p FROM Post p JOIN FETCH p.comments")
Page<Post> findAll(Pageable pageable);
```

메모리에서 페이징이 이뤄지고, OOM 위험이 있다.
**해결**: Constructor Projection 또는 @BatchSize

### ❌ 불필요한 Entity 로딩

```java
Post post = postRepository.findById(postId).get();
post.incrementViewCount();
```

카운터 증감을 위해 Entity를 조회할 필요가 없다.
**해결**: Bulk Update 사용

---

## 마치며

JPA 성능 최적화는 **쿼리를 예측 가능하게** 만드는 일이다. 코드를 보면 몇 개의 쿼리가 나갈지 알 수 있어야 한다.

`@Transactional(readOnly = true)`도 잊지 말자. 조회 전용 트랜잭션에서는 Dirty Checking이 비활성화되어 성능이 향상되고, 의도도 명확해진다.

N+1 문제는 코드 리뷰에서 잡기 어렵다. 실행해봐야 안다. 그래서 원칙이 중요하다. 항상 쿼리 개수를 세어보자.

중요한 건 **왜 이 전략을 선택했는가**를 이해하는 것이다. 맥락이 다르다면 다른 전략을 써야 한다.
