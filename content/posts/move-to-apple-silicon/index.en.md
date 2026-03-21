---
title: "Migrating to Apple Silicon Mac"
date: 2021-12-19
categories: 
  - recent
image: "../../images/magic.webp"
tags:
  - mac
---
It has been more than a year since the M1 Mac was released. At first, even though [Rosetta](https://en.wikipedia.org/wiki/Rosetta_(software)) was available, I did not think of buying an Apple Silicon Mac right away because there were few native apps and there were many cases of problems and performance deterioration. However, now that there are more native apps and there have been OS updates, I felt that it was a good time to make the transition. Then, this year, new Macs equipped with the M1 Pro and M1 Max were announced, and I decided to purchase one because I thought they had many advantages compared to existing Intel-based models.

I have the impression that the new Macs are quite good in terms of chip performance and improved hardware (display, speakers, keyboard, etc.), but I will not discuss that here. I think this has already been made clear by many benchmarks and reviews, so I would like to talk about my migration experience from an Intel machine here.

## How to migrate

First, after deciding to migrate to Apple Silicon Mac, I set the following criteria.

- Migrate from an existing Mac
- Use native apps as much as possible

The reason I decided to migrate was because I didn't want to review the settings of many of the apps I was currently using from scratch. Since the architecture is different, I thought there might be some problems compared to those who don't migrate, but I haven't had any problems so far while actually using it, so if it's too troublesome to start over from scratch, it seems like it might be a good idea to migrate.

I also used "Migration Assistant" for the migration. In this case, the following migration methods are available:

- Time Machine
- Migrate from Mac
  - Use Wi-Fi
  - Connect with Thunderbolt cable

If I use Time Machine, the latest status will not be reflected, so I decided to migrate from the Mac I was using. Also, since it takes the least amount of time to copy data, I connect the two Macs directly using Thunderbolt to perform the migration. The migration took less time than I expected, and the state immediately after it was finished was the same as the existing one.

After migration, I want to use Apple Silicon native apps, so I decided to check the app information one by one and switch binaries from non-Universal ones.

## Check the binaries

First, once the migration is complete, there is a way to tell whether the installed app is native to Apple Silicon or not.

- "View information" from Finder
- See the "type" of activity monitor
- Search from "About this Mac" → "System Report" → "Software" → "Applications"
- Search from [Is Apple Silicon ready?](https://isapplesiliconready.com)

Another option is to use the Intel version of the app, and when you run it for the first time, a dialog box will appear asking if you want to install Rosetta. This is mainly the case with programming languages ​​used in terminals. However, once you install Rosetta, even the Intel version of the binary will run without a dialog, so I think it is better not to install Rosetta until the entire migration is complete.

## Application switching

The following provides a Universal Binary (compatible with both Intel and Apple Silicon), so you don't need to do anything in particular even if you are migrating from an Intel machine.- [Chrome](https://www.google.com/chrome/)
- [Edge](https://www.microsoft.com/edge)
- [Firefox](https://www.mozilla.org/firefox/new/)
- [EdgeView2](https://apps.apple.com/us/app/edgeview-2/id1206246482)
- [Movist Pro](https://movistprime.com/en)
- [Amphetamine](https://apps.apple.com/us/app/amphetamine/id937984704)
- [Bandizip](https://apps.apple.com/us/app/id1265704574)
- [Obsidian](https://obsidian.md)
- [Magnet](https://apps.apple.com/us/app/magnet/id441258766)
- [Macs Fan Control](https://crystalidea.com/macs-fan-control/download)
- [iStat Menus](https://apps.apple.com/us/app/istat-menus/id1319778037)
- [Unicorn](https://apps.apple.com/us/app/unicorn-block-ads-for-mobile/id1231935892)
- [Microsoft Remote Desktop](https://apps.apple.com/us/app/microsoft-remote-desktop/id1295203466)
- [Microsoft Word](https://apps.apple.com/us/app/microsoft-word/id462054704)
- [Microsoft PowerPoint](https://apps.apple.com/us/app/microsoft-powerpoint/id462062816)
- [Microsoft Excel](https://apps.apple.com/us/app/microsoft-excel/id462058435)
- [Microsoft OneNote](https://apps.apple.com/us/app/microsoft-onenote/id784801555)
- [Microsoft Outlook](https://apps.apple.com/us/app/microsoft-outlook/id985367838)

One, in the case of Macs Fan Control, I use it to display the CPU temperature on the menu bar, but when selecting the item to display the temperature, I think it is more common to be able to select "CPU PECI" on Intel machines, but there is no such item on Apple Silicon. After looking at the overall temperature for a while, "CPU Performance Core" seemed to have the highest temperature, so I think it would be better to select that if you want to check the temperature.

## Cases in which you need to switch binaries

The following binaries for Apple Silicon are provided separately, so all you had to do was download them from the website and overwrite the existing apps.

- [Notion Desktop](https://www.notion.so/desktop)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Postman](https://www.postman.com/downloads/)
- [Zoom](https://zoom.us/download)
- [Webex](https://www.webex.com/downloads.html)
- [draw.io](https://github.com/jgraph/drawio-desktop/releases)

However, even though Docker uses Apple Silicon native for the desktop app itself, there are many cases where the image only supports AMD64, so I think there is a need for various verifications here. In my environment, MySQL (5.7) was not yet compatible with Apple Silicon, but it is working without any problems.

## Homebrew

It seems that it is possible to use the existing Intel version binary using Rosetta, but the installation path for the version compatible with Apple Silicon is different (the existing version is `/usr/local`, and the Apple Silicon version is `/opt/homebrew`), and in the Apple Silicon version, the installed packages are basically native to Apple Silicon, or the Intel version also seems to build a new one, so I decided to delete the Intel version I was using and reinstall it.There was no need to be conscious of anything when uninstalling and installing. As shown on the official website, all you need to do is run the following command. It will automatically delete the Intel version, and if you install a new one, it will be the Apple Silicon version.

```bash
# Uninstall
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"

# Install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

However, in my case, my policy was to delete all the packages installed with homebrew and install them later if I needed them, but it seems that you can back up the list of packages used with the following command.

```bash
/usr/local/homebrew/bin/brew bundle dump
```

## Development environment

There are various cases when it comes to building a development environment, so we will explain them separately here.

## JetBrains IDE

JetBrains products (+ Android Studio) can be easily managed using [Toolbox](https://www.jetbrains.com/toolbox-app/), but this app itself also needs to be converted to a binary compatible with Apple Silicon.

After switching Toolbox to Apple Silicon native, simply uninstall and reinstall the IDE from the menu. If you are not using Toolbox, you will need to redownload the binaries for your IDE.

By the way, the IDE installed from Toolbox has a Launcher under `~/Applications/JetBrains Toolbox`, and when you check it with the system report, it is an Intel version binary. However, don't worry as it is native to Apple Silicon when run (you can check it from Activity Monitor).

## Visual Studio Code

In the case of [Visual Studio Code](https://code.visualstudio.com/download), we provided all binaries for Universal/Intel/Apple Silicon. I think if I hadn't purposely installed the Intel version, it would have automatically updated to Universal.

There's no need to use Universal, and I think it's better to download the Apple Silicon version because it's smaller.

Also, I think there are many cases where Visual Studio Live Share is used in Mob Pro etc. these days, but it is not yet compatible with Apple Silicon. This is a GitHub issue [planned to be addressed in the future](https://github.com/MicrosoftDocs/live-share/issues/4527#issuecomment-984823885), so we'll just have to wait for the time being.

## Java

In the case of Java, if it's Intel, it doesn't matter which vendor you choose, but if it's Apple Silicon, the story is a little different. The reason is that the versions of JDK that support Apple Silicon differ depending on the vendor. If you are not using Rosetta, you will get a "bad CPU type in executable" error, so please check the installed version.

Below is a list of LTS versions of JDKs that support Apple Silicon by each vendor.

| JDK | Supported versions |
|---|---|
| [Amazon Corretto](https://docs.aws.amazon.com/corretto/latest/corretto-17-ug/downloads-list.html) | 17 |
| [Azul Zulu](https://www.azul.com/downloads/?version=java-11-lts&os=macos&architecture=arm-64-bit&package=jdk) | 1.8, 11, 17 |
| [Bellsoft Liberica](https://bell-sw.com/pages/downloads/#/java-11-lts) | 1.8, 11, 17 |
| [Eclipse Temurin](https://adoptium.net) | 17 |
| [Microsoft](https://learn.microsoft.com/java/openjdk/download) | 17 |
| [Oracle Java SE](https://www.oracle.com/java/technologies/downloads/#jdk17-mac) | 17 |
| [SapMachine](https://sap.github.io/SapMachine) | 11, 17 |

Some of the above JDKs can be downloaded from intellij, and the version compatible with Apple Silicon is displayed as `aarch64` on intellij, so be sure to check this when installing.

There are also [Red Hat](https://developers.redhat.com/products/openjdk/download) and [IBM Semeru](https://www.ibm.com/support/pages/java-sdk-downloads-version-110), but they do not provide a JDK for Apple Silicon.These days, I don't think there's any particular problem with which JDK you choose, so if you choose 17, you can choose Oracle, or you can choose any other JDK you like. In my case, I had been using AdpotOpenJDK, so I chose Temurin again this time. However, Temurin 11 does not support Apple Silicon, so I chose Zulu for that. I used homebrew for installation.

Although there is no direct relationship with Apple Silicon, you can easily switch the JDK version used with the following command, so you may want to try using JDKs from various vendors.

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

GraalVM does not yet support Apple Silicon. However, the [issue](https://github.com/oracle/graal/issues/2666) on Github has been open since 2020, and binaries compatible with Linux+aarch64 are being provided, so I think it will be released eventually.

## Kotlin

[Apple Silicon support](https://kotlinlang.org/docs/whatsnew1530.html#apple-silicon-support) has been announced for Kotlin from 1.5.30, but this is related to Kotlin/Native, so in the case of Kotlin/JVM, you only need to be careful about Java. I also installed it using homebrew and had no problems.

## Gradle

In the case of Gradle, I had a project that used v6.8.3, so when I tried to build it after configuring Java, the following error occurred. (I think the type of error may change depending on dependencies and project settings.)

### When running on Java 17

When running on Java 17 (Temurin), Gradle itself throws an error at runtime. I think the problem may have been the use of a deprecated API related to referral.

```bash
> java.lang.IllegalAccessError: class org.gradle.internal.compiler.java.ClassNameCollector (in unnamed module @0x8f1317) cannot access class com.sun.tools.javac.code.Symbol$TypeSymbol (in module jdk.compiler) because module jdk.compiler does not export com.sun.tools.javac.code to unnamed module @0x8f1317
```

### When running on Java 11

When running with Java 11, it seems to compile, but the following problem occurred when running the test (junit).

```bash
*** java.lang.instrument ASSERTION FAILED ***: "result" with message agent load/premain call failed at src/java.instrument/share/native/libinstrument/JPLISAgent.c line: 422
FATAL ERROR in native method: processing of -javaagent failed, processJavaStart failed
Process 'Gradle Test Executor 1679' finished with non-zero exit value 134
```

When I looked into it, it seems that Gradle has been compatible with Apple Silicon since v6.9, so I updated the wrapper to the latest version. This can be done simply by changing the version specification of `gradle/wrapper/gradle-wrapper.properties`, but I am using the following command.

```bash
./gradlew wrapper --gradle-version=7.3.1 --distribution-type=bin
```

I suddenly updated from v6.8.3 to v7.3.1, and the test completed successfully. This project has a Kotlin + Spring boot configuration, so if you have a project with a similar configuration, try updating the Java/Kotlin/Gradle version before trying to build the app.

## Ruby / Python / Go

Ruby and Python are already installed on macOS, but problems may occur depending on the version and project settings, so I installed them using homebrew. [jekyll](https://jekyllrb.com), which I use in this blog, reinstalled ruby, so I had to reinstall it as well. This can also be installed using homebrew, and I was able to run it in existing projects without any problems.

For Python, I had to reinstall packages for existing projects. But with `requirements.txt`, it's not really a problem.

Go has also been compatible with Apple Silicon since 1.16, so unless you specifically need to specify the version, I think you can install the latest version using homebrew. However, there are problems with GOROOT and GOPATH for existing projects, so there may be cases where you need to download and configure them separately from the homepage.

## Cases where you have no choice but to use Rosetta

Many apps have become compatible with Apple Silicon, but for apps that have been updated or run without problems through Rosetta (and have a policy of putting support for Apple Silicon on the back burner), native binaries may not exist, so I think I have no choice but to use Rosetta.

I think there is a way to download the source code and build it locally, but if the Swift version being used is lower, it may not be possible to build it immediately with XCode. It looks like Rosetta will no longer be supported at some point, so from a long-term perspective, it might be better to replace this type of app with something else.

## Mattermost

[Mattermost](https://mattermost.com/download) is a communication tool similar to Slack. It can be used for free by installing it on the server, and it has excellent markdown support, so I often use it in my private life. Unfortunately, it seems that there is no official release version for Apple Silicon yet.There seems to be a plan for an official release, so you have the option of using the intel version until the version is updated, but if you look at [GitHub repository](https://github.com/mattermost/desktop/releases), there are binary beta versions for Universal and Apple Silicon, so if you really don't want to use Rosetta, you may want to choose this option.

## KeyboardCleanTool

[KeyboardCleanTool](https://folivora.ai/keyboardcleantool) is a simple tool that ignores all keystrokes while your app is running. I often use this when my keyboard is dirty and I want to clean it. Unfortunately, it still does not support Apple Silicon. [BetterTouchTool](https://folivora.ai), which is developed by the same company, is provided as a universal binary, but it only became compatible in November, so it may take quite a while for the rest of the products to become Apple Silicon compatible.

Apps like this don't really need to be native, so I feel like native support for Apple Silicon is a fairly low priority.

## OneDrive

This is a fairly rare case for a Microsoft product, but the response has been delayed. However, it seems that [the Universal version is available as a Preview this month](https://techcommunity.microsoft.com/t5/microsoft-onedrive-blog/onedrive-sync-for-native-arm-devices-now-in-public-preview/ba-p/3031668), so an Apple Silicon native version may be released soon. In that case, it should be updated automatically on the App Store, so you just have to wait.

## Flutter

Dart has been [Native compatible since v2.14](https://medium.com/dartlang/announcing-dart-2-14-b48b9bb2fb67), but Flutter doesn't seem to be compatible yet, and the official version recommends [Rossetta](https://github.com/flutter/flutter/wiki/ㅅDeveloping-with-Flutter-on-Apple-Silicon). So it would be faster to install Rosetta and run it.

There may be other cases where you may run into some problems when running `flutter doctor`. In such a case, I was able to deal with it by following the steps below.

- If `cmdline-tools component is missing` appears
  - Check `Appearance & Behavior` -> `System Settings` -> `Android SDK` -> `SDK Tools` -> `Android SDK Command-Line Tools` from Android Studio
- If `CocoaPods installed but not working` appears
  - Installed with `brew install cocoapods`

## Finally

It's been less than a week since I've been using an Apple Silicon Mac in earnest since the transition, but the transition was smoother than I expected, and many apps were natively compatible, so if you're like me right after the M1 was announced and have doubts about compatibility, I think it might be a good idea to migrate to a new Mac (assuming you've done enough research beforehand).

Initially, Apple was talking about a two-year plan to officially migrate all Macs to Apple Silicon, but as a user, I thought it would take longer. However, after actually trying it out, I think it is still in a good enough condition to make the transition.

See you soon!
