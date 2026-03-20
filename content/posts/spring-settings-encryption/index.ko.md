---
title: "Jasypt로 프로퍼티를 암호화하기"
date: 2020-03-16
categories:
  - spring
image: "../../images/spring.webp"
tags:
  - yaml
  - java
  - spring
translationKey: "posts/spring-settings-encryption"
---

Spring에서는 `application.properties`나 `application.yml`에 설정값을 따로 분리해 둘 수 있습니다. 이렇게 데이터와 애플리케이션을 분리하면, 하드코딩된 값을 직접 고쳐야 하는 불편을 줄일 수 있습니다.

다만 외부 설정 파일에 민감한 값을 그대로 적어 두는 것은 보안상 부담이 큽니다. 예를 들어 DB 자격 증명이나 내부 업무와 관련된 정보가 평문으로 들어 있다면, 해킹이 아니더라도 정보 노출로 이어질 수 있습니다. 이런 설정값을 암호화할 수 있다면 좋을 텐데, 이를 쉽게 해 주는 도구가 바로 [Jasypt](http://www.jasypt.org)입니다.

Jasypt를 쓰면 평문을 암호화하고, 암호문을 다시 복호화해 사용할 수 있습니다. Spring Boot Starter도 제공되기 때문에 이미 만들어 둔 애플리케이션에 붙이는 것도 어렵지 않습니다. 이번 글에서는 Spring Boot에 Jasypt를 도입하는 흐름을 간단히 정리해 보겠습니다.

## Jasypt 암호화 흐름

Jasypt를 이용한 Spring Boot 애플리케이션의 설정값 암호화 흐름은 대략 다음과 같습니다.

1. Encryptor 클래스를 Bean으로 등록한다
1. Encryptor로 평문을 암호화한다
1. YAML에 암호문을 적는다
1. 애플리케이션 시작 시 YAML의 암호문을 복호화해 사용한다

Jasypt에는 기본 Encryptor도 있지만, 필요하면 직접 설정해서 쓸 수도 있습니다. 여기서는 그 방식으로 설명하겠습니다.

## 의존성 추가

Spring Boot 기준으로 의존성은 다음처럼 추가합니다.

Maven의 경우

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.2</version>
</dependency>
```

Gradle의 경우

```groovy
dependencies {
    implementation 'com.github.ulisesbocchio:jasypt-spring-boot-starter:3.0.2'
}
```

Jasypt 본체만 일반 Java 애플리케이션에서 사용할 수도 있지만, 이번 글은 Spring Boot 기준으로 설명하므로 설정이 더 간단한 스타터 방식을 사용하겠습니다.

## 커스텀 Encryptor 만들기

먼저 Spring Bean으로 Encryptor를 등록합니다. 여러 옵션이 있지만, 실제로 중요한 것은 비밀번호와 알고리즘입니다. 이 둘이 맞지 않으면 복호화할 수 없기 때문입니다. 여기서는 YAML 파일에 Encryptor 비밀번호를 따로 두고, 그 값을 읽어 Bean으로 등록하는 예를 보겠습니다.

```java
@Configuration
public class StringEncryptorConfig {

    // YAML에서 읽어 올 비밀번호
    @Value("${settings.jasypt.password}")
    private String password;

    // encryptorBean이라는 이름으로 Encryptor를 Bean 등록한다
    @Bean(name = "encryptorBean")
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword("password");
        config.setAlgorithm("PBEWithMD5AndDES");
        // 아래 항목은 필수는 아니다
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

이제 Encryptor Bean 등록은 끝났습니다. 다음은 외부 설정 파일 쪽 설정입니다.

## 외부 파일 설정

외부 설정 파일에는 Bean으로 등록한 Encryptor 이름과 암호화된 프로퍼티를 적습니다. Encryptor 이름이 맞지 않거나 아예 적혀 있지 않으면 애플리케이션 시작 시 오류가 납니다.

`application.properties`의 경우

```properties
jasypt.encryptor.bean=encryptorBean
properties.password=ENC(askygdq8PHapYFnlX6WsTwZZOxWInq+i)
```

`application.yml`의 경우

```yml
jasypt:
  encryptor:
    bean: encryptorBean
properties:
  password: ENC(askygdq8PHapYFnlX6WsTwZZOxWInq+i)
```

암호화된 값은 반드시 `ENC()`로 감싸야 합니다. 그렇지 않으면 Jasypt는 그 값을 그냥 문자열로 읽어 버리기 때문에 복호화되지 않습니다.

## 암호화 알고리즘

Jasypt 패키지 안을 보면 기본적으로 몇 가지 Encryptor가 미리 정의되어 있습니다. 문자열뿐 아니라 숫자나 바이너리도 암호화할 수 있어서, 필요하면 Spring이 아닌 일반 Java 애플리케이션에서도 활용할 수 있습니다.

여기서는 문자열 Encryptor만 소개하겠습니다. 미리 정의된 문자열 Encryptor는 다음과 같습니다.

```java
// 기본 Encryptor
public BasicTextEncryptor() {
    super();
    this.encryptor = new StandardPBEStringEncryptor();
    this.encryptor.setAlgorithm("PBEWithMD5AndDES");
}

// 더 강한 알고리즘을 쓰는 Encryptor
public StrongTextEncryptor() {
    super();
    this.encryptor = new StandardPBEStringEncryptor();
    this.encryptor.setAlgorithm("PBEWithMD5AndTripleDES");
}

// AES256을 쓰는 가장 강한 Encryptor
public AES256TextEncryptor() {
    super();
    this.encryptor = new StandardPBEStringEncryptor();
    this.encryptor.setAlgorithm("PBEWithHMACSHA512AndAES_256");
    this.encryptor.setIvGenerator(new RandomIvGenerator());
}
```

위 알고리즘은 앞서 만든 커스텀 Encryptor에도 그대로 적용할 수 있습니다. 다만 알고리즘이 강할수록 처리 비용이 커져서 애플리케이션 시작이 느려질 수 있습니다. AES256을 쓸 때는 `IvGenerator`도 함께 지정해야 한다는 점도 기억해 두면 좋습니다.

코드 안에서 직접 암호화하고 싶다면, Bean으로 등록한 Encryptor를 호출하거나 새 인스턴스를 만들어 `encrypt()`를 쓰면 됩니다. 당연하지만 같은 비밀번호를 써야 올바르게 암호화와 복호화가 됩니다.

## 명령줄 도구

Jasypt를 [다운로드](https://github.com/jasypt/jasypt/releases/download/jasypt-1.9.3/jasypt-1.9.3-dist.zip)하면 명령줄 도구로 암호화와 복호화를 할 수 있습니다. 압축을 풀면 `bin` 폴더 안에 `bat`와 `sh` 파일이 들어 있습니다. 각 파일의 기능은 다음과 같습니다.

1. `encrypt.sh(bat)`: 비밀번호를 이용해 평문을 암호화한다
1. `decrypt.sh(bat)`: 비밀번호를 이용해 암호문을 복호화한다
1. `digest.sh(bat)`: 복호화할 수 없는 해시 코드를 생성한다
1. `listAlgorithm.sh(bat)`: 암호화에 사용할 수 있는 알고리즘 목록을 보여 준다

`encrypt`와 `decrypt`는 비밀번호와 암호화하거나 복호화할 문자열을 커맨드라인 인자로 넘기면 결과를 표준 출력으로 보여 줍니다. 옵션으로 사용할 알고리즘도 지정할 수 있습니다. 사용 예시는 아래와 같습니다.

```shell
bin % ./encrypt.sh password=password input=this_is_input
```

출력 결과는 다음과 같습니다.

```shell
----ENVIRONMENT-----------------

Runtime: AdoptOpenJDK OpenJDK 64-Bit Server VM 11.0.6+10 



----ARGUMENTS-------------------

input: this_is_input
password: password



----OUTPUT----------------------

2lgKlL4gECBBtjch4WZITWDBHWhIxvVz
```

또 `listAlgorithm`을 실행하면 현재 시스템에서 사용할 수 있는 알고리즘 목록이 출력됩니다.

```shell
PBE ALGORITHMS:      [PBEWITHHMACSHA1ANDAES_128, PBEWITHHMACSHA1ANDAES_256, PBEWITHHMACSHA224ANDAES_128, PBEWITHHMACSHA224ANDAES_256, PBEWITHHMACSHA256ANDAES_128, PBEWITHHMACSHA256ANDAES_256, PBEWITHHMACSHA384ANDAES_128, PBEWITHHMACSHA384ANDAES_256, PBEWITHHMACSHA512ANDAES_128, PBEWITHHMACSHA512ANDAES_256, PBEWITHMD5ANDDES, PBEWITHMD5ANDTRIPLEDES, PBEWITHSHA1ANDDESEDE, PBEWITHSHA1ANDRC2_128, PBEWITHSHA1ANDRC2_40, PBEWITHSHA1ANDRC4_128, PBEWITHSHA1ANDRC4_40]
```

이 목록의 알고리즘은 Encryptor Bean을 등록할 때도 그대로 사용할 수 있으므로, 요구사항에 맞는 것을 고르면 됩니다. 다만 강한 알고리즘을 쓸수록 복호화 비용이 커지고, Spring Boot는 시작 시점에 설정 파일을 읽기 때문에 애플리케이션 기동 시간에도 영향을 줄 수 있습니다.

## 마지막으로

애플리케이션에서 보안의 중요성은 아무리 강조해도 지나치지 않습니다. Spring 설정 파일에는 DB나 외부 시스템 연동용 접속 정보처럼 민감한 값이 자주 들어갑니다. 이런 설정 파일이 그대로 노출되면 바로 문제가 될 수 있으니, 평소 관리에 신경 쓰는 것과 별개로 암호화 같은 방어 수단도 함께 준비해 둘 필요가 있습니다.

특히 Jasypt의 Encryptor는 외부 설정 파일뿐 아니라 코드 안에서도 쓸 수 있어서 활용 범위가 넓습니다. 민감한 정보를 다루는 애플리케이션이라면 적극적으로 활용해 볼 만합니다. 성능도 안정성도 중요하지만, 무엇보다 정보가 새지 않도록 하는 것이 먼저입니다.
