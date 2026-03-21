---
title: "Manipulating xml files with Python (2)"
date: 2019-06-20
categories: 
  - python
image: "../../images/python.webp"
tags:
  - xml
  - python
---

In [previous post](../python-xml-modifier-1/), I introduced a script that manipulates xml files. However, because I decided not to use the script for a while at work, I didn't actually use it and left it alone, but when my policy changed and I actually applied the script I had created, a certain problem came up. At first, I didn't think the script was the problem, and it took me quite a while to figure out the cause, but I'm relieved now that I've managed to solve it. This time as well, I thought that in the act of coding, it is more important to check whether something is behaving strangely than to implement it as designed.

In this post, I would like to explain what the specific problem was and how I improved the code. Strictly speaking, it might be more accurate to say that I made a mistake during the detailed design stage rather than calling it a bug, but whatever the reason, it made me think that I had to reflect on the fact that I had written code that didn't work as originally planned. It would be nice to be able to say that I was able to grow again.

## Old script issue

At first glance, the script introduced in the previous post seemed to be working as expected. When I actually tested it, the text part of the specified element was changed correctly. However, the problem arose because the number of elements in the xml file to be processed was larger than expected. In other words, the old script had a simple configuration with one DB element that issues SELECT statements and one DB element that issues INSERT statements, but this time we are in a situation where we have to process multiple elements.

When I tried to process it with the old script, after rewriting one element, the remaining elements were ignored and processing moved on to the next file. Without knowing this, I believed that it was working as designed when the processing was completed, but when I used the processed file, an error occurred, which led me to revise this script.

First, now that we know the cause, let's fix the goal of the script. The goal this time is to "correct all elements that match the conditions."

## Modify the code

As you modify the code to meet your goals, you will also try to improve any minor issues. In the previous script, we used the `glob` module to get the files in a folder as a list. It was convenient because it could recursively collect files in lower folders with a small amount of code, but the problem is that glob's recursive option can only be used with Python 3.5 or higher. If you normally use Python3, there shouldn't be much of a problem, but all other Python scripts you use are created based on Python2. So I decided to change this to an option that can also be used with Python2.

The main improvement is to change `find` to `findall`. First, get all the elements of the DB connection, and then use the `if` statement to perform branch processing based on the DB connection name. Also, the DB connection name that we want to change this time is to change the name with an underscore and a consecutive number like `From_PostgreSQL_01` to remove only the serial number like `From_PostgreSQL`, so we will also think about how that works. There is also a way to use `replace`, but with this you have to write conditions for all cases, and depending on the example of condition specification, there is a possibility of duplication. Therefore, we will use `rsplit`[^1] to split the original string based on the underscore and then replace the original text.

The following has changed from these requirements definitions.

```python
# -*- coding: UTF-8 -*-

import xml.etree.ElementTree as ET
import os

# Declare namespaces (prefixes) in a map
ns = {'fb': 'http://builder',
      'fe': 'http://engine',
      'mp': 'http://mapper'}

# Recursively collect XML file names (for Python 2)
fileList = []
base_dir = os.path.normpath('./baseFolder') # Set the starting directory for the search

for (path, dir, files) in os.walk(base_dir):
    # Remove `.xml2` files and collect only `.xml` files for rewriting
    for fname in files:
        if ('.xml2' in fname):
            fullfname = path + "/" + fname
            os.remove(fullfname)
        elif ('.xml' in fname and not '.xml2' in fname):
            fullfname = path + "/" + fname
            fileList.append(fullfname)

# Rewrite connection names while iterating over the collected files
for fileName in fileList:
    # Start parsing the file
    tree = ET.parse(fileName)

    # If the INSERT component's connection name has suffixes like `_01`
    PutConnections = tree.findall("fe:Flow/fe:Component[@type='RDB(Put)']/fe:Property[@name='Connection']", ns)
    for PutConnection in PutConnections:
        if ('_0' in PutConnection.text):
            PutConnection.text = PutConnection.text.rsplit('_', 1)[0] # Split with rsplit and keep the left side

    # If the SELECT component's connection name has suffixes like `_01`
    GetConnections = tree.findall("fe:Flow/fe:Component[@type='RDB(Get)']/fe:Property[@name='Connection']", ns)
    for GetConnection in GetConnections:
        if ('_0' in GetConnection.text):
            GetConnection.text = GetConnection.text.rsplit('_', 1)[0] # Split with rsplit and keep the left side

    # Prevent prefixes from changing
    ET.register_namespace('fb', 'http://builder')
    ET.register_namespace('fe', 'http://engine')
    ET.register_namespace('mp', 'http://mapper')

    # Write the updated file
    tree.write(fileName, 'UTF-8', True)
```

## lastly

The file acquisition part is a little more complicated than what could be easily done with the `glob` option. Specify the starting directory with `os.walk()` to get the path and file name. However, with `os.walk()`, the file name and path are separated, so you need to connect them. Add processing to add or remove files from the list of files to be processed. This allows behavior similar to the previous `glob`.

Then, at `rsplit('_', 1)`, the string `From_PostgreSQL_01` is first split into `From_PostgreSQL` and `01` only once. Since the divided string becomes an array, if you specify [0], `From_PostgreSQL_01` will be replaced with `From_PostgreSQL` as intended. Also, just by changing `find` to `findall`, it will search for all elements in the file that match the conditions and create a list. It's just a loop process inside. When I didn't understand this, I tried adding another loop to the previous code and failed, but there was a surprisingly simple solution.

The completed code now works as intended. Personally, I'm satisfied because even if there are changes later, I only have to make a few changes. There may be a better way to write it. And as a lesson, I was reminded once again that testing is always important. Always check and confirm.

[^1]: The difference between `rsplit` and `split` is the direction. If the former splits the string based on the right side, the latter splits from the left side. This time, I wanted to get the sequential number at the end of the string, so I chose `rsplit`.
