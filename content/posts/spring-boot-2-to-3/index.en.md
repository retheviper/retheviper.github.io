---
title: "Migrating to Spring Boot 3"
date: 2023-01-29
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - kotlin
  - java
  - spring
---
I upgraded from Spring Boot 2 (Spring Framework 5) to Spring Boot 3 (Spring Framework 6).

## Java 17

When I used Java at work, I was waiting for Java 17. I previously organized it as an article on this blog, but this is because many useful functions have been added, such as using text blocks and Switch in formulas. However, now that we are using Kotlin, there is no need to consider the Java version in terms of functionality or language specifications. So if your existing applications are running on Java 11, there may not seem to be much motivation to upgrade to Java 17.

However, I still decided to upgrade because support milestones were approaching. More precisely, Premier Support for Java 11 would end in September 2023, and Extended Support in September 2026. Rather than leaning too much on those labels, I felt Java 11 was already old enough by that point, while Java 17 had already been in practical use for more than a year.

Just talking about the support period, there is still more than half a year left until the end of the support period, but the reason we are doing it at this timing is because the company has set up a ``refactor period.'' During this period, we will mainly focus on eliminating technical debt and updating dependencies, so we wanted to avoid anything that would cause problems due to the overlap of new feature development and version updates.

There are other ways in which performance can be improved by adopting a new Java version. For example, [GC performance has been improved compared to Java 11](https://www.optaplanner.org/blog/2021/09/15/HowMuchFasterIsJava17.html), and depending on the situation, there is an option to [consider implementing ZGC](https://malloc.se/blog/zgc-jdk17). Upgrading the Java version will also include improvements to the JVM, so Kotlin will be able to benefit from this as well.

So, I decided to adopt Java 17 and would like to share my experience since then.

## Records

We use [jOOQ](https://www.jooq.org/) as our ORM within our company, and the structure is such that the automatically generated table definition code is called using Kotlin code (Repository). However, in some cases, this automatically generated code may fail to compile with Java 17. In this case, I was able to see that the compilation failed with the following error message:

```text
error: reference to Record is ambiguous
    public <O extends Record> SomeTable(Table<O> child, ForeignKey<O, SomeRecord> key) {
                      ^
  both interface org.jooq.Record in org.jooq and class java.lang.Record in java.lang match
```

This is because jOOQ has a class called [Record](https://www.jooq.org/javadoc/latest/org.jooq/org/jooq/Record.html), and it is used in the automatically generated code. Since Java 14, [Record Class](https://docs.oracle.com/en/java/javase/15/language/records.html), which has a similar functionality to Lombok's [@Value](https://projectlombok.org/features/Value), has appeared, and all classes defined using the `record` keyword implement [java.lang.Record](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Record.html). As a result, an error occurs saying, ``The compiler cannot determine whether you mean a jOOQ Record or a Java Record.''

This can be avoided by simply modifying the existing auto-generated code imports. In the existing code, the automatically generated code imports the jOOQ class as shown below.

```java
import org.jooq.*;
```

Correct this to clearly specify jOOQ Record.

```java
import org.jooq.Record;
import org.jooq.*;
```

Now it compiles without any problems. If you have defined a class called `Record` in your own library, you may get the same error as here, so it might be a good idea to rename the class if possible.

## Docker Images

The service we were developing is containerized with [Jib](https://github.com/GoogleContainerTools/jib). That means we also need to update the base image. We had been using OpenJDK's [11-jre](https://hub.docker.com/layers/library/openjdk/11-jre/images/sha256-762d8d035c3b1c98d30c5385f394f4d762302ba9ee8e0da8c93344c688d160b2?context=explore), but from Java 17 onward the official OpenJDK images no longer provide a JRE-only tag, only [17-jdk](https://hub.docker.com/layers/library/openjdk/17-jdk/images/sha256-98f0304b3a3b7c12ce641177a99d1f3be56f532473a528fda38d53d519cafb13?context=explore). That is something to watch for during the upgrade.

However, if you are using an image other than OpenJDK, the situation may differ. For example, [Temurin](https://hub.docker.com/layers/library/eclipse-temurin/17-jre/images/sha256-402c656f078bc116a6db1e2e23b08c6f4a78920a2c804ea4c2d3e197f1d6b47c?context=explore) and [Zulu](https://hub.docker.com/layers/azul/zulu-openjdk/17-jre/images/sha256-09163c13aefbe5e0aa3dc7910d9997e416031e777ca0c50bd9b66a63b46be481?context=explore) still provide `17-jre`, while Liberica provides only [17](https://hub.docker.com/layers/bellsoft/liberica-openjdk-alpine/17/images/sha256-50feb980f142b152e19c9dc25c835e0e451321eb3b837a3a924f457b11ce8c59?context=explore). Since tag names vary by JDK distribution, it is worth checking the tags for the exact image you use during the upgrade.

## Dependencies

Although this did not happen in the application I own directly, one of our other Kotlin services ran into a problem with [JasperReports](https://github.com/TIBCOSoftware/jasperreports) when moving from Java 11 to 17. We use it to generate PDFs, and while the layout itself was fine, the number of characters shown in table cells decreased slightly. Fortunately, this was not a major issue for us, so for now we decided not to address it. Depending on your system, though, it could be a serious problem.

If such a problem occurs, I think it can probably be resolved by upgrading the version of the dependent library to one that is compatible with Java 17, but there may be libraries that are not compatible with Java 17 yet, so it would be better to check the dependencies in advance.

## Spring Boot 3

Java 17 was one thing, but Spring Boot 3 itself had only been released in December of the previous year, so strictly speaking there was no urgent need to migrate immediately. Still, Spring Boot 3 makes Java 17 the minimum version, and I had several projects running on serverless [GAE](https://en.wikipedia.org/wiki/Google_App_Engine), so I also wanted to try [Spring Native](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html) in order to reduce startup time. Spring Native became officially supported starting with Spring Boot 3.

Also, even if there were cases where dependencies changed or the way properties were written changed in Spring Boot 3, since the application I was working on was a microservice, I could predict that the impact would be relatively small. I think it may be difficult to migrate if it is monolithic and depends on various Spring functions. Therefore, if you are thinking of migrating, it is better to check the release notes etc. as much as possible.

When migrating from Spring Boot 2 to 3, the [official migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide) is the right place to start. Still, some issues do not become obvious until you try the upgrade yourself, and changes in Spring Framework dependencies can affect other libraries. So I want to summarize how I handled it in my own case.

### `@ConstructorBinding`

[@ConstructorBinding](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/context/properties/ConstructorBinding.html) is an annotation for constructor injection of the value read from the application properties file into the class with [@ConfigurationProperties](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/context/properties/ConfigurationProperties.html). In the current project, the AWS S3 bucket name, email template, endpoint for calling other APIs, etc. are written in `application.yaml` and loaded.

Here, in Spring Boot 3, the property is read even without `@ConstructorBinding`, and this annotation itself is Deprecated in the first place. So, at first a compile error occurred, but by simply deleting `@ConstructorBinding` it started working without any problems. It was the easiest solution.

## Tracing changes

In the current application, when calling APIs of other microservices, we use [Brave](https://github.com/openzipkin/brave) and put `Trace Context` in the header. That way the same trace ID is linked across services, making logs easier to follow. Until now, Brave had been injected through [Spring Cloud Sleuth](https://spring.io/projects/spring-cloud-sleuth), but in Spring Boot 3 that approach is gone because the functionality has effectively been absorbed into Spring Boot, with tracing now centered on the new [Micrometer](https://micrometer.io/) APIs.

In other words, the existing DI using Spring Cloud Sleuth can no longer be used, and the dependencies will also change. First of all, [BraveAutoConfiguration](https://docs.spring.io/spring-boot/docs/3.0.1/api/org/springframework/boot/actuate/autoconfigure/tracing/BraveAutoConfiguration.html), which was in charge of DI, has been moved to [Spring Boot Actuator](https://github.com/spring-projects/spring-boot/tree/v3.0.2/spring-boot-project/spring-boot-actuator). Therefore, Spring Cloud Sleuth is no longer needed.

However, DI using `BraveAutoConfiguration` is not possible with only Spring Boot Actuator. So you need to add the bridge dependency as below. (The version used is Spring Boot.)

```kotlin
implementation("io.micrometer:micrometer-tracing-bridge-brave")
```

Since the dependencies will change, the settings for `application.yaml` will also need to be changed. Previously, it was written as follows.

```yml
spring:
  sleuth:
    enabled: true
    propagation:
      type:
        - w3c
```

Since I have moved to Micrometer, I will change it like this. Actually, the following settings are defaults, so there is no problem even if they are not listed, but they are listed to make it easier to understand the current settings during maintenance.

```yml
management:
  tracing:
    enabled: true
    propagation:
      type: w3c
```

However, the most difficult part was the unit test. When I ran the test, the following error message appeared.

```shell
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'brave.Tracing' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {}
```

When I was using Spring Cloud Sleuth, this dependency could be injected in unit tests, but for some reason the bean from `BraveAutoConfiguration` was no longer available. Because the same error did not occur during local startup, it looked like this auto-configuration was simply not being applied in tests.

I think there is a way to register the bean directly using something like [@TestConfiguration](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/context/TestConfiguration.html), but after doing some research, I found out that [Micrometer has a library for testing](https://www.baeldung.com/spring-boot-3-observability#testing-observations), so I decided to use that. This library was released in December 2022, so it is difficult to find information on it. Anyway, the DI problem can be resolved by adding the library as below.

```kotlin
testImplementation("io.micrometer:micrometer-tracing-test")
```

This time, simply adding the library solved the problem. The difficult part was that there was not much information about this new library yet. The same kind of issue may appear in other libraries as well, so if a bean that should be provided by auto-configuration cannot be found in tests, it is worth checking whether a dedicated test library has been added.

Additionally, in the case of [Datadog](https://www.datadoghq.com/), the way some properties are written has changed, probably because the way Micrometer settings are written has changed overall. Previously, it was written as follows.

```yml
management:
  metrics:
    export:
      datadog:
        enabled: true
        apiKey: ${API_KEY}
        step: 1m
```

This will change as shown below.

```yml
management:
  datadog:
    metrics:
      export:
        enabled: true
        api-key: ${API_KEY}
        step: 1m
```

If you are using other Micrometer features, you should check whether the property setting method has changed compared to the existing Spring Boot 2.

## Jakarta EE

Spring Framework 6 has moved from Oracle's `Java EE` to Eclipse's `Jakarta EE`, so compilation may fail until imports are updated. The most common examples are probably servlet packages such as [HttpServletRequest](https://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpServletRequest.html) and validation packages such as [@Valid](https://docs.oracle.com/javaee/7/api/javax/validation/Valid.html). These can be handled simply by changing the package names in your imports. For example, suppose you are using the following packages:

- `javax.servlet`
- `javax.validation`

Change these as follows.

- `jakarta.servlet`
- `jakarta.validation`

This is an easy fix, as only the package name has changed and the contents have not changed.

## Springdoc

If you are using [springdoc-openapi](https://springdoc.org/) to generate API documentation, the existing version does not seem to be compatible with Spring Boot 3. CI parses the JSON obtained from `/v3/api-docs` and uploads the document to S3, but with Spring Boot 3, a 404 error occurred.

Fortunately, this can also be addressed by changing the dependencies. Previously, I added dependencies like this:

```kotlin
implementation("org.springdoc:springdoc-openapi-ui:1.6.14")
implementation("org.springdoc:springdoc-openapi-kotlin:1.6.14")
```

Just change this to [springdoc-openapi v2](https://springdoc.org/v2/). It also includes support for Kotlin, so it will work without any problems with just the following.

```kotlin
implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.0.2")
```

Now you can upload documents to S3 without getting 404 errors.

## Liquibase

Although it is not directly related to Spring Framework, I will also introduce this case because it was affected by upgrading the Spring Boot version. When using [Liquibase](https://www.liquibase.org/) as a Gradle plugin for DB migration, the following error may occur depending on the version.

```text
SLF4J: No SLF4J providers were found.
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See https://www.slf4j.org/codes.html#noProviders for further details.
SLF4J: Class path contains SLF4J bindings targeting slf4j-api versions 1.7.x or earlier.
SLF4J: Ignoring binding found at [jar:file:/root/.gradle/caches/modules-2/files-2.1/ch.qos.logback/logback-classic/1.2.11/4741689214e9d1e8408b206506cbe76d1c6a7d60/logback-classic-1.2.11.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See https://www.slf4j.org/codes.html#ignoredBindings for an explanation.
Exception in thread "main" java.lang.ClassCastException: class org.slf4j.helpers.NOPLogger cannot be cast to class ch.qos.logback.classic.Logger (org.slf4j.helpers.NOPLogger and ch.qos.logback.classic.Logger are in unnamed module of loader 'app')
  at liquibase.integration.commandline.Main.setupLogging(Main.java:233)
  at liquibase.integration.commandline.Main.run(Main.java:145)
  at liquibase.integration.commandline.Main.main(Main.java:129)
```

This is probably because the version of [SLF4J](https://www.slf4j.org/) has become 2 series since Spring Boot 3. Until now, the Liquibase plugin used [Logback](https://logback.qos.ch/index.html) to output the migration status, but since the version of this Logback was 1.2, it was not compatible with SLF4J 2 series. It seems that it is compatible with Logback 1.3, so if you are using an earlier version, you will need to update this version as well.

## Finally

I managed to successfully upgrade Java and Spring Boot, but as I mentioned earlier, I think this was only possible because the application I was developing was a microservice and had relatively few dependencies. If your application is monolithic or has more complex dependencies, you are likely to encounter problems in areas other than those listed here.

However, Spring Boot 2 support will eventually end, and at that point many teams will have no choice but to move to Spring Boot 3. By then there will probably be more migration examples available.

As for Java itself, I do not expect major issues when moving from 11 to 17, but migrating from 8 deserves more caution because of things like the [module system](https://www.oracle.com/webfolder/technetwork/jp/javamagazine/Java-JA18-LibrariesToModules.pdf) introduced in Java 9.

Fortunately, although I have not yet tried a native build, I was able to confirm that startup time improved even on the JVM. On local startup, the application took 31 seconds with Java 11 and Spring Boot 2, and 23 seconds with Java 17 and Spring Boot 3. That is not a rigorous benchmark, but it suggests there may be some overall performance improvement.

See you soon!
