---
title: "[GitHub 갖고놀기] GitHub Organization 만들기"
layout: post
categories: [github]
tags : [github]
toc: true
toc_sticky: true
toc_label: 목차
author_profile : true
permalink: /github/organization
---

> 🥰 깃허브 갖고놀기 기록집
> {: .notice--info}

# 🤔 Intro

- 원활한 협업 및 팀 프로젝트를 위해 Organization을 만들어서 작업하는 것이 좋다.
- 특히 Organization으로 프로젝트를 관리하면 권한 관리 및, 이슈나 풀리퀘 관리가 수월하여 협업을 위해서는 Organization을 생성하여 협업하는 것이 좋다.
- 그러므로 깃허브 Organization을 만드는 방법을 공부해보고자 한다!

# 😀 Start!

## 목표

- Github Organization을 조직하고 권한을 부여하는 방법을 학습한다.

## Organization 생성
<img src="/images/2025-06-21-github-organization/1.png" style="display: block; margin: 0 auto;" />
- 깃허브에 접속 후 로그인을 해서 내 계정에 접속한다.
- 상단에 + 버튼 클릭 > New organization을 클릭한다.

<img src="/images/2025-06-21-github-organization/2.png" style="display: block; margin: 0 auto;" />
- 여러 organization 옵션이 있는데, Free를 선택해준다.

<img src="/images/2025-06-21-github-organization/3.png" style="display: block; margin: 0 auto;" />
- 이제 test용 Organization을 생성해보자!
- Organization name과 대표 contact email을 설정해주면 된다.

<img src="/images/2025-06-21-github-organization/4.png" style="display: block; margin: 0 auto;" />
- 다음으로 우리 Organization에 추가할 사람을 선택한다.
- 나의 경우는 현재 test용 Organization이므로, 내 부계정을 선택해서 추가해주었다.

<img src="/images/2025-06-21-github-organization/5.png" style="display: block; margin: 0 auto;" />
- 초대를 받은 사람은 자신의 이메일에 온 초대 링크를 타고 들어가면 Organization에 참여가 가능하다!

<img src="/images/2025-06-21-github-organization/6.png" style="display: block; margin: 0 auto;" />
- 현재 Organization에 나와, 내가 추가한 멤버까지 총 두 명의 People이 추가된 것을 볼 수 있다!

## Repository 생성
<img src="/images/2025-06-21-github-organization/7.png" style="display: block; margin: 0 auto;" />
- 조직을 구성했으니 프로젝트에 사용할 Repository를 하나 생성해보자!
- 상단에 Repository를 클릭한다.

<img src="/images/2025-06-21-github-organization/8.png" style="display: block; margin: 0 auto;" />
- new Repository 클릭

<img src="/images/2025-06-21-github-organization/9.png" style="display: block; margin: 0 auto;" />
- Repository 생성 정보들을 입력한다.
- 나의 경우는, test Repo이기 때문에 private로 설정해주었다.

## Team 생성 및 관리
- member 단위로 권한을 관리할 수도 있지만, team 단위로 묶어서 관리도 가능하다.
- 그러므로 Team을 생성하는 방법을 알아보자!

<img src="/images/2025-06-21-github-organization/10.png" style="display: block; margin: 0 auto;" />
- team > new team을 클릭한다.

<img src="/images/2025-06-21-github-organization/11.png" style="display: block; margin: 0 auto;" />
- 다음과 같이  create Team으로 team을 하나 만들어준다.

<img src="/images/2025-06-21-github-organization/12.png" style="display: block; margin: 0 auto;" />
- 해당하는 team에 추가하고 싶은 멤버들을 추가하면 team 생성이 완료된다.

## member 권한 관리
- 팀 프로젝트를 진행할 경우, 팀장의 권한은 admin으로, 팀원의 권한은 Write로 설정해 주는 것 이 좋다.
- 그러므로 멤버별로 권한을 설정하는 방법을 알아보자.

<img src="/images/2025-06-21-github-organization/13.png" style="display: block; margin: 0 auto;" />
- repo로 이동해서 settings를 클릭한다.

<img src="/images/2025-06-21-github-organization/14.png" style="display: block; margin: 0 auto;" />
- Collaborators and teams을 선택한다.

<img src="/images/2025-06-21-github-organization/16.png" style="display: block; margin: 0 auto;" />
- Add people에서 추가할 멤버들을 선택하고, 권한을 부여한다.
- Team 단위로 권한 부여도 가능하다.
- 나의 경우는 내 부계정, 내 본계정 두 개로 test 중이기 때문에 Add people로 진행하였다:)

