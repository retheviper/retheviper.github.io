---
title: "IO to NIO"
date: 2020-01-20
translationKey: "posts/java-nio"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - nio
  - java
---
If you are learning I/O in Java for the first time, as I was, I think it is common to create a File object, read it with InputStream, and write with OutputStream. If we progress a little further, we will probably reach the level of using classes such as Writer and Reader, wrapping Streams with Buffers, and performing I/O of objects with Serializable.

Even if it's an old API, I don't think it's necessary to change all the code to a new API unless there are major problems with operation or performance. In fact, code that has been forcibly rewritten to a new API may cause problems, and it cannot always be said to be excellent. For example, forEach() added in Java 1.8 is convenient, and I like Lambda, so I use it in many situations, but in reality, JVMs up until now are optimized for traditional for loops, so forEach() seems to have inferior performance. There is a possibility that the performance of forEach() will improve in the future, but looking at the recent Java version update history, it is a bit of a mystery how long it will take to improve the performance of the functional API.

There are always pitfalls when you adopt a new API, so you need to be careful. But APIs become standard for a reason, and if you are writing new code or simplifying existing code, proactively adopting them is not a bad idea. The API I want to introduce this time is one of those. NIO is a newer way to handle file I/O, though at this point it is not exactly new anymore because it was introduced back in Java 1.7.

## What is NIO?

NIO stands for Java's new I/O. I tend to think that it stands for New, but it actually stands for Non-blocking. Java is slower than C and C++, and one of the reasons seems to be I/O. So NIO was created to improve that.

There are various differences, such as whether it is Blocking or Non-blocking, or whether it is Stream-based or Channel-based, but in general, when frequent I/O is required, better performance can be expected by choosing NIO. Other benefits include:

- No thread blocking occurs
- Code is more concise
- Easy to specify copy, move, and import options
- Since Buffer is basically used, wrapping with Buffered~ is no longer necessary.

I'm not very familiar with the structure of the JVM, so I won't try to explain it here based on my shallow knowledge. However, the fact that the code is more concise from my point of view is a definite benefit. So I would like everyone to try using it.Now, let's explain how to use NIO in actual code.

## File → Path

NIO uses Path objects rather than File objects. The biggest advantage of Path over the File object is that you can specify the file path separately by directory and file name.

For example, let's say a file path is nested in multiple folders.

```java
// Multiple directories and files exist as strings (path/to/file.txt)
String rootDirectory = "path";
String toDirectory = "to";
String fileName = "file.txt";
```

If you want to create an instance from these multiple strings, the File constructor takes one string argument, so it would look like this: The directory string does not contain a slash, so you need to include a slash while concatenating the strings.

```java
// Create a File object
File file = new File(rootDirectory + "/" + toDirectory + "/" fileName);
```

However, in the case of Path, multiple strings can be specified. All you need to do is specify the directory and file name strings in order.

```java
// Create a Path object
Path path = Paths.get(rootDirectory, directory, fileName);
```

In this way, Path is more convenient for creating instances. Also, even if you absolutely need a File object, there is a function that allows you to convert File methods to Path, which is convenient. Of course, you can also do the opposite.

```java
// Create a File from Path
Path path = file.toPath();

// Create a Path from File
File file = path.toFile();
```

There are many other methods for Path that perform the same functions as File, such as the ability to generate a URI object using the toURI() method, so use whichever is more convenient.

## Files

In traditional I/O, classes such as InputStream, OutputStream, Writer, and Reader have been used to perform operations such as copying and deleting files. In NIO, these tasks are mainly performed using the Files class. Additionally, the Files class has functions to generate Writer, Reader, InputStream, and OutputStream, so it is an easy-to-use class.

## Copy file

Copying files with the Files class is easy. See the code below. Basically, you just specify the copy source and copy destination files as Path objects.

```java
// Copy Path to Path
Files.copy(source, target);
```

When copying with the Files class, you can also specify copy options using ENUM.

```java
// Specify options (also copy file attributes)
StandardCopyOption option = StandardCopyOption.COPY_ATTRIBUTES;
Files.copy(source, target, option);
```

You can also specify an InputStream as the copy source instead of an actual file. In this case, you can also write the data to a file.

```java
// Copy InputStream to Path
Files.copy(sourceStream, target);
```

## Delete file

Similar to copying, deleting a file in the Files class passes a Path object as an argument.

```java
// Delete
Files.delete(path);
```

There are also methods with boolean return values. If the file exists, delete it and return the result as a boolean.

```java
// Delete if it exists
Files.deleteIfExists(path);
```

## Moving filesMoving files is like a combination of copying and deleting. It can also be used to change file names. Since it is basically a copy, you can use the same options as when copying.

```java
// Move or rename
Files.move(path);

// Specify options (overwrite)
StandardCopyOption option = StandardCopyOption.REPLACE_EXISTING;
Files.move(path, option);
```

## Write file

InputStream can be used with copy(), but there is also a method for writing to a file.

```java
// Write data to a Path
Files.write(path, content);
```

`byte[]`, `List<String>`, etc. can be passed as arguments to the write() method. Also, options can be specified as in the case of copy. This option allows you to choose whether to overwrite or append to a file if it exists, so it can be used separately from copy() in some cases.

```java
// Specify options (append)
StandardOpenOption option = StandardOpenOption.APPEND;
Files.write(path, content, option);
```

## Load file

Just as writing is separated by string or byte[], there is a method for reading that allows you to retrieve files in the same way. In the case of string acquisition, the difference in syntax sugar is whether the result is a Stream or a List.

```java
// Read all lines as strings
Stream<String> lines = Files.lines(path);
List<String> liness = Files.readAllLines(path);

// Read as byte[]
byte[] bytes = Files.readAllBytes(path);
```

Like File, Path can also be a directory rather than a file, so the methods of Files also support this. The list() method obtains the directory entry as a Path and generates a Stream.

```java
// Get a Stream whose elements are the directory entries
Stream<Path> files = Files.list(path);
```

## Use in combination with I/O

As mentioned earlier, some of Files' methods can be used in conjunction with traditional I/O. I would like to introduce some of them.

```java
// For reading
InputStream is = Files.newInputStream(path);
BufferedReader br = Files.newBufferedReader(path);

// For writing
OutputStream os = Files.newOutputStream(path);
BufferedWriter bw = Files.newBufferedWriter(path);
```

Of course, you can also specify OpenOption.

```java
// Create the file if it does not exist
StandardOpenOption option = StandardOpenOption.CREATE;
InputStream is = Files.newInputStream(path, option);
```

## Finally

How was it? It may not seem like there is much benefit to using it if it only performs the same function, but when you actually use it, you can clearly specify what you want to do by specifying options using ENUM, and I think NIO provides a convenient class that can reduce the amount of code. In particular, even if you use File as is, Files' methods are useful and powerful, so I highly recommend them to everyone.

There are also many other useful methods in the Files class, such as providing the Channel class that allows two-way communication, getting file attributes and symbolic links, checking if a specified path is a directory, and checking if two paths are the same file, so please try using them.

See you soon!
