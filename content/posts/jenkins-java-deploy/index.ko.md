---
title: "Jenkins로 Jar 파일 배포하기"
date: 2019-05-30
categories:
  - jenkins
image: "../../images/jenkins.webp"
tags:
  - ci/cd
  - jenkins
translationKey: "posts/jenkins-java-deploy"
---

이번 글에서는 빌드한 Jar 파일을 다른 서버로 전달하는 배포 Job을 만들어 보겠습니다. 구조 자체는 단순합니다. [지난 글](../../../05/30/jenkins-javabuild)에서 만든 Job으로 Jar 파일을 생성하고, 그 결과물을 다른 서버로 전송한 뒤 실행하면 됩니다. 원래는 하나의 흐름으로 묶고 싶었지만, 당시에는 서버 역할을 분리해 두는 편이 더 자연스러운 구조였습니다.

어쨌든 Jar 파일을 SSH로 전송하는 Job을 만들어 보겠습니다. 이미 Jar 파일을 빌드하는 Job은 만들어 두었으니 그 산출물을 이용합니다.

## 산출물 확보

먼저 새 Job의 작업 공간으로 Jar 파일을 어떻게 가져올지 생각해야 합니다. 가장 단순한 방법은 이전 Job의 빌드 후 단계에서 쉘 명령으로 복사하는 것입니다. 저도 실제로 처음엔 이 방법부터 시도했습니다.

하지만 파일 경로나 Jar 파일 이름이 바뀌면 명령도 바꿔야 해서 그다지 좋은 방법은 아니라고 느꼈습니다. 그래서 이번에는 플러그인을 사용했습니다. Jenkins 메인 화면에서 `Manage Jenkins`, `Manage Plugins` 순서로 들어가 설치 화면으로 이동합니다. 그다음 `Available` 탭을 열고 오른쪽 위 `Filter`에 `Copy`를 입력합니다.

![Jenkins Artifact Install](jenkins_artifactInstall.webp)

`Copy Artifact` 플러그인이 목록에 나타나는지 확인한 뒤 설치합니다. 가능하면 플러그인 설치나 업데이트 후에는 Jenkins를 다시 시작하는 편이 좋습니다. 재시작 중에는 다음과 같은 화면이 표시됩니다.

![Jenkins Restart](jenkins_restart.webp)

터미널에서 `service jenkins restart`로 직접 재시작할 수도 있지만, 플러그인 설치 후 바로 재시작하는 옵션도 있습니다. 재시작이 끝나면 원래 화면으로 돌아갑니다.

재시작이 완료되고 플러그인이 정상 설치되었는지 확인한 뒤, Job에서 제대로 쓸 수 있는지 살펴봅니다. 이 플러그인은 Job이 끝난 뒤 산출물을 저장하도록 설정할 수 있고, 저장된 산출물을 다른 Job에서 가져와 작업 공간에 복사할 수도 있습니다. 그래서 먼저 복사 원본이 되는 Job에 산출물 저장 단계를 추가하겠습니다.

이전에 만든 `JavaBuild` Job 설정으로 들어가 `Post-build actions` 탭에서 `Archive the artifacts`를 선택한 뒤 저장할 산출물 경로를 입력합니다.

![Jenkins Artifact Post](jenkins_artifactpost.webp)

Job에 변경 사항을 적용한 뒤 저장합니다. 빌드하지 않으면 의도대로 동작하는지 알 수 없는 점이 Jenkins의 몇 안 되는 단점 중 하나인 것 같습니다만, 그래도 확인은 중요합니다.

![Jenkins Artifact Post Check](jenkins_artifactpostcheck.webp)

정상적으로 빌드되었습니다. 저장된 산출물은 Job 메인 화면에서 확인할 수 있습니다.

![Jenkins Artifact Post Check 2](jenkins_artifactpostcheck2.webp)

의도대로 빌드한 파일만 저장되었습니다. [^1] `*.jar`로 지정했지만 실제로 빌드되는 파일이 하나뿐이니 당연한 결과입니다. 이 단계로 원본 Job 설정은 끝났습니다. 다음 단계로 넘어가 보겠습니다.

## 산출물 전달

이번에는 배포 전용 Job으로 `JavaDeploy`를 새로 만들었습니다. 먼저 저장한 산출물을 작업 공간으로 가져오는 설정이 필요합니다.

Job 설정 화면에서 `Build` 탭으로 가서 `Add Build Step`을 누르면 `Copy artifacts from another project` 항목이 보입니다.

![Jenkins Artifact Config](jenkins_artifactconfig.webp)

`Project name`에는 다른 Job 이름을 선택합니다. 저는 방금 만든 Job을 지정했습니다. `Which build`에서는 어느 빌드에서 산출물을 가져올지 정할 수 있는데, 보통은 `Lastest successful build`가 적당합니다. `Stable build only`도 체크해 두면 좋습니다. 나머지는 원본 파일 경로와 복사 대상 경로를 지정하면 됩니다.

복사하고 싶지 않은 파일이 있다면 `Artifacts not to copy`에 적습니다. 저는 빌드한 Jar 파일만 복사하도록 아래처럼 설정했습니다.

![Jenkins Artifact Config 2](jenkins_artifactconfig2.webp)

이제 다시 Job을 빌드해 의도대로 동작하는지 확인합니다.

![Jenkins Artifact Copied](jenkins_artifatccopied.webp)

정상적으로 빌드가 끝났고 산출물도 복사되었습니다.

이제 이 산출물을 다른 환경으로 전송해 보겠습니다.

## 산출물 전송 1

SSH로 파일을 전송하려면 `Publish over SSH` 플러그인이 필요합니다. 이 플러그인을 사용하면 SSH 연결을 통해 파일 전송과 원격 셸 명령 실행을 할 수 있습니다. 플러그인 설치 화면에서 `ssh`로 검색하면 목록에서 확인할 수 있습니다.

![Jenkins Arfact Copied](jenkins_artifatccopied.webp)

플러그인 설치와 Jenkins 재시작을 마친 뒤에는 연결 대상 설정이 필요합니다. Jenkins 설정에서 `Configure System`으로 들어가면 `Publish over SSH` 설정 항목이 생깁니다.

![Jenkins Publish Over SSH Server Setting](jenkins_publishoversshserversetting1.webp)

`Key`에 공개키를 넣어도 접속할 수 있지만, 아직 그 설정은 하지 않았기 때문에 여기서는 일반적인 ID와 비밀번호 방식으로 진행합니다. `SSH Servers`의 `Add` 버튼을 누르면 접속 정보를 입력할 수 있는 필드가 나타납니다. `Name`에는 원하는 이름을, `Hostname`에는 실제 IP 주소나 호스트 이름을 넣습니다. 저는 SSH 서버가 따로 없어서 제 Mac에 연결해 보았습니다. 내부 IP와 Mac 계정을 그대로 사용했습니다. [^2]

`Username`에 계정을 입력하고, 비밀번호로 접속하려면 `Advanced`를 열어 `Use password authentication, or use a different key`를 체크한 뒤 `Passphrase / Password`에 비밀번호를 넣습니다. SSH 기본 포트는 22이므로, 포트가 열려 있는지도 확인해야 합니다. 정보를 모두 입력하면 `Test configuration`으로 연결 가능 여부를 확인할 수 있습니다.

![Jenkins Publish Over SSH Server Setting 2](jenkins_publishoversshserversetting2.webp)

설정이 맞으면 `Test configuration`을 눌렀을 때 `Success`가 표시됩니다. 저장하고 Job으로 돌아갑니다.

## 산출물 전송 2

Job 설정에서 `Build Environment` 탭을 보면 `Send files or execute commands over SSH before the build starts`와 `Send files or execute commands over SSH after the build runs`가 생겨 있습니다. 산출물을 옮긴 뒤 SSH를 실행하고 싶으니 후자를 선택합니다. 물론 `Build` 탭에도 같은 메뉴가 있으니 거기서 설정해도 됩니다.

`Name`에는 Jenkins 설정에서 등록한 SSH 대상 서버를 선택합니다. `Source files`에는 전송할 파일 경로를 넣습니다. 옵션으로 `Remove prefix`를 쓰면 지정한 경로까지 제거할 수 있고, `Remote directory`에서는 전송할 원격 폴더를 지정할 수 있습니다. [^3]

![Jenkins Transfer](jenkins_transfer1.webp)

제 설정은 이렇습니다. 폴더 구조를 그대로 옮기고 싶지는 않아서 파일만 전송하도록 했습니다. 정상이라면 사용자 홈 아래의 `fromJenkins` 폴더로 전달될 것입니다. 혹시 모르니 전송 대상 폴더의 권한과 소유자도 확인해 두는 게 좋습니다. 이제 Job을 빌드해 봅니다.

![Jenkins Transfer 2](jenkins_transfer2.webp)

빌드는 성공했습니다. 콘솔에는 전송에 성공한 파일 수가 표시됩니다. 실제로 전송되었는지 Mac 쪽에서 확인해 보겠습니다.

![Jenkins Transferred](jenkins_transfered.webp)

여기서도 확인할 수 있었습니다. 이제 파일 전송 작업은 성공입니다. 기왕이면 Mac에서 Jar 파일도 실행해 보겠습니다.

![Jenkins JAR](jenkins_jar.webp)

테스트용 데모라 `test`라는 문자열만 출력하게 해 두었지만, 어쨌든 실행은 성공입니다. 이번 글의 작업도 여기서 끝입니다.

## 마지막으로

이번 글에서는 파일 복사와 전송만 연결했지만, 여기서 조금만 더 확장하면 기존 프로세스를 종료하고 새 Jar를 실행하는 배포 흐름까지 Jenkins 안에서 구성할 수 있습니다. 결국 어디까지 자동화할지는 프로젝트 구조와 운영 방식에 달려 있습니다.

이 연작은 여기까지지만, Jenkins를 처음 만질 때 기본 흐름을 잡는 데는 이 정도 구성만으로도 충분히 도움이 된다고 생각합니다. 나중에 더 실무에 가까운 구성이 생기면 이어서 정리해 보겠습니다.

[^1]: 이 화면은 zip 파일처럼 보이지만, 실제로는 zip으로 다운로드할 수 있다는 뜻입니다. 산출물은 폴더 구조 그대로 저장됩니다.
[^2]: 다만 스크린샷에는 만약을 대비해 실제 정보는 적지 않았습니다.
[^3]: 기본적으로는 SSH로 접속한 사용자의 홈 디렉터리가 기준입니다.
