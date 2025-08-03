---
title: "[GitHub 갖고놀기/Action] 깃허브 action으로 자동 reviewers 추가하기"
layout: post
categories: [github, github-action]
tags : [github, github-action]
toc: true
toc_sticky: true
toc_label: 목차
author_profile : true
permalink: /github/action/auto_reviewers
---

> 🥰 깃허브 갖고놀기 기록집
> {: .notice--info}

# 🤔 Intro

- 깃허브 pr시 리뷰어를 설정해서 리뷰어가 approve할 시에만 merge가 가능하도록 하면 코드리뷰나 로직 검증이 가능하기에 원활한 협업이 가능하다. 
- 깃허브 action을 이용하면, 각 브랜치마다 지정된 리뷰어들을 자동으로 할당하는 것이 가능하다.
- 그러므로 오늘은 action을 이용해서 리뷰어 자동할당을 하는 방법을 알아보자!

# 😀 Start!
## 목표
- Github Action을 이용해서 pr 리뷰어를 자동주입하는 방법을 알아본다.

## git action 사용을 위한 토큰 생성
- 기존에 사용하던 action 토큰이 있긴 하나, pr 리뷰어 전용으로 토큰을 하나 더 만들어보자.

![main](/images/2025-06-22-github-action-autoReviewers/1.png)
- settings > developer settings으로 들어간다.
- 특정 레포지토리만 설정 & PR에 적용하기 위해 파인 그레인드 토큰 (Fine-grained)을 선택해서 생성해주자.

![main](/images/2025-06-22-github-action-autoReviewers/2.png)
- Resource Owner는 리뷰어 자동할당기능을 주입할 레포를 선택한다.
- 만료일은 test token 이므로 30일로 짧게 지정하였다.
- 또한 내가 선택한 레포 안에서만 사용이 가능하도록 지정하였다.

![main](/images/2025-06-22-github-action-autoReviewers/3.png)
- 다음으로는 Permission을 지정해준다.
- 각각은 다음과 같이 권한을 선택해준다.
    - Actions : Read and Write
        - WorkFlow 사용하므로 Write까지 선택해야 한다.
    - Metadata
    - Pull Request : Read and Write
- 토큰은 안전한 곳에 잘 보관해둔다!

## action yml 파일 생성
### yml 파일이란?
- YML 파일은 `.yml` 또는 `.yaml` 확장자를 가진 설정(configuration) 파일을 말한다.
- 즉, **앱이나 시스템이 어떤 방식으로 동작해야 하는지**를 정의해 놓은 설정서를 말하는 것이다.
- GitHub Actions, Docker, Kubernetes, CI/CD 설정 등에 자주 사용되며, github action에서는 CI/CD workflow에서 사용된다.

## token 등록
- yml 설정 파일에 secrets.GH_TOKEN라고 하여 secrets 변수에 GH_TOKEN을 등록해야 하므로, 위에서 발급한 토큰을 변수로 추가해야 한다.

![main](/images/2025-06-22-github-action-autoReviewers/5.png)
- repo > settings > Secrets and Variable > Action 을 클릭한다.

![main](/images/2025-06-22-github-action-autoReviewers/6.png)
- New repository secret을 클릭한다.

![main](/images/2025-06-22-github-action-autoReviewers/7.png)
- 아까 받은 GT_TOKEN을 변수로 등록한다.

## assign-reviewers.yml
- 참고로 나는 dev > main으로 pr을 test할 것이므로, 이 파일은 base branch인 main에 올라가 있어야 한다.
  ![main](/images/2025-06-22-github-action-autoReviewers/4.png)
- 해당 설정 파일을 .github/workflows/아래에 넣어준다.

```yml
name: Assign Reviewers by Branch

on:
  pull_request:
    types: [opened, reopened, synchronize]

jobs:
  assign-reviewers:
    runs-on: ubuntu-latest

    steps:
      - name: Assign reviewers based on base branch
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GH_TOKEN }}
          script: |
            const author = context.payload.pull_request.user.login;
            const branch = context.payload.pull_request.base.ref;
            let reviewers = [];

            if (branch === 'dev') {
              reviewers = ['sunJEE0120'].filter(u => u !== author);
            } else if (branch === 'main') {
              reviewers = ['sunJ0120','sunJEE0120'].filter(u => u !== author);
            }
            
            if (reviewers.length > 0) {
                await github.rest.pulls.requestReviewers({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  pull_number: context.payload.pull_request.number,
                  reviewers: reviewers
                });
            } else {
              console.log(`No reviewers to assign. PR author (${author}) is the only candidate.`);
            }
```
- 코드 설명은 다음과 같다.
- 상태 설명
    - opened: PR 생성 시
    - reopened: 닫혔던 PR을 다시 열었을 때
    - synchronize: 커밋을 추가로 push했을 때 이 모든 경우에 reviewers 자동 할당이 실행된다.
- const author = context.payload.pull_request.user.login;
    - PR을 연 작성자의 Github ID를 가져온다.
- const branch = context.payload.pull_request.base.ref;
    - PR 대상 브랜치를 가져온다.

```javascript
if (branch === 'dev') {
  reviewers = ['sunJEE0120'].filter(u => u !== author);
} else if (branch === 'main') {
  reviewers = ['sunJ0120','sunJEE0120'].filter(u => u !== author);
}
```
- 브랜치에 따라 reviewers 아이디를 설정한다.
- filter를 이용해 author는 리뷰에서 제외하도록 한다.

```javascript
await github.rest.pulls.requestReviewers({
  owner: context.repo.owner,
  repo: context.repo.repo,
  pull_number: context.payload.pull_request.number,
  reviewers: reviewers
});
```
- 깃허브 REST API로 리뷰어를 자동 요청하는 비동기 로직이다.
    - 이를 통해 github action 실행시 리뷰어가 자동 등록된다.

## TEST
- 이제 리뷰어 자동 주입이 잘 반영 되었는지 TEST 하기 위해 PR을 하나 생성해서 리뷰어를 살펴보자.
- 나의 경우는 pr 생성을 위해 dev에서 이미지 파일 두개를 제거한 후 dev > main으로 가는 pr을 하나 생성하였다.

![main](/images/2025-06-22-github-action-autoReviewers/8.png)
- Create pull request를 클릭한다.

![main](/images/2025-06-22-github-action-autoReviewers/9.png)
- Action이 실행되고 나면 다음과 같이 Assign Reviewers 라는 이름의 yml 파일이 오류없이 잘 실행되었다는 것을 알 수 있다.

![main](/images/2025-06-22-github-action-autoReviewers/10.png)
- Action 성공 후 Reviewers도 설정 했던대로 자동으로 잘 들어온 것을 확인할 수 있다:)
