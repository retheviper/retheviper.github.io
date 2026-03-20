---
title: "Jenkins 자동화하기 3"
date: 2019-05-16
categories:
  - jenkins
image: "../../images/jenkins.webp"
tags:
  - ci/cd
  - jenkins
translationKey: "posts/jenkins-automation-3"
---

직장에서 Jenkins로 자동화하고 싶은 작업이 몇 가지 있었습니다. 다만 실제로 Job을 돌리려면 그전에 준비해야 할 것이 생각보다 많았습니다. 무엇을 자동화할지 정하는 것도 중요하지만, 먼저 필요한 실행 환경을 제대로 갖추는 일이 더 급했습니다. 이번 글에서는 그 준비 과정을 정리해 보겠습니다.

먼저 Git에서 Spring Boot 애플리케이션을 받아 빌드하는 작업이 필요했습니다. 저는 Spring Framework는 다뤄 본 적이 있었지만 Spring Boot는 처음이었습니다. Maven은 써 봤지만 이번처럼 Gradle 기반 프로젝트를 Jenkins에서 다루는 경험은 없었습니다. Spring Boot와 Gradle이 초기 설정을 쉽게 만들어 준다고는 해도, Eclipse에서만 앱을 실행해 본 수준이라 처음에는 어디서부터 시작해야 할지 감이 잘 오지 않았습니다.

그래서 Jenkins에서 Job을 어떻게 구성하는지부터 차근차근 확인하기로 했습니다.

Jenkins에서는 Job이라는 단위로 작업을 만들고, 그 안에 여러 동작을 넣어 실행할 수 있습니다. 반복해서 해야 하는 작업을 자동화하기에 적합하고, 수동 실행이나 트리거 실행도 지원합니다.

제가 해야 할 일은 결국 다음 흐름이었습니다. Git에 있는 Java 애플리케이션을 가져와서, Jenkins가 실행 중인 서버에서 빌드하고, 실행 가능한 패키지로 만든 뒤 배포 준비까지 이어 가는 것입니다. 그럼 우선 필요한 도구부터 준비해야 합니다.

## OpenJDK를 설치합시다

Jenkins 자체는 Java 8에서도 실행할 수 있지만, 이번 애플리케이션은 Java 11을 사용하고 있었습니다. 그래서 OpenJDK 11을 설치해야 했습니다. `yum install java`를 하면 Oracle Java가 들어갈 수 있으므로, OpenJDK를 직접 설치하는 절차가 필요했습니다. 이번에는 `wget` 대신 `curl`을 사용해 보겠습니다.

```bash
curl -O https://download.java.net/java/GA/jdk11/13/GPL/openjdk-11.0.1_linux-x64_bin.tar.gz
```

다운로드한 뒤 압축을 풉니다.

```bash
tar zxvf openjdk-11.0.1_linux-x64_bin.tar.gz
mv jdk-11.0.1 /usr/local/
```

그 다음에는 환경 변수를 설정하기 위해 간단한 스크립트를 만듭니다.

```bash
vi /etc/profile.d/jdk11.sh
```

안에 다음 내용을 적습니다.

```bash
export JAVA_HOME=/usr/local/jdk-11.0.1
export PATH=$PATH:$JAVA_HOME/bin
```

저장한 뒤 바로 적용합니다.

```bash
source /etc/profile.d/jdk11.sh
```

이제 OpenJDK 11이 준비되었습니다. CentOS 7을 쓰고 있었는데, 설치 직후 `java -version`을 확인하니 이미 다른 버전이 올라와 있었습니다.

![Java version is 8](jenkins_java8.webp)

## 사용할 Java 버전 전환

이미 다른 버전의 JDK가 설치되어 있으면, 이후 사용할 Java 버전을 선택해 줄 수 있습니다. 먼저 새로 설치한 Java를 alternatives에 등록합니다.

```bash
alternatives --install /usr/bin/java java $JAVA_HOME/bin/java 2
```

그다음 등록된 Java 목록을 확인합니다.

```bash
alternatives --config java
```

CentOS에는 이미 두 개의 Java 버전이 들어 있어서, Java 11이 세 번째 항목이었습니다. 해당 번호를 선택하면 됩니다.

![Jenkins Alter](jenkins_alter.webp)

Jenkins도 Java로 만들어졌기 때문에, Jenkins 자체도 Java 11로 띄울 수 있습니다. 설정 파일을 열어 Java 11 경로를 추가합니다.

```bash
vi /etc/init.d/jenkins
```

설정 파일의 `candidates` 부분에 OpenJDK 11의 `bin` 경로를 넣고 저장합니다.

![Jenkins JVM](jenkins_jvm.webp)

## Gradle도 설치합시다

다음은 Gradle입니다. macOS에서는 `brew install gradle`로 쉽게 설치할 수 있었지만, Linux에서는 직접 내려받아 설치해야 했습니다.

```bash
wget https://services.gradle.org/distributions/gradle-5.4.1-bin.zip -P /tmp
sudo unzip -d /opt/gradle /tmp/gradle-5.4.1-bin.zip
```

환경 변수를 위한 스크립트도 하나 만듭니다.

```bash
sudo nano /etc/profile.d/gradle.sh
```

내용은 다음과 같습니다.

```bash
export GRADLE_HOME=/opt/gradle/gradle-5.4.1
export PATH=${GRADLE_HOME}/bin:${PATH}
```

저장한 뒤 실행 권한을 주고 적용합니다.

```bash
sudo chmod +x /etc/profile.d/gradle.sh
source /etc/profile.d/gradle.sh
```

설치가 잘 됐는지는 버전을 확인하면 됩니다.

```bash
gradle -v
```

![Jenkins Gradle Version](gradle_version.webp)

## Jenkins에 JDK와 Gradle 연결

이제 Jenkins 쪽 설정입니다. 웹 브라우저에서 Jenkins 메인 화면으로 들어가 `Manage Jenkins`를 클릭합니다.

![Jenkins Manage](jenkins_manage.webp)

그다음 `Global Tool Configuration`으로 들어갑니다.

![Jenkins Global Tool Settings](jenkins_globaltoolsettings.webp)

JDK 항목의 `Add JDK`를 누르고, 자동 설치는 끕니다. 대신 앞에서 설치한 Java 11 경로를 직접 넣습니다.

![Jenkins JDK](jenkins_jdk.webp)

Gradle도 같은 방식입니다. 자동 설치 대신 수동으로 경로를 지정합니다.

![Jenkins Gradle](jenkins_gradle.webp)

저장하면 Jenkins에서 JDK와 Gradle을 사용할 준비가 끝납니다.

다음 글에서는 실제 Spring Boot 애플리케이션을 Git에서 받아 빌드하는 Job을 만들어 보겠습니다.

[^1]: 엄밀히 말하면 Spring Boot는 Spring Framework 위에서 동작합니다. 언젠가 Spring Boot도 따로 정리해 보고 싶습니다.
[^2]: 당시에는 Jenkins에서 Java 11 지원이 막 시작된 시점이라 우선 Java 11을 선택했습니다.
