---
title: "Creating ZIP Files in Kotlin"
date: 2021-04-14
translationKey: "posts/kotlin-zip-files"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
---

When creating server-side functions, there are times when you need a file download function. However, in addition to returning files saved in storage as they are, in some cases it may be necessary to generate a file and return it as is, or to transfer multiple files at once.

It's not difficult to implement if you want to download one file per request, but it gets a little more complicated if you want to download multiple files. If you are sending multiple files all at once, it is better to compress them into a ZIP file. Fortunately, Java basically provides an API called [ZipOutputStream](https://docs.oracle.com/javase/8/docs/api/java/util/zip/ZipOutputStream.html), so all you have to do is add the files to be compressed as entries and output the ZIP file.

However, if there are not only multiple files but also multiple nested directories, additional processing is required such as compressing the file while preserving the directory structure. In some cases, there may be files that you do not want to include. I would like to use one function regardless of the number of files (whether it is one file or a directory). So, this time I would like to briefly introduce the process of migrating from Java code to Kotlin, and how to create a simple method tailored to such a use case.

The code I will introduce this time is based on Baeldung's article about [how to compress ZIP files in Java](https://www.baeldung.com/java-compress-and-uncompress).

## Java

First, let's look at the Java code. The above article introduces the following code:

```java
public class ZipDirectory {
    public static void main(String[] args) throws IOException {
        String sourceFile = "zipTest";
        FileOutputStream fos = new FileOutputStream("dirCompressed.zip");
        ZipOutputStream zipOut = new ZipOutputStream(fos);
        File fileToZip = new File(sourceFile);

        zipFile(fileToZip, fileToZip.getName(), zipOut);
        zipOut.close();
        fos.close();
    }

    private static void zipFile(File fileToZip, String fileName, ZipOutputStream zipOut) throws IOException {
        if (fileToZip.isHidden()) {
            return;
        }
        if (fileToZip.isDirectory()) {
            if (fileName.endsWith("/")) {
                zipOut.putNextEntry(new ZipEntry(fileName));
                zipOut.closeEntry();
            } else {
                zipOut.putNextEntry(new ZipEntry(fileName + "/"));
                zipOut.closeEntry();
            }
            File[] children = fileToZip.listFiles();
            for (File childFile : children) {
                zipFile(childFile, fileName + "/" + childFile.getName(), zipOut);
            }
            return;
        }
        FileInputStream fis = new FileInputStream(fileToZip);
        ZipEntry zipEntry = new ZipEntry(fileName);
        zipOut.putNextEntry(zipEntry);
        byte[] bytes = new byte[1024];
        int length;
        while ((length = fis.read(bytes)) >= 0) {
            zipOut.write(bytes, 0, length);
        }
        fis.close();
    }
}
```

Looking at the `zipFile` method, the argument `fileToZip` specifies the path of the file or directory you want to compress with ZIP, `fileName` specifies the file or directory name, and `zipOut` specifies the compressed ZIP file name.

The implementation includes processing such that if the specified file or directory has the `hidden` attribute, it will not be compressed, and if the compressed file is a directory, all the files inside will be included in the ZIP. After adding all the target files and directories to the entry, the process is to read the compression source and write it to ZipOutputStream. Let's turn this into Kotlin code.

## Try changing to Kotlin code

Converting Java code to Kotlin code is not difficult. In IntelliJ, there is already a feature that automatically converts pasted Java code into Kotlin code. However, that alone is not enough. Even if conversion is easy, it may not actually be Kotlin-like code.

And I'm sure there's a way to make the process itself simpler or easier to read. First, we will convert the above Java code to Kotlin, and then make changes including various areas that we would like to improve.

## Change the code to Kotlin-like code

Based on Intellij 2021.1, if you paste the Java code as is, it will be automatically converted to the code below.

```kotlin
@Throws(IOException::class)
private fun zipFile(fileToZip: File, fileName: String, zipOut: ZipOutputStream) {
    if (fileToZip.isHidden) {
        return
    }
    if (fileToZip.isDirectory) {
        if (fileName.endsWith("/")) {
            zipOut.putNextEntry(ZipEntry(fileName))
            zipOut.closeEntry()
        } else {
            zipOut.putNextEntry(ZipEntry("$fileName/"))
            zipOut.closeEntry()
        }
        val children = fileToZip.listFiles()
        for (childFile in children) {
            zipFile(childFile, fileName + "/" + childFile.name, zipOut)
        }
        return
    }
    val fis = FileInputStream(fileToZip)
    val zipEntry = ZipEntry(fileName)
    zipOut.putNextEntry(zipEntry)
    val bytes = ByteArray(1024)
    var length: Int
    while (fis.read(bytes).also { length = it } >= 0) {
        zipOut.write(bytes, 0, length)
    }
    fis.close()
}
```

The part I would like to change to make the code more Kotlin-like is how to use `InputStream` and `OutputStream`. Java also has `try-with-resource`, and Kotlin has `use()`, so I feel it's better to use them than `close`.

Also, it seems that you can clarify what you want to do by changing `if` to `when`, or changing `for` to `forEach()`. Personally, I think it's better to separate the scopes to make responsibilities clearer and avoid confusion when following the process, so I would like to actively use scope functions and collection-specific operations as much as possible to clearly separate the units of processing. There is no problem with the processing using the Java method, but I would like to take the opportunity to write code that is unique to Kotlin.

I feel that intentionally displaying that an IOException is thrown is not appropriate for Kotlin, which does not force exception handling at runtime, so I decided to remove the annotation.

## Change from IO to NIO

I have mentioned NIO several times in previous posts, but I think it is better to actively adopt it in cases where there is frequent and large number of concurrent executions such as servers. Also, since there are many useful functions added since Java 1.8, it is possible to reduce the amount of code considerably by simply changing IO to NIO.

In particular, when a directory is specified, the function provided by NIO's `Files` is powerful for cycling through child elements of that directory, so this time we will actively utilize it.

## change signature

The above method takes three arguments, but only the source path and the destination path are actually required. There is no particular reason to pass ZipOutputStream at the caller; rather, it is inconvenient because you need to define it every time you use this method. I also don't think it's a good idea to simply add an entry in the method and share the responsibility of processing the object with the caller. Therefore, we will change the generation and use of ZipOutputStream to be completed within the method.

By doing this, it will be easier to use outside the method (the caller), and you can avoid the complicated situation where `InputStream` used to read the compressed data is closed inside, but the argument `OutputStream` is closed outside.

## eliminate recursion

If the compression source path is a directory, recursion is used to compress nested directories and files at once. Although recursion is important as an algorithm, it may not be very efficient in terms of processing efficiency because it stores all data and processing in memory until all processing is completed. What I want to do is simply exclude files and directories that have the `hidden` attribute, and include all other files and directories in the ZipOutputStream entry.

Fortunately, by using NIO, you can get all the child elements of a directory, and since you can get the child elements as `Stream<Path>`, you can use methods like `filter()` and `forEach()`. This seems to be sufficient to achieve the purpose without using recursion.

## completed code

The modified code reflecting the above is as follows.

```kotlin
object ZipService {

    fun archive(source: Path, target: Path): Unit =
        ZipOutputStream(Files.newOutputStream(target)).use { zos ->
            Files.walk(source)
                .filter { Files.isHidden(it).not() }
                .forEach {
                    if (Files.isDirectory(it)) {
                        zos.putNextEntry(ZipEntry("$it/"))
                        zos.closeEntry()
                    } else {
                        zos.putNextEntry(ZipEntry(it.toString()))
                        Files.copy(it, zos)
                    }
                }
        }
}
```

To explain briefly, I made it a utility class that can be used anywhere by placing it in a Singleton class declared as `object`, and the method signatures were made simpler. The argument `source` is supposed to specify the file or directory to be compressed, and `target` is supposed to specify the ZIP file to be compressed. ZipOutputStream is created in the method and is automatically closed using `use()`.

First, I preferentially use `Files.walk()` to retrieve all child elements. Since the acquired child element is sorted out if it is `filter()` and not `hidden`, there will be no branching. Also, if the child element is a directory, add `/` to represent the directory name, add and close `ZipEntry`. If the child element is a file, add `ZipEntry` and copy the content. Now you have a shorter, simpler code.

## lastly

Although I mentioned Kotlin-like code, the above code is only valid for `Kotlin/JVM`. If you want to use it with `Kotlin/Native`, `Kotlin/JS`, or similar targets, you will need another approach. Also, `Files.walk()` was added in Java 1.8, so on 1.7 you would need to use `Files.walkFileTree()`, and for earlier versions you would need a method other than NIO.

So, this may be the best for `Kotlin/JVM` (Java 1.8 or higher), but there seems to be room for further research. I think there is some value in switching Java APIs using Kotlin methods like this.

See you soon!
