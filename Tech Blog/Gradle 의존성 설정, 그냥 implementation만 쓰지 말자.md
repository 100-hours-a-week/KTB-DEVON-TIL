# Gradle 의존성 설정, 그냥 implementation만 쓰지 말자

> "모든 의존성을 implementation으로 선언하는 순간, Gradle의 반은 포기한 것이다."

## 0. 이 글을 쓰게 된 계기

처음 Spring Boot 프로젝트를 시작했을 때, `build.gradle`의 의존성 설정은 대충 이랬다.

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'com.mysql:mysql-connector-j'
    implementation 'org.projectlombok:lombok'
    implementation 'org.springframework.boot:spring-boot-starter-test'
}
```

잘 동작했다. 문제가 없었다. 그래서 그냥 썼다.

그러다가 팀에서 코드 리뷰를 받으면서 처음으로 지적을 받았다.

> "JDBC 드라이버는 `runtimeOnly`로 두는 게 맞지 않나요?"
> "Lombok은 `compileOnly` + `annotationProcessor`로 선언해야 해요."
> "테스트 라이브러리를 왜 `implementation`에 두셨나요?"

솔직히 그때는 "어차피 되는데 왜요?" 싶었다.

근데 제대로 이해하고 나니, 의존성 설정은 단순히 "동작하면 OK" 문제가 아니었다.
**빌드 속도, 모듈 캡슐화, 런타임 안전성**에 직접 영향을 주는 설계 결정이었다.

---

## 1. 클래스패스가 뭔지부터 이해하자

Gradle 의존성을 제대로 이해하려면 **클래스패스** 개념이 먼저다.

Java 빌드는 크게 두 단계로 나뉜다.

```
컴파일 단계 → .java → .class
런타임 단계 → .class → 실행
```

각 단계에서 JVM/컴파일러가 참조할 수 있는 클래스 목록을 **클래스패스**라고 부른다.

- **컴파일 클래스패스**: `javac`가 소스를 컴파일할 때 참조하는 의존성
- **런타임 클래스패스**: 애플리케이션이 실제로 실행될 때 필요한 의존성

Gradle의 의존성 설정은 결국 **"이 라이브러리를 어느 클래스패스에 넣을 것인가"**를 선언하는 것이다.

---

## 2. 네 가지 설정, 각각 언제 쓰는가

### 2-1. implementation – 가장 기본, 대부분의 의존성

```gradle
implementation 'org.springframework.boot:spring-boot-starter-web'
```

**컴파일 + 런타임** 모두에 포함된다.
"내 코드가 직접 사용하는 라이브러리"에 쓰는 게 기본 용도다.

중요한 특징이 하나 있다. **의존성이 외부로 전파되지 않는다.**

멀티 모듈 프로젝트에서 `A → B → C` 관계를 생각해보자.
B가 C를 `implementation`으로 선언하면, A는 C를 직접 참조할 수 없다.
B 내부에서만 사용하는 의존성을 외부에 숨기는 셈이다.

> 반대 개념인 `api`를 쓰면 C가 A에까지 전파된다. 라이브러리를 개발할 때, 외부 API에 그 타입이 노출되어야 할 경우에만 사용한다.

---

### 2-2. runtimeOnly – 실행할 때만 필요한 것들

```gradle
runtimeOnly 'com.mysql:mysql-connector-j'
```

**런타임에만** 포함되고, 컴파일 클래스패스에는 들어가지 않는다.

대표적인 사례가 **JDBC 드라이버**다.

코드에서는 `java.sql.Connection`, `javax.sql.DataSource` 같은 표준 인터페이스만 쓴다.
실제로 MySQL 드라이버 클래스를 직접 참조하지 않는다.
컴파일 시에는 드라이버가 없어도 되고, 실행할 때만 있으면 된다.

```java
// 이런 코드는 없다 (컴파일 시 MySQL 클래스 직접 참조)
com.mysql.cj.jdbc.Driver driver = new com.mysql.cj.jdbc.Driver();

// 이런 코드만 있다 (표준 인터페이스 사용)
DataSource dataSource = // Spring이 주입
Connection connection = dataSource.getConnection();
```

`runtimeOnly`로 두면 **컴파일 시 실수로 MySQL 특정 클래스를 직접 참조하는 코드를 방지**할 수 있다. 컴파일 에러가 나기 때문에.

---

### 2-3. testImplementation – 테스트 코드 전용

```gradle
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

**테스트 컴파일 + 테스트 런타임**에만 포함된다.
프로덕션 코드의 컴파일/런타임 클래스패스에는 들어가지 않는다.

JUnit, Mockito, Testcontainers 등은 여기에 선언한다.

테스트 라이브러리를 `implementation`으로 선언하면:
- 프로덕션 배포 jar에 테스트 코드 의존성이 포함될 수 있다.
- 빌드 결과물 크기가 불필요하게 커진다.
- 프로덕션 코드에서 실수로 테스트 유틸리티를 사용할 수 있게 된다.

---

### 2-4. annotationProcessor – 컴파일 때 코드를 생성하는 것들

```gradle
compileOnly 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok'
```

`annotationProcessor`는 컴파일 단계에서 어노테이션을 분석해서 코드를 생성하는 도구다.
컴파일이 끝나면 그 역할은 끝이고, 런타임에는 포함되지 않는다.

Lombok이 대표적이다. `@Getter`, `@Builder` 등의 어노테이션을 읽어서
컴파일 타임에 바이트코드를 생성한다. 런타임에 Lombok이 필요하지 않다.

그래서 `compileOnly`와 항상 함께 쓴다.

- `compileOnly`: 소스 코드에서 Lombok 어노테이션을 사용하려면 컴파일 클래스패스에는 있어야 한다.
- `annotationProcessor`: 컴파일 시 Lombok 프로세서를 실행하는 용도다.

```gradle
// 나쁜 예 - Lombok을 implementation으로
implementation 'org.projectlombok:lombok'  // 런타임 jar에 포함된다. 불필요하다.

// 좋은 예
compileOnly 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok'
```

---

## 3. 한눈에 비교

| 설정 | 컴파일 클래스패스 | 런타임 클래스패스 | 테스트 클래스패스 | 외부 전파 |
|------|:-----------:|:-----------:|:-----------:|:----:|
| `implementation` | O | O | O | X |
| `runtimeOnly` | X | O | X | X |
| `testImplementation` | X | X | O | X |
| `annotationProcessor` | 프로세서로만 | X | X | X |
| `compileOnly` | O | X | X | X |
| `api` | O | O | O | **O** |

---

## 4. 왜 구분해야 할까? – 빌드 속도와 캡슐화

### 빌드 성능

Gradle의 증분 빌드(Incremental Build)는 **컴파일 클래스패스가 변경되었을 때만 재컴파일**한다.

`runtimeOnly`로 선언된 의존성이 버전이 바뀌어도,
컴파일 클래스패스에 영향이 없으면 재컴파일이 발생하지 않는다.

모든 걸 `implementation`으로 두면 불필요한 재컴파일이 더 자주 일어난다.

### 의존성 캡슐화

멀티 모듈 프로젝트에서 `implementation`은 의존성을 모듈 내부에 가둔다.
외부 모듈이 내 내부 구현 라이브러리를 실수로 참조하는 걸 컴파일 단계에서 막아준다.

### 프로덕션 배포 안전성

테스트 라이브러리가 `implementation`으로 선언되면
프로덕션 jar에 포함될 수 있다. 배포 결과물이 커지고, 의도치 않은 코드가 포함된다.

---

## 5. 실전 예시

```gradle
dependencies {
    // 일반 애플리케이션 의존성
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

    // 런타임 전용 - 컴파일 시 직접 참조하지 않음
    runtimeOnly 'com.mysql:mysql-connector-j'

    // Lombok - 컴파일 시 코드 생성, 런타임 불필요
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // QueryDSL - 컴파일 시 Q클래스 생성
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jakarta'
    annotationProcessor 'jakarta.annotation:jakarta.annotation-api'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api'

    // 테스트 전용
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.testcontainers:mysql'
}
```

---

## 6. 마치며

"어차피 implementation으로 해도 잘 돌아가던데?"

맞다. 기능은 동작한다.
하지만 그건 마치 모든 변수를 `public`으로 선언해도 코드가 돌아가는 것과 같다.

의존성 설정은 **빌드 시스템에게 의도를 전달하는 행위**다.

- "이건 런타임에만 필요해" → `runtimeOnly`
- "이건 컴파일 때 코드 생성용이야, 런타임엔 없어도 돼" → `annotationProcessor` + `compileOnly`
- "이건 테스트 코드에서만 써" → `testImplementation`

의도를 명확히 선언할수록 Gradle은 더 똑똑하게 빌드하고,
나는 프로덕션 배포 결과물을 더 신뢰할 수 있게 된다.

중요한 건, **왜 이 설정인가**를 이해하는 것이다.
그걸 알면 새로운 라이브러리를 추가할 때도 망설임 없이 올바른 위치에 둘 수 있다.
