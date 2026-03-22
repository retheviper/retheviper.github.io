---
title: "KotlinConf 2024 Recap"
date: 2024-05-25
translationKey: "posts/kotlin-conf-2024"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - kotlinconf
  - compose
  - ios
---

This year's KotlinConf has taken place. Since it is a multi-day event, I could not watch every session, so I summarized the main points from the keynote. You can check the full schedule [here](https://kotlinconf.com/schedule/), and there is also a [live stream on YouTube](https://www.youtube.com/@Kotlin/streams) if you want to watch it yourself.

First, the official KotlinConf'24 app was introduced. The app lets you browse the conference information and session schedule, and its [source code is available on GitHub](https://github.com/JetBrains/kotlinconf-app). It was built with Kotlin Multiplatform and runs on iOS, Android, and the web, so it looks like a useful reference project.

## Kotlin now

Let's start with the current state of Kotlin. The number of engineers using Kotlin on a regular basis is said to be over 2 million.

![Number of Kotlin engineers](kotlin-usage.webp)

Additionally, the number of companies that are implementing Kotlin is increasing rapidly. Representative companies are as shown in the image below.

![Companies adopting Kotlin](kotlin-usage-companies.webp)

## Kotlin 2.0 and K2 Compiler

The main theme is that from Kotlin 2.0, the introduction of the K2 compiler has improved overall compilation speed. However, some projects may experience delays. Intellij also has a K2 compiler mode, which is said to make code highlighting 1.8x faster.

![K2 compiler speed comparison](k2-mode-performance.webp)

K2 mode can be set on the following settings screen from Intellij 2024.1 or higher. From 2024.2, it will be provided as a beta version, including performance improvements.

![IntelliJ K2 mode settings screen](intellij-k2-mode.webp)

Migration to 2.0 has been tested with 10 million lines of code, 18,000 engineers, and 80,000 projects, and migration from 1.9 is said to be smooth.

## For Meta

In the case of Meta, they said they are actively developing with Kotlin first. The company uses Kotlin in everything from the IDE to code optimization, and has created a tool that can automatically convert existing Java apps such as Facebook, Instagram, Facebook Messenger, and WhatsApp to Kotlin to automate the migration.

![Meta adopting Kotlin](meta-kotlin.webp)

Meta has already adopted the K2 compiler in some projects since Kotlin 1.8, and now 95% of projects are using the K2 compiler. As a result, it seems that the build time for Android projects can be reduced by up to 20%.

![Meta adopting the K2 compiler](meta-k2-compiler.webp)

## For Google

Google is also collaborating with the K2 compiler, contributing to Android tooling such as Lint, [Parcelize](https://developer.android.com/kotlin/parcelize?hl=ja), and [KSP](https://kotlinlang.org/docs/ksp-overview.html). There has also been an improvement to the Jetpack Compose compiler, and previously there was a difference in version with Kotlin, but from 2.0 it is now possible to specify the same version.

![Compose compiler version settings](compose-compiler-update.webp)

Additionally, Android Studio is also planning to support Kotlin 2.0. Typical functions are shown in the image below.

![Android Kotlin 2.0 support](android-kotlin-2.0.webp)

It seems that new features will be added to Jetpack Compose as well. The main features listed in the image below are scheduled to be available from July.

![Upcoming Jetpack Compose features](jetpack-compose-upcoming.webp)

There have also been improvements to the compiler itself, resulting in improved stability and performance.

![Jetpack Compose performance improvements](jetpack-compose-performance.webp)

In addition, the adoption of server-side Kotlin is progressing within Google, and development using KMP is also progressing. Also, the Jetpack Compose library is also Multiplatform compatible, and libraries such as ViewModel and Room can now be used with Kotlin Multiplatform.

![Jetpack Compose Multiplatform support](jetpack-compose-multiplatform.webp)

## Kotlin Multiplatform

With the introduction of the K2 compiler, it is now possible to directly convert Kotlin code to Swift code. As a result, you can now use Kotlin Multiplatform to develop iOS apps.

![Kotlin to Swift conversion](kotlin-to-swift.webp)

It was also introduced that [Fleet](https://www.jetbrains.com/fleet/) can also be used to develop iOS applications. With Fleet, it is possible to simultaneously develop iOS and Android apps using Compose Multiplatform, and it is possible to develop apps consistently from refactoring to debugging. This is great news since support for AppCode was scheduled to end in 2023.

![Multiplatform development in Fleet](fleet-multiplatform-development.webp)

A new build tool, [Amper](https://github.com/JetBrains/amper), was also introduced. Although it was only announced recently, it is already supported by JetBrains IDEs, and you can configure builds using just a YAML file, so it may be worth trying in a new project.

![Introduction to Amper](amper.webp)

It seems that the following new features have been added to Compose Multiplatform. I'm happy because all of the features were what I was looking for. Personally, when I implemented a file selection dialog in a desktop app, I had to use Java's AWT because there was no corresponding function, so I think it would be nice to have an API like this.

![New Compose Multiplatform features](compose-multiplatform.webp)

## Upcoming

Next, there was an introduction to the features planned to be introduced in the beta version of Kotlin 2.1, as well as announcements of new libraries and AI models. As shown in the image below.

## Guard

This is a function to prevent duplication of variables in when branches. With existing code, even if `status` were duplicated in the code below, there was nothing I could do about it.

```kotlin
fun render(status: Status): String =
    when {
        status == Status.Loading -> "Loading"

        status is Status.OK && status.data.isEmpty() -> "No data"
        status is Status.OK -> status.data.joinToString()

        status is Status.Error && status.isCritical -> "Critical problem"
        else -> "Unknown problem"
    }
```

This is because if you rewrite the code as shown below, the `&&` conditions cause a compile error.

```kotlin
fun render(status: Status): String =
    when (status) {
        Status.Loading -> "Loading"

        is Status.OK && status.data.isEmpty() -> "No data" // Error: expecting '->'
        is Status.OK -> status.data.joinToString()

        is Status.Error && status.isCritical -> "Critical problem" // Error: expecting '->'
        else -> "Unknown problem"
    }
```

In order to improve this, they are planning to introduce `Guarded Condition` as shown below.

![Introduction to Guard](kotlin-guard.webp)

## $-escaping problem

`$` is used in Kotlin to embed variables within strings. That also means that if you want to use `$` as a string, there will be escaping issues. This is especially true for Multi-line String. For example:

```kotlin
val jsonSchema: String = """
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "$id": "https://example.com/product.schema.json",
    "$dynamicAnchor": "meta",
    "title": "Product",
    "type": "object"
}"""
```

Here, you may want to use `schema`, `id`, etc. as a string rather than a variable, but in the case of Multi-line String, escaping is not possible. So, in some cases, the code would look like this:

```kotlin
val jsonSchema: String = """
    "${'$'}schema": "https://json-schema.org/draft/2020-12/schema",
    "${'$'}id": "https://example.com/product.schema.json",
    "${'$'}dynamicAnchor": "meta",
    "title": "Product",
    "type": "object"
}"""
```

To solve this, a feature is planned that lets you write `$$` in a string literal so that `$` itself is treated literally without resorting to awkward escaping patterns.

![$-escaping problem solved](dollar-escape.webp)

## Non-local break/continue

Until now, `break` and `continue` could not be used because the compiler could not determine where Lambda was executed. Therefore, code like the following will result in an error.

```kotlin
for (i in 0..n) {
    val date = state[i]?.let {
        when (it) {
            -1 -> break // Error: 'break' is not allowed here
            0 -> continue // Error: 'continue' is not allowed here
            else -> process(it)
        }
    }
}
```

Also, starting with Kotlin 2.1, inline functions like `let` will be parsed correctly, so `break` and `continue` will be able to be used.

![Non-local break/continue solved](non-local-break-and-continue.webp)

## Contexts

There was also an announcement regarding `Context`, which was announced last year. This has already been introduced as a preview and will be available as a beta version starting with Kotlin 2.2. Now, if you want to do something similar to DI or need to reuse it in various functions such as sessions and transactions, you can use `Context` to share it without having to pass it as a function argument.

![Introduction to Context](contexts.webp)

## Core Libraries

Kotlin's core library will also be improved. There were announcements of new libraries like [kotlinx.rpc](https://github.com/Kotlin/kotlinx-rpc) as well as things like [kotlinx.io](https://github.com/Kotlin/kotlinx-io) that have already been announced. These core libraries are provided to support Multiplatform development and can be used on any platform.

![Introduction to Core Libraries](core-libraries.webp)

## AWS SDK for Kotlin

There was also an announcement that an AWS SDK for Kotlin will also be provided. Until now, Java SDKs have often been used, but an SDK will be provided that can be used on Multiplatform while taking advantage of Kotlin's features such as Coroutine and Null Safety.

![Introduction to AWS SDK for Kotlin](aws-kotlin-sdk.webp)

## Kotlin Language Model

The Kotlin language model is already available in Fleet and will be available in Intellij starting in 2024.2. Compared to some existing models, it has a relatively small number of parameters, perhaps because it is specialized for Kotlin, and has shown high accuracy in benchmarks. However, in the case of Llama used for comparison, version 3 has already been released, so I am curious about how accurate it is when compared with the latest model.

![Introduction to the Kotlin Language Model](kotlin-language-model.webp)

## Finally

The above is a summary of the presentations at the KotlinConf'24 keynote. There are many other sessions, and if you look at Kotlin 2.0 and [Changelog](https://github.com/JetBrains/kotlin/releases/tag/v2.0.0), there are many changes, so I'd like to check them out again.

Compose's development has been good, but there are other frameworks like Flutter as well, so I'm looking forward to seeing how much market share it can gain. Other languages ​​are developing quickly as a language, so I hope Kotlin will do its best to keep up.

See you soon!
