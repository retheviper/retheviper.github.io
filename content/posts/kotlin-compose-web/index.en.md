---
title: "Building a File Server Using Only Kotlin"
date: 2022-10-10
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - compose
  - gui
---

There are many different programming languages ​​in the world, each with distinct characteristics, and in many cases the things that each language can and cannot do are different. Companies select technologies based on practical considerations such as hiring engineers and costs, so I think there are clear and general standards for which language to use in a project. However, at an individual level, I think there is a tendency to choose a language based on one's preferences and familiarity, without considering team work. Therefore, there may be cases where relatively minor languages ​​and frameworks are used.

That's exactly what I do, and I use Kotlin or Python as much as possible to create apps and automation scripts that I implement for personal use. In the case of Kotlin in particular, it's because I'm the most familiar with it because I use it at work, but I like the fact that I can try out a variety of things, not just in the server-side genre or JVM environment, thanks to the various frameworks and characteristics of the language itself.

So, this time as well, I tried making a web application in a slightly unusual way for my private life. As stated in the title, what is different is that this is a story about implementing a file server application using only "Kotlin".

## Background

First of all, I have to explain what kind of app I created and why. My parents have a Windows computer that they have been using for a long time. I assembled it about 8 years ago, and I don't use it much these days as I don't often go back to my parents' house. However, I sometimes use it to send and receive files from my current PC (Mac).

Up until now, we have been using Microsoft's [OneDrive](https://www.microsoft.com/microsoft-365/onedrive/online-cloud-storage) to exchange files. If you copy the files you need on one side to the OneDrive folder, those files will be uploaded to the cloud and automatically synced. Even with this, I was using it stably without any problems, but I suddenly felt that the step of going through the cloud was a waste. Also, copying and moving files before and after synchronization is a tedious task.

At this point, I thought why not create my own app that allows me to exchange files over the Internet? There may already be some kind of service that provides the functionality I'm thinking of, but since it's not that complicated, I felt like I could create it in a few days, so I decided to give it a try. (There was also an option called SFTP, but I wanted to make it easier with a GUI, so I rejected it.)

## Requirements

Now, if there is something I want to make, I have decided what to do. As always, before creating an app, we briefly determine the requirements. First of all, I thought it would be nice to be able to do something like the following functionally.

- When you start the server app, the client can access the server's storage.
- Once you specify the server path, you can see its contents (files and folders)
- Clicking on a folder changes the displayed path
- Click on the file to download
- You can upload files to your path

Once the functionality is decided, it's time to develop the technology to realize it. Above all, I want to solve everything with Kotlin! With this in mind, all technology selection is centered around Kotlin.

First of all, with Frontend, I wrote on this blog the other day that I created a simple app using [Compose for Desktop](https://www.jetbrains.com/lp/compose-desktop/), and since there was also something called [Compose for Web](https://compose-web.ui.pages.jetbrains.team/), I thought it would be a good idea to try using it this time. The biggest reason for this was that we wanted to unify the language, but there was also the other reason that we didn't have much experience or knowledge of Frontend, so we wanted to use the technology that we had at least a little exposure to.

We decided to use [Ktor](https://ktor.io/) as the backend framework. I usually use Spring, so I don't really have much experience with it, but from what I've used before, the app starts up quickly and it's easy to implement, so I decided to use it. Similarly, the biggest reason is that it is for Kotlin.

These two are the main ones, and of course there are various other libraries that are required, but as we proceeded with the implementation, we decided to implement the required items based on whether they were made by Kotlin or JetBrains. Alternatively, our policy is to adopt what appears in official documents that will be helpful in implementation.

## Frontend

For Frontend, I used Compose for Web as mentioned earlier. It was my first time doing this, but it was still a new technology, and I didn't know much about Frontend in the first place, so this was the part that took the most man-hours. I would like to explain what I experienced first-hand in terms of three points: what was good, what was different from what I expected, and what was problematic.

## Good points

The good part was that I could build on my experience creating desktop applications with Compose. In Compose, state is often managed with a combination of [`remember` and `MutableState<T>`](https://developer.android.com/jetpack/compose/state#state-in-composables), and screen components are split into functions annotated with [@Composable](https://developer.android.com/reference/kotlin/androidx/compose/runtime/Composable). The same basic idea worked here as well.

So, when I implemented a function to ``browse a specified path'' and wanted to add a function to return to the previous path, it was easier than I expected to include a path in the state to retain that path, or to define and use a component to draw objects (files, folders, etc.) in the path retrieved from the server as a single `@Composable` function on the screen.

Another good thing about Kotlin is that Coroutine can be used easily, and the source code can be created in the same repository as the server side. Especially in the latter case, I like that by setting the Kotlin plugin in Gradle to `multiplatform`, it will be compiled to JavaScript in Frontend, and compiled to JVM bytecode as usual on the server side.

## Something different from what I thought

I was naive in my thinking, but there are some things that are quite different from Desktop. The point is that even if you use Kotlin as a language, you cannot exclude HTML and CSS. Again, I had to use tags like `div` and `form`, and change the tag's `attr` to change the cursor when hovering over the tag. For example, the file upload component below is written in Kotlin, but it actually feels like writing HTML as is.

```kotlin
@Composable
private fun FileUploadForm(currentPath: String) {
    Div {
        Form(
            action = "$API_URL$ENDPOINT_UPLOAD",
            attrs = {
                method(FormMethod.Post)
                encType(FormEncType.MultipartFormData)
                target(FormTarget.Blank)
            }
        ) {
            HiddenInput {
                name("target")
                value(currentPath)
            }
            Input(InputType.File) { name("file") }
            Input(InputType.Submit)
        }
    }
}
```

I have a strong impression that this is just providing a class wrapped like Kotlin, rather than using Compose on another platform completely, and I think this is a part that requires some knowledge of Frontend. Therefore, if you have knowledge of major Frontend frameworks such as [React](https://reactjs.org/) and [Vue.js](https://vuejs.org/), I don't think there is much reason to choose Compose.

Other than that, perhaps because this is a project where Kotlin/JS and Kotlin/JVM coexist, unlike usual, the autocompletion and build behavior on intellij seem a little different. For example, changing dependencies in Gradle may not be reflected immediately...

## What was the problem

The unexpected problem was building the project. Compose for Web uses `index.html` files and `js` files built using Webpack, etc., and the build itself can be easily done with a single Gradle command. However, it seems that it uses [yarn](https://yarnpkg.com/) internally, but with the default settings of the project generated with intellij, errors often occurred during the build.

When I looked into it, I found that [this kind of build failure can be resolved by using Kotlin `v1.6.20` or later](https://github.com/Kotlin/kotlinx-datetime/issues/193). The problem in my case was the Compose version. The latest version when I built this app was [v1.1.1](https://github.com/JetBrains/compose-jb/releases/tag/v1.1.1), but it only supported Kotlin up to `v1.6.10`. In the end, I resolved the issue by switching to the beta version of `v1.2.0` and upgrading Kotlin to `v1.7.10`. This is probably the kind of trouble you run into when choosing a relatively minor stack.

I also use [Ktor Client](https://ktor.io/docs/getting-started-ktor-client.html) as an HTTP client. I wanted to use it instead of submitting multipart data directly with a `form` tag, especially for large file uploads, but that did not work. Because Ktor Client is multiplatform, you can [choose which engine to use](https://ktor.io/docs/http-client-engines.html) when declaring the client. However, with the engines available on Kotlin/JS, even if you implement requests as shown in the [official docs](https://ktor.io/docs/request.html#binary), `File` objects cannot be handled directly. So in this case I could not send the file that way. It may be better to wait for future improvements or use another approach such as WebSockets.

## Backend

Next, regarding Backend, this is an area that I am familiar with, and I have talked about Ktor itself in other posts, so I would like to mainly talk about the trial and error in the logic side rather than the technical side.

## Browsing the file tree

The app first has the ability to browse files, so we need to explore the path specified on the client and return the content (files and folders) inside it. The problem is how to structure the JSON. I decided to try one method first and then make a decision.

### Get all

At first, I thought of implementing it in the following way. Once a path is specified, all folders under it are traced and parent-child relationships are expressed as nests.

```json
{
  {
    "name": "Documents",
    "type": "directory",
    "children": [
      {
        "name": "SomeDocument.pdf",
        "type": "file",
        "size": "1391482",
        "mimeType": "application/pdf"
      }
    ]
  },
  {
    "name": "Images",
    "type": "directory",
    "children": []
  }
}
```

To return such a file tree, I used the following server-side code:

```kotlin
// Given the root path, fetch all child elements (files and folders)
val files =  Files.list(root)
        .filter { !it.isHidden() }
        .map { it.toFileTree() }
        .toList()

// Convert Path into a JSON-friendly object
fun Path.toFileTree(): FileTree {
    return FileTree(
        name = this.fileName.toString(),
        size = if (this.isDirectory()) null else this.fileSize(),
        type = if (this.isDirectory()) FileType.DIRECTORY else FileType.FILE,
        children = if (this.isDirectory()) {
            Files.list(this)
                .filter { !it.isHidden() }
                .map { it.toFileTree() }
                .toList()
        } else {
            null
        }
    )
}
```

If you use [Files.walk()](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#walk-java.nio.file.Path-java.nio.file.FileVisitOption...-), you will get all the nested file trees based on the specified path as `Stream<Path>`, but it is not easy to process them into the above JSON format. Based on the results obtained once, it would be a fairly complicated process to compile them as a JSON object while tracking parent-child relationships.

So, here we use [Files.list()](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#list-java.nio.file.Path-) to retrieve the element included in the specified path, and if that element is a directory, we use recursion to retrieve the child element again. It's a simple process, but it's efficient.

However, although I was able to return the desired file tree as JSON using this method, there was a problem. First, there was a problem that the closer the specified path was to the root, the longer it took to search, the slower the response, and the larger the JSON size. Also, after receiving the JSON, I felt that there might be some difficulty in drawing it with Frontend. So, I decided to scrap this idea and go for the second option.

### Do not nest

The next method I tried was to limit processing to only specified paths. What it means is that it eliminates the nesting of JSON objects and only returns a list of what files and folders are included in the specified path. In other words, it will look like this:

```json
[
  {
    "name": "Documents",
    "type": "directory"
  },
  {
    "name": "Images",
    "type": "directory"
  }
]
```

In this case, there is no need to trace the folders inside, so the response time will be faster and lighter. I should have thought about this from the beginning, but this is easier to implement as a Frontend, and if you want to access a subpath folder, you just need to send that path to the Backend again. As far as code goes, I no longer use recursion.

```kotlin
val files =  Files.list(root)
        .filter { !it.isHidden() }
        .map { it.toFileTree() }
        .toList()

// Convert Path into a JSON-friendly object
fun Path.toFileTree(): FileTree {
    return FileTree(
        name = this.fileName.toString(),
        size = if (this.isDirectory()) null else this.fileSize(),
        type = if (this.isDirectory()) FileType.DIRECTORY else FileType.FILE
    )
}
```

## File Upload

Regarding file upload, how to handle the data sent as Multipart, but this was handled with a simple process typical of Ktor. The code below is the actual implementation.

```kotlin
// router
post(ENDPOINT_UPLOAD) {
    // Receive multipart data
    val multipart = call.receiveMultipart()
    // Destination path for saving files
    var path = Path.of(ROOT_DIRECTORY)
    multipart.forEachPart { part ->
        when (part) {
            // Update the destination when a non-root path is specified
            is PartData.FormItem -> {
                if (part.name == "target") {
                    path = path.resolve(part.value)
                }
            }
            // Save the uploaded file
            is PartData.FileItem -> {
                withContext(Dispatchers.IO) {
                    val file = Files.createFile(path.resolve(part.originalFileName!!))
                    part.streamProvider().use { input ->
                        Files.newOutputStream(file).use { output ->
                            input.copyTo(output)
                        }
                    }
                }
            }
            // Log anything unexpected for now
            else -> {
                println("Unknown part: $part")
            }
        }
        // Dispose each part after processing
        part.dispose()
    }
}
```

However, I personally want to use NIO for processes that require storage access, so I initially thought of using [Files.copy()](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#copy-java.io.InputStream-java.nio.file.Path-java.nio.file.CopyOption...-), but for some reason it didn't work when I created a file save process like the one below. There may be some problem with compatibility with Coroutine, so be careful here.

```kotlin
val file = Files.createFile(path.resolve(part.originalFileName!!))
Files.copy(part.streamProvider(), file) // The file is not saved for some reason
```

## File Download

In the case of file downloading, there is no particular logic, so the code is mostly Ktor-only. I only use Path because I like it, so I'll just show you the code. One thing to keep in mind, as with uploading, is to encode the file name as a URL path when returning it.

```kotlin
get(ENDPOINT_DOWNLOAD) {try {
    val filepath = call.request.queryParameters["filepath"] ?: ""
    val path = FileService.getFullPath(filepath) // Get the full path under the root directory
    if (Files.notExists(path)) {
        call.respond(HttpStatusCode.BadRequest, "File not found")
    } else {
        call.response.header(
            name = HttpHeaders.ContentDisposition,
            value = ContentDisposition.Attachment.withParameter(
                key = ContentDisposition.Parameters.FileName,
                value = path.fileName.toString().encodeURLPath()
            ).toString()
        )
        call.respondFile(path.toFile())
    }
}
```

## Points to note

First, if you want to use both Kotlin/JS and Kotlin/JVM in one project, you can specify the following in the `build.gradle.kts` file for what is written as `dependencies`.

```kotlin
kotlin {
    sourceSets {
        // Kotlin/JS dependencies
        val jsMain by getting {
            dependencies {
                implementation(compose.web.core)
                implementation(compose.runtime)
                // ...omitted
            }
        }
        // Kotlin/JVM dependencies
        val jvmMain by getting {
            dependencies {
                implementation("io.ktor:ktor-server-core-jvm:$ktor_version")
                implementation("io.ktor:ktor-server-auth-jvm:$ktor_version")
                // ...omitted
            }
        }
    }
}
```

However, in order to use Compose, it had to be specified as `plugin`, and this was supposed to be added to the project-wide dependencies. Therefore, to create an app, we first build Frontend's Compose, and when we start the server, the built file is provided as static, but the Compose runtime is also required to start the Backend. If this runtime is not added, an error will be thrown and Ktor will not be able to start. There may be other ways to add the plugin to the Kotlin/JS-only dependencies, but for now I was able to resolve the problem by adding the runtime to the JVM dependencies as follows.

```kotlin
val jvmMain by getting {
    dependencies {
        implementation("io.ktor:ktor-server-core-jvm:$ktor_version")
        implementation("io.ktor:ktor-server-auth-jvm:$ktor_version")
        // ...omitted
        implementation(compose.runtime) // Compose runtime
    }
}
```

## others

What I was most happy about was that when dealing with Kotlin/JS and Kotlin/JVM as one project, we could share the code by creating a package called `common`. For example, by defining a JSON object as a data class and placing it in the common package, you can use the same object in both Frontend and Backend. Of course, you can also share Enums and consts, so it was quite easy to implement.

Also, although I didn't use it this time, Ktor Server supports something called [Type-safe Routing](https://ktor.io/docs/type-safe-routing.html#resource_classes), so I thought it would be great if I could make good use of it. This is also supported by Ktor Client as [Type-safe Request](https://ktor.io/docs/type-safe-request.html#define_route), so it is a feature that can be used on both Frontend and Backend. If I have a chance to use Ktor again, I would definitely like to use it.

## lastly

I wasn't able to improve file uploading as much as I had hoped, so it may be a while before the app is complete, but it was a very interesting experience. There are many things you can do with Kotlin, so if there's something you want to create, you'll want to give it a try. However, since it is still an immature technology, I feel that it cannot be used at the production level because unexpected problems may occur and there are not many references.

The entire app code is published on GitHub, so you can find it [here](https://github.com/retheviper/FileTransporter).

See you soon!
