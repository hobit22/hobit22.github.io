---
layout: post
title: 검색 & 추천 시스템 학습 노트 01
date: 2025-11-21
tags: [ElasticSearch, Search]
---

사이드 프로젝트에 Elasticsearch를 도입하면서 학습한 내용을 정리한 글입니다. 검색 시스템의 기본 원리부터 실무에서 마주칠 수 있는 문제와 해결 방법까지 다룹니다. 한글 처리, 페이지네이션 전략, 검색 품질 측정 등을 구체적인 예시와 함께 설명합니다.

## 목차

1. [검색과 추천의 기본 개념](#검색과-추천의-기본-개념)
2. [Elasticsearch 기본 개념](#elasticsearch-기본-개념)
3. [Analyzer와 한글 처리](#analyzer와-한글-처리)
4. [검색 플로우와 데이터 처리](#검색-플로우와-데이터-처리)
5. [검색 품질 지표](#검색-품질-지표)
6. [Pagination 전략](#pagination-전략)
7. [캐싱 전략](#캐싱-전략)
8. [Solr vs Elasticsearch](#solr-vs-elasticsearch)
9. [Vector DB와 하이브리드 검색](#vector-db와-하이브리드-검색)

---

## 검색과 추천의 기본 개념

### 검색 (Search)

**사용자가 명시적으로 찾고자 하는 것을 찾아주는 것**

#### 핵심 개념들

**1. Inverted Index (역색인)**

- 전통적인 DB: "문서 → 단어들"로 저장
- 검색 엔진: "단어 → 해당 단어가 있는 문서들"로 저장
- 빠른 검색의 핵심

**2. 텍스트 분석 과정**

- Tokenization: "Spring Boot 개발"을 ["Spring", "Boot", "개발"]로 분리
- Normalization: 대소문자 통일, 동의어 처리
- Stemming/Lemmatization: "running", "ran" → "run"

**3. 검색 결과 랭킹**

- **TF-IDF**: 단어가 문서에 얼마나 자주 나오고, 전체 문서에서는 얼마나 희귀한지
- **BM25**: TF-IDF를 개선한 현대적 알고리즘 (Elasticsearch 기본값)
  - 문서 길이 정규화
  - 단어 빈도의 포화 효과 적용
  - 더 정교한 관련도 계산

### 추천 (Recommendation)

**사용자가 좋아할 만한 것을 예측해서 제시하는 것**

**1. Collaborative Filtering (협업 필터링)**

- "비슷한 취향을 가진 사람들이 좋아한 것을 추천"
- User-based: 나와 비슷한 사용자들이 좋아한 아이템
- Item-based: 내가 좋아한 아이템과 비슷한 아이템들
- Netflix, YouTube가 이걸 많이 사용

**2. Content-based Filtering (콘텐츠 기반)**

- "내가 좋아한 것과 비슷한 특성을 가진 것을 추천"
- 아이템의 메타데이터(장르, 키워드, 카테고리 등)를 활용
- Cold start 문제(신규 사용자)에 강함

**3. Hybrid 접근**

- 둘을 섞어서 각각의 단점을 보완

### 실무에서 자주 마주치는 문제들

- **Cold Start**: 신규 사용자/아이템은 데이터가 없어서 추천이 어려움
- **Scalability**: 사용자가 백만명이면 계산량이 폭발적으로 증가
- **Real-time vs Batch**: 실시간으로 계산할지, 미리 계산해둘지

---

## Elasticsearch 기본 개념

### Elasticsearch란?

**Elasticsearch는 Lucene 기반의 분산형 검색 및 분석 엔진**

- 실시간에 가까운 검색 가능
- RESTful API로 쉽게 사용
- 분산 처리 지원

### 핵심 개념 1: 데이터 구조

#### Document (문서)

- 가장 기본적인 데이터 단위
- JSON 형태로 저장
- RDB의 "Row"와 비슷한 개념

```json
{
  "_id": "1",
  "title": "Spring Boot 성능 최적화",
  "author": "hobit22",
  "tags": ["Spring", "Performance"],
  "content": "N+1 문제를 해결하는 방법...",
  "created_at": "2024-11-20",
  "view_count": 1500
}
```

#### Index (인덱스)

- Document들의 논리적인 모음
- RDB의 "Database" 또는 "Table"과 비슷
- 예: `tech_blogs`, `products`, `users`

### 핵심 개념 2: 클러스터 아키텍처

#### Cluster (클러스터)

- 여러 노드(서버)들의 집합
- 전체 데이터를 저장하고 모든 노드에서 검색 제공
- 클러스터 이름으로 식별: `my-application`

#### Node (노드)

- 클러스터 내의 단일 서버
- 데이터를 저장하고 클러스터의 인덱싱 및 검색에 참여
- 역할별로 구분:
  - **Master Node**: 클러스터 관리
  - **Data Node**: 실제 데이터 저장 및 검색
  - **Ingest Node**: 데이터 전처리
  - **Coordinating Node**: 요청 라우팅

#### Shard (샤드)

**Index를 여러 조각으로 나눈 것**

왜 필요할까?

- 하나의 인덱스가 너무 크면 단일 노드에 저장할 수 없음
- 병렬 처리로 성능 향상

```
tech_blogs Index (총 1TB)
├── Shard 0 (250GB) - Node 1
├── Shard 1 (250GB) - Node 2
├── Shard 2 (250GB) - Node 3
└── Shard 3 (250GB) - Node 4
```

**Primary Shard (주 샤드)**

- 원본 데이터를 가진 샤드
- 인덱스 생성 시 개수 결정 (기본값: 1개)
- **한번 설정하면 변경 불가** (reindex 필요)

#### Replica (레플리카)

**Primary Shard의 복사본**

목적:

1. **고가용성(High Availability)**: 노드 장애 시에도 서비스 가능
2. **검색 성능 향상**: 검색 요청을 여러 복제본에 분산

```
Primary Shard 0 (Node 1)
  └── Replica Shard 0 (Node 2)  # 백업

Primary Shard 1 (Node 2)
  └── Replica Shard 1 (Node 3)  # 백업
```

**중요 규칙**: Primary와 Replica는 절대 같은 노드에 있지 않습니다!

### 핵심 개념 3: Inverted Index (역색인)

**Elasticsearch가 빠른 검색의 비밀**

#### 일반 DB의 방식 (Forward Index)

```
Document 1: "Spring Boot is awesome"
Document 2: "Spring Framework tutorial"
Document 3: "Boot camp for developers"
```

"Spring"을 검색하려면? → 모든 문서를 다 뒤져야 함 (느림)

#### Inverted Index 방식

```
Term(단어)    → Document IDs
"spring"      → [1, 2]
"boot"        → [1, 3]
"awesome"     → [1]
"framework"   → [2]
"tutorial"    → [2]
"camp"        → [3]
"developers"  → [3]
```

"Spring"을 검색하려면? → "spring" 키를 찾으면 바로 [1, 2] 반환 (빠름!)

#### 실제 동작 과정

```
원본 텍스트: "Spring Boot Performance Optimization"

1. Tokenization (토큰화)
   → ["Spring", "Boot", "Performance", "Optimization"]

2. Lowercasing (소문자 변환)
   → ["spring", "boot", "performance", "optimization"]

3. Stemming (어간 추출, 선택적)
   → ["spring", "boot", "perform", "optim"]

4. Inverted Index에 저장
   "spring" → [doc_1, doc_5, doc_12, ...]
   "boot" → [doc_1, doc_3, doc_8, ...]
```

---

## Analyzer와 한글 처리

### Analyzer란?

**"텍스트를 검색 가능한 토큰(단어)으로 변환하는 파이프라인"**

```
입력: "Spring Boot를 사용한 REST API 개발"
     ↓
[Analyzer 처리]
     ↓
출력: ["spring", "boot", "rest", "api", "개발"]
```

### Analyzer의 3단계 구조

#### 1. Character Filter (문자 필터)

**텍스트를 토큰화하기 전에 전처리**

```
입력: "Spring Boot™를 사용한 REST-API 개발!!!"

Character Filter 적용:
- HTML 태그 제거: <b>Spring</b> → Spring
- 특수문자 제거: ™ 제거
- 문자 치환: REST-API → REST API

출력: "Spring Boot를 사용한 REST API 개발"
```

#### 2. Tokenizer (토크나이저)

**텍스트를 개별 토큰으로 분리** (가장 중요!)

**Standard Tokenizer (영어용)**

```
"Spring Boot REST API"
→ ["Spring", "Boot", "REST", "API"]
```

공백과 구두점 기준으로 분리

**Nori Tokenizer (한글용)**

```
"스프링부트를사용한다"
→ ["스프링", "부트", "를", "사용", "한다"]
```

형태소 분석으로 의미 단위로 분리

#### 3. Token Filter (토큰 필터)

**토큰을 추가로 가공**

```
입력 토큰: ["Spring", "Boot", "REST", "API"]

Token Filters:
1. Lowercase: ["spring", "boot", "rest", "api"]
2. Synonym (동의어): "rest" → ["rest", "restful"]
3. Stop words 제거: "를", "은", "는" 등 제거

출력: ["spring", "boot", "rest", "restful", "api"]
```

### 한글 처리: Nori Analyzer

**Elasticsearch에서 공식 제공하는 한글 형태소 분석기**

#### 설치

```bash
bin/elasticsearch-plugin install analysis-nori
```

#### Nori의 동작 방식

```
입력: "스프링부트를 사용해서 API를 개발했습니다"

Nori Tokenizer:
└─→ ["스프링", "부트", "를", "사용", "해서", "API", "를", "개발", "했", "습니다"]

Nori Part of Speech Filter (품사 필터):
- 조사 제거: "를", "해서" 제거
- 어미 제거: "했", "습니다" 제거
└─→ ["스프링", "부트", "사용", "API", "개발"]

Lowercase:
└─→ ["스프링", "부트", "사용", "api", "개발"]
```

#### Decompound Mode (복합어 처리)

한글의 복합어를 어떻게 분리할지 결정:

**1. none (분리 안함)**

```
"백엔드개발자"
→ ["백엔드개발자"]
```

**2. discard (완전 분리)**

```
"백엔드개발자"
→ ["백엔드", "개발자"]
```

**3. mixed (둘 다 저장) - 추천**

```
"백엔드개발자"
→ ["백엔드개발자", "백엔드", "개발자"]
```

**왜 mixed가 좋을까?**

- "백엔드개발자" 검색 → 정확히 매칭
- "백엔드" 검색 → 부분 매칭 가능
- "개발자" 검색 → 부분 매칭 가능

#### 사용자 사전 (User Dictionary)

**Nori가 모르는 IT 용어를 등록**

```
# userdict_ko.txt
스프링부트
카프카
엘라스틱서치
쿠버네티스
REST API
마이크로서비스
```

등록하지 않으면:

```
"스프링부트"
→ ["스프링", "부트"]  # 분리됨
```

등록하면:

```
"스프링부트"
→ ["스프링부트"]  # 하나의 단어로 인식
```

---

## 검색 플로우와 데이터 처리

### 인덱싱 전략

#### 필드별 인덱싱 전략

**1. 검색 대상 필드 (Analyzed)**

- `title`, `description`, `content`
- **토큰화, 소문자 변환, 형태소 분석** 등을 거침
- Inverted Index에 저장됨

```
title: "Kafka를 사용할 때"
→ Tokenization: ["Kafka", "를", "사용", "할", "때"]
→ Inverted Index:
  "kafka" → [doc_12345, ...]
  "사용" → [doc_12345, ...]
```

**2. 필터링/정렬 필드 (Not Analyzed 또는 Keyword)**

- `company`, `author`, `created_at`, `view_count`
- **원본 그대로 저장** (토큰화 안함)
- 정확한 매칭, 정렬, 집계에 사용

```
company: "NAVER" → 그대로 "NAVER"로 저장
(토큰화하면 의미 없음)
```

**3. 배열 필드**

- `tags`
- 각 요소가 개별적으로 인덱싱됨

```
tags: ["Kafka", "Backend"]
→ Inverted Index:
  "kafka" → [doc_12345, ...]
  "backend" → [doc_12345, ...]
```

#### Mapping 설정 예시

```json
PUT /tech_blogs
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "nori_mixed": {
          "type": "nori_tokenizer",
          "decompound_mode": "mixed",
          "user_dictionary": "userdict_ko.txt"
        }
      },
      "filter": {
        "tech_synonym": {
          "type": "synonym",
          "synonyms": [
            "스프링, Spring => spring",
            "리액트, React => react",
            "REST API, RESTful API => rest_api",
            "백엔드, backend => backend"
          ]
        },
        "nori_posfilter": {
          "type": "nori_part_of_speech",
          "stoptags": ["E", "J", "SC", "SE", "SF"]
        }
      },
      "analyzer": {
        "korean_analyzer": {
          "type": "custom",
          "tokenizer": "nori_mixed",
          "filter": [
            "nori_posfilter",
            "lowercase",
            "tech_synonym"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "korean_analyzer",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "korean_analyzer"
      },
      "content": {
        "type": "text",
        "analyzer": "korean_analyzer"
      },
      "tags": {
        "type": "keyword"
      },
      "company": {
        "type": "keyword"
      }
    }
  }
}
```

#### 검색 시 필드별 가중치 (Boosting)

실무에서는 **필드마다 중요도를 다르게** 설정:

```json
{
  "query": {
    "multi_match": {
      "query": "Kafka Consumer Group",
      "fields": ["title^3", "description^2", "content^1", "tags^2"]
    }
  }
}
```

**왜 이렇게 하나요?**

- 제목에 "Kafka"가 있으면 → 핵심 주제일 가능성 높음
- 본문에만 "Kafka"가 있으면 → 언급만 한 것일 수도 있음

### 검색 플로우

**일반적인 플로우:**

```
1. DB에 데이터 존재
2. 데이터를 가지고 Elasticsearch 인덱싱
3. 사용자가 검색어를 통한 검색
4. 검색어를 가지고 Elasticsearch에서 인덱스 조회, id list 반환
5. id List를 가지고 DB에서 조회
6. 사용자에게 반환
```

#### 구체적인 플로우 (TechBlogHub 예시)

**1단계: 초기 인덱싱**

```java
// DB에 저장된 데이터
Blog {
  id: 1,
  title: "Kafka Consumer Group Protocol",
  description: "네이버 사내 기술 교류...",
  content: "전체 본문 5000자...",
  company: "NAVER",
  tags: ["Kafka", "Backend"],
  author: "NAVER D2",
  createdAt: "2024-11-20",
  viewCount: 1500,
  url: "https://d2.naver.com/..."
}

// Elasticsearch에 인덱싱 (검색용 필드만)
{
  "_id": "1",  // DB의 ID 그대로 사용
  "title": "Kafka Consumer Group Protocol",
  "description": "네이버 사내 기술 교류...",
  "content": "전체 본문 5000자...",  // 검색용
  "tags": ["Kafka", "Backend"],
  "company": "NAVER"
  // viewCount, url 같은 자주 바뀌는 건 안넣음
}
```

**2단계: 사용자 검색**

```
사용자 입력: "Kafka 성능 최적화"
```

**3단계: Elasticsearch 쿼리**

```json
POST /tech_blogs/_search
{
  "query": {
    "multi_match": {
      "query": "Kafka 성능 최적화",
      "fields": ["title^3", "description^2", "content", "tags^2"]
    }
  },
  "_source": false,
  "size": 20,
  "from": 0
}
```

**4단계: Elasticsearch 응답 (ID만)**

```json
{
  "hits": {
    "total": { "value": 150 },
    "hits": [
      { "_id": "1", "_score": 12.5 },
      { "_id": "45", "_score": 10.2 },
      { "_id": "123", "_score": 8.7 }
    ]
  }
}
```

**5단계: DB에서 조회**

```java
List<Long> ids = Arrays.asList(1L, 45L, 123L, ...);

// JPA로 IN 쿼리
List<Blog> results = blogRepository.findAllById(ids);

// 중요: Elasticsearch 순서대로 재정렬!
Map<Long, Blog> blogMap = results.stream()
    .collect(Collectors.toMap(Blog::getId, b -> b));

List<Blog> orderedResults = ids.stream()
    .map(blogMap::get)
    .collect(Collectors.toList());
```

**6단계: 응답**

```json
{
  "totalCount": 150,
  "results": [
    {
      "id": 1,
      "title": "Kafka Consumer Group Protocol",
      "description": "...",
      "company": "NAVER",
      "tags": ["Kafka", "Backend"],
      "viewCount": 1500,
      "url": "https://...",
      "createdAt": "2024-11-20"
    }
  ]
}
```

### Pattern 1 vs Pattern 2

**Pattern 1: Elasticsearch만 사용 (Full Document 저장)**

```
1. DB 데이터 → Elasticsearch에 전체 문서 저장
2. 검색 → Elasticsearch에서 바로 결과 반환
3. DB 조회 없음

장점: 빠름, DB 부하 없음
단점: 데이터 중복, 동기화 복잡
```

**Pattern 2: Elasticsearch + DB 조합 (추천)**

```
1. DB 데이터 → Elasticsearch에 검색 필드만 인덱싱
2. 검색 → Elasticsearch에서 ID 리스트만 반환
3. ID로 DB 조회 → 전체 데이터 가져옴
4. 사용자에게 반환

장점: 데이터 정합성 유지, 실시간 데이터 보장
단점: DB 조회 한 번 더 필요
```

#### Pattern 2의 장점

**1. 데이터 정합성**

```
Elasticsearch: 검색용 스냅샷
DB: 실제 데이터 (Single Source of Truth)

예: viewCount가 실시간으로 증가
- ES에는 1500으로 저장되어 있어도
- DB에서 조회하면 최신값 1523 반환
```

**2. 저장 공간 절약**

```
Elasticsearch:
- title, description, content (검색용)
- tags, company (필터용)

DB에만:
- viewCount (자주 변함)
- likeCount (자주 변함)
- comments (검색 불필요)
- metadata (많은 필드들)
```

**3. 동기화 부담 감소**

```
ES에 저장하는 필드가 적을수록
→ 업데이트가 필요한 경우가 적음
→ 동기화 로직이 단순함
```

---

## 검색 품질 지표

### 핵심 지표

#### 비즈니스 지표 (가장 중요)

**CTR (Click Through Rate)**

```
= 클릭 수 / 검색 수
```

**예시:**

```
검색 1000번 → 클릭 150번
CTR = 15%

ES 도입 전: 12%
ES 도입 후: 18%
→ 50% 개선
```

**전환율 (Conversion Rate)**

```
= 구매(또는 목표 행동) / 검색 수

ES 도입 전: 3.2%
ES 도입 후: 4.8%
→ 50% 개선
```

**Zero Result Rate**

```
= 결과 없음 / 전체 검색

ES 도입 전: 15% (오타 처리 약함)
ES 도입 후: 5% (Fuzzy search)
→ 67% 개선
```

### 사용자 행동 지표

**평균 검색 깊이**

```
= 평균 몇 페이지까지 보는가?

ES 도입 전: 2.3 페이지
ES 도입 후: 1.5 페이지
→ 1페이지에서 더 잘 찾음
```

**검색 재시도율**

```
= 같은 세션에서 검색어 수정 비율

ES 도입 전: 35%
ES 도입 후: 20%
→ 첫 검색에서 만족
```

**평균 체류 시간**

```
= 검색 결과 페이지 체류

ES 도입 전: 8초 (빨리 떠남)
ES 도입 후: 25초 (관련 결과 보는 중)
→ 품질 향상
```

### 검색 품질 지표 (IR Metrics)

**MRR (Mean Reciprocal Rank)**

```
= 평균 (1 / 첫 관련 문서의 순위)

사용자 A: 3번째 클릭 → 1/3
사용자 B: 1번째 클릭 → 1/1
사용자 C: 5번째 클릭 → 1/5

MRR = (1/3 + 1/1 + 1/5) / 3 = 0.53

ES 도입 전: 0.45
ES 도입 후: 0.68
→ 더 위에서 찾음
```

**NDCG (Normalized Discounted Cumulative Gain)**

```
= 상위 결과의 관련도 품질

간단히:
- 위에 있을수록 가중치 높음
- 관련도 높은게 위에 있으면 점수 높음

ES 도입 전: 0.65
ES 도입 후: 0.82
→ 랭킹 개선
```

### 측정 방법

#### A/B 테스트

```java
public class SearchService {

    public SearchResult search(String query, String userId) {
        // 50% 사용자는 기존 DB 검색
        // 50% 사용자는 ES 검색

        if (isTestGroup(userId)) {
            return elasticsearchSearch(query); // 실험군
        } else {
            return databaseSearch(query); // 대조군
        }
    }

    // 각 그룹의 CTR, 전환율 비교
}
```

#### 로그 수집

```java
@Aspect
@Component
public class SearchLoggingAspect {

    @AfterReturning("execution(* search(..))")
    public void logSearch(JoinPoint jp, SearchResult result) {
        SearchLog log = SearchLog.builder()
            .userId(userId)
            .query(query)
            .resultCount(result.size())
            .searchEngine(result.getEngine()) // "ES" or "DB"
            .timestamp(now())
            .build();

        kafkaProducer.send("search-logs", log);
    }

    @AfterReturning("execution(* clickResult(..))")
    public void logClick(JoinPoint jp) {
        ClickLog log = ClickLog.builder()
            .userId(userId)
            .query(query)
            .clickedRank(rank) // 몇 번째 결과 클릭?
            .clickedDocId(docId)
            .timestamp(now())
            .build();

        kafkaProducer.send("click-logs", log);
    }
}
```

### 실전 대시보드 예시

```
검색 품질 대시보드 (Kibana)

[주요 지표]
┌─────────────────────────────────────┐
│ CTR: 18.5% (↑ 6.5%p)              │
│ Zero Result: 5.2% (↓ 9.8%p)       │
│ 평균 검색 깊이: 1.5페이지 (↓ 0.8)   │
│ 전환율: 4.8% (↑ 1.6%p)            │
└─────────────────────────────────────┘

[검색어별 성능]
검색어          | CTR  | Zero Result
──────────────────────────────────
"Kafka"        | 25%  | 0%
"스프링부트"     | 22%  | 0%
"쿠버네티스 설치" | 8%   | 35%  ← 개선 필요!
```

### 개선이 필요한 검색어 찾기

**Zero Result가 많은 검색어**

```sql
SELECT
    query,
    COUNT(*) as search_count,
    SUM(CASE WHEN result_count = 0 THEN 1 ELSE 0 END) as zero_count,
    (SUM(CASE WHEN result_count = 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*)) as zero_rate
FROM search_logs
WHERE search_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY query
HAVING search_count >= 10
ORDER BY zero_rate DESC
LIMIT 20;
```

**CTR이 낮은 검색어**

```sql
SELECT
    query,
    COUNT(DISTINCT search_id) as searches,
    COUNT(DISTINCT click_id) as clicks,
    (COUNT(DISTINCT click_id) * 100.0 / COUNT(DISTINCT search_id)) as ctr
FROM search_logs s
LEFT JOIN click_logs c ON s.search_id = c.search_id
WHERE s.search_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY query
HAVING searches >= 10
ORDER BY ctr ASC
LIMIT 20;
```

### 커머스 특화 지표 (컬리 등)

**장바구니 추가율**

```
= 검색 → 장바구니 / 검색 수
```

**즉시 구매율**

```
= 검색 → 바로 구매 / 검색 수
```

**평균 주문 금액 (AOV)**

```
= 검색 경로로 들어온 주문의 평균 금액
```

**재검색까지 시간**

```
= 같은 사용자가 다시 검색하기까지 시간
(오래 걸릴수록 만족도 높음)
```

### 실무 팁

#### 단계적 측정

```
Week 1-2: 기본 지표 (CTR, Zero Result)
Week 3-4: 행동 지표 (검색 깊이, 재시도)
Month 2+: 비즈니스 지표 (전환율, 매출)
```

#### 세그먼트별 분석

```
- 신규 vs 재방문
- 모바일 vs PC
- 검색어 길이별
```

#### 정기 리뷰

```
- 주간: CTR, Zero Result 모니터링
- 월간: 전환율, MRR 분석
- 분기: A/B 테스트 결과 정리
```

---

## Pagination 전략

### Elasticsearch Pagination의 3가지 방식

#### 1. from/size (Offset Pagination) - 기본 방식

```json
GET /tech_blogs/_search
{
  "query": {
    "match": { "title": "Kafka" }
  },
  "from": 0,
  "size": 20
}
```

**Deep Pagination 문제**

```
from=9000, size=20을 요청하면?

각 샤드에서:
1. 상위 9020개 문서를 모두 메모리에 로드
2. Coordinating Node로 전송
3. 모든 샤드 결과를 병합
4. 상위 9020개 중 9000~9020만 반환

5개 샤드라면: 9020 × 5 = 45,100개 문서를 메모리에!
```

**Elasticsearch 기본 제한:**

```
from + size ≤ 10,000 (기본값)

from=9990, size=20 → OK
from=10000, size=20 → ERROR!

참고: index.max_result_window 설정으로 변경 가능하지만
성능 문제로 권장하지 않음
```

**from/size 적합한 경우**

- 일반적인 웹 검색 (1~50페이지 정도)
- 사용자가 깊은 페이지까지 안감

#### 2. Search After (Cursor Pagination) - 추천

**마지막 문서의 sort 값을 기준으로 다음 페이지 조회**

```json
// 1페이지 조회
GET /tech_blogs/_search
{
  "query": { "match": { "title": "Kafka" } },
  "size": 20,
  "sort": [
    { "_score": "desc" },
    { "id": "asc" }
  ]
}

// 응답
{
  "hits": [
    {
      "_id": "123",
      "_score": 10.5,
      "sort": [10.5, 123]
    }
  ]
}

// 2페이지 조회 (search_after 사용)
GET /tech_blogs/_search
{
  "query": { "match": { "title": "Kafka" } },
  "size": 20,
  "sort": [
    { "_score": "desc" },
    { "id": "asc" }
  ],
  "search_after": [10.5, 123]
}
```

**장점:**

- Deep pagination 문제 없음
- 메모리 효율적
- 성능 일정 (페이지가 깊어져도 속도 동일)
- 실시간 데이터에 강함

**단점:**

- 특정 페이지로 점프 불가 (1→5페이지 직접 이동 안됨)
- 이전 페이지로 돌아가기 어려움
- 총 페이지 수 계산 불가

**Search After 적합한 경우:**

- 무한 스크롤 UI
- "다음" 버튼만 있는 경우
- 실시간 피드 (Twitter, Instagram)
- 대량 데이터 Export

#### Search After의 한계

**새로운 데이터 추가 시 문제:**

```
시간 T0: 초기 상태
Rank 1-10:  [A(15.5), B(15.2), C(14.8), ..., J(12.1)]

사용자가 1페이지 조회
→ [A, B, C, D, E, F, G, H, I, J]
→ 마지막 sort 값: [12.1, "J"]

시간 T1: 새 문서 3개 추가 (높은 점수)
신규1(16.0), 신규2(15.8), 신규3(14.5)

사용자가 2페이지 조회 (search_after: [12.1, "J"])
→ search_after는 "12.1보다 작은 것"을 찾음
→ [K(11.9), L(11.5), M(11.2), ...]

신규3(14.5)는 12.1보다 크므로 건너뜀!
중복은 없음
하지만 신규1, 신규2는 영원히 못봄 (누락)
```

#### 3. Point in Time (PIT) API

**검색 시점의 스냅샷을 고정**

```json
// 1. PIT 생성 (검색 시작 시점 고정)
POST /tech_blogs/_pit?keep_alive=5m

// 2. 1페이지 조회
GET /_search
{
  "pit": {
    "id": "46ToAwMD...",
    "keep_alive": "5m"
  },
  "query": { "match": { "title": "Kafka" } },
  "size": 10,
  "sort": [{ "_score": "desc" }]
}

// [이 시점에 새 문서가 추가되어도 무시됨]

// 3. 2페이지 조회 (같은 PIT 사용)
GET /_search
{
  "pit": {
    "id": "46ToAwMD...",
    "keep_alive": "5m"
  },
  "query": { "match": { "title": "Kafka" } },
  "size": 10,
  "search_after": [12.1, "doc_J"],
  "sort": [{ "_score": "desc" }]
}
```

**PIT 내부 구현:**

```
Elasticsearch의 각 샤드는 Lucene 인덱스:

Segment 기반 구조:
┌─ Shard 1 ─────────────┐
│ Segment 1 (불변)       │  ← T0 시점
│ Segment 2 (불변)       │  ← T1 시점
│ Segment 3 (불변)       │  ← T2 시점
│ Segment 4 (현재 쓰기중) │  ← T3 시점
└───────────────────────┘

PIT 생성 = "이 시점의 Segment들을 기억"
```

**PIT 특징:**

- 세그먼트 "참조"만 저장 (실제 데이터 복사 안함)
- 특정 시점의 세그먼트만 검색
- 새 세그먼트는 무시
- Keep alive로 자동 만료

### 실무 선택 가이드

**Case 1: 일반 검색 결과 (구글처럼)**

- from/size + 페이지 제한 (최대 50페이지)

**Case 2: 무한 스크롤 (모바일 앱, 소셜 피드)**

- search_after

**Case 3: 하이브리드 (컬리 같은 커머스)**

- 앞쪽 페이지는 from/size
- 깊은 페이지는 search_after

| 방식             | 사용 케이스             | 장점                        | 단점                               |
| ---------------- | ----------------------- | --------------------------- | ---------------------------------- |
| **from/size**    | 일반 검색 (페이지 버튼) | 구현 간단, 페이지 점프 가능 | Deep pagination 느림, 10000건 제한 |
| **search_after** | 무한 스크롤, 피드       | 성능 일정, 제한 없음        | 페이지 점프 불가                   |
| **PIT**          | 일관성 필수 상황        | 완벽한 일관성               | 리소스 사용                        |

---

## 캐싱 전략

### 검색 결과 캐싱이 필요할까?

#### Case A: 일반 검색어 - 캐싱 효과 없음

```
사용자들의 검색어:
"Spring Boot JPA N+1"
"스프링 부트 성능 최적화"
"Kafka consumer group rebalancing"
"쿠버네티스 배포 전략"
// ... 수십만 개의 다양한 검색어

캐시 키: "Spring Boot JPA N+1:page=1"
캐시 히트율: < 5%

이유:
- 검색어 조합이 무한대
- 롱테일 검색어가 대부분
- 같은 검색어를 여러 사용자가 칠 확률 낮음
```

**결론: 일반 검색은 캐싱 안함**

#### Case B: 인기/추천 검색어 - 캐싱 효과 있음

```
홈 화면의 "인기 검색어"
1. "Kafka"
2. "Docker"
3. "Spring Boot"
4. "Kubernetes"
5. "React"

캐시 키: "popular:Kafka:page=1"
캐시 히트율: > 70%

이유:
- 같은 검색어를 많은 사용자가 검색
- 검색어 개수가 제한적 (10~20개)
```

#### Case C: 자동완성 - 캐싱 필수

```
자동완성은 앞 글자가 같으면 히트율 높음
"Kaf" → ["Kafka", "Kafka Consumer", "Kafka Stream"]
"Spr" → ["Spring", "Spring Boot", "Spring Cloud"]

캐시 키: "autocomplete:Kaf"
캐시 히트율: > 80%
```

### 대안 1: 부분 캐싱 - ES 응답만 캐싱

```java
// 전체 결과 캐싱 (비효율)
@Cacheable(key = "#keyword + ':' + #page")
public SearchResult search(String keyword, int page) {
    List<Long> ids = searchES(keyword, page);
    List<Blog> blogs = searchDB(ids);
    return SearchResult.of(blogs);
}

// ES 응답만 캐싱 (더 나음)
public SearchResult search(String keyword, int page) {
    // ES 결과는 캐싱 (ID 리스트만)
    List<Long> ids = searchESWithCache(keyword, page);

    // DB는 항상 최신 조회 (viewCount 등 실시간 반영)
    List<Blog> blogs = searchDB(ids);

    return SearchResult.of(blogs);
}

@Cacheable(value = "esResults", key = "#keyword + ':' + #page")
private List<Long> searchESWithCache(String keyword, int page) {
    return searchES(keyword, page);
}
```

**장점:**

- ES 쿼리는 무거움 → 캐싱 효과
- DB 쿼리는 IN query로 빠름 → 실시간 데이터 보장

### 대안 2: 캐싱 안하고 성능 최적화

```java
// 캐싱 대신 최적화에 집중
public SearchResult search(String keyword, int page) {
    // 1. ES 쿼리 최적화
    // - 필드 제한 (_source: false)
    // - 불필요한 집계 제거
    // - 적절한 shard 수

    // 2. DB 쿼리 최적화
    // - Covering Index
    // - Connection Pool
    // - Query Cache (MySQL)

    // 3. 네트워크 최적화
    // - ES와 BE를 같은 AZ에 배치
    // - DB Read Replica 사용
}
```

---

## Solr vs Elasticsearch

### Solr란?

**Apache Solr** = Lucene 기반의 검색 플랫폼 (Elasticsearch와 형제뻘)

```
Apache Lucene (검색 라이브러리)
    ├── Elasticsearch (2010년 출시)
    └── Apache Solr (2006년 출시, 더 오래됨)
```

### 주요 차이

| 특성            | Elasticsearch         | Solr                        |
| --------------- | --------------------- | --------------------------- |
| **출시**        | 2010년 (최신)         | 2006년 (역사 깊음)          |
| **설정**        | JSON 기반, 쉬움       | XML 기반, 복잡함            |
| **커뮤니티**    | 매우 활발             | 활발 (하지만 ES보다 작음)   |
| **실시간 검색** | 강력 (Near Real-time) | 가능하지만 ES보다 약간 느림 |
| **분산 환경**   | 자동 Shard 분배       | SolrCloud (별도 설정)       |
| **분석/집계**   | 강력 (Kibana)         | 약함                        |
| **학습 곡선**   | 완만                  | 가파름                      |
| **성능**        | 대용량 데이터에 강함  | 정적 데이터에 강함          |

### 왜 컬리는 Solr를 언급했을까?

**1. 레거시 시스템**

- 많은 회사들이 Solr를 먼저 도입했음
- 오래된 회사들은 Solr 사용 중
- 새 서비스는 ES, 기존 서비스는 Solr 유지

**2. 특정 기능이 Solr에 강함**

Solr의 장점:

- **Faceted Search (패싯 검색)** - 커머스 필터링에 강함
- **Join 쿼리** - 관계형 조인을 더 잘 지원
- **Group By** - 결과 그룹핑

**3. 대기업의 다중 검색엔진 전략**

- 상품 검색: Solr (facet, 필터 중요)
- 로그 분석: Elasticsearch (분석, 대시보드)
- 추천 시스템: 자체 구축 또는 별도 솔루션

**실무에서는:**
학습은 **Elasticsearch 먼저 추천** - 더 현대적이고 자료도 많음

---

## Vector DB와 하이브리드 검색

### Vector DB가 주목받는 이유

**전통적 검색 (ES)의 한계:**

```
검색어: "스프링부트 성능"
→ 키워드 매칭: "스프링부트", "성능" 포함된 문서

문제:
- "Spring Boot 최적화" 못찾음 (다른 단어)
- "JPA N+1 문제 해결" 못찾음 (의미는 관련있지만 키워드 다름)
```

**Vector 검색:**

```
검색어: "스프링부트 성능"
→ 임베딩: [0.23, -0.45, 0.78, ...]
→ 의미적으로 유사한 문서 찾기

찾아냄:
- "Spring Boot 최적화"
- "JPA N+1 해결법"
- "애플리케이션 속도 개선"
```

### 주요 Vector DB

1. **Pinecone** - 관리형, 쉬움
2. **Weaviate** - 오픈소스, 하이브리드 검색
3. **Milvus** - 대규모 특화
4. **Qdrant** - Rust 기반, 빠름
5. **Chroma** - LangChain 친화적

**그런데 Elasticsearch도 Vector 지원함**

### 하이브리드 검색: Reciprocal Rank Fusion (RRF)

**점수를 직접 비교할 수 없어서 RRF를 사용**

```
ES 키워드 검색 (BM25):
doc1: 15.3
doc2: 12.7
doc3: 10.1

Vector 검색 (Cosine Similarity):
doc1: 0.92
doc5: 0.88
doc3: 0.85
```

15.3 vs 0.92를 어떻게 비교?

**Reciprocal Rank Fusion (RRF)**

순위만 사용해서 병합

**공식:**

```
RRF Score = Σ 1 / (k + rank)

k = 상수 (보통 60)
rank = 해당 검색에서의 순위
```

**구체적 예시**

검색어: "Kafka 성능 최적화"

```
ES 키워드 결과:
1위: doc_A (score: 15.3)
2위: doc_B (score: 12.7)
3위: doc_C (score: 10.1)
4위: doc_D (score: 9.5)

Vector 검색 결과:
1위: doc_C (similarity: 0.92)
2위: doc_A (similarity: 0.88)
3위: doc_E (similarity: 0.85)
4위: doc_F (similarity: 0.82)
```

RRF 계산 (k=60):

```
doc_A:
  ES: 1/(60+1) = 0.0164
  Vector: 1/(60+2) = 0.0161
  합계: 0.0325

doc_B:
  ES: 1/(60+2) = 0.0161
  Vector: 없음 = 0
  합계: 0.0161

doc_C:
  ES: 1/(60+3) = 0.0159
  Vector: 1/(60+1) = 0.0164
  합계: 0.0323

doc_D:
  ES: 1/(60+4) = 0.0156
  Vector: 없음 = 0
  합계: 0.0156

doc_E:
  ES: 없음 = 0
  Vector: 1/(60+3) = 0.0159
  합계: 0.0159

doc_F:
  ES: 없음 = 0
  Vector: 1/(60+4) = 0.0156
  합계: 0.0156
```

최종 순위:

```
1위: doc_A (0.0325) ← 양쪽 모두 상위
2위: doc_C (0.0323) ← 양쪽 모두 상위
3위: doc_B (0.0161)
4위: doc_E (0.0159)
5위: doc_D (0.0156)
6위: doc_F (0.0156)
```

**왜 이게 좋은가?**

- 점수 스케일 무관
- 양쪽에서 모두 높으면 최상위
- 한쪽만 높으면 중간
- 구현 간단

**코드 예시**

```java
public List<Document> hybridSearch(String query, int k) {
    // 1. ES 키워드 검색
    List<SearchResult> esResults = elasticsearchSearch(query);

    // 2. Vector 검색
    List<SearchResult> vectorResults = vectorSearch(query);

    // 3. RRF 병합
    return reciprocalRankFusion(esResults, vectorResults, 60);
}

private List<Document> reciprocalRankFusion(
    List<SearchResult> list1,
    List<SearchResult> list2,
    int k
) {
    Map<String, Double> scores = new HashMap<>();

    // list1의 순위 점수
    for (int i = 0; i < list1.size(); i++) {
        String docId = list1.get(i).getId();
        double score = 1.0 / (k + i + 1);
        scores.merge(docId, score, Double::sum);
    }

    // list2의 순위 점수
    for (int i = 0; i < list2.size(); i++) {
        String docId = list2.get(i).getId();
        double score = 1.0 / (k + i + 1);
        scores.merge(docId, score, Double::sum);
    }

    // 점수 순 정렬
    return scores.entrySet().stream()
        .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
        .map(e -> getDocument(e.getKey()))
        .collect(Collectors.toList());
}
```

### 다른 병합 방법들

#### 1. Weighted Sum (가중 합)

```
문제: 점수 정규화 필요

final_score = α × normalize(es_score) + β × normalize(vector_score)

normalize(score) = (score - min) / (max - min)

단점: 정규화가 어려움, 분포 차이
```

#### 2. Linear Combination

```
ES 점수: [0-100] 범위로 정규화
Vector 점수: [0-100] 범위로 정규화

final_score = 0.7 × es_normalized + 0.3 × vector_normalized

단점: 적절한 가중치 찾기 어려움
```

#### 3. Distribution-based Fusion

```
각 점수를 확률 분포로 변환

P(relevant | ES) × P(relevant | Vector)

단점: 복잡함
```

### Elasticsearch의 Hybrid Search

**ES 8.4+부터 내장 지원** (2022년 8월 출시)

```json
{
  "query": {
    "hybrid": {
      "queries": [
        {
          "match": {
            "title": "Kafka 성능"
          }
        },
        {
          "knn": {
            "field": "title_vector",
            "query_vector": [0.23, -0.45, ...],
            "k": 10,
            "num_candidates": 100
          }
        }
      ]
    }
  },
  "rank": {
    "rrf": {
      "window_size": 50,
      "rank_constant": 60
    }
  }
}
```

**ES가 자동으로 RRF 수행!**

### 실전 팁

#### 상황별 검색 전략

**1. 정확한 키워드 검색 (상품명, 고유명사)**

```
키워드 70% + Vector 30%

예: "iPhone 15 Pro", "스프링부트 3.2"
→ 정확한 매칭 우선, 유사 문서는 보조
```

**2. 모호한 의도 검색 (개념, 질문)**

```
키워드 40% + Vector 60%

예: "성능 개선 방법", "에러 해결"
→ 의미적 유사도 우선
```

**3. 이미지 기반 검색**

```
Vector 100%

예: 비슷한 상품 찾기, 시각적 유사도
→ 키워드 없이 벡터만 사용
```

#### 하이브리드 검색 도입 시점

```
1단계: 키워드 검색만 (ES 기본)
2단계: 분석 및 개선 (동의어, 형태소)
3단계: Vector 추가 (하이브리드)

하이브리드 검색이 필요한 경우:
- Zero Result가 10% 이상
- 동의어 처리가 복잡함
- 다국어 지원 필요
- 의미적 검색이 중요
```

---

## 마치며

### 핵심 요약

**1. 검색 시스템 선택**

- 텍스트 검색: Elasticsearch (추천)
- 레거시 시스템: Solr 고려
- 의미 검색 필요: Vector DB 또는 ES Hybrid Search

**2. 단계별 도입 전략**

```
Phase 1: 기본 검색 구축
- ES 인덱싱
- 키워드 검색
- 기본 Analyzer

Phase 2: 품질 개선
- 한글 형태소 분석 (Nori)
- 동의어 처리
- 필드 가중치 조정
- A/B 테스트

Phase 3: 고도화
- Vector 검색 추가
- 개인화
- 실시간 랭킹 조정
```

**3. 성능 최적화 체크리스트**

- [ ] 적절한 샤드 수 설정
- [ ] 검색 필드만 ES에 저장 (패턴 2)
- [ ] 인기 검색어 캐싱
- [ ] DB 인덱스 최적화
- [ ] Pagination 전략 선택
- [ ] 모니터링 대시보드 구축

**4. 측정이 중요**

```
도입 전후 비교:
- CTR (Click Through Rate)
- Zero Result Rate
- 평균 검색 깊이
- 전환율

→ 데이터 기반 개선
```

### 참고 자료

- [Elasticsearch 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Nori 한글 형태소 분석기](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori.html)
- [검색 품질 지표 (IR Metrics)](<https://en.wikipedia.org/wiki/Evaluation_measures_(information_retrieval)>)

검색 시스템은 한 번에 완벽하게 만들 수 없습니다. 작게 시작해서 데이터를 보며 점진적으로 개선하는 것이 중요합니다.
