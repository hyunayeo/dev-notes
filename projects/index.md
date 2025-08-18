---
layout: page
title: Projects
permalink: /projects/
---

# 📁 Projects

프로젝트별 개발 경험과 주요 구현 사항, 트러블슈팅 내역을 정리하는 공간입니다.

## 📋 프로젝트 목록

{% assign project_posts = site.pages | where_exp: "page", "page.path contains 'projects/'" | where_exp: "page", "page.name != 'index.md'" | sort: "date" | reverse %}

{% if project_posts.size > 0 %}
  {% for post in project_posts %}
- [{{ post.title | default: post.name | remove: '.md' }}]({{ post.name }}) - {{ post.date | default: "날짜 없음" }}
  {% endfor %}
{% else %}
### 📝 작성 예정
- 현재 진행 중인 프로젝트들을 정리하여 추가할 예정입니다.
{% endif %}

## 📝 문서 작성 가이드

새로운 프로젝트 문서를 작성할 때는 [`templates/project-template.md`](../templates/project-template.md)를 참고하세요.

### 파일명 규칙
- `프로젝트명-YYYY-MM.md` 형식으로 작성

### 포함할 내용
- 프로젝트 개요 및 기술스택
- 주요 구현 사항
- 발생한 문제와 해결 과정
- 배운 점과 개선사항