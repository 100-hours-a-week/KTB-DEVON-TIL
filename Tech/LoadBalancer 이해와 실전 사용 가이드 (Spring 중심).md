# LoadBalancer 이해와 실전 사용 가이드 (Spring 중심)

## 1. Load Balancer를 왜 쓰는가
로드밸런서(Load Balancer)는 여러 서버 인스턴스에 트래픽을 분산해 **가용성(HA)**, **확장성(Scalability)**, **운영 안정성**을 확보하는 핵심 컴포넌트다. 서버를 1대에서 N대로 늘리는 순간, 단순 라우팅을 넘어서 헬스체크, 재시도, 연결 종료 처리, TLS 종료, 관측(로그/메트릭)까지 함께 다뤄야 한다. 이때 LB가 없으면 장애 전파가 빠르고, 특정 인스턴스로 부하가 편중되며, 배포 중(롤링/블루그린) 트래픽 제어가 어려워진다.

현업에서는 보통 다음 3가지 패턴을 조합한다.
- Edge LB: 외부 트래픽 진입점(ALB/Nginx/Ingress)
- East-West LB: 서비스 간 내부 호출 분산(Service Mesh/Envoy/Spring Cloud LoadBalancer)
- Client-side LB: 애플리케이션이 대상 인스턴스를 선택(Spring Cloud LoadBalancer)

## 2. L4와 L7의 차이

### L4 Load Balancing (Transport Layer)
- TCP/UDP 연결 단위로 분산한다.
- HTTP 헤더/경로를 해석하지 않는다.
- 일반적으로 성능 오버헤드가 낮고 단순하다.
- 예: DB 프록시, Redis, 게임/실시간 TCP, TLS 패스스루

대표 분산 기준
- `round robin`, `least connections`, `source hash`

### L7 Load Balancing (Application Layer)
- HTTP/HTTPS 요청을 이해하고, URL/헤더/쿠키/호스트 기반 라우팅이 가능하다.
- TLS 종료(Termination), 인증 연계, 경로 재작성, 캐시/압축, 세밀한 재시도 정책이 가능하다.
- 기능이 풍부한 대신 설정 복잡도와 CPU 비용이 증가할 수 있다.

대표 기능
- Path/Host 기반 라우팅
- Sticky session(쿠키/IP)
- Canary/Weighted routing
- 상세 헬스체크(예: `/healthz`)

실무 포인트: "L4가 항상 빠르다"보다는, **요구 기능과 장애 모델**에 맞춰 선택해야 한다. 예를 들어 API Gateway 성격이면 L7이 사실상 필수다.

## 3. 트래픽 분산 알고리즘과 함정
- `Round Robin`: 단순하고 예측 가능. 인스턴스 성능 차이가 크면 불리.
- `Weighted RR`: 인스턴스 스펙 차이를 가중치로 보정.
- `Least Connections`: 장기 연결이 많은 워크로드에 유리.
- `Hash(IP/URI/Header)`: 캐시 적중/세션 고정에 유리. 노드 증감 시 쏠림 가능.
- `Random + P2C(power of two choices)`: 대규모에서 분산 균형이 좋다.

많이 놓치는 문제
- 재시도 폭주(retry storm): LB 재시도 + 앱 재시도가 중첩되면 장애를 증폭시킨다.
- 느린 시작 미흡(slow start 없음): 새 인스턴스가 준비 전 과부하를 받는다.
- 연결 종료 처리 미흡(connection draining 없음): 배포 시 in-flight 요청 손실.
- 타임아웃 불일치: LB timeout < 앱 timeout이면 정상 요청도 502/504가 된다.

## 4. 헬스체크/드레이닝/세션 전략
- Active health check: LB가 주기적으로 엔드포인트를 호출해 생존 여부 판단.
- Passive health check: 실제 요청 실패율로 비정상 서버를 감지.
- Connection draining(=graceful shutdown): 제거 대상 인스턴스에 신규 요청을 중단하고 기존 연결만 마무리.
- Session persistence(sticky): 로그인 세션이 서버 메모리에 있는 구조면 필요할 수 있으나, 분산 캐시/무상태 JWT 구조에서는 의존도를 낮추는 것이 바람직하다.

권장 원칙
- 세션 상태는 서버 메모리보다 외부 저장소(Redis 등)로 분리
- LB와 애플리케이션의 timeout/retry/circuit-breaker를 일관되게 설계
- 배포 시 preStop + readiness/liveness + draining을 함께 구성

## 5. 주요 로드밸런서 비교

### Nginx
장점
- 사실상 표준에 가까운 레퍼런스, 생태계가 크다.
- HTTP(L7)뿐 아니라 stream 모듈로 TCP/UDP(L4)도 지원한다.
- `least_conn`, `ip_hash`, `hash` 등 다양한 upstream 정책 지원.

주의점
- OSS 기준으로 고급 기능(예: 일부 고급 active health check)은 제약이 있다.
- 설정이 유연한 만큼 운영 표준화가 필요하다.

예시 (L7)
```nginx
upstream chat_backend {
    least_conn;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=10s;
    server 10.0.1.12:8080 max_fails=3 fail_timeout=10s;
}

server {
    listen 80;
    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://chat_backend;
    }
}
```

### Caddy
장점
- 설정이 간결하고 자동 HTTPS가 매우 강력하다.
- `reverse_proxy`에서 LB 정책, active/passive health checks, retry 설정을 비교적 쉽게 제공.
- 개발/소규모~중규모 운영에서 생산성이 높다.

주의점
- 조직에서 세밀한 고급 프록시 정책이 매우 많으면 학습/검증 체계가 필요하다.

예시
```caddy
api.example.com {
    reverse_proxy 10.0.1.11:8080 10.0.1.12:8080 {
        lb_policy least_conn
        health_uri /actuator/health/readiness
        fail_duration 30s
        max_fails 3
        lb_try_duration 5s
    }
}
```

### HAProxy
장점
- 고성능/저지연, 대규모 트래픽 운영 레퍼런스가 많다.
- L4/L7 모두 강력하며, 세밀한 timeout/queue/stick-table 제어가 가능하다.
- 장애 상황 제어(백엔드 격리, rate-limit, circuit-like protection)에 강하다.

주의점
- 러닝커브가 있고 운영자가 설정 의미를 정확히 이해해야 한다.

## 6. Spring에서의 실전 적용

### 6.1 Spring Cloud LoadBalancer (Client-side LB)
서비스 디스커버리(Eureka, Kubernetes DNS 등)와 함께 자주 쓴다.

```java
@Configuration
public class HttpClientConfig {

    @Bean
    @LoadBalanced
    WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class ChatHistoryClient {

    private final WebClient.Builder webClientBuilder;

    public String getLatest(String conversationId) {
        return webClientBuilder.build()
                .get()
                .uri("http://chat-service/api/v1/conversations/{id}/latest", conversationId)
                .retrieve()
                .bodyToMono(String.class)
                .block();
    }
}
```

핵심: URL의 `chat-service`는 고정 호스트가 아니라 서비스 ID다. LoadBalancer가 인스턴스 목록 중 하나를 선택해 호출한다.

### 6.2 Spring Cloud Gateway의 `lb://` 라우팅
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: chat-route
          uri: lb://chat-service
          predicates:
            - Path=/chat/**
```

API Gateway 계층에서 경로 기반 L7 라우팅 + 서비스 단위 분산을 한 번에 처리한다.

### 6.3 LB 뒤에서 원본 클라이언트 정보 보존
LB를 거치면 애플리케이션 입장에서는 LB IP만 보일 수 있다. `X-Forwarded-*` 처리가 필수다.

```yaml
server:
  forward-headers-strategy: framework
```

```java
@Component
public class ClientIpExtractor {

    public String extract(HttpServletRequest request) {
        String xff = request.getHeader("X-Forwarded-For");
        if (xff == null || xff.isBlank()) {
            return request.getRemoteAddr();
        }
        return xff.split(",")[0].trim();
    }
}
```

실무 주의: 신뢰할 수 있는 프록시 체인에서만 `X-Forwarded-*`를 신뢰해야 한다.

## 7. 일반적으로 잘 모르는 운영 포인트
- Outlier detection: 특정 인스턴스만 지연이 급증하면 자동 격리(Envoy 등)
- Slow start: 신규 인스턴스 트래픽을 점진적으로 증가
- PROXY protocol: L4에서도 원본 클라이언트 IP 전달
- HTTP/2, gRPC, WebSocket: 프로토콜별 keepalive/timeout 최적화가 필요
- 관측성: LB access log + upstream response time + 재시도 횟수를 반드시 함께 본다

## 8. 학습 순서 제안
1. L4/L7 개념 + timeout/retry 기본기
2. Nginx 또는 Caddy로 단일 리버스 프록시 구성
3. 헬스체크/드레이닝/가중치 라우팅 실습
4. Spring Cloud LoadBalancer + Gateway `lb://` 연동
5. 장애 주입(지연/5xx) 후 재시도, 서킷브레이커, 타임아웃 상호작용 관찰

## 9. 공식 문서 링크 (Primary Sources)
- [Spring Cloud LoadBalancer](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)
- [Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/reference/index.html)
- [Nginx HTTP Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [Nginx TCP/UDP Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/tcp-udp-load-balancer/)
- [Nginx upstream 모듈(알고리즘/설정)](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [Caddy `reverse_proxy` (LB/health/retry)](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [HAProxy load balancing 개요](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/load-balancing/)
- [HAProxy health checks](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/health-checks/)
- [Envoy load balancing 아키텍처](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers)
