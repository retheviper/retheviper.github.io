---
title: "Building a Desktop App with Kotlin"
date: 2022-09-09
translationKey: "posts/kotlin-compose-desktop"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - compose
  - gui
---

When developing backends, there may be times when test automation cannot be used. In an ideal scenario, you would be able to create everything from unit tests to integration tests that assume all scenarios, and those in the development and planning departments would be able to fully understand them, but in reality, this is quite difficult. In particular, I understand that as a service grows, the initial specifications keep changing due to attempts to eliminate technical debt, add new features, require modifications that could not be done in the past, or require operational irregularities, and it tends to be difficult for engineers who make modifications and those who operate the service to judge what the current state is and how it should change.

Therefore, there is a good chance that there will be a need to manually verify whether the current application works as expected. If that happens, you will also need to think about how to conduct the test. There are various testing methods, and it would be a good idea to perform unit tests on small functional units and eventually move up to integration tests and scenario tests, but there may be cases where it is difficult to automate all of this. For example, there may be cases where it is necessary to prepare various data patterns for testing, or engineers may not fully understand the specifications. Therefore, there may be situations where manual testing (monkey testing) is necessary.

This time, I will talk about how I created a testing tool to help with that ``human testing.'' The system I was working on was a type of microservice, and the business specifications were complex, so I had to test the functionality with various patterns. So, as an engineer, I decided to proceed with the implementation while at the same time creating a tool that would allow people familiar with business specifications to conduct tests using various patterns of data.

## Goal, design, and technology selection

In fact, testing tools already existed in the repository for some time. However, when I looked into it, I found that it had been abandoned for over two years and was made using Ruby on rails, a framework that I had never touched (or had any interest in). In this case, I might have learned Ruby and modified the existing tool, but I decided to start from scratch for the following reasons.

1. It is not good for Kotlin engineers to maintain Ruby apps.
2. Poor documentation and inconvenient to use

After making this decision, we decided to compile the functions required for the tool in response to requests from those who wanted to conduct the test (planning). Since we had a clear goal of a tool that could be tested, the specifications were very simple. The required functions that the tool should have are as follows.

- Load test data file
- Call backend app API
- Write API execution results to a file

As a test tool, if it meets the requirements listed above, it passes as a test tool. However, the people who want to perform the actual tests are not engineers, and there is a possibility that non-engineers will continue to use the tools in the future. With this in mind, we have set the following additional goals.

- Make it usable without building an environment
- Make it so simple that it can be used without a manual

Once you've decided on this, it's time to dig into the details of the required functionality. It's not enough to write a design document, but it's like a basic design.

- Reading and writing data
  - People who can use tools can use SQL = can read tables.
  - It is easier to understand if data can be input and output in the form of a table.
  - Load test data in CSV
  - Also write API execution results to CSV
- API calls can be made
  - GET/POST with HTTP client
  - API calls require tokens
    - It is not allowed to embed the token in the source code due to security issues.
    - But it's a pain to enter it every time.
    - Enter a token the first time you run the app, and then reuse that token from next time.
- Targets non-production environments
  - There are multiple environments so you can choose one
  - This is also a pain to enter every time.
  - I want to be able to select it only once at the beginning.

Then, I look back on the goals I have set and consider whether I can achieve the above requirements. If you can use it without building an environment, it would be better to provide it as a single executable binary. Also, if it is easy to use, it is better to include a GUI. In particular, I thought that if we adopted a GUI, we would be able to complete the functions in a fairly good manner in accordance with the requirements.

For example, reading or writing a file requires specifying a path, and selecting a token and environment also requires input, which is still inconvenient with CLI. Even more so if someone who is not an engineer touches it. If you are a Windows user, you may also have to consider the command line. On the other hand, with a GUI, you can use a dialog for file paths, a text box for inputting tokens, and a pool-down menu for selecting environments. Therefore, I came to the conclusion that I would create a GUI application that can be started by executing a binary.

## Compose for Desktop

Once the specifications and technical requirements for the test tool have been determined, the next step is technology selection. First of all, regarding which language to use, I chose Kotlin because there is a high possibility that the same team other than myself, that is, Kotlin engineers, will continue to maintain it. By using Kotlin, it becomes easier to select the libraries needed to implement the functions. Since there is already an HTTP client used in the backend app being tested, some of the code should be able to be ported as-is. Also, using the same library will make maintenance easier.

The rest is the GUI, and this time I decided to use [Compose for Desktop](https://www.jetbrains.com/lp/compose-desktop/). Since Kotlin is compatible with Java, you naturally have the option of using Java GUI toolkits such as [Swing](https://en.wikipedia.org/wiki/Swing_(Java)) and [JavaFX](https://openjfx.io/) as they are. There are other options like [TornadoFX](https://github.com/edvin/tornadofx), but there were several reasons I chose Compose this time.

First of all, I'm personally interested in mobile and have been interested in it for a while, so I wanted to seriously try making an app using it, but if Kotlin engineers are going to maintain it in the future, I think most of them will have experience with mobile, or at least have an interest in it. Although Compose is a new product that has only been officially released for about a year, it is a so-called "declarative" framework that has become popular recently, so we decided that it would have a good chance of becoming mainstream, at least in Android app development.

Also, Compose was developed not only for mobile, but also for multi-platforms, so it was attractive because it could build executable binaries regardless of Windows/Mac/Linux environment. With this, testers can use the tool in the same way no matter what OS they are using.

However, since it's a technology I've never had much experience with, there was a lot of study as well as trial and error, so I'd like to list a few points that I thought it would be good to keep in mind while creating a test tool.

## State Management

I mentioned state management in [SwiftUI post](../swift-ui-first-impression-2/), but Compose also deals with GUI, so state management is important. In this case, the app only has one screen, so I thought there was no need to manage the status across multiple screens, but there were still some things that needed to be managed as shared status across the app in order to perform processing.

However, as mentioned in the above post, the state management method is slightly different between SwiftUI and Compose. In SwiftUI, if the annotations and classes that are clearly used change depending on where the state is used, in Compose, the combination of [remember()](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary?hl=ja#remember(kotlin.Function0)) and [`MutableState<T>`](https://developer.android.com/reference/kotlin/androidx/compose/runtime/MutableState?hl=ja) is usually sufficient. The fact that Compose uses the same minimum unit of screen components as widgets means that it is easier to define them than in SwiftUI, but I felt that you needed to be a little more careful in how you used them.

First, states in Compose can be defined in three ways:

```kotlin
// Defined with delegation
var isOn: Boolean by remember { mutableStateOf(false) }
// The value can be updated directly
isOn = false

// Defined with destructuring
val (isOff: Boolean, setIsOff: (Boolean) -> Unit) = remember { mutableStateOf(true) }
// Reading and updating are separated
if (isOff) {
    setIsOff(!isOff) // toggle
}

// Treat it as MutableState<T>
val isNotOff: MutableState<Boolean> = remember { mutableStateOf(false) }
// Because it is wrapped, you need to access .value to update it
isNotOff.value = !isNotOff.value
```

Defining it as `var` in Delegate is the easiest to use, but it tends to cause compilation errors on Intellij. The reason is that to use Delegate, [androidx.compose.runtime.setValue](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#(androidx.compose.runtime.MutableState).setValue(kotlin.Any,kotlin.reflect.KProperty,kotlin.Any)) and [androidx.compose.runtime.getValue](https://developer.android.com/reference/kotlin/androidx/compose/runtime/package-summary#(androidx.compose.runtime.State).getValue(kotlin.Any,kotlin.reflect.KProperty)), which is not done automatically. I feel like there are many cases where you don't use it because you don't know the reason for this error at first, or when you're busy and it's a hassle to write import statements one by one. However, this is because Intellij's Compose support is not yet perfect, so we can expect this to be resolved eventually.

I think there are cases where it is difficult to know where to use it when referencing and updating values ​​separately in a decomposition declaration, but there are cases where it is used when passing the state to some widgets in Compose. A typical example is [TextField](https://developer.android.com/reference/kotlin/androidx/compose/material/package-summary#TextField(kotlin.String,kotlin.Function1,androidx.compose.ui.Modifier,kotlin.Boolean,kotlin.Boolean,androidx.compose.ui.text.TextStyle,kotlin.Function0,kotlin.Function0,kotlin.Function0,kotlin.Function0,kotlin.Boolean,androidx.compose.ui.text.input.VisualTransformation,androidx.compose.foundation.text.KeyboardOptions,androidx.compose.foundation.text.KeyboardActions,kotlin.Boolean,kotlin.Int,androidx.compose.foundation.interaction.MutableInteractionSource,androidx.compose.ui.graphics.Shape,androidx.compose.material.TextFieldColors)), whose purpose is immediately obvious when you look at the code. In actual code, it is used as follows.

```kotlin
val (text: String, updateText: (String) -> Unit) = remember { mutableStateOf("") }
TextField(
    onValueChange = updateText, // Update text when the user types
    value = text // Show the current text value
)
```

Finally, in the case of defining it as `MutableState<T>`, it is the most inconvenient to use because the value cannot be updated directly, but I think it is actually used most often. The reason is that if you want to use multiple widgets, such as sharing the state across the entire app, you will need to define `MutableState<T>` as a field in the class as shown below.

```kotlin
// Define state in the class so it can be shared across the whole app
class AppState(
    val isOn: MutableState<Boolean>
)
```

Of course, if you define a separate [getter/setter](https://kotlinlang.org/docs/properties.html#getters-and-setters) in the class, you can use it as if you were directly accessing the property without having to access the value inside. The image is as follows. I think the disadvantage of this is that the more items you want to manage as state, the more the amount of code increases, which is cumbersome.

```kotlin
class AppState(
    private val _isOn: MutableState<Boolean>
) {
    var isOn: Boolean
        get() = _isOn.value
        set(value) { _isOn.value = value }
}
```

In this way, there are various ways to define states in Compose, and each has its own characteristics, so I feel that it is most important to consider the appropriate definition method depending on the situation in which it will be used.

## Swing/AWT

One of the features of Compose for Desktop is that it is compatible with Swing and [AWT](https://en.wikipedia.org/wiki/Abstract_Window_Toolkit). At first, it also supported [Mac tray, menu bar, notifications](https://github.com/JetBrains/compose-jb/blob/master/tutorials/Tray_Notifications_MenuBar_new/README.md), so I thought it had all the basic functions, but that was not the case, and in some cases some functions had to be implemented by borrowing functions from Swing and AWT. In fact, some of the test tools I created use Swing and AWT functions.

For example, consider file selection. I wanted to show a file picker for reading CSV files, but Compose did not yet provide an appropriate widget, so I had to use AWT's [FileDialog](https://docs.oracle.com/javase/8/docs/api/java/awt/FileDialog.html). Below is an example.

```kotlin
 // Keep the selected file name as state
var fileName by remember { mutableStateOf("") }
// Use the AWT file selection dialog
FileDialog(ComposeWindow()).apply {
        // Limit selection to CSV files only
        setFilenameFilter { _, name -> name.endsWith(".csv", ignoreCase = true) }
        isVisible = true
        // Update the state when a file is selected
        if (!file.isNullOrBlank()) {
            fileName = file
        }
    }
```

However, sometimes this was not enough. `FileDialog` was not a good choice if you wanted to be able to select only folders. As the name suggests, it is intended to be used to select files, so it was not possible to select only folders. Therefore, in order to be able to select only folders, we also need to borrow the power of Swing. In that case, you can implement it as follows.

```kotlin
// Keep the selected folder path as state
var selectedPath by remember { mutableStateOf("") }
// Configure the Swing file chooser so only directories can be selected
val fileChooser = JFileChooser().apply {
        dialogTitle = "Choose Directory"
        fileSelectionMode = JFileChooser.DIRECTORIES_ONLY
    }
// Show the dialog
if (fileChooser.showOpenDialog(ComposeWindow()) == JFileChooser.APPROVE_OPTION) {
    // If the selected path differs from the stored state, update it with the absolute path of the chosen directory
    val path = fileChooser.selectedFile.absolutePath
    if (selectedPath != path) {
        selectedPath = path
    }
}
```

This time I only had to rely on Swing or AWT for these two use cases, but they are good examples of the kinds of gaps you may still need to fill with other APIs depending on the application. Compose had only been out for about a year at that point, so I expected this area to improve over time.

## Build

One of the reasons I chose Compose was that it was able to build binaries, and I was quite satisfied with it. Using `gradle`, an executable binary is generated with a single command. When you build and view it on a Mac, a package will be generated just like any other app. When I looked inside, it contained the JRE and dependency Jars required for execution, and was structured to be started on the JVM rather than natively.

There are various options when building a binary, and you can use different icons depending on the OS type (Windows, Mac, Linux), or specify to include modules that are not normally included. Below is a sample of actual build options.

```kotlin
compose.desktop {
    application {
        mainClass = "com.testtool.MainKt" // Specify the main class at runtime
        nativeDistributions {
            packageName = "Test Tool"
            packageVersion = "1.0.0"
            modules("java.sql", "java.naming") // Add packages that are not included by default
            macOS {
                iconFile.set(project.file("misc/appicon/icon-mac.icns"))
            }
            windows {
                iconFile.set(project.file("misc/appicon/icon-win.ico"))
            }
        }
    }
}
```

However, you need to be careful when building. Compose uses [jpackage](https://docs.oracle.com/en/java/javase/14/docs/specs/man/jpackage.html) internally when building, so you first need Java 15 or higher. Also, since different JDKs are installed depending on the CPU architecture, it is not possible to target a machine that uses a CPU with a different architecture than the building machine.

In other words, if you use a Mac, a binary for Apple Silicon will be generated, and if you use a Mac with an Intel chip, a binary for x64 will be generated. It seems that some people have actually submitted apps to the App Store using Compose, but they are building them using Macs with Intel chips because they can run them on Rosetta. If you want to create a Universal Binary, it seems that you have no choice but to wait for the JDK itself to be provided as a Universal Binary first.

## lastly

This time, I created a simple desktop app with very limited functions and simple logic, so if I were to use Compose again to implement it with various functions (multi-window, dark mode support, navigation, etc.), I feel like I might discover many more things. Personally, it was a very informative experience, and the implementation was not as difficult as I expected, so I would like to use Compose again if I have a chance to extend the functionality of the tool or create a new tool.

It hasn't been long since it was released, and there are a lot of missing features and information, and there are a lot of competing frameworks, so it's hard to know what the future holds, but if you're an engineer like me who mainly uses Kotlin and you're interested in GUIs, I would recommend that you give Compose a try at least once.

See you soon!
