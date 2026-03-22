---
title: "A story about the module problem"
date: 2019-09-08
translationKey: "posts/java-conflict-of-module"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - jigsaw
  - module
  - java
---
Java updates are quick these days. The first one I learned was 1.8, but 9 soon came out and now 13 is about to be released. Version updates have many positive aspects such as bug fixes and performance improvements, so I always want to keep the programs I use up to date, but every time a language version is updated, it's not a simple matter to check what has changed and review the existing code.

Java has a long history, so when you compare it with current trends, there are many aspects that are inconvenient (mostly because the paradigm has changed). And 1.8 was maintained for a long time, which is why it has fallen behind the trend. In some ways, it seems to be following trends, such as the introduction of type inference in version 10, but compared to other languages ​​that also use the JVM, like Kotlin, I feel like it still has a long way to go.

Of course, the changes are positive, and the ability to write in a style that matches trends while maintaining the original characteristics can also be seen as an expansion of the pool of users who can use the language. However, it seems that "coexistence of old and new" is not possible in all elements. In that case, you need to choose which one to use.

The Module I want to talk about in this post is a typical example. Although it was introduced to improve an old problem, it ends up impacting existing code and needs to be addressed. At first, I thought that I didn't need to take this into consideration in the code I was writing, but that didn't seem to be the case. So here I would like to explain what Java Module is and what problems I encountered.

## Project Jigsaw

The module is called Project Jigsaw, and seems to have been considered for introduction since version 1.7. As you can see from the name Module, it is a system that allows you to select the library (built-in Java) to be loaded when starting an application. Until version 1.8, even if you created an application that started from the command line, basic system libraries such as Swing were included, but now you can adjust them. Removing unnecessary system libraries reduces the size of your application and has the benefit of saving memory. In addition, the problem of ``it takes a long time to load completely'', which was a characteristic of Java, can now be resolved to some extent by setting this module.In addition, Module seems to be able to solve the problem of packages being too public. Java classes can be accessed in the same package by using a Protected declaration, but if the packages are divided into smaller parts, they cannot be accessed even within the same library. In that case, the only option was to declare it as Public. Classes declared Public can be accessed anywhere, not just in the library, which can cause problems. While creating the library, it was difficult to separate the classes that we wanted our clients to use and the classes that we didn't want them to use. This can now be divided into classes that are exposed externally and classes that are exposed internally within the library using the Module settings.

## Module example

Now, I will explain with code how I was able to solve the Public problem using Module. I haven't actively used Module yet, so I can only cover the basics, but the three important points are as follows.

-Name
-Exports
-Requires

First, Name means the name of the Module itself. Write it using the same naming convention as the package name. Next, Exports refers to packages that are published outside of this Module. Even if the module is Public, packages that are not explicitly marked as Exports cannot be accessed from the outside. And finally, Requires represents dependencies on other modules.

If you write these in actual code, it will look like this: Described as `module-info.java` in the default package. (You can check from the system library after Java9)

```java
// How to write module-info.java
module com.module.mylibrary {
    exports com.module.mylibrary.api;
    requires com.module.exlibrary;
}
```

For Exports, you can specify the publication target. This means that you can specify which Mobiles you can access.

```java
// Public setting limited to exlibrary
module com.module.mylibrary {
    exports com.module.mylibrary.api
    to com.module.exlibrary;
}
```

It can be used not only for modules but also for external libraries. There is also a way to create `module-info.java`, but it is highly likely that it does not exist for libraries created before Java 9. If you need to include a library that is not a module like this, you can use the library separately using either `Automatic Module` or `Unnamed Module`. Both are the same in that they are automatically treated as Modules, and they are the same in that they can access all packages, but the former belongs to `modulepath` and has a name (this is the Jar file name), while the latter belongs to `classpath` and has no name, so it cannot be specified in Requires.

## Where I got hooked on ModuleThe problem I had with Module was due to a conflict between two libraries with the same package. The problem occurred because I was trying to add a Gradle task to an existing project. You can create a Gradle task by directly creating a task in `build.gradle`, but the method I was initially referring to (following Gradle's official documentation) was to include a plugin called `java-gradle-plugin`. This will automatically add Java libraries and allow you to write plugins in Java, but the libraries included here conflicted with the Java system libraries.

In the original project (using Java 11), `javax.xml` was imported, which became `Deprecated` from Java 9, and was finally removed from Java 11. It seems that it was loaded as `Unnnamed Module` on Eclipse, and a conflict occurred because the `java-gradle-plugin` package also included a package with the same name. It's strange that there would be a conflict since it was treated as removed in the first place...but the error message outputted `The package javax.xml.transform is accessible from more than one module: <unnamed>, javax.xml`.

Referring to [Similar Case](https://stackoverflow.com/questions/51094274/eclipse-cant-find-xml-related-classes-after-switching-build-path-to-jdk-10), two solutions were proposed, but neither could be used in my project. When `module-info.java` was created, it became a multi-project, and a complicated procedure was required to take into account the package dependencies between subprojects. Also, when the system library `javax.xml` was removed from the Eclipse module dependency settings, there was a problem that the other imported `java.sql` depended on `javax.xml` and could no longer be used.

When I read the document linked above, it said that this problem, while not exactly identical to my case, still had not been resolved even in Java 13. That left me with very little I could do. Since `java-gradle-plugin` is a library managed by Gradle itself, it was not something I could safely patch on my own side.

## What should I do in the end?

At this time, there seems to be no way to avoid conflicts while keeping the external libraries. I think it's because I don't have enough understanding of Modules, but in the end, when a situation like this occurs, the only option seems to be to exclude the library that causes the conflict as much as possible. It's not uncommon for a new feature introduced for convenience to cause unexpected problems, but after thinking about it for about three days, my only choice was not to use the library.

Of course, since it is a module issue, if you are not particular about the version, there is also a method to lower Java to 1.8. However, support for 1.8 should end at some point, and Java versions will continue to improve, so this is a problem that we may face someday. I hope that such problems will not occur in Java 14.
