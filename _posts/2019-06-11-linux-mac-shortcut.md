---
title: "macOSでもショートカットが使いたい"
date: 2019-06-11
categories: 
  - linux
photos:
- /assets/images/sideimage/linux_terminal.jpg
tags:
  - terminal
  - macos
  - linux
---

## お知らせ

macOSがCatalina(10.15)になってから、ターミナルの基本シェルがbashから[zsh](https://ja.wikipedia.org/wiki/Z_Shell)に変わりました。なのでこのポストをそのまま適用すると、追加したコマンドを使えない場合があります。

すでにBashにコマンドを追加した場合は以下の手順で移行ができます。

```bash
cat ~/.bash_profile >> ~/.zprofile
```

新規で追加する場合は、下の「プロファイルを作る」節のコマンドの代わりに以下を入力します。そのあとは同じ手順で進めてください。

```bash
$ vi ~/.zprofile
```

以下、前のポストです。

---

パソコンではMS-DOSとWindowsしか使ってみたことのない私がmacを使ってかれこれ一年となります。macといえばやはり高く、Windowsとは全く違う環境なので使いづらいのではないかなと思っていました。動画や音楽の編集には最適だという話もありましたが、自分とは関係のない世界の話でした。

そんな私がmacを使うきっかけとなったのは、様々な理由がありますが、開発の仕事をしている今は何よりもUnix系のOSということが最大のメリットとなっているのではないかと思います。仕事ではLinuxを触ることが多いのですが、macのターミナルでのほぼ同じことができますので。

ただ、同じくBashを使っているといっても、macとLinuxは完全に同一ではありません。例えばショートカットがそうです。Linuxの便利なショートカットがmacにはないですね。でも調べてみると、やはり方法は存在していました。

今回のポストはそのLinuxのショートカットをmacで使うための設定の話です。といっても、自分が研究した訳ではなく、インターネットで拾ってきた情報にすぎませんがね。でもこの設定をみるだけでも様々なコマンドやAliasを勉強できて、かなり貴重な資料ではないかと思います。誰か知りませんが、作った方には敬意を。

## プロファイルを作る

まずはBashのプロファイルを作ります。ターミナルで以下のように入力します。

```bash
$ vi ~/.bash_profile
```

`bash_profile`と`bashrc`の違いは、前者がターミナルを開くときすぐ適用される設定なら、後者はターミナルで改めてBashを実行した時に適用される設定という違いがあるらしいです。ただコマンドでわかりますが、現在ユーザーのホームにプロファイルを作るのでユーザーが変わる場合はまた同じことをする必要があります。

## プロファイルの内容

以下のコードをプロファイルに入れます。

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

そして`source ~/.bash_profile`を叩き、現在のユーザーに適用すれば終わりです。ユーザーのプロファイルに保存されるので、複数のユーザーを使う場合(rootを含め)はまた同じやり方で適用することを忘れずにしましょう。

## 最後に

`ll`をmacでも使えないかなーという軽い気持ちで調べて得られた情報ですが、まさか`ll | more`みたいなものまでAliasの指定ができるとは知らなかったです。またよく`cd`を`xs`に間違えたりしていますが、それもあえでコマンドとして登録するという発想も斬新ですね。まだまだ知らないコマンドもたくさんありますし、軽い気持ちから始まった作業にしては貴重な情報が得られ他ので嬉しい限りです。これがインターネット時代の恩義というものではないでしょうか。

一部はLinuxでも登録すると便利そうなショートカットが多いように見えるので、Linuxを使われる肩がいらっしゃるのならぜひ一度は試してみてくださいとオススメしたいです。では、楽なBashライフをお楽しみください。