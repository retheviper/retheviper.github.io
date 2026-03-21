---
title: "An Overdue Look at I/O"
date: 2021-01-12
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
  - io
  - nio
---

I previously wrote a post about NIO, which was introduced in Java 1.7, but there are still many problems with file I/O in Java. It is a bit embarrassing, but as [The Newsroom](https://en.wikipedia.org/wiki/The_Newsroom_(American_TV_series)) says, "The first step in solving any problem is recognizing there is one." So I decided to look back at the code I have written so far and reflect on how it could be improved.

So this time, I want to look back at the code I have written so far and think about which methods I should prefer. Covering every I/O case would be too much, so I will focus only on the case of `copying files in Java code`. The questions I want to verify are these:

1. How should I create InputStream and OutputStream?
    1. Should I use FileInputStream and FileOutputStream?
    2. Should I use the Files method?
2. What method should I use to copy files?
    1. Should I write from InputSteam to OutpuStream?
        1. readAllBytes()
        2. transferTo()
    2. Should I use the Files method?

This time, I would like to think about these questions while looking at each commonly used file copy code one by one.

## How should I create InputStream and OutputStream?

First of all, InputStream and OutputStream. I think that in many cases even now, files are treated as Streams as much as possible, considering memory issues. In particular, Java is often used in web applications these days, and Spring, a typical framework for creating web applications, uses the Stream format for file uploads and downloads, and in the sense that it can handle file data regardless of whether it's local or over a network, I think it's more versatile than using File or Path objects.

However, in Java 1.7 and later, there are two ways to generate InputStream and OutputStream to copy files locally. One is to generate `FileInputStream`/`FileOutputStream` from a File object, and the other is to generate one using a Path object and Files class.

I would like to write future code using NIO as much as possible, but I would like to first check whether that is really enough or whether it is necessary to change the existing code (which uses FileInputStream and FileOutputStream) to NIO.

## Copy method

First, let's look at the code that uses InputStream and OutputStream in Java to copy a file.

I think that the file copy code that many people, including me, who are new to Java will encounter is probably something like the following. It can be said to be the most common and orthodox form.

```java
// Example using byte[]
private void copy(File source, File dest) throws IOException {
    InputStream is = null;
    OutputStream os = null;
    try {
        is = new FileInputStream(source);
        os = new FileOutputStream(dest);
        byte[] buffer = new byte[8192];
        int length;
        while ((length = is.read(buffer)) > 0) {
            os.write(buffer, 0, length);
        }
    } finally {
        is.close();
        os.close();
    }
}

// Example using BufferedStream
private void copy(File source, File dest) throws IOException {
    int buff = 8192;
    InputStream is = null;
    OutputStream os = null;
    try {
        is = new BufferedInputStream(new FileInputStream(source), buff);
        os = new BufferedOutputStream(new FileOutputStream(dest), buff);
        int length;
        while ((length = is.read()) > 0) {
            os.write(length);
        }
    } finally {
        is.close();
        os.close();
    }
}
```

First, `FileInputStream` can be replaced with `Files.newInputStream`, and `FileOutputStream` can be replaced with `Files.newOutputStream`. The first noticeable difference is that `FileInputStream` and `FileOutputStream` take `File` as an argument, and `Files.newInputStream` and `Files.newOutputStream` take `Path` as an argument. However, this difference cannot be said to be a very decisive difference because conversion between `File` and `Path` is free. This means that you can easily switch between the two methods.

At first glance, it seems better in terms of performance to generate instances of InputStream and OutputStream from the `Files` class because it uses the latest API. However, Java's NIO does not necessarily have an advantage over existing IO in terms of performance. In fact, some say that when it comes to file I/O, even if you use NIO, it only works in Blocking mode, so performance doesn't change much.

In such a case, even though there is no particular problem, I feel that there is no need to switch the existing code to NIO. But is that really okay?

## FileInputStream and FileOutputStream issues

Actually, it doesn't seem to be the case. `FileInputStream` and `FileOuputStream` have issues other than performance. This means that the entire application may be in a paused state for a long time due to GC.

### finalize() override issue

GC pauses the entire application, which means memory has filled up. If memory fills up while using InputStream and OutputStream for file I/O, the first instinct is to think, `doesn't this just mean I need to call close() properly?`

However, the real issue lies in the implementation of `FileInputStream` and `FileOutputStream`. These classes override `finalize()`, and even if `close()` is called properly, data can still remain in memory longer than expected. This issue was [raised as a problem](https://issues.jenkins.io/browse/JENKINS-42934) in Jenkins, and OpenJDK also discussed the need to [remove `finalize()`](https://bugs.openjdk.java.net/browse/JDK-8212050), as explained in [this article](https://dzone.com/articles/fileinputstream-fileoutputstream-considered-harmful).

As for JDK support, `finalize()` of `FileInputStream` and `FileOuputStream` became `Deprecated` from Java 9, and the problem was solved by adding another implementation from Java 10, but this means that problems may still occur when using Java 1.7 or 1.8.

Therefore, I think it is necessary to avoid using `FileInputStream`/`FileOutptStream` as much as possible from now on. Habits are scary.

## What method should I use to copy files?

Based on the conclusion so far, we will use NIO for InputStream and OutputStream instances. Therefore, the above code can be modified as follows.

```java
// Example using byte[]
private void copy(Path source, Path dest) throws IOException {
    InputStream is = null;
    OutputStream os = null;
    try {
        is = Files.newInputStream(source); // Avoid FileInputStream
        os = Files.newOutputStream(dest); // Avoid FileOutputStream
        byte[] buffer = new byte[8192];
        int length;
        while ((length = is.read(buffer)) > 0) {
            os.write(buffer, 0, length);
        }
    } finally {
        is.close();
        os.close();
    }
}

// Example using BufferedStream
private void copy(Path source, Path dest) throws IOException {
    int buff = 8192;
    InputStream is = null;
    OutputStream os = null;
    try {
        is = new BufferedInputStream(Files.newInputStream(source), buff); // Avoid FileInputStream
        os = new BufferedOutputStream(Files.newOutputStream(dest), buff); // Avoid FileOutputStream
        int length;
        while ((length = is.read()) > 0) {
            os.write(length);
        }
    } finally {
        is.close();
        os.close();
    }
}
```

## try-with-resource

If you do not perform `close()` on InputStream and OutputStream at the end, data will remain in memory even if you have already used it. Therefore, I think it is common to close with a finally block, but in such cases, additional exception handling may be necessary even in the finally block, and it is dangerous to use `close()` every time because you may forget.

So from Java 1.7, `AutoCloseable` and `try-with-resource` were introduced, making it possible to write more concise and secure code as shown below. For example, the above code can be replaced with the following code using `try-with-resource`.

```java
// Example using byte[]
private void copy(Path source, Path dest) throws IOException {
    try (InputStream is = Files.newInputStream(source);
        OutputStream os = Files.newOutputStream(dest)) {
        byte[] buffer = new byte[8192];
        int length;
        while ((length = is.read(buffer)) > 0) {
            os.write(buffer, 0, length);
        }
    }
}

// Example using BufferedStream
private void copy(Path source, Path dest) throws IOException {
    int buff = 8192;
    try (InputStream is = new BufferedInputStream(Files.newInputStream(source), buff);
        OutputStream os = new BufferedOutputStream(Files.newOutputStream(dest), buff)) {
        int length;
        while ((length = is.read()) > 0) {
            os.write(length);
        }
    }
}
```

Try-with-resource has only advantages compared to existing methods, so I will definitely use it.

## readAllBytes()

The next thing to consider is Buffer in file copy. In the above examples, we use `byte[]`, `BufferedInputStream`, and `BufferedOutputStream`, but you should know that this is for performance purposes, so we will omit Buffer.

As far as we know, the larger the Buffer size, the better the performance. If so, specifying the largest possible Buffer size as long as the memory allows will naturally maximize the performance that the machine can provide.

Starting with Java 9, there is a method called `readAllBytes()` that reads the entire InputStream at once and returns it as `byte[]`. Using this method, you can generate a `byte[]` of `Integer.MAX_VALUE` size and read the entire InputStream. Theoretically, if you use this, you should be able to copy files in no time.

However, what you need to consider is that the data read in this way will remain in memory. For example, in a web application used by multiple users, you may be uploading files that are several GB in size, but if you use `readAllBytes()`, you will quickly run out of memory. No matter how fast file copying becomes, if multiple users upload files at the same time, a large amount of file data may become clogged in memory even in one round, so it is not a good choice. Therefore, you should refrain from using `readAllBytes()` specifications as much as possible.

## transferTo()

Among the methods added from Java 9, InputStream also has a method called `transferTo()` that allows you to more easily transfer data to OutputStream. In addition to `try-with-resource`, you can use `transferTo()` to copy files with even simpler code. For example, something like the following.

```java
private void copy(Path source, Path dest) throws IOException {
    try (InputStream is = Files.newInputStream(source);
        OutputStream os = Files.newOutputStream(dest)) {
        is.transferTo(os);
    }
}
```

As you can see from the source code, `transferTo()` uses byte[] created with the default Buffer size for copying, so if you use the default value of Buffer (`8192` bytes), it is attractive that you do not need to specify Buffer. Below is the source code.

```java
public long transferTo(OutputStream out) throws IOException {
    Objects.requireNonNull(out, "out");
    long transferred = 0;
    byte[] buffer = new byte[DEFAULT_BUFFER_SIZE]; // 8192
    int read;
    while ((read = this.read(buffer, 0, DEFAULT_BUFFER_SIZE)) >= 0) {
        out.write(buffer, 0, read);
        transferred += read;
    }
    return transferred;
}
```

The only thing I'm concerned about is whether Buffered is really necessary when using `transerTo()`. For example, I wonder if wrapping InputStream with `BufferedInputStream` will at least increase the speed of reading files. Anyway, just in case, I also performed a simple benchmark (actually, I just wanted to try it). I generated a file of about 10GB and tested it in the following case.

- InputStream to OutputStream
- BufferedInputStream to OutputStream
- InputStream to BufferedOutputStream
- BufferedInputStream to BufferedOutputStream

And the code is below.

```java
@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
public class StreamBufferTest {

    private Path source;

    private Path output = Path.of("/Users/retheviper/temp/benchmarkOutput");

    // Create a file for testing
    @Setup
    public void init() throws IOException {
        final String path = "/Users/retheviper/temp/benchmarkSource";
        final RandomAccessFile file = new RandomAccessFile(path, "rw");
        long size = (1024 * 1024 * 1024) * 10L; // 10GB
        file.setLength(size);
        this.source = Path.of(path);
    }

    @Benchmark
    public void noBuffer() throws IOException {
        try (InputStream in = Files.newInputStream(source);
             OutputStream out = Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
            in.transferTo(out);
        }
    }

    @Benchmark
    public void withInputBuffer() throws IOException {
        try (InputStream in = new BufferedInputStream(Files.newInputStream(source));
             OutputStream out = Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
            in.transferTo(out);
        }
    }

    @Benchmark
    public void withOutputBuffer() throws IOException {
        try (InputStream in = Files.newInputStream(source);
             OutputStream out = new BufferedOutputStream(Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING))) {
            in.transferTo(out);
        }
    }

    @Benchmark
    public void withBothBuffer() throws IOException {
        try (InputStream in = new BufferedInputStream(Files.newInputStream(source));
             OutputStream out = new BufferedOutputStream(Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING))) {
            in.transferTo(out);
        }
    }
}
```

And the benchmark results are as follows.

```dos
Benchmark                          Mode  Cnt   Score   Error  Units
StreamBufferTest.noBuffer          avgt   25  13.055 ± 0.583   s/op
StreamBufferTest.withInputBuffer   avgt   25  13.302 ± 0.460   s/op
StreamBufferTest.withOutputBuffer  avgt   25  13.663 ± 0.535   s/op
StreamBufferTest.withBothBuffer    avgt   25  12.668 ± 0.355   s/op
```

As expected, when copying with `transferTo()`, the performance did not change much even without using BufferedInputStream or BufferedOutputStream. I feel like it will be necessary again if it is not a simple file copy or a data transfer from InputStream to OutputStream, but if you can use this method, it seems like you don't need to be aware of it too much.

## Is Files.copy() good?

Java 1.7 allows you to copy the following files through `Files.copy()`.

- InputStream to Path
- Path to OutputStream
- Path to Path

And because Java's NIO is written in native code, some say that Files.copy() has better performance than writing from an InputStream to an OutputStream. If this story is true, it seems better to copy using Path than writing from InputStream to OutputStream, at least when dealing with local files.

### Check with source

Since the implementations are said to be different, it is better to check directly. Let's look at the source code for the three methods mentioned above. First, `InputStream to Path` and `Path to OutputStream`. These simply create an `OutputStream` or `InputStream` from a `Path` and then copy data using `transferTo()`. (This is the Java 11 JDK source, so it may differ in Java 9 or earlier.)

```java
// InputStream -> Path
public static long copy(InputStream in, Path target, CopyOption... options) throws IOException {
    // Omitted non-copy logic

    OutputStream ostream;
    try {
        ostream = newOutputStream(target, StandardOpenOption.CREATE_NEW,
                                            StandardOpenOption.WRITE);
    } catch (FileAlreadyExistsException x) {
        if (se != null)
            throw se;
        // someone else won the race and created the file
        throw x;
    }

    // do the copy
    try (OutputStream out = ostream) {
        return in.transferTo(out);
    }
}

// Path -> OutputStream
public static long copy(Path source, OutputStream out) throws IOException {
    // ensure not null before opening file
    Objects.requireNonNull(out);

    try (InputStream in = newInputStream(source)) {
        return in.transferTo(out);
    }
}
```

However, `Path to Path` is implemented differently. If the source and destination are on the same file system, it uses [FileSystemProvider](https://docs.oracle.com/javase/7/docs/api/java/nio/file/spi/FileSystemProvider.html). Otherwise it falls back to `CopyMoveHelper`.

```java
// Path -> Path
public static Path copy(Path source, Path target, CopyOption... options) throws IOException {
    FileSystemProvider provider = provider(source);
    if (provider(target) == provider) {
        // same provider
        provider.copy(source, target, options);
    } else {
        // different providers
        CopyMoveHelper.copyToForeignTarget(source, target, options);
    }
    return target;
}
```

In the case of `CopyMoveHelper.copyToForeignTarget()`, the implementation eventually ends up calling `Files.copy(InputStream, Path)`. In the same-file-system case, however, the method is completely different, so there may be a performance difference. In short, performance may improve only when copying `Path` to `Path` within the same file system.

This also needed verification, so I ran another benchmark. Results can vary depending on the file system, so this is not absolute, but it should still show whether a difference exists. The other `Files.copy()` overloads are essentially the same as `transferTo()`, so this comparison focuses only on `InputStream to OutputStream` and `Path to Path`. Since there were fewer cases to compare, I also increased the file size beyond the earlier `transferTo()` benchmark. The test code is below.

```java
@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
public class StreamCopyTest {

    private Path source;

    private Path output = Path.of("/Users/youngbinkim/Downloads/benchmarkOutput");

    // Create a file for testing
    @Setup
    public void init() throws IOException {
        final String path = "/Users/youngbinkim/Downloads/benchmarkSource";
        final RandomAccessFile file = new RandomAccessFile(path, "rw");
        long size = (1024 * 1024 * 1024) * 10L; // 10GB
        file.setLength(size);
        this.source = Path.of(path);
    }

    @Benchmark
    public void streamToStream() throws IOException {
        try (InputStream in = Files.newInputStream(source);
             OutputStream out = Files.newOutputStream(output, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
            in.transferTo(out);
        }
    }

    @Benchmark
    public void pathToPath() throws IOException {
        Files.copy(source, output, StandardCopyOption.REPLACE_EXISTING);
    }
}
```

And the benchmark results are as follows.

```dos
Benchmark                      Mode  Cnt   Score   Error  Units
StreamCopyTest.streamToStream  avgt   25  12.128 ± 0.331   s/op
StreamCopyTest.pathToPath      avgt   25  12.257 ± 0.342   s/op
```

For a 10GB file, the results were within the error range, so I changed the file size to 100GB and ran the same benchmark. The result is below.

```dos
Benchmark                      Mode  Cnt    Score   Error  Units
StreamCopyTest.streamToStream  avgt   25  160.046 ± 2.538   s/op
StreamCopyTest.pathToPath      avgt   25  153.505 ± 2.662   s/op
```

The difference only becomes visible around 100 GB, but the conclusion here is that `Path to Path` is faster. If I get the chance, I would like to benchmark this again with multi-threaded I/O and across different operating systems, but for now the result is in line with expectations.

## lastly

Summarizing what we learned from the source code and benchmarks above, we can conclude that file copying in Java is as follows.

- For Java 1.7 or higher
  - Use `Files.newInputStream`/`Files.newOutputStream` instead of `FileInputStream`/`FileOutputStream`
  - Use `try-with-resource`
  - If both the copy source and destination are paths on the same file system, both `Path` uses the argument `Files.copy()`.
- For Java 9 or higher
  - If Buffer size is `8192`, use `transferTo()`
    - When using `transferTo()`, `BufferedInputStream` and `BufferedOutputStream` are not required.

In many cases, enterprise applications are likely to use LTS 1.8 or 11, so virtually all of the items listed above apply.

Although this post has a rather up-to-date feel, I was able to organize it in a way that made sense to me, and I felt refreshed. It's a good learning experience to casually look back at the codes you used ``because that's what I was taught''. Next time, I would like to try this kind of verification using source code and benchmarks.

See you soon!
