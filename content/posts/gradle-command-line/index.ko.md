---
title: "Gradle로 명령줄 인수 넘기기"
date: 2019-09-17
categories:
  - gradle
image: "../../images/gradle.webp"
tags:
  - groovy
  - java
  - gradle
translationKey: "posts/gradle-command-line"
---

최근에 맡은 일은 기존 라이브러리를 확장하는 작업이었습니다. 처음에는 단순히 라이브러리만 조금 손보는 일이라고 생각했는데, 실제로는 Spring Boot 기반으로 동작하는 실행 프로그램까지 함께 다뤄야 했습니다. Spring Boot 자체는 처음 본 것은 아니었지만, 웹 화면이 없는 도구형 프로그램을 Spring Boot로 구성하는 방식은 그전까지 거의 경험이 없었습니다.

이 일을 하면서 새로 알게 된 점은, 언어와 프레임워크를 안다는 것과 그것을 어떻게 조합해 쓰느냐는 전혀 다른 문제라는 것입니다. 화면이 없어도 `main` 클래스를 두면 Jar로 독립 실행할 수 있고, 외부에서 임포트하는 라이브러리로도 사용할 수 있습니다. 다만 그렇게 하려면 약간의 준비가 필요했습니다.

이번 글에서 정리하려는 요구사항은 아래와 같았습니다.

## Gradle task에서 Spring Boot 애플리케이션을 실행하고 명령줄 인수를 전달하기

문장만 보면 거창해 보이지만, 실제로는 그렇게 복잡한 이야기는 아닙니다. Gradle은 Maven처럼 라이브러리 관리와 빌드를 맡는 도구이고, `gradlew`에 여러 옵션을 붙여 Jar 빌드나 테스트 같은 작업을 실행할 수 있습니다. 여기서 하고 싶은 일은 Gradle task 하나를 추가해, 그 task를 실행하면 Spring Boot 프로젝트의 `main` 클래스를 함께 띄우는 것입니다.

보통은 Spring Boot 프로젝트에 `main` 클래스만 있으면 `java -jar project.jar` 형태로 직접 실행할 수 있습니다. 그런데 굳이 Gradle task로 감싼 이유는 다음과 같습니다.

### 멀티 프로젝트 구조이기 때문입니다

현재 구조는 하나의 루트 프로젝트 아래에 여러 하위 프로젝트가 붙는 멀티 프로젝트입니다. 대략 이런 형태입니다.

```text
rootproject
┣ target
┣ generator
┗ runtime
```

간단히 설명하면 `generator`가 실제 처리를 담당하고, `runtime`은 그 실행을 보조합니다. 최종적으로는 `generator`와 `runtime`을 Jar로 배포하고, 사용자는 처리하고 싶은 대상 프로젝트를 `target` 위치에 둔 뒤 이 도구를 실행하게 됩니다. 이런 구조에서는 매번 커맨드라인에서 `generator` Jar를 직접 실행하고, 대상 경로와 옵션을 손으로 넘기는 과정이 꽤 번거롭습니다. 그래서 이 과정을 Gradle task로 묶고 싶었습니다.

### 넘겨야 할 인수가 많기 때문입니다

또 다른 이유는 라이브러리가 처리에 필요한 인수가 많았기 때문입니다. 대략 7개 정도 되는 값을 매번 기억해서 입력하는 것은 쉽지 않고, 오타가 나면 처리 자체가 실패할 수도 있습니다.

처음에는 `.bat` 파일이나 `.sh` 파일을 따로 두고 거기에 값을 적어 두는 방법도 떠올렸습니다. 하지만 OS마다 파일을 따로 관리해야 하고, 실행 환경이 달라질수록 유지보수가 어려워집니다. 그래서 환경 차이는 Gradle이 맡고, 사용자는 파일만 수정하면 되도록 만드는 편이 더 낫다고 판단했습니다.

## [target] build.gradle

먼저 task를 실행할 쪽인 `target` 프로젝트의 `build.gradle`부터 보겠습니다. Gradle 프로젝트는 기본적으로 `build.gradle` 파일에 의존성, 플러그인, task 등을 정의합니다.

```groovy
apply from: 'default.gradle' // 기본 설정을 읽어온다

task taskname(type: GradleBuild) {
    group = 'application'
    description = 'run program'
    dir = '../generator'
    tasks = ['bootRun']
    startParameter.projectProperties = [args: "${defaultArgs}"]
}
```

여기서 `default.gradle`을 따로 읽는 이유는 기본 인수를 파일에 분리해 두기 위해서입니다. 실제로 task를 실행할 때는 이 파일의 값을 읽어 `args`로 넘깁니다. 이렇게 해 두면 사용자는 `default.gradle`만 수정해서 실행 인수를 바꿀 수 있습니다.

## [target] default.gradle

`build.gradle`에서 읽어올 기본 인수는 다음처럼 적어 둡니다.

```groovy
ext {
    defaultArgs = '-arg1 value -arg2 value'
}
```

인수 이름을 명시해 두면 순서는 크게 중요하지 않습니다. 필요한 값만 바꿔서 쓰면 되므로 실수도 줄어듭니다.

## [generator] build.gradle

다음은 실제로 실행되는 쪽인 `generator` 프로젝트 설정입니다. 여기서는 `bootRun` 시 어떤 인수를 받고, Jar의 진입점이 무엇인지 정합니다.

```groovy
bootRun {
    if (project.hasProperty('args')) {
        args project.args.split('\\s+')
    }
}

jar {
    manifest {
        attributes 'Main-Class': 'com.package.Main'
    }
}
```

명령줄 인수를 공백 기준으로 나누는 이유는 Java의 `main` 메서드가 문자열 배열을 받기 때문입니다. 이렇게 해 두면 실제 처리 코드에서 인수를 다루기 쉬워집니다.

여기까지가 Gradle 설정입니다. 다음은 Java의 `main` 클래스입니다.

## [generator] Main.java

이 클래스는 `generator`의 Jar를 실행했을 때 호출됩니다. 일반적인 Java `main` 메서드, Spring Boot 진입점, JCommander를 이용한 명령줄 파싱, Lombok이 함께 섞여 있어서 처음 보면 조금 복잡해 보일 수 있습니다. 하지만 실제 동작은 단순합니다.

```java
@SpringBootApplication
@EnableAutoConfiguration(exclude = { DataSourceAutoConfiguration.class })
public class Main implements CommandLineRunner {

    @Autowired
    private CoreProcessClass coreProcessClass;

    @Override
    public void run(final String... args) throws Exception {
        final CommandLineOptions options = CommandLineOptions.parse(args);
        coreProcessClass.startProcess(options.getArg1(), options.getArg2());
    }

    public static void main(final String[] args) {
        SpringApplication.run(Main.class, args);
    }

    @Data
    public static class CommandLineOptions {

        @Parameter(names = "-arg1", description = "File", required = true, converter = ArgumentsToFileConverter.class)
        private File arg1;

        @Parameter(names = "-arg2", description = "String", required = true)
        private String arg2;

        private CommandLineOptions() {}

        public static CommandLineOptions parse(final String... args) {
            try {
                final CommandLineOptions options = new CommandLineOptions();
                JCommander.newBuilder()
                        .addObject(options)
                        .acceptUnknownOptions(true)
                        .build()
                        .parse(args);
                return options;
            } catch (ParameterException e) {
                e.getJCommander().usage();
                throw e;
            }
        }
    }

    public class ArgumentsToFileConverter implements IStringConverter<File> {

        @Override
        public File convert(final String argument) {
            return new File(argument);
        }
    }
}
```

[JCommander](http://jcommander.org)를 쓰면 명령줄 파싱을 꽤 간단하게 할 수 있습니다. 필수 인수 지정, 타입 변환, 옵션 무시 같은 처리를 한 번에 묶을 수 있어서 이런 도구에 잘 맞습니다. 예를 들어 문자열로 받은 경로를 `File`로 바꾸거나, 복수 인수를 객체로 정리하는 식입니다.

파싱이 끝나면 그 결과를 실제 처리 클래스에 넘기기만 하면 됩니다. 복잡해 보이지만 핵심은 “파일에서 기본 인수를 읽고, Gradle task가 그 값을 넘기고, Java 쪽에서 파싱해서 처리한다”는 흐름입니다.

## 마지막으로

사실 이 구조는 제가 처음부터 설계한 것은 아닙니다. Gradle task를 만드는 과정에서 겪은 라이브러리 충돌 문제 때문에 한동안 막혔고, 이미 완성된 예제를 참고해 풀어낸 부분도 있습니다. 그래도 이런 방식이 실제로 쓸 만하다고 판단했기 때문에 정리해 두었습니다.

특히 서버용 애플리케이션이나 도구형 프로그램은 커맨드라인으로 실행하는 경우가 많고, 환경마다 넘겨야 하는 값도 달라집니다. 이럴 때 인수를 파일로 분리해 두고, Gradle task가 그 값을 읽어 실행하도록 만들면 실행 방법을 꽤 단순하게 정리할 수 있습니다.

처음에는 조금 돌아가는 방식처럼 느껴졌지만, 여러 프로젝트와 실행 옵션을 함께 관리해야 하는 상황에서는 생각보다 실용적인 구조였습니다.
