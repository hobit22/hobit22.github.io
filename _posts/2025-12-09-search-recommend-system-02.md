---
layout: post
title: 검색 & 추천 시스템 학습 노트 02 - PostgreSQL FTS vs Elasticsearch 비교
date: 2025-12-09
tags: [ElasticSearch, Search, PostgreSQL, Nori]
---

[1편]({% post_url 2025-11-21-search-recommend-system-01 %})에서 검색 시스템의 이론을 학습했다면, 이번 편에서는 실제로 3가지 검색 엔진을 구현하고 비교합니다.

## 1. PostgreSQL FTS의 한계

### 초기 구현: PostgreSQL FTS

**데이터:** [기술블로그 모음](https://teckbloghub.kr)에 있는 데이터 2,961개 포스트

```python
# 검색 쿼리
SELECT *
FROM posts
WHERE keyword_vector @@ to_tsquery('simple', 'query')
ORDER BY ts_rank(keyword_vector, query) DESC
```

### 문제점

**Zero Result Rate: 15.4%**

```
"kubernets" 검색 → 0건 (오타 처리 불가)
"recat" 검색 → 0건
```

**한글 형태소 분석 없음**

```
"데이터베이스를 최적화" → "데이터베이스" 매칭 실패
"리액트로 시작하는" → "리액트" 매칭 실패
```

**결과:**

- 평균 62.3개 결과
- 평균 응답 시간 175.8ms

---

## 2. Elasticsearch 도입

### Phase 1: Standard 분석기

```yaml
# docker-compose.yml
elasticsearch:
  image: elasticsearch:8.11.0
  environment:
    - discovery.type=single-node
```

**개선점:**

- Fuzziness로 오타 허용 (`"kubernets"` → `"kubernetes"`)
- 공백 기준 토큰화로 검색 범위 확장

**결과:**

- 평균 421.5개 결과 (FTS 대비 **6.8배**)
- Zero Result Rate **0%**
- 평균 응답 시간 473.9ms

### Phase 2: Nori 형태소 분석기

```bash
# Nori 플러그인 설치
docker exec techbloghub-elasticsearch \
  bin/elasticsearch-plugin install analysis-nori
```

```python
# Nori 매핑
POST_INDEX_MAPPING_NORI = {
    "settings": {
        "analysis": {
            "tokenizer": {
                "nori_user_dict": {
                    "type": "nori_tokenizer",
                    "decompound_mode": "mixed"
                }
            },
            "analyzer": {
                "nori_analyzer": {
                    "type": "custom",
                    "tokenizer": "nori_user_dict"
                }
            }
        }
    }
}
```

**Nori 동작 원리:**

```
입력: "데이터베이스를 최적화했습니다"

Standard:
→ ["데이터베이스를", "최적화했습니다"]

Nori:
→ ["데이터베이스", "를", "최적화", "했", "습니다"]
```

**결과:**

- 평균 794.5개 결과 (FTS 대비 **12.7배**, Standard 대비 **1.9배**)
- Zero Result Rate **0%**
- 평균 응답 시간 413.8ms

---

## 3-Way 비교 결과

### 종합 통계

| 메트릭        | PostgreSQL FTS | ES Standard | ES Nori     |
| ------------- | -------------- | ----------- | ----------- |
| **평균 결과** | 62.3개         | 421.5개     | **794.5개** |
| **Zero Rate** | 15.4%          | 0%          | **0%**      |
| **응답 시간** | 175.8ms        | 473.9ms     | **413.8ms** |

### 실제 검색 케이스

**"리액트" 검색:**

```
FTS: 38개
Standard: 223개
Nori: 605개 (15.9배 차이)

이유:
"리액트로 시작하는" → Standard: "리액트로" (미매칭)
                    → Nori: "리액트" + "로" (매칭!)
```

**"마이크로서비스 아키텍처":**

```
FTS: 18개
Standard: 414개
Nori: 2,572개 (142배 차이!)

이유:
Nori의 decompound_mode="mixed"가
"마이크로서비스" → ["마이크로서비스", "마이크로", "서비스"]
```

**오타 검색 "kubernets":**

```
FTS: 0개
Standard: 108개 (Fuzziness)
Nori: 115개
```

---

## 3. 현재 측정의 한계와 다음 단계

### 이번 비교에서 측정한 것

**양적 지표:**

- Zero Result Rate (15.4% → 0%)
- 결과 개수 (62.3개 → 794.5개)
- 응답 시간 (175.8ms → 413.8ms)

→ **Nori가 Standard보다 1.9배, FTS보다 12.7배 많은 결과**

### 측정하지 못한 것

**결과가 많다고 좋은 게 아니다:**

```
"쿠버네티스" 검색 시:
- 1위: 쿠버네티스 핵심 아키텍처 설명?
- 100위: "쿠버네티스" 단어만 언급?

→ 순위의 품질을 평가하지 못함
```

**실제 필요한 지표** ([1편 참고]({% post_url 2025-11-21-search-recommend-system-01 %}#검색-품질-지표)):

- Precision@K: 상위 K개가 정말 관련 있는가?
- NDCG: 관련도 높은 문서가 상위에 있는가?
- CTR: 사용자가 실제로 클릭하는가?

### 사이드 프로젝트의 현실

**유저 테스트가 불가능한 상황:**

- 실사용자 부족
- A/B 테스트 환경 없음
- 클릭 로그 데이터 없음

**대신 할 수 있는 것:**

**1. 대표 쿼리 샘플링**

```json
{
  "query": "쿠버네티스 성능",
  "relevant_posts": [123, 456, 789], // 직접 평가
  "irrelevant_posts": [999, 888]
}
```

**2. 상위 10개 수동 평가**

```
"React 성능" 검색 결과:
1위: React 성능 최적화 가이드 (5점)
2위: React Compiler 소개 (5점)
3위: 브라우저 렌더링 원리 (3점)
...

→ Precision@5, NDCG 직접 계산
```

**3. Zero Result 분석**

```
FTS에서 결과 0개였던 쿼리:
- "kubernets" (오타)
- "recat" (오타)

→ Nori는 모두 해결 (Fuzziness)
```

---

## 핵심 교훈

**1. 형태소 분석의 중요성**

```
한글: "를/을/이/가" 같은 조사 분리 필수
→ Nori가 FTS보다 12.7배 많은 결과
```

**2. Pattern 2 아키텍처**

```
Elasticsearch: 검색용 (post_id, title, content)
PostgreSQL: 저장용 (전체 데이터)

장점: Single Source of Truth 유지
```

**3. 단계적 도입**

```
Week 1: FTS 구현 (베이스라인)
Week 2: ES Standard 비교
Week 3: Nori 추가
Week 4: 품질 지표 측정
```

**4. 검색 품질은 복잡하다**

```
결과 개수 ≠ 검색 품질
→ Relevance, Ranking, User Satisfaction 측정 필요
```

---

## 다음 단계

**학습 로드맵:**

1. ~~PostgreSQL FTS~~
2. ~~Elasticsearch Standard~~
3. ~~Nori 형태소 분석기~~
4. Precision@K, NDCG 측정
5. 동의어 사전
6. Vector Search (Semantic Search)

**실전 적용:**

- Ground Truth 데이터셋 구축
- A/B 테스트 설계
- 사용자 행동 로그 분석

---

## 참고

- [검색 학습 노트 01]({% post_url 2025-11-21-search-recommend-system-01 %})
- [Elasticsearch Nori](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori.html)
