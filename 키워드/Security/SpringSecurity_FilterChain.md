# Spring Security Filter Chain

## 정의
Spring Security Filter Chain은 HTTP 요청이 컨트롤러에 도달하기 전에 인증/인가/예외 처리를 수행하는 필터 실행 체인입니다.
보안 정책은 필터 순서와 설정으로 구성됩니다.

## 특징
- 요청은 여러 보안 필터를 순서대로 통과합니다.
- 인증 필터와 인가 필터의 역할이 분리됩니다.
- 필터 순서가 잘못되면 인증 누락, 중복 처리, 401/403 오동작이 발생합니다.
- 커스텀 JWT 필터를 추가할 때는 기존 필터 위치를 정확히 지정해야 합니다.

## 실제 사용 예시(사례)
- `UsernamePasswordAuthenticationFilter` 앞에 JWT 인증 필터를 배치합니다.
- 경로별로 `permitAll`, `hasRole` 규칙을 설정해 접근 제어를 분리합니다.

## 보조 개념 정리
- SecurityContext: 인증 정보를 저장하는 컨텍스트
- Authentication: 현재 사용자 인증 객체
- 401 vs 403: 인증 실패와 권한 부족의 구분
