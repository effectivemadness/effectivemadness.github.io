---
title:  " 그날 내 블로그가 멈췄다. - Github Workflow deprecated 해결하기"
date:   2025-03-02 233:29:10+0900
categories: [TIL]
tags: [tech]
---
여러분이 지금 보고 있는 블로그는 지킬 + Github Pages 를 사용하고 있는데요, 새 포스팅을 .md파일로 작성하고 Repo에 Push를 하고 나면 Github Action Workflow가 .md 파일을 읽어 정적 리소스로 빌드하고 퍼블리시를 해 줍니다.

한달 전쯤이었을까요? 어느 때와 같이 블로그 글을 추가하고, 포스팅을 하기 위해 Repo에 Push를 했습니다. 잠깐 기다리면 블로그에서 방금 작성한 글을 볼 수 있겠지... 하며 새로고침을 했습니다.

업데이트가 되지 않았군요. 가끔 사람 몰리는 때는 Github Action Runner가 큐에 넣은 Job을 들고 가기까지 조금 걸리더라구요.

몇분 뒤에도 업데이트가 되지 않았군요. Job을 픽업하고 나서도 조금 걸릴 때가 있더라구요.

5분정도면 거의 백퍼센트 될 법 한데... 아직도 되지 않았네요. 문제가 있어 보입니다.

Github Repo에 가니 Workflow가... Fail입니다? 

![workflow fail](/assets/img/A642F6DB-1361-4682-9948-B5C3A15E1FC0.png)

그날, 제 블로그가 잠시 꺼졌습니다.

![img](/assets/img/C60FCA7C-5586-4A66-923F-5244B6DC0D4B.jpg)

---

충격을 뒤로하고, 원인을 빠르게 찾아 블로그에 글을 업데이트해야합니다. 열심히 쓴 글을 날짜 넘겨서 올리기는 너무 아쉽거든요. 다행히도 Github Action Workflow 에러를 보니 생각보다 명확하게 원인을 이야기해주고 있습니다.

> This request has been automatically failed because it uses a deprecated version of `actions/upload-artifact: v3`. Learn more: https://github.blog/changelog/2024-04-16-deprecation-notice-v3-of-the-artifact-actions/

워크플로우에서 사용하는 것 중, 어떤 부분의 버전이 Deprecated 되었다고 합니다. 같이 제공해준 Github 공식 Blog의 내용을 보면, `actions/upload-artifact`와 `actions/download-artifact` 의 v3 버전을 2025년 1월 30일부로 사용할 수 없다는 내용으로 시작합니다.

![deprecation notice blog post](/assets/img/252AAFE4-C1A4-42A4-B638-FA3850F7E447.png)

Github Workflow는 repo의 `.github/workflows` 디렉토리에 있습니다. 공식 블로그에서는 deprecated 되는 것을 마이그레이션 하는 방법에 대한 공식 문서도 있긴 했는데요. 저 같은 경우는 지킬 테마의 템플릿 Repo를 Fork 해서 사용중이었기에, 원본 템플릿 Repo 를 가보니 새로운 버전을 사용하는 workflow 파일이 있었습니다. 해당 파일을 복사 해 제 repo의 파일을 덮어쓰고 다시 배포하자, 글이 정상적으로 배포되었습니다.

![workflow success](/assets/img/1F79260D-A986-468F-8C0E-744D6DB172CA.png)

---

블로그 내용을 보니, 꼭 이런 Workflow 에서 사용되는 구성요소의 Deprecation 뿐만 아니라 Github 에서 사용되는 다양한 구성요소들에 대한 새로운 Release와 Deprecation 내용이 많더라구요. 크게 다가온 것은 Github Hosted Action Runner의 `ubuntu-latest` 태그가 가르키는 버전이 달라지는 것이라던가, 사용할 수 있는 이미지에서 ubuntu 20.04가 제외된다거나 하는 점이 있습니다. 신기한 것은 ubuntu 20.04 이미지를 사용하고 있는 Workflow의 경우, 3월 중 매주 특정시간에 무조건 에러가 뜨게 해 공지를 보지 못한 사람들도 인지할 수 있게 하더라구요.(이걸 `brownout` 이라고 하나 봅니다.)

업무 중에서도 이미지나 배포 파이프라인에서 Deprecate 되는 것들을 F/up 할 일이 있는데, 업무 외에서 이런 일이 벌어지니 신기하기도 하고 알고 있던 것을 활용할 수도 있어 기분이 좋았습니다.