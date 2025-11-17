---
layout: post
title: Linkary 개발일지 01 - 왜 만들게 되었나
date: 2025-11-17
tags: [recap]
---

## 시작 - 개인 지식 관리에 대한 고민

요즘 개인 지식 DB(PKM, Personal Knowledge Management)에 관심이 많았습니다.  
정보는 넘쳐나는데, 정작 내가 저장한 건 찾기 어렵고, 연결되지 않은 채 흩어져 있었습니다.
이를 개선하고자 URL을 통해 만드는 나만의 지식 도서관을 만들어 보려고 합니다.
[Github Linkary Repository](https://github.com/hobit22/Linkary)

---

## Obsidian - 좋지만 귀찮았던 것

Obsidian을 써보니 그래프 기반으로 노트를 연결하는 방식이 정말 좋았습니다.  
하지만 문제가 있었습니다.

**"연결을 직접 만들어야 한다"**

```markdown
# 노트A

[[노트B]] 참고
[[노트C]]와 연관됨

# 결과:

- 매번 [[]] 타이핑
- 기존 노트 이름 기억해야 함
- 자동으로 연결되지 않음
```

→ "자동으로 연결되면 안 될까?"

---

## 내 경험 - 모바일에서의 링크 저장

저는 사실 이렇게 저장해왔습니다:

```
PC:
→ 브라우저 북마크

모바일:
→ 카카오톡 나에게 보내기
→ 1:1 오픈채팅방 (주제별로 분류)
```

**장점:** 공유 메뉴에서 2초 만에 저장
**단점:** 나중에 찾기 너무 어려움

→ **"카톡의 편리함 + 자동 정리 + 그래프 연결 = 이게 필요하다"**

---

## 기존 솔루션 조사

비슷한 고민을 하는 사람들이 많았습니다.

### 웹 서비스

- **Link Dropper**: 크롬 익스텐션으로 원클릭 저장, AI 자동 요약 [(개발일지)](https://velog.io/@link_dropper/link-auto-summary-and-tags)
- **Pocket**: 나중에 읽기 서비스 [Pocket 2025년 7월 종료](https://web-highlights.com/blog/pocket-shuts-down-in-july-2025-the-10-best-alternatives/)

### 모바일 앱 (플레이스토어)

- **Raindrop.io**: 북마크 관리 + 협업 기능 [App Store Link](https://play.google.com/store/apps/details?id=io.raindrop.raindropio)
- **Smarter Bookmarks**: AI 카테고리 분류, 자동 태깅 [App Store Link](https://play.google.com/store/apps/details?id=com.smarter.technologist.android.smarterbookmarks)
- **VisiMarks**: 썸네일 기반 북마크 [App Store Link](https://play.google.com/store/apps/details?id=jp.fuukiemonster.thumbnailbookmark)
- 그 외 30개 이상의 북마크 관리 앱

### Notion 활용법

YouTube에서 [Notion AI로 링크 요약하기](https://youtu.be/FzZ1kBO9K0U?si=pGjN29bBRAFkKK48) 같은 워크플로우도 발견했습니다.

---

## 공통적인 아쉬움

여러 서비스를 써보면서 느낀 점:

**1. 모바일 저장이 불편함**

- 크롬 익스텐션은 PC 전용

**2. 저장하고 잊어버림**

- 정리는 되지만 다시 보게 만들지 않음
- "나중에 읽기" 큐에 쌓이기만 함

**3. 관계 시각화 부재**

- 폴더/태그로 분류는 되지만
- 링크 간 연결은 보이지 않음

---

## Linkary - 만들고 싶은 것

### 핵심 컨셉

> 모바일/PC 모두에서 쉽게 저장하고,  
> 자동으로 정리하고,  
> Obsidian처럼 그래프로 연결되는 링크 관리 도구

### 차별화 포인트

1. **모바일 퍼스트**: Share Sheet 통합 (카톡처럼 간편)
2. **자동 정리**: AI 기반 태깅/카테고리
3. **자동 연결**: 관련 링크 자동 그래프 생성
4. **재발견**: 주간 다이제스트로 저장한 링크 상기

---

## 기술 스택

- **Frontend**: Next.js 15, React 18, TypeScript
- **Backend**: FastAPI
- **추후**: AI 자동 태깅, 브라우저 확장

---

## 개발 계획

### Phase 1 (MVP)

- 기본 CRUD
- 메타데이터 자동 추출
- 그래프 시각화

### Phase 2

- 모바일 WebView 앱 배포
- 공유 메뉴(Share Sheet) 저장
- AI 자동 태깅
- 주간 다이제스트

### Phase 3

- 브라우저 확장(원클릭 저장)
- 자동 관계 생성
- AI 요약 기능 고도화

---

## 마치며

기존 서비스들에서 본 자동 요약과 태깅, Obsidian에서 느낀 그래프의 힘, 그리고 내 모바일 저장 경험.

이 모든 걸 합쳐서 **내가 쓰고 싶은 도구**를 만들어보려 합니다.

_"저장한 링크를 잊지 않고, 자동으로 연결되는 나만의 지식 도서관"_

다음 개발일지에서는 실제 구현 과정을 공유하겠습니다.
