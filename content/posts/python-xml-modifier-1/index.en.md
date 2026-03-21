---
title: "Manipulating xml files with Python (1)"
date: 2019-06-09
categories: 
  - python
image: "../../images/python.webp"
tags:
  - xml
  - python
---
I think the best thing about choosing an IT-related job is that you can gain a lot of experience by being placed in a variety of situations. This is because even if you understand the basic grammar of a language by studying on your own, you may not know how to design it or what modules (including libraries) to use when actually coding. In many cases, you don't know what to make in the first place. Broadly speaking, there are things like web applications and batch-run code, and I think it would be quite difficult for one person to imagine all the details of what kind of database you would like to use and what kind of work you would like to do (targets to join, etc.). However, in work, the requirements are set to a certain extent, so once you know how to combine them and the necessary work, all you have to do is do your best. This may be why goal setting is more important than anything else.

So this time too, I was assigned to do it for work. Here are the requirements: I am developing using a certain tool. This tool uses a GUI to place icons on the map, and each icon serves as a unit of work. The overall workflow is then created by connecting icons and setting certain actions for each icon. For example, if you enter information about the DB to connect to and the SQL statement to be issued in the DB icon, and connect it to the file output icon, when you execute it, the results of executing the SQL statement from the DB will be saved as a file. Workflows created with this tool are saved as xml files.

The problem is that there are some differences in the settings between the development environment and the production environment. The work I do mainly through this tool is DB-related, but the DB connection information is different between the development environment and the production environment. The settings for the DB connection destination are saved in the xml file, so when deploying the developed xml file to the production environment, it is incomplete to simply copy it, so it is necessary to ``rewrite the DB connection information''. How I implemented this work will be the theme of this post.

## Advance preparation

Both the destination and source servers are using Linux. And the source of the xml file is to be managed with Git. So, you can easily do a Git pull from the deployment source, rewrite the DB information, and then copy it using a command such as rsync. I decided to automate this work using Jenkins. If so, the remaining problem is how to implement the task of rewriting the DB information.When I analyzed the xml, I found that under each icon's tag there was detailed setting information for that icon. There are two icons for DB processing; the first one is for issuing SELECTs (hereinafter referred to as "From"), and the second is for issuing INSERTs (hereinafter referred to as "To"). The DBs connected to From and To are different types (one is PostgreSQL, the other is SQL Server), and the schemas and table names are also different, so it is necessary to process them separately.

Considering the environment, it would be better to choose a language that can be used with Linux. First, I decided to write the code using a shell script. This is my first shell script. I chose shell because I thought it would be easier than Java. Linux also comes with Python, but I had never used it before, so I thought I'd try using the shell first, and if that didn't work, I'd try Python. In the end, I decided that I should have written it in Python from the beginning.

Anyway, now that I had what I wanted to do, the environment, and the tools, I immediately started implementing it.

## Coding with shell script

It was my first time using a shell, so there was a lot of trial and error. The first language I learned was Java, so when I tried to write it the same way, it did not work at all. After several failures, I came to the conclusion that it was important to stop thinking in terms of functions and instead focus on how to combine commands. It took me quite a while to realize that, but once I understood what mattered, the next steps became straightforward.

First, let's load the file. Since xml files are text-based after all, they can be read by a shell as well. Apparently it is possible to cycle through files with a specific extension and read them line by line with a single For statement. Then, all you have to do is figure out the DB connection information line for the existing icon (component, in the words of the tool that uses this file), and rewrite it.

However, as mentioned above, we need to distinguish between the From and To components. Looking at the xml file, it seems that the component structures (tag types) are almost the same, so the problem was how to judge them. There doesn't seem to be any module that can parse xml in the shell. So, I decided to first compare the number of lines, and if the From component was higher up, I would determine that the DB setting that was caught first was the From one. Below is the code for its implementation.

## Code example (shell script)

```bash
#!/bin/bash
# Walk through lower folders and store `.xml` files in `fileName`
for fileName in `\find . -name '*.xml'`; do
    # Get the line number for each component (check the component tag with grep and extract the line with sed)
    getComponentLine=$(grep -n RDBGet ${fileName} | grep Component | sed -e 's/:.*//g')
    putComponentLine=$(grep -n RDBPut ${fileName} | grep Component | sed -e 's/:.*//g')

    # Get the line number for each connection
    ConnectionOneLine=$(grep -n Connection ${fileName} | sed -e 's/:.*//g' | awk 'NR==1')
    ConnectionTwoLine=$(grep -n Connection ${fileName} | sed -e 's/:.*//g' | awk 'NR==2')

    # Get the connection name (use cut to isolate the DB setting name, then awk to select the matching line)
    ConnectionOneName=$(grep Connection ${fileName} | cut -d ">" -f 2 | cut -d "<" -f 1 | awk 'NR==1')
    ConnectionTwoName=$(grep Connection ${fileName} | cut -d ">" -f 2 | cut -d "<" -f 1 | awk 'NR==2')

    if [ "${getComponentLine}" -lt "${putComponentLine}" ]; then
        # If get < put, ConnectionOneLine is the get connection
        sed -i -e "${ConnectionOneLine} s/${ConnectionOneName}/HONBADBFROM/g" ${fileName}
        sed -i -e "${ConnectionTwoLine} s/${ConnectionTwoName}/HONBADBTO/g" ${fileName}
    else
        # If get > put, ConnectionOneLine is the put connection
        sed -i -e "${ConnectionTwoLine} s/${ConnectionTwoName}/HONBADBFROM/g" ${fileName}
        sed -i -e "${ConnectionOneLine} s/${ConnectionOneName}/HONBADBTO/g" ${fileName}
    fi
done
```

## Problem

The file format is xml, and the current method of tag parsing, which does not reliably separate the components, is not very secure. With this method, it may be okay if there is one each of From and To components, but if the number of either component increases by even one, you will have no choice but to change the processing method. If such cases increase, you may need to write different code for each case. And it is necessary to check each file and match it with the corresponding code. I think this is not much different from changing it by hand.The result was code that was neither versatile nor secure. This kind of code cannot be used locally. So I decided to change the method.

## Rewrite in Python

As a next method, I decided to use Python to perform proper parsing. After trying it out, I realized that it was so easy that I thought Python was the correct answer for such a simple task. Additionally, it seems that Python is basically installed in many Linux environments (yum is a typical example of using Python), so the advantage is that you don't need to install it. Also, since Bash is a basic feature of Linux, I thought it would be faster than Python, but it seems that is not necessarily the case. In that case, there is no reason to be particular about shell scripts.

However, it seems that most of the Python built into Linux is Python 2 (I checked and the Mac I use also has Python 2 installed), and there are many syntax differences between Python 2 and 3, so you need to be careful when using certain functions. In fact, there are parts of the code I wrote that can only be used with Python3. There is also a way to specify the Python link to 3 with a command like alternative, but this may cause problems with programs that use Python2. [^1] Therefore, I had to devise a method to write the code in Python2 from the beginning, to run the code as Python3, or to change the execution environment of the program that uses Python2.

Here, I configured the code I wrote to run in Python3 (I'm embedding it in Jenkins, so I configured it there). The code is below.

## Code example (Python)

```python
# -*- coding: UTF-8 -*-
# Specify the encoding first because the comments are written in Japanese

# Import the XML parser and the module used to collect files from folders
import xml.etree.ElementTree as ET
import glob

# Declare namespaces (prefixes) in a map
ns = {'fb':'http://foo.com/builder', 'fe':'http://foo.com/engine', 'mp':'http://foo.com/mapper'}

# Recursively collect file names (`recursive` is Python 3 only)
fileList = glob.glob("**/HOGE*.xml", recursive=True)

# Rewrite connection names while iterating over the collected files
for fileName in fileList:
    # Start parsing the file
    tree = ET.parse(fileName)

    # Get the child connection name from the To component (within the namespace)
    putCon = tree.find("fe:WorkFlow/fe:Component[@type='RDB(Put)']/fe:Property[@name='Connection']", ns)
    putCon.text = putCon.text + 'x'
 
    # Get the child connection name from the From component (within the namespace)
    getCon = tree.find("fe:WorkFlow/fe:Component[@type='RDB(Get)']/fe:Property[@name='Connection']", ns)
    getCon.text = getCon.text + 'y'

    # Prevent prefixes from changing when writing
    ET.register_namespace('fb', 'http://foo.com/builder')
    ET.register_namespace('fe', 'http://foo.com/engine')
    ET.register_namespace('mp', 'http://foo.com/mapper')

    # Write the updated file
    tree.write(fileName, 'UTF-8', True)
```

## Finally

The amount of code is not very long, and the elements are properly parsed, making it a safer writing method than shell scripts. Additionally, since components are distinguished by tags, it has the advantage of being able to be used as is even if the number of components changes. When a tag has a namespace, initial settings and processing are required just before writing, which can be a bit troublesome [^2], but it certainly takes less time to maintain and repair compared to shell scripts, so I think I was able to write code that I was satisfied with. I didn't directly measure the speed, but it was quite fast (maybe it was just my bias that Python was slow). Wow, Python is great.

The best part is that even if you omit the main functions and classes, even if you write it in a script-like way, it still works as intended. If you want to automate simple repetitive tasks in a Linux environment, I would highly recommend that you try using Python. It's a very simple language, so I'd like to continue using it and trying out various things.

[^1]: For example, I'm having trouble with yum. There is a way to change the yum execution environment (see /usr/bin/yum settings), but it is not recommended as it is not possible to check which one uses Python2.
[^2]: Especially without `ET.register_namespace()` at the end, the namespace will change automatically.
