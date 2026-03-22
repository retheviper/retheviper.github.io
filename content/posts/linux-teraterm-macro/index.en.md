---
title: "Using Tera Term"
date: 2019-06-24
translationKey: "posts/linux-teraterm-macro"
categories: 
  - linux
image: "../../images/linux_terminal.webp"
tags:
  - terminal
  - linux
---

I had used PuTTY and the Mac Terminal before, and even MS-DOS if you count the CLI era, but I had never used [Tera Term](https://ttssh2.osdn.jp/). At work, though, we run Linux servers on AWS, so SSH connections are required. The first thing I made for that was a Tera Term macro that could handle the SSH login. Until then I had mostly been doing document work, so I was happy to finally do something that felt like coding.

There are probably many other good tools, but it is convenient that macros can easily issue shell commands and that the screen can be customized without much effort. Windows 10 now has a terminal and WSL exists, but if the PC your company gives you is not running Windows 10, this is still a useful option.

Here is how I designed the macro and implemented it in code.

## Macro design

I am not sure this is true for every AWS EC2 setup, but in the project I worked on, we connected to EC2 servers through a so-called "bastion server." I did not even know the concept at first, so I wondered why the setup was structured that way, but apparently it is necessary because each EC2 instance is not directly connected to the internet. So the macro first connects to the bastion server and then issues SSH commands to connect to the target EC2 instances.

This time we had three destinations: a batch server, a Web API server, and a CI server, so I wanted to connect to each of them through the bastion host. I also wanted to customize the terminal window a little for each server, since I might work on multiple servers at once. After some research, I learned that the macro can contain the connection steps, and then it can load an INI file where the window settings were already prepared. The CI server also needed port forwarding for services like Jenkins, and that setting would be stored in the same INI file, so I wanted to load a different INI file only when connecting to the CI server.

Summarizing the requirements, the macro works like this:

1. Select a destination from a list of three servers.
2. Connect to the bastion server first, then SSH into the selected server.
3. Once connected, load the INI file for that server.

I prepared the INI settings in advance through the UI, including the terminal, window, font, and port forwarding settings, and created three files for the three destinations. The macro just loads them. Here is the actual macro code.

## Macro code

```bash
;==============================================
;; Bastion user ID / password
;==============================================
USERNAME = 'bastion user id'
PASSWORD = 'bastion password'

;==============================================
;; Bastion server IP address
;==============================================
HOSTIP = 'enter IP here'

;==============================================
;; Work user IDs by destination
;==============================================
strdim WORKUSERLIST 3
WORKUSERLIST[0] = 'batch server user id'
WORKUSERLIST[1] = 'web app server user id'
WORKUSERLIST[2] = 'CI server user id'

;==============================================
;; Work user passwords by destination
;==============================================
strdim WORKPWLIST 3
WORKPWLIST[0] = 'batch server password'
WORKPWLIST[1] = 'web app server password'
WORKPWLIST[2] = 'CI server password'

;==============================================
;; Server IPs
;==============================================
strdim SERVERipLIST 3
SERVERipLIST[0] = 'batch server IP'
SERVERipLIST[1] = 'web app server IP'
SERVERipLIST[2] = 'CI server IP'
 
;==============================================
;; Server names shown in the list
;==============================================
strdim SERVERnameLIST 3
SERVERnameLIST[0] = 'batch server'
SERVERnameLIST[1] = 'web app server'
SERVERnameLIST[2] = 'CI server'
 
;==============================================
;; Server-specific INI file
;==============================================
strdim INILIST 3
INILIST[0] = '/BatchServer.INI'
INILIST[1] = '/WebAPServer.INI'
INILIST[2] = '/CIServer.INI'

;==============================================
;; Host selection screen
;==============================================
listbox 'Select a server' 'OK' SERVERnameLIST
if result >= 0 then
    SERVERIP = SERVERipLIST[result]
    WORKUSER = WORKUSERLIST[result]
    WORKPASSWORD = WORKPWLIST[result]
    INIFILE = INILIST[result]
else
    end
endif
 
;==============================================
;; Load the INI file path
;==============================================
getdir INIPATH
strconcat INIPATH INIFILE

;==============================================
;; Build and execute the command to connect to the bastion server
;==============================================
PROXY = '-proxy=http://proxy.server.com:6000'
COMMAND = PROXY
strconcat COMMAND ' '
strconcat COMMAND HOSTIP
strconcat COMMAND ':22 /ssh /auth=password /user='
strconcat COMMAND USERNAME
strconcat COMMAND ' /passwd='
strconcat COMMAND PASSWORD
connect COMMAND

wait '$'

;==============================================
;; SSH connection processing by destination
;==============================================
SSHCOMMAND = 'ssh '
strconcat SSHCOMMAND WORKUSER
strconcat SSHCOMMAND '@'
strconcat SSHCOMMAND SERVERIP
sendln SSHCOMMAND

;==============================================
;; Initial SSH login processing
;==============================================
wait 'Are you sure you want to continue connecting (yes/no)?' "'s password: "
if result = 1 then
    sendln 'yes'
    wait "'s password: "
elseif result = 2 then
    goto INPUTPWD
endif

:INPUTPWD
sendln WORKPASSWORD
wait '$'

sendln 'sudo su -'
wait 'sudo'

sendln WORKPASSWORD
wait '#'

restoresetup INIPATH
;==============================================
;; Macro end
;==============================================
end
```

## How it works

The macro first builds an SSH connection command using the bastion user ID, password, and IP address, then connects to the bastion host. I used `strconcat` to join strings and issue the command. In the real project I also used a proxy server, so I included that in the bastion connection command; if you do not need one, just remove that part. Because the main purpose is connecting to a different server after the bastion hop, the macro starts by showing a list of the three destination servers as soon as it launches.

I prepared arrays (`strdim`) for the user ID, password, IP address, and displayed server name for each destination. Then I show a selectable list with `listbox`. When you choose a destination and press the confirm button, the result is stored in `result` as a number, which I use as the array index. The macro then concatenates the strings just like in the bastion connection command and performs the SSH connection. I also prepared INI filenames in arrays for each destination, got the macro's relative path with `getdir`, combined it with the INI filename, and then loaded the INI file with `restoresetup`. (This assumes the macro file and the INI file are in the same folder.)

When issuing commands, `wait` is used to wait for the terminal's response. It works the same way as `expect` in shell scripts. This prevents the macro from issuing commands too quickly and getting out of sync with the server's response. By combining `wait` and `if`, the macro can also handle the first-connection prompt that asks whether to continue connecting. I do not think there is anything else particularly difficult here, so that is enough explanation.

Save the macro with a `.ttl` extension and either load it from Tera Term or run it directly from the macro runner. (You can configure Tera Term during installation so macros can be launched directly.) It is simpler than it sounds.

## Final thoughts

By using `if` and `wait`, you can issue almost any command, so in a sense this macro behaves a bit like a shell. In this example it logs in all the way to root, but you could also change the working directory or run a shell script. Structurally, it is just a terminal that issues commands.

That is all for this post. For an easier SSH life, I recommend giving Tera Term a try.
