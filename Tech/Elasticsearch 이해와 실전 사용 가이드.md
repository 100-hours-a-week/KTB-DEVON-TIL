# Elasticsearch 깊이 있게 이해하기 (학습 노트, Spring 중심)

> 이 문서는 "검색 API를 일단 동작시키는 수준"에서 "왜 느려지고 왜 품질이 흔들리는지 설명할 수 있는 수준"으로 올라가기 위해 정리한 학습 노트다.  
> 단순 개념 나열 대신, 내가 실제로 헷갈렸던 포인트와 실무에서 자주 터지는 문제를 같이 기록했다.

## 1. 처음에 관점을 잘못 잡으면 계속 헤맨다

내가 처음 Elasticsearch(이하 ES)를 볼 때 가장 크게 오해한 건 두 가지였다.

1. ES는 DB 대체제가 아니라 **검색/집계 엔진**이다.
2. 검색 성능은 쿼리 튜닝보다 **인덱스 설계(매핑/분석기/샤드)**에서 먼저 결정된다.

실무 기준으로는 보통 이렇게 역할을 나눈다.
- RDB: 원본 데이터, 트랜잭션, 강한 정합성
- Elasticsearch: 검색/추천/집계/로그 분석, 조회 최적화 뷰

즉, "RDB에 저장하고 ES로 동기화"가 기본 패턴이다. 이걸 처음에 받아들이면 아키텍처 판단이 훨씬 단순해진다.

## 2. 내부 동작을 이해하면 튜닝이 쉬워진다

### 2.1 Lucene 기반 역색인(Inverted Index)

ES는 내부적으로 Lucene을 사용한다. 핵심은 역색인이다.
- 일반 DB 인덱스: 문서 -> 단어를 찾기 쉬움
- 역색인: 단어 -> 어떤 문서에 있는지 찾기 쉬움

검색어 "성능"을 입력하면, 역색인에서 "성능" 토큰의 posting list(문서 ID 목록)를 바로 찾고 연산한다. 그래서 전문검색이 빠르다.

### 2.2 쓰기 경로: translog, refresh, flush, merge

내가 가장 자주 헷갈렸던 부분이다.

- 문서 색인 요청
- in-memory buffer + translog 기록
- `refresh` 시점에 새 segment가 열려 검색 가능(near real-time)
- `flush` 시점에 commit + translog 정리
- background merge가 segment를 합쳐 검색 비용을 낮춤

중요 포인트:
- ES는 "쓰기 즉시 검색"이 아니다. **refresh 이후 검색 가능**이다.
- refresh를 너무 자주 강제하면 인덱싱 처리량이 떨어진다.
- merge는 백그라운드에서 디스크/CPU를 많이 쓸 수 있어, 대량 색인 시 지연 스파이크 원인이 된다.

### 2.3 읽기 경로: query phase + fetch phase

검색 요청은 대략 아래 흐름이다.
1. Coordinator 노드가 요청 수신
2. 각 샤드에서 query phase로 상위 문서 ID/score 계산
3. coordinator가 합쳐 상위 N개 결정
4. fetch phase로 실제 `_source`를 가져와 응답

즉, `size`를 무턱대고 크게 잡으면 fetch 비용이 급증한다.

## 3. Mapping/Analysis가 검색 품질의 대부분을 결정한다

### 3.1 `text`와 `keyword`는 용도가 완전히 다르다

- `text`: 분석기 적용, 전문검색용 (`match`, `multi_match`)
- `keyword`: 원문 그대로, 정확일치/정렬/집계 (`term`, `terms`, agg)

실무에서는 문자열을 보통 멀티필드로 둔다.
- `title`(text): 검색
- `title.keyword`(keyword): 정렬/집계

### 3.2 `object` vs `nested`를 헷갈리면 결과가 틀린다

배열 객체를 `object`로 두면 내부 필드가 평탄화되어 잘못된 매칭이 발생할 수 있다.
- 예: 한 문서에 `[{name: "kim", role:"admin"}, {name:"lee", role:"viewer"}]`
- `name=kim AND role=viewer`가 의도치 않게 true가 되는 문제

이런 케이스는 `nested`로 설계하고 `nested query`를 사용해야 정확하다.

### 3.3 `doc_values`와 `fielddata`

- 정렬/집계는 보통 `doc_values`(columnar storage)를 사용
- `text` 필드 집계를 위해 fielddata를 켜면 메모리 부담이 커서 위험

원칙:
- 집계/정렬이 필요한 문자열은 `keyword`로 별도 필드 생성
- `text`에 fielddata 활성화는 마지막 수단

### 3.4 Dynamic mapping은 편하지만 운영비를 만든다

초기 개발 속도는 빠르지만, 필드 폭증(mapping explosion)으로 힙/클러스터 상태를 망가뜨리기 쉽다.
- 인덱스 템플릿
- 명시적 mapping
- `index.mapping.total_fields.limit`
- ingestion 단계에서 불필요 필드 제거

이 4가지를 세트로 가져가야 안정적이다.

## 4. 한국어 검색 품질: 분석기 설계가 핵심

### 4.1 Analyzer 파이프라인

`char_filter -> tokenizer -> token_filter`

한국어는 형태소 경계가 영어보다 복잡해서, nori와 사용자 사전 품질이 검색 품질을 크게 좌우한다.

### 4.2 nori에서 내가 자주 놓친 포인트

- 복합명사 처리(`decompound_mode`)가 recall/precision에 영향
- 도메인 용어(서비스명, 신조어)는 사용자 사전에 추가해야 정확도 상승
- index analyzer와 search analyzer를 분리하면 품질 개선 여지가 큼

### 4.3 동의어(synonym) 주의

동의어는 단순히 많이 넣는다고 좋아지지 않는다.
- 동의어 확장으로 노이즈 증가 가능
- 다중 토큰 동의어는 `synonym_graph` 고려
- 검색 시점 확장(search-time)과 색인 시점 확장(index-time)은 운영 트레이드오프가 다름

## 5. Query DSL: 속도와 정확도를 같이 보는 법

### 5.1 Query context vs Filter context

- query context: `_score` 계산(느릴 수 있음)
- filter context: yes/no 판별 + 캐시 친화적(빠름)

실무 원칙:
- 검색어 relevance에 필요한 절만 `must`/`should`
- 상태/카테고리/권한/날짜 범위는 `filter`

### 5.2 BM25 점수 감각

ES 기본 점수는 BM25다. 수식 전체를 외우기보다 감각이 중요하다.
- term frequency가 높을수록 점수 증가
- 드문 단어일수록(IDF) 점수 증가
- 문서 길이가 길수록 길이 정규화로 패널티 가능

그래서 제목 가중치(`title^3`)를 주거나, 너무 긴 본문만으로 점수가 과대평가되지 않도록 튜닝한다.

### 5.3 내가 자주 쓰는 쿼리 조합

- `multi_match` + `minimum_should_match`
- `match_phrase` + `slop`
- `function_score`(최신성, 인기도 보정)
- `rank_feature`(사전 계산 점수 반영)

검색 품질은 결국 "텍스트 적합도 + 비즈니스 점수"를 섞는 문제다.

## 6. Aggregation은 분석 기능의 핵심이지만 비용이 크다

### 6.1 자주 쓰는 집계

- `terms`: 카테고리 TOP N
- `date_histogram`: 시간대별 추이
- `cardinality`: 유니크 수(근사값)
- `percentiles`: 지연 시간 분포
- `composite`: 대량 버킷 페이지네이션

### 6.2 실수 포인트

- 고카디널리티 필드에 무거운 `terms` 집계 남발
- `size` 과대 설정
- 전체 데이터 대상 집계를 실시간 API에서 바로 수행

대안:
- 사전 집계/배치 전처리
- 집계 전용 인덱스 분리
- 필요한 기간만 필터링

## 7. 성능 튜닝에서 진짜 효과 큰 항목

### 7.1 샤드 수

샤드는 많다고 좋은 게 아니다. 샤드마다 메모리/스케줄링/검색 오버헤드가 있다.
- 너무 큰 샤드: recovery/merge 부담
- 너무 작은 샤드가 너무 많음: coordination 비용 증가

처음부터 "성장 시나리오 + 롤오버 전략"으로 설계하는 게 안전하다.

### 7.2 Deep Pagination

`from + size`는 뒤 페이지로 갈수록 비싸다. 기본 윈도우 한계(`index.max_result_window`)도 있다.
- 사용자 탐색형 페이지: `search_after + PIT`
- 관리자 대량 추출: scroll 또는 비동기 배치

### 7.3 Bulk 색인

대량 입력은 단건 색인보다 bulk가 유리하다. 하지만 무조건 크게 보내면 안 된다.
- bulk 크기(문서 수/바이트) 점진 튜닝
- 429(too many requests) 재시도(backoff)
- refresh 정책 조정(`refresh_interval`)

### 7.4 캐시와 프로파일링

- Request cache: 동일 쿼리/집계 재사용
- Query cache: filter 맥락에서 유리
- Profile API: 느린 쿼리 병목 확인
- Explain API: 왜 이 문서가 상위인지 설명

## 8. 운영 관점: 장애를 줄이는 필수 패턴

### 8.1 Alias 기반 무중단 재색인

매핑 변경은 대부분 인플레이스가 불가능하다. 정석은 아래다.
1. 새 인덱스(`posts-v2`) 생성
2. `_reindex` 수행
3. write/read alias 스위칭
4. 구 인덱스 정리

이 패턴을 자동화하지 않으면 운영 중 매핑 변경이 매우 위험해진다.

### 8.2 ILM/Data Stream

로그/이벤트성 데이터는 data stream + ILM 조합이 운영 효율이 좋다.
- hot: 쓰기/조회
- warm: 저비용 저장
- cold/frozen: 장기 보관
- delete: 만료 삭제

### 8.3 Snapshot/Restore

노드 복제(replica)는 백업이 아니다. 진짜 백업은 snapshot이다.
- S3/오브젝트 스토리지 repo
- 주기적 snapshot 정책(SLM)
- 복구 리허설 필수

### 8.4 Hot shard와 불균형

특정 tenant/키로 트래픽이 몰리면 샤드 핫스팟이 생긴다.
- 라우팅 키 전략 재검토
- 인덱스 분할 전략
- 쓰기 분산/버킷화

## 9. Spring 중심 실전 예시

아래 예시는 "게시글 검색"을 기준으로, 실무에서 자주 필요한 요소를 넣었다.

### 9.1 의존성

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-elasticsearch'
}
```

### 9.2 문서 모델 (`text + keyword`, `nested`)

```java
package com.example.search;

import java.time.Instant;
import java.util.List;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.DateFormat;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;
import org.springframework.data.elasticsearch.annotations.InnerField;
import org.springframework.data.elasticsearch.annotations.MultiField;
import org.springframework.data.elasticsearch.annotations.Setting;

@Document(indexName = "posts-v1", createIndex = false)
@Setting(settingPath = "elasticsearch/posts-settings.json")
public class PostDocument {

    @Id
    private String id;

    @MultiField(
        mainField = @Field(type = FieldType.Text, analyzer = "ko_index_analyzer", searchAnalyzer = "ko_search_analyzer"),
        otherFields = {
            @InnerField(suffix = "keyword", type = FieldType.Keyword)
        }
    )
    private String title;

    @Field(type = FieldType.Text, analyzer = "ko_index_analyzer", searchAnalyzer = "ko_search_analyzer")
    private String content;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Nested)
    private List<TagInfo> tags;

    @Field(type = FieldType.Date, format = DateFormat.date_time)
    private Instant createdAt;

    protected PostDocument() {
    }

    public PostDocument(String id, String title, String content, String category, List<TagInfo> tags, Instant createdAt) {
        this.id = id;
        this.title = title;
        this.content = content;
        this.category = category;
        this.tags = tags;
        this.createdAt = createdAt;
    }

    public String getId() { return id; }
    public String getTitle() { return title; }
    public String getContent() { return content; }
    public String getCategory() { return category; }
    public List<TagInfo> getTags() { return tags; }
    public Instant getCreatedAt() { return createdAt; }

    public static class TagInfo {
        @Field(type = FieldType.Keyword)
        private String name;

        @Field(type = FieldType.Keyword)
        private String type;

        protected TagInfo() {
        }

        public TagInfo(String name, String type) {
            this.name = name;
            this.type = type;
        }

        public String getName() { return name; }
        public String getType() { return type; }
    }
}
```

### 9.3 분석기 설정 (`posts-settings.json`)

```json
{
  "index": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "refresh_interval": "1s",
    "mapping": {
      "total_fields": {
        "limit": "1500"
      }
    }
  },
  "analysis": {
    "filter": {
      "ko_synonyms": {
        "type": "synonym_graph",
        "synonyms": [
          "ai, 인공지능",
          "검색, 서치"
        ]
      }
    },
    "analyzer": {
      "ko_index_analyzer": {
        "type": "custom",
        "tokenizer": "nori_tokenizer",
        "filter": ["lowercase"]
      },
      "ko_search_analyzer": {
        "type": "custom",
        "tokenizer": "nori_tokenizer",
        "filter": ["lowercase", "ko_synonyms"]
      }
    }
  }
}
```

### 9.4 검색 서비스 (`query`와 `filter` 분리)

```java
package com.example.search;

import java.util.List;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.client.elc.NativeQueryBuilder;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.stereotype.Service;

@Service
public class PostSearchService {

    private final ElasticsearchOperations operations;

    public PostSearchService(ElasticsearchOperations operations) {
        this.operations = operations;
    }

    public List<PostDocument> search(String keyword, String category, int page, int size) {
        NativeQuery query = new NativeQueryBuilder()
            .withQuery(q -> q.bool(b -> b
                .must(m -> m.multiMatch(mm -> mm
                    .query(keyword)
                    .fields("title^3", "content")
                    .minimumShouldMatch("75%")
                ))
                .filter(f -> f.term(t -> t.field("category").value(category)))
            ))
            .withPageable(PageRequest.of(page, size))
            .build();

        return operations.search(query, PostDocument.class)
            .stream()
            .map(SearchHit::getContent)
            .toList();
    }
}
```

### 9.5 `search_after + PIT` (딥 페이지네이션)

Spring Data만으로도 가능하지만, PIT는 Java API Client를 직접 쓰면 제어가 명확하다.

```java
package com.example.search;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch._types.SortOrder;
import co.elastic.clients.elasticsearch.core.OpenPointInTimeResponse;
import co.elastic.clients.elasticsearch.core.SearchResponse;
import co.elastic.clients.elasticsearch.core.search.Hit;
import java.io.IOException;
import java.util.List;

public class PostSearchAfterExample {

    private final ElasticsearchClient client;

    public PostSearchAfterExample(ElasticsearchClient client) {
        this.client = client;
    }

    public List<Hit<PostDocument>> searchAfter(String keyword, List<String> lastSortValues) throws IOException {
        OpenPointInTimeResponse pit = client.openPointInTime(o -> o
            .index("posts-read")
            .keepAlive("1m"));

        SearchResponse<PostDocument> response = client.search(s -> {
            var builder = s
                .pit(p -> p.id(pit.id()).keepAlive("1m"))
                .size(20)
                .sort(so -> so.field(f -> f.field("createdAt").order(SortOrder.Desc)))
                .sort(so -> so.field(f -> f.field("id").order(SortOrder.Asc)))
                .query(q -> q.multiMatch(mm -> mm.query(keyword).fields("title^3", "content")));

            if (lastSortValues != null && !lastSortValues.isEmpty()) {
                builder.searchAfter(lastSortValues.stream().map(v -> co.elastic.clients.json.JsonData.of(v)).toList());
            }
            return builder;
        }, PostDocument.class);

        client.closePointInTime(c -> c.id(pit.id()));
        return response.hits().hits();
    }
}
```

### 9.6 무중단 별칭 전환(curl 예시)

```bash
# 1) 새 인덱스 생성
PUT /posts-v2
{ ... new mappings/settings ... }

# 2) 데이터 재색인
POST /_reindex
{
  "source": { "index": "posts-v1" },
  "dest": { "index": "posts-v2" }
}

# 3) alias 원자적 교체
POST /_aliases
{
  "actions": [
    { "remove": { "index": "posts-v1", "alias": "posts-read" } },
    { "add":    { "index": "posts-v2", "alias": "posts-read" } }
  ]
}
```

## 10. 내가 자주 실수한 항목 체크리스트

1. `text` 필드에 집계 걸기 위해 fielddata를 켰다.
2. 배열 객체를 `object`로 둬서 교차 매칭 버그를 만들었다.
3. deep pagination에 `from/size`를 끝까지 사용했다.
4. refresh를 남발해서 인덱싱 성능을 잃었다.
5. 동적 매핑을 방치해 필드 수 폭증을 만들었다.
6. alias 없이 인덱스명을 코드에 하드코딩했다.
7. replica가 있으니 백업도 된다고 착각했다(snapshot 없음).

## 11. 공부 순서 추천 (입문 -> 실전 -> 심화)

1. 역색인/segment/refresh 개념을 먼저 확실히 이해
2. mapping(`text/keyword`, `nested`) + analyzer(nori) 실습
3. bool query에서 `must` vs `filter` 분리 습관화
4. profile/explain API로 점수와 병목을 관찰
5. alias + reindex + ILM으로 운영 시나리오 구축
6. 마지막에 vector 검색과 하이브리드 검색으로 확장

## 12. 공식 문서 (주제별)

### 12.1 Spring Data Elasticsearch
- 버전 호환표: [https://docs.spring.io/spring-data/elasticsearch/reference/elasticsearch/versions.html](https://docs.spring.io/spring-data/elasticsearch/reference/elasticsearch/versions.html)
- ElasticsearchOperations: [https://docs.spring.io/spring-data/elasticsearch/reference/elasticsearch/template.html](https://docs.spring.io/spring-data/elasticsearch/reference/elasticsearch/template.html)
- Repositories: [https://docs.spring.io/spring-data/elasticsearch/reference/elasticsearch/repositories/elasticsearch-repositories.html](https://docs.spring.io/spring-data/elasticsearch/reference/elasticsearch/repositories/elasticsearch-repositories.html)

### 12.2 Elasticsearch 핵심 개념/쿼리
- Near real-time 검색: [https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html)
- Query vs Filter context: [https://www.elastic.co/docs/reference/query-languages/query-dsl/query-filter-context](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-filter-context)
- Bool query: [https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html)
- Multi match query: [https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html)

### 12.3 Mapping/분석기
- Mapping 개요: [https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html)
- `text` field: [https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html)
- `keyword` field: [https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html)
- `nested` field type: [https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)
- Nori analyzer: [https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori.html](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori.html)
- Synonym token filter: [https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-synonym-tokenfilter.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-synonym-tokenfilter.html)

### 12.4 성능/운영
- Pagination(`search_after`, PIT): [https://www.elastic.co/docs/reference/elasticsearch/rest-apis/paginate-search-results](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/paginate-search-results)
- ILM: [https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- Index templates: [https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html)
- Aliases: [https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html)
- Reindex API: [https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html)
- Snapshot/Restore: [https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html)
- Mapping limit settings: [https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-settings-limit.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-settings-limit.html)

### 12.5 Java Client
- Java API Client: [https://www.elastic.co/docs/reference/elasticsearch/clients/java](https://www.elastic.co/docs/reference/elasticsearch/clients/java)

---

정리하면, ES는 "쿼리 잘 쓰는 기술"이라기보다 "검색 제품을 운영하는 기술"에 가깝다.  
내가 체감한 핵심은 세 가지였다.

1. 매핑/분석기 설계가 검색 품질을 만든다.
2. 샤드/페이지네이션/집계 설계가 성능을 만든다.
3. alias/ILM/snapshot 자동화가 운영 안정성을 만든다.

이 세 축을 같이 설계하면, 초반에는 조금 느려도 후반 운영 비용이 압도적으로 줄어든다.
