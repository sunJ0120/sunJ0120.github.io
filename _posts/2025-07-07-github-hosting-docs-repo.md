---
title: "[GitHub 갖고놀기] Github 웹 호스팅 docs 이미 존재하는 repo에 추가하기"
layout: post
categories: [github]
tags : [github]
toc: true
toc_sticky: true
toc_label: 목차
author_profile : true
permalink: /github/io/docs/existing-repo
---

> 🥰 깃허브 갖고놀기 기록집
> {: .notice--info}

# 🤔 Intro
- 우리 프로젝트 안에서 docs를 띄우기 위해서는 이미 존재하는 프로젝트 repo안에 docs 폴더를 만들어서 연동하는 작업이 필요하다.
- 공식 리드미를 참고해서 프로젝트 안에 docs를 띄워보는 실습을 해보겠다.
- 실험은 내 개인 test team repo 안에서 실행한다.

# 😀 Start!
- 참고 리드미 : [just-the-docs](https://github.com/just-the-docs/just-the-docs)
- 리드미를 참고한 결과, 방법은 간단하다.
- 기존에 있던 just-the-docs 파일을 내려받은 다음에
- github-action에 여기(docs)를 루트 삼아 빌드하라고 하면 된다.

## docs 파일 가져오기
1. 내가 만들어 뒀던 test-docs를 zip으로 내려받는다.
![main](/images/2025-07-07-github-hosting-docs-repo/1.png)

2. 해당 docs를 올리고 싶은 내 organization test 레포를 클론해서 로컬에 연결한다.
   ![main](/images/2025-07-07-github-hosting-docs-repo/2.png)

3. .github/workflows 폴더를 만들어서 zip 파일로 받은 것 중, pages.yml을 넣는다.
 - 리드미에 따르면, action은 이 workflow 디렉토리에서 이 YML을 찾아서 실행한다고 한다.
   ![main](/images/2025-07-07-github-hosting-docs-repo/3.png)
 - 해당 yml 파일은 다운받은 zip 파일의 workflow 안에 있다.

## pages.yml 파일 고치기

이제, 리드미에서 pages.yml을 고치라고 한 그대로 고쳐주면 된다.

```yml
jobs:
  # Build job
  build:
    runs-on: ubuntu-latest 
    defaults: #docs 파일 기준 action 실행을 위해 default를 docs로 설정
      run:
        working-directory: docs
    steps:
```
- build 라인에 defaults:에 docs를 추가한다.
  - main 변화가 아닌, docs 변화를 추적하기 위함이다.

```yml
steps:
  - name: Checkout
    uses: actions/checkout@v4
  - name: Setup Ruby
    uses: ruby/setup-ruby@v1
    with:
      ruby-version: '3.3' # Not needed with a .ruby-version file
      bundler-cache: true # runs 'bundle install' and caches installed gems automatically
      cache-version: 0 # Increment this number if you need to re-download cached gems
      working-directory: '${{ github.workspace }}/docs' #docs 사용을 위해 working-directory 추가
```
- steps 라인에 working-directory를 추가한다. 마찬가지로 /docs 라고 되어있다.

```yml
- name: Upload artifact
  # Automatically uploads an artifact from the './_site' directory by default
  uses: actions/upload-pages-artifact@v3
  with:
    path: docs/_site/
```
- artifact 라인에 path-param을 추가한다.

```yml
on:
  push:
    branches:
      - "main"
    paths:
      - "docs/**"
```
- on 라인의 paths에 docs/**를 추가한다.
- 마찬가지로 트리거 역시 docs/ 변화를 추적하도록 하기 위함이다.

![main](/images/2025-07-07-github-hosting-docs-repo/5.png)

- 나머지 파일들은 다음과 같이 옮겨준다.
- docs 안에 zip 파일에 들어있던 assets, _config.yml 설정 파일, Gemfile, index.md를 넣어주면 된다.
- 아까 action yml 설정 파일에서 docs를 기준으로 동작하도록 해두었기 때문에 문제없이 작동한다.
- 이제 이를 push 하고, github 홈페이지를 action으로 연다음 들어가보면 페이지가 호스팅 되어 있는 것을 볼 수 있다.

## 브랜치에 push

![main](/images/2025-07-07-github-hosting-docs-repo/4.png)

- 현재 브랜치 상태이다. docs가 잘 반영된 것을 볼 수 있다.
- 또한 action 주입도 성공적으로 완료되었다.

## 결과

![main](/images/2025-07-07-github-hosting-docs-repo/6.png)

- 사이트가 원하는 방향으로 잘 호스팅 된 것을 볼 수 있다.
