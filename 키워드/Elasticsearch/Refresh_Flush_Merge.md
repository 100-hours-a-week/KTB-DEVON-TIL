# Refresh, Flush, Merge

## 정의
Refresh는 최근 색인 데이터를 검색 가능하게 여는 작업이고, Flush는 트랜잭션 로그를 디스크에 반영하는 작업이며, Merge는 세그먼트를 병합하는 백그라운드 작업입니다.
쓰기-조회 지연과 디스크 I/O에 직접 영향합니다.

## 특징
- refresh 주기가 짧을수록 검색 반영은 빠르지만 성능 비용이 증가합니다.
- flush는 durability 관점의 안정성을 높입니다.
- merge는 검색 효율을 높이지만 순간 I/O 부하를 유발할 수 있습니다.

## 실제 사용 예시(사례)
- 실시간 대시보드는 refresh interval을 짧게 유지합니다.
- 대량 적재 배치 중에는 refresh를 완화하고 완료 후 수동 refresh를 수행합니다.

## 보조 개념 정리
- Segment: Lucene 색인의 기본 저장 단위
- Translog: 색인 작업 로그
- Near Real-Time: 즉시가 아닌 거의 실시간 검색 반영 특성
