---
title: "Jenkins 자동화하기 1"
date: 2019-05-13
categories:
  - jenkins
image: "../../images/jenkins.webp"
tags:
  - ci/cd
  - jenkins
translationKey: "posts/jenkins-automation-1"
---

자동화는 이제 개발 현장에서도 너무 자연스러운 주제가 됐습니다. 반복 작업을 줄여 주고, 사람이 실수하기 쉬운 구간을 대신 처리해 주기 때문입니다.

그런 흐름 속에서 [Jenkins](https://jenkins.io/)도 다시 살펴보게 됐습니다. 거창하게 "모든 걸 자동화하겠다"는 생각이 있었다기보다는, 회사에서 실제로 쓰고 있었기 때문에 제대로 이해해 보고 싶었습니다.

## Jenkins란

우선은 Jenkins가 정확히 어떤 도구인지부터 정리해 보고 싶었습니다. 회사에서 실제로 쓰고 있다면 분명 이유가 있을 테고, 결국 이런 미들웨어는 "손이 덜 가게 해 주기 때문에" 살아남는 경우가 많기 때문입니다.

### 지속적 통합

처음에는 "지속적 통합이 정확히 뭐지?"라는 수준이었습니다. 쉽게 말하면 빌드, 테스트, 검증 같은 과정을 자동으로 이어 주는 도구라고 보면 됩니다. Git으로 이미 버전 관리를 하고 있는데 Jenkins가 왜 또 필요한지 의문이 들 수 있지만, 핵심은 그 이후 과정을 자동화할 수 있다는 점입니다.

Git이나 Subversion으로 버전 관리를 해도 누가 언제 `push`했는지 확인하려면 로그를 봐야 합니다. 누가 변경했는지 알더라도 사람이 직접 테스트하고 머지해야 합니다. 그런데 Jenkins를 쓰면 이런 과정을 상당 부분 자동화할 수 있습니다. 예를 들어 누군가 `push`하면 Jenkins에 알림이 가고, `pull`부터 빌드, JUnit 테스트까지 수행한 뒤 결과를 알려 줍니다. 설정에 따라서는 테스트가 성공하면 배포[^1]까지 해 줄 수 있으니 꽤 강력한 도구입니다.

## 써 봐야 안다

Jenkins의 자동화를 실제로 경험하려면 설치부터 해야 합니다. 회사 환경은 Linux였는데, 바로 `yum`으로 설치할 수 있을 줄 알았지만 생각처럼 간단하지는 않았습니다.

[Jenkins 홈페이지](https://jenkins.io/)를 찾아보니 Linux 설치 절차가 따로 있었습니다. 다행히 업무에서 쓰는 Linux가 AWS 환경이어서, RedHat Linux[^2]와 같은 절차로 설치할 수 있었습니다.

## 설치해 보자

먼저 Jenkins 저장소를 추가합니다.

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
```

여기서 `wget -O`는 파일을 지정한 경로에 저장하는 옵션인 것 같았습니다. 이런 것도 하나씩 배웁니다.

```bash
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```

`rpm`은 패키지를 설치할 때 쓰는 명령인데, 여기서는 `--import` 옵션으로 키를 가져와 검증합니다.

이제 일반 패키지처럼 `yum`으로 설치할 수 있습니다.

```bash
yum install jenkins
```

이렇게 설치는 끝납니다. 이후에는 포트 설정을 하고 서비스를 시작하면 됩니다.

(계속)

[^1]: 다른 서버 등에 배포하는 것을 말합니다. 예를 들어 개발 환경과 운영 환경은 분리해야 하므로, 개발 환경에서 검증이 끝난 것을 운영 환경에 배포해 실행하게 됩니다.
[^2]: Fedora와 CentOS에서도 같은 방식으로 할 수 있는 것 같습니다. 저는 CentOS 7에서 확인해 봤습니다.
