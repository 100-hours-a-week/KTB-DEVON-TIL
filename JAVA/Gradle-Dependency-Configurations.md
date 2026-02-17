# Gradle 의존성 설정: annotationProcessor, implementation, runtimeOnly, testImplementation

이 문서는 Gradle(Java 플러그인 기준)에서 자주 쓰이는 의존성 설정인 `annotationProcessor`, `implementation`, `runtimeOnly`, `testImplementation`의 차이를 정리한다. 각각이 **어떤 클래스패스에 포함되는지**, **빌드 단계에서 어떤 용도인지**, **모듈 공개 API에 영향을 주는지**를 중심으로 설명한다.

## 한 줄 요약
- `implementation`: **컴파일 + 런타임**에 필요하며, **내 모듈의 공개 API로 전파되지 않는** 일반 의존성
- `runtimeOnly`: **런타임에만** 필요한 의존성(컴파일 시에는 불필요)
- `testImplementation`: **테스트 코드의 컴파일 + 런타임**에 필요한 의존성
- `annotationProcessor`: **어노테이션 프로세서 실행용** 의존성(컴파일 단계에서만 사용)

## 클래스패스 관점 비교

| 설정 이름 | main 컴파일 | main 런타임 | test 컴파일 | test 런타임 | 공개 API 전파 |
| --- | --- | --- | --- | --- | --- |
| `implementation` | 포함 | 포함 | 포함 | 포함 | 아니오 |
| `runtimeOnly` | 제외 | 포함 | 제외(필요 시 testRuntimeOnly로 별도) | 포함(간접적으로) | 아니오 |
| `testImplementation` | 제외 | 제외 | 포함 | 포함 | 아니오 |
| `annotationProcessor` | 컴파일 시 프로세서로만 사용 | 제외 | 제외(필요 시 testAnnotationProcessor) | 제외 | 아니오 |

> 참고: 표는 기본적인 Java 프로젝트 기준이며, `testImplementation`은 테스트 소스셋에만 적용된다. 어노테이션 프로세서가 테스트에서도 필요하면 `testAnnotationProcessor`를 추가한다.

## 각각의 상세 설명

### 1) `implementation`
- **대상:** 일반적인 애플리케이션/라이브러리 의존성
- **사용 시점:** 컴파일 + 런타임
- **전파 여부:** 상위 모듈의 `compileClasspath`에 **노출되지 않음**
- **의미:** API에 직접 노출되지 않는 내부 구현 의존성에 적합

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

### 2) `runtimeOnly`
- **대상:** 컴파일에는 불필요하지만 실행 시 필요한 라이브러리
- **사용 시점:** 런타임
- **전파 여부:** 상위 모듈에 **노출되지 않음**
- **대표 사례:** JDBC 드라이버, 로깅 구현체 등

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    runtimeOnly 'com.mysql:mysql-connector-j'
}
```

### 3) `testImplementation`
- **대상:** 테스트 코드에서만 필요한 의존성
- **사용 시점:** 테스트 컴파일 + 테스트 런타임
- **전파 여부:** 본 모듈 또는 상위 모듈에 **영향 없음**
- **대표 사례:** JUnit, Mockito, Testcontainers 등

```gradle
dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter'
}
```

### 4) `annotationProcessor`
- **대상:** 컴파일 단계에서 어노테이션 프로세서를 실행하기 위한 의존성
- **사용 시점:** 컴파일 시점에만 사용, 결과물(생성 코드/메타데이터)을 만듦
- **전파 여부:** 상위 모듈에 **노출되지 않음**
- **대표 사례:** Lombok, MapStruct, QueryDSL APT 등

```gradle
dependencies {
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}
```

> 주의: Lombok처럼 컴파일 시 어노테이션 프로세서를 쓰는 라이브러리는 보통 `compileOnly`와 `annotationProcessor`를 함께 추가한다.

## 자주 헷갈리는 포인트

1. `implementation` vs `api`
- `implementation`은 의존성을 **외부에 숨긴다**.
- `api`는 상위 모듈의 `compileClasspath`에 **노출**되므로, 라이브러리 API에 직접 노출되는 타입이 있을 때만 사용한다.

2. `runtimeOnly`는 컴파일이 되지 않는다
- 코드에서 해당 라이브러리의 클래스를 직접 참조하면 **컴파일 에러**가 발생한다.

3. `annotationProcessor`는 런타임에 포함되지 않는다
- 런타임에 필요한 기능이면 `implementation` 또는 `runtimeOnly`로 별도 추가해야 한다.

## 실전 의존성 예시

```gradle
dependencies {
    // 일반 의존성
    implementation 'org.springframework.boot:spring-boot-starter-web'

    // 런타임 전용
    runtimeOnly 'com.mysql:mysql-connector-j'

    // 어노테이션 프로세서
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // 테스트
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

## 정리
- **implementation**: 대부분의 일반 의존성
- **runtimeOnly**: 실행 시 필요한 구현체/드라이버
- **testImplementation**: 테스트 코드 전용
- **annotationProcessor**: 컴파일 타임 코드 생성기

이 기준을 바탕으로 의존성을 분리하면 빌드 속도와 의존성 노출을 모두 관리하기 쉬워진다.
