---
title: "Passing command line arguments from Gradle"
date: 2019-09-17
categories: 
  - gradle
image: "../../images/gradle.webp"
tags:
  - groovy
  - java
  - gradle
---
What I've been creating at work lately is a unique library. However, what was different from what I expected was that it was not a completely homemade product, but rather an improvement on an existing library. The second thing is that it is made with Spring Boot. This was my first time using Spring boot, but I had experience creating a website with Spring MVC before. However, since Spring is a framework for creating web applications, I had never imagined using it to create a library or create an application without web pages.

However, in my work I encountered the uses of both, and I realized that there were still many things in this world that I didn't know. Programming is not only a question of whether you can use a language or framework, but also a question of how to use it to create something. And even if there is no screen, by creating a Main class, you can run it independently from the Jar, and it is unique that you can import it and use it as a library. I decided to do something. And of course, doing so required some preparation.

Although the title is short, this is the subject of this article and the requirements for the job.

## Make a Spring boot application run with Gradle task and pass command line arguments at startup

It may seem complicated when you write it like this, but I don't think it's actually that difficult of a concept. First of all, Gradle is known as a library management tool like Maven, and by adding various options to `gradlew`, you can do things like build and test Jars. What we want to do here is to add tasks (options) that can be done with Gradle. And executing the added task will start the main class of the library that is created as a Spring Boot project.

However, as you can see, if a Main class is prepared in the Sping Boot project, it can be started normally from the command line using `java -jar project.jar`. However, the reason why I decided to include it in the Grale task is as follows.

## Has a multi-project structure

In the current design, the entire library project is a multi-project with multiple subprojects. I'm not sure if this will convey it properly, but anyway, if you express it in a table, it looks like the following.

```text
rootproject
┣ target
┣ generator
┗ runtime
```

To explain briefly, the current library uses a subproject called generator (runtime is then referenced from generator) to manipulate the code of the target project. Here, the generator and runtime are finally provided as a Jar, and the user uses it by placing the project to which he/she wants to apply the processing of this library in the target position. In this case, starting the generator Jar that depends on the runtime from the command line and specifying it to operate on the target source file is quite a tedious task. I wanted to automate this.

## Too many arguments to pass at startup

Another reason I wanted to run it as a Gradle task is that there are about seven types of arguments that the library needs to perform its processing, and it would be difficult to remember them all. Additionally, there is a higher possibility that processing will fail due to typos. At this point, I thought for a moment that the `.bat` file was prepared and the user could just modify it, but I guess that's not a good idea after all. Gradle itself is like this now, but preparing a separate file means you need to create that file for each OS. Since we don't know what environment the user will run it in, we need to prepare both `.bat` and `.sh` for now.

Also, I can't say for sure that this library will only be used on Windows or Linux, so there are more variables in those cases. If that happens, you'll need to create a method for passing command line arguments that suits each OS and environment. Rather than go through the trouble of doing that, I felt it would be better from the standpoint of code management and convenience to prepare it as a Gradle task (of course, write the arguments in a file and have them read) and leave the environment to Gradle. There is a sense of unity, and I think it will leave a good impression on users.

## [target] build.gradle

Let's start with the target project where you want to run the task. Gradle projects basically have a file called `build.gradle`, and by modifying this file, you can configure various settings such as dependencies and plugins. Similarly, if you want to add a custom task, enter the details of the task you want to add here.

```groovy
apply from: 'default.gradle' // This means the file is loaded

task taskname(type: GradleBuild) { // Configure task name and type
    group = 'application' // This task appears under the application tab in `gradlew tasks`
    description = 'run program' // Description of the task
    dir = '../generator' // Working directory for the task
    tasks = ['bootRun'] // What to execute
    startParameter.projectProperties = [args: '${defaultArgs}'] // Load defaultArgs as a command-line argument
}
```

The reason why `default.gradle` is loaded here is to write the default value in that file and use the loaded value as a command line argument when executing the task. The last line says `defaultArgs`, which is the default value as a variable. By separating the files in this way, when an actual user performs this task, they will only have to modify `default.gradle` to adjust only the values ​​passed as arguments.

## [target] default.gradleNext, the settings of the file read when executing the task with `build.gradle` are as follows. Here, we simply list the settings to be read and the command line arguments declared in the form of variables.

```groovy
ext { // Values to be loaded
    defaultArgs = '-arg1 value -arg2 value' // When there are two arguments, arg1 and arg2
}
```

This one is easy. Since the argument names are written, the order does not matter; all you need to do is change the values ​​for each.

## [generator] build.gradle

Next, let's configure the side that will be executed by the task. Since we specified the generator as `bootRun` in the target task, we will set the behavior at `bootRun` accordingly. For example, settings such as how arguments are received and what the main class is.

```groovy
bootRun { // Behavior during bootRun
    if (project.hasProperty('args')) {
        args project.args.split('\\s+') // If command-line arguments are provided, split them on whitespace
    }
}

jar { // Settings for the Jar
    manifest {
        attributes 'Main-Class': 'com.package.Main' // Specify the main class path (package and class name)
    }
}
```

The reason for splitting the command line arguments is, as you might have guessed, that Java main methods usually accept arguments as arrays of strings. If you divide it like this, it will be easier to parse the arguments during actual processing.

This completes the Gradle configuration. The only thing left to do is set up the main class in Java.

## [generator] Main.java

Create a main class that will be called when the generator Jar is executed. The code here is a mix of general Java main methods, main class manners for Spring boot, manners for parsing command line arguments with JCommander, and Lombok, so the code may be a little difficult for those who are not familiar with each.

However, the behavior when it is executed is simple, so if you are familiar with Spring annotations to some extent, you will be able to understand it easily. First, the code will look like this:

```java
@SpringBootApplication // Attach this to the main class for Spring Boot
@EnableAutoConfiguration(exclude = { DataSourceAutoConfiguration.class }) // Added because H2-related errors occurred
public class Main implements CommandLineRunner {

    @Autowired
    private CoreProcessClass coreProcessClass; // The actual processing class annotated with `@Component`

    @Override
    public void run(final String... args) throws Exception { // Override runtime behavior by implementing CommandLineRunner
        final CommandLineOptions options = CommandLineOptions.parse(args); // Create the bean while parsing
        coreProcessClass.startProcess(options.getArg1, options.getArg2); // Start the main process
    }

    public static void main(final String[] args) {
        SpringApplication.run(Main.class, args); // The main method only passes through the arguments
    }

    @Data
    public static class CommandLineOptions { // Class for parsing command-line arguments

        // Arguments configured with JCommander
        @Parameter(names = "-arg1", description = "File", required = true, converter = ArgumentsToFileConverter.class) // Since the argument is a string, use a converter class
        private File arg1;

        @Parameter(names = "-arg2", description = "String", required = true) // For a normal string value
        private String arg2;

        private CommandLineOptions() {}

        public static CommandLineOptions parse(final String... args) { // Method that performs parsing
            try {
                final CommandLineOptions options = new CommandLineOptions();
                JCommander.newBuilder()
                        .addObject(options)
                        .acceptUnknownOptions(true)
                        .build()
                        .parse(args);
                return options;
            } catch (ParameterException e) {
                e.getJCommander().usage();
                throw e;
            }
        }
    }

    public class ArgumentsToFileConverter implements IStringConverter<File> { // Class that converts arguments into objects with JCommander

        @Override
        public File convert(final String argument) {
            return new File(argument);
        }
    }
}
```

You can easily parse command lines using [JCommander](http://jcommander.org). Parsing here does not simply mean strings, but it also means that various things can be done, such as specifying it as a required item (if it is not present, it will be an exception) and converting it to an object. For example, if the string passed as an argument is a file path, you can read it and convert it into a File object, or get multiple arguments as a List.

Then, all you have to do is pass the parsed object to the object used in the main process. It's surprisingly easy to finish.

## Finally

Actually, the content I have explained so far was not something I created by thinking about it myself. At first, there was a [problem with the library](../java-conflict-of-module/) that I introduced to create a Gradle task, so I couldn't make much progress, so I just used something that was already completed as a reference. But I'm going to introduce it here because I think it's worth sharing with everyone.

Especially when creating an application that runs on a server, it is often started from the command line, and depending on the environment, it may be necessary to pass different values ​​as arguments. In such cases, it would be quite convenient to create a task that passes arguments from a file and executes it, changing the settings for each environment or reading a different file. So I wanted to introduce it to everyone.

Personally, I am currently learning from what my seniors have accomplished, but it was an important experience that made me wish I could someday be able to tell my juniors that this is the way they think. There is still a long way to go!
