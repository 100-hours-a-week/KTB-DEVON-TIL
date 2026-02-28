# Jackson, 그냥 쓰지 말고 제대로 이해하고 쓰자

> "JSON 직렬화/역직렬화, 생각보다 많은 함정이 숨어 있다."

## 0. 이 글을 쓰게 된 계기

Spring으로 API를 만들면 Jackson은 그냥 따라온다.
별도로 설정하지 않아도 `@RestController`에서 객체를 반환하면 JSON이 되고,
`@RequestBody`로 JSON을 받으면 객체가 된다.

처음에는 "역시 Spring, 알아서 다 해주는구나" 싶었다.

문제는 그 다음이었다.

- 양방향 연관관계가 있는 엔티티를 반환했더니 **StackOverflowError**가 터졌다.
- `LocalDateTime`을 JSON으로 내렸더니 **배열 형태**로 나왔다. `[2024, 1, 15, 14, 30, 0]` 이런 식으로.
- 프론트에서 `user_name`으로 보내는데 백엔드에서 `userName`으로 받으니 **null**이 들어왔다.
- API 스펙이 바뀌어서 필드가 추가됐는데, 갑자기 **UnrecognizedPropertyException**이 터졌다.

이런 일들을 겪고 나니,
Jackson은 "알아서 해주는 것"이 아니라 **"설정해야 제대로 동작하는 것"**이라는 걸 깨달았다.

이 글은 그 과정에서 배운 것들을 정리한 시도다.

---

## 1. Jackson이 뭐길래 이렇게 많이 쓸까?

한 줄로 말하면,

> **"Java 객체 ↔ JSON 문자열 변환을 담당하는 라이브러리"**

조금 더 풀어보면:

- **직렬화(Serialization)**: Java 객체 → JSON 문자열
- **역직렬화(Deserialization)**: JSON 문자열 → Java 객체
- Spring MVC, Spring WebFlux에서 기본 JSON 처리기로 사용 (`MappingJackson2HttpMessageConverter`)
- 핵심 모듈: `jackson-core`, `jackson-databind`, `jackson-annotations`

### ObjectMapper – Jackson의 심장

Jackson을 쓴다는 건 결국 `ObjectMapper`를 쓴다는 것이다.

```java
ObjectMapper objectMapper = new ObjectMapper();

// 직렬화: 객체 → JSON
String json = objectMapper.writeValueAsString(user);

// 역직렬화: JSON → 객체
User user = objectMapper.readValue(json, User.class);
```

Spring Boot에서는 `ObjectMapper`가 자동으로 빈으로 등록되어 있고,
컨트롤러의 요청/응답 처리에서 내부적으로 이 `ObjectMapper`를 사용한다.

문제는, **기본 설정만으로는 실무의 다양한 상황을 커버하지 못한다**는 점이다.

---

## 2. 실무에서 가장 많이 만나는 Jackson 함정들

### 2-1. 순환 참조 – 양방향 연관관계의 무한 루프

**증상**

JPA 엔티티에서 양방향 연관관계를 맺고, 그대로 JSON으로 직렬화하려고 하면
`StackOverflowError` 또는 무한 JSON이 생성된다.

```java
@Entity
public class Member {
    @ManyToOne
    private Team team;
}

@Entity
public class Team {
    @OneToMany(mappedBy = "member")
    private List<Member> members;
}
```

이 상태에서 `Team`을 JSON으로 변환하면:
`Team → members → Member → team → Team → members → ...` 무한 반복.

**원인**

Jackson은 객체 그래프를 그대로 탐색하면서 직렬화한다.
순환 구조가 있으면 끝없이 돌게 된다.

**해결 방법**

| 방법 | 설명 | 권장도 |
|-----|------|-------|
| `@JsonManagedReference` / `@JsonBackReference` | 부모-자식 관계 명시, 자식 쪽은 직렬화에서 제외 | 중 |
| `@JsonIgnore` | 한쪽 참조를 완전히 직렬화 대상에서 제외 | 중 |
| `@JsonIdentityInfo` | ID 기반으로 순환 참조를 끊고 동일 객체 식별 | 하 |
| **DTO 분리** | 엔티티를 직접 반환하지 않고 DTO로 변환 | **상** |

내가 선호하는 방법은 **DTO 분리**다.

```java
// 나쁜 예 - 엔티티 직접 반환
@GetMapping("/teams/{id}")
public Team getTeam(@PathVariable Long id) {
    return teamRepository.findById(id).orElseThrow();
}

// 좋은 예 - DTO로 변환
@GetMapping("/teams/{id}")
public TeamResponse getTeam(@PathVariable Long id) {
    Team team = teamRepository.findById(id).orElseThrow();
    return TeamResponse.from(team);
}
```

이유는 복합적이다:

1. **순환 참조 문제를 근본적으로 제거**할 수 있다.
2. API 스펙과 도메인 모델을 **분리**할 수 있다.
3. 민감 정보(비밀번호 등)가 실수로 노출되는 걸 **방지**할 수 있다.

---

### 2-2. 날짜/시간 – LocalDateTime이 이상하게 나온다

**증상**

`LocalDateTime`을 JSON으로 변환했더니 이상한 배열이 나온다.

```json
{
  "createdAt": [2024, 1, 15, 14, 30, 0]
}
```

또는 역직렬화할 때 `InvalidFormatException`이 터진다.

**원인**

기본 Jackson은 Java 8 날짜/시간 타입을 제대로 처리하지 못한다.
`jackson-datatype-jsr310` 모듈이 필요하다.

**해결 방법**

Spring Boot 2.x 이상에서는 보통 자동으로 등록되지만, 포맷 지정이 필요하다.

```java
// 필드별 설정
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Seoul")
private LocalDateTime createdAt;
```

또는 전역 설정:

```yaml
# application.yml
spring:
  jackson:
    date-format: "yyyy-MM-dd HH:mm:ss"
    time-zone: "Asia/Seoul"
```

나는 보통 **ISO 8601 포맷**(`2024-01-15T14:30:00`)을 기본으로 두고,
특별히 다른 포맷이 필요한 필드만 `@JsonFormat`으로 개별 지정하는 편이다.

---

### 2-3. Unknown Property – 왜 갑자기 역직렬화가 실패하지?

**증상**

프론트에서 보내는 JSON에 백엔드 DTO에 없는 필드가 포함되면
`UnrecognizedPropertyException`이 터진다.

```json
// 프론트에서 보내는 JSON
{
  "name": "Devon",
  "age": 30,
  "hobby": "coding"  // DTO에 없는 필드
}
```

```java
// 백엔드 DTO
public class UserRequest {
    private String name;
    private int age;
    // hobby 필드 없음 → 에러!
}
```

**원인**

Jackson 기본 설정은 "알 수 없는 필드가 있으면 실패"다.
API 스펙이 점진적으로 변하는 환경에서는 이게 문제가 된다.

**해결 방법**

```java
// 방법 1: 클래스에 애노테이션 추가
@JsonIgnoreProperties(ignoreUnknown = true)
public class UserRequest {
    // ...
}

// 방법 2: ObjectMapper 전역 설정
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```

```yaml
# 방법 3: Spring Boot application.yml
spring:
  jackson:
    deserialization:
      fail-on-unknown-properties: false
```

다만, 무조건 무시하는 게 좋은 건 아니다.
**API 계약이 명확한 환경**에서는 오히려 엄격하게 두는 게 버그를 빨리 잡는 데 도움이 된다.

---

### 2-4. snake_case vs camelCase – 프론트엔드와 싸우는 네이밍

**증상**

프론트에서는 `user_name`으로 보내는데,
백엔드 DTO는 `userName`으로 받으니 `null`이 들어온다.

**원인**

Jackson 기본 전략은 Java 필드명을 그대로 사용한다 (camelCase).
많은 프론트엔드/외부 API는 snake_case를 쓴다.

**해결 방법**

```java
// 방법 1: 필드별 지정
@JsonProperty("user_name")
private String userName;

// 방법 2: 전역 네이밍 전략
objectMapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
```

```yaml
# 방법 3: Spring Boot application.yml
spring:
  jackson:
    property-naming-strategy: SNAKE_CASE
```

내 경험상, **전역 설정**으로 통일하는 게 편했다.
단, 기존 API가 이미 camelCase로 나가고 있다면 영향 범위를 먼저 확인해야 한다.

---

## 3. 조금 더 깊이 들어가면 만나는 이슈들

### 3-1. Enum 직렬화 – 순서가 바뀌면 큰일난다

**증상**

Enum을 추가/삭제했더니 기존 데이터와 호환이 안 된다.

**원인**

Jackson의 기본 Enum 직렬화 전략은 `name()` 값을 사용한다.
하지만 DB에 `@Enumerated(EnumType.ORDINAL)`로 저장하고 있다면,
Enum 순서가 바뀌면서 데이터가 꼬일 수 있다.

```java
// 기존
public enum UserStatus {
    ACTIVE,   // 0
    INACTIVE  // 1
}

// 변경 후 - PENDING 추가
public enum UserStatus {
    PENDING,  // 0 (새로 추가!)
    ACTIVE,   // 1 (기존 0 → 1로 변경!)
    INACTIVE  // 2
}
```

**해결 방법**

```java
public enum UserStatus {
    ACTIVE("A"),
    INACTIVE("I"),
    PENDING("P");

    private final String code;

    @JsonValue
    public String getCode() {
        return code;
    }

    @JsonCreator
    public static UserStatus fromCode(String code) {
        return Arrays.stream(values())
            .filter(s -> s.code.equals(code))
            .findFirst()
            .orElseThrow();
    }
}
```

**핵심 포인트:**

1. 외부에 노출되는 값은 Enum `name()` 대신 **별도 코드 필드**를 사용한다.
2. `@JsonValue`로 직렬화 값을, `@JsonCreator`로 역직렬화 로직을 지정한다.
3. DB에는 `@Enumerated(EnumType.STRING)` 또는 코드 값을 저장한다.

---

### 3-2. 제네릭 타입 – TypeReference를 모르면 고생한다

**증상**

`List<User>`를 역직렬화했는데, `List<LinkedHashMap>`이 나온다.

```java
// 이렇게 하면 안 된다
List<User> users = objectMapper.readValue(json, List.class);
// 결과: List<LinkedHashMap> - User 타입 정보가 사라짐
```

**원인**

Java는 런타임에 제네릭 타입 정보가 소거(Type Erasure)된다.
Jackson이 `List<User>`인지 `List<Order>`인지 알 방법이 없다.

**해결 방법**

```java
// TypeReference 사용
List<User> users = objectMapper.readValue(json, new TypeReference<List<User>>() {});

// 또는 JavaType 사용
JavaType type = objectMapper.getTypeFactory()
    .constructCollectionType(List.class, User.class);
List<User> users = objectMapper.readValue(json, type);
```

Spring MVC에서는 컨트롤러 메서드 시그니처를 통해 자동 처리되는 경우가 많지만,
**직접 ObjectMapper를 사용할 때는 반드시 TypeReference를 써야 한다.**

---

### 3-3. 민감 정보 노출 – 비밀번호가 JSON에 찍힌다

**증상**

User 엔티티를 그대로 반환했더니 비밀번호가 JSON에 노출됐다.

```json
{
  "id": 1,
  "email": "devon@example.com",
  "password": "$2a$10$..."  // 이런!
}
```

**해결 방법**

```java
// 방법 1: @JsonIgnore
public class User {
    private String email;

    @JsonIgnore
    private String password;
}

// 방법 2: @JsonInclude (null이면 제외)
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserResponse {
    // ...
}

// 방법 3: DTO 분리 (권장)
public class UserResponse {
    private Long id;
    private String email;
    // password 필드 자체가 없음

    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getEmail());
    }
}
```

다시 한번 강조하지만, **DTO 분리가 가장 확실한 방법**이다.
엔티티에 Jackson 애노테이션을 덕지덕지 붙이는 것보다
처음부터 API용 DTO를 별도로 두는 게 장기적으로 관리하기 좋다.

---

## 4. 성능까지 신경 쓰고 싶다면

### ObjectMapper는 재사용해야 한다

가장 흔한 실수 중 하나가 `ObjectMapper`를 매번 새로 생성하는 것이다.

```java
// 나쁜 예 - 매번 생성
public String toJson(Object obj) {
    ObjectMapper mapper = new ObjectMapper();  // 비용이 크다!
    return mapper.writeValueAsString(obj);
}

// 좋은 예 - 재사용
private static final ObjectMapper MAPPER = new ObjectMapper();

public String toJson(Object obj) {
    return MAPPER.writeValueAsString(obj);
}
```

`ObjectMapper`는 **thread-safe**하다.
한 번 설정하고 싱글턴/빈으로 재사용하는 게 맞다.

### 대용량 JSON은 스트리밍 API를 고려하자

수십 MB 이상의 JSON을 한 번에 메모리에 올리면 GC 부하, OOM이 발생할 수 있다.

```java
// 스트리밍 파싱 (JsonParser)
try (JsonParser parser = objectMapper.getFactory().createParser(inputStream)) {
    while (parser.nextToken() != null) {
        // 토큰 단위로 처리
    }
}

// 스트리밍 생성 (JsonGenerator)
try (JsonGenerator generator = objectMapper.getFactory().createGenerator(outputStream)) {
    generator.writeStartArray();
    for (User user : users) {
        objectMapper.writeValue(generator, user);
    }
    generator.writeEndArray();
}
```

또는 `ObjectReader`/`ObjectWriter`를 재사용해서 반복 생성 비용을 줄일 수 있다.

```java
private final ObjectReader userReader = objectMapper.readerFor(User.class);
private final ObjectWriter userWriter = objectMapper.writerFor(User.class);
```

---

## 5. 실무에서 가져가면 좋은 패턴들

### DTO 분리 전략

이미 여러 번 언급했지만, 다시 한번 정리하면:

1. **엔티티를 직접 API 응답으로 쓰지 않는다.**
2. 요청용 DTO(`XxxRequest`)와 응답용 DTO(`XxxResponse`)를 분리한다.
3. Jackson 관련 애노테이션은 DTO에만 둔다.

### 전역 설정 vs 필드별 설정

| 설정 대상 | 전역 설정 권장 | 필드별 설정 권장 |
|---------|--------------|----------------|
| 네이밍 전략 | O (snake_case 등 통일) | - |
| 날짜 포맷 | O (기본 포맷) | O (특수 포맷) |
| Unknown Property 무시 | O | - |
| 특정 필드 제외 | - | O (`@JsonIgnore`) |

### 테스트 코드로 직렬화 검증

API 스펙이 바뀌면 직렬화/역직렬화가 깨질 수 있다.
테스트 코드로 미리 검증해두면 안전하다.

```java
@Test
void 직렬화_포맷_검증() throws Exception {
    UserResponse response = new UserResponse(1L, "devon@example.com");

    String json = objectMapper.writeValueAsString(response);

    assertThat(json).contains("\"id\":1");
    assertThat(json).contains("\"email\":\"devon@example.com\"");
    assertThat(json).doesNotContain("password");
}

@Test
void 역직렬화_호환성_검증() throws Exception {
    String json = """
        {
          "name": "Devon",
          "age": 30,
          "unknownField": "ignored"
        }
        """;

    UserRequest request = objectMapper.readValue(json, UserRequest.class);

    assertThat(request.getName()).isEqualTo("Devon");
    assertThat(request.getAge()).isEqualTo(30);
}
```

---

## 6. 마무리

처음 Jackson을 쓸 때는 "Spring이 알아서 해주겠지"라고 생각했다.
근데 실무에서 겪어보니, Jackson은 **"설정하는 만큼 동작하는"** 라이브러리였다.

이 글에서 다룬 내용을 요약하면:

1. **순환 참조**는 DTO 분리로 근본적으로 해결한다.
2. **날짜/시간**은 `jackson-datatype-jsr310`과 포맷 설정이 필수다.
3. **Unknown Property**는 상황에 따라 무시 설정을 고려한다.
4. **네이밍 전략**은 전역으로 통일하는 게 편하다.
5. **Enum**은 코드 기반으로 직렬화하면 안전하다.
6. **ObjectMapper**는 싱글턴으로 재사용한다.

마지막으로, Jackson 문제의 80%는 **"엔티티를 직접 반환하지 말고 DTO를 쓰자"**로 해결된다.

처음에는 귀찮아 보이지만,
API 스펙 변경, 민감 정보 노출, 순환 참조 등의 문제를 미리 차단할 수 있다.

결국 Jackson을 잘 쓴다는 건,
**어떤 데이터를, 어떤 형태로, 어디까지 노출할 것인지**를 명확히 설계하는 것과 같다.
