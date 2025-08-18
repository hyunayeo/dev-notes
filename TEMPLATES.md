# 📝 Templates

일관된 문서 작성을 위한 템플릿 모음입니다.

## 📋 사용 가능한 템플릿

### Project Template
프로젝트별 개발 경험과 주요 구현 사항을 정리할 때 사용하는 템플릿입니다.

**템플릿 복사**: [project-template.md](https://raw.githubusercontent.com/hyunayeo/dev-notes/main/TEMPLATES.md#project-template)

```markdown
---
layout: page
title: "프로젝트명"
date: 2025-01-01  # 실제 작성일로 변경하세요
categories: [projects]
tags: [프로젝트명, 기술스택]
---

# 프로젝트명

## 📝 프로젝트 개요

- 기간:
- 기술스택:
- 역할:
- 목적:

## 🛠 주요 구현 사항

## 🚨 트러블슈팅

### 문제 상황

- 발생 시점:
- 현상:

### 원인 분석

### 해결 과정

### 최종 해결책

`코드 블록`

### 배운 점

## 🔗 참고 자료
```

---

### Troubleshooting Template
개발 중 발생한 문제와 해결 과정을 체계적으로 정리할 때 사용하는 템플릿입니다.

**템플릿 복사**: [troubleshooting-template.md](https://raw.githubusercontent.com/hyunayeo/dev-notes/main/TEMPLATES.md#troubleshooting-template)

```markdown
---
layout: page
title: "[카테고리] 문제 제목"
date: 2025-01-01  # 실제 작성일로 변경하세요
categories: [troubleshooting, 카테고리]
tags: [bug, database, api]
---

# [카테고리] 문제 제목

**발생일:** YYYY-MM-DD  
**프로젝트:**  
**카테고리:** 카테고리명

## 🚨 문제 상황

## 🔍 원인 분석

## 🛠 해결 과정

1. 시도한 방법 1
2. 시도한 방법 2
3. 최종 해결책

## 💡 배운 점

## 🔗 참고 자료
```

---

### TIL Template
일일 학습 내용을 간단히 기록할 때 사용하는 템플릿입니다.

**템플릿 복사**: [til-template.md](https://raw.githubusercontent.com/hyunayeo/dev-notes/main/TEMPLATES.md#til-template)

```markdown
---
layout: page
title: "YYYY-MM-DD - 주제명"
date: 2025-01-01  # 실제 작성일로 변경하세요
categories: [til]
tags: [카테고리, 기술명]
---

# YYYY-MM-DD - 주제명

## 📚 새로 배운 것

### [주제명]

- 내용
- 코드 예시 (있다면)

## 🤔 궁금한 점

## 📝 내일 해볼 것
```

## 📖 템플릿 사용법

1. 위의 템플릿 코드를 복사
2. 적절한 폴더에 새 파일로 저장
3. 템플릿 내용을 실제 내용으로 교체
4. 파일명 규칙에 따라 저장

### 파일명 규칙
- 프로젝트: `프로젝트명-YYYY-MM.md`
- 트러블슈팅: `카테고리-문제요약-YYYY-MM-DD.md`
- TIL: `YYYY-MM-DD-주제.md`