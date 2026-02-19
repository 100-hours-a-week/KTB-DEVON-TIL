# OAuth2

## 정의
OAuth2는 사용자 비밀번호를 직접 공유하지 않고, 권한 위임 토큰으로 서드파티 접근을 허용하는 인증/인가 프레임워크입니다.
소셜 로그인과 API 위임 권한에 널리 사용됩니다.

## 특징
- Resource Owner, Client, Authorization Server, Resource Server 역할이 분리됩니다.
- Authorization Code Grant가 웹 서비스에서 가장 일반적으로 사용됩니다.
- 액세스 토큰 만료와 리프레시 토큰 재발급 정책이 핵심입니다.

## 실제 사용 예시(사례)
- 구글 로그인에서 사용자가 동의하면 서버가 인가 코드를 받아 토큰으로 교환합니다.
- 발급된 토큰으로 사용자 프로필 API를 호출합니다.

## 보조 개념 정리
- Grant Type: 토큰 발급 흐름 유형
- Redirect URI: 인가 응답이 반환되는 콜백 주소
- Scope: 위임 권한 범위
