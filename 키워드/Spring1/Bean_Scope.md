# Bean Scope

## 정의
Bean Scope는 Spring 컨테이너에서 Bean 인스턴스를 어떤 범위와 생명주기로 관리할지 정의하는 설정입니다.
기본은 싱글톤이며, 웹 환경에서는 request/session 같은 스코프를 사용할 수 있습니다.

## 특징
- singleton: 컨테이너당 하나의 인스턴스를 재사용합니다.
- prototype: 요청할 때마다 새 인스턴스를 생성합니다.
- request/session/application: 웹 요청/세션/서블릿 컨텍스트 단위로 관리합니다.
- 잘못된 스코프 조합은 동시성 문제나 메모리 누수로 이어질 수 있습니다.

## 실제 사용 예시(사례)
- 상태 없는 서비스는 `singleton`으로 운영합니다.
- 사용자 요청 단위 컨텍스트 객체는 `request` 스코프로 관리합니다.

## 보조 개념 정리
- Thread Safety: 여러 스레드에서 동시에 접근해도 안전한 성질
- Scoped Proxy: 스코프가 다른 Bean 주입 시 프록시로 연결하는 방식
- Lifecycle Callback: 초기화/소멸 시점 실행 훅
