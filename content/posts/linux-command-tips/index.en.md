---
title: "Useful Linux tips to know"
date: 2019-05-19
translationKey: "posts/linux-command-tips"
categories: 
  - linux
image: "../../images/linux_terminal.webp"
tags:
  - linux
---
I first encountered MS-DOS when I was in elementary school, and have been using Windows PCs ever since, so I'm not that familiar with Linux. I even thought that `CLI`[^1] had a different sensibility than `GUI`. When I installed CentOS on a virtual machine for the first time, my only impression was that it looked like DOS. However, while using it at work, I found it to be more convenient and faster than I expected. As expected, I think it's true that it boasts the market share of being the most popular server.

So, this time I would like to write about some tips that I learned through my work, even though I only know the basics such as `cd` and `mkdir`. This is also the basics, but there are many things that I thought I wouldn't understand unless I actually used them, and I thought that commands and shortcuts really depend on how you use them, so this is also to organize them.

## Autocomplete with tab key

I think the reason `CLI` is more difficult than `GUI` is that it is based on command input more than anything else. The simple action of moving a file from its original folder to a different folder can be done by simply dragging it with the mouse in the GUI, but in the CLI, it requires typing a command one by one. In the first place, you have to know a command like `mv` to make it work, and depending on the command options you can get completely different results. And the most annoying thing is that you have to enter the path of each file or folder.

And it seems I wasn't the only one who found it inconvenient. I don't know how long this shortcut existed, but it turns out that you can autocomplete it by pressing TAB. This way, it doesn't matter if the alphabet is long or contains spaces. Auto-completion in tab works as follows.

1. Enter the file or folder name
2. Press the tab key while writing
3. Auto-completion completes file and folder names
4. Output a list if there are two or more files or folders

For example, let's say you move to a folder called `/home/retheviper/task01`.

```bash
cd h # press tab here
$ cd home/ # auto-completed
$ cd home/r # press tab here
$ cd home/retheviper/ # auto-completed
$ cd home/retheviper/t # press tab here
task01 task02 task03 # multiple folders start with t, so the list is shown
```

This is a very useful shortcut, so keep it in mind.

## Commands related to folder movement and catalog output

There are commands such as `ls` and `ll` to see what's in the current folder. By adding a few more commands to this command, you can output the contents of different folders. It's obvious, but the method I mainly used to achieve a similar operation is as follows.

```bash
# Check the contents of the current folder
$ ll

# Since there is a /var folder, look inside it
$ cd var
$ ll

# I want to see folders above the starting point
$ cd ..
$ cd ..
$ ll
```

However, with a little application, you can view different folders without changing folders.

```bash
# Use an absolute path
$ ll /var/lib

# Look into a child folder
$ ll ./lib
$ ll lib

# Look into the parent folder
$ ll ../

# Look even further up
$ ll ../../
```

Another option is to simply go back if the path is long and it takes time to get back to the previous position.

```bash
# You are inside a folder with a long path
$ pwd
/var/lib/jenkins/workspace/job01/git_repository/git
# Enter a completely different folder using another long path
$ cd /home/retheviper/todo/task01/awesomeblog
$ pwd
/home/retheviper/todo/task01/awesomeblog

# I want to return to the previous folder
$ cd -
$ pwd
/var/lib/jenkins/workspace/job01/git_repository/git
```

However, please note that when moving folders using `cd -`, you cannot return to just the previous path. If you enter it twice in a row, it will repeat only two passes and go back and forth.

## rsync command for file synchronization

`scp` seems to be the most commonly used file copy on Linux. However, the `rsync` command is more efficient, so I would recommend it. `rsync` can not only simply copy files, but also synchronize folders and files. Another good thing is that if you want to synchronize remote folders, you can copy them without a password if you include public key authentication.

Also, being able to synchronize means that it is also possible to find differences. If a file already exists at the copy destination, performance is better because it compares the copy source file and transfers only the differences. Additionally, if the file is deleted on the source, you can optionally delete the file on the destination as well.

```bash
# Synchronize folders
$ rsync origin destination

# Remove files from the destination when they are deleted from the source
$ rsync --delete origin destination
```

However, as with other commands, the presence or absence of `/` is quite important when specifying the copy target with rsync.

For example, when specifying the copy source folder, writing something like `folder` will copy the entire folder, but specifying something like `folder/` will target the contents under the folder. Also, be careful that if the destination folder does not have permissions or the owner name is different, you will not be able to copy. Permissions can be changed with `chmod` and owner with `chown`.

Additionally, if there are any folders you want to synchronize that should not be deleted, you can also specify them in the options. By including the option `--exclude='folder'`, the specified folder will be excluded from synchronization. This option is useful because it applies to both the source and destination.

## View system specs and status

I had a performance test at work, and I needed to monitor the system status of a server for a specific task. First, I would like to check the machine specs of the server. It seems that the /proc folder contains information used by the kernel.

```bash
# View detailed CPU information
$ less /proc/cpuinfo

# View detailed memory information
$ less /proc/meminfo
```

If you want to see CPU and memory usage in real time, use `vmstat`.

```bash
# Output the current status
$ vmstat

# Output memory usage
$ vmstat -s

# Output disk activity
$ vmstat -d

# Update and output every second
$ vmstat 1
```

This command is worth remembering because it allows you to monitor memory, disk, and CPU usage.

## I want to see long output bit by bit

When outputting certain content using a command such as `ls -al`[^2], the list may be too large to be displayed all on the screen. In this case, if the output of the `more` command fills the entire terminal, pressing the enter key will display the next list.

```bash
ls -al | more
```

There are probably many other useful commands and shortcuts to remember. I think this is the charm of Linux. The more you use it, the more you might like Linux.

Well, that's it for this post. I hope this knowledge will help Linux beginners like me to apply it in real life.

[^1]: I'm more used to the word CUI (I thought it was an abbreviation for Character User Interface), but the correct name seems to be CLI, which is an abbreviation for Command Line Interface.

[^2]: `ll` seems to be a shortcut for `ls -l`. It's convenient because it outputs the approximate content even if you type the same character twice. However, `ls -al` also displays hidden files and folders.
