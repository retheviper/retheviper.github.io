---
title: "Create a class that handles external configuration files"
date: 2019-11-24
translationKey: "posts/java-yaml-for-configuration"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - yaml
  - java
---
When creating an application, one of the things you need to consider is "creating a configuration file." If the configuration data is inside the application, there is no way to change it without decompiling it. Therefore, in order to increase the flexibility of the application, settings that need to be changed dynamically are often placed in separate files outside the application. It's like a save file in a game.

There are already many ways to read external files, but this time I will share the approach I use myself. One easy option is to write the external configuration file in [YAML](https://en.wikipedia.org/wiki/YAML), read it when the program starts, and keep it as a field in a singleton class. That has the advantage of performing I/O only once while still allowing the values to be referenced from anywhere. I chose YAML because it is easy to write and read, and because I often use it with Spring. JSON also works, but it becomes harder to read as the number of settings grows.

Anyway, what we need to prepare is a library for reading YAML. Here we use [SnakeYaml](https://bitbucket.org/asomov/snakeyaml/src/default). You can use other libraries as it just reads the file and converts it into an object.

First, let's say we have a YAML file like the one below.

```yaml
Development:
  id: "develop"
  version: 8
  use_local_storage: true
Debugging:
  id: "debug"
  version: 11
  use_local_storage: false
```

I wrote a simple scenario where I want to use different values in two cases: Development and Debugging. The code to use this is below.

```java
// Shared configuration information class
public final class Settings {

    // Since this is a singleton class, it keeps its own instance
    private static final Settings UNIQUE_INSTANCE = new Settings();

    // Field used to temporarily hold values read from the configuration file
    private static final Map<String, Map<String, Object>> SETTINGS_FROM_FILE = new HashMap<>();

    // Configuration file name
    private static final String SETTINGS_FILENAME = "application-settings.yml";

    // Map holding development configuration
    @Getter(lazy = true)
    private final Map<String, Object> developmentSettings = setDevelopmentSettings();

    // Map holding debugging configuration
    @Getter(lazy = true)
    private final Map<String, Object> debuggingSettings = setDebuggingSettings();

    // Load the file from the initializer block
    static {
        // Read the specified file
        ClassLoader classloader = UNIQUE_INSTANCE.getClassLoader();
        URL resource = classloader.getResource(SETTINGS_FILENAME);
        try (InputStreamReader reader = new InputStramReader(resource.openStream())) {
            // Parse the loaded YAML file and store the values in a Map
            Yaml yaml = new Yaml();
            Map<String, Map<String, Object>> importedMap = autoCast(yaml.load(reader));
            // Move the loaded values into the field Maps
            for (Map.Entry<String, Map<String, Object>> entry : importedMap.entrySet()) {
                SETTINGS_FROM_FILE.put(entry.getKey(), entry.getValue());
            }
        } catch (IOException e) {
            // Exception handling
        }
    }

    // Constructor cannot be accessed from outside
    private Settings() {
    }

    // Method used by the lazy getter to create the instance on first request
    private Map<String, Object> setDevelopmentSettings() {
        return Collections.unmodifiableMap(SETTINGS_FROM_FILE.get("Development"));
    }

    // Method used by the lazy getter to create the instance on first request
    private Map<String, Object> setDebuggingSettings() {
        return Collections.unmodifiableMap(SETTINGS_FROM_FILE.get("Debugging"));
    }

    // Getter used to retrieve the instance from outside
    public static Settings getInstance() {
        return UNIQUE_INSTANCE;
    }

    // Method that makes object casting easier
    @SuppressWarnings("unchecked")
    private static <T> T autoCast(final Object object) {
        return (T)object;
    }
}
```

Here, we use Lazy Getter to create instances of the Development and Debugging configuration fields when a get request is received. The reason for this is that after reading the file in the initializer block, I want to enter values ​​into individual fields + I want the fields to be private final + I want the Settings class to be instantiated as a singleton when the program starts. By setting a Lazy Getter, you can ensure that the field is initialized later while maintaining private final, and once the instance is generated, it remains as a cache, which is convenient. If this is not the case and you try to initialize it with a constructor or field, be careful because that point will be before the file is loaded.

## Finally

Using the code introduced this time, the configuration class will be instantiated as a singleton class when the application starts, and at that time the file will be read and placed in memory as a Map, and fixed values can be retrieved from anywhere using Getters. If your YAML settings need to be nested deeper, have more items, or change file names, you can easily change them by simply adjusting the Map and fields.

It is important to write code that not only works, but also is easy to maintain and repair, and is efficient, so I think it is necessary to consider a system like this (even if it is not the same as the one I introduced) in order to handle data that changes frequently. We need to continue to be aware of what constitutes good code.
