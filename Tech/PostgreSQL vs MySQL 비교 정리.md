# PostgreSQL vs MySQL 비교 정리 (학습/실무 겸용)

## 1. 문서 목적
이 문서는 PostgreSQL과 MySQL을 "기능 비교표" 수준이 아니라, **왜 성능/동시성/운영 결과가 달라지는지**를 이해하기 위해 작성했다. 특히 Spring Boot + JPA/JDBC 환경에서 실제로 맞닥뜨리는 문제(락 대기, 실행계획 흔들림, 스키마 변경 중 장애, JSON 인덱싱 실패)를 중심으로 정리한다.

핵심 메시지는 간단하다.
- MySQL과 PostgreSQL 모두 훌륭한 DB다.
- 차이는 "좋고 나쁨"이 아니라 **트래픽 패턴과 운영 난이도 곡선**에서 드러난다.
- 선택은 벤치마크 숫자 1개가 아니라, 동시성/DDL/복제/관측성까지 포함한 시스템 의사결정이어야 한다.

---

## 2. 한눈에 보는 결론
- **전형적 CRUD + 빠른 팀 온보딩 + 기존 MySQL 운영자산**이 있다면 MySQL이 유리하다.
- **복잡 질의, JSON 활용, 데이터 무결성/표준 SQL, 확장(예: GIS/벡터)**이 중요하면 PostgreSQL이 유리하다.
- 고트래픽 시스템에서는 둘 다 가능하지만, 장애 패턴이 다르다.
  - PostgreSQL: VACUUM/통계/쿼리플랜 관리 실패가 누적형 장애로 나타나기 쉽다.
  - MySQL: gap lock/metadata lock/복제 지연이 급성 장애로 체감되기 쉽다.

---

## 3. 내부 구조 차이: 왜 같은 SQL인데 체감이 다를까?

### 3.1 저장 엔진 관점
- PostgreSQL: 사실상 단일 통합 엔진에 가까운 구조(일관된 기능 모델).
- MySQL: 서버 + 스토리지 엔진 구조이며, 실무에서는 대부분 InnoDB 사용.

이 차이는 운영 시 "일관성 있는 동작" vs "엔진 옵션/버전에 따른 차이 관리"로 나타난다.

### 3.2 로그/복구 체계
- PostgreSQL: WAL(Write-Ahead Log) 중심. 물리 복제/논리 복제 생태계가 명확하다.
- MySQL(InnoDB): redo/undo + binlog 체계. 복제는 binlog 기반 설계(ROW/MIXED/STATEMENT 모드).

실무 포인트:
- 장애 복구(RTO/RPO) 시 백업 체계 + 로그 보존 정책을 같이 설계해야 한다.
- 단순 "마스터-레플리카"만으로는 장애 훈련이 부족하다. failover 자동화와 읽기 일관성 정책까지 필요하다.

---

## 4. 트랜잭션/MVCC 심화: 기본 개념을 넘어서

### 4.1 PostgreSQL MVCC 포인트
- UPDATE 시 새 row version(tuple)을 만든다.
- 오래된 버전 정리는 VACUUM이 담당한다.
- 장시간 트랜잭션이 있으면 dead tuple 정리가 지연되어 bloat가 증가할 수 있다.

운영에서 흔한 실수:
- autovacuum을 기본값으로 방치하고, 대용량 테이블에서 freeze/vacuum 지연을 늦게 발견.
- 결과적으로 인덱스/테이블 비대화 -> I/O 증가 -> 지연 악화.

### 4.2 MySQL(InnoDB) MVCC 포인트
- undo log 기반 snapshot read.
- purge thread가 오래된 undo를 정리.
- long transaction은 undo 누적과 purge 지연을 만들 수 있다.

운영에서 흔한 실수:
- "트랜잭션을 길게 잡아도 읽기니까 괜찮다"고 생각하는 것.
- 실제로는 purge 지연/히스토리 길이 증가/복제 지연과 연결될 수 있다.

---

## 5. 락/격리수준: 서비스 장애를 가장 많이 만드는 구간

### 5.1 기본 격리수준 차이
- PostgreSQL 기본: `READ COMMITTED`
- MySQL(InnoDB) 기본: `REPEATABLE READ`

둘 다 MVCC를 쓰지만 팬텀/범위 잠금 처리 방식이 달라서, 동일 코드라도 교착/대기 시간이 달라질 수 있다.

### 5.2 MySQL gap lock 체감 포인트
범위 조건 + 인덱스 상황에서 next-key lock이 예상보다 넓게 걸리면, 삽입/갱신이 연쇄 대기한다.
"간헐적 timeout"의 전형적 원인이다.

### 5.3 PostgreSQL에서 자주 보는 패턴
- `FOR UPDATE` 사용은 명시적이고 예측 가능하지만,
- 인덱스/조건이 부정확하면 lock wait가 빠르게 쌓인다.
- `SKIP LOCKED`를 잘 쓰면 큐성 워크로드 처리량이 크게 좋아진다.

---

## 6. 실행계획/통계: ORM이 숨겨주는 진짜 성능 이슈

### 6.1 PostgreSQL
- 통계 기반 플래너가 매우 강력하다.
- 부분 인덱스, 표현식 인덱스, Bitmap scan 등 전략이 다양하다.
- 대신 통계가 낡거나 데이터 분포가 바뀌면 플랜 변동성이 생긴다.

### 6.2 MySQL
- InnoDB + B-Tree 중심의 예측 가능한 플랜이 장점.
- 단, 특정 조건에서 옵티마이저가 비효율 플랜을 고르면 힌트/인덱스 조정이 필요할 수 있다.

공통 원칙:
- JPA 쿼리 튜닝은 "JPQL 문장"이 아니라 **실행 SQL + EXPLAIN ANALYZE(또는 EXPLAIN FORMAT)**로 해야 한다.

---

## 7. 인덱스/데이터 타입 고급 비교

### 7.1 PostgreSQL 강점
- 부분 인덱스(WHERE 조건부 인덱스)
- 표현식 인덱스(lower(email) 같은 함수 결과 인덱스)
- GIN/GiST/BRIN 등 워크로드별 특화 인덱스
- `jsonb` + GIN으로 고급 문서 검색

### 7.2 MySQL 강점/특성
- 범용 B-Tree 최적화가 익숙하고 운영자 풀이 넓다.
- JSON도 강력해졌지만, 복잡 JSON 조건은 generated column + 인덱스 전략을 함께 고민해야 하는 경우가 많다.

---

## 8. 복제/고가용성(HA) 관점 비교

### 8.1 PostgreSQL
- Streaming replication(물리) + logical replication(논리) 선택 폭이 넓다.
- read replica 확장과 특정 테이블 논리 복제 설계가 가능하다.

### 8.2 MySQL
- binlog + GTID 기반 운영이 일반적.
- 관리형 환경(RDS/Aurora 등)에서 운영 경험/도구 축적이 매우 풍부하다.

실무 체크포인트:
- "읽기 분산" 시 replica lag를 애플리케이션에서 어떻게 처리할지 정책이 필요하다.
- 예: 주문 직후 조회는 writer 강제, 일반 목록은 replica 허용.

---

## 9. 온라인 DDL/마이그레이션: 기능보다 더 중요한 운영 리스크

스키마 변경은 기능개발보다 장애를 더 잘 만든다.

- PostgreSQL
  - `CREATE INDEX CONCURRENTLY` 등 온라인에 가까운 작업 지원
  - 대용량 테이블 변경 시 트랜잭션/락 전략을 명시적으로 설계해야 함

- MySQL
  - Online DDL(`ALGORITHM=INPLACE/INSTANT`) 지원 범위를 버전별로 확인해야 함
  - metadata lock 때문에 "변경은 되었는데 서비스가 멈춘" 상황이 발생 가능

권장:
- Flyway/Liquibase로 변경 이력 관리
- 대용량 DDL은 점진 배포 + 사전 리허설 + 롤백 시나리오 필수

---

## 10. Spring 중심 실전 예시

### 10.1 프로파일 기반 DB 분기
```yaml
# application-postgres.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/app?reWriteBatchedInserts=true
    username: app
    password: app
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
  hikari:
    maximum-pool-size: 20
```

```yaml
# application-mysql.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/app?useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true
    username: app
    password: app
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
  hikari:
    maximum-pool-size: 20
```

핵심:
- 배치 삽입 최적화 옵션(`reWriteBatchedInserts`, `rewriteBatchedStatements`)은 체감 성능 차이가 크다.

### 10.2 동시성 제어 예시 (`FOR UPDATE SKIP LOCKED`)
```java
@Repository
@RequiredArgsConstructor
public class JobQueueRepository {
    private final NamedParameterJdbcTemplate jdbc;

    public List<Long> fetchBatchForWorker(int size) {
        String sql = """
            SELECT id
            FROM job_queue
            WHERE status = 'READY'
            ORDER BY id
            FOR UPDATE SKIP LOCKED
            LIMIT :size
            """;

        return jdbc.query(
            sql,
            Map.of("size", size),
            (rs, rowNum) -> rs.getLong("id")
        );
    }
}
```

`SKIP LOCKED`는 다중 워커 큐 처리에서 lock wait 폭발을 줄이는 데 효과적이다.

### 10.3 DB별 UPSERT 분리 구현
```java
public interface MemberCommandRepository {
    void upsert(long id, String email);
}

@Repository
@Profile("postgres")
@RequiredArgsConstructor
class PostgresMemberCommandRepository implements MemberCommandRepository {
    private final NamedParameterJdbcTemplate jdbc;

    @Override
    public void upsert(long id, String email) {
        String sql = """
            INSERT INTO member(id, email)
            VALUES (:id, :email)
            ON CONFLICT (id)
            DO UPDATE SET email = EXCLUDED.email
            """;
        jdbc.update(sql, Map.of("id", id, "email", email));
    }
}

@Repository
@Profile("mysql")
@RequiredArgsConstructor
class MysqlMemberCommandRepository implements MemberCommandRepository {
    private final NamedParameterJdbcTemplate jdbc;

    @Override
    public void upsert(long id, String email) {
        String sql = """
            INSERT INTO member(id, email)
            VALUES (:id, :email)
            ON DUPLICATE KEY UPDATE email = VALUES(email)
            """;
        jdbc.update(sql, Map.of("id", id, "email", email));
    }
}
```

### 10.4 JSON 조회 + 인덱스 전략
```java
public interface EventLogQueryRepository extends JpaRepository<EventLog, Long> {

    // PostgreSQL: payload->>'type'
    @Query(value = "SELECT * FROM event_log WHERE payload ->> 'type' = :type", nativeQuery = true)
    List<EventLog> findTypePostgres(@Param("type") String type);

    // MySQL: JSON_EXTRACT
    @Query(value = "SELECT * FROM event_log WHERE JSON_UNQUOTE(JSON_EXTRACT(payload, '$.type')) = :type", nativeQuery = true)
    List<EventLog> findTypeMySql(@Param("type") String type);
}
```

DDL 예시(개념):
- PostgreSQL: `CREATE INDEX idx_event_type ON event_log USING GIN (payload jsonb_path_ops);`
- MySQL: generated column으로 추출 후 인덱스 생성 전략 검토

---

## 11. 실무에서 잘 모르는 포인트 (중요)

### 11.1 "DB가 느린가? 애플리케이션이 느린가?"
많은 경우 DB보다 커넥션 풀/트랜잭션 경계/직렬화 비용이 먼저 병목이다.
- Hikari pool을 CPU 코어 대비 과도하게 키우면 오히려 지연이 상승할 수 있다.
- 읽기 API인데 트랜잭션을 길게 열어 MVCC 정리를 방해하는 경우가 많다.

### 11.2 격리수준은 기능 스위치가 아니라 비용 스위치
- 무결성 요구가 높은 기능만 높은 격리수준을 선택해야 한다.
- 전체 서비스 기본을 높은 격리수준으로 올리면 처리량이 급락한다.

### 11.3 "EXPLAIN만 보고 끝"은 위험
- 실제 실행시간/버퍼 접근/락 대기까지 봐야 한다.
- 느린 쿼리는 SQL 하나의 문제가 아니라 통계/인덱스/트랜잭션 패턴의 합성 결과다.

### 11.4 마이그레이션 비용은 과소평가되기 쉽다
- 함수/문법 차이, 문자열 정렬(collation), timestamp/timezone 처리 차이는 데이터 품질 이슈로 이어진다.

---

## 12. 선택 의사결정 프레임워크

아래 질문에 "예"가 많은 쪽을 우선 검토한다.

### PostgreSQL 우선 검토 신호
- JSON 문서 질의가 핵심인가?
- 부분 인덱스/표현식 인덱스 같은 고급 인덱싱이 필요한가?
- 향후 pgvector/PostGIS 등 확장을 계획하는가?
- 복잡한 리포팅/분석성 SQL 비중이 높은가?

### MySQL 우선 검토 신호
- 전형적 CRUD 트랜잭션이 대부분인가?
- 운영팀의 MySQL 경험과 자동화 자산이 충분한가?
- 생태계/호스팅 도구/레퍼런스가 MySQL 중심인가?

최종 결정은 반드시 다음 3개를 함께 비교해야 한다.
1. 동일 스키마 + 동일 데이터셋의 부하 테스트
2. 장애 주입(락 경합, replica lag, 장기 트랜잭션) 테스트
3. DDL 변경 리허설(온라인 인덱스 생성, 컬럼 추가) 테스트

---

## 13. 학습/검증 실험 시나리오 (추천)
1. 1천만 건 데이터 준비 후 동일 인덱스 적용
2. API 70%(조회) / 20%(갱신) / 10%(집계) 혼합 부하
3. P50/P95/P99, lock wait, deadlock, slow query 수집
4. 장기 트랜잭션 1개 주입 후 성능/지연 변화 관찰
5. 온라인 DDL 수행 중 API 지연 비교

이 실험을 하면 "블로그 요약"이 아닌 실제 선택 근거가 생긴다.

---

## 14. 공식 문서 (반드시 원문 확인)
- PostgreSQL Docs: [https://www.postgresql.org/docs/](https://www.postgresql.org/docs/)
- PostgreSQL MVCC: [https://www.postgresql.org/docs/current/mvcc.html](https://www.postgresql.org/docs/current/mvcc.html)
- PostgreSQL Transaction Isolation: [https://www.postgresql.org/docs/current/transaction-iso.html](https://www.postgresql.org/docs/current/transaction-iso.html)
- PostgreSQL VACUUM: [https://www.postgresql.org/docs/current/routine-vacuuming.html](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- PostgreSQL JSON Types: [https://www.postgresql.org/docs/current/datatype-json.html](https://www.postgresql.org/docs/current/datatype-json.html)
- MySQL 8.0 Reference Manual: [https://dev.mysql.com/doc/refman/8.0/en/](https://dev.mysql.com/doc/refman/8.0/en/)
- MySQL InnoDB Multi-Versioning: [https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)
- MySQL InnoDB Locking: [https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- MySQL Online DDL: [https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl.html)
- MySQL JSON Functions: [https://dev.mysql.com/doc/refman/8.0/en/json-function-reference.html](https://dev.mysql.com/doc/refman/8.0/en/json-function-reference.html)
- Spring Data JPA: [https://docs.spring.io/spring-data/jpa/reference/](https://docs.spring.io/spring-data/jpa/reference/)
- Hibernate ORM Guide: [https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)

---

## 15. 결론
PostgreSQL vs MySQL은 "누가 더 빠른가"보다 **어떤 실패를 더 잘 통제할 수 있는가**의 문제다. Spring 애플리케이션 관점에서 핵심은 DB 선택 자체보다, 트랜잭션 경계/락 전략/인덱스 설계/DDL 운영 전략을 함께 설계하는 것이다. 이 네 가지를 못 맞추면 어떤 DB를 선택해도 병목은 다시 생긴다.
