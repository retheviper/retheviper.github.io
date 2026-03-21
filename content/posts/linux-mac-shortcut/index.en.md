---
title: "I Want Shortcuts on macOS Too"
date: 2019-06-11
categories: 
  - linux
image: "../../images/linux_terminal.webp"
tags:
  - terminal
  - macos
  - linux
---

## Notice

Since macOS became Catalina (10.15), the default shell in Terminal changed from `bash` to [zsh](https://en.wikipedia.org/wiki/Z_shell). If you apply this post as-is, some of the added commands may no longer work.

If you have already added commands to Bash, you can migrate with the following step:

```bash
cat ~/.bash_profile >> ~/.zprofile
```

If you are adding things for the first time, use the following command instead of the one in the "Create a profile" section below, then continue with the same steps.

```bash
vi ~/.zprofile
```

Below is the original post.

---

It has been about a year since I started using a Mac, after spending most of my life on MS-DOS and Windows. I thought Macs were expensive and that they would feel completely different from Windows, so I assumed they would be hard to use. I had also heard that they were great for video and music editing, but that felt like someone else's world.

There were many reasons I ended up using a Mac, but now that I work in development, the biggest advantage is that it is a Unix-based OS. I use Linux a lot at work, and I can do almost the same things from the macOS Terminal.

That said, even though both systems use Bash, Mac and Linux are not identical. Shortcuts are a good example. Some of the useful Linux shortcuts are missing on macOS. But after looking into it, I found that there is a way to recreate them.

This post is about configuring macOS so that I can use Linux-style shortcuts there. I did not research this from scratch; I just collected information from the internet. Still, just by looking at these settings, you can learn a lot of commands and aliases, so I think it is a valuable reference. Whoever originally put it together, thank you.

## Create a profile

First, create a Bash profile. Enter the following in Terminal:

```bash
vi ~/.bash_profile
```

The difference between `bash_profile` and `bashrc` is that the former applies settings as soon as you open a terminal, while the latter applies when Bash is launched again from the terminal. Since the profile is created in the current user's home directory, you need to repeat the same thing for each user.

## Profile contents

Put the following code into the profile.

```bash
#.bash_profile
if [ -f ~/.bashrc ]; then
. ~/.bashrc
fi

#============================================================
#
#  ALIASES AND FUNCTIONS
#  Arguably, some functions defined here are quite big.
#  If you want to make this file smaller, these functions can
#+ be converted into scripts and removed from here.
#
#============================================================

#-------------------
# Personnal Aliases
#-------------------

alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
alias cl='clear'
# -> Prevents accidentally clobbering files.
alias mkdir='mkdir -p'

alias h='history'
alias j='jobs -l'
alias which='type -a'
alias ..='cd ..'

# Pretty-print of some PATH variables:
alias path='echo -e ${PATH//:/\\n}'
alias libpath='echo -e ${LD_LIBRARY_PATH//:/\\n}'


alias du='du -kh'    # Makes a more readable output.
alias df='df -kTh'

#-------------------------------------------------------------
# The 'ls' family (this assumes you use a recent GNU ls).
#-------------------------------------------------------------
# Add colors for filetype and  human-readable sizes by default on 'ls':
alias ls='ls -h'
alias lx='ls -lXB'         #  Sort by extension.
alias lk='ls -lSr'         #  Sort by size, biggest last.
alias lt='ls -ltr'         #  Sort by date, most recent last.
alias lc='ls -ltcr'        #  Sort by/show change time,most recent last.
alias lu='ls -ltur'        #  Sort by/show access time,most recent last.

# The ubiquitous 'll': directories first, with alphanumeric sorting:
alias ll="ls -alv"
alias lm='ll |more'        #  Pipe through 'more'
alias lr='ll -R'           #  Recursive ls.
alias la='ll -A'           #  Show hidden files.
alias tree='tree -Csuh'    #  Nice alternative to 'recursive ls' ...


#-------------------------------------------------------------
# Tailoring 'less'
#-------------------------------------------------------------

alias more='less'
export PAGER=less
export LESSCHARSET='latin1'
export LESSOPEN='|/usr/bin/lesspipe.sh %s 2>&-'
                # Use this if lesspipe.sh exists.
export LESS='-i -N -w  -z-4 -g -e -M -X -F -R -P%t?f%f \
:stdin .?pb%pb\%:?lbLine %lb:?bbByte %bb:-...'

# LESS man page colors (makes Man pages more readable).
export LESS_TERMCAP_mb=$'\E[01;31m'
export LESS_TERMCAP_md=$'\E[01;31m'
export LESS_TERMCAP_me=$'\E[0m'
export LESS_TERMCAP_se=$'\E[0m'
export LESS_TERMCAP_so=$'\E[01;44;33m'
export LESS_TERMCAP_ue=$'\E[0m'
export LESS_TERMCAP_us=$'\E[01;32m'


#-------------------------------------------------------------
# Spelling typos - highly personnal and keyboard-dependent :-)
#-------------------------------------------------------------

alias xs='cd'
alias vf='cd'
alias moer='more'
alias moew='more'
alias kk='ll'

#-------------------------------------------------------------
# Using in MAC
#-------------------------------------------------------------

alias desk='cd ~/Desktop'
alias cl='clear'
```

Then run `source ~/.bash_profile` to apply it for the current user. Because the profile is saved per user, remember to repeat the same setup for other users, including root.

## Final thoughts

I originally only looked this up because I wanted to use `ll` on macOS, but I had no idea I could even alias something like `ll | more`. I also kept mistyping `cd` as `xs`, so the idea of registering that as a command was pretty clever. There are still many commands I do not know, and for such a casual search I ended up finding surprisingly useful information. That is the value of the internet, I guess.

Some of these shortcuts look useful on Linux too, so if you use Linux, I recommend trying them out. Enjoy a more comfortable Bash life.
