# Spring Security CORS, 설정했는데 왜 여전히 에러가 나지?

> "분명히 CORS 설정을 했는데, 브라우저는 여전히 CORS 에러를 뱉는다."

## 0. 이 글을 쓰게 된 계기

Spring Boot로 API를 만들고, React 프론트엔드와 붙이는 작업을 하면서
처음으로 CORS 에러를 만났다.

```
Access to fetch at 'http://localhost:8080/api/users' from origin 'http://localhost:3000'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present.
```

"간단하네" 싶어서 `WebMvcConfigurer`에 `addCorsMappings`을 추가했다.
그런데도 에러가 사라지지 않았다.

구글링을 해서 `@CrossOrigin`도 붙여봤다. 여전히 에러.

그때는 몰랐다. Spring Security가 그 사이에 끼어 있다는 걸.

CORS 설정은 **어디에 하느냐**와 **어느 순서에서 처리되느냐**가 함께 맞아야 한다.
이 글은 그 구조를 필터 체인을 따라가며 정리한 것이다.

---

## 1. CORS가 뭔지부터 정확히 이해하자

### 브라우저의 Same-Origin 정책

브라우저는 보안을 위해, **출처(Origin)가 다른 서버에 대한 요청을 기본적으로 차단**한다.

여기서 출처는 `프로토콜 + 도메인 + 포트`의 조합이다.

```
http://localhost:3000  (React 개발 서버)
http://localhost:8080  (Spring API 서버)
```

이 둘은 포트가 다르기 때문에 **다른 출처(Cross-Origin)**다.
브라우저 입장에서는 3000 포트의 페이지가 8080 포트 서버에 요청을 보내는 게 수상하다.

### CORS는 "허용 신호"를 주는 규약

CORS는 이 제약을 서버 측에서 **응답 헤더로 풀어주는 방법**이다.

서버가 응답에 이 헤더를 담으면, 브라우저는 해당 출처를 허용한 것으로 판단하고 요청을 통과시킨다.

```
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
```

### Preflight 요청 – 핵심 개념

`Authorization` 헤더를 쓰거나, `PUT`/`DELETE` 같은 메서드를 쓰는 경우
브라우저는 **실제 요청 전에 먼저 `OPTIONS` 메서드로 사전 확인 요청**을 보낸다.

이를 **Preflight 요청**이라고 한다.

```
OPTIONS /api/users HTTP/1.1
Origin: http://localhost:3000
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type
```

서버가 이 Preflight 요청에 CORS 허용 헤더를 돌려주지 못하면,
브라우저는 **실제 요청 자체를 보내지 않고** CORS 에러를 띄운다.

여기서 Spring Security가 문제를 일으킨다.

---

## 2. Spring Security 필터 체인과 CORS의 충돌

### 필터 체인이 뭔지

Spring Security는 단순한 하나의 컴포넌트가 아니다.
HTTP 요청이 컨트롤러에 도달하기 전에, **여러 개의 필터가 체인으로 연결되어** 요청을 처리한다.

```
HTTP 요청
  ↓
SecurityContextPersistenceFilter
  ↓
CsrfFilter
  ↓
UsernamePasswordAuthenticationFilter
  ↓
ExceptionTranslationFilter
  ↓
FilterSecurityInterceptor (인가 처리)
  ↓
DispatcherServlet → Controller
```

### Preflight 요청이 필터 체인에서 막히는 과정

CORS 설정을 `WebMvcConfigurer`에만 해두면 어떻게 될까?

`WebMvcConfigurer`의 CORS 처리는 **DispatcherServlet 레벨**, 즉 필터 체인을 다 통과한 이후에 동작한다.

Preflight `OPTIONS` 요청이 들어오면:

```
OPTIONS /api/users
  ↓
Spring Security 필터 체인 진입
  ↓
인증 확인 → 토큰 없음 → 401 반환 ← 여기서 막힘
  ↓
(DispatcherServlet까지 도달하지 못함)
  ↓
브라우저: CORS 에러
```

Spring Security는 Preflight 요청도 **보안 검사 대상**으로 본다.
인증 토큰이 없는 `OPTIONS` 요청은 401로 막힌다.

그러면 브라우저는 CORS 에러를 띄우고, 실제 요청은 아예 전송되지 않는다.

---

## 3. Spring에서 CORS를 처리하는 두 가지 방법

### 3-1. WebMvc 레벨 – DispatcherServlet 이후에 동작

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("http://localhost:3000")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true);
    }
}
```

Spring Security 없이 순수 Spring MVC만 쓸 때는 잘 동작한다.
하지만 **Spring Security를 함께 쓰면 Preflight가 필터 체인에서 먼저 막히기 때문에** 이 설정만으로는 부족하다.

### 3-2. Spring Security 레벨 – 필터 체인 초반에 동작

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .csrf(AbstractHttpConfigurer::disable)
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        );
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(List.of("http://localhost:3000"));
    configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    configuration.setAllowedHeaders(List.of("*"));
    configuration.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

`http.cors()`를 설정하면 Spring Security는 **필터 체인 최초 단계에서 CORS 처리를 먼저 수행**한다.
Preflight `OPTIONS` 요청이 인증 필터에 도달하기 전에 CORS 헤더와 함께 응답이 나간다.

**Spring Security를 쓴다면, CORS 설정은 반드시 Security 레벨에서 해야 한다.**

---

## 4. 자주 만나는 시나리오 세 가지

### 4-1. WebMvc에만 설정하고 Security에는 안 한 경우

**증상**: `addCorsMappings`로 설정했는데 계속 CORS 에러가 난다.

**원인**: Security 필터가 Preflight를 먼저 잡아서 401을 반환한다. WebMvc 레벨까지 도달하지 못한다.

**해결**: `http.cors()` + `CorsConfigurationSource` 추가.

---

### 4-2. OPTIONS 요청을 permitAll()로 열지 않은 경우

**증상**: `http.cors()`를 켰는데도 여전히 Preflight가 401로 막힌다.

`http.cors()`를 설정하면 대부분의 경우 Preflight가 자동으로 처리된다.
하지만 필터 설정이 복잡한 경우 `OPTIONS`를 명시적으로 허용해야 할 때가 있다.

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()  // Preflight 명시 허용
    .anyRequest().authenticated()
);
```

Preflight는 실제 비즈니스 요청이 아니다. 인증 검사 없이 통과시키는 게 맞다.

---

### 4-3. Origin/메서드/헤더 설정이 실제 요청과 맞지 않는 경우

**증상**: 특정 API나 특정 메서드만 CORS 에러가 난다.

**원인**: `CorsConfiguration`에 허용한 값이 실제 요청과 정확히 일치하지 않는다.

흔한 실수들:

```java
// allowedOrigins에 trailing slash가 있는 경우
configuration.setAllowedOrigins(List.of("http://localhost:3000/"));  // 틀림
configuration.setAllowedOrigins(List.of("http://localhost:3000"));   // 맞음

// allowCredentials(true)와 allowedOrigins("*") 조합 - 브라우저가 거부함
configuration.setAllowedOrigins(List.of("*"));
configuration.setAllowCredentials(true);  // 이 조합은 동작하지 않음

// allowedOriginPatterns를 써야 하는 경우
configuration.setAllowedOriginPatterns(List.of("*"));  // credentials와 함께 쓸 때
configuration.setAllowCredentials(true);
```

---

## 5. CORS 문제 디버깅 순서

CORS 에러가 났을 때 내가 확인하는 순서다.

### 1단계: OPTIONS 요청이 서버까지 오는지 확인

브라우저 개발자 도구 → Network 탭 → `OPTIONS` 요청 확인.
서버 로그에 `OPTIONS /api/...`가 찍히는지 본다.

로그가 아예 없다면 프록시나 게이트웨이에서 막힌 것일 수 있다.

### 2단계: Spring Security 디버그 로그 켜기

```yaml
logging:
  level:
    org.springframework.security: DEBUG
```

어느 필터에서 요청이 거부됐는지 로그로 확인할 수 있다.

### 3단계: `http.cors()` 설정 확인

Security 설정에 `http.cors()`가 있는지 확인한다.
없다면 WebMvc CORS 설정이 Security 필터 체인을 통과하지 못한다.

### 4단계: CorsConfigurationSource 내용 확인

허용 Origin, 메서드, 헤더가 실제 브라우저 요청과 정확히 일치하는지 비교한다.
개발/운영 환경마다 Origin이 다르다면 환경별로 분리해서 관리한다.

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();

    // 환경별 Origin 분리
    configuration.setAllowedOrigins(allowedOrigins);  // @Value로 주입
    configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    configuration.setAllowedHeaders(List.of("*"));
    configuration.setAllowCredentials(true);
    configuration.setMaxAge(3600L);  // Preflight 캐시 시간 (초)

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

---

## 6. 핵심 원칙 정리

### CORS 처리는 Security 레벨에서 한다

Spring Security를 쓰는 환경에서는 `WebMvcConfigurer#addCorsMappings`만으로는 부족하다.
반드시 `http.cors()` + `CorsConfigurationSource` 조합을 사용한다.

### Preflight는 인증 없이 통과시킨다

`OPTIONS` 요청은 실제 비즈니스 요청이 아니다.
`http.cors()`가 자동으로 처리하지만, 복잡한 설정에서는 `permitAll()`을 명시적으로 추가한다.

### credentials와 wildcard origin은 함께 쓸 수 없다

`allowCredentials(true)`를 쓸 때는 `allowedOrigins("*")` 대신
`allowedOriginPatterns("*")` 또는 구체적인 Origin을 명시한다.

---

## 마치며

CORS 에러는 처음 만나면 막막하다.
원인이 브라우저에서 나오는데, 정작 고쳐야 할 건 서버 설정이고,
Spring Security라는 중간 레이어가 더해지면 어디서 막히는지 찾기가 어렵다.

하지만 구조를 이해하면 단순해진다.

1. CORS는 서버가 응답 헤더로 주는 "허용 신호"다.
2. Preflight `OPTIONS` 요청은 실제 요청 전에 먼저 온다.
3. Spring Security 필터 체인이 Preflight를 먼저 잡기 때문에, CORS 처리는 **필터 체인 초반**에 있어야 한다.
4. 그게 바로 `http.cors()` + `CorsConfigurationSource`다.

중요한 건 **왜 이 순서여야 하는가**를 이해하는 것이다.
그러면 다음에 CORS 에러가 다시 나도, 어디서부터 확인해야 할지 막히지 않는다.
