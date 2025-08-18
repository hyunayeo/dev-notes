---
layout: page
title: Troubleshooting
permalink: /troubleshooting/
---

# 🔧 Troubleshooting

개발 중 발생한 문제와 해결 과정을 카테고리별로 분류하여 정리하는 공간입니다.

## 📂 트러블슈팅 목록

{% assign troubleshooting_posts = site.pages | where_exp: "page", "page.path contains 'troubleshooting/'" | where_exp: "page", "page.name != 'index.md'" | sort: "date" | reverse %}

{% if troubleshooting_posts.size > 0 %}
  {% for post in troubleshooting_posts %}
- [{{ post.title | default: post.name | remove: '.md' }}]({{ post.name }}) - {{ post.date | default: "날짜 없음" }}
  {% endfor %}
{% else %}
### 📝 작성 예정
- 트러블슈팅 경험들을 추가할 예정입니다.

### 카테고리별 분류 예시
- **Frontend**: React, Vue, JavaScript 관련 문제들
- **Backend**: Node.js, Python, API 관련 문제들  
- **DevOps**: Docker, 배포, CI/CD 관련 문제들
- **Database**: MySQL, MongoDB 관련 문제들
- **Mobile**: React Native, Flutter 관련 문제들
{% endif %}

## 📝 문서 작성 가이드

새로운 트러블슈팅 문서를 작성할 때는 [`templates/troubleshooting-template.md`](../templates/troubleshooting-template.md)를 참고하세요.

### 파일명 규칙
- `카테고리-문제요약-YYYY-MM-DD.md` 형식으로 작성

### 포함할 내용
- 문제 상황 상세 기록
- 원인 분석 과정
- 해결 방법 및 근거
- 예방 방법 및 참고사항