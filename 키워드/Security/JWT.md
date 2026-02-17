# JWT

## 정의
JWT(JSON Web Token)는 사용자 인증/인가 정보를 안전하게 전달하기 위한 토큰 포맷입니다.
서버는 토큰의 서명을 검증해 위변조 여부를 확인하고, 토큰에 담긴 클레임으로 사용자 신원을 식별합니다.

## 특징
- 자체 포함(Self-contained) 구조라 세션 저장소 의존도를 줄일 수 있습니다.
- 기본 구조는 `Header.Payload.Signature`입니다.
- 만료 시간(`exp`)을 포함해 토큰 유효 기간을 제어합니다.
- 탈취 시 재사용 위험이 있으므로 만료/재발급/폐기 전략이 중요합니다.

## 실제 사용 예시(사례)
- 로그인 성공 시 Access Token(짧은 수명) + Refresh Token(긴 수명)을 발급합니다.
- API 서버는 Access Token을 검증하고, 만료 시 Refresh Token으로 재발급합니다.
- 로그아웃/강제 만료 시 Refresh Token 저장소를 폐기하거나 블랙리스트 정책을 적용합니다.

## 설계 포인트
- Access Token은 짧게, Refresh Token은 상대적으로 길게 설정
- 민감정보(비밀번호, 주민번호 등)는 Payload에 절대 저장하지 않음
- 서명 키 롤링(키 교체) 절차를 운영 문서로 준비
- HTTPS 강제 + 저장 위치(HttpOnly Cookie/보안 스토리지) 정책 명확화

## 보조 개념 정리
- 클레임(Claim): 토큰 안에 담기는 사용자/권한 정보
- JWS: 서명 기반 JWT 포맷
- 토큰 회전(Token Rotation): Refresh Token 재발급 시 이전 토큰을 폐기하는 전략
