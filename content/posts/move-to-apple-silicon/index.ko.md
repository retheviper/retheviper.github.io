---
title: "Apple Silicon Mac으로 옮기기"
date: 2021-12-19
categories: 
  - recent
image: "../../images/magic.webp"
tags:
  - mac
translationKey: "posts/move-to-apple-silicon"
---

M1 Mac이 나온 지도 벌써 1년이 넘었습니다. 처음에는 [Rosetta](https://ko.wikipedia.org/wiki/Rosetta)가 있다고 해도 네이티브 앱이 적었고, 경우에 따라서는 문제나 성능 저하가 생기기도 했기 때문에 바로 Apple Silicon Mac을 사야겠다는 생각은 들지 않았습니다. 하지만 지금은 네이티브 앱도 많이 늘었고 macOS 업데이트도 꾸준히 이뤄졌기 때문에, 이제는 충분히 넘어가도 괜찮은 시점이 됐다고 느꼈습니다. 게다가 올해 M1 Pro와 M1 Max를 탑재한 새 Mac이 나오면서 기존 Intel 모델보다 장점이 많아 보였고, 그래서 구매를 결정하게 됐습니다.

새 Mac 자체에 대해서는 칩 성능도 좋고, 디스플레이나 스피커, 키보드 같은 하드웨어도 꽤 좋아졌다는 인상을 받았습니다. 다만 그런 내용은 이미 벤치마크나 리뷰에서 많이 다뤄지고 있으니, 여기서는 굳이 반복하지 않겠습니다. 대신 Intel Mac에서 Apple Silicon Mac으로 옮겨 가는 과정에서 겪은 점들을 정리해 보려고 합니다.

## 마이그레이션 방법

Apple Silicon Mac으로 옮기기로 한 뒤, 저는 다음 두 가지 기준을 세웠습니다.

- 기존 Mac의 환경을 최대한 그대로 가져간다
- 가능하면 네이티브 앱을 쓴다

기존 Mac에서 마이그레이션하기로 한 이유는, 이미 쓰고 있는 앱 설정을 처음부터 다시 맞추고 싶지 않았기 때문입니다. 아키텍처가 달라졌으니 새로 세팅하는 편이 더 안전하지 않을까 고민도 했지만, 실제로 마이그레이션해서 써 본 결과 지금까지는 특별한 문제 없이 잘 사용하고 있습니다. 처음부터 전부 다시 구성하는 게 번거롭다면, 마이그레이션을 그대로 진행하는 것도 충분히 괜찮아 보입니다.

마이그레이션에는 macOS의 "마이그레이션 지원"을 사용했습니다. 선택 가능한 방법은 아래와 같습니다.

- Time Machine에서 복원
- Mac에서 직접 마이그레이션
  - Wi-Fi 사용
  - Thunderbolt 케이블 사용

Time Machine은 최신 상태가 아닐 수 있어서, 실제로 쓰던 Mac에서 직접 가져오는 방식을 택했습니다. 그리고 데이터 복사 시간을 줄이기 위해 Thunderbolt로 두 Mac을 직접 연결했습니다. 생각보다 오래 걸리지 않았고, 완료 직후 상태도 기존 Mac과 거의 차이가 없었습니다.

이후에는 Apple Silicon 네이티브 앱을 최대한 쓰고 싶었기 때문에, 설치된 앱을 하나씩 확인하면서 Universal이 아니거나 별도 arm 버전이 있는 앱부터 순서대로 정리해 나갔습니다.

## 바이너리 확인

마이그레이션이 끝난 뒤, 설치된 앱이 Apple Silicon 네이티브인지 확인하는 방법은 몇 가지가 있습니다.

- Finder에서 "정보 가져오기"
- Activity Monitor의 `종류` 컬럼 확인
- `이 Mac에 관하여` -> `시스템 리포트` -> `소프트웨어` -> `응용 프로그램`
- [Is Apple Silicon ready?](https://isapplesiliconready.com)에서 확인

또 Intel 전용 앱은 처음 실행할 때 Rosetta 설치 여부를 묻는 대화상자가 뜨기도 합니다. 특히 터미널에서 사용하는 프로그래밍 언어 런타임이 이런 경우가 많습니다. 다만 Rosetta를 한 번 설치해 버리면 이후에는 Intel 바이너리도 별도 경고 없이 실행되기 때문에, 전체 점검이 끝날 때까지는 Rosetta 설치를 미루는 편이 낫다고 생각합니다.

## 애플리케이션 전환

아래 앱들은 Universal Binary를 제공하므로, Intel Mac에서 옮겨 왔다고 해도 별도 작업 없이 그대로 사용할 수 있었습니다.

- [Chrome](https://www.google.com/intl/en/chrome)
- [Edge](https://www.microsoft.com/ko-kr/edge)
- [Firefox](https://www.mozilla.org/en/firefox/new/)
- [EdgeView2](https://apps.apple.com/ko/app/edgeview-2/id1206246482)
- [Movist Pro](https://movistprime.com/en)
- [Amphetamine](https://apps.apple.com/ko/app/amphetamine/id937984704)
- [Bandizip](https://apps.apple.com/ko/app/id1265704574)
- [Obsidian](https://obsidian.md)
- [Magnet](https://apps.apple.com/ko/app/magnet-%E3%83%9E%E3%82%B0%E3%83%8D%E3%83%83%E3%83%88/id441258766)
- [Macs Fan Control](https://crystalidea.com/macs-fan-control/download)
- [iStat Menus](https://apps.apple.com/ko/app/istat-menus/id1319778037)
- [유니콘](https://apps.apple.com/ko/app/%E3%83%A6%E3%83%8B%E3%82%B3%E3%83%BC%E3%83%B3-%E5%BA%83%E5%91%8A%E3%83%96%E3%83%AD%E3%83%83%E3%82%AF%E5%BF%85%E9%A0%88%E3%82%A2%E3%83%97%E3%83%AA/id1231935892)
- [Microsoft Remote Desktop](https://apps.apple.com/ko/app/microsoft-remote-desktop/id1295203466)
- [Microsoft Word](https://apps.apple.com/ko/app/microsoft-word/id462054704)
- [Microsoft PowerPoint](https://apps.apple.com/ko/app/microsoft-powerpoint/id462062816)
- [Microsoft Excel](https://apps.apple.com/ko/app/microsoft-excel/id462058435)
- [Microsoft OneNote](https://apps.apple.com/ko/app/microsoft-onenote/id784801555)
- [Microsoft Outlook](https://apps.apple.com/ko/app/microsoft-outlook/id985367838)

참고로 `Macs Fan Control`은 메뉴 막대에 CPU 온도를 표시하려고 쓰고 있는데, Intel Mac에서 흔히 보이던 `CPU PECI` 항목은 Apple Silicon에서는 없습니다. 한동안 비교해 보니 `CPU Performance Core`가 가장 참고하기 좋아 보여서, 온도를 표시하려면 그 항목을 선택하는 편이 괜찮아 보였습니다.

### 바이너리를 직접 바꿔야 하는 경우

아래 앱들은 Apple Silicon용 바이너리를 따로 제공하고 있어서, 홈페이지에서 새 버전을 내려받아 기존 앱을 덮어쓰는 방식으로 정리할 수 있었습니다.

- [Notion Desktop](https://www.notion.so/desktop)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Postman](https://www.postman.com/downloads/)
- [Zoom](https://zoom.us/download)
- [Webex](https://www.webex.com/ko/downloads.html)
- [draw.io](https://github.com/jgraph/drawio-desktop/releases)

다만 Docker는 데스크톱 앱 자체가 네이티브라고 해도, 실제로 돌리는 이미지가 amd64만 지원하는 경우가 많아서 별도 검증이 필요합니다. 제 환경에서는 MySQL 5.7이 아직 Apple Silicon을 정식 지원하지 않았지만, 일단은 큰 문제 없이 돌아가고 있습니다.

## Homebrew

Rosetta를 사용해서 기존 Intel용 Homebrew 패키지를 그대로 쓸 수도 있긴 합니다. 하지만 Apple Silicon용 Homebrew는 설치 경로가 다르고(Intel은 `/usr/local`, Apple Silicon은 `/opt/homebrew`), 설치되는 패키지도 기본적으로 Apple Silicon 네이티브이거나 arm 환경에 맞게 다시 빌드되는 경우가 많습니다. 그래서 저는 기존 Intel용 Homebrew를 지우고 새로 설치하기로 했습니다.

실제 제거와 설치는 크게 복잡하지 않았습니다. 공식 사이트 안내대로 아래 명령만 실행하면 됩니다.

```bash
# 언인스톨
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"

# 인스톨
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

제 경우에는 Homebrew로 설치한 패키지를 일단 전부 비워 두고, 나중에 필요한 것만 다시 설치하는 쪽을 택했습니다. 다만 기존 패키지 목록은 아래 명령으로 백업할 수 있습니다.

```bash
/usr/local/homebrew/bin/brew bundle dump
```

## 개발 환경

개발 환경 쪽은 상황이 다양해서 항목별로 나눠 정리하겠습니다.

### JetBrains IDE

JetBrains 제품과 Android Studio는 [Toolbox](https://www.jetbrains.com/ko-kr/toolbox-app)로 관리하면 편합니다. 다만 Toolbox 자체도 Apple Silicon 대응 바이너리로 바꿔야 합니다.

Toolbox를 네이티브 버전으로 바꾼 뒤에는, 메뉴에서 IDE를 제거하고 다시 설치하면 됩니다. Toolbox를 사용하지 않는 경우라면, 쓰고 있는 IDE 바이너리를 직접 다시 받아야 합니다.

참고로 Toolbox로 설치한 IDE는 `~/Applications/JetBrains Toolbox` 아래에 launcher가 놓이는데, 시스템 리포트에서 보면 Intel 바이너리처럼 보일 수 있습니다. 하지만 실제 런타임은 제대로 Apple Silicon 네이티브로 동작하고 있었고, 이 부분은 Activity Monitor에서도 확인할 수 있었습니다.

### Visual Studio Code

[Visual Studio Code](https://code.visualstudio.com/download)는 Universal, Intel, Apple Silicon 버전을 모두 제공합니다. 아마 의도적으로 Intel 버전을 설치해 두지 않았다면 자연스럽게 Universal로 업데이트됐을 가능성이 큽니다.

굳이 Universal을 고집할 필요는 없고, 용량도 더 작으니 Apple Silicon 전용 버전을 내려받는 편이 나아 보였습니다.

그리고 최근에는 모브 프로그래밍 등에서 Visual Studio Live Share를 쓰는 경우가 많은데, 이 기능은 당시 기준으로 아직 Apple Silicon을 정식 지원하지 않았습니다. GitHub 이슈를 보면 [추후 대응 예정](https://github.com/MicrosoftDocs/live-share/issues/4527#issuecomment-984823885)이라고 하니, 당분간은 기다릴 수밖에 없겠습니다.

### Java

Java는 Intel Mac에서는 어느 벤더를 고르든 큰 차이가 없지만, Apple Silicon에서는 이야기가 조금 달라집니다. 벤더마다 Apple Silicon을 지원하는 JDK 버전이 다르기 때문입니다. Rosetta 없이 Intel용 JDK를 실행하려 하면 `bad CPU type in executable` 오류가 날 수 있으니, 설치할 때 아키텍처를 꼭 확인해야 합니다.

각 벤더별 Apple Silicon 지원 LTS JDK는 대략 아래와 같았습니다.

| JDK | 지원 버전 |
|---|---|
| [Amazon Corretto](https://docs.aws.amazon.com/corretto/latest/corretto-17-ug/downloads-list.html) | 17 |
| [Azul Zulu](https://www.azul.com/downloads/?version=java-11-lts&os=macos&architecture=arm-64-bit&package=jdk) | 1.8, 11, 17 |
| [Bellsoft Liberica](https://bell-sw.com/pages/downloads/#/java-11-lts) | 1.8, 11, 17 |
| [Eclipse Temurin](https://adoptium.net) | 17 |
| [Microsoft](https://docs.microsoft.com/ja-jp/java/openjdk/download) | 17 |
| [Oracle Java SE](https://www.oracle.com/java/technologies/downloads/#jdk17-mac) | 17 |
| [SapMachine](https://sap.github.io/SapMachine) | 11, 17 |

위 JDK 중 일부는 IntelliJ에서도 내려받을 수 있고, IntelliJ에서는 Apple Silicon 대응 버전을 `aarch64`로 표시합니다. 설치할 때 이 표기를 꼭 확인하는 편이 좋습니다.

그 외에 [Red Hat](https://developers.redhat.com/products/openjdk/download)이나 [IBM Semeru](https://www.ibm.com/support/pages/java-sdk-downloads-version-110) 같은 선택지도 있지만, 당시에는 Apple Silicon용 JDK를 제공하지 않았습니다.

요즘은 어떤 JDK를 써도 크게 문제될 일은 많지 않다고 생각합니다. Java 17을 쓴다면 Oracle을 써도 되고, 다른 벤더를 취향에 따라 골라도 괜찮아 보입니다. 제 경우에는 예전부터 AdoptOpenJDK를 써 왔기 때문에 이번에도 Temurin을 선택했습니다. 다만 Temurin은 Java 11이 Apple Silicon을 지원하지 않았기 때문에, Java 11 쪽은 Zulu를 사용했습니다. 설치는 Homebrew로 했습니다.

Apple Silicon과 직접 관련된 내용은 아니지만, 아래 명령으로 `JAVA_HOME`을 쉽게 바꿀 수 있으니 벤더별 JDK를 시험해 보기에도 편합니다.

```bash
export JAVA_HOME=`/usr/libexec/java_home -v 11`
$ java -version
openjdk version "11.0.13" 2021-10-19 LTS
OpenJDK Runtime Environment Zulu11.52+13-CA (build 11.0.13+8-LTS)
OpenJDK 64-Bit Server VM Zulu11.52+13-CA (build 11.0.13+8-LTS, mixed mode)
$ export JAVA_HOME=`/usr/libexec/java_home -v 17`
$ java -version                                  
openjdk version "17.0.1" 2021-10-19
OpenJDK Runtime Environment Temurin-17.0.1+12 (build 17.0.1+12)
OpenJDK 64-Bit Server VM Temurin-17.0.1+12 (build 17.0.1+12, mixed mode)
```

GraalVM은 당시 기준으로 아직 Apple Silicon을 지원하지 않았습니다. 다만 2020년부터 GitHub [이슈](https://github.com/oracle/graal/issues/2666)가 열려 있고, Linux + aarch64용 바이너리는 이미 제공되고 있었기 때문에 언젠가는 출시될 것으로 보였습니다.

### Kotlin

Kotlin은 1.5.30부터 [Apple Silicon 지원](https://kotlinlang.org/docs/whatsnew1530.html#apple-silicon-support)을 발표했지만, 이건 Kotlin/Native에 관한 이야기입니다. Kotlin/JVM을 쓰는 경우에는 사실상 Java 쪽만 신경 쓰면 된다고 봐도 괜찮았습니다. 이쪽도 Homebrew로 설치했고 별다른 문제는 없었습니다.

### Gradle

Gradle은 제 경우 6.8.3을 쓰는 프로젝트가 하나 있었는데, Java 설정을 끝내고 빌드를 시도했더니 에러가 발생했습니다. 물론 실제 에러는 의존성이나 프로젝트 구성에 따라 다를 수 있습니다.

#### Java 17에서 실행한 경우

Java 17(Temurin)에서 실행하면 Gradle 자체가 런타임 에러를 냈습니다. 아마 reflection 관련 deprecated API 사용이 문제였던 것 같습니다.

```bash
> java.lang.IllegalAccessError: class org.gradle.internal.compiler.java.ClassNameCollector (in unnamed module @0x8f1317) cannot access class com.sun.tools.javac.code.Symbol$TypeSymbol (in module jdk.compiler) because module jdk.compiler does not export com.sun.tools.javac.code to unnamed module @0x8f1317
```

#### Java 11에서 실행한 경우

Java 11에서는 컴파일까지는 진행됐지만, 테스트(JUnit) 실행에서 아래와 같은 문제가 생겼습니다.

```bash
*** java.lang.instrument ASSERTION FAILED ***: "result" with message agent load/premain call failed at src/java.instrument/share/native/libinstrument/JPLISAgent.c line: 422
FATAL ERROR in native method: processing of -javaagent failed, processJavaStart failed
Process 'Gradle Test Executor 1679' finished with non-zero exit value 134
```

찾아보니 Gradle은 6.9부터 Apple Silicon을 지원하는 것 같아서 wrapper를 올렸습니다. 단순히 `gradle/wrapper/gradle-wrapper.properties`의 버전을 바꾸는 방법도 있지만, 저는 아래 명령을 사용했습니다.

```bash
./gradlew wrapper --gradle-version=7.3.1 --distribution-type=bin
```

6.8.3에서 7.3.1로 바로 올렸는데도 테스트까지 정상적으로 끝났습니다. 이 프로젝트가 Kotlin + Spring Boot 구성이었으니, 비슷한 환경이라면 Java/Kotlin/Gradle 버전부터 점검해 보는 것이 좋겠습니다.

### Ruby / Python / Go

Ruby와 Python은 macOS에 기본 설치돼 있지만, 버전이나 프로젝트 설정에 따라 문제가 생길 수 있어서 Homebrew로 다시 설치했습니다. 이 블로그에서 쓰는 [Jekyll](https://jekyllrb.com)도 Ruby를 다시 설치하면서 함께 재설치해야 했습니다. 다행히 Homebrew 기반으로 다시 맞춰도 기존 프로젝트는 별문제 없이 동작했습니다.

Python도 기존 프로젝트의 패키지를 다시 설치해야 했지만, `requirements.txt`만 잘 정리돼 있으면 크게 어렵지는 않았습니다.

Go는 1.16부터 Apple Silicon을 지원하므로, 특별히 버전을 고정해야 하는 경우가 아니라면 Homebrew로 최신 버전을 설치해도 괜찮아 보였습니다. 다만 기존 프로젝트의 `GOROOT`, `GOPATH` 설정에 따라서는 공식 사이트에서 직접 설치해 환경을 맞추는 편이 나을 수도 있습니다.

## Rosetta를 써야 하는 경우

많은 앱이 Apple Silicon을 지원하게 됐지만, 어떤 앱은 업데이트가 사실상 멈췄거나, Rosetta로도 큰 문제 없이 돌아가기 때문에 Apple Silicon 대응을 뒤로 미루는 경우가 있습니다. 이런 앱은 네이티브 바이너리가 아예 없어서 결국 Rosetta를 쓸 수밖에 없는 경우도 있습니다.

소스 코드를 받아 직접 빌드하는 방법도 생각해 볼 수는 있습니다. 하지만 사용 중인 Swift 버전이 오래된 경우에는 Xcode에서 바로 빌드되지 않는 일도 있었습니다. 장기적으로는 Rosetta 지원도 끝날 가능성이 높으니, 이런 앱은 언젠가 다른 대안으로 바꾸는 편이 나을 수도 있습니다.

### Mattermost

[Mattermost](https://mattermost.com/download)는 Slack과 비슷한 커뮤니케이션 도구입니다. 서버에 설치하면 무료로 쓸 수 있고, 마크다운 지원도 좋아서 개인적으로 자주 사용하고 있습니다. 다만 당시 기준으로는 Apple Silicon용 정식 릴리스가 아직 없었습니다.

정식 버전이 나올 예정이라면 당분간 Intel 버전을 Rosetta로 쓰는 것도 방법입니다. 하지만 [GitHub 릴리스 페이지](https://github.com/mattermost/desktop/releases)를 보면 Universal과 Apple Silicon용 베타 바이너리도 있었기 때문에, Rosetta를 정말 피하고 싶다면 그쪽을 써보는 것도 가능해 보였습니다.

### KeyboardCleanTool

[KeyboardCleanTool](https://folivora.ai/keyboardcleantool)은 실행 중 모든 키 입력을 무시하게 해 주는 단순한 도구입니다. 키보드를 닦을 때 유용해서 자주 사용합니다. 하지만 이 앱도 당시에는 Apple Silicon을 지원하지 않았습니다.

같은 회사의 [BetterTouchTool](https://folivora.ai)는 Universal 바이너리로 제공되고 있었지만, 그 대응도 비교적 최근에 이뤄졌기 때문에 다른 제품들까지 모두 Apple Silicon을 지원하려면 시간이 꽤 걸릴 것 같았습니다.

사실 이런 도구는 꼭 네이티브가 아니어도 큰 문제가 없어서, Apple Silicon 대응 우선순위가 낮은 것도 이해는 됩니다.

## OneDrive

Microsoft 제품치고는 조금 드물게, OneDrive는 대응이 꽤 느린 편이었습니다. 다만 그달에 [Preview 형태로 Universal 버전을 사용할 수 있게 됐다](https://techcommunity.microsoft.com/t5/microsoft-onedrive-blog/onedrive-sync-for-native-arm-devices-now-in-public-preview/ba-p/3031668)는 소식이 있었기 때문에, 곧 정식 Apple Silicon 지원 버전이 나올 것으로 보였습니다. App Store로 설치한 경우에는 자동 업데이트를 기다리면 될 것 같습니다.

## Flutter

Dart는 [2.14부터 네이티브 대응](https://medium.com/dartlang/announcing-dart-2-14-b48b9bb2fb67)을 하고 있지만, Flutter는 당시 기준으로 아직 정식 대응이 아니었습니다. 공식 문서에서도 [Rosetta 사용을 권장](https://github.com/flutter/flutter/wiki/Developing-with-Flutter-on-Apple-Silicon)하고 있었기 때문에, 이쪽은 Rosetta를 설치해서 쓰는 편이 더 빠릅니다.

또 `flutter doctor`를 실행했을 때 몇 가지 에러가 뜨는 경우가 있었는데, 제 경우 아래 방식으로 해결할 수 있었습니다.

- `cmdline-tools component is missing`가 뜨는 경우
  - Android Studio에서 `Appearance & Behavior` -> `System Settings` -> `Android SDK` -> `SDK Tools` -> `Android SDK Command-Line Tools`를 체크
- `CocoaPods installed but not working`가 뜨는 경우
  - `brew install cocoapods` 실행

## 마지막으로

Apple Silicon Mac으로 옮긴 뒤 본격적으로 사용한 기간은 아직 일주일이 채 되지 않았지만, 생각보다 마이그레이션은 매끄러웠고 네이티브 대응이 끝난 앱도 많았습니다. 그래서 M1 발표 직후의 저처럼 호환성이 걱정돼 망설이고 있는 분이 있다면, 충분히 사전 조사를 한다는 전제하에 이제는 넘어가도 괜찮은 시점이라고 생각합니다.

처음에는 Apple이 모든 Mac을 Apple Silicon으로 옮기는 데 2년이 걸릴 거라고 했을 때, 실제 사용자 환경은 훨씬 더 천천히 바뀔 거라고 생각했습니다. 그런데 막상 써 보니, 이미 이 시점에도 충분히 실사용 가능한 수준에 올라와 있었습니다.
