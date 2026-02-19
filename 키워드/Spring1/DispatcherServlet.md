# DispatcherServlet

## 정의
DispatcherServlet은 Spring MVC의 프론트 컨트롤러로, 모든 HTTP 요청을 받아 적절한 핸들러로 라우팅하고 응답을 조립합니다.
MVC 요청 처리 흐름의 중심 진입점입니다.

## 특징
- HandlerMapping으로 요청 URL에 맞는 핸들러를 찾습니다.
- HandlerAdapter로 실제 컨트롤러 메서드를 실행합니다.
- ViewResolver 또는 HttpMessageConverter를 통해 응답을 생성합니다.

## 실제 사용 예시(사례)
- `@RestController` 메서드 반환 객체를 JSON으로 직렬화해 응답합니다.
- 예외 발생 시 `@ControllerAdvice`와 연계해 표준 에러 응답을 반환합니다.

## 보조 개념 정리
- Front Controller: 요청 진입점을 단일화하는 패턴
- HandlerMapping: 요청-핸들러 매핑 컴포넌트
- HttpMessageConverter: 객체와 HTTP 본문 변환기
