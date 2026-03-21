---
title: "Things I've been paying attention to recently"
date: 2020-06-29
categories: 
  - recent
image: "../../images/magic.webp"
tags:
  - apple
  - mac
  - deno
  - javascript
  - c#
  - dotnet
  - flutter
  - rust
---
I think it's a rule in this industry that you always have to keep learning, but change is accelerating every day, and there are times when you don't know what to follow or what to rely on. As new languages ​​are appearing one after another, languages ​​that previously did not attract attention due to performance issues suddenly become popular, and paradigms that were taken for granted are sometimes turned upside down. I think it would be no exaggeration to say that we are now in the era of JavaScript, but what will happen in the future? I have summarized some trends in the IT industry that I am personally interested in.

## Apple Silicon mac

The other day, Apple's developer event ["WWDC2020"](https://developer.apple.com/videos/play/wwdc2020/101) was held. I don't always watch it live every year (it starts late at night, so...), but lately I've been working remotely and have no time to commute, so I was able to watch it live from beginning to end.

Of course, the changes in iOS, iPadOS, and macOS were interesting, but the most impressive thing about this event was that the main processor of the Mac would be [changed to one made by Apple](https://www.apple.com/newsroom/2020/06/apple-announces-mac-transition-to-apple-silicon). I have never experienced a Mac from the PowerPC era, but the transition from PPC to Intel was successful, so I think it will be the same this time. I personally use an iPad, and I have never had the impression that its performance is inferior to Intel's CPUs.

Of course, I might not be able to use Bootcamp, and I might not be able to use other third-party applications built on x86, so even if a Mac with a new processor is released this year, I won't immediately buy a new one (it's expensive...). However, seeing that companies have invested years to build environments that allow applications to run on different platforms (processors) without any problems, such as [LLVM](https://llvm.org) and [Catalyst](https://developer.apple.com/mac-catalyst), I think that the conversion can be done surprisingly quickly and without any problems. In particular, since I only write web applications, I don't have much exposure to low-level technology, and I just hope that the compiler for the language I use is compatible with new processors. In fact, in the WWDC introduction, it was said that all you need to do is build the project created with Xcode again.

The only thing I'm curious about is how long the OS will continue to support my Mac equipped with an Intel CPU. Recently, the development of Windows has been amazing, but in my environment I don't need to be too particular about Windows, so I feel like I can switch to it without any problems. I also think that the big advantage is that it's not just a processor, but because it's an SoC, you can use the various sensors and neural engines that are already used in iPads and iPhones.

However, these opinions are just personal issues from my perspective as a developer, and I think you need to be careful because no matter how good [Rosetta2](https://www.apple.com/newsroom/2020/06/apple-announces-mac-transition-to-apple-silicon) or [Universal Binary2](https://developer.apple.com/documentation/xcode/building_a_universal_macos_binary) are at the enterprise level, compatibility or performance problems may occur somewhere. The Office demonstration was good because it seemed to have better performance than it does now, but there's no reason for an office worker to use a Mac in the first place...

In addition, it seems that you can already rent a Mac equipped with the A12Z for $500, so the performance and compatibility issues of the new processor Mac may become clear sooner than expected. I think we should focus on that first. I would like to know how much the performance has improved, as well as the temperature and power consumption during work. I think we'll be able to see the benchmarks in the next week or so.

## Deno

I haven't touched JavaScript much, so I'm not familiar with Node.js, but we've come to the age where modern web apps can't be done without Node.js. In my case, I had a better impression of TypeScript than JavaScript, so I thought it would be nice to have native TypeScript support in Node. It seems that a new runtime called [Deno](https://deno.land) has been released by the developers of Node.

Basically, it seems to have been created based on the points learned from Node (like async/await?), and the other feature is that it has a built-in TypeScript compiler, and for me, the biggest advantage is that it can be used without having to compile it each time (without transpiling to JavaScript).However, the problems with Deno are that it cannot use existing Node.js modules and that TypeScript compilation is slow. There seems to be a plan to create its own TypeScript compiler using Rust, but I don't know when it will be completed, so I may not be able to publish many projects using Deno for a while.

## Blazor

In May, [Blazor WebAssembly release announcement](https://devblogs.microsoft.com/aspnet/blazor-webassembly-3-2-0-now-available) was held at [``Microsoft Build 2020''](https://news.microsoft.com/build2020). It seems that now you can create a web application using .NET and C# that can be executed from a browser.

One of the benefits of web applications using Node.js is that you can develop both server-side and client-side with a single language, but frameworks and runtimes like this are emerging that make this possible without using JavaScript. JavaScript is also a good language, but it has clear limitations, so I think there are many advantages to being able to use a compiled language like C# on the browser compared to the opposite pattern (using JavaScript on the server side).

Additionally, Blazor PWA, which can implement PWAs as part of the Blazor family, Blazor Hybrid, which can implement desktop applications using Electron and WebView, and Blazor Native, which can implement native applications without HTML elements, are scheduled to be released one after another, so I think this will stimulate people to be able to run other compiled languages ​​in the browser.

There's also WSL and GitHub, but in many ways Microsoft's recent changes and investments have been amazing...

## Flutter for web

[Flutter](https://flutter.dev) has the advantage of being able to develop apps for two platforms, iOS and Android, at the same time, and is comparable in performance to native apps compared to React Native, which has the same functionality.Recently, web applications using Flutter seem to be appearing one after another. This is not surprising since Dart (https://dart.dev), the language used to write Flutter, was developed with the goal of next-generation JavaScript, but the fact that you can develop both mobile and web applications with one language is as attractive as Blazor.

However, since it is being developed by Google, wouldn't it be better to use Kotlin instead of Dart? I feel that way. There is also the question of how long Flutter will be able to survive since Google tends to abandon its own services quickly. Since Microsoft's Xamarin is not so good, I think you might decide to use Flutter for mobile development, but if you want to develop a web application, I think Blazor, which uses C#, is more attractive.

## Rust

[Rust](https://www.rust-lang.org) is attracting attention as a post-C and C++ language, but recently the popularity of this language is frightening. Rust does not seem to be used in many cases at the enterprise level due to the lack of existing systems or skilled engineers, but it seems that its biggest advantage is that it is stable while offering performance comparable to C or C++.

Personally, I don't have much to do directly with the system at the web application level, but with the advent of PWA, there are times when web applications are required to have the same performance as desktops, and I feel that Java has its limits in handling binary files that cannot be handled directly, so I think that if I could handle a language like Rust, I would be able to create better applications.

In particular, there was an article about [Discord](https://discord.com), which is famous as a communication tool, deliberately replacing [Golang](https://golang.org), which was originally used by Rust, so I'm starting to wonder what the benefits are that overwhelm even Golang, which was also developed as an alternative language to C and C++, and I'm getting more and more curious about it. In the case of the recently popular Kotlin, I don't think it would have been able to succeed without the feature of being fully compatible with Java, but I'd like to know what advantages Rust has that would make me switch to a language that isn't compatible.

## Finally

There are various technologies and changes, and although I have mixed feelings about happiness and pressure, I believe that all of them are bringing about positive changes. In particular, with regard to changes in each language, it seems that Java's motto `Write once, run everywhere` is being realized in a more developed form in each language. I feel like all languages ​​are becoming similar in the end, but on the other hand, I feel like we're living in an era where it doesn't really matter which language you use anymore.Therefore, it is important to hone your skills so that you can keep up with such changes. I think that the fact that which language you use becomes less important means that what you can do with that language becomes more important, so I feel that the priority should be to try out various experiences using the languages ​​that you are currently capable of. I haven't been able to implement it much lately, but I wanted to try it out while working on a personal project.

See you soon!