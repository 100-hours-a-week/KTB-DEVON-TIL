# CSRF

## 정의
CSRF(Cross-Site Request Forgery)는 사용자가 의도하지 않은 요청을 공격자가 브라우저를 통해 실행하게 만드는 공격입니다.
특히 쿠키 기반 인증 환경에서 방어가 중요합니다.

## 특징
- 사용자가 로그인한 세션을 악용해 요청이 실행될 수 있습니다.
- GET보다 상태 변경 요청(POST/PUT/DELETE)에서 위험이 큽니다.
- CSRF 토큰, SameSite, Origin/Referer 검증으로 방어합니다.

## 실제 사용 예시(사례)
- 폼 요청 시 서버가 발급한 CSRF 토큰을 함께 제출하도록 강제합니다.
- Spring Security에서 기본 CSRF 보호를 유지하고, API 특성에 맞게 예외 경로를 최소화합니다.

## 보조 개념 정리
- SameSite Cookie: 크로스 사이트 전송 제한 정책
- Origin 검증: 요청 출처를 확인해 위조 요청 차단
- 상태 변경 요청: 서버 데이터에 영향을 주는 요청
