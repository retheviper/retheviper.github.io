---
title: "I want to use beans in a new instance"
date: 2019-12-22
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - dependency injection
  - java
  - spring
---
For general Java projects, if you want to write an external configuration file (YAML) and read its values, you can do it like my [previous post](../java-yaml-for-configuration/). However, this time I started doing something similar as a Spring project. Spring has its own specifications and usage when reading YAML. The YAML value read in this way can be set in the bean, and has the advantage that it can be called using `@Autowired` anywhere in the application.

However, although DI is such a convenient method, there are some difficulties in its usage. For example, there is a problem that `@Autowired` cannot be used among the instances that are normally used by new. I was really into it this time as well, but I wanted to create an object with the builder and use a bean obtained from YAML for values ​​not specified by the user. However, with Builder, I was unable to load beans because it created a new instance, so I got really hooked.

As a result, I found out that it is possible to obtain a bean without `@Autowired` by using a different method, so in this post I would like to explain the process leading up to it with code. We will show you how to create YAML and obtain and use beans within a new instance.

## Create a bean from YAML

In Spring, you can specify to load a specific YAML by writing the following in application.yml.

```yaml
spring:
  profiles:
    active: buildingdefault
```

Prepare a custom YAML file using what is described in active here. `application-` is included as the prefix of the file name. So the file name this time will be `application-buildingdefault.yml`.

Create a file and write the items and values ​​as shown below.

```yaml
settings:
  material: "cement"
```

Place the created YAML file in src/main/resource. Now we will create a class to load YAML with Spring.

There are two ways to read YAML and create beans with Spring. The first step is to annotate the field and associate it with the YAML item.

```java
@Getter
@Component
public class DefaultSettings {

    @Value("${settings.material}")
    private String material;
}
```

Add `@Value` to the field and enter the YAML item name as the annotation argument. By doing this, the value read from YAML will be imported into the bean in String format. The field does not necessarily have to be a String; it supports primitive types such as int and double as well as ENUM. If you write Locale as ja_JP etc. in YAML, it will be imported properly.

Another way to turn YAML values ​​into beans is to annotate the class instead of the field. If you specify prefix in the argument of `@ConfigurationProperties` as shown below, all items under the specified item will be subject to field mapping.

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "settings")
public class DefaultSettings {

    private String material;
}
```

## When you want to load multiple settings from YAML

When reading configuration values from YAML, you may want to write multiple settings and use them according to the situation. Of course, you can write it as an array in YAML, and you can also write it as a List when reading it with Spring. So I will explain how to make multiple settings into beans.

In YAML, write it as follows.

```yaml
settings:
  - preset-name: "default"
    material: "cement"
  - preset-name: "cabin"
    material: "wood"
```

Here, preset-name is a key to distinguish each setting set when actually using settings in Java. There is no problem in reading the values ​​without them, but if you name them like this, it will be easier to tell which is which later.

After writing the YAML, create a Bean class for each setting set.

```java
@Data
public class Material {

    private String presetName;

    private String material;
}
```

Finally, create a class that loads YAML. By writing a List of beans as a field in this class, you can ensure that these settings are loaded as soon as the Spring application starts.

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "settings")
public class MultiSettings {

    private List<Material> presets;
}
```

Later, you will be able to easily use it by simply obtaining a List from this class and searching for each setting value by presetName.

## Using Bean from Builder (Failure example)

With the settings so far, you will be able to use DI beans in a normal Spring application. However, this post is to explain how to obtain beans without DI, so I will explain the process.

The first thing I wanted to do, as I mentioned earlier, was to use a bean that reads YAML values ​​in the builder. The values ​​written in YAML here will be used as default values, and I would like to overwrite some items when building () as necessary. The code I tried first and it didn't work is as follows.

```java
public class Building {

    public static BuildingBuilder() {
        return new BuildingBuilder();
    }

    public static class BuildingBuilder {

        // DI is not possible
        @Autowired
        private DefaultSettings settings;

        private String material;

        public BuildingBuilder() {
            this.material = this.settings.getMaterial();
        }

        ...
    }
}
```

When using a builder, the first thing you have to do is create a new builder instance. And even if it is written as `@Autowired` in the new instance, DI cannot be done properly. If you actually write the code above, you will see that the bean's fields are null.

So, forget about DI and use a method that allows you to obtain beans within the new instance.

## Create ApplicationContextProviderApplicationContext is an interface that is responsible for various functions in Spring, such as creating beans and setting relationships between objects. The important thing here is that ApplicationContext generates and manages the beans that are registered in advance when starting a Spring application. In other words, if you can access this interface, you can retrieve the bean.

However, ApplicationContext itself is just an interface, so in order to obtain an instance, you need to create a class that plays that role. You can get an instance using the code below.

```java
@Component
public class ApplicationContextProvider implements ApplicationContextAware {
     
    private static ApplicationContext context = null;
 
    public static ApplicationContext getApplicationContext() {
        return this.context;
    }
 
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        this.context = context;
    }
}
```

The structure is simple: the field is ApplicationContext, and there are getters and setters for it. It's strange that this works, but when the Spring application runs, an instance of ApplicationContext is automatically set to the field through the setter. However, if you create an instance of this class, you will no longer be able to use it, so make the fields and getters static.

## Using beans from Builder (successful example)

Now that we can get an instance of ApplicationContext, let's modify the Builder.

```java
public class Building {

    public static BuildingBuilder() {
        return new BuildingBuilder();
    }

    public static class BuildingBuilder {

        private DefaultSettings settings;

        public BuildingBuilder() {
            this.settings = ApplicationContextProvider.getApplicationContext().getBean(DefaultSettings.class);
        }

        ...
    }
}
```

Get the ApplicationContext from the ApplicationContextProvider class you created earlier, and then call getBean(). By passing the class of the bean you want to get as an argument to getBean(), you can get the instance of that bean. Of course, you can also write it in the field itself instead of the constructor. If you do that, it will look like this:

```java
public class Building {

    public static BuildingBuilder() {
        return new BuildingBuilder();
    }

    public static class BuildingBuilder {

        private DefaultSettings settings = ApplicationContextProvider.getApplicationContext().getBean(DefaultSettings.class);

        public BuildingBuilder() {
        }

        ...
    }
}
```

When you run the modified code, you can confirm that the bean field is not Null but contains the value read from YAML.

## Finally

I think I failed this time because while I was using Spring, I was shamefully unaware of what was actually happening inside the application. I wouldn't have gotten as hooked as I did if I hadn't not only made sure it worked, but also properly understood the characteristics of the language and framework I was using. So while I gained new knowledge, I also had to reflect on myself. From now on, I need to understand exactly how and why the things I use work before using them.
