---
layout: post
title: TechBlogHub 개발일지 04
date: 2025-09-26
tags: [recap]
---

## 이번 주 개발 요약

[지난 개발일지](https://hobit22.github.io/posts/techbloghub_recap_03/)에 이어 이번 주 작업 내용을 정리합니다.
핵심 주제는 **Auto Tagging Process V2 구축**과 **태깅 정확도 개선**입니다.

---

## 1. Auto Tagging Process V2 개발

### V1의 한계

기존 시스템은 **RSS title/description만 사용**했기 때문에 다음과 같은 문제가 있었습니다.

- **태그 누락**: 짧은 텍스트만으로는 정확한 태깅 어려움
- **잘못된 태깅**: 컨텍스트 부족으로 인한 오분류
- **사전 정의 태그 한계**: 약 200개 고정 태그로는 기술 글 다양성을 커버하기 어려움

### 문제 해결 가설 설정

이 문제를 해결하기 위해 다음 세 가지 가설을 세웠습니다.

1. **가설 1**: 글 전문이 없어서 태깅이 부정확하다
2. **가설 2**: 사전 정의된 태그 자체가 부족하다
3. **가설 3**: 프롬프트가 구려서 태깅이 부정확하다

---

## 2. 가설 1 검증 – 글 전문 크롤링 시스템 구축

### 크롤링 전략 및 구현

**하이브리드 크롤링 시스템**을 구축했습니다:

- **WebBaseLoader**: 기본 정적 HTML 크롤링 (빠름)
- **PlaywrightURLLoader**: JavaScript 렌더링 필요 시 자동 전환 (안정적)
- **자동 Fallback**: 1,000자 미만일 경우 Playwright로 재시도

**초기 시도**: [mdream](https://github.com/harlan-zw/mdream) 도구를 시도했으나 Medium 글 크롤링 실패로 LangChain 기반으로 전환

```python
# 최종 구현된 크롤링 로직
loader = WebBaseLoader(url)
content = loader.load()
if len(content) < 1000:
    content = PlaywrightURLLoader(url).load()
```

### 검증 결과

**63개 글을 대상으로 A/B 테스트**를 진행했습니다.

| 기준              | 태그 누락률 | 태그 성공률 |
| ----------------- | ----------- | ----------- |
| Description 기반  | **51%**     | 49%         |
| TotalContent 기반 | **27%**     | **73%**     |

**➡ 태깅 성공률이 49% → 73%로 24%p 개선됨**

### Content Processor 시스템

전용 Python 시스템을 구축하여 자동화된 처리를 구현했습니다.

```
content_processor/
├── main.py                 # 메인 오케스트레이터
├── processors/
│   ├── database_manager.py # PostgreSQL 연동
│   ├── content_processor.py# 하이브리드 크롤링
│   └── summary_generator.py# AI 요약 생성
└── utils/logger.py         # 로깅
```

**처리 성능**

```
Hybrid processing completed in 33.92s:
✅ Success: 9/10 URLs
❌ Failed: 1/10 URLs
📄 Total content: 75,714 chars
⏱️  Average processing time: 1.32s/URL

...

Batch summary generation completed:
✅ Success: 9/9
❌ Failed: 0/9
⏱️  Average processing time: 12.19s
📄 Total input: 75,714 chars
📄 Total output: 2,530 chars
```

### 추가 활용 방안

크롤링된 글 전문은 태깅뿐만 아니라 다양한 기능으로 확장 활용됩니다:

- **자동 요약**: GPT를 활용한 구조화된 요약으로 글 상세페이지 개발
- **검색 품질 향상**: 전체 내용 기반 검색으로 더 정확한 결과 제공
- **관련 글 추천**: 내용 유사도 기반 추천 시스템 구축 예정

![Summary UI](assets/img/posts/teckbloghub-summary.png)

---

## 3. 가설 2 검증 – 사전 정의 태그의 한계와 대안 탐색

### 기존 태깅 시스템의 한계점

현재 약 200개의 사전 정의된 태그를 사용하고 있지만 다음과 같은 한계가 있습니다:

- **기술 다양성 부족**: 신기술 부재
- **동적 대응 불가**: 새로운 프레임워크, 라이브러리 출현 시 수동 추가 필요

### 대안 실험: HuggingFace T5 모델

사전 정의 태그 대신 **동적 태그 생성**을 위해 [fabiochiu/t5-base-tag-generation](https://huggingface.co/fabiochiu/t5-base-tag-generation) 모델을 실험했습니다.

**모델 특징**

- Text-to-Text 변환 기반 태그 생성
- Medium 글 데이터셋으로 학습 (190k 글)
- 영어 컨텐츠에 최적화

### 실험 결과: 사용 불가 판정

**테스트 결과**

- **4개 한국어 기술 글** 모두에서 **거의 동일한 태그 출력**
- 예시: `"AI, Artificial Intelligence, Machine Learning, Technology, Software"`

**실패 원인**

1. **도메인 불일치**: Medium 일반 글 vs 기술 특화 콘텐츠
2. **언어 처리 한계**: 영어 학습 데이터로 인한 한국어 맥락 이해 부족
3. **토큰 제한**: 512 토큰으로 긴 기술 글의 전체 맥락 파악 어려움

**결론**

```
❌ T5 기반 태그 생성 → 사용 불가
✅ 현재 GPT 방식 유지하며 개선 방향 모색
```

**향후 탐색할 대안**

- **Multi-label 분류 모델**: BERT 기반 한국어 분류 모델 활용
- **TF-IDF 기반 키워드 추출**: 전통적 방식과의 성능 비교
- **GPT 프롬프트 최적화**: 현재 방식의 정교한 튜닝
- **하이브리드 접근**: 여러 방식을 조합한 앙상블 모델

**평가 방법론 구상**

태깅 품질을 객관적으로 평가하기 위한 메트릭 개발 예정:

- 정확도, 재현율, F1-score 기반 정량 평가
- 전문가 검토를 통한 정성 평가
- A/B 테스트를 통한 사용자 만족도 측정

---

## 4. 시스템 구현 및 데이터베이스 확장

### Post 엔티티 확장

```java
@Entity
public class Post {
    ...
    @Column(columnDefinition = "TEXT")
    private String totalContent;   // 크롤링된 글 전문

    @Column(columnDefinition = "TEXT")
    private String summaryContent; // AI 생성 요약
    ...
}
```

### TaggingProcessor 개선

```java
@Component
public class TaggingProcessor {
    public TaggingResult processWithTotalContent(Post post) {
        String content = StringUtils.hasText(post.getTotalContent())
            ? post.getTotalContent()
            : post.getDescription();
        return performTagging(content, POST_TAGGING_CATEGORIES);
    }
}
```

**처리 전략**

- Content 우선순위: TotalContent → SummaryContent → Description 순
- 길이 기반 최적화: 너무 긴 컨텐츠는 요약 후 태깅
- 실패 처리: 태깅 실패 시 fallback 전략 적용

### 태깅이 어려운 콘텐츠 식별

기술 블로그라도 다음과 같이 **태깅이 불가능한 글**들이 존재함을 확인했습니다:

- **회사 문화/브랜딩**: 기술과 직접적 연관이 적은 콘텐츠
- **디자인 사례**: 순수 기술이 아닌 디자인/UX 중심 글
- **비즈니스 케이스**: 제품 소개, 마케팅 관련 글

**예시**: [여행도 하고 지구도 지킨다, 여기어때 쓰봉크럽 디자인 리뉴얼](https://techblog.gccompany.co.kr/%EC%97%AC%ED%96%89%EB%8F%84-%ED%95%98%EA%B3%A0-%EC%A7%80%EA%B5%AC%EB%8F%84-%EC%A7%80%ED%82%A8%EB%8B%A4-%EC%97%AC%EA%B8%B0%EC%96%B4%EB%95%8C-%EC%93%B0%EB%B4%89%ED%81%AC%EB%9F%BD-%EB%94%94%EC%9E%90%EC%9D%B8-%EB%A6%AC%EB%89%B4%EC%96%BC-b14f692d9218)

→ **기술 블로그라 하더라도 순수 기술 외 다양한 콘텐츠가 존재함을 확인**

---

## 이번 주의 배움

### 1. 데이터의 질이 AI 성능을 좌우한다

RSS title/description만으로는 한계가 명확했습니다. 글 전문을 확보하니 태깅 정확도가 크게 개선되었습니다.

### 2. 사전학습 모델의 도메인 적합성 중요성

범용 모델이라도 특정 도메인(한국어 기술 블로그)에서는 성능이 떨어질 수 있음을 확인했습니다.

### 3. 단계별 검증의 중요성

각 가설을 체계적으로 검증하면서 어떤 요소가 실제로 성능에 영향을 미치는지 명확히 파악할 수 있었습니다.

---

## 다음 주 계획

### 1. Auto Tagging Process V3 설계

- **가설 3 검증**: 프롬프트 엔지니어링 개선
- 태깅 품질 평가 방법 구상

### 2. 크롤링 시스템 고도화

- prod 환경에서의 배치 처리 시스템 구축
- 크롤링 실패 재시도 메커니즘

### 3. 사용자 경험 개선

- 검색 성능 최적화
- 모바일 반응형 개선
- 관련글 검색 기능 추가

---

## 마치며

이번 주는 **"가설 검증의 주간"**이었습니다.

체계적인 실험을 통해 태깅 성능을 개선할 수 있는 실질적인 방법을 찾았고, 특히 데이터 품질의 중요성을 다시 한 번 깨달았습니다.

다음 주에는 프롬프트 개선을 통해 더욱 정확한 태깅 시스템을 만들어보겠습니다!
