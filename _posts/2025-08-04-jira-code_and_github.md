---
title: "[JIRA 뜯어보기] 6. JIRA 페이지별 소개 - 코드 & 깃허브 연결"
layout: post
categories: [JIRA]
tags: [JIRA, tools, agile]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: true
permalink: /jira/6
---

> 👽 JIRA 뜯어보기 기록집
> {: .notice--info}

# 🤔 Intro

- JIRA의 기능을 페이지별로 하나하나 뜯어보자!

# 😀 Start!

## 코드

- 코드를 통해 JIRA에 우리 깃허브나 비트 버킷 등의 형상관리 툴을 연결할 수 있다.
- 이를 통해 좀 더 쉽게 워크 플로우를 확인하거나 PR 체크가 가능하다.
- 실제로, 예시로 내 알고리즘 스터디 레포지토리를 하나 연결해 보도록 하겠다.

![image.png](/images/2025-08-04-jira-code_and_github/1.png)

### 깃허브 연결

![image.png](/images/2025-08-04-jira-code_and_github/2.png)

- 현재는 내 지라에 활성화된 사이트가 하나밖에 없으므로 이렇게 선택 창이 비활성화 되어 있다.
- Review를 눌러서 다음으로 이동하자.

![image.png](/images/2025-08-04-jira-code_and_github/3.png)

- 해당 어플리케이션을 설치하면 github와 연동이 가능하다.
- Get it now를 눌러서 연동을 시작하자.

![image.png](/images/2025-08-04-jira-code_and_github/4.png)

- 앱을 구성해야 JIRA랑 깃허브를 같이 사용할 수 있다고 한다. 앱 구성을 클릭해주자.

![image.png](/images/2025-08-04-jira-code_and_github/5.png)

- 깃허브와 지라 연동을 시작하자.
- 여기서 Continue를 누르면 연동을 시작할 수 있다.

### 연결 organization 선택

![image.png](/images/2025-08-04-jira-code_and_github/6.png)

![image.png](/images/2025-08-04-jira-code_and_github/7.png)

test를 위해 내 개인 레포인 sunJ0120을 선택한다.

![image.png](/images/2025-08-04-jira-code_and_github/8.png)

- 내 알고리즘 레포에만 지정하기 위해 Select repositories를 선택한다.
- 원하는 레포 또는 전체 레포를 선택하고 Install을 누른다.

![image.png](/images/2025-08-04-jira-code_and_github/9.png)

- “코드”란에 들어가보면, 내 깃허브와 지라가 연동되어 있는 것을 볼 수 있다.
- 현재 업무와 연결되어 있는 pr이 없으므로 확인이 불가능하다. 그러므로 pr을 생성해야 한다.

### JIRA & GITHUB TICKET 연동

- 설명을 보면, 이 둘을 자동으로 연결해서 PR을 자동 링크 시키려면 WorkItem의 키를 브랜치 이름에 담아서 새로운 브랜치를 만들어야 한다.
- epic 하나를 선택해서 키를 복사하고, 브랜치를 만들어보자.

![image.png](/images/2025-08-04-jira-code_and_github/10.png)

SCRUM-1-test-jira

- 목록을 보면, 내 할일 목록들이 전부다 나온다.
- 키를 복사해서 새로운 브랜치를 만들어보자.
- https://sspure1214.atlassian.net/browse/SCRUM-1 이 에픽을 중심으로 만들어보겠다.

```PowerShell
SCRUM-1-test-jira
```

![image.png](/images/2025-08-04-jira-code_and_github/11.png)

![image.png](/images/2025-08-04-jira-code_and_github/12.png)

- 만든 브랜치로 이동한 다음 임시 mark down 파일을 입력했다.
- 이걸 pull해서 올린다음 pr을 만들어보자.

```PowerShell
sspur@sunj-PC MINGW64 /c/developer/GitHub/SinhanDS_AlgorithmST (SCRUM-1-test-jira)
$ git add .

sspur@sunj-PC MINGW64 /c/developer/GitHub/SinhanDS_AlgorithmST (SCRUM-1-test-jira)
$ git commit -m "Test : jira test를 위한 임시 commit"
[SCRUM-1-test-jira a4269e3] Test : jira test를 위한 임시 commit
 1 file changed, 3 insertions(+)
 create mode 100644 algo/src/test_jira/test.md

sspur@sunj-PC MINGW64 /c/developer/GitHub/SinhanDS_AlgorithmST (SCRUM-1-test-jira)
$ git push origin SCRUM-1-test-jira
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 16 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (6/6), 538 bytes | 269.00 KiB/s, done.
Total 6 (delta 2), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/sunJ0120/SinhanDS_AlgorithmST.git
   e1c1deb..a4269e3  SCRUM-1-test-jira -> SCRUM-1-test-jira

```

![image.png](/images/2025-08-04-jira-code_and_github/13.png)

- 업무 키를 pr 제목에 달아서 임시 pr을 생성하였다.

![image.png](/images/2025-08-04-jira-code_and_github/14.png)

- 업무와 연결된 pr들이 잘 나오는 것을 볼 수 있다.
- 📌 굳이 브랜치에 지라 키를 붙이지 않아도, pr 추적은 pr 앞에 붙이기만 하면 되는 것 같다.
  - 컨벤션 통일, CI/CD 자동화, 자동링크 강화 등을 위해 둘 다 쓰는 것이라고 한다.
- 연결된 PR을 누르면 해당 PR로 이동하고, 필터링을 통해 리포지토리나 상태 등으로 필터링해서 확인하는 것도 가능하다.
