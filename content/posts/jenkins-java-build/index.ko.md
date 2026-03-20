---
title: "Jenkins에서 Java 프로젝트 빌드"
date: 2019-05-26
categories:
  - jenkins
image: "../../images/jenkins.webp"
tags:
  - ci/cd
  - jenkins
translationKey: "posts/jenkins-java-build"
---

이번 글에서는 Jenkins에서 Java 프로젝트를 빌드하는 Job을 만드는 과정을 정리해 보겠습니다. 지난 글에서도 잠깐 언급했지만, 당시 받은 요청은 "Git에 있는 Spring Boot 애플리케이션을 자동으로 빌드하는 Job을 만들어 달라"는 것이었습니다. 저장소를 확인하고 Java와 Gradle을 준비한 뒤, 바로 Job 설정으로 넘어갔습니다.

당시와 비슷한 환경을 만들기 위해, 미리 Spring Boot 프로젝트를 만들어 Git에 올려 둔 상태입니다.

## Job이란

Jenkins를 쓰는 이유는 결국 이 Job 때문입니다. Job은 `자동화된 작업`으로, 실행 조건과 빈도, 작업 내용을 모두 포함할 수 있습니다. 예를 들어 Git 저장소에 누군가 Push하면 Job이 실행되도록 설정할 수 있고, 실행 시 자동으로 Git을 Pull하도록 만들 수도 있습니다.

즉 Jenkins를 서버에서 띄워 두고 필요할 때 Job을 구성해 두면, 수작업을 크게 줄일 수 있습니다. 이번 글에서는 이 Job을 만드는 과정을 정리하겠습니다.

## Job 생성

먼저 Jenkins 메인 페이지로 들어갑니다. 아직 Job이 없는 상태라면 메인 화면에 `create new jobs` 링크가 보입니다. Job이 없다면 여기서 바로 만들 수 있고, 이미 Job이 있다면 왼쪽 메뉴의 `New item`을 눌러 새 Job 생성 화면으로 들어갈 수 있습니다.

![Jenkins Mainpage](jenkins_mainpage.webp)

`Enter an item name`에 Job 이름을 입력합니다. 아래에는 어떤 종류의 Job을 만들지 고르는 템플릿이 있습니다. 여기서는 `Freestyle project`를 선택합니다. 참고로 Job 이름과 같은 이름의 폴더가 Jenkins의 `Workspace` 아래에 생성되므로, 가능하면 반각 영문자와 공백 없이 만드는 편이 좋습니다.

![Jenkins Create Job](jenkins_createjob.webp)

이름과 템플릿을 선택했다면 `Ok`를 눌러 Job을 생성합니다.

![Jenkins Job Main](jenkins_jobmain.webp)

여기가 Job 설정의 메인 화면입니다. 상단 탭의 역할은 다음과 같습니다.

- `General`: Job 전반의 설정입니다. 설명을 적거나, 빌드 시 이전 빌드를 삭제할지 같은 설정을 할 수 있습니다.[^1]
- `Source Code Management`: 버전 관리 설정입니다. Git과 Subversion을 지원합니다. 이번에는 Git을 사용합니다.
- `Build Triggers`: Job을 어떤 방식으로 실행할지 정하는 트리거 설정입니다. URL로 원격 실행할지, 정기 실행할지 등을 설정할 수 있습니다.
- `Build Environment`: 빌드 시 사용할 환경 설정입니다. Job은 기본적으로 `Workspace`라는 공간을 사용하고, 빌드 중 생성된 파일은 모두 그 폴더에 저장됩니다. 기본 경로는 `/var/lib/jenkins/workspace/[생성한 Job 이름]`입니다.
- `Build`: 실제 빌드 작업을 지정하는 탭입니다. 여기서 주요 처리가 이뤄집니다.
- `Post-build Actions`: 빌드 후 수행할 작업을 지정합니다. 다른 Job을 빌드하는 등의 동작도 여기서 설정할 수 있습니다.

탭에 보이는 항목은 설치한 플러그인에 따라 늘어나거나 줄어들 수 있습니다. 나중에 따로 다룰 수도 있겠지만, 예를 들면 `Publish over SSH` 같은 플러그인도 있습니다.

## Git 저장소 설정

이제 본격적으로 작업을 만들어 보겠습니다. 먼저 Git에서 Java 코드를 가져와야 하므로 `Source Code Management`에서 Git을 선택합니다.

![Jenkin Git Config](jenkins_gitconfigure.webp)

`Repository URL`에 Git 저장소 주소를 입력합니다. 처음에는 아무 값도 없어서 연결 오류가 보이지만, URL을 넣으면 자동으로 연결을 시도하고 문제가 없으면 오류가 사라집니다. `Credentials`에서는 접속에 사용할 인증 정보를 선택합니다. 처음에는 없으니 `Add`를 눌러 새 자격 증명을 추가합니다.

지금 만든 저장소는 브랜치가 하나뿐이지만, 특정 브랜치만 가져오고 싶다면 `Branches to Build`에 브랜치 이름을 입력하면 됩니다.

![Jenkins Git Credential](jenkins_gitcredential.webp)

그 외에는 기본값으로 두고, Git ID와 비밀번호만 입력한 뒤 `Add`를 누르면 됩니다. 여기서 입력한 인증 정보는 전역 설정이므로 다른 Job에서도 사용할 수 있습니다.

인증 정보 입력이 끝나면 `Save`를 눌러 저장합니다. 그러면 다음과 같은 화면으로 돌아갑니다.

![Jenkins Job Saved](jenkins_jobsaved.webp)

## 빌드해 보기(1)

이제 Git에서 Pull하는 작업 설정은 끝났습니다. 실제로 의도대로 동작하는지 확인하기 위해 빌드해 보겠습니다. 왼쪽 메뉴에서 `Build now`를 클릭하면 잠시 후 왼쪽 아래 `Build History`에 첫 빌드가 표시됩니다. `#1`에 파란색 원이 붙어 있으면 성공, 노란색이면 불안정, 빨간색이면 실패입니다.

빌드가 성공하면 `#1`을 클릭해 빌드 세부 정보를 봅니다.

![Jenkins Git Job](jenkins_gitjob1.webp)

실제 빌드에서 어떤 작업이 수행됐는지 보려면 왼쪽 메뉴에서 `Console Output`을 클릭하면 됩니다. Linux 콘솔처럼 로그가 표시됩니다.

![Jenkins Git Job 2](jenkins_gitjob2.webp)

설정한 대로 Git 저장소에서 Pull이 됐고, 브랜치도 master로 확인됩니다. 커밋 메시지도 출력됩니다.

Linux 콘솔에서 `/var/lib/jenkins/workspace/` 아래를 직접 확인할 수도 있지만, Jenkins 웹 콘솔에서도 작업 공간을 볼 수 있습니다. `Back to Project`를 누른 뒤 `Workspace`를 클릭하면 다음과 같은 화면이 나옵니다.

![Jenkins Git Job 3](jenkins_gitjob3.webp)

실제로 Git Pull이 정상적으로 수행되어 폴더와 파일이 생성된 것을 여기서 확인할 수 있습니다.

## Gradle 설정

이제 Gradle 빌드 설정을 해 보겠습니다. 톱니바퀴 아이콘이 있는 `Configure`를 눌러 Job 설정 화면으로 돌아갑니다. 목적은 Gradle로 Jar 파일을 빌드하는 것이므로, Git에서 Pull한 뒤의 작업이 됩니다.

`Build` 탭으로 이동해 `Add build step`을 누르고, 드롭다운에서 `Invoke Gradle script`를 선택합니다. 그리고 `Advanced...`를 클릭합니다.

![Jenkins Gradle Config](jenkins_gradleconfigure1.webp)

여기서 `Invoke Gradle`에는 이전 글에서 등록한 Gradle을 선택합니다.

그다음 `Use Gradle Wrapper` 아래의 `Tasks`에 `bootJar`를 입력합니다. 이 명령으로 실행 가능한 Jar 파일이 생성됩니다. 또한 `bootJar`를 쓰려면 `build.gradle` 경로를 정확히 지정해야 하므로, Gradle이 Workspace의 Job 폴더에서 실행된다는 점을 고려해 `Build File`에도 맞는 경로를 적어 줍니다.

물론 Gradle 명령에 빌드 파일 경로를 직접 지정할 수도 있습니다. `bootJar`뿐 아니라 `-b SpringBootDemo/build.gradle bootJar`처럼 써도 빌드할 수 있습니다.

![Jenkins Gradle Config 2](jenkins_gradleconfigure2.webp)

이제 Gradle 빌드 설정은 끝났습니다. Git 때처럼 다시 검증해 봅시다.

## 빌드해 보기(2)

빌드 절차도 Git 때와 크게 다르지 않습니다. `Build Now` -> `#2` -> `Console Output` 순서로 빌드 로그를 확인하면 됩니다.

![Jenkins Gradle Build](jenkins_gradlebuild.webp)

무사히 Gradle 빌드가 끝났습니다. 이제 Jar 파일이 제대로 만들어졌는지 확인해 봅시다.

![Jenkins Gradle Build JAR](jenkins_gradlebuildjar.webp)

`build/libs` 폴더에 Jar 파일이 잘 생성된 것을 확인할 수 있습니다.

## 마지막으로

Jenkins에서 할 수 있는 일은 여기서 끝이 아닙니다. 빌드가 끝난 뒤 배포 Job을 만들고 연결할 수도 있고, JUnit 연동으로 테스트를 수행하도록 설정할 수도 있습니다. 그래서 다음 글에서는 완성된 Jar 파일을 테스트하고 배포하는 방법을 다뤄 보려고 합니다.

또 Job 설정의 `Build Environment` 탭에서 `Delete workspace before build starts` 옵션을 확인해 두는 것도 좋습니다. Job 실행 전에 Workspace를 먼저 비워 주기 때문에, 이전 빌드에서 남은 파일로 인한 이상 동작을 줄일 수 있습니다.

아직 Jenkins의 모든 기능을 깊게 써 본 것은 아니지만, 플러그인과 Job 구성을 어떻게 조합하느냐에 따라 활용 범위가 꽤 넓다는 점은 확실히 느낄 수 있었습니다. 다음 글에서는 이번에 만든 빌드 Job을 바탕으로 배포까지 이어 보겠습니다.

[^1]: Jenkins의 Job에서 말하는 빌드는 Job 버전에 가까운 이미지입니다. 특정 시점의 Job을 설정부터 실행까지 한 단위로 묶은 것을 빌드라고 합니다. Java 빌드와는 조금 다른 개념이니 혼동하지 않도록 합시다.
