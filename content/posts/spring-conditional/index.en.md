---
title: "Use annotations that operate on conditions"
date: 2020-04-26
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - annotation
  - yaml
  - java
  - spring
---

Annotations can be used in regular Java, and are actively used in various libraries and frameworks. Among them, I think Spring makes the most effective use of annotations. Whether it's for DI or class positioning, everything can be easily defined by adding annotations. Therefore, when implementing a web application with Spring, I think it is important to find out what kind of annotations are available.

The reason I am bringing this up is that in [the previous post](../spring-switching-service/), I explained how to switch services. While looking for other approaches, I found that Spring also lets you register beans and configuration conditionally through annotations.

Also, there were various annotations depending on the conditions, so in order to study them, I would like to introduce Spring annotations that were created with various conditions and cases in mind. These annotations seem to belong to Spring's auto-configuration. Spring Boot already automates many settings, but it provides annotations so that you can customize and use them.

*I haven't tried all of these, but I'll just introduce them for now.

## DI with conditions

I'll start with the coat I introduced in a previous post. I said that if you do something like the following, you can decide which repository to DI based on the values ​​written in application.yml.

```java
@Configuration
public class SomeServiceConfig {

    // Read the value configured in YAML
    @Value("${settings.TestMode:false}")
    private boolean testMode;

    // Register a bean by deciding which implementation class to use from YAML
    @Bean
    public SomeService someService(SomeRepository repository, SomeTestRepository testRepository) {
        return this.testMode ? new SomeTestSerivce(testRepository) : new SomeServiceImple(repository);
    }
}
```

If you were to implement something similar using the Spring annotations that we will introduce later, it would change as follows.

```java
@Configuration
public class SomeServiceConfig {

    // Service class for production
    @Bean
    @ConditionalOnProperty(value = "settings.Testmode", havingValue = "false")
    public SomeService someService(SomeRepository repository) {
        return new SomeServiceImple(repository);
    }

    // Service class for testing
    @Bean
    @ConditionalOnProperty(value = "settings.Testmode", havingValue = "true")
    public SomeService someTestService(SomeTestRepository testRepository) {
        return new SomeTestService();
    }
}
```

As you can see from the code above, using the annotation `@ConditionalOnProperty` allows you to execute the contents of a method under certain conditions. The code will be much cleaner than writing branches or preparing custom `@Value` annotations.

Also, this annotation cannot only be combined with Bean annotations. It can be used with other Spring class annotations (Configuration, Component, Service, Repository, Controller), so it has more flexibility.

There are several other annotations such as `@Conditional...` in Spring Boot's org.springframework.boot.autoconfigure.condition package, and I would like to briefly introduce them.

## Predefined Conditional annotation

## ConditionalOnProperty

This is an annotation that will be executed if the specified annotation is found in the file where the property is written, such as application.yml, or in the system property. It seems to be the most commonly used in Spring Boot applications. Inside the parentheses, you can specify the property name and value, and whether to execute even if the property does not exist.

The code below is an example of a Configuration class that will be executed if the property use.testmode exists and is true. If matchIfMissing is set to true, it will be executed even if the property use.testmode does not exist. Of course, if not specified, the default value will be false.

```java
@Configuration
@ConditionalOnProperty(value = "use.testmode", havingValue = "true", matchIfMissing = true)
public class TestConfig {
  // Configuration used in test mode
}
```

## ConditionalOnExpression

This annotation is execution based on the property description method. Conditions can be specified in parentheses. The condition (expression) here uses [SpEL](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions), which is also used in Value annotations. It will be executed if the properties listed in application.yml etc. match the conditions in parentheses. The code below is an example that will be executed when both use.testmode and use.submode are true.

```java
@Configuration
@ConditionalOnExpression("${use.testmode:true} and ${use.submode:true}")
public class TestSubConfig {
  // Configuration used when both test mode and sub-mode are true
}
```

## ConditionalOnBean

This is an annotation that executes if the specified bean exists. To use it, just specify the class you want to check whether it is registered as a bean in the parentheses. I think it can be used when a certain bean is registered using ConditionalOnProperty, and you also want to register necessary submodules. The code below is an example that is executed when a class called TestRepository is registered as a bean.

```java
@Configuration
@ConditionalOnBean(TestRepository.class)
public class TestBeanConfig {
  // Configuration used when a test bean has been registered
}
```

## ConditionalOnMissingBean

It is the opposite of ConditionalOnBean. This will be executed if the specified class is not registered as a bean. The code below is an example of automatically registering TestRepository as a bean if Repository is not registered as a bean.

```java
@Configuration
public class AlternativeConfigutration {

    @Bean
    @ConditionalOnMissingBean
    public Repository testRepository() {
        return new TestRepository();
    }
}
```

## ConditionalOnResource

This annotation is executed if the resource file exists. For example, when using LogBack, an xml file is required, but if that xml file exists, you can use it if you want to have a configuration where Configuration is also executed. The code below is an example of such a case.

```java
@Configuration
@ConditionalOnResource(resources = "/logback.xml")
public class LogbackConfig {
  // Configuration used when logback.xml exists in the resources folder
}
```

## ConditionalOnClass

This annotation is executed if the specified class exists. It is similar to ConditionalOnBean, but the difference is that it does not need to be a bean. For example, I think it can be used depending on whether there is a certain library as a dependency. The code below is an example that will be executed if a class called SomeClass in the com.custom.library.module package exists.

```java
@Configuration
@ConditionalOnClass(name = "com.custom.library.module.SomeClass")
public class CustomLibraryConfig {
  // Configuration used when SomeClass from a custom library exists
}
```

## ConditionalOnMissingClass

This annotation is the opposite of ConditionalOnClass. This is similar to ConditionalOnMissingBean. Similarly, the specified class does not have to be a bean. The code below is an example of the opposite case of ConditionalOnClass above.

```java
@Configuration
@ConditionalOnMissingClass(name = "com.custom.library.module.SomeClass")
public class NoCustomLibraryConfig {
  // Configuration used when SomeClass from a custom library does not exist
}
```

## ConditionalOnJava

This is an annotation based on which version of Java the application is running on. The API specifications may change depending on the Java version, so you may consider using it if you need to run your application in multiple environments. The code below is an example when Java version is 1.8.

```java
@Configuration
@ConditionalOnJava(JavaVersion.EIGHT)
public class JavaEightConfig {
  // Configuration used when the Java version is 1.8
}
```

## Custom Condition

You can also create a custom Condition by implementing the Condition interface. For example, as shown in the code below, you can create your own Condition when the OS on which the application is executed is Linux.

It's easy to use, just implement matches whose return value is boolean.

```java
class OnUnixCondition implements Condition {

    // Condition that determines whether the OS is Linux
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return SystemUtils.IS_OS_LINUX;
    }
}
```

The implemented Condition is used by specifying it in the Conditional annotation.

```java
@Configuration
public class OsConfig {

    // The bean is registered on Linux
    @Bean
    @Conditional(OnUnixCondition.class)
        public UnixBean unixBean() {
        return new UnixBean();
    }
}
```

By inheriting the AnyNestCondition class, you can specify more complex conditions. Let's say that in addition to the OnUnixCondition implemented above, we also implemented an OnWindwosCondition class that determines whether or not it is running on Windows. In such a case, you can implement it as follows.

```java
public class OnWindowsOrUnixCondition extends AnyNestedCondition {

    OnWindowsOrUnixCondition() {
        super(ConfigurationPhase.REGISTER_BEAN);
    }

    @Conditional(OnWindowsCondition.class)
    public static class OnWindows {
        // Windows case
    }

    @Conditional(OnUnixCondition.class)
    public static class OnUnix {
        // Linux case
    }
}
```

The implemented Condition is also specified in the Conditional annotation in the same way.

```java
@Configuration
public class OsConfig {

    // The bean is registered on either Windows or Linux
    @Bean
    @Conditional(OnWindowsOrUnixCondition.class)
    public WindowsOrUnixBean windowsOrUnixBean() {
        return new WindowsOrUnixBean();
    }
}
```

## lastly

In addition to the annotations introduced here, there are various classes under the org.springframework.boot.autoconfigure.condition package. For example, there are annotations for whether it is a web application or not, and whether it is a cloud platform, and various conditions may be added later.

It seems that these Conditional annotations are also used in Spring Boot's Auto Configuration. I think this is a more stable method than reading the properties directly and writing an if statement as I introduced before. Also, there are annotations that correspond to various conditions, and I think there are some parts that can be shared by creating custom conditions, which is useful in many ways.

As with Java itself, I felt that the world of Spring still had a lot of depth to it. I will continue to study.
