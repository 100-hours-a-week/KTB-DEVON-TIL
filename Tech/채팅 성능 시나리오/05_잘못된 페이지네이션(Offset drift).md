# _5 잘못된 페이지네이션(Offset drift)

## 문제 정의
append-heavy 채팅에서 offset 기반 페이지를 캐싱하면 새 메시지 유입 때마다 페이지 경계가 밀려 캐시 재사용률이 떨어진다.

## 재현 조건
- offset 기반으로 history 조회
- 조회 중간에 지속적으로 새 메시지 유입
- 무한 스크롤 사용자 비율을 높여 반복 조회 유도

## 개선 전략
- cursor pagination(`before:{cursor}`) 전환
- immutable page key 설계
- 페이지 크기 고정(예: 20~30) + 100개 범위 캐시

## 검증 지표
- history hit ratio
- `reads/request`
- 무한 스크롤 구간 체감 응답 시간

## 포트폴리오 포인트
기능은 정상인데 비용과 캐시 효율만 망가지는 숨은 문제를 개선한 사례로 설득력이 높다.
