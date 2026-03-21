---
title: "Personal 2021 Roadmap"
date: 2021-01-24
categories: 
  - recent
image: "../../images/map.webp"
tags:
  - blog
  - javascript
  - typescript
  - frontend
  - go
  - rust
  - flutter
  - mac
---

When working as an engineer, I think it's not uncommon for your technology stack to be determined by things other than your own intentions, such as company policy, client requests, or career history. Since the company is a profit group, it can't be helped, but what about you as an individual? I believe that engineers are a profession that must always move forward with trends, so even if there aren't many opportunities for work, I think they should create a roadmap for themselves and try to improve their skills through self-study.

For example, I have mostly worked as a server-side engineer working on Java and Spring, or as an infrastructure engineer working with Jenkins, Shell, and Linux, but I would like to create my own app or service. If you have such a goal or desire, you start thinking about what it takes to achieve it, and from there, what platform is suitable? What language? What is the framework? You start thinking like this, and eventually choose the path that makes the most sense for you. The criteria for what is reasonable differs from person to person (and it cannot be determined solely by subjectivity), but it is unlikely that your company will consider your goals as an engineer, so you should set these goals yourself.

In that sense, I have created a roadmap for myself this year based on the criteria of ``things I want to do'' and ``things that seem good.'' I haven't fleshed out any plans as a roadmap yet, so it might just be my interests... Anyway, I'll talk about what I'm interested in and what I'm thinking about at the moment, using Google Trends. Just think to yourself, ``This is what this guy is going to pay attention to in 2021.''

## language

## TypeScript

I don't know how long it will last, but it looks like JavaScript will continue to dominate for at least the next few years. However, the reason for this is that it has a strong foundation as a web standard, and thanks to Node.js and Electron, it can now be used in many situations other than browsers, so I feel that it is becoming difficult to explain just by saying that. Nowadays, there are many cases where people come into contact with JavaScript as an opportunity to start learning coding or as an introductory language, and since the advent of SPA, I feel that the front end has become more important than the server side. Considering that applications are ultimately designed for users, it may be natural that languages ​​that are more closely related to the screen have more authority.

And even if we only look at the backend, recently it seems like we are reducing the role of the server side as much as possible, or dividing it into smaller parts. I think the proof of this is the popularity of keywords like [microservices](https://www.redhat.com/en/topics/microservices), BFF, and serverless. Of course, some of this is due to the development of the JavaScript language itself, but it can't be helped because the architecture and design philosophy of applications have changed.

So I thought I should at least be able to learn the basics of JavaScript. I have learned some simple syntax in training, but I haven't had much experience writing a full-fledged application, so at least the experience of creating a simple REST API with [Express](https://expressjs.com) might be quite useful. Also, it would be better if you could touch the front end a little.

When I thought about this, TypeScript came to mind. I had previously learned about TypeScript in a course on Udemy and liked it, but it seems that it has become quite popular recently. First, I checked Google Trends to see if it was true.

{{< typescript_trend >}}

Looking at the results, it certainly seems that interest in TypeScript is increasing day by day. I think this is probably because famous frameworks and libraries like Angular, React, and Vue have started to officially support TypeScript, and it's probably because people have realized that static typing is more productive. Since I am someone who started using Java, I think it would be better to learn TypeScript, which allows for static typing.

## Go or Rust

Personally, I like the JVM language, but since it has a structure that includes both a high-level language and a VM, I would also like to work with lower-level languages. It's not like I need it right away, but I'm just curious to try something that can only be done with a low-level language, such as controlling hardware or handling binary data. Recently, the C language has become popular in IOT and other fields, but as someone who develops so-called "applied software", I think it would be better to try a language like Go or Rust rather than C, C++, or an older language.

However, the troublesome thing is which one to choose between Go and Rust. If you only consider performance, Rust may be the obvious choice. Rust is often said to outperform Go in terms of performance. As a real-life example, [Discord](https://discord.com/), a well-known voice chat tool, migrated from Go to Rust, but they say it was for [performance reasons](https://blog.discord.com/why-discord-is-switching-from-go-to-rust-a190bbca2b1f). However, it seems that learning the language itself is more difficult in Rust than in Go. And in general, Go is said to be better in terms of productivity.

So, based on the above, I wondered what I should try, and as a result, I thought why not try using Rust as a low-level language similar to C or C++. Although both are minor languages, Rust has been selected several times as the "most loved language by engineers" in the [Stack Overflow survey](https://insights.stackoverflow.com/survey), so I thought Rust could be expected to grow its community in the future. In particular, [2020 results](https://insights.stackoverflow.com/survey/2020#technology-most-loved-dreaded-and-wanted-languages-loved) are an amazing 86.1%.

However, Go is still commonly used at a practical level, and I feel that Go is still a little superior in terms of the amount of references and the interest of engineers (although it can't be helped if you use it for work). Compared to Rust, which was designed from the beginning with the goal of replacing C and C++, we still don't know much about the extent to which Go can fulfill such a role, but if it can do the same thing, I think there's no need to stick to Rust. In particular, the growth potential of a language is also related to the size of its community, so I decided to check Google Trends to find out the level of interest in the two languages.

In the case of Go, in order to distinguish it from the general verb (to go), it seems that there are many cases where `golang` is searched. However, the situation hasn't changed much with Rust (and it seems that it is also used as a game name), and I don't think the search term `rustlang` is used much, so it's difficult to make a direct comparison. Therefore, I chose `go programming` and `rust programming` as value-neutral keywords. And the result is below.

{{< go_rust_trend >}}

Looking at the results alone, Rust seems to be much more popular, but Go still has an advantage. So, I'm thinking of going this way (because it's not in a hurry), and I'm going to take my time and decide while observing a little more.

## Kotlin

Nowadays, Java's `Write once, run everywhere` can do the same thing in any language (although there are some areas that Java can't do), but I still think the JVM language is attractive. Since the JVM sets the heap from the beginning, it is stable in terms of memory management, and its performance is also superior compared to the currently popular high-level languages. Also, since it has been the most popular language in the world for over 10 years, it has a wealth of libraries, frameworks, and references. Also, since you only need to generate bytecode, you can use any language before compiling. So, in addition to Java, [Scala](https://www.scala-lang.org), [Clojure](https://clojure.org), [Groovy](https://groovy-lang.org), [Jython](https://www.jython.org), [JRuby](https://www.jruby.org ), [Ceylon](https://ceylon-lang.org/), [Frege](https://github.com/Frege/frege), [Eta](https://eta-lang.org), [Haxe](https://haxe.org), and many other languages can now use the JVM. In other words, although the JVM will not die, the Java language itself can be replaced by one of these languages.

There are many language candidates, but I personally chose Kotlin. Although Java has been improved in recent years through rapid updates, the only time you can actually expect the effects of such updates at the enterprise level is when an LTS version is released. So I think it's the right time to consider switching to a modern language like Kotlin, in terms of being able to use the JVM as-is while immediately increasing productivity. Of course, this is especially true if you're thinking about developing a mobile app like I am.

Another good thing is that it is a language recommended by Google, and that it can be compiled with other languages ​​such as [Kotlin/Native](https://kotlinlang.org/docs/reference/native-overview.html) and [Kotlin/JS](https://kotlinlang.org/docs/reference/js-overview.html) (In fact, Wantedly seems to have [already introduced Kotlin Multiplatform](https://www.wantedly.com/companies/wantedly/post_articles/282562)). Best of all, since Kotlin is developed by JetBrain, Intellij provides complete support, which is an advantage that cannot be ignored. Although I only used it for a short time, I found it to be a language with a much higher level of perfection (than Swift), so I think the future of Kotlin is quite bright.

## Frameworks & Libraries

## Svelte

I talked a little bit about JavaScript earlier, and although there is no need to talk about the demand and importance of JavaScript itself, I think that the question of which JavaScript framework/library is the best will be at a point where it can be said that this is the correct answer in at least a few years. Over the past few years, many frameworks and libraries have come and gone. Fortunately, it seems that React is becoming the winner among the so-called three front-ends: Angular, React, and Vue. Google Trend results also show this.

{{< angular_react_vue_trend >}}

However, the world outside of the front end is a different story. There are still many frameworks and libraries proliferating, and it looks like the Warring States period. Under these circumstances, it can be difficult to know which one to choose, and it takes a lot of time and effort just to do the research to make a decision. Because of this situation, there is a term that has been popular for several years now: [JavaScript Fatigue](https://www.google.com/search?newwindow=1&biw=1680&bih=836&sxsrf=ALeKk03Q7zTnfCMWJbsybKG4qkODOhqViA%3A1611463509708&ei=VfsMYOPbKpvahwO8zpiwCQ&q=javascript+fatigue&oq=javascript+fatigue&gs_lcp=CgZwc3ktYWIQAzIGCAAQBxAeMgIIADIECAAQHjIECAAQHjIECAAQHjIGCAAQBRAeOgQIIxAnOggIABAIEAcQHjoICAAQBRAKEB46CAgAEAgQChAeOgYIABAIEB46BAgAEBM6CAgAEAcQHhATOgoIABAHEAUQHhATUOeYAVjBwAFgnsQBaAFwAHgAgAGyA4gB8BOSAQkwLjcuMy4xLjGYAQCgAQGqAQdnd3Mtd2l6wAEB&sclient=psy-ab&ved=0ahUKEwij2sGw4bPuAhUb7WEKHTwnBpY4ChDh1QMIDQ&uact=5). Learning modern JavaScript can be quite difficult.

For example, if a person like me with little JavaScript experience becomes a front-end engineer and decides to do React because it is the most popular, he may start with Node.js, use npm or yarn for package management, or choose JavaS as the language. There are many things you need to know and learn, such as deciding whether to stick with cript or using TypeScript, and then learning [Webpack](https://webpack.js.org), [Babel](https://babeljs.io), and [Redux](https://redux.js.org). What's more, it's hard to tell what each framework or library is just by its name. [Nuxt.js](https://ja.nuxtjs.org) is a Vue-based framework, but [Nest.js](https://nestjs.com) is a framework for Node.js. And [Next.js](https://nextjs.org) is also a React-based framework. It can be confusing to know which ones to learn and which ones are good. So it's no surprise that engineers who work with JavaScript feel fatigued.

In my case, I can already do server-side implementation to some extent, so I would like to be able to touch the front end as well, and be able to write full-stack applications by myself. However, if there is a front-end framework used by your company, it would be a good idea to use it, but on an individual level, it is still difficult to know which one is better. Since React is the most popular, should you still choose it? That may be a good choice, but unless you plan to get involved in full-scale front-end development from now on, I feel like it would be a waste to seriously invest time in front-end development. The alternative I came up with was [Svelte](https://svelte.dev).

There are many features (benefits) of Svlete, but the one that caught my attention the most was that it was quite simple. Since the code is short, I feel that it will be much quicker to get used to writing it. Other than that, it's a good additional benefit, and I think it's pretty good as something you can put on quickly when you need it. Of course, since it is a proper framework, it is also good for creating full-scale applications.

However, the disadvantage is that it is not as well known or used as the three major players. Fortunately, when I checked Google Trends, it seems to be gaining attention little by little, so I feel like it's coming soon.

{{< svelte_trend >}}

## Flutter

I'm currently only writing web applications, but I'm also interested in mobile, so I'd like to know what languages ​​and frameworks are available. And these days, it seems like mobile is often more of a hybrid/close platform than native. I haven't looked at exact data or statistics, so this is just a guess, but my impression is that startups and venture companies that often don't have the time or budget to invest in native apps prefer hybrid/close platforms. Of course, if you want to use complex calculations or OS-specific functions, it is said to be native, but from my personal experience, there are surprisingly many things that can be done on a hybrid close platform, so it is no longer necessary to use the OS level, and if you do not need to utilize device-specific functions, I think that a hybrid close platform will generally suffice.

(My personal experience here is that I wanted to create a simple app that takes advantage of the widget feature introduced in iOS 14, so I did some research and found out that something I thought would be difficult because it is an OS-specific feature, could surprisingly be done with React Native or Flutter.)

Also, I feel like in the past there were many apps that were just WebViews, but if that were the case, there would be no need to make them as mobile apps (I understand if it's a PWA). However, perhaps because of the existence of such apps, I get the impression that there are quite a number of hybrid mobile app frameworks that were born out of influence from web technology. Therefore, there are quite a lot of frameworks that allow you to write code in JavaScript and to write applications based on JavaScript frameworks. Examples are [Apache Cordova](https://cordova.apache.org), [Ionic](https://ionicframework.com), [NativeScript](https://nativescript.org), and [React Native](https://reactnative.dev). Of course, there are C#-based [Xamarin](https://dotnet.microsoft.com/apps/xamarin) and Dart-based [Flutter](https://flutter.dev), which are frameworks that seem to be in a different lineage than JavaScript (Web), that is, traditional desktop apps.

There are so many frameworks for hybrid and cross-platform mobile apps, but there are some technologies that are likely to be weeded out soon. Let's also take a look at the Google Trend results. Since I can only compare 5 items, I didn't include Flutter.

{{< mobile_frameworks_trend >}}

At the very least, I can see that not many people are interested in NativeScript, and interest in Xamarin and Cordova is gradually decreasing. Then, the remaining results are Ionic and React Native. I talked a little bit about front ends earlier, but given the reality that React is likely to be the winner of the top three front ends these days, I think React Native is the appropriate framework for hybrid cross-platform mobile apps based on web technology.

But the problem is Flutter. Flutter is often compared to React Native. So, I decided to compare Flutter and Google Trends. As a result, I feel that Flutter is superior to React native.

{{< flutter_react_native_trend >}}

Considering that both allow iOS and Android apps to be developed at the same time, I think the reason cited is performance issues. In React Native, it is said that the structure of calling native code from JavaScript inevitably creates a bottleneck. And, although this is just a guess, I feel that Flutter's unique advantage is that even though it uses a different language called Dart, its syntax is quite similar to languages ​​like Java and C#, and it allows for declarative UI implementation that is different from HTML and XML.

If I were to create a mobile app, I think it would most likely be native (if a hybrid cross-platform is needed, a web-based app would probably suffice), but depending on the case, a hybrid cross-platform might also be a good option. And Flutter is going to grow as a framework for more platforms than just mobile, so if you want to learn from now, Flutter might be a better choice. Of course, if you are a front-end engineer who can already use React, I think React Native is better, but in other cases I think Flutter is better. Therefore, I would like to keep Flutter in mind for the time being.

In addition, there were some interesting articles regarding React Native, so I will list some examples below.

- [React Native at Airbnb](https://medium.com/airbnb-engineering/react-native-at-airbnb-f95aa460be1c)
- [React Native: A retrospective from the mobile-engineering team at Udacity](https://engineering.udacity.com/react-native-a-retrospective-from-the-mobile-engineering-team-at-udacity-89975d6a8102)
- An article about whether React Native should be adopted, based on Shopify's case

## hardware

## Apple silicon Mac

I've been using only Microsoft products for OS for about 20 years. It just so happened that it was a better compatibility with me than I expected after my first experience with Apple products on my iPhone and iPad (I was using Linux for work, so being able to easily use the terminal was a big factor), so I would like to continue using Mac in the future. At least in my environment, I don't have many problems unless I use Windows, although I do have some problems if I don't use Mac.

So I naturally became interested in Apple Silicon Macs, but suddenly changing the CPU architecture means that compatibility cannot be guaranteed, so I was wondering how Apple would come up with a solution to that problem. After reading [various articles](https://jp.techcrunch.com/2020/07/11/the-real-reason-why-apple-is-putting-apple-slicon-on-the-mac) immediately after the announcement, I could at least predict that the performance (including calculations, heat generation, and power consumption) would be better than Intel, but I didn't know if that was because the architecture was created using a more advanced process, or if it was due to customization, or how much better it was. So I believed the story that ``the transition would take place in two years,'' and decided to wait and see how things turned out.

Nowadays, there are various Macs using M1 chips, and their performance has been verified. What's certain is that when you look at performance alone, there seems to be no reason to buy an existing Intel Mac anymore. I think this is just a declaration that the CPU architecture will be changed, even though there are concerns about compatibility. However, as an engineer, I look first at compatibility and stability. This is because I have learned from experience that the first major version is always likely to have some unknown problem. In fact, there have been some reports of Bluetooth issues, difficulty with initialization, and problems with not waking up from sleep mode, and there is also a problem that officially only one external display is supported (I think this is probably a problem with the bandwidth of Thunderbolt 3). Also, although apps compiled with Universal binaries and M1 Native are being announced one after another, there are still many apps that are not compiled with this, which is a cause for concern.

However, all Macs will eventually be converted to Apple Silicon, so even if you don't buy a model equipped with the M1 chip right away, I think it's worth paying attention to. Well, not only is it attracting attention, but there are also rumors of a complete change to the 16-inch Macbook Pro this year, so if that's true, I'm thinking that I might switch too. If it comes out, it should improve the issues raised by the M1 chip-equipped models (at least the external display and [Bluetooth issues](https://9to5mac.com/2021/01/21/macos-big-sur-11-2-rc-now-available) are likely to be improved), and looking at the current speed at which applications are compatible with M1 Native, I think we'll be able to use a surprisingly large number of apps natively by the end of this year. There is still a lot of attention to be paid in the future, but I think the current M1-equipped model has sufficient benefits for those who do JavaScript-centered development (AdoptOpenjdk is still x64 only, so I'm putting it off...). Also, recently it has become possible to use Linux on M1-equipped models (https://www.theverge.com/2021/1/21/22242107/linux-apple-m1-mac-port-corellium-ubuntu-details), so I think it would be a good choice to consider using these Macs as a home server.

## lastly

I can't say I'm used to all the features of Java and Spring yet, so it might be unreasonable for me to plan to learn a new language or framework now. This is always a troubling subject. Is it better to hone your knowledge and skills in one language, or should you always keep up with trends and have a skill set in a wide range of fields? It's good to have both deep and broad knowledge, but the question remains as to which one is more efficient for completing the career you're building.

If I had to give my own answer, I think following trends would be more effective for those who want to deepen their skill set. It's something that only Java can do, so to use Java's API as an analogy, Lambda was introduced in Java 1.8, influenced by Closure in other languages. Other improvements such as var type inference and text blocks were also influenced by other languages. These changes would not have happened if Java developers had not turned their attention to other languages ​​in the first place. Therefore, I think that ``comparing yourself to others can help you understand yourself more deeply.'' In that sense, I think it's important not only to rely on the skill set you already have, but also to quickly catch and embrace industry trends and trends.

This is a place where I can only express my subjective opinion, but what do you think? My thoughts and trends may change in the future, but for now I think it's good to just share my conclusions. And the more I research and study about fields that I don't know about, the more I realize that I don't have anything, and it's a good stimulus. I have to continue to learn a lot from now on.

See you soon!
