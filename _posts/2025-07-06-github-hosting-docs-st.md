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

<img src="/images/2025-07-06-github-hosting-docs-st/1.png" style="display: block; margin: 0 auto;" />

- 사이트 띄우기 완료~

## home 기본 내용 수정

<img src="/images/2025-07-06-github-hosting-docs-st/2.png" style="display: block; margin: 0 auto;" />

- 리드미를 보면, index.md를 고치면 된다고 한다.

```markdown
---
title: Hello world!
layout: home ## 이건 기본 레이아웃이라 변경하면 안된다.
---

이건 docs home 페이지이다.
hello world!😉
```
- INDEX.md를 다음과 같이 고쳐보자.

<img src="/images/2025-07-06-github-hosting-docs-st/3.png" style="display: block; margin: 0 auto;" />

## Configuration
- 이제 하나하나 설정을 바꿔보면서 test해보는 작업을 해보자.
- 이 부분 모든 작업은 _config.yml file에서 진행하면 된다.
- 또한, yml 파일은 저장 새로고침 해도 반영이 안되므로, 이걸 수정할 경우 반드시 ruby 서버를 다시 띄워줘야 한다.

### Site logo
```markdown
# Set a path/url to a logo that will be displayed instead of the title
logo: "/assets/images/aespa.png"
```
<img src="/images/2025-07-06-github-hosting-docs-st/4.png" style="display: block; margin: 0 auto;" />

- 에스파 노래중에서 girls 노래 타이틀이 제일 내 취향이라 이걸로 해보았다

### Site favicon
```markdown
# Set a path/url to a favicon that will be displayed by the browser
favicon_ico: "/assets/images/favicon.ico"
```
- favicon이란?
    - 사이트 아이콘을 말한다.
    - 보통은 16*16을 많이 사용한다고 함
    - 직접 만들어 보자.
        - 사이트 : [로고 메이커](https://ko.wix.com/logo/maker)

<img src="/images/2025-07-06-github-hosting-docs-st/5.png" style="display: block; margin: 0 auto;" />
- 에스파 느낌 내려고 ae text로 해봤는데, 이게 제일 힙해서 이걸로 했다.
- 또한, favicon의 경우 확장자가 .ico라 변경해야 한다.
    -  사이트 : [png to ico 변환기](https://www.freeconvert.com/ko/png-to-ico)

### Aux links
- 깃허브 링크 연결이 가능해서 필요한 기능이다.

```markdown
aux_links:
  "test_docs on GitHub":
    - "https://github.com/sunJ0120/test_docs.git"

# Makes Aux links open in a new tab. Default is false
aux_links_new_tab: false
```

- 나의 경우는, 내 해당 레포로 이동하도록 해보겠다.

### Navigation sidebar

```markdown
# Enable or disable the side/mobile menu globally
# Nav menu can also be selectively enabled or disabled using page variables or the minimal layout
nav_enabled: true
```

- 사이드 네비게이션을 키는 기능이다. true로 한다.

### Heading anchor links

```markdown
# Heading anchor links appear on hover over h1-h6 tags in page content
# allowing users to deep link to a particular heading on a page.
#
# Supports true (default) or false
heading_anchors: true
```
- 헤딩 (제목)에 커서를 올렸을때, 링킹이 가능하게끔 하는 기능이다.

<img src="/images/2025-07-06-github-hosting-docs-st/6.png" style="display: block; margin: 0 auto;" />

- 그니까 요런걸 말하는 건데, 필요한 기능이라 추가한다.

### Color scheme

```markdown
# Color scheme supports "light" (default) and "dark"
color_scheme: dark
```
- light는 기본이 보라색, dark는 검정 + 파랑 조합이다.
- 보통은 light로 하는데 나는 실험삼아 dark로 바꿔보았다.

## Navigation

- 참고 페이지 : [just-the-docs](https://just-the-docs.com/docs/navigation/main/levels/) 
- 우리 페이지의 경우, 다음과 같은 구조로 되어 있어야 한다.
    - OhGoodFood
    - 가이드
        - 사용자 가이드
        - 사장님 가이드
        - 어드민 가이드
- 그러므로, navigation 안에 child를 만들어서 페이지를 구성해야 한다.

```markdown
---
title: parents_pages1
nav_order: 2 ## index.md 다음으로 두번째로 와야한다는 의미이다.
---
```

```markdown
---
title: child_pages1
parent: parents_pages1
nav_order: 1
---
```

```markdown
---
title: child_pages2
parent: parents_pages1
nav_order: 2
---
```

<img src="/images/2025-07-06-github-hosting-docs-st/7.png" style="display: block; margin: 0 auto;" />
- 페이지 파일들은 전부 docs 안에 넣어준다.

# 결과
<img src="/images/2025-07-06-github-hosting-docs-st/8.png" style="display: block; margin: 0 auto;" />

- 하위 네비게이션까지 구현이 잘 된 모습이다
- favicon이 있기 때문에, 사이트 왼쪽에 로고가 생겼다.

<img src="/images/2025-07-06-github-hosting-docs-st/10.png" style="display: block; margin: 0 auto;" />
- 참고로 aux link가 false여서, 깃허브로 이동시 새 창이 안뜨고 바로 창이 변환된다.