# Elasticsearch Query DSL 기초

## 정의
Query DSL은 JSON 구조로 검색 조건을 표현하는 Elasticsearch 쿼리 언어입니다.
정확 일치, 전문 검색, 필터, 집계를 조합해 복합 검색을 구성합니다.

## 특징
- `must`, `should`, `filter` 조합으로 조건 의미를 세밀히 제어합니다.
- `filter`는 점수 계산을 생략해 성능에 유리합니다.
- 복합 쿼리는 가독성이 떨어질 수 있어 템플릿화가 유용합니다.

## 실제 사용 예시(사례)
- 키워드 검색은 `match` + `should`로 관련도 기반 결과를 구성합니다.
- 날짜 범위/상태값 조건은 `filter`로 분리해 점수 왜곡을 줄입니다.

## 보조 개념 정리
- Relevance Score: 문서 검색 관련도 점수
- Bool Query: 조건 조합을 위한 핵심 쿼리 구조
- Aggregation: 검색 결과 통계/버킷 분석 기능
