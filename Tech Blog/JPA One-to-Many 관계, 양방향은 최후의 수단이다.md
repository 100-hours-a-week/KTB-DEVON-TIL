# JPA One-to-Many 관계, 양방향은 최후의 수단이다

> Devon의 양방향 관계 최소화와 도메인 분리 원칙

---

## 들어가며

JPA를 사용하다 보면 **양방향 관계**를 습관적으로 선언하게 된다. Post와 Comment가 있다면 자연스럽게 이렇게 작성한다.

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Comment> comments = new ArrayList<>();
}
```

"나중에 필요할 수도 있으니까", "양방향 탐색이 편하니까"라는 이유로.

하지만 이것이 **진짜 필요한 관계인가?** 이 글은 내가 실무에서 One-to-Many 양방향 관계를 최소화하게 된 이유와 그 전략을 정리한 것이다.

---

## 1. 핵심 원칙: 양방향 관계는 지양하자

나는 기본적으로 이렇게 설계한다.

```java
@Entity
@Table(name = "post")
public class Post extends BaseTimeEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", nullable = false)
    private Member member;

    // ❌ OneToMany는 선언하지 않는다
    // @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    // private List<Comment> comments = new ArrayList<>();
}
```

```java
@Entity
@Table(name = "comment")
public class Comment extends BaseTimeEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    private Post post;  // ✅ 단방향만 유지

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", nullable = false)
    private Member member;
}
```

**핵심**: Comment는 Post를 알아야 하지만, **Post는 Comment를 몰라도 된다.**

---

## 2. 왜 양방향을 피하는가

### 2.1 Fetch Join 시 메모리 문제

Post 목록을 조회하면서 Comment도 함께 가져오고 싶다고 가정하자.

```java
@Query("SELECT p FROM Post p JOIN FETCH p.comments")
List<Post> findAllWithComments();
```

이 쿼리가 실행하는 SQL을 보면 문제가 보인다.

```sql
SELECT p.*, c.*
FROM post p
LEFT JOIN comment c ON p.id = c.post_id
```

결과 데이터:
```
post_id | post_title    | comment_id | comment_content
--------|---------------|------------|------------------
1       | "JPA 입문"    | 1          | "좋은 글이네요"
1       | "JPA 입문"    | 2          | "감사합니다"
1       | "JPA 입문"    | 3          | "도움됐어요"
2       | "Spring Boot" | 4          | "잘 봤습니다"
2       | "Spring Boot" | 5          | "따라해볼게요"
```

Post 2개 + Comment 5개 = **5개의 Row**가 반환된다. Post 1개당 Comment가 100개면 100배 증가다.

더 심각한 것은 **페이징이 깨진다**는 점이다.

```java
@Query("SELECT p FROM Post p JOIN FETCH p.comments")
Page<Post> findAllWithComments(Pageable pageable);

// 실행
Page<Post> posts = postRepository.findAllWithComments(PageRequest.of(0, 10));
```

Hibernate 경고:
```
WARN HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
```

실제 쿼리에서 `LIMIT 10`이 없다. DB에서 **전체 데이터**를 메모리로 가져와, Hibernate가 메모리에서 중복 제거 후 10개를 잘라낸다.

Post가 10,000개, Comment가 평균 50개면 500,000개 Row를 메모리에 올린다. OutOfMemoryError다.

### 2.2 도메인 책임이 불명확해진다

Post가 Comment를 컬렉션으로 들고 있으면 이런 코드가 생긴다.

```java
Post post = postRepository.findById(1L).get();
List<Comment> comments = post.getComments();  // Post의 책임인가? Comment의 책임인가?
```

**Post의 책임**: 게시글 제목, 내용, 조회수, 좋아요 수 관리
**Comment의 책임**: 댓글 내용, 목록 조회, 정렬, 페이징

이 둘은 **독립적인 도메인**이다. 실무에서 댓글 조회 요구사항을 생각해보면 더 명확해진다.

- 댓글을 최신순/좋아요순으로 정렬
- 댓글 페이징 (20개씩)
- 신고된 댓글 필터링
- 작성자가 삭제한 댓글은 "삭제된 댓글입니다" 표시

이걸 `post.getComments()`로 해결하는 건 불가능하다. 어차피 별도의 Comment 조회 로직이 필요하다.

### 2.3 N+1 유발 가능성

```java
@Transactional(readOnly = true)
public List<PostDto> getAllPosts() {
    List<Post> posts = postRepository.findAll();

    return posts.stream()
        .map(post -> new PostDto(
            post.getId(),
            post.getTitle(),
            post.getComments().size()  // ⚠️ 여기서 N+1 발생!
        ))
        .toList();
}
```

Post 100개면 **101개의 쿼리**가 실행된다. `@OneToMany` 필드 자체가 이 실수를 가능하게 한다.

---

## 3. N+1 문제 원천 차단

가장 강력한 해결책은 **`@OneToMany` 자체를 제거**하는 것이다.

```java
@Entity
public class Post {
    // ✅ comments 필드 자체가 없다
}
```

`post.getComments()` 자체가 **컴파일 에러**가 된다. 개발자가 실수로 N+1을 유발할 가능성이 0%다.

### 올바른 조회 방법

#### Comment Count만 필요할 때

```java
@Query("""
    SELECT new PostWithCountDto(
        p.id, p.title, p.createdAt,
        COUNT(c.id)
    )
    FROM Post p
    LEFT JOIN Comment c ON c.post.id = p.id AND c.isDeleted = false
    GROUP BY p.id, p.title, p.createdAt
    """)
Page<PostWithCountDto> findAllWithCommentCount(Pageable pageable);
```

**쿼리 1개**로 Post + Comment 개수 조회.

#### Comment 상세 정보가 필요할 때

```java
// 1. Post 목록 조회 (쿼리 1개)
Page<PostQueryDto> posts = postRepository.findAll(pageable);

// 2. 필요한 경우에만 Comment 조회 (쿼리 1개)
Page<CommentQueryDto> comments = commentRepository.findByPostId(postId, commentPageable);
```

분리된 쿼리지만 **명시적이고 제어 가능**하다.

---

## 4. 실전 비교

### ❌ 양방향 + Fetch Join (문제 많음)

```java
@Query("SELECT p FROM Post p JOIN FETCH p.comments WHERE p.id = :id")
Optional<Post> findByIdWithComments(@Param("id") Long id);

Post post = postRepository.findByIdWithComments(postId).get();
List<Comment> comments = post.getComments();
// ⚠️ 전체 댓글을 메모리에 로드
// ⚠️ 페이징 불가
// ⚠️ 정렬 기준 변경 불가
```

### ✅ 단방향 + 별도 조회 (권장)

```java
@Entity
public class Post {
    // comments 필드 없음
}

// Repository
@Query("""
    SELECT new PostDetailDto(
        p.id, p.title, p.content, p.createdAt,
        m.id, m.nickname
    )
    FROM Post p
    JOIN p.member m
    WHERE p.id = :id AND p.isDeleted = false
    """)
Optional<PostDetailDto> findDetailById(@Param("id") Long id);

// Service
@Transactional(readOnly = true)
public PostDetailResponse getPostDetail(Long postId, Pageable commentPageable) {
    PostDetailDto post = postRepository.findDetailById(postId)
        .orElseThrow(() -> new NotFoundException("Post not found"));

    Page<CommentDto> comments = commentRepository.findByPostId(postId, commentPageable);

    return new PostDetailResponse(post, comments);
}
```

쿼리 2개로 명확하게 분리되고, Comment 페이징과 정렬이 자유롭게 된다.

---

## 5. 양방향이 적합한 경우

모든 One-to-Many가 나쁜 것은 아니다. **진짜 필요한 경우**도 있다.

### Order - OrderItem 패턴

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> orderItems = new ArrayList<>();

    public void addItem(OrderItem item) {
        orderItems.add(item);
        item.setOrder(this);
    }
}
```

Order와 OrderItem은 **생명주기를 같이한다.** Order 없이 OrderItem은 존재할 수 없고, 주문 생성 시 항목이 함께 저장되어야 한다. `cascade`, `orphanRemoval`로 일관성을 보장한다.

### 양방향이 적합한지 판단하는 질문

| 질문 | Order-OrderItem | Post-Comment |
|------|----------------|-------------|
| 생명주기가 같은가? | ✅ 같다 | ❌ 독립적 |
| 항상 함께 조회하는가? | ✅ 주문 조회 시 항목도 필요 | ❌ Post만 볼 때가 많다 |
| 부모를 통해서만 자식을 관리하는가? | ✅ Order를 통해서만 추가/삭제 | ❌ Comment는 독립적으로 CRUD |
| 자식의 개수가 적은가? | ✅ 보통 1~20개 | ❌ 수백, 수천 개 가능 |

**모두 ✅면 양방향 고려, 하나라도 ❌면 단방향.**

---

## 6. Comment Count 최적화 전략

양방향 없이 Post 목록에 댓글 개수를 표시하는 방법.

### 전략 1: 조인 + 집계

```java
@Query("""
    SELECT new PostListDto(
        p.id, p.title, p.createdAt,
        m.nickname,
        COUNT(c.id)
    )
    FROM Post p
    JOIN p.member m
    LEFT JOIN Comment c ON c.post.id = p.id AND c.isDeleted = false
    WHERE p.isDeleted = false
    GROUP BY p.id, p.title, p.createdAt, m.nickname
    ORDER BY p.createdAt DESC
    """)
Page<PostListDto> findAllWithCommentCount(Pageable pageable);
```

### 전략 2: 역정규화 - commentCount 컬럼

```java
@Entity
public class Post {
    private Long commentCount = 0L;

    public void incrementCommentCount() {
        this.commentCount++;
    }

    public void decrementCommentCount() {
        if (this.commentCount > 0) {
            this.commentCount--;
        }
    }
}
```

조회 시 조인이 필요 없어 빠르지만, 동기화 로직과 동시성 제어가 필요하다. 조회 빈도가 매우 높고, 정합성 허용 범위가 있을 때 적합하다.

---

## 7. 실무 체크리스트

### One-to-Many를 선언하기 전에

- [ ] 부모와 자식의 생명주기가 같은가?
- [ ] 항상 함께 조회하는가?
- [ ] 자식의 개수가 충분히 적은가? (< 50개)
- [ ] 자식을 별도로 페이징할 필요가 없는가?
- [ ] 부모를 통해서만 자식을 관리하는가?

5개 모두 Yes: 양방향 관계 고려
1개라도 No: 단방향 관계 권장

---

## 마치며

One-to-Many 양방향 관계는 **편리하지만 위험하다.**

- Fetch Join 시 메모리 폭증
- 페이징이 메모리에서 실행됨
- N+1 문제 유발 가능성
- 도메인 책임이 불명확해짐

반면 **단방향 + 별도 조회**는 각 도메인의 책임이 명확하고, 페이징과 정렬을 자유롭게 제어하며, 쿼리가 예측 가능하다.

양방향이 필요한 경우도 분명 있다. 하지만 **기본은 단방향**이어야 한다. "나중에 필요할지도"라는 이유로 양방향을 선언하지 말자.

필요할 때 추가하기는 쉽지만, 잘못 추가한 것을 제거하기는 어렵다.
