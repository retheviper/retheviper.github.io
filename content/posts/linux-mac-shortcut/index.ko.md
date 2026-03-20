---
title: "macOS에서 단축 명령 쓰기"
date: 2019-06-11
categories:
  - linux
image: "../../images/linux_terminal.webp"
tags:
  - terminal
  - macos
  - linux
---

## 공지

macOS가 Catalina(10.15)로 올라가면서 기본 셸이 bash에서 [zsh](https://ja.wikipedia.org/wiki/Z_Shell)로 바뀌었습니다. 그래서 이 글을 그대로 적용하면 추가한 명령이 바로 먹지 않을 수도 있습니다.

이미 Bash에 명령을 추가한 경우에는 아래 명령으로 옮길 수 있습니다.

```bash
cat ~/.bash_profile >> ~/.zprofile
```

새로 추가할 경우에는 아래 "프로필 만들기" 절의 명령 대신 다음을 입력하면 됩니다. 그 뒤에는 같은 순서로 진행하면 됩니다.

```bash
vi ~/.zprofile
```

이하 내용은 예전 글입니다.

---

예전에는 MS-DOS와 Windows만 써 보던 제가 Mac을 사용한 지도 벌써 1년이 넘었습니다. Mac 하면 일단 비싸고, Windows와는 완전히 다른 환경이라 쓰기 불편하지 않을까 생각했습니다. 영상이나 음악 편집에는 최적이라는 이야기도 있었지만, 그건 저와는 별 상관없는 세계라고 여겼습니다.

제가 Mac을 쓰게 된 이유는 여러 가지가 있지만, 개발 일을 하게 된 지금은 무엇보다 Unix 계열 OS라는 점이 가장 큰 장점입니다. 업무에서는 Linux를 자주 만지는데, Mac의 터미널도 그와 비슷한 작업을 거의 그대로 할 수 있기 때문입니다.

다만 같은 Bash를 쓴다고 해도 Mac과 Linux가 완전히 같은 것은 아닙니다. 예를 들어 단축 명령이 그렇습니다. Linux에서 편하게 쓰는 단축 명령이 Mac에는 없는 경우가 있습니다. 그런데 찾아보니 역시 방법은 있었습니다.

이번 글은 그런 Linux용 단축 명령을 Mac에서 쓰기 위한 설정 이야기입니다. 제가 직접 연구한 것은 아니고 인터넷에서 찾은 정보에 가깝지만, 이런 설정만 봐도 여러 명령과 별칭을 배울 수 있어서 꽤 귀중한 자료라고 생각했습니다. 만든 분께 감사할 따름입니다.

## 프로필 만들기

먼저 Bash 프로필을 만듭니다. 터미널에서 아래처럼 입력합니다.

```bash
vi ~/.bash_profile
```

`bash_profile`과 `bashrc`의 차이는, 전자는 터미널을 열 때 바로 적용되는 설정이고, 후자는 터미널에서 Bash를 다시 실행할 때 적용되는 설정이라고 보면 됩니다. 또한 이 프로필은 현재 사용자 홈에 만들어지므로, 사용자가 바뀌면 다시 설정해야 합니다.

## 프로필 내용

아래 코드를 프로필에 넣습니다.

```bash
# .bash_profile
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
# Personal Aliases
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
# Add colors for filetype and human-readable sizes by default on 'ls':
alias ls='ls -h'
alias lx='ls -lXB'         # Sort by extension.
alias lk='ls -lSr'         # Sort by size, biggest last.
alias lt='ls -ltr'         # Sort by date, most recent last.
alias lc='ls -ltcr'        # Sort by/show change time, most recent last.
alias lu='ls -ltur'        # Sort by/show access time, most recent last.

# The ubiquitous 'll': directories first, with alphanumeric sorting:
alias ll="ls -alv"
alias lm='ll |more'        # Pipe through 'more'
alias lr='ll -R'           # Recursive ls.
alias la='ll -A'           # Show hidden files.
alias tree='tree -Csuh'    # Nice alternative to 'recursive ls' ...

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
# Spelling typos - highly personal and keyboard-dependent :-)
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

그다음 `source ~/.bash_profile`를 실행하면 현재 사용자에게 적용됩니다. 프로필은 사용자별로 저장되므로, 여러 계정을 쓴다면 root를 포함해 각각 같은 작업을 해 줘야 합니다.

## 마지막으로

`ll`을 Mac에서도 쓸 수 없을까 하는 가벼운 생각으로 찾아본 정보였는데, `ll | more` 같은 별칭까지 지정할 수 있다는 걸 처음 알았습니다. 또 `cd`를 `xs`로 잘못 치는 경우가 많은데, 그걸 아예 명령으로 등록해 버리는 발상도 꽤 재미있었습니다. 사소한 설정 같아 보여도, 자주 쓰는 명령을 손에 맞게 바꾸는 것만으로 작업 흐름이 꽤 편해집니다.

일부는 Linux에서도 그대로 쓰기 좋은 별칭이니, 비슷한 환경을 쓰고 있다면 한 번쯤 적용해 볼 만합니다.
