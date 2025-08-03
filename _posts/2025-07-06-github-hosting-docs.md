---
title: "[GitHub 갖고놀기] Github 웹 호스팅으로 docs 만들기"
layout: post
categories: [github]
tags : [github]
toc: true
toc_sticky: true
toc_label: 목차
author_profile : true
permalink: /github/io/docs/start
---

> 🥰 깃허브 갖고놀기 기록집
> {: .notice--info}

# 🤔 Intro

- 우리 1차 프로젝트를 진행하며, notion을 사용해서 프로젝트 명세를 정리하는 것 보다는 docs로 정리를 한 번 해보고 싶엇다!
- 그리고 일단..간지난다... 깃허브 호스팅 문서….너무 멋져
- 또한, README에 너무 많은 내용이 들어가면 핵심을 볼 수 없고, 난잡해져서 사용자 가이드와 같은 부분은 docs로 따로 정리하면 좋을 것 같았다.
- 일단 대강 호스팅 하는 방법은 아니까 그걸 이용해서 우리 프로젝트 문서를 한 번 만들어 보자!

# 😀 Start!

## 목표
- 깃허브 웹 호스팅을 이용해서 docs 문서를 만들어보자

## 테마 & 호스팅 시작
- docs를 생성하기 위한 테마는, 유명한 jekyll기반 테마 중 하나인 이것을 이용한다.
- 기본적인 사용 방법은 해당 테마의 리드미, 사이트 기반으로 시도하였다.
- [just-the-docs GitHub 저장소](https://github.com/just-the-docs/just-the-docs)

![main](/images/2025-07-06-github-hosting-docs/1.png)

- 해당 사이트의 use the template를 클릭한다.

![main](/images/2025-07-06-github-hosting-docs/2.png)

```java
sspur@sunj-PC MINGW64 /c/developer/GitHub
$ git clone https://github.com/sunJ0120/test_docs.git
Cloning into 'test_docs'...
remote: Enumerating objects: 14, done.
remote: Counting objects: 100% (14/14), done.
remote: Compressing objects: 100% (14/14), done.
remote: Total 14 (delta 1), reused 8 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (14/14), 7.89 KiB | 7.89 MiB/s, done.
Resolving deltas: 100% (1/1), done.
```

-  “use the template”으로 해당 레포를 가져온 다음에 clone한다.

![main](/images/2025-07-06-github-hosting-docs/3.png)

- 그 다음에 리드미에서 하라는 대로 Gemfile에 다음을 추가한다.

```java
title: Just the Docs Template
description: A starter template for a Jeykll site using the Just the Docs theme!
theme: just-the-docs

url: https://sunJ0120.github.io/test_docs

aux_links:
  Template Repository: https://github.com/sunJ0120/test_docs

plugins:
  - jekyll-default-layout
```

- _config.yml 파일에 다음을 추가한다.
    - plugins을 추가하고,
    - url은 https://내 깃허브 이름.github.io/레포 이름으로 구성한다.
    - aux_links 역시 리드미에 있는대로 https://github.com/[내](https://내) 깃허브 이름/사이트 이름으로 구성한다.

```java
sspur@sunj-PC MINGW64 /c/developer/GitHub/test_docs (main)
$ git add .

sspur@sunj-PC MINGW64 /c/developer/GitHub/test_docs (main)
$ git commit -m "init my docs"
[main ea87179] init my docs
 2 files changed, 7 insertions(+), 2 deletions(-)

sspur@sunj-PC MINGW64 /c/developer/GitHub/test_docs (main)
$ git push origin main
Enumerating objects: 7, done.
Counting objects: 100% (7/7), done.
Delta compression using up to 16 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (4/4), 447 bytes | 447.00 KiB/s, done.
Total 4 (delta 3), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (3/3), completed with 3 local objects.
To https://github.com/sunJ0120/test_docs.git
   6ec2e3b..ea87179  main -> main
```
- config.yml 파일 변경 사항을 내 깃허브 레포에 올린다.

![main](/images/2025-07-06-github-hosting-docs/4.png)

- Settings > Pages > Build and devloyment에서 github actions 선택한다.
- README에 따르면, 기존 프로젝트 안에 docs를 만들고, 거기서 action을 띄우는 방법도 있는데 이는 지금말고 차후에 해보기로 한다.

## 🚨 ERROR! ACTION 호스팅 중 에러
```java
/opt/hostedtoolcache/Ruby/3.3.8/x64/bin/bundle config --local path /home/runner/work/test_docs/test_docs/vendor/bundle
/opt/hostedtoolcache/Ruby/3.3.8/x64/bin/bundle config --local deployment true
Cache key: setup-ruby-bundler-cache-v6-ubuntu-24.04-x64-ruby-3.3.8-wd-/home/runner/work/test_docs/test_docs-with--without--only--Gemfile.lock-ff0d61d1279fe4798d1e1ab5b7404856cce58b21a77047238d496b4c668bb392
Cache hit for: setup-ruby-bundler-cache-v6-ubuntu-24.04-x64-ruby-3.3.8-wd-/home/runner/work/test_docs/test_docs-with--without--only--Gemfile.lock-ff0d61d1279fe4798d1e1ab5b7404856cce58b21a77047238d496b4c668bb392
Received 18742379 of 18742379 (100.0%), 116.1 MBs/sec
Cache Size: ~18 MB (18742379 B)
/usr/bin/tar -xf /home/runner/work/_temp/6f987c64-d62b-45bc-8c37-40aaa95d8845/cache.tzst -P -C /home/runner/work/test_docs/test_docs --use-compress-program unzstd
Cache restored successfully
Found cache for key: setup-ruby-bundler-cache-v6-ubuntu-24.04-x64-ruby-3.3.8-wd-/home/runner/work/test_docs/test_docs-with--without--only--Gemfile.lock-ff0d61d1279fe4798d1e1ab5b7404856cce58b21a77047238d496b4c668bb392
/opt/hostedtoolcache/Ruby/3.3.8/x64/bin/bundle install --jobs 4
The dependencies in your gemfile changed, but the lockfile can't be updated
because frozen mode is set

You have added to the Gemfile:
* jekyll-default-layout

Run bundle install elsewhere and add the updated Gemfile to version control.
If this is a development machine, remove the Gemfile.lock freeze by running
bundle config set frozen false.
Error: The process '/opt/hostedtoolcache/Ruby/3.3.8/x64/bin/bundle' failed with exit code 16
```

- 이 에러 메시지는 Bundler가 “frozen mode”로 실행 중이라서, Gemfile이 바뀌었을 때 자동으로 Gemfile.lock을 갱신하지 못하고 있는 상태를 뜻한다.

> 1. Gemfile에 `jekyll-default-layout` 같은 새 의존성을 추가했지만,
> 2. CI(또는 `bundle install --deployment` 모드)에서는 `Gemfile.lock`을 수정하지 못하도록 잠궈(`frozen`),
> 3. 그래서 잠긴 상태에서 Gemfile이 바뀌었으니 “lockfile을 업데이트할 수 없습니다”라며 오류(Exit code 16)를 내고 있는 것이다.

### 해결책
```java
bundle install
```
- 내 클론된 레포 안에 번들을 설치한다.

```java
sspur@sunj-PC MINGW64 /c/developer/GitHub/test_docs (main)
$ git add Gemfile.lock
warning: in the working copy of 'Gemfile.lock', LF will be replaced by CRLF the next time Git touches it

sspur@sunj-PC MINGW64 /c/developer/GitHub/test_docs (main)
$ git commit -m "Update Gemfile.lock: add jekyll-default-layout"
[main 131e968] Update Gemfile.lock: add jekyll-default-layout
 1 file changed, 10 insertions(+)

sspur@sunj-PC MINGW64 /c/developer/GitHub/test_docs (main)
$ git push origin main
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 16 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 402 bytes | 402.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/sunJ0120/test_docs.git
   ea87179..131e968  main -> main
```
- Gemfile.lock을 커밋한다.

![main](/images/2025-07-06-github-hosting-docs/7.png)
- 워크플로우 업데이트 완료~

## 에러 해결 완료 후 호스팅 완료~

![main](/images/2025-07-06-github-hosting-docs/5.png)
- 사이트가 호스팅 되었다는 문구가 뜨면 사이트에 접근이 가능하다!

![main](/images/2025-07-06-github-hosting-docs/6.png)
- 초기 템플릿 모습이다~ 호스팅이 잘 된 것을 볼 수 있다.
