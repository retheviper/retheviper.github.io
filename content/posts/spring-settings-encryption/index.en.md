---
title: "Encrypt properties with Jasypt"
date: 2020-03-16
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - yaml
  - java
  - spring
---
With Spring, data can be exported from the application by defining additional settings in the application.properties and application.yml files. This separation of data and application frees you from the inconvenience of hard-coding and modifying the application itself when changes occur.

However, writing various setting values ​​in separate external files like this may lead to security issues. For example, if credential information such as a database or code related to a company's business is written in plain text, there is a good chance that it will become a security problem even if it is not hacking. In such cases, it would be a good idea to be able to encrypt the various setting values ​​that you want to record. There is already an API that makes this possible, and the one I would like to introduce today is called [Jasypt](http://www.jasypt.org).

Jasypt allows you to encrypt plaintext and decrypt ciphertext back to plaintext. Spring Boot Starter is also provided, making it easy to add encryption functionality to already created Spring Boot applications. In this post, I will explain how you can use Jasypt to bring encryption and decryption functionality to your Spring Boot applications.

## Encryption flow with Jasypt

The encryption and decryption of configuration values written in an external file of a Spring Boot application using Jasypt is as follows.

1. Register the Encryptor class as a bean
2. Encrypt plaintext with Encryptor class
3. Describe the ciphertext in YAML
4. Use decrypted YAML ciphertext when starting the application

Some Encryptor classes provided by Jasypt are provided as default classes, but they can also be customized, so this time I will introduce how to do that.

## Add dependencies

Based on Spring boot, add dependencies as follows.

For Maven

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.2</version>
</dependency>
```

For Gradle

```groovy
dependencies {
    implementation 'com.github.ulisesbocchio:jasypt-spring-boot-starter:3.0.2'
}
```

Although it is possible to use just Jasypt itself in a normal Java application, this time we are using a Spring Boot application as the standard, so we will introduce one that is easier to set up.

## Create a custom Encryptor

First, register Encrytor as a Spring bean. You can specify various options for the Encryptor, but what really matters are the password and algorithm. This is because if the password and algorithm do not match, it cannot be combined. Here, we will introduce the code for writing the password for the Encryptor as a custom setting in the YAML file and registering the Encryptor as a bean using that password.

```java
@Configuration
public class StringEncryptorConfig {

    // Password read from YAML
    @Value("${settings.jasypt.password}")
    private String password;

    // Register an Encryptor bean with the name encryptorBean
    @Bean(name = "encryptorBean")
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword("password");
        config.setAlgorithm("PBEWithMD5AndDES");
        // The following items are optional
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setProviderName("SunJCE");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setStringOutputType("base64");
        encryptor.setConfig(config);
        return encryptor;
    }
}
```

The Encryptor bean registration settings are now complete. Next is the external configuration file settings.

## External file settings

In the external configuration file, specify the name of the Encryptor registered as a bean and write the encrypted properties. Please note that if the Encryptor name does not match or is not listed, an error will occur when starting the application.

For application.properties

```properties
jasypt.encryptor.bean=encryptorBean
properties.password=ENC(askygdq8PHapYFnlX6WsTwZZOxWInq+i)
```

For application.yml

```yml
jasypt:
  encryptor:
    bean: encryptorBean
properties:
  password: ENC(askygdq8PHapYFnlX6WsTwZZOxWInq+i)
```

As some of you may have noticed, encrypted items must always be enclosed in `ENC()`. If you do not do this, Jasypt's Encryptor will read the setting value as it is as a string, so it will not be decoded.

## Encryption algorithm

If you follow the Jasypt package, you will see that basically several Encryptors are defined. Not only strings, but also numeric types and binary can be encrypted, so if necessary, you can import and use it in regular Java applications instead of Spring.

This time we will only introduce string encryption, but the Encryptor for this string is predefined as follows.

```java
// Encryptor used as the default
public BasicTextEncryptor() {
    super();
    this.encryptor = new StandardPBEStringEncryptor();
    this.encryptor.setAlgorithm("PBEWithMD5AndDES");
}

// Encryptor that uses a stronger algorithm
public StrongTextEncryptor() {
    super();
    this.encryptor = new StandardPBEStringEncryptor();
    this.encryptor.setAlgorithm("PBEWithMD5AndTripleDES");
}

// Strongest Encryptor using AES256
public AES256TextEncryptor() {
    super();
    this.encryptor = new StandardPBEStringEncryptor();
    this.encryptor.setAlgorithm("PBEWithHMACSHA512AndAES_256");
    this.encryptor.setIvGenerator(new RandomIvGenerator());
}
```

The algorithm described here can also be used as is in a custom Encryptor defined as a bean. However, the more complex the algorithm, the more difficult it will be to decrypt the encrypted result, so it will be safer, but it may also slow down the application startup, so choose the appropriate one depending on the situation. Also, note that if you use AES256, you also need to specify IvGenerator.

If you want to encrypt within your code, you can do so by calling an Encryptor registered as a bean, or by creating a new Encryptor instance and calling the encrypt() method. Of course, be aware that if you do not specify the same password, you will not be able to encrypt and decrypt correctly.

## Command line tools

If you [download](https://github.com/jasypt/jasypt/releases/download/jasypt-1.9.3/jasypt-1.9.3-dist.zip) Jasypt, you will be able to encrypt and decrypt data using command line tools. When you download the ditributable version from the link and unzip it, you will find a bat file and a sh file in the bin folder. The functions of the case file are as follows.1. encrypt.sh(bat): Password-based encryption of plaintext
2. decrypt.sh(bat): Decrypt ciphertext based on password
3. digest.sh(bat): Generates a hash code that cannot be decomposed
4. listAlgorithm.sh(bat): List the types of algorithms that can be used for encryption

For encrypt and decrypt, when you enter the password and the text you want to encrypt/decrypt as command line arguments, the results are displayed on the screen as standard output. You can also optionally specify the algorithm you want to use. The usage is the following command.

```shell
bin % ./encrypt.sh password=password input=this_is_input
```

The output of this command is below.

```shell
----ENVIRONMENT-----------------

Runtime: AdoptOpenJDK OpenJDK 64-Bit Server VM 11.0.6+10 



----ARGUMENTS-------------------

input: this_is_input
password: password



----OUTPUT----------------------

2lgKlL4gECBBtjch4WZITWDBHWhIxvVz
```

Also, when you run listAlgoritym, a list of algorithms that can be used on the current system will be output as shown below.

```shell
PBE ALGORITHMS:      [PBEWITHHMACSHA1ANDAES_128, PBEWITHHMACSHA1ANDAES_256, PBEWITHHMACSHA224ANDAES_128, PBEWITHHMACSHA224ANDAES_256, PBEWITHHMACSHA256ANDAES_128, PBEWITHHMACSHA256ANDAES_256, PBEWITHHMACSHA384ANDAES_128, PBEWITHHMACSHA384ANDAES_256, PBEWITHHMACSHA512ANDAES_128, PBEWITHHMACSHA512ANDAES_256, PBEWITHMD5ANDDES, PBEWITHMD5ANDTRIPLEDES, PBEWITHSHA1ANDDESEDE, PBEWITHSHA1ANDRC2_128, PBEWITHSHA1ANDRC2_40, PBEWITHSHA1ANDRC4_128, PBEWITHSHA1ANDRC4_40]
```

The algorithms in this list are also the ones that can be specified when registering the Encryptor as a bean, so choose the appropriate one according to your needs. Using powerful algorithms can also slow down application startup. (Since the YAML file of the Spring Boot application is loaded at startup)

## Finally

It goes without saying that security is of high importance when creating applications. As I mentioned earlier, Spring configuration files often contain sensitive information such as connection information for DB and external system linkage, so if external configuration files were to be leaked as is, it could cause problems. Of course, it is important to be careful not to get angry about such things, but I think that protecting information through encryption is also a good method.

In particular, Jasypt's Encryptor can be used not only in external configuration files but also in code, so it has a wide range of uses. If you are dealing with sensitive information, you should actively use it in your applications. Performance and stability are important, but above all, it's important not to leak information.
