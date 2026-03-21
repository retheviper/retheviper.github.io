---
title: "This year's blog story"
date: 2022-02-06
categories: 
  - recent
image: "../../images/magic.webp"
tags:
  - jekyll
  - hugo
---
Some of you may have already noticed, but I've changed the theme of my blog for the first time in two years. To be more precise, I changed not only the theme but also the static page generation tool from [Jekyll](https://jekyllrb.com/) to [Hugo](https://gohugo.io/). Now I'm really glad that I published my blog on Github Pages from the beginning because it's easy to do things like this.

Due to the change in the site generation tool, the URL of each post has also changed, and I am sorry for those who have marked it as a favorite, but even with that in mind, I think there are many things that could have been improved, so please understand.

Well, this time we will be talking about the renewal of the blog, but I would like to broadly explain ``how the blog has changed'' and ``what I want to do with the blog from now on''.

## UX improvements

Improvements in terms of UX were also the number one goal of the blog revamp. I'm sure there are many other things, but here are some things to start with:

## Improved screen transition

Previous blogs included animations when transitioning from the main to individual posts. (The same thing was used to display the list of posts) Inserting animations during screen transitions is already an old trend, and above all, I felt that screen transitions were slow, so I wanted to improve it. So this time I tried to react faster.

## Search function

I thought tags, categories, and archives would be enough, but there are times when you want to search for posts by keyword. Previously, I tried adding a search function but it didn't work, so I decided to use a theme that allows for proper search.

## Design

Personally, I like dark mode and thought I would choose a black theme overall, but luckily there was a theme that allows you to switch to dark mode with the press of a button, so I chose this. What's even better is that this dark mode is tied to your system settings. So, if you don't like dark mode, you can see a white screen. One disappointing thing is that the style of code blocks is different from that of blogs...I would like to address this later if there is a way to do it.

Other nice things about it are that the images look good even on mobile screens, and the menus and layouts have a modern design.

## Post display

Previously, when you clicked on a post image from the list of posts, the image would be enlarged instead of going to the details screen for that post, but this has been resolved. Also, since it is now possible to display the rough reading time for each post, I think we are able to provide good information to those reading the article to help them decide which ones to read.

## Internal changesNow, up until now, we have been talking mainly from a UX perspective, but in fact, the items listed above can be addressed simply by changing the theme of Jekyll. I would like to tell you now why I decided to change it to HUGO and pursue a new blog.

## Easy to manage

In the case of Jekyll, one theme handled one application written in Ruby, so it was necessary to update Gemfile dependencies, it took time to start the server locally, and it was necessary to tinker with various settings and configurations every time you changed the theme. In the case of HUGO, there were almost no such problems. Themes can import Github repositories as submodules, and all you have to do is modify the content and basic settings. So, I think it actually took about an hour, excluding things like dealing with changes in resource paths such as attached images.

Also, previously, images were collected in one folder, so managing the images attached to posts was quite a tedious task, but now each post uses a separate folder, and all you have to do is put the images in that folder, making management easier. So, from now on, I will be more proactive about attaching images than before.

Since HUGO is written in Go, starting a server locally is fast.

## Easy to customize

Previously, there was a function that automatically created an RSS feed, but it did not target all posts. I think the problem could be resolved by tweaking the theme or Jekyll settings, but as I mentioned earlier, a theme is like an app, so customization is not easy. In comparison, HUGO is simpler and allows you to easily customize themes and settings. Even now, some of the menus can be customized by simply changing the theme settings, but there are also some that I have modified myself. It didn't take much time at all, and it was appealing to me that I could do it right away.

Personally, I think the appeal of HUGO is that you can use [shortcode](https://gohugo.io/content-management/shortcodes/) to create templates for HTML and JavaScript attachments. It's easy to use and can be used in many different ways.

## From now on

Now that I've talked about how I physically improved by changing the blog generation tool, I would like to talk about what I want to do with that blog.

At first, I think I was full of enthusiasm and wrote a variety of articles about what I learned and felt while experiencing various technologies, as well as trial and error. Looking back now, there are moments when I feel embarrassed and think, ``There were so many things I didn't understand back then,'' but at least I feel like I was putting more effort into it than I am now.Personally, I aim to update this blog at least twice a month, but I feel that I've gotten ahead of myself and written some posts that aren't really worth reading.

So this year, I've set a goal of actually using some of the techniques, and I'd like to write articles about them. At the moment I'm thinking of the following:

## Jetpack Compose

It's something I've set as a New Year's goal, but I've been dabbling with it little by little since last year, and this year I'd like to actually create an Android and desktop application. Just last year, [Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/) was also officially released, so I think the timing is perfect.

I'm also interested in SwiftUI due to the announcement of [XCode Cloud](https://developer.apple.com/documentation/Xcode/About-Continuous-Integration-and-Delivery-with-Xcode-Cloud), [the ability to publish apps via links](https://developer.apple.com/support/unlisted-app-distribution/), and the ability to build apps with [Swift Playgrounds](https://www.apple.com/swift/playgrounds/), but I think it would be better to try it after I use Kotlin at work and have a certain level of proficiency with Compose.

Since I'm using a Mac, I think I should try creating an app with Swift at least once.

## Svelte

My company uses [Nuxt.js](https://nuxtjs.org/), so I thought it would be fine, but in the end I felt like I would only have a chance to touch the screen in private, so I chose [Svelte](https://svelte.dev/) out of curiosity.

I don't think it's a mature technology yet, but things like [Sveltekit](https://kit.svelte.dev/) are coming out soon, and when I thought about the time it takes to learn and the efficiency, I thought it would be the most productive technology if I were to make screens myself. Well, you won't know until you actually touch it...

Another reason is that it was selected as the most loved web framework in the [Stackoverflow Survey](https://insights.stackoverflow.com/survey/2021#section-most-loved-dreaded-and-wanted-web-frameworks). When it comes to technologies that many engineers like, the first thing you want to know is why. (I'd also like to try Rust for the same reason.)

## Quarkus

I want to shorten the time it takes to build, test, and deploy the Spring boot that my company uses, and I'm considering migrating to Quarkus as a countermeasure. Changing the framework of a running service is quite risky, but if it's successful, I feel like it will increase productivity and have significant benefits such as startup speed and memory, so I'm not sure when it will happen, but I've set it as a task I'd definitely like to try. Whether the migration is successful or unsuccessful, you will likely learn a lot from it.

## Finally

The appearance of the blog has changed, and I think it has been a successful transition, so the next challenge is to improve the content. As always, I hope to keep in mind my desire to become a full-fledged engineer and write posts that will help me move forward little by little (and be helpful to those who read it).

See you soon!
