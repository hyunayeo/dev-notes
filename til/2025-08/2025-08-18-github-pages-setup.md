---
layout: page
title: "2025-08-18 - GitHub Pages 설정"
date: 2025-08-18
categories: [til]
tags: [github-pages, jekyll, 웹사이트]
permalink: /til/2025-08/2025-08-18-github-pages-setup.md
---

# 2025-08-18 - GitHub Pages 설정

## 📚 새로 배운 것

### GitHub Pages와 Jekyll 연동

- GitHub Pages는 Jekyll을 기본으로 지원함
- `_config.yml` 파일로 사이트 전체 설정 가능
- YAML front matter가 있어야 Jekyll이 파일을 처리함

### Jekyll Liquid 태그 활용

```liquid
{% assign posts = site.pages | where_exp: "page", "page.path contains 'til/'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.name }})
{% endfor %}
```

## 🤔 배운 점

- 마크다운 파일에 front matter 없으면 디자인이 적용되지 않음
- Jekyll의 Liquid 태그로 동적으로 파일 목록 생성 가능
- GitHub Pages는 몇 분의 빌드 시간이 필요함

## 📝 참고 자료

- [Jekyll 공식 문서](https://jekyllrb.com/)
- [GitHub Pages 문서](https://pages.github.com/)
- [Liquid 태그 문법](https://shopify.github.io/liquid/)