# CORS

## 정의
CORS(Cross-Origin Resource Sharing)는 다른 출처(Origin) 간 리소스 요청을 브라우저가 안전하게 허용/차단하는 정책입니다.
브라우저가 응답 헤더를 검사해 요청 가능 여부를 결정합니다.

## 특징
- 서버가 `Access-Control-Allow-Origin` 등 헤더로 허용 정책을 명시해야 합니다.
- 단순 요청이 아니면 사전 요청(Preflight, `OPTIONS`)이 먼저 발생합니다.
- 브라우저 보안 정책이므로 서버-서버 통신에는 직접 적용되지 않습니다.
- 인증 쿠키를 쓰는 경우 `Allow-Credentials`와 Origin 설정을 엄격히 맞춰야 합니다.

## 실제 사용 예시(사례)
- 프론트엔드(`https://app.example.com`)에서 API(`https://api.example.com`) 호출 시 CORS 허용 헤더를 설정합니다.
- Spring에서 `CorsConfigurationSource`로 허용 Origin/Method/Header를 지정합니다.

## 보조 개념 정리
- Origin: 스킴 + 호스트 + 포트 조합
- Preflight: 실제 요청 전에 브라우저가 보내는 사전 검증 요청
- Credentials: 쿠키/Authorization 헤더 등 인증 정보 포함 요청
