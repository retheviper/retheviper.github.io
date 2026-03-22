---
title: "Creating a Linux System Service"
date: 2019-07-11
translationKey: "posts/linux-systemctl-service"
categories: 
  - linux
image: "../../images/linux_terminal.webp"
tags:
  - linux
---

It is obvious that programs running on a server behave differently from ordinary end-user programs. That is partly about what data they hold and what they do, but here I want to focus on execution. In simple terms, how do you make a Linux program keep running until someone stops it?

On CentOS and RHEL, there is a `service` command. When you install various programs with package managers such as `yum`, some of them are not meant to run just once. They need to stay resident in memory. Jenkins, which I introduced on this blog before, is one of those programs.

Programs like that can usually be started and stopped automatically with `service` after installation, but can you do the same with a program you wrote yourself? Linux lets users do almost anything, so it seemed possible.

I happened to need exactly that at work: I had to register and manage a Java application as a service. Since I was handling Jenkins jobs, the real task was to pull, build, stop the running Java application, and start the newly built one when commits landed in Git. In practice I did not need to touch the service itself directly; I just wired Jenkins to run shell commands that stopped and restarted the service after the build. Still, I got curious about how it all worked, so I looked into it.

## Creating a Service Daemon

A service is also called a daemon. When I looked it up, I found definitions like "a program that stays resident on the system and automatically runs when a certain state is reached," "a program that keeps running to process periodic service requests," and "something running in the background." That gives a pretty good sense of what kind of program we are talking about. Web server programs written in `Spring` or `Node.js` fit that definition.

So, how do we actually make such a service or daemon? The main ingredients are the program you want to register as a service, a service configuration file describing what kind of service it should be, and commands to register and run it. At work, we made a `Spring Boot` application into a service, but that also required shell scripts and external file references, so I will use a relatively simple `Node.js` example here.

With `Node.js`, you can run something as simple as `$ node /node/index.js`. Registering that as a service looks like this.

## Creating the service file

First, create a file that describes what the service should do and what it should be called.

```bash
vi /etc/systemd/system/NodeServer.service
```

Fill it in like this:

```bash
# Service configuration
[Unit]
Description = NodeServer # The service is registered under this name
After = syslog.target network.target # Start after syslog and network during boot

# Program configuration
[Service]
Type = simple # Defines the service type; simple is the default
ExecStart = /usr/bin/node /node/index.js # Same as running node /node/index.js, but written without symbolic links
Restart = on-failure # Restart on failure
User = nodeservice # User to run as; watch permissions

# Symbolic link and alias settings
[Install]
WantedBy = multi-user.target # Specifies where the symbolic link is created; this is the common choice
```

There are many other options, but that should cover the basics. The Node.js web server I prepared for testing worked with this setup. It only printed `Hello Node.js!`, but it was enough.

The Java application I worked with in practice also specified scripts for managing the PID[^1] and for stopping the process. For example, `ExecStop` lets you define the command you want to run when repairing a service, and `ExecReload` lets you define what should happen when the service is reloaded. Those options are useful when you need to run a shell script or take some action when the state changes.

Once the service file is ready, the next step is to register the process as a system service and run it.

## Enable & Start Service

I mentioned the `service` command earlier, but from CentOS 7 onward you are supposed to use `systemctl`.[^2] In practice, `service` alone cannot register system services on CentOS 7, so `systemctl` is the command you use. It can register and unregister services, start and stop them, and restart them. When a service is registered as a system service, it starts automatically when the system boots.

Compared to Windows, it is like combining "Startup" registration/removal with process management through Task Manager.

Here are the commands, since they are straightforward.

```bash
# Register as a system service
$ systemctl enable NodeServer
$ systemctl start NodeServer

# Search among running services
$ systemctl list-units | grep NodeServer
```

```bash
# Reload after modifying the service file
$ systemctl daemon-reload

# Stop and restart
$ systemctl stop NodeServer
$ systemctl restart NodeServer

# Check service status
$ systemctl status NodeServer
# Remove from system services
$ systemctl enable NodeServer

# If there is a problem, you can inspect the service log with the following command
$ journalctl -xe
```

Even if you cannot register or unregister services, you can still control them with the `service` command. But since that command may disappear eventually, it is probably better to get used to `systemctl`.

```bash
# What you can do with service
$ service NodeServer start
$ service NodeServer status
$ service NodeServer stop
$ service NodeServer restart
```

With this, the simple Node.js web server I made became a system service and kept running without stopping. Problem solved. This is the kind of knowledge that could also be useful if I ever need to build another automation program.

That is it for this post. See you again!

[^1]: Short for Process ID, the ID assigned to a running process. You can check it with `ps`, and use it to look up a process name or vice versa.
[^2]: `service` still exists on CentOS 7, but it seems to be redirected to `systemctl`. Until CentOS 6, services were managed from `/etc/rc.d/init.d`, and from version 7 onward they became service units.
