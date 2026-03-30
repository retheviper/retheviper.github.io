---
title: "What I Want from Libraries in the AI Era"
date: 2026-03-26
translationKey: "posts/ai-library-design"
categories:
  - ai
image: "../../images/ai.webp"
tags:
  - ai
  - graphql
  - library
  - architecture
---

Recently, it has become fairly normal to let AI write code. That does not mean humans no longer need to think, but it does mean that the amount of code people have to write by hand is clearly shrinking.

Because of that, I think mechanisms that were mainly valued for reducing manual code should be reexamined. GraphQL is one example of the kind of thing that may deserve a fresh look under this new condition. I do not mean that GraphQL itself has lost all value. I only mean that "it reduces the amount of code I have to write" is no longer as strong a reason as it used to be.

## What used to be rewarded

For a long time, development naturally favored tools that reduced how much people had to write. That made sense. Repeating the same kind of code by hand is tedious, and it is also an easy place to make mistakes.

That includes things like:

- DSLs or helper libraries built mainly to reduce boilerplate
- mappers that hide trivial transformations
- abstractions introduced mostly to save typing
- internal platforms that only work if everyone memorizes their conventions

There was a period when these kinds of choices were very reasonable. If humans were doing all the repetitive work themselves, even small reductions in manual effort could feel meaningful.

## The premise has changed

Now the premise is different. AI can generate code from scratch, read an existing codebase, and suggest edits. The cost of writing repetitive code or obvious transformations has dropped quite a bit.

That is why abstractions whose main value is "people do not have to write as much" no longer feel as compelling to me. If reducing manual typing increases the cost of understanding, debugging, or changing the system later, that trade-off should now be judged much more strictly.

## What seems more important now

For libraries and internal platforms going forward, I think the center of gravity should shift away from "can I write this in fewer lines?" and toward questions like these:

- Is it easy to read?
- Is it hard to break?
- Is it easy to debug?
- Is it easy for AI to generate, modify, and review?
- Is it easy for the whole team to understand?

The AI part matters more than it may seem. AI loses reliability quickly around vague local conventions and overly compressed abstractions. Code with direct structure and visible responsibilities tends to be easier for both humans and AI to work with.

## Fewer lines is no longer enough

If a mapper layer hides simple field moves, or a custom DSL compresses ordinary control flow and configuration, the code may become shorter. But when something goes wrong, it can become harder to see what changed where and why.

And today, AI can often generate that kind of straightforward transformation code almost immediately. In that situation, leaving a few more lines of explicit code behind may actually be the better trade. If brevity becomes the goal by itself, the result can be worse for the reader, the maintainer, and the AI.

## Custom foundations also cost more

The same applies to internal foundations that require people to memorize a private set of rules. If they solve a large and recurring problem, they may still be worth it. But if they mostly exist to save a little writing, I am not sure the ongoing learning cost is justified anymore.

That cost is paid not only by humans, but also by AI. The farther a codebase moves away from direct, standard, readable structures, the easier it becomes for generation, edits, and reviews to get unstable.

## Finally

I do not think AI means people will stop writing code altogether. But I do think design driven mainly by "how much hand-written code can we eliminate?" is entering a period where it should be questioned more often.

What matters more now is whether the code is readable, resilient, debuggable, and easy for both humans and AI to work with. If that standard makes some old abstractions look unnecessary, reducing them may be one of the better ways to build stronger libraries and more maintainable platforms from here on.
