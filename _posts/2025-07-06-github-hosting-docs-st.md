---
title: "[GitHub 갖고놀기] Github 웹 호스팅 docs 기본설정 & navigation 설정하기"
layout: single
categories: [github]
tags : [github]
toc: true
toc_sticky: true
toc_label: 목차
author_profile : true
permalink: /github/io/docs/standard
---

> 🥰 깃허브 갖고놀기 기록집
> {: .notice--info}

# 🤔 Intro
- docs 연결이 끝났으니, 이제 docs를 꾸미고 설정 바꾸는걸 해보면서, 우리 프로젝트 템플릿에 필요한 기능들을 익히는 연습을 해보도록 한다.

# 😀 Start!

## 목표
- 기본 config.yml 설정 바꾸는 방법을 익힌다.
- navigation 설정하는 방법을 익힌다.

## RUBY 호스팅
- 우선, 깃허브 호스팅 사이트의 경우, 커밋 푸쉬를 해야만 수정사항을 확인할 수 있다는 치명적인 단점이 존재한다.
- 그러므로 RUBY를 이용해서 localhost:4000에 호스팅해서 개발을 진행하도록 한다.
- 나의 경우는 이미 RUBY를 설치해둔 상태라, clone 한 레포에 들어가서 다음을 입력하면 된다.
```java
bundle exec jekyll serve
```

<img src="/images/2025-07-06-github-hosting-docs/1.png" style="display: block; margin: 0 auto;" />