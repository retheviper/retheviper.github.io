---
title: "TypeScript as seen by a Java programmer"
date: 2020-05-16
translationKey: "posts/typescript-first-impression"
categories: 
  - typescript
image: "../../images/typescript.webp"
tags:
  - node
  - javascript
  - typescript
---

In a new project, I will be in charge of developing a web application using [Angular](https://angular.io) and Spring Boot. I've been using Spring Boot for a long time, so I don't think there's much of a problem, but the problem is with Angular. It seems that Angular has been using TypeScript since version 2, and since I only have a little experience with JavaScript and jQuery, I didn't even know what TypeScript was in the first place. All I knew was that if I wrote code like a static language like Java or C#, it would compile it nicely and turn it into JavaScript.

So I started studying TypeScript, but I realized that the knowledge required to create something with TypeScript was surprisingly large, including the characteristics of the language itself. In this post, I would like to give my honest impressions and what I need from my perspective as a beginner who only knows Java.

## surprisingly similar to Java

I had originally heard that TypeScript was similar to the so-called C-family languages ​​such as C/C++, C#, and Java, but when I actually tried TypeScript, I realized that it was indeed true. At first, when I didn't have any information about it (I only heard that it was possible to specify a type and that it would turn into JavaScript when compiled), I thought it was just a way to specify a type in the declaration of a normal variable, but I got the impression that it was not just that, but that it was evolving or regressing in line with object orientation.

## Type specification looks good

Type specification is what turns JavaScript, which is originally a dynamic language, into an old-fashioned static language. However, the writing style itself is not traditional, and is similar to modern languages ​​such as Kotlin and Swift. Rather than writing the type first, you declare it as a variable and then add the type [^1]. For example:

Type specification in Java

```java
int a = 1000;
```

Typescript type specification

```typescript
var a: number = 1000;
```

Also, this kind of type specification is also called "type annotation" in TypeScript. TypeScript is compiled into JavaScript whose type is determined at runtime, so it seems to be similar to the meaning of adding annotations for the compiler.

## Similarities other than type

In addition to type specifications, TypeScript also provides Access modifiers, Interface, Class, Generics, Decorator (Annotation), etc. that C# and Java programmers are already familiar with. Although some of these functions are recently supported by JavaScript, it is inconvenient that in actual application development work, you may not be able to use the latest JavaScript due to browser restrictions. However, with TypeScript, you can select which version of JavaScript to output the source code as by specifying compiler options, so you can write the same code without worrying too much about the browser. I really appreciate this.

Another good thing is that dependencies can be managed with a package manager such as npm. This can be done with JavaScript (or rather, that's the first step), but the good thing about it is that when you combine it with Import, you can operate it almost in a similar way to the C-family. JavaScript modules are also supported starting with ES6, so I thought it might be quite troublesome if you use an earlier version. Maybe it's just a lack of knowledge...

## What I was curious about

I think the good thing about TypeScript is that it provides an environment in which people who are used to traditional languages ​​can write code in a fairly comfortable manner, and it is also a good thing that it allows you to look for errors at compile time. However, there were some things that bothered me, so I'll write about them as well.

## JavaScript after all

TypeScript is said to be a super set of JavaScript, so it is said that the feature is that you can use the functionality of JavaScript as is. At first, I saw this as an advantage, but as I was studying, I realized that there were also disadvantages. In other words, applying JavaScript as is means that there is no problem even if JavaScript (Vanilla) and TypeScript are mixed in the source code. If this happens, I don't think the code will stop working unless TypeScript gives up its super set feature, but I think it could create code that is quite confusing from a human perspective.

### What it means to be able to write in JavaScript

To begin with, JavaScript has a longer history, and there are definitely more users using JavaScript. Also, there are people like me who started programming from JavaScript rather than C-family languages, so I think that for those people TypeScript may be seen as nothing more than an inconvenience with no real benefit to using it. Also, even if someone who is accustomed to JavaScript moves to TypeScript, it is possible that the writing style will end up being the same as JavaScript.

I think the reason why TypeScript was planned as a super set was to absorb existing JavaScript programmers, but I think that the more freedom a programming language has, the easier it is to get confused, so I thought this was both an advantage and a disadvantage.

### prototype

Perhaps because I started with a class-based object-oriented language, I don't really understand the concept of JavaScript's `prototype`. In any case, JavaScript has this prototype, which has the characteristic that objects can be used as variables [^2]. As a Java programmer, it is common to think of an object as a class, and a class as a single file, so when you come into contact with this degree of freedom, you feel confused as to what to do. And, of course, this is more a feature of JavaScript than TypeScript, but it's an unavoidable problem since TypeScript is super set.

### Module

Modules have been introduced into JavaScript and can of course be used in TypeScript as well. If you create classes one by one in separate files, import them, and use them, it may seem like there is no difference from Java. However, it seems that this module is actually somewhat complicated, and you may have to think about how to write it.

For example, like in Python, you can give an imported module a different name by including `as`, but at first I thought it was quite complicated, as there were cases where doing so would cause problems, and there were cases where it would be blocked by CORS settings. I haven't really dug into it yet, so I don't think I'll know until I actually use it, but I'm still curious about the fact that you won't know until you try it.

```typescript
// Two forms are possible, but they are not the same
import express from "express";
import * as express from "express";
```

### compile

After installing TypeScript, you will be able to use the command `tsc` from the terminal. The file created with TypeScript is written in a file called `.ts`, and the tsc command changes it to `.js`. You can also specify various options for how to compile by defining the `tsconfig.json` file. For example, when compiling to JavaScript, what version to compile with, which files to compile, where to output the `.js` file, etc.

However, it takes a lot of time to compile, and the compiled file ends up being JavaScript [^3]. And this means that even if you create it with TypeScript, it will become a weak type at runtime. I got the impression that there are many cases where simply writing code with the types clearly specified (without using `any`) is not enough, so it may be necessary to ensure that type-related problems do not occur at runtime as well.

Recently, something like [Deno](https://deno.land) has appeared as an alternative runtime to npm that allows you to use TypeScript as is, but it seems that it still uses tsc internally, and the biggest problem was that it was slow to compile. I hear that you are creating a new compiler using Rust, but I don't know when it will be completed.

In addition to using tsc, when using [webpack](https://webpack.js.org), I got the impression that the initial settings were quite complicated when using ts-loader. It seems that you may also use something like [Babel](https://babeljs.io), but it is a little inconvenient that you have to learn such a bender and compiler again just to compile. It may be the fate of a language that is tied to JavaScript...

## A clean place

## Use JavaScript libraries

I've heard that TypeScript can also use JavaScript libraries, but I was wondering how that was actually possible. For example, TypeScript has type inference, so there are some cases where you don't need to declare a type, but there are some parts that won't compile properly unless you explicitly specify the type with a type annotation. However, it is unlikely that all existing JavaScript libraries were created with TypeScript in mind, so the biggest question was how TypeScript would handle this.

The answer is surprisingly simple: it is best to create a "type definition file" in the form of `.d.ts`. Even if the library is already compatible with TypeScript, this type definition file allows TypeScript to reference the types, and you can also create your own. In the case of major products, type definition files were sometimes provided in a form that could be installed in `node_modules`.

Installing type definition files that have already been created is easy. For example, if you need a TypeScript type definition for node, you can install it with a single command like `npm install --save-dev @types/node`. You can also check what library type definition files are provided by referring to the [repository](https://github.com/DefinitelyTyped/DefinitelyTyped) on GitHub.

## lastly

There are still a lot of concerns, but most of them are due to the characteristics of JavaScript rather than TypeScript itself, so when asked which language a front-end engineer should choose at this point, I think TypeScript is better. Of course, I think this is partly because I started with Java, but I still think that the advantage of being able to catch many problems in advance at compile time is irreplaceable.

Currently, there are some problems such as complicated initial settings and long compilation time due to compatibility with JavaScript, but I think these problems will be solved naturally if more frameworks and libraries are created based on TypeScript from the beginning like Angular, or if browsers support TypeScript (I think this is unlikely...).

At first, I studied Angular simply to use it, but it turned out to be surprisingly solid, and I feel like the future is bright, so I would definitely recommend it to other engineers.

Oh, it would be nice if Python could also be typed...[^4]

[^1]: Of course, these days you can declare variables with `var` in both C# and Java, but there are customs, and I personally think it's better to start with types in static languages.
[^2]: Moreover, the Class that appeared in ES6 ended up being just a slight change in the way the prototype-based `function` was written.
[^3]: TypeScript compilation is also called `transpilation` because it is a little different from things like Java, which converts to bytecode, or C, which converts to machine language.
[^4]: I lacked knowledge! Python has also been able to declare types since version 3.6.
