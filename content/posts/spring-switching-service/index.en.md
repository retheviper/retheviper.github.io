---
title: "Select Service Impl class in YAML"
date: 2020-02-25
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - yaml
  - java
  - spring
---
When writing business logic in Spring, you generally create a class called Service. Service is an important class in development and testing because it performs important processing, but depending on the situation during development, it may not work even if it is implemented. For example, the environment is not yet set up, or the design is dependent on another class, but that class does not exist yet. In such cases, you may write a class that always returns the same result without performing any actual processing.

In such cases, it would be convenient if you could write multiple Service classes in advance and select which Service class to use depending on the situation, such as development mode or test mode, in an external configuration file (`application.yml`). In fact, the Service class is often written as an interface and an implementation class (named Impl), so it is not impossible to create multiple Impl classes and register different ones as beans depending on the case.

So this time, I will show you how to read the YAML settings and decide which ServieImpl to register as a bean depending on the case.

## Service's original configuration

The configuration of a general Service class is as follows. It means creating an interface and creating a service that embodies it. In some cases, I use DI to use classes that are in charge of linking with external elements of the application, such as the DB.

```java
public interface SomeService {

    public void doSomething();
}

@Service
public class SomeServiceImpl implements SomeService {

    private SomeRepository repository;

    @Autowired
    public SomeServiceImpl(SomeRepository repository) {
        this.repository = repository;
    }

    @Override
    public void doSomething() {
        // ...
    }
}
```

Now, suppose we have created the Impl class that we want to use during testing as follows.

```java
@Service
public class SomeTestServiceImpl implements SomeService {

    private SomeTestRepository testRepository;

    @Autowired
    public SomeServiceImpl(SomeTestRepository testRepository) {
        this.testRepository = testRepository;
    }

    @Override
    public void doSomething() {
        // ...
    }
}
```

In this case, when you start the application, Spring will ask you which implementation class of Interface? You will be asked. This is because there are two implementation classes for one Interface, and both are being registered as beans.

## Delete annotation

Generally, you will add `@Service` to the Service class. By adding this annotation, Spring will automatically register this class as a bean. Therefore, if you create a class with multiple `@Service` for one Interface, Spring will not know which one you want to use. Therefore, we will not use the `@Service` annotation here.

## Creating YAML

Creating YAML is easy. This time, we will use boolean to make it work in test mode (using SomeTestServiceImpl) if it is true, and in normal mode (using SomeServiceImpl) if it is false. For example:

```yml
settings:
  TestMode: true
```

You can write this custom property directly in application.yml, but since you are creating your own settings, you can also create a separate file with an appropriate name. If you create a separate file, don't forget to include it in application.yml.

```yml
spring:
  profile:
    include: settings # Use this when the file name is `application-settings.yml`
```

## Configuration settings

If you remove the annotation, ServiceImpl can no longer be registered as a bean. However, choosing the ServiceImpl class you want to use means that you want to register the class you want to use as a bean depending on the situation. Therefore, you need to select a class somewhere and register it as a bean. We also need to read the settings written in YAML. Let's implement these together as a class with `@Configuration`.

```java
@Configuration
public class SomeServiceConfig {

    // Read the values configured in YAML
    @Value("${settings.TestMode:false}")
    private boolean testMode;

    // Decide which implementation class to use from the YAML settings and register it as a bean
    @Bean
    public SomeService someService(SomeRepository repository, SomeTestRepository testRepository) {
        return this.testMode ? new SomeTestSerivce(testRepository) : new SomeServiceImple(repository);
    }
}
```

In Spring, you can read values specified in YAML by using `@Value` and `@ConfigurationProperties`. `@ConfigurationProperties` allows you to match YAML values ​​against fields of the entire class, but here we only want to read one value, so we will use `@Value`, which can be used for individual fields. An exception will occur if there is no YAML file, so we specified false as the default value.

To register a bean, simply add the `@Bean` annotation, create a new instance, and return it. In this example, the Repository class that SerivceImpl depends on is injected using Autowired in the constructor, so its instance is also required. If you write Repository as a method argument, if the class is registered as a bean, it will be automatically included as an argument. Therefore, depending on whether this is test mode or normal mode, pass the appropriate arguments to each constructor and return the instance to complete the bean registration. It's easy!

## When using annotations

If you want to switch beans depending on the situation, you can also use the `@Profile` annotation. This is also not difficult to do. First, suppose you define a YAML file as follows.

```yml
spring:
  profile:
    active: dev
```

Once you have defined the YAML, specify which profile to use using annotations. Like the code below.

```java
@Configuration
public class SomeServiceConfig {

    // Register this bean for dev or debug
    @Bean
    @Profile({"dev", "debug"})
    public SomeService someService(SomeTestRepository testRepository) {
        return new SomeTestSerivce(testRepository);
    }

    // Register this bean for prod
    @Bean
    @Profile("prod")
    public SomeService someService(SomeRepository repository) {
        return new SomeServiceImple(repository);
    }
}
```

The `@Profile` annotation allows you to specify the profiles that can be specified as an array, so you can easily specify what beans to register by writing YAML. Either method is fine, so choose the one that suits your situation.

## Finally

Lately, I've been wanting to try web frameworks in other languages, such as Django and Express, but I'm constantly discovering and learning new things every day, so it's hard for me to leave Spring behind. Every time I discover something that can be done this way or that I didn't know about, I wonder if Spring has been able to survive in the enterprise market for so long despite the fact that there are other good frameworks out there. There may be no shortage of articles to post on the blog for a while just in Spring!
