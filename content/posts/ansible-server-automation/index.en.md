---
title: "Build a server with Ansible"
date: 2019-07-05
translationKey: "posts/ansible-server-automation"
categories: 
  - ansible
image: "../../images/ansible.webp"
tags:
  - ansible
  - yaml
  - linux
---

This time I had a chance to try using [Ansible](https://www.ansible.com/) a little. `Ansible` is also an automation tool, similar to Jenkins in the sense that it can be applied in multiple environments by specifying tasks in advance. The difference is that Jenkins is mainly specialized in automating deployment, release, testing, etc., whereas Ansible is aimed at automatically building servers.

In other words, if you use Ansible, you can set up multiple environments in the same way. For example, in my experience, the flow was ``development'' → ``inner connection'' → ``outer connection,'' and the environment used for each was different. It would be a waste of time to build the same server every time a stage changes, and as the number of settings increases, human errors can occur in various ways, so I wanted to automate this task with Ansible.

After a little digging, Ansible uses a YAML file called `Playbook` to configure the server. It seems that you can do more complicated movements as you go deeper, but basically it starts with YAML that stores the connection destination information (hosts file) and setting values. You can create folders and users, install programs with `yum`, execute shell commands, transfer files, etc., and if you set it up properly, it seemed like it could be used effectively when distributing multiple servers.

However, since the YAML file and folder structure are a little complicated, I will explain the settings for each file one by one. The following YAML settings are written based on AMAZON Linux.

## Installing Ansible

Ansible can be installed with `yum`, `brew`, `pip`, `apt-get`. It's easy! However, in some cases Python2 or PIP may be required, so be sure to install them in advance.

```bash
yum install ansible
$ brew install ansible
$ pip install ansible
$ sudo apt-get install ansible
```

Install using the method appropriate for your environment.

## batchserver.yml

First, generate a YAML file for each server. This is used when Ansible is executed, and describes what kind of operation will be performed on which server. The idea is to prepare separate YAML files for what you want to do for each server, and create YAML files for the common, batch server, and WEBAP server.

The code below is an example assuming a batch server.

```yaml
- hosts: batchserver
  become: true
  roles:
    - common
    - batch
```

`hosts` means the host written in the hosts file. If you write `batchserver` here, when Ansible is executed, it will automatically connect to all servers belonging to the group called batchserver from the hosts file and perform the same operation. If you want to execute it for all groups, write `all`.

If you set `become` to `true`, all commands at the connection destination will be executed as sudo. `roles` describes the YAML file that specifies the actions to be performed on the server, and in my case, I separated `common`, which describes things that are commonly executed on any server, and `batch`, which I want to execute only on batch servers, so I wrote both.

## hosts

As mentioned above, this file describes the connection destinations that you want to automatically configure with Ansible.

```text
[batchserver]
192.168.0.1 ansible_ssh_user=batchuser1
192.168.0.2 ansible_ssh_user=batchuser2

[webapserver]
192.168.10.1 ansible_ssh_user=webapuser1
192.168.10.2 ansible_ssh_user=webapuser2
```

Basically, if you specify the groove and write the host information, it will be automatically classified. In the YAML file written above, only `batchserver` is specified, so the `webapserver` group will be ignored at runtime. As the name suggests, `ansible_ssh_user` specifies the user name used when connecting with SSH with Ansible. Of course, you can also enter your username and password at runtime without doing this.

## roles/batch/tasks/main.yml

This is `roles` specified in `batchserver.yml`, and it contains a YAML file that describes what you actually want to do. Please note that `roles` can only be executed if it is created under the alliance folder.

This is a file where you can write what you really want to do, so please use it as a reference to see what actions you can specify.

```yaml
  - name: Create user groups
    group:
      name: "{ { item.group_name } }" # Spaces are shown here only because of Markdown formatting
      gid: "{ { item.group_id } }"
    with_items:
    - { group_name: 'group01', group_id: '101' }
    - { group_name: 'group02', group_id: '201' }

  - name: Create users
    user:
      name: "{ { item.user_name } }"
      password: "{ { item.user_passwd } }"
      uid: "{ { item.user_id } }"
      group:  "{ { item.user_group } }"
      shell: /bin/bash
    with_items:
    - { user_name: 'user01', user_group: 'group01', user_id: '101', user_passwd: 'user01' }
    - { user_name: 'user02', user_group: 'group02', user_id: '201', user_passwd: 'user02' }

  - name : Create folders
    file:
      path={ { item } }
      owner=user01
      group=user01
      mode=0755
      state=directory
    with_items:
    - /user01

  - name: Package install by yum
    yum:
      name: "{ { packages } }"
    vars:
      packages:
        - python2-pip
        - postgresql
        - postgresql-devel
 
  - name: Upgrade pip by shell command
    shell: bash -lc "pip install --upgrade pip"

  - name: Install python modules
    pip:
      name: "{ { item } }"
      executable: pip
    with_items:
      - cx_Oracle
      - psycopg2
      - boto3
      - paramiko

  - name: Copy files
    copy:
      src= { { item.source } }
      dest= { { item.dest } }
      owner=root
      group=root
      mode=0755
    with_items:
      - { source: etc/somefile.zip, dest: /etc/somefile.zip }
```

Starting from the top, create a user group, create a user, create a folder, install a package with `yum`, execute a shell command, and transfer files. In the end, it's like connecting via SSH and running a shell script. However, I think the good thing about it is that it can be easily configured using a YAML file. You can run it over and over again.

However, after connecting via SSH, the commands are read line by line from the top of the YAML file and executed, so you need to be careful about the order in which you want to execute them. For example, it is obvious, but please note that you cannot create a user in a specific group before creating a user group.

## roles/batch/files

Place the files you want to use for transfer in this folder. For example, if you want to transfer `etc/somfile.zip` written in `main.yml` above, place a file with the same path under this folder. Of course, you can transfer multiple files or separate them into different folders.

## roles/common/tasks/main.yml

This file collects commands that are commonly executed on any server.

```yaml
  - name: Upgrade all packages by yum
    yum: name=* state=latest

  - name: Install openjdk 11
    shell: bash -lc "amazon-linux-extras install java-openjdk11"

  - name: Correct java version selected
    alternatives:
      name: java
      path: /usr/lib/jvm/java-11-openjdk-11.0.2.7-0.amzn2.x86_64/bin/java
```

What this file does is update all packages with `yum` and install OpenJDK. If you just install JDK, the basic version when running Java on the server will not be OpenJDK11, so we have included a part to select the Java version from Alternative. You can also install Python3 in the same way and specify the basic execution version with Alternative.

At this point, the basic settings using Ansible are complete. It's not difficult! (The deeper you go, the more difficult it will be.)

## execute

Now that the playbook is prepared, let's run it. It can be executed with the following command.

```bash
# Standard run
$ ansible-playbook server.yml -i hosts
# For dry runs (syntax check for the playbook)
$ ansible-playbook server.yml -i hosts -C
# When specifying the SSH connection user name
$ ansible-playbook server.yml -i hosts -u hostuser
```

This is also easy. During execution, you may be asked for the password of the user making the SSH connection, but this can be avoided by registering the public key of the user making the SSH connection in advance. In the case of `sudo`, it is convenient to set `visudo` to `NOPASSWD` at the connection destination.

## lastly

How was it? Lately, everything has been automated, and with Jenkins and Ansible, you can easily build an environment from building servers to deploying your creations, so I feel like productivity will increase even more. If you are still building a server manually, I highly recommend that you give it a try.

So that's the end of this post. Let's meet again!

[^1]: A type of data format whose structure is quite similar to JSON. However, when compared to JSON, it feels more like a markup language. Although it has some quirks, it allows for more intuitive expression than JSON.
