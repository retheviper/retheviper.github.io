---
title: "Personal 2022 Roadmap"
date: 2022-01-11
translationKey: "posts/my-roadmap-in-2022"
categories: 
  - recent
image: "../../images/map.webp"
tags:
  - kotlin
  - swift
  - swiftui
  - compose
  - nextjs
  - nuxtjs
---
This year marks the fourth year since I started writing this blog. Looking back, when I first started writing this blog, I was mainly working as an SE on the infrastructure team of an SIer, so I had a lot of exposure to a variety of technologies, including infrastructure and libraries, but I changed jobs last year and started working as a full-fledged business engineer in charge of backend, so I think my interests and the technologies I was exposed to changed. So, this time I would like to look back and talk a little about this year's roadmap.

Looking back at past posts, at first most of them were about Java, Spring, Linux, Jenkins, etc., but recently I feel like my interest has shifted to the various frameworks that can be used with Kotlin. Basically, both Java and Kotlin are JVM languages, so there is no big difference in what they can do or in what fields they can be used in. Even after changing jobs, the framework I use is Spring, but JetBrains, the developer of Kotlin, develops a variety of frameworks, so I have naturally become interested in them.

## Overcoming Sophomore jinx

Sophomore jinx is a Japanese word that means "second year jinx." This term seems to refer to the fact that in the second year of university, students' grades drop and their enthusiasm wanes compared to when they were freshmen. Expanding the meaning of this, it now refers to cases in which a player who was selected as Rookie of the Year in his first year has deteriorated in some way from the beginning, such as when his performance worsens the following year, or when a sequel to a popular movie is not interesting.

In my case, this is my fourth year as an engineer, and I do feel like I'm less enthusiastic from the second year onwards than I was during the first year. That's not to say that I'm no longer interested in new technology or that I'm tired of programming itself, but the difference is that before, if I wanted to do something, I'd stay up all night writing code while staring at the monitor, but now I don't feel like doing that anymore. I think it's because I'm getting older,

Regarding this, I think we need to do some self-analysis, calmly organize what we want to do, and what we can do, so that we can produce some results little by little. As was the case last year, that is also why I am writing a post like this. I don't think I'll be able to accomplish everything I plan, but I feel like it's better to accomplish at least some of the many goals than to set fewer goals from the beginning. So, first of all, I think I'll look at various technologies with the feeling of ``I don't know if I'm going to do it, but I'll keep my antenna up.''

## FrontendI only learned the basics of JavaScript and TypeScript through training and Udemy courses, and I didn't do much front-end work. However, looking at the recent trends in web application development, it seems like it would be better to learn at least one front-end technology, and I don't even feel like they are absorbing the back-end role. Above all, it's hard for end users to imagine an app without a screen, so I think the time has come to create something that fully utilizes a GUI, not just the APIs, libraries, and command line apps that I've been using.

Above all, in the case of front-ends, a few years ago there were so many different libraries and frameworks that it was difficult to know which one to use, but recently, even though it was said to be the top three, only React and Vue, which had surpassed Angular, survived, and there are many frameworks based on them. I think we are now in a time when the technology can be said to have matured, such as with the advent of Mworks, so I think it's time to choose either [Next.js](https://nextjs.org) based on React or [Nuxt.js](https://nuxtjs.org) based on Vue. I myself would like to try one of them this year.

It might be a good idea to try out server-side frameworks for JavaScript like [NestJS](https://nestjs.com), but since I have a server-side background, I think it would be a good challenge to start with just the front end. As I will explain later, there are other things on the backend that I would like to touch on, so it's even more important.

## Quarkus

Personally, I have been using Spring for a long time, so I have a desire to try a new framework. Once you get used to one framework, I don't think it's a bad option to continue using it, but new technology has its advantages and disadvantages, so I think it's necessary to at least try it out. So last year I tried out two: [Quarkus](https://quarkus.io) and [Ktor](https://ktor.io).

Personally, I have confidence in JetBrains' products, and I think Ktor is not bad in terms of being compatible with Kotlin, but I'm hesitant because of the lack of functionality and some issues with the architecture. On the other hand, Quarkus can be compiled natively, some Spring libraries can be used as-is, and I feel like I can write code quickly and without much difference from Spring, so I'd like to try using it seriously. The biggest problem is that it takes a lot of time to build natively, but the only way to solve this may be to work well with CI.

## FastAPII'll be using Python right away, but I'd also like to try out [FastAPI](https://fastapi.tiangolo.com). I've always thought that even if I change jobs to a different field, there's a good chance I'll continue working on the backend, so I've wanted to try out different languages ​​and frameworks. Candidates include [Express](https://expressjs.com/ja), [Rocket](https://rocket.rs), and [Vapor](https://vapor.codes), and after trying them all, I decided to continue using the one that most suited my tastes for private use.

Meanwhile, I usually use Python for purposes such as creating simple automation scripts, so I was thinking of trying out [Django](https://www.djangoproject.com), but recently I heard that FastAPI allows for extremely fast development, so I became interested. You can still develop with Kotlin and Spring, but I feel like having one of these lightweight options wouldn't be a bad idea for a simple project. Since it is an interpreted language, it starts up quickly and shows surprisingly good performance in [Techempower's benchmark](https://www.techempower.com/benchmarks), which is also attractive, but the fact that the documentation is automatically done using [Swagger](https://github.com/swagger-api/swagger-ui) and [ReDoc](https://github.com/Redocly/redoc) seems to be quite good. It feels like you can make anything quickly.

Also, even if I don't use it directly, I've heard that the code is clean and educational, so I want to read it at least once.

## SwiftUI and Jetpack Compose

Personally, what I really wanted to do was create an app with a GUI. Even before I became an engineer, I created a desktop app using [JavaFX](https://openjfx.io). Nowadays, you can do anything with just JavaScript using [Electron](https://www.electronjs.org) or [React Native](https://reactnative.dev), but now that you can use Java and Kotlin, I think it would be better to try making a native app.

Shortly after [Flutter](https://flutter.dev) was announced, I once tried out a tutorial along with React Native, but even back then I was fascinated by Flutter's so-called "declarative GUI", so I was wondering if I was going to do mobile from now on, I would choose Flutter. It's even more so now that it's become possible to create not only mobile apps, but also desktop and web apps.

However, after the introduction of [SwiftUI](https://developer.apple.com/xcode/swiftui) and [Jetpack Compose](https://developer.android.com/jetpack/compose?hl=ja), I completely turned to this. I think Flutter has an advantage in terms of being able to develop multi-platform apps at the same time, but now that you can create a native UI in a similar way, I don't think you need to worry about it anymore.

In particular, with SwiftUI you can create desktop apps for Mac, but with Jetpack Compose you can create not only desktop apps but also web apps, and if you use [Kotlin Multiplatform Mobile](https://kotlinlang.org/docs/kmm-getting-started.html) you will be able to share business logic, so I feel that this is more suitable for my case. When I say it suits me, it's just that I'm a lazy person and want to solve everything with one language... Anyway, I think I'll try using it once, and if I like it, I might try using Jetpack Compose on the front end as well. There is also the option of [Kotlin/JS](https://kotlinlang.org/docs/js-overview.html), but I will leave that for another time.

However, the disadvantage of these is that they are still technologies that have not yet been perfected. This is a dilemma with any new technology, regardless of the field, but no matter how good the new technology is, there may come a time when it is not perfect (or extremely inconvenient), so for the time being, I'd like to wait and see how things go and try creating a simple app first. There is a good precedent with Flutter, so there is hope that new features will be quickly adopted.

## Oracle Cloud

Perhaps because it was a latecomer compared to other clouds, Oracle Clou is known for its unprecedented policy of providing two VM instances with 1GB of memory even on its free plan, but starting in 2021, it has added an option called ARM ([Ampere A1](https://www.oracle.com/cloud/compute/arm/)) to the VM instances you can choose from.What stands out about Ampere compared to instances that use existing AMD or Intel CPUs is its performance. Perhaps because we thought that ARM would have compatibility issues and the usage rate would be lower than x86, we offer instances with good performance with 4 OCPUs and 24GB of memory even in the free plan. Since you can create up to two free instances, you can also specify a configuration of two OCPUs and 12GB of memory between two instances.

Personally, I was disappointed that the existing free instances had an option of 1 OCPU and 1GB of memory, so they were not suitable for heavy work such as building Java applications. Therefore, I could only use it lightly, such as using it as a server for [Mattermost](https://mattermost.com), but I think that with the introduction of Ampere, I could also use it as a CI server. In addition to what is introduced on the Oracle Cloud homepage, if you look at other [benchmarks](https://jiuyu.medium.com/oracle-cloud-ampere-a1-cpu-benchmarks-6464ef43593d), it seems like you can expect good CPU performance. Oracle DB is also provided free of charge, so I feel like there are various options you can use, such as using that, using one VM instance for the DB, or using it in combination with free services from GCP and AWS.

However, I was still concerned about compatibility, and personally, after using a Mac equipped with Apple Silicon, I came to the conclusion that it's not as bad as I expected. Many programming languages ​​are already compatible with ARM, and if you want to use them on a server, you might use something like [FFmpeg](https://www.ffmpeg.org), [ImageMagick](https://imagemagick.org/index.php), or [GraphicsMagick](http://www.graphicsmagick.org), but since you can install or build the ARM version of any of them, there doesn't seem to be any problem.

If there are other problems, the problem is that even if you try to create a VM instance at the moment, the hardware is not enough, or you can't create one with specs higher than 2 OCPUs. It may be a problem that time will solve, but it is certainly a problem that you do not know when you will be able to freely create instances.

## Finally

Even though I say I don't have the motivation, the fact that there are so many things I want to try makes me think that I'm glad that I'm not dead as an engineer yet. Maybe I should correct myself by saying that I'm just a pain in the ass, rather than saying I'm not really motivated...

So, I've listed all the things I want to do, and this year I think my first goal is to create an app that can actually be used. Just trying to make something will definitely be a good experience and a good career.

See you soon!
