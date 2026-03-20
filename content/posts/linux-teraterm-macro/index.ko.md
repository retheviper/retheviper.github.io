---
title: "Tera Term 매크로로 SSH 접속 자동화하기"
date: 2019-06-24
categories:
  - linux
image: "../../images/linux_terminal.webp"
tags:
  - terminal
  - linux
translationKey: "posts/linux-teraterm-macro"
---

PuTTY나 macOS 터미널은 써 봤지만, [Tera Term](https://ttssh2.osdn.jp/)은 그때까지 사용해 본 적이 없었습니다. 그런데 회사에서는 AWS에 Linux 서버를 세워 놓고 접속해야 했기 때문에 SSH 연결 도구가 필요했습니다. 그래서 처음 해 본 일이 Tera Term 매크로를 만들어 SSH 접속을 자동화하는 것이었습니다.

문서 작업만 하다가 처음으로 이런 식의 스크립트를 만들어 보니 꽤 재미있었습니다. 게다가 매크로로 간단하게 쉘 명령을 보낼 수 있고, 화면 설정도 쉽게 바꿀 수 있어서 생각보다 편리했습니다. 이제 실제로 어떻게 만들었는지 정리해 보겠습니다.

## 매크로 디자인

이번 프로젝트에서는 먼저 발판 서버를 거친 뒤 EC2 서버에 접속하는 구조를 사용했습니다. 처음에는 왜 이렇게 복잡하게 접속해야 하는지 이해가 잘 안 됐지만, 외부에서 직접 접근할 수 없는 서버가 섞여 있다 보니 이런 단계가 필요한 구조였습니다.

그래서 매크로는 다음 순서로 동작하게 만들었습니다.

1. 목록에서 접속할 서버를 고른다.
2. 먼저 발판 서버에 접속한 뒤, 거기서 다시 대상 서버로 SSH 접속한다.
3. 대상 서버에 맞는 INI 파일을 불러온다.

배치 서버, WebAP 서버, CI 서버별로 터미널 설정이 조금씩 달라서 INI 파일도 각각 따로 준비했습니다. 특히 CI 서버는 Jenkins 접속용 포트 포워딩까지 필요했기 때문에, 서버마다 다른 INI 파일을 읽게 했습니다.

## 매크로 코드

```text
;==============================================
;; 발판 사용자 ID / 비밀번호
;==============================================
USERNAME = '발판 사용자 ID'
PASSWORD = '발판 사용자 비밀번호'

;==============================================
;; 발판 서버 IP 주소
;==============================================
HOSTIP = '여기에 IP'

;==============================================
;; 접속 대상별 작업용 사용자 ID
;==============================================
strdim WORKUSERLIST 3
WORKUSERLIST[0] = '배치 서버 사용자 ID'
WORKUSERLIST[1] = 'WebAP 서버 사용자 ID'
WORKUSERLIST[2] = 'CI 서버 사용자 ID'

;==============================================
;; 접속 대상별 작업용 사용자 비밀번호
;==============================================
strdim WORKPWLIST 3
WORKPWLIST[0] = '배치 서버 비밀번호'
WORKPWLIST[1] = 'WebAP 서버 비밀번호'
WORKPWLIST[2] = 'CI 서버 비밀번호'

;==============================================
;; 서버 IP
;==============================================
strdim SERVERipLIST 3
SERVERipLIST[0] = '배치 서버 IP'
SERVERipLIST[1] = 'WebAP 서버 IP'
SERVERipLIST[2] = 'CI 서버 IP'

;==============================================
;; 목록에 표시될 서버 이름
;==============================================
strdim SERVERnameLIST 3
SERVERnameLIST[0] = '배치 서버'
SERVERnameLIST[1] = 'WebAP 서버'
SERVERnameLIST[2] = 'CI 서버'

;==============================================
;; 서버별 INI 파일
;==============================================
strdim INILIST 3
INILIST[0] = '/BatchServer.INI'
INILIST[1] = '/WebAPServer.INI'
INILIST[2] = '/CIServer.INI'

;==============================================
;; 접속 대상 선택 화면
;==============================================
listbox '서버를 선택해 주세요' '결정' SERVERnameLIST
if result >= 0 then
    SERVERIP = SERVERipLIST[result]
    WORKUSER = WORKUSERLIST[result]
    WORKPASSWORD = WORKPWLIST[result]
    INIFILE = INILIST[result]
else
    end
endif

;==============================================
;; INI 파일 경로 읽기
;==============================================
getdir INIPATH
strconcat INIPATH INIFILE

;==============================================
;; 발판 서버 접속 명령 조립 및 실행
;==============================================
PROXY = '-proxy=http://proxy.server.com:6000'
COMMAND = PROXY
strconcat COMMAND ' '
strconcat COMMAND HOSTIP
strconcat COMMAND ':22 /ssh /auth=password /user='
strconcat COMMAND USERNAME
strconcat COMMAND ' /passwd='
strconcat COMMAND PASSWORD
connect COMMAND

wait '$'

;==============================================
;; 대상 서버 SSH 접속
;==============================================
SSHCOMMAND = 'ssh '
strconcat SSHCOMMAND WORKUSER
strconcat SSHCOMMAND '@'
strconcat SSHCOMMAND SERVERIP
sendln SSHCOMMAND

;==============================================
;; 첫 SSH 로그인 처리
;==============================================
wait 'Are you sure you want to continue connecting (yes/no)?' "'s password: "
if result = 1 then
    sendln 'yes'
    wait "'s password: "
elseif result = 2 then
    goto INPUTPWD
endif

:INPUTPWD
sendln WORKPASSWORD
wait '$'

sendln 'sudo su -'
wait 'sudo'
sendln WORKPASSWORD
wait '#'

restoresetup INIPATH

;==============================================
;; 매크로 종료
;==============================================
end
```

## 코드 설명

처음에는 발판 서버의 사용자 ID, 비밀번호, IP를 조합해 접속 명령을 만듭니다. 프록시가 필요한 환경이라면 접속 문자열에 프록시 설정을 붙이면 됩니다. 그 다음은 목록에서 서버를 하나 고르게 하고, 선택한 값에 따라 사용자명, 비밀번호, IP, INI 파일 경로를 꺼냅니다.

`listbox`에서 선택한 값은 `result`에 숫자로 들어오므로, 그 값을 배열 인덱스로 사용합니다. 서버별 INI 파일도 같은 인덱스로 찾아서 `restoresetup`에 넘깁니다. 매크로 파일과 INI 파일은 같은 폴더에 있다고 가정하면 됩니다.

`wait`는 터미널의 반응을 기다리는 역할입니다. 쉘의 `expect`처럼, 다음 명령을 보내기 전에 서버가 어떤 상태인지 확인하는 데 사용합니다. 그래서 첫 접속 시 호스트 키 확인 메시지가 나오더라도 자연스럽게 대응할 수 있습니다.

매크로를 `.ttl` 확장자로 저장한 뒤 Tera Term에서 불러오거나, 설치 시 매크로 실행 옵션을 사용해 바로 실행할 수 있습니다.

## 마지막으로

`wait`와 `sendln`만 잘 조합해도 꽤 많은 작업을 자동화할 수 있습니다. 이 매크로도 결국 쉘 작업을 좀 더 편하게 쓰기 위한 도구였습니다. 여기서는 발판 서버를 거쳐 각 서버에 접속한 뒤 root 권한까지 올리는 흐름을 만들었지만, 필요한 경우 작업 경로나 추가 명령도 얼마든지 넣을 수 있습니다.

Windows 환경에서 SSH 접속을 자주 반복해야 한다면, Tera Term 매크로도 생각보다 꽤 실용적인 선택입니다.
