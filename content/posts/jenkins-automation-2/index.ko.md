---
title: "Jenkins 자동화하기 2"
date: 2019-05-14
categories:
  - jenkins
image: "../../images/jenkins.webp"
tags:
  - ci/cd
  - jenkins
translationKey: "posts/jenkins-automation-2"
---

지난 글에 이어, 이번에는 Jenkins 초기 설정을 정리해 보겠습니다. 어떤 도구든 처음 세팅이 가장 번거롭지만, 이 구간을 잘 잡아 두면 이후 작업은 훨씬 수월해집니다.

Jenkins 초기 설정은 대체로 다음 순서로 진행됩니다. 그 밖에도 더 편리한 설정이 있을 수 있지만, 어디까지나 초보자인 제 관점에서 정리한 것이니 참고용으로 봐 주세요.

## 포트 설정

Jenkins의 기본 포트는 `8080`입니다. 이 상태로 Jenkins를 시작하면 웹 브라우저에서 `Jenkins를 실행 중인 시스템의 IP 주소:8080`으로 접속할 수 있습니다.

하지만 웹 애플리케이션을 개발해 본 분이라면 `8080` 포트가 늘 좋은 선택은 아니라는 점을 아실 겁니다. 같은 포트를 사용하는 서비스가 둘 이상 있으면 충돌이 날 수 있기 때문입니다.[^1]

저는 나중에 Tomcat도 사용할 수 있다고 생각해서 Jenkins 포트를 `8088`로 바꿨습니다. `vi`나 `vim`[^2]으로 Jenkins 설정 파일을 엽니다.

```bash
sudo vim /etc/sysconfig/jenkins
```

그러면 아래처럼 설정 화면이 나옵니다. 조금 아래로 내려가면 `JENKINS_PORT`가 보일 겁니다. `I`를 눌러 입력 모드로 전환한 뒤 원하는 포트로 바꿔 주세요.

![Jenkins Config Port](jenkins_configport.webp)

수정이 끝나면 `ESC`를 누르고 `:wq`로 저장 후 종료하면 됩니다.

## 시작과 초기 비밀번호

Jenkins는 처음 시작할 때 초기 비밀번호를 요구합니다. 이 비밀번호는 기본적으로 `/var/lib/jenkins/secrets/initialAdminPassword`에 저장되어 있습니다. 다만 OS나 설치 방식에 따라 경로가 달라질 수 있습니다.[^3] 그래도 Jenkins를 한 번 실행하면 초기 설정 화면에서 경로를 확인할 수 있으니 너무 걱정할 필요는 없습니다.

포트 설정이 끝났다면, 8080을 그대로 써도 되지만 어쨌든 Jenkins를 시작합니다.

```bash
service jenkins start
```

`[OK]` 메시지가 나오겠지만, 포트 설정에 문제가 있을 수도 있으니 상태도 한 번 확인합니다.

```bash
service jenkins status
```

사실 예전에 멀쩡하던 Jenkins가 갑자기 접속되지 않았던 적이 있어서, 그런 경험 이후로는 더 신중하게 확인하게 되었습니다. `Active: active (running)`이 보이면 이제 Jenkins 페이지에 접속해도 됩니다.

![Jenkins Service Status](jenkins_servicestatus.webp)

브라우저에서 `Jenkins를 실행 중인 시스템의 IP 주소:포트번호`로 접속합니다. 물론 같은 시스템에서는 `localhost:8080` 같은 주소로도 접근할 수 있습니다.

![Jenkins Init Pass](jenkins_initpass.webp)

초기 비밀번호 경로는 역시 `/var/lib/jenkins/secrets/initialAdminPassword`였습니다. 포트 설정 때처럼 `vi`나 `vim`으로 열어서 비밀번호를 확인하고 입력합니다.

```bash
sudo vim /var/lib/jenkins/secrets/initialAdminPassword
```

## 플러그인과 관리자 계정

비밀번호를 입력하면 잠시 뒤 플러그인 설정 화면이 나옵니다. 직접 골라도 되지만 저는 자신이 없어서 추천 플러그인 버튼을 눌렀습니다. 당연하지만 플러그인은 나중에 추가 설치할 수도 있습니다.

![Jenkins Init Plugins](jenkins_initplugins.webp)

추천 버튼을 누르면 플러그인 설치가 자동으로 진행됩니다. 조금 기다리면 됩니다.

![Jenkins Installing Plugins](jenkins_installingplugins.webp)

설치가 끝나면 다음은 관리자 계정 설정입니다. 하나라도 비어 있으면 진행이 안 되니 모두 입력합니다.

![Jenkins Setup Admin](jenkins_setupadmin.webp)

관리자 계정 설정 다음에는 접속 주소 설정이 나옵니다. 지금은 따로 바꿀 필요가 없다고 생각해서 그대로 두었습니다. 가상 머신에 CentOS를 설치하고 Jenkins를 실행하는 환경이었기 때문입니다.

![Jenkins Address Setting](jenkins_addresssetting.webp)

그러면 Jenkins 준비가 끝났다는 화면이 나옵니다. 꽤 길었습니다. 이제 사용 시작 버튼을 누르면 됩니다.

![Jenkins Ready](jenkins_ready.webp)

드디어 Jenkins 메인 화면에 도착했습니다. 여기까지 오는 과정이 제법 길었지만, 한 번만 제대로 해 두면 이후 자동화 작업을 시작하기가 훨씬 쉬워집니다.

![Jenkins Mainpage](jenkins_mainpage.webp)

다음 글에서는 Jenkins로 실제 Job을 만들고 실행하는 과정을 정리해 보겠습니다.

[^1]: 특히 Spring Framework로 웹 애플리케이션을 만들 때 그렇습니다. Tomcat의 기본 포트도 `8080`이기 때문입니다.
[^2]: `vi`와 `vim` 중 무엇을 쓸지는 늘 고민됩니다. `vim`이 더 보기 좋고 읽기 편하지만, 시스템에 따라 설치되어 있지 않을 수 있습니다.
[^3]: macOS에 설치했을 때는 초기 비밀번호 경로가 사용자 홈 디렉터리 바로 아래인 경우도 있었습니다. root가 아닌 사용자로 설치했기 때문일 수도 있습니다.
