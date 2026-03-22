---
title: "Talking about the future of Kotlin"
date: 2022-05-22
translationKey: "posts/kotlin-prospect"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - go
  - rust
  - python
  - javascript
  - dart
---

After working with server-side Kotlin for about a year, I suddenly started thinking, ``How far has Kotlin come now, and what will it do in the future?'' I'm sure there are various points of view, but I wanted to think about what I feel about the so-called "status" of the language kotlin, such as how much demand there is in the market and what the prospects are (will it continue to be actively adopted, is it likely to struggle, etc.).

Looking at recent trends, it seems that there is a demand for so-called `Polyglot` programmers who can use various languages ​​rather than being experts in one language, and it seems that it has become common knowledge. It is true that I personally encounter many cases where I come into contact with various languages ​​depending on the project, regardless of whether I have the experience or not. Nowadays, the Internet is full of comprehensive documents and articles, and there are various sites such as Udemy and Coursera that offer high-quality lectures, so even if you have no experience with a language, it feels like we are living in an era where you can no longer say that it is difficult to get started. Therefore, it may be said that the question of whether the language you currently use belongs to the mainstream is less important than it used to be.

However, depending on your position or point of view, you may want to concentrate on one language. For example, it would be difficult to ask a student or someone with no experience to suddenly become proficient in two or more languages ​​to become an engineer. The same goes for the technology pursued by engineers. There should be no need for front-end engineers to suddenly learn languages ​​used in the back-end, such as Go or Java, which they don't plan to use right away. And as a company, looking for engineers who can handle multiple languages ​​must be a very strict hiring condition. Therefore, I believe that the status of a language in the market cannot be ignored.

So, this time I will talk from a somewhat subjective perspective, but I would like to talk about my thoughts on the prospects of the language Kotlin in other languages ​​and fields. Well then, here you go.

## vs Java

## Better java concept

Comparing Kotlin (JVM) to Java, I think the general opinion of Kotlin is that it is "perfectly compatible with Java, and the performance is the same" because the compilation result generates bytecode. Furthermore, it has various functions such as extension functions, coroutines, scope functions, and null safety, so on the surface, it seems like it could be read as `better java`. Additionally, if the JVM is improved with a Java version update, that will eventually lead to improvements in Kotlin. Since Java 1.8, version updates have become faster due to the release policy once every six months, but from an application engineer's perspective, I think there are still some drawbacks compared to Kotlin.

So far, Kotlin sounds like a perfect language to replace Java. In other words, from now on there is no reason to use Java at all, and the only option is to move everything to Kotlin. But what about the industry?

First, let's consider the history of Java. Java has long been the ``world's most used language'' and remains one of the top 5 most popular languages ​​even as other languages ​​gain popularity.[^1] And this suggests that it is not just about its current popularity, that is, the possibility that it will continue to be used, but also that the number of times it has been used is overwhelmingly high. Many systems and applications that have been created to date are based on Java, so unless there is something special, you will likely need Java engineers for maintenance and functionality expansion.

There is also this aspect. Kotlin is not the only language that has emerged with the concept of being able to write more advanced code while taking advantage of the benefits of Java as a JVM language. Until now, various languages ​​such as Clojure, Scala, and Groovy have appeared, and although each language has secured and expanded its own demand and field, I think that the current situation is that none of them has received the reputation of ``exceeding Java.'' Similarly, in the case of Kotlin, it cannot be said that its position is significantly different from other JVM languages. Therefore, I think that the characteristics of ``It's a JVM language'' and ``It's more modern than Java'' are at least evidence that Kotlin will be able to surpass Java in the future.

On mobile, I think Kotlin is being used more often than Java as the Android language, and I think one of the reasons for this is that due to the lawsuit between Oracle and Google, only Java 1.8 could be used. In the case of the web, where Java is currently widely used, there are no legal issues with the version of OpenJDK, and starting with Java 17, OracleJDK can be used free of charge, so I personally think that the situation is different from mobile.

Of course, Jetbrains was aware of the above problem from the beginning, so Kotlin was designed from the beginning as a language that could interoperate with Java. So, rather than just replacing existing Java applications with Kotlin, I think the goal is to gradually increase the market share through partial migration and new development. That strategy makes a lot of sense, and if a company is comfortable with operating two languages, Java and Kotlin, at the same time, I think they will be able to accept Kotlin without any problems even if they are already using Java. In fact, in my case, there was no problem at all migrating from Java to Kotlin.

## Kotlin will also become stronger

Looking at recent frameworks and libraries, although Kotlin is still less well known in fields other than mobile, it seems that it is gradually being adopted in fields where Java was the mainstream. For example, the famous web applications that I use at work, such as [Spring boot](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.kotlin), [Jackson](https://github.com/FasterXML/jackson-module-kotlin), and [AWS SDK](https://github.com/awslabs/aws-sdk-kotlin), are compatible with Kotlin, and more and more are also compatible with both Java and Kotlin, such as [jOOQ](https://www.jooq.org/doc/latest/manual/getting-started/jooq-and-kotlin/), [jooby](https://jooby.io/v1/doc/lang-kotlin/), and [Javalin](https://javalin.io/).

Alternatively, there are libraries that existed in Java that have been adapted for Kotlin. For example, [TornadoFX](https://tornadofx.io/), [gRPC](https://github.com/grpc/grpc-kotlin), [RxKotlin](https://github.com/ReactiveX/RxKotlin). And there are many that were designed specifically for Kotlin from the beginning. [Kotlin Serialization](https://github.com/Kotlin/kotlinx.serialization), [Klaxon](https://github.com/cbeust/klaxon), [DGS](https://github.com/Netflix/dgs-framework), [Ktorm](https://www.ktorm.org/), [Kotest](https://github.com/kotest/kotest), [MockK](https://github.com/mockk/mockk), [Exposed](https://github.com/JetBrains/Exposed), [KMongo](https://litote.org/kmongo/), [Xodus](https://github.com/JetBrains/xodus), [Koin](https://insert-koin.io/), [Kodein-DI](https://github.com/Kodein-Framework/Kodein-DI), etc. So, unlike a few years ago when I was parasitic to the Java world, I think we have reached a level where it is sufficient to build web applications using Kotlin alone.

In conclusion, when comparing the two languages, Java is overwhelmingly larger and more well-known, but Kotlin has also acquired the ability to compete, so I think the situation may change in the future.

## vs Go

## The virtue of being “fast”

As someone who uses Go at work, I think the superiority of Go over Kotlin is that it is simply faster. Since it is basically a language that is compiled natively, the runtime performance should be excellent, and I thought that it was certainly good that compiling and building were fast. In particular, I often verify code with unit tests after modifying it, but compared to Kotlin projects, it's faster and less stressful. (In the case of Kotlin, the web framework used is Spring, there are more test cases, and the build time is sometimes single-threaded.)

Other than that, I was impressed by the fact that you can use GitHub packages as is, and you can immediately handle structs as JSON without using a separate library (`omitempty` is also useful in some cases), and I even got the impression that it is quite specialized for web development. Since it is native, the size of the binary generated by building is small, which is also good. Considering these characteristics, I think there are many advantages to adopting Go over Kotlin in areas such as serverless and microservices, which are trending recently.

When servers had just migrated to VMs on the cloud, problems with languages ​​that used the JVM could generally be ignored due to improved machine specifications. However, as serverless and microservices architectures become popular, the characteristics of the JVM are once again becoming an issue. First, with serverless, it takes longer for the JVM to fully load, and it also takes longer to cold start [^2]. Another problem with microservices is that as the heap memory and storage occupied by the JVM increases, the cost per instance increases.

## Kotlin != slow

In response to these problems, improvements have been made to the JVM language in line with recent trends, such as the development of serverless frameworks such as [Kotless](https://site.kotless.io/), and the development of [Quarkus](https://ja.quarkus.io/) and [Spring Native](https://github.com/spring-projects-experimental/spring-native), which allow native builds using [GraalVM](https://www.graalvm.org/).

In terms of runtime performance, even JVM languages can be comparable to Go thanks to JIT optimization. Looking at benchmarks, Kotlin is slower than Go in some areas and faster in others, as shown in [this Kotlin/JVM comparison](https://programming-language-benchmarks.vercel.app/go-vs-kotlin) and [this Kotlin/Native article](https://www.lambrospetrou.com/articles/kotlin-http4k-graalvm-native-and-golang/).

Also, while [Go 1.18 introduced generics](https://go.dev/blog/intro-generics), there is also discussion about [the possibility that generics can make Go code slower](https://planetscale.com/blog/generics-can-make-your-go-code-slower). So if more language features are added to Go in the future, they may affect compilation speed and runtime performance.

So, considering the two languages ​​Kotlin and Go, I don't think it's necessary to stick to Go, at least from a performance standpoint. However, the criteria for choosing a language for application development is not just performance, but also various aspects such as productivity, languages ​​supported in the cloud, availability of engineers, etc., so it is true that choosing Kotlin instead of Go is not more efficient. When I decided to change jobs, there were more companies recruiting Go engineers on the server side than Kotlin engineers, but if performance were simply the criterion, this wouldn't have happened. I think it can be said that this is the result of a combination of various reasons, such as the fact that it is a language recommended by Google, and that it is superior not only in terms of performance but also in terms of productivity.

## Still, it is advantageous

Then there's the issue of popularity. Although it is possible to build native images in Kotlin, and the performance is not inferior, I think that most people generally think of Kotlin as a language for mobile (Android only). So, unless this perception changes between engineers and companies, I think Go will continue to have an advantage in the server-side market for some time.

In addition, I think Go is popular because of its ease of writing and ease of introduction, but based on that, I think it can be inferred that Kotlin, which is relatively complex to write, may be inferior. For me, Kotlin's writing style is concise and nice, but some people seem to think that Go's writing style is concise. Certainly, if you have fewer keywords and it takes relatively little effort to remember them, you should be able to write in a way that focuses more on logic. The increase in the number of applications and CLI tools made with Go is probably due to these characteristics. Personally, I prefer writing in Python when creating simple tools, but I also feel that Go is better than Kotlin in terms of ease of writing code and ease of use. Therefore, it is not surprising that more and more engineers prefer Go, as it is often used for personal hobbies and side projects, and I think that will lead to its popularity again.

## vs Rust

## Strongest performance?

Rust is said to have better performance than Go even though it's native because it doesn't have GC, but like Kotlin, I think it's also struggling due to its popularity. It may be unavoidable since the purpose of its development was to replace C/C++ in the first place, but I feel like it is more likely to be used in embedded systems. Surprisingly, it has been adopted by various organizations such as [Figma](https://www.figma.com/), [1Password](https://1password.com/), [Discord](https://discord.com/), [Dropbox](https://www.dropbox.com/), [Mozilla](https://www.mozilla.org/), [Line](https://line.me/en/), [npm](https://www.npmjs.com/), [Cloudflare](https://www.cloudflare.com/), and [exa](https://github.com/ogham/exa), [bat](https://github.com/sharkdp/bat), [difftastic](https://github.com/Wilfred/difftastic), [bottom](https://github.com/ClementTsang/bottom). Many GUIs and web frameworks such as [yew](https://yew.rs/), [seed](https://seed-rs.org/), [Dioxus](https://dioxuslabs.com/), [Rocket](https://rocket.rs/), [tide](https://github.com/http-rs/tide), and [poem](https://github.com/poem-web/poem) have been developed from which CLI tools, but again, you won't know unless you do some research.

Rust's performance has been validated in many benchmarks, and engineers who have used it often speak highly of it, but it is still relatively unfamiliar, so adoption can be a difficult decision for companies. In fact, in JetBrains' survey from the previous year, many engineers answered that [they use Rust for hobbies, personal use, or side projects](https://www.jetbrains.com/lp/devecosystem-2021/rust/#Rust_how-do-you-use-rust), which suggests that enterprise demand is still limited. On the other hand, if the number of engineers who like Rust keeps growing and it starts being used across more projects, the market could eventually change. As with Go, even a relatively young language can enter the mainstream if it proves its value strongly enough. So I personally think Rust's future is bright. Even after wider adoption, though, I suspect it will remain stronger in systems and embedded programming than in web application development.

## After making Kotlin native

If we compare Rust with Kotlin, we have Kotlin/Native, so what the language itself can do is not that different, but while Rust tends to replace C/C++ in the fields of embedded and system programming, I think the problem is that we don't see many results like this. In particular, since Kotlin/Native is based on LLVM, I feel that its position is becoming more and more ambiguous now that web frameworks that can be combined with GraalVM have appeared. It is said that it is possible to interop with Object-C and C/C++, but in such use cases, wouldn't it be more advantageous to use a language such as Object-C or C/C++ in the first place? Of course, Rust has concepts like ownership, and is said to be more difficult to program than other languages, so coding may be easier if you adopt Kotlin/Native. However, if you are pursuing Native, there are many situations where performance is important, so I feel that Kotlin with GC is at a disadvantage in those situations. From this perspective, I feel that the positioning of Kotlin/Native is difficult.

In conclusion, Kotlin (JVM) and Rust each specialize in different areas, and unless there are major changes, they will likely develop without undermining each other's areas. If anything, there is a possibility that Kotlin/Native will become a direct competitor, but since the positioning is somewhat ambiguous to begin with, I feel that there is a high possibility that Rust will be used in situations where Native is absolutely necessary.

## vs Python

## all purpose tool

Python, one of the most popular languages ​​over the past few years, how does it compare to Kotlin? First of all, in my case, I often use Python for everyday automation and creating simple tools, but when I want to develop full-fledged web applications, I often choose Kotlin. Of course, Python is a language that can do anything, so it's not like you can't use Python to create large-scale applications. In fact, major companies such as Uber, Google, PayPal, and Netflix use Python, and the server side of the famous Instagram is also said to be written in Python.

However, Python has the impression that it is often used in fields such as data science and AI, and although it is easy to use and is good in situations where high performance is not required, I personally feel that the problem is that its limitations are clear. I don't have any experience developing full-scale business applications, so I can only speak from my impressions and speculations, but I think most companies that incorporate Python on the server side are startups, and as the service ages, there is a high possibility that the problems inherent to interpreted languages ​​will become difficult to maintain. There is also an example of JavaScript, but Python's type hints are just hints, and they do not reliably tell you about errors that can be detected at compile time like TypeScript. As for performance, there are also issues like [GIL](https://wiki.python.org/moin/GlobalInterpreterLock). Because I am aware of this kind of problem, I am wondering if there are any examples of writing a verification application (prototype) in Python and then migrating to another language.

## It's not just a Python area either.

On the Kotlin side, [Kotlin can also be used for data science](https://kotlinlang.org/docs/data-science-overview.html#kotlin-libraries), including Jupyter support. But since Python already dominates that market, the real question is how far Kotlin can grow there. As JetBrains argues, Kotlin does have advantages over Python in areas such as static typing, null safety, and performance, but I do not think it will gain much market share unless the number of users first grows. Python is easy to start with, has many courses available, and is often used by non-engineers as well. Kotlin still seems weaker on that front.

Based on the above, it seems that demand for Python will not change significantly in fields where it has always been strong, such as data science. Although there is a possibility of competition in the web field, if anything Kotlin allows for more stable development, so I think Kotlin will be used for large-scale application development and Python for small-scale application development. Of course, there are other options when developing large-scale applications, so it seems more likely that a language other than Kotlin will be used, but this is just a comparison of the two languages.

## vs JavaScript

## Versatile

When it comes to languages ​​that can do everything in one language, I think it's Java in the past, Python a while ago, and now JavaScript. It is a language that is used in various fields such as front-end, back-end, mobile, and data science. New runtimes such as [Deno](https://deno.land/) have appeared to solve issues with runtime performance, and continuous improvements to the V8 engine have gradually supplemented these issues, and static typing has also been resolved with the rise of TypeScript. It seems like an invincible language.

For the front end, it is impossible to think of anything other than JavaScript [^3], and technologies such as [WebAssembly](https://webassembly.org/) have been developed, but this seems to be heading in a different direction than just web screen drawing, so even if something happens (I don't think it will) that it will stop being used in various fields, JavaScript itself will not stop being used. And in the same sense, I think it would be quite a hurdle for Kotlin to enter such a field.

## Front end with Kotlin?

In terms of Kotlin, there are three other axes besides Kotlin/JVM and Kotlin/Native: [Kotlin/JS](https://kotlinlang.org/docs/js-overview.html), and looking at JetBrains' communications, it seems like they are putting a lot of effort into it. In addition, through [Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/), it has become possible to create GUIs with Kotlin not only for mobile but also for web and desktop applications, so I would like to support this as I want to complete my side projects with Kotlin as much as possible. However, it is not yet that major in areas other than mobile, and there may be problems with new technology (lack of libraries, rapid changes due to version upgrades, etc.), so it will be a wait-and-see approach for a while. I also think that if you don't have a special purpose like me, there is little merit in hiring someone either as an individual or as a company.

I think there is still a good chance that Kotlin will become a competitor on the back end. In particular, in fields where Java has been used up until now, there are mainly verified safety issues such as JVM stability and accuracy of numerical calculations, so if you are going to change languages ​​from now on, I think there is a high probability that you will choose Kotlin, but in such fields, it seems likely that you will not consider using JavaScript as a backend from the beginning. I don't think there are many cases where companies actively adopt JavaScript as a backend language, unless they are in a special situation like a startup where it is difficult to recruit engineers and are trying to reduce the number of technologies used to reduce man-hours as much as possible, or create a prototype application like Python, and I don't think that will change much in the future. However, if there are people like me who want to solve everything with Kotlin on the JavaScript side, it might be a different story. If you want to support front-end, back-end, mobile, and desktop, there is nothing better than JavaScript, so there is a high possibility that JavaScript will be adopted depending on the purpose of the company or individual, and I think it is highly likely that Kotlin will not be adopted in such a situation.

## vs Dart

## The strongest GUI?

In the case of Dart, [Flutter](https://flutter.dev/) has been hot lately rather than the language itself. At first, Flutter attracted attention for its ability to do cross-platform development on mobile, but I think Flutter's biggest competitor to Dart is [React Native](https://reactnative.dev/), but looking at recent trends, it seems like that is slowly reversing. Of course, this is just a comparison based on the criteria of "cross-platform frameworks", and in reality there are probably many complicated circumstances. For example, if you consider a situation where a front-end engineer is also responsible for mobile development and is using React as the front-end library, it is unlikely that they will suddenly adopt Flutter.

I think the biggest problem with Dart, regardless of its original intention (to replace JavaScript), is that the language itself is not very impressive. After playing around with it a little bit, I couldn't really sense anything particularly appealing about it, even though I was familiar with it as a so-called C-Family language. Now, thanks to Flutter, the usage rate is increasing, but there are still questions about how it is used in other fields.

## There may be other possibilities though

However, there have been rumors for some time that Google's next-generation OS, [Fuchsia](https://fuchsia.dev/), will be the main development environment, and it is still unclear what kind of OS Fuchsia itself will be, but if it becomes the next-generation OS for Android as rumored, there is a possibility that native development itself will be based on Dart. If that happens, we will be abandoning legacy environments including ChromeOS, and it is difficult to imagine that the impact will be incomparable to that of specifying Kotlin as the official development language.

Of course, Dart is also a programming language, so the situation could still change depending on how its frameworks and libraries evolve. If you look at [this repository](https://github.com/yissachar/awesome-dart), there are already some server-side frameworks, so if your goal is to solve everything with one language, Dart might even be easier than Kotlin in some cases. Kotlin has [Kotlin Multiplatform Mobile](https://kotlinlang.org/lp/mobile/), but because its goal is to share business logic, you still end up writing iOS code. Some companies do combine Flutter for the UI with native business logic, but I do not think that will become a mainstream approach. Swift has server-side frameworks such as [Vapor](https://vapor.codes/), yet very few engineers and companies seem to use them, which suggests that merely being technically possible is not enough.

## Going strong on mobile too

In particular, as can be seen at [Google I/O](https://io.google/2022/) held this year, Flutter 3 has shown progress in various aspects such as further performance improvements and the official release of Flutter Desktop, and the future of Flutter looks bright. With more and more companies adopting Flutter, it feels like it's only a matter of time before they embrace the benefits of this development. Of course, I think there will continue to be demand for native app development, but if the number of fields where cross-platform apps are sufficient increases, it's easy to see which will become mainstream.

Under these circumstances, it would be no exaggeration to say that the mobile field is Kotlin's home base at its current share, but I feel that Flutter's growth is, in a sense, a threat to Kotlin. Therefore, it seems necessary to further strengthen the unique benefits of Kotlin. I hope that something like Kotlin/Multiplatform Mobile, which I mentioned earlier, will play that role. There are many other things that can be done with Kotlin, so the benefits of using Kotlin will continue to emerge as we strengthen collaboration across fields.

## lastly

This time, unlike usual, this article focuses on my own thoughts, so I may be biased in various ways, but for now I've summarized my impressions as a Kotlin engineer. Of course, if you don't have enough knowledge and are active in various fields such as mobile, front-end, data scientists, DevOps engineers, etc., I think you will notice that there are many mistakes or inaccurate information and judgments.

However, as an engineer, I feel that it is also necessary to pay attention to the technologies used to achieve the goals and the things that interest me, and to make my own judgments, rather than just looking at trends indifferently, so I decided to write this article. Also, by writing articles like this, I feel that it will be a meaningful way to look back and see how much my outlook is in line with the reality, given the various changes that have occurred.

There isn't much information this time, and it's more like a small talk that might be better posted on Twitter, but I'd be happy if some of you were able to gain a new understanding of Kotlin from here.

See you soon!

[^1]: Referenced the survey results of [IEEE Spectrum](https://spectrum.ieee.org/top-programming-languages), [TIOBE](https://www.tiobe.com/tiobe-index/), [Stack Overflow](https://insights.stackoverflow.com/survey/2021#section-most-popular-technologies-programming-scripting-and-markup-languages), and [Jetbrains](https://www.jetbrains.com/lp/devecosystem-2021/#Main_programming-languages).
[^2]: This can be handled by warming (always starting up idle instances), but scaling out may require a cold start, so it is unavoidable that starting up instances will increase costs.
[^3]: There has been a history of attempts to replace JavaScript with languages ​​like Dart, but they have failed, and now that JavaScript has become more sophisticated, it seems difficult to replace the front-end language unless it is a superset like TypeScript or a language that can be transpiled to JavaScript.
