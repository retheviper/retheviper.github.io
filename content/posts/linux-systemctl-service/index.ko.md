---
title: "Linux 시스템 서비스 만들기"
date: 2019-07-11
categories:
  - linux
image: "../../images/linux_terminal.webp"
tags:
  - linux
translationKey: "posts/linux-systemctl-service"
---

서버에서 실행되는 프로그램은 일반적인 사용자용 프로그램과 동작 방식이 다릅니다. 어떤 데이터를 다루고 어떤 처리를 하는지도 중요하지만, 이번에는 간단히 "실행" 관점에서 이야기해 보겠습니다. 쉽게 말하면, 누가 멈출 때까지 계속 실행되는 프로그램을 Linux에서 어떻게 서비스로 만들 수 있는지에 대한 내용입니다.

CentOS와 RHEL에는 `service`라는 명령이 있습니다. `yum` 같은 패키지 관리 도구로 여러 프로그램을 설치하다 보면, 한 번 실행하고 끝나는 것이 아니라 메모리에 계속 상주해야 하는 프로그램도 있습니다. 예전에 이 블로그에서 소개했던 `Jenkins`도 그런 프로그램 가운데 하나입니다.

그런 프로그램은 한 번 설치해 두면 `service` 명령으로 시작하거나 멈출 수 있습니다. Linux라면 뭐든 할 수 있을 것 같지만, 실제로 서비스로 등록하고 관리하는 방법을 한 번은 정리해 둘 필요가 있습니다.

저는 업무에서 Java 애플리케이션을 서비스로 등록하고 관리해야 했습니다. Jenkins 작업을 관리하면서, Git에 커밋이 들어오면 Pull -> 빌드 -> 실행 중인 Java 애플리케이션 중지 -> 새로 빌드한 애플리케이션 실행까지 이어지는 작업이 필요했습니다. 실제로는 서비스 자체를 직접 다루기보다는, 빌드 후에 서비스를 멈추거나 다시 시작하는 셸 커맨드를 실행하는 형태로 구성했지만, 이것이 어떻게 동작하는지 궁금해서 조사해 봤습니다.

## Service(Daemon) 만들기

서비스가 되는 프로그램은 `Daemon`이라고도 부릅니다. 조사해 보면 "시스템에 상주하면서 특정 상태가 되면 자동으로 동작하는 프로그램", "주기적인 서비스 요청을 처리하기 위해 계속 실행되는 프로그램", "백그라운드에서 동작하는 프로그램" 같은 정의가 있습니다. 이 정도면 어떤 성격의 프로그램을 말하는지 알 수 있습니다. `Spring`이나 `Node.js`로 만든 웹 서버 프로그램도 이런 정의에 들어갑니다.

그렇다면 이런 서비스, 혹은 데몬을 어떻게 만들면 될까요? 준비할 것은 크게 서비스로 등록할 프로그램, 그 프로그램을 어떤 서비스로 취급할지 적는 설정 파일, 그리고 명령으로 서비스를 등록하고 실행하는 절차입니다. 업무에서는 `Spring Boot` 프로그램을 서비스로 쓰고 있었지만, 이 경우에는 실행용 셸 스크립트 설정이나 외부 파일 참조도 필요해서, 비교적 단순한 `Node.js` 프로그램을 예시로 쓰겠습니다.

`Node.js`는 `$ node /node/index.js` 같은 간단한 명령으로 실행할 수 있습니다. 이를 서비스로 등록하면 다음과 같습니다.

## service 파일 만들기

먼저 서비스가 어떤 방식으로 동작해야 하는지, 어떤 이름으로 등록할지 등을 적는 파일을 만듭니다.

```bash
vi /etc/systemd/system/NodeServer.service
```

아래처럼 내용을 작성합니다.

```bash
# 서비스 설정
[Unit]
Description = NodeServer # 이 이름으로 서비스가 등록된다
After = syslog.target network.target # 시스템 시작 시 실행 우선순위(syslog와 network 뒤에 실행)

# 실행할 프로그램 설정
[Service]
Type = simple # 동작 방식을 정한다. 기본값은 simple
ExecStart = /usr/bin/node /node/index.js # node /node/index.js와 같지만 심볼릭 링크 없이 명시
Restart = on-failure # 시작에 실패하면 다시 실행
User = nodeservice # 실행할 사용자. 권한에 주의

# 심볼릭 링크나 별칭 관련 설정
[Install]
WantedBy = multi-user.target # 어느 위치에 심볼릭 링크를 만들지 지정. 보통 이 값을 쓴다
```

다른 옵션도 많지만, 기본적인 내용은 이 정도면 충분합니다. 테스트용으로 만든 Node.js 웹 서버는 이렇게 해서 잘 동작했습니다. 물론 `Hello Node.js!`만 출력하는 단순한 서버지만요.

실제로 업무에서 다루던 Java 애플리케이션에는 PID[^1]를 관리하는 스크립트 지정이나, 종료 시 실행할 셸 스크립트 지정도 있었습니다. 예를 들어 `ExecStop`은 서비스를 멈출 때 실행할 명령을, `ExecReload`는 다시 불러올 때 실행할 명령을 적는 식으로 쓸 수 있습니다. 셸 스크립트에서 실행하거나 상태 변경 시 추가 조치가 필요한 경우에 유용한 옵션입니다.

이렇게 `service` 파일을 만들었다면, 다음은 프로세스를 시스템 서비스로 등록하고 실행하면 됩니다.

## Enable & Start Service

앞서 `service` 명령을 언급했지만, CentOS 7부터는 `systemctl`을 쓰는 편이 일반적입니다.[^2] 실제로 CentOS 7에서는 `service`만으로 시스템 서비스 등록이 되지 않으므로 `systemctl`을 사용하게 됩니다. 이 명령으로는 서비스 등록과 해제, 시작과 중지, 재시작 등을 할 수 있습니다. 먼저 서비스를 시스템 서비스로 등록하면 시스템 시작 시 자동으로 실행됩니다.

Windows에 비유하면, 프로그램을 시작 프로그램에 등록하거나 해제하는 것과 태스크 매니저로 프로세스를 다루는 기능이 같이 있는 느낌입니다.

명령 자체는 간단합니다.

```bash
# 시스템 서비스로 등록
$ systemctl enable NodeServer
$ systemctl start NodeServer

# 실행 중인 서비스 목록에서 찾기
$ systemctl list-units | grep NodeServer
```

```bash
# service 파일을 수정했으면 다시 읽어 들인다
$ systemctl daemon-reload

# 중지와 재시작
$ systemctl stop NodeServer
$ systemctl restart NodeServer

# 서비스 상태 확인
$ systemctl status NodeServer

# 시스템 서비스에서 해제
$ systemctl disable NodeServer

# 실행에 문제가 있으면 서비스 로그 확인
$ journalctl -xe
```

`service` 명령으로도 제어는 가능하지만, 앞으로는 없어질 수도 있는 명령이니 가능하면 `systemctl`에 익숙해지는 편이 좋습니다.

```bash
# service로 할 수 있는 것
$ service NodeServer start
$ service NodeServer status
$ service NodeServer stop
$ service NodeServer restart
```

이제 간단한 Node.js 웹 서버도 시스템 서비스로 등록해 계속 실행되도록 만들 수 있습니다. 한 번 익혀 두면 웹 서버뿐 아니라 각종 배치 프로그램이나 자동화 도구를 등록할 때도 그대로 활용할 수 있습니다.

[^1]: `Process ID`의 약자로, 실행 중인 프로세스에 부여되는 ID입니다. `ps` 명령으로 확인할 수 있고, PID로 프로세스 이름을 찾거나 반대로 프로세스 이름으로 PID를 찾을 수도 있습니다.
[^2]: CentOS 7에서도 `service`는 남아 있지만 `systemctl`로 리다이렉트되는 것 같습니다. CentOS 6까지는 `/etc/rc.d/init.d`에서 서비스를 관리했고, 7부터는 서비스 유닛이라는 이름을 쓰게 된 것으로 보입니다.
