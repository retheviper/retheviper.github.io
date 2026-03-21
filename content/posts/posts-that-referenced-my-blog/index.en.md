---
title: "Posts I Found That Referenced My Blog"
date: 2026-03-21
categories:
  - recent
image: ../../images/magic.webp
tags:
  - blog
  - reference
  - retrospective
translationKey: posts/posts-that-referenced-my-blog
---

Sometimes I wonder if the articles I've written are actually being used somewhere. Of course, PV and search traffic have meaning, but whether someone leaves them as reference material for their own article is a different kind of trace.

This time, I would like to briefly introduce some examples that I found.

## 1. Qiita article based on Spring DI article

The first one we were able to confirm was Qiita's article about using `@Autowired` with constructor injection in Spring.

This article explains how to use `@Autowired` with Spring, and in particular why constructor injection is preferable to field injection. The references at the end of the article included articles related to Spring DI that I previously wrote.

What was interesting was that the link was the URL for the old blog configuration, not the current `/posts/...` configuration. This also means that the address from before moving to Hugo is still in someone's article. I honestly felt happy that an article I wrote a long time ago remained as reference material to support someone else's explanation.

## 2. Java 9-11 articles referenced in the Collectors API article

The next thing I found was Qiita's article on the Java `Collectors` API.

This article is a comprehensive review of Java's `Collectors` API with examples, and where `flatMapping` is explained, my blog's article on the new methods from Java 9 to 11 was linked as reference material.

What was particularly impressive was that there was a direct link not to the top of the article, but to the anchor with the explanation of `flatMapping`. I think it was used to pinpoint the necessary context. Technical articles are often read as supplementary material for specific APIs or detailed specifications, but these links feel more like ``actual use'' than just an introduction.

## 3. File copy article referenced in the Java ZIP compression article

The third one is an article from SOFTEMCOM Developers Blog about compressing files into ZIP format in Java.

This article organizes the code to create a ZIP file in Java and also touches on file processing using `transferTo`. In that vein, my blog article about file I/O was introduced as a reference site.

What's interesting about this example is that it's not about the exact same topic as my article, but rather a close technical context. The theme of "file copying" was reused more broadly in discussions about stream processing and how to handle I/O, and I thought it was quite interesting to see a blog post serve as material for another explanation.

## 4. Kotlin performance article referenced in Loglass' tech stack article

The fourth one is Zenn's article on Loglass's backend technology stack in 2023.

In this article, Loglass's backend team summarizes the technology stack as of 2023, the reasons for its adoption, and the advantages and disadvantages of each. In the process of explaining Kotlin, my blog article on Kotlin's hidden cost was linked as a reference material along with the context that "Kotlin has hidden costs."

This left quite an impression on me personally. This is because my writing was referenced in an article that was not just an introduction to grammar, but actually talked about service development and performance. Regardless of whether I like Kotlin or not, I was a little moved that my record of thinking about where the costs are and what I should be concerned about was used again in this context.

## 5. Ktor first impressions article referenced in Ktor introductory article

The fifth one is Zenn's article about choosing Ktor + Exposed instead of Spring.

This article summarizes my experience using Ktor and Exposed instead of Spring, and included my blog's Ktor first-impressions article in the references at the end.

Looking at this example, I realized that the "first impressions" type of articles I wrote a long time ago can also serve as a supplement to someone else's introductory article. The name "First Impressions" may seem a bit light, but it may be a surprisingly useful record in the sense that it records what caught the eye of first-time users and what they found interesting.

## What I Thought After Looking Through Them

The examples I found this time were not ostentatious citations, but rather were left as references at the end of the article or as links to supplement specific context. But personally, I feel more happy with such links. This is because when the person actually wrote the article, they decided that this was helpful and left a mark.

Another interesting thing was how varied the referenced articles were. They covered Spring DI basics, detailed Java API changes, everyday topics like file I/O, Kotlin's performance and cost, and even my first impressions of Ktor. I believe that value does not belong only to widely read articles, but also to long-lived posts that someone later picks up to fill a gap in their own explanation.

When I reread old articles, I find that some of them are unsatisfactory by today's standards. There are some parts that are poorly explained, and there are some themes that I think could be written in a better way now. Still, seeing that it remains as a reference for someone like this makes me feel that even if it's not perfect, there was a point in writing down what I was thinking at that time.

## Finally

Although I wasn't able to find many external references, at least some of the articles explicitly used my blog as a reference. Some URLs still had the same structure as before, and others were reconnected in unexpected contexts.

I don't think a technical blog needs to be a place where you only write the latest correct answers. Rather, a record of what you were thinking at that moment, where you got stuck, and how you understood it may be useful to someone later. The link I found this time felt like a trace quietly showing me that.

I would be happy if these kinds of traces continue to increase little by little. If I were to write a similar article someday, I would like to collect more examples and compile it as a sequel.
