---
title: "Create multiple projects with Gradle"
date: 2019-10-20
translationKey: "posts/gradle-multi-project"
categories: 
  - gradle
image: "../../images/gradle.webp"
tags:
  - groovy
  - java
  - gradle
---
Originally, I was learning how to create Spring MVC projects using Maven, but recently I have been working with Spring Boot projects managed with Gradle. At first, I thought that Gradle was simply a tool for dependencies similar to Maven, but after trying various things such as directly creating tasks and building Jars, I realized that it's not that simple.

What I thought was the best part was being able to manage individual projects with different functions using the concept of multi-project. From an OOP perspective, I think it is possible to create applications that are easier to maintain by dividing them by project, rather than by dividing only classes and packages.

So, first of all, I would like to briefly explain what a Gradle multi-project is and how to create one.

## Gradle Multi Project

Gralde's multi-project, as the name suggests, refers to a project that is made up of multiple projects. Specifically, there is a root project that is the starting point for everything, and multiple Gradle projects are contained under it. Projects managed under the root project are called subprojects.

There seems to be a different way to create this kind of multi-project than the method introduced this time, and there also seems to be a difference in the folder structure of the project.

However, I believe that the purpose of multi-projects is to simplify the management of multiple projects, so we will adopt a format in which sub-projects are contained within the root project so that they can be managed in one repository unit. This is actually the configuration of a Spring Boot project I am writing at work.

Also, the explanation introduced here is based on Eclipse. (I'm not familiar with other IDEs yet...)

## Root Project

Create a root project that will contain all subprojects. First, create a general Gradle project. However, since we plan to use `build.gradle` in a subproject from now on, let's copy it. Delete files and folders that are no longer needed. The only files and folders left here are the following.

- gradle (folder)
- build.gradle
-gradlew
-gradlew.batAfter leaving the above files and folders, the next step is to modify the `build.gradle` file. The root project only manages subprojects, and necessary tasks, plugins, and dependency settings are written in the subprojects. So let's keep it as simple as possible.

```groovy
subprojects {
    repositories {
        mavenCentral()
    }
}
```

All we are doing here is specifying the subproject's repository, but you can use the same method to enter anything that must be commonly written in `build.gradle` of the subproject. For example, there may be plugins.

Next, create the `settings.gradle` file. This file links the root project and subprojects and determines the project name.

```groovy
include 'core'
include 'web'

rootProject.name = 'TestProject'
rootProject.children.each { it.name = rootProject.name + '-' + it.name }
```

The one written as `include` is a subproject, and `rootProject.name`, as the name suggests, means the root project name. There are other naming methods, but the root project name refers to the entire project, and subprojects are named as one functional unit of the project.

In `rootProject.children.each { it.name = rootProject.name + '-' + it.name }`, a dash is added to the root project name, followed by subproject names. This can also be expressed in the following way.

```groovy
rootProject.children.each { p -> p.name = "${rootProject.name}-${p.name}"}
```

With this nomenclature, each subproject is named `TestProject-core`, `TestProject-web`, etc.

If you look at the project configured in this way from Eclipse's Package Explorer, it will look like the following.

-TestProject
-TestProject-core
-TestProject-web

If you need to add a subproject later, by writing `include` in the root project, the subproject name will be automatically assigned as `TestProject-...`.

## Sub Project

After preparing the root project, the next step is to create subprojects for child elements. The subproject can be a normal Gradle project and can be used simply by placing it under the root project, but here we will show you how to create an empty folder.

First, create a folder to be used as a subproject under the root project. Since we plan to use the folder name as the subproject name, choose an appropriate folder name. Here, the folder name must match `include` written in `settings.gradle` of the root project. For example, in the root project above, we have already written `core` and `web`, so we will create folders for `core` and `web` accordingly.After creating the folder, place `build.gradle` that you copied when you created the root project in it and edit it. This time, we will generate a Java project based on Eclipse, so the following plugins are required.

```groovy
plugins {
    id 'eclipse'
}
```

When I save the file and run `gradlew tasks` from the root project, there is `eclipse` in the ide clause. Perform this task. On the command line, you can execute it by entering `gradlew eclipse`. Of course, you can run it on Eclipse, but it may take some time to update, so we recommend using the command line.

By performing this task, the created folder will be prepared to function as an Eclipse project. After that, just like a normal Java project, create a source folder (src/main/java), create a package, and fill in the subproject.

If your multi-project is not recognized correctly in Eclipse, right-click the project and refresh it, or run Gradle project refresh from the Gradle menu.

## Sub Project Dependency

In multi-projects created using this method, subprojects can also depend on each other. For example, if you need to load the class created in the `core` project to start the `web` project, you can import the `core` class by writing the following in `build.gralde` of `web`.

```groovy
dependencies {
    implementation project(':TestProject-core')
}
```

However, in this case, please note that the `core` project will be compiled once when `web` is started, so changes made on the `core` side will not be reflected immediately during testing.

## Finally

There is still no easy way to generate Gradle multi-projects on Eclipse, and it is inconvenient that projects can only be created using this somewhat inconvenient method. It may be added later...

But first of all, if you understand how to configure multi-projects and what their structure is, I think there will be more ways to utilize them in the future. Especially as the scale of the project grows, this type of management becomes necessary.

So everyone, please give Gradle's multi-project a try!
