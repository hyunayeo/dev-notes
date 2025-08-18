---
layout: page
title: TIL - 2025년 8월
permalink: /til/2025-08/
---

# 📖 TIL - 2025년 8월

2025년 8월에 새롭게 배운 내용들을 정리한 공간입니다.

## 📚 이번 달 학습 내용

{% assign til_posts = site.pages | where_exp: "page", "page.path contains 'til/2025-08/'" | where_exp: "page", "page.name != 'index.md'" | sort: "date" | reverse %}

{% if til_posts.size > 0 %}
{% for post in til_posts %}

- [{{ post.title | default: post.name | remove: '.md' }}]({{ post.name }}) - {{ post.date | default: "날짜 없음" }}
  {% endfor %}
  {% else %}

## 📝 문서 작성 가이드

- 파일명: `YYYY-MM-DD-주제.md` 형식
- 템플릿: [TEMPLATES.md](../../TEMPLATES.md#til-template) 참고

[← TIL 메인으로 돌아가기](../)
