---
title: "Ansible로 서버 구축 자동화하기"
date: 2019-07-05
categories:
  - ansible
image: "../../images/ansible.webp"
tags:
  - ansible
  - yaml
  - linux
translationKey: "posts/ansible-server-automation"
---

이번에는 [Ansible](https://www.ansible.com/)을 잠깐 사용할 기회가 있었습니다. Ansible은 미리 정의한 작업을 여러 환경에 같은 방식으로 적용할 수 있다는 점에서 Jenkins와 비슷한 자동화 도구입니다. 다만 Jenkins가 배포나 테스트 자동화에 더 가깝다면, Ansible은 서버 초기 구축과 설정 자동화에 더 초점이 맞춰져 있습니다.

즉, 서버를 매번 손으로 맞추는 대신 같은 설정을 여러 환경에 반복 적용할 수 있습니다. 제가 다루던 환경은 `개발`에서 `내부 검증`, `외부 검증`으로 이어지는 구조였는데, 환경이 바뀔 때마다 서버를 다시 손으로 세팅하는 일은 비효율적이었습니다. 설정 항목이 늘수록 실수도 쉽게 생기기 때문에, 이 작업을 Ansible로 자동화해 보자는 것이 이번 이야기입니다.

Ansible은 `Playbook`이라는 YAML 파일 [^1]로 서버를 설정합니다. 구조를 조금만 익히면 호스트 정보와 설정 값을 조합해 여러 서버에 같은 작업을 쉽게 보낼 수 있습니다. 폴더 생성, 사용자 생성, `yum` 패키지 설치, 쉘 명령 실행, 파일 복사 같은 작업을 한 흐름으로 묶을 수 있어서, 반복 작업을 줄이는 데 꽤 유용합니다.

여기서는 Amazon Linux를 기준으로 파일 구성을 하나씩 살펴보겠습니다.

## Ansible 설치

Ansible은 `yum`, `brew`, `pip`, `apt-get`으로 설치할 수 있습니다.

```bash
yum install ansible
$ brew install ansible
$ pip install ansible
$ sudo apt-get install ansible
```

사용 중인 환경에 맞는 방법으로 설치하면 됩니다.

## batchserver.yml

먼저 서버별 YAML 파일을 만듭니다. Ansible 실행 시 이 파일이 사용되며, 어떤 호스트 그룹에 어떤 작업을 적용할지 적습니다. 서버별로 실행할 작업이 다르면 공통용, 배치 서버용, 웹 서버용 YAML을 따로 둡니다.

다음은 배치 서버를 가정한 예시입니다.

```yaml
- hosts: batchserver
  become: true
  roles:
    - common
    - batch
```

`hosts`는 `hosts` 파일에 정의한 호스트 그룹을 뜻합니다. 여기서 `batchserver`라고 적으면 `hosts` 파일에서 해당 그룹에 속한 서버에 같은 작업을 수행합니다. 모든 그룹에 적용하려면 `all`을 쓰면 됩니다.

`become: true`는 연결된 서버에서 명령을 sudo 권한으로 실행한다는 뜻입니다. `roles`에는 실제 작업 내용이 들어 있는 역할 폴더를 적습니다. 여기서는 공통 작업인 `common`과 배치 서버 전용 작업인 `batch`를 함께 지정했습니다.

## hosts

이 파일에는 Ansible이 접속할 대상 서버를 적습니다.

```text
[batchserver]
192.168.0.1 ansible_ssh_user=batchuser1
192.168.0.2 ansible_ssh_user=batchuser2

[webapserver]
192.168.10.1 ansible_ssh_user=webapuser1
192.168.10.2 ansible_ssh_user=webapuser2
```

기본적으로는 그룹 이름 아래에 호스트 정보를 적으면 자동으로 분류됩니다. 앞의 `batchserver.yml`에서는 `batchserver`만 지정했으므로 `webapserver` 그룹은 실행 대상에서 제외됩니다. `ansible_ssh_user`는 SSH 접속 시 사용할 계정입니다. 필요하면 실행 시점에 사용자 이름과 비밀번호를 직접 넣을 수도 있습니다.

## roles/batch/tasks/main.yml

여기에는 `batchserver.yml`에서 지정한 `batch` 역할의 작업이 들어갑니다. 역할 폴더 아래에 둔 파일만 실행되므로 경로를 맞춰 두는 것이 중요합니다.

```yaml
  - name: Create user groups
    group:
      name: "{{ item.group_name }}"
      gid: "{{ item.group_id }}"
    with_items:
      - { group_name: 'group01', group_id: '101' }
      - { group_name: 'group02', group_id: '201' }

  - name: Create users
    user:
      name: "{{ item.user_name }}"
      password: "{{ item.user_passwd }}"
      uid: "{{ item.user_id }}"
      group: "{{ item.user_group }}"
      shell: /bin/bash
    with_items:
      - { user_name: 'user01', user_group: 'group01', user_id: '101', user_passwd: 'user01' }
      - { user_name: 'user02', user_group: 'group02', user_id: '201', user_passwd: 'user02' }

  - name: Create folders
    file:
      path: "{{ item }}"
      owner: user01
      group: user01
      mode: 0755
      state: directory
    with_items:
      - /user01

  - name: Package install by yum
    yum:
      name: "{{ packages }}"
    vars:
      packages:
        - python2-pip
        - postgresql
        - postgresql-devel

  - name: Upgrade pip by shell command
    shell: bash -lc "pip install --upgrade pip"

  - name: Install python modules
    pip:
      name: "{{ item }}"
      executable: pip
    with_items:
      - cx_Oracle
      - psycopg2
      - boto3
      - paramiko

  - name: Copy files
    copy:
      src: "{{ item.source }}"
      dest: "{{ item.dest }}"
      owner: root
      group: root
      mode: 0755
    with_items:
      - { source: etc/somefile.zip, dest: /etc/somefile.zip }
```

위 순서대로 사용자 그룹 생성, 사용자 생성, 폴더 생성, 패키지 설치, 쉘 명령 실행, 파일 복사가 진행됩니다. 결국 SSH로 접속해 필요한 작업을 순서대로 실행하는 셈이지만, YAML로 정리해 두면 반복 작업을 훨씬 편하게 다룰 수 있습니다.

다만 Ansible은 위에서 아래로 순서대로 실행하므로 작업 순서에는 주의해야 합니다. 예를 들어, 사용자 그룹을 만들기 전에 그 그룹에 사용자를 넣을 수는 없습니다.

## roles/batch/files

이 폴더에는 파일 전송에 사용할 파일을 둡니다. 예를 들어 위의 `main.yml`에서 `etc/somefile.zip`을 복사하려면 같은 경로 구조로 파일을 넣어 두면 됩니다. 여러 파일을 전송해야 한다면 이 폴더 아래에 적절히 나눠 두면 됩니다.

## roles/common/tasks/main.yml

이 파일에는 모든 서버에서 공통으로 실행할 작업을 넣습니다.

```yaml
  - name: Upgrade all packages by yum
    yum: name=* state=latest

  - name: Install openjdk 11
    shell: bash -lc "amazon-linux-extras install java-openjdk11"

  - name: Correct java version selected
    alternatives:
      name: java
      path: /usr/lib/jvm/java-11-openjdk-11.0.2.7-0.amzn2.x86_64/bin/java
```

여기서는 `yum`으로 전체 패키지를 업데이트하고, OpenJDK 11을 설치한 뒤, `alternatives`로 기본 `java` 버전을 맞춥니다. Java만 설치한다고 기본 실행 버전이 자동으로 바뀌지는 않기 때문에, 이런 식으로 명시적으로 지정해 주는 편이 안전합니다.

여기까지 하면 Ansible의 기본 설정은 끝입니다. 생각보다 어렵지 않습니다.

## 실행하기

Playbook이 준비되면 다음처럼 실행할 수 있습니다.

```bash
# 일반 실행
$ ansible-playbook server.yml -i hosts
# Dry run
$ ansible-playbook server.yml -i hosts -C
# SSH 사용자명을 직접 지정할 때
$ ansible-playbook server.yml -i hosts -u hostuser
```

실행 중 SSH 비밀번호를 물어보게 할 수도 있지만, SSH 공개 키를 미리 등록해 두면 그런 과정을 줄일 수 있습니다. `sudo`도 `visudo`에서 `NOPASSWD`를 설정해 두면 편합니다.

## 마지막으로

지금은 무엇이든 자동화하는 쪽으로 많이 바뀌고 있습니다. Jenkins와 Ansible을 함께 쓰면 서버 구축부터 배포까지 꽤 많은 부분을 자동화할 수 있어서 생산성이 크게 올라갑니다.

아직 수동으로 서버를 세팅하고 있다면, 반복 작업을 YAML로 정리해 두는 것만으로도 운영 부담이 많이 줄어듭니다. 그런 점에서 Ansible은 한 번쯤 익혀 둘 만한 도구라고 생각합니다.

[^1]: 구조는 JSON과 비슷하지만 사람이 읽기에는 마크업 언어에 더 가까운 느낌입니다. 버릇은 있지만 직관적으로 표현하기 좋습니다.
