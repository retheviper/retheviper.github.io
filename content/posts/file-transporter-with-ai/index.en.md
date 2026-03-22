---
title: "Revisiting FileTransporter with AI"
date: 2026-03-22
translationKey: "posts/file-transporter-with-ai"
categories:
  - recent
image: "../../images/tools.webp"
tags:
  - kotlin
  - ktor
  - compose
  - gradle
  - ai
---

I recently made a fairly large pass over [FileTransporter](https://github.com/retheviper/FileTransporter). It was originally a small app I built for personal use as a simple file browser server, and after the core features were in place, it ended up sitting around for a long time with very little attention.

That tends to happen with this kind of project. There are always many things I want to try, but before I can even get to the interesting part, I get blocked by environment setup, architecture review, bug fixing, and all the surrounding investigation. Once that happens, the project quietly drops in priority again. FileTransporter had become exactly that kind of backlog.

What changed this time was that coding agents have become practical enough that modifying an existing codebase now feels much less heavy than it used to. I decided to use that opportunity not only to change code, but also to revisit the surrounding guidance, including things like `AGENTS.md`, and treat this as a chance to clean up accumulated technical debt in a more serious way.

## The work that had been waiting for a while

The actual changes included external YAML configuration, a reworked file browser and upload flow, ZIP downloads for directories, stronger path validation, cleaner service wiring with Koin, regression tests on both the frontend and backend, and some Gradle cleanup.

As a list, it can sound like a burst of new features. In reality, most of it was work I had already known I should do. It felt less like inventing something new and more like finally dealing with a pile of delayed cleanup and structural debt.

That is especially common in personal projects. If the visible feature works, it is easy to leave configuration, boundaries, and tests for later. But when you come back after some time, those are often the exact things that make the code uncomfortable to touch again.

## Why it was possible this time

The biggest difference was that coding agents lowered the entry cost of doing this kind of cleanup.

Moving config into YAML, splitting state, breaking UI into smaller pieces, adding regression tests, and cleaning the build are not especially difficult in isolation. The real problem is that they require rereading the code, reasoning about impact, keeping the shape of the changes consistent, and checking for gaps. That kind of work is tedious enough that it often gets postponed even when the need is obvious.

AI helped by making those first steps lighter. It was useful for drafting structure changes from the existing code, comparing cleanup directions, surfacing missing test angles, and breaking refactors into smaller units before touching everything at once.

Of course, it still required judgment. I still had to decide what to prioritize, how far to push a refactor, and which structure actually made sense for the project. But the difference was that the work no longer felt blocked by a large initial setup cost.

## It helped me revisit easy-to-miss details

One thing that turned out to be more valuable than I expected was how much this process helped me recheck things that are easy to overlook. Even in areas I already knew were important, such as path validation, working through the code with AI made it easier to stop and ask whether a part that looked "probably fine" had actually been reviewed carefully enough.

That kind of second pass is easy to skip when you are alone with familiar code. Older personal code is especially dangerous in that way, because it feels familiar enough that you can mistake that familiarity for confidence. This time, I was able to revisit those parts more deliberately, including safety checks and error handling that could easily have stayed half-reviewed.

In that sense, the value was not just speed. It was also the chance to look again at parts I might otherwise have waved through.

## It also pushed me to refine implementation rules

Another interesting part was that using AI in the refactoring loop pushed me to revisit the implementation guidelines themselves.

Questions like where responsibilities should be split, which layer should own state, and how much testing a change should require are usually present in the background, but they can become blurry when returning to an old project. Since I was not only editing code but also revisiting guidance around the repository, including `AGENTS.md`, the work naturally expanded into clarifying how I want this project to be maintained going forward.

So AI was not only acting as a code assistant. It also worked as a kind of feedback partner for the implementation. Explaining why a responsibility should be separated, why a test belongs at a certain level, or why a rule should exist forced me to restate those standards more clearly. That turned out to be one of the most useful outcomes of the process.

## What actually changed

On the configuration side, storage settings and upload limits now live in `config/application.yaml`, with system property overrides when needed. That alone makes the application easier to run and adjust.

On the frontend, the file browser was reorganized, the upload and download flow became cleaner, and the UI gained a brighter theme and a dark mode toggle. The screen was also split into clearer pieces such as the header, upload panel, details modal, and transfer history.

On the backend, path confinement was tightened so file operations stay inside the configured root, and responses around invalid targets became clearer. Koin was also introduced to clean up service wiring, and transfer state and server bootstrap responsibilities were separated further.

And perhaps most importantly, I finally added regression tests across both frontend and backend areas. That was exactly the kind of work I had known I needed, but had kept postponing.

## Finally

This FileTransporter update was less about adding a few new capabilities and more about making an old personal app maintainable again. In that process, AI was useful not only as a way to write code faster, but as a way to start the cleanup I had been putting off, revisit details I could easily have missed, and refine the implementation rules behind the code itself.

I suspect this will become a much more common pattern in my personal projects. Work that once felt too heavy to even begin now feels much easier to re-enter, which may be the most meaningful change of all.
