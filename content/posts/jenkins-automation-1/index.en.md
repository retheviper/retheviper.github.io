---
title: "I want to make everything easier with Jenkins (1)"
date: 2019-05-13
categories: 
  - jenkins
image: "../../images/jenkins.webp"
tags:
  - ci/cd
  - jenkins
---
Automation seems to be the trend in the world. I think AI also ultimately falls into that category of automation. I think it will still be a while before we reach an age where humans lose their jobs to AI.

In that sense, I investigated [Jenkins](https://jenkins.io/). Of course, I wasn't thinking like ``I'm going to use Jenkins to automate all the troublesome things this time and reduce the amount of work that I have to do'', as I'm still a kid who doesn't know what's going on, but it's simply because I use it for work.

## So, what is Jenkins?

I want to know what Jenkins is in the first place. If you use it for work, there must be some reason. Also, the middleware used in this industry is probably used because it has some kind of convenience (it saves time and effort). So what is so useful about Jenkins that you would want to use it? I looked into it from that perspective.

## Continuous Integration

Even though you said that, I felt like... In other words, it seems to be a tool that does things like build, test, and verify. At first, I was told that it would be used in conjunction with Git, so I was already using Git for version control, so is there any benefit to using it in conjunction with other tools? I thought, but I realized that it's good because this behavior called "continuous integration" can be automated.

Even if you use Git or Subversion for version control, you won't know who pushed and when until you look at the logs. Also, if you find out that you have pushed something, you have no choice but to manually test it and merge it again. This problem can be solved using Jenkins. For example, when someone pushes, a notification is sent to Jenkins (although apparently this requires separate work), it pulls, builds, and tests with JUnit, and then notifies you of the results. Also, depending on the settings, it seems to be an amazing tool that can also deploy [^1] once the test is successfully completed.

## But you won't know until you try it

Let's start by installing it to experience the amazing automation of Jenkins. The environment I use for work is Linux. I thought it would be possible to install it with yum right away, but that doesn't seem to be the case.

When I connected to the [Jenkins homepage](https://jenkins.io/), there were instructions for installing it on Linux. Fortunately, the Linux I use at work is AWS, and I was able to install it using the same steps as RedHat Linux[^2].

## Now let's install

First, bring the Jenkins repository.

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
```

Here wget -O seems to be an option to read the folder and output it as a file. You can learn a lot here too.

```bash
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```

rpm is a command used when installing a package, but here we use the --import option to hold the key and verify it.

Once this is done, you can install it using yum just like any other package.

```bash
yum install jenkins
```

The installation is now complete. If there are no problems, just configure the port and start it up.

(continued)

[^1]: Place it on another server, etc. For example, the development environment and the production environment need to be separated, so once the operation has been verified in the development environment, it is placed in the production environment and executed.

[^2]: Apparently it can be done in the same way on Fedora and CentOS. We tested it on CentOS7.
