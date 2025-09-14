---
title: "🎱 깃허브 브랜치 전략에 대한 고찰"
layout: post
categories: [project, github]
tags: [project, github]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: true
permalink: /project/2/github
---

# 🖤 Intro
이번에 2차 프로젝트를 진행하면서, 1차 프로젝트때 스스로 아쉬웠던 부분을 메꿔보고자 여러 방안을 생각했었다.

아쉬웠던 것 중 하나는, 내 이름을 딴 브랜치에서만 작업하다보니 먼저 완성된 작은 기능별로 pr을 올려서 통합할 수 없었던것!

그래서 이번에는 좀 더 깃허브 전략을 세부적으로 가져가기로했다.

이미 BE, ML은 이런 방식으로 진행하고 있는 상황이라, FE 예시를 통해 이번 프로젝트에서 깃허브 전략을 어떻게 가져가고자 하는지를 알아보자.

# 🩶 Start

## Fork...Upstream...브랜치 네이밍...

이번에 사용하고자 하는 전략은, 우리 레포를 fork해온 다음에, fork 레포를 origin으로 두고 우리 원본 레포를 upstream으로 둬서 관리하는 전략이다

이전 프로젝트때는 그냥 원본 레포 하나만 두고 작업했었는데, 이번에 이 방식을 택하게 된 결정적 이유중 하나는 우리 팀원들 중 나만 브랜치 네이밍 전략을 다르게 가져가기로 했기 때문이다. 

나 혼자 브랜치 명이 다르면 다른 팀원들이 브랜치 땡겨왔을때 헷갈릴 것이 분명하기에…기능별 세부 브랜치명을 내 fork에만 두고, 공통 규칙인 우리 “이름” 브랜치만 원본 레포에 남겨두기로 한 것이다.

이것 외에도 이 fork-upstream 전략을 사용하면 다음과 같은 이점이 있다.

### 1. 코드 품질 관리 측면

- 모든 코드가 PR을 통해서만 원본에 진입할 수 있기에 강제 리뷰 기능을 가진다.
- 직접 푸시로 인한 실수를 방지할 수 있다.
- CI/CD 파이프라인을 더 안전하게 운영할 수 있다.
- 또한, 오픈소스 기여 방식을 미리 학습할 수 있다는 장점도 있다.

### 2. 개인 작업 백업 가능

- 특히 fork본이 존재하기 때문에, 원본 레포에 문제가 생겨도 백업이 가능하다.
  - 또한, 멀티 레포 관리 경험이 생겨 나중에 MSA시 유용하다.

## 그렇다면 브랜치 네이밍은 어떤 식으로 하는데?

처음 이 방식을 사용하기로 하면서, 가장 헷갈렸던 부분이 바로 브랜치 네이밍을 하는 방식이었다.

처음 시도했던 BE, ML 레포에서는 이걸 제대로 하지 못했었는데… FE레포에서는 제대로 잡고 들어가려고 한다.

> ❤️‍🔥 브랜치 네이밍은 보통 “pr 규칙/개발중인 기능” 이렇게 구성한다.

나는 처음에 내 개인 이름 브랜치에서 분기한거니까 이름을 넣어야 하나? 싶었는데, 어차피 우리 팀 브랜치 규칙상 원본 이름 브랜치 > 다음이 각각 팀 브랜치 이기도 하고, 이렇게 되면 어차피 원본 이름 브랜치 아래에서 분기한거라는 것이 보장되기 때문에 굳이 이름을 넣을 필요는 없다고 한다.

그래서 이번에 내가 FE 레포에서 가져갈 브랜치 리스트는 다음과 같다.

```bash
1. feat/chat-ui
   → 기본 레이아웃, 채팅창 틀
2. feat/message-component  
   → 메시지 말풍선, 사용자/봇 구분
3. feat/input-handler
   → 텍스트 입력, 전송 기능
4. feat/api-integration
   → 백엔드 API 연동
5. feat/chat-history
   → 이전 대화 로드, 스크롤 관리
6. feat/typing-indicator
   → 타이핑 중 표시, UX 개선
```

이를 플로우로 보면, 다음과 같다.

![image.png](/images/2025-09-14-shinhanDSsw-project2-3/1.png)

## 개발 환경 맞추기

### 1. GitHub에서 원본 레포를 Fork > Fork한 레포를 로컬에 클론

```bash
sspur@sunj-PC MINGW64 /c/developer/GitHub
$ git clone https://github.com/sunJ0120/OhgoodpayFE.git
Cloning into 'OhgoodpayFE'...
remote: Enumerating objects: 809, done.
remote: Counting objects: 100% (131/131), done.
remote: Compressing objects: 100% (103/103), done.
remote: Total 809 (delta 30), reused 78 (delta 16), pack-reused 678 (from 1)
Receiving objects: 100% (809/809), 7.21 MiB | 9.51 MiB/s, done.
Resolving deltas: 100% (386/386), done.
```

### 2. upstream으로 원본 레포를 등록

```bash
sspur@sunj-PC MINGW64 /c/developer/GitHub/OhGoodpayFE (main)
$ git remote add upstream https://github.com/OhGoodTeam/OhgoodpayFE.git

sspur@sunj-PC MINGW64 /c/developer/GitHub/OhGoodpayFE (main)
$ git remote -v
origin  https://github.com/sunJ0120/OhgoodpayFE.git (fetch)
origin  https://github.com/sunJ0120/OhgoodpayFE.git (push)
upstream        https://github.com/OhGoodTeam/OhgoodpayFE.git (fetch)
upstream        https://github.com/OhGoodTeam/OhgoodpayFE.git (push)

```

### 3. 초기 브랜치 등록

```bash
sspur@sunj-PC MINGW64 /c/developer/GitHub/OhGoodpayFE (sunjung)
$ git branch -d feat/sunjung/chat-ui
Deleted branch feat/sunjung/chat-ui (was b4c54c3).
```
나의 경우는, chat-ui를 짜는 것부터 시작할 계획이라 다음과 같이 구성하였다.

```bash
sspur@sunj-PC MINGW64 /c/developer/GitHub/OhgoodpayFE (feat/chat-ui)
$ git fetch upstream
From https://github.com/OhGoodTeam/OhgoodpayFE
 * [new branch]      dev        -> upstream/dev
 * [new branch]      eunhyo     -> upstream/eunhyo
 * [new branch]      gaeun      -> upstream/gaeun
 * [new branch]      gahee      -> upstream/gahee
 * [new branch]      hwajun     -> upstream/hwajun
 * [new branch]      login      -> upstream/login
 * [new branch]      main       -> upstream/main
 * [new branch]      minjung    -> upstream/minjung
 * [new branch]      mypage     -> upstream/mypage
 * [new branch]      pay        -> upstream/pay
 * [new branch]      recodash   -> upstream/recodash
 * [new branch]      shorts     -> upstream/shorts
 * [new branch]      sunjung    -> upstream/sunjung
```
일단 현재 데이터팀 브랜치에 올라가 있는 commit들을 땡겨오기 위해 upstream에 있는 팀 브랜치를 전부 가져왔다.

```bash
sspur@sunj-PC MINGW64 /c/developer/GitHub/OhgoodpayFE (feat/chat-ui)
$ git rebase upstream/recodash
Current branch feat/chat-ui is up to date.
```
이전 작업물 있을 수도 있으니 땡겨오기~

```bash
sspur@sunj-PC MINGW64 /c/developer/GitHub/OhgoodpayFE (feat/chat-ui)
$ git rebase upstream/dev
Current branch feat/chat-ui is up to date.
```
혹시 몰라서 dev에 있는것도 땡겼는데 아직 dev에는 아무것도 없었다.

### 4. 작업 후 PUSH

```bash
sspur@sunj-PC MINGW64 /c/developer/GitHub/OhGoodpayFE (feat/chat-ui)
$ git commit -m "[sunjung] feat: 챗봇 1차 ui 구성 완료 및 컴포넌트 분리 완료"
[feat/chat-ui 246668f] [sunjung] feat: 챗봇 1차 ui 구성 완료 및 컴포넌트 분리 완료
 16 files changed, 455 insertions(+), 10 deletions(-)
 delete mode 100644 src/features/recommend/.gitkeep
 delete mode 100644 src/features/recommend/component/.gitkeep
 create mode 100644 src/features/recommend/component/chat/ChatInput.css
 create mode 100644 src/features/recommend/component/chat/ChatInput.jsx
 create mode 100644 src/features/recommend/component/chat/ChatToggle.css
 create mode 100644 src/features/recommend/component/chat/ChatToggle.jsx
 create mode 100644 src/features/recommend/component/chat/MessageBubble.css
 create mode 100644 src/features/recommend/component/chat/MessageBubble.jsx
 create mode 100644 src/features/recommend/component/chat/ProfileAvatar.css
 create mode 100644 src/features/recommend/component/chat/ProfileAvatar.jsx
 create mode 100644 src/features/recommend/layout/chat/ChatLayout.css
 create mode 100644 src/features/recommend/layout/chat/ChatLayout.jsx
 create mode 100644 src/pages/recommend/chat/Chat.css
 create mode 100644 src/pages/recommend/chat/Chat.jsx
 create mode 100644 src/shared/assets/img/chat_profile.png
```
그렇다면 이제 작업 후에 push를 어떻게 해야 하는지를 알아보자.
우선 add commit은 너무 당연하니까...일단 commit으로 올려준다. 

```bash
sspur@sunj-PC MINGW64 /c/developer/GitHub/OhGoodpayFE (feat/chat-ui)
$ git remote -v
origin  https://github.com/sunJ0120/OhgoodpayFE.git (fetch)
origin  https://github.com/sunJ0120/OhgoodpayFE.git (push)
upstream        https://github.com/OhGoodTeam/OhgoodpayFE.git (fetch)
upstream        https://github.com/OhGoodTeam/OhgoodpayFE.git (push)
```
그 다음에 remote를 확인해보면, origin이 내 fork 레포로 되어 있고 upstream이 원본 레포로 되어 있는 것을 볼 수 있다.

이 상황에서 우선 우리 fork 레포에 push를 해줘야 한다.

```bash
git push origin feat/chat-ui
```
이렇게 해준 다음에 이제 원본 레포에 PR을 날려주면 된다.

![image.png](/images/2025-09-14-shinhanDSsw-project2-3/2.png)

이렇게 해주면 손쉽게 작은 단위 기능별 리뷰 및 통합이 가능~
