# Spring AOP vs AspectJ

## 정의
Spring AOP는 런타임 프록시 기반 AOP이고, AspectJ는 컴파일/로드타임 위빙까지 지원하는 더 강력한 AOP 기술입니다.
적용 범위와 복잡도, 도입 비용이 다릅니다.

## 특징
- Spring AOP: 메서드 실행 조인포인트 중심, Spring Bean에 주로 적용
- AspectJ: 필드 접근/생성자/정적 초기화 등 더 넓은 조인포인트 지원
- Spring AOP는 도입이 쉽고 운영 부담이 낮습니다.
- AspectJ는 강력하지만 빌드/운영 복잡도가 증가할 수 있습니다.

## 실제 사용 예시(사례)
- 일반 서비스 로깅/트랜잭션은 Spring AOP로 충분한 경우가 많습니다.
- 생성자 호출 추적, 도메인 객체 단위 위빙이 필요하면 AspectJ를 검토합니다.

## 보조 개념 정리
- Load-time Weaving: 클래스 로딩 시점에 부가 기능을 결합하는 방식
- Compile-time Weaving: 컴파일 단계에서 위빙하는 방식
- 프록시 기반 제한: 프록시를 거치지 않는 호출에는 적용되지 않는 한계
