---
title: "I/O에서 NIO로"
date: 2020-01-20
categories:
  - java
image: "../../images/java.webp"
tags:
  - nio
  - java
translationKey: "posts/java-nio"
---

저도 그랬지만, 처음 Java의 I/O를 배우면 보통 `File` 객체를 만들고 `InputStream`으로 읽거나 `OutputStream`으로 쓰는 방식부터 익히게 됩니다. 조금 더 나아가면 `Writer`와 `Reader`를 쓰고, Stream을 Buffer로 감싸거나, Serializable로 객체 I/O를 다루게 됩니다.

옛 API라고 해도 동작이나 성능에 큰 문제가 없다면 굳이 새 API로 전부 갈아엎을 필요는 없습니다. 무리하게 새 API로 다시 쓴 코드가 더 문제를 만들 수도 있습니다. 예를 들어 Java 1.8에서 추가된 `forEach()`는 편리하지만, 전통적인 `for` 루프에 비해 성능이 떨어진다는 이야기도 있습니다. 물론 앞으로 좋아질 가능성은 있지만, 기능이 추가된 시점과 최적화 시점은 항상 같지 않습니다.

새 API를 쓸 때는 이런 점을 고려해야 하지만, 그럼에도 새 API가 나온 이유는 분명 있습니다. 코드를 더 단순하게 쓰거나, 하고 싶은 일을 더 명확하게 표현하기 위해서입니다. 이번에 소개할 NIO도 그런 경우입니다. 파일 I/O를 다루는 방식이 조금 달라지고, 코드도 더 짧아지는 편입니다. Java 1.7부터 들어왔으니 지금은 아주 새롭다고 하긴 어렵지만요.

## NIO란

NIO는 Java의 New I/O입니다. 보통 New의 약자라고 생각하기 쉬운데, 실제로는 Non-blocking의 뜻이 더 가깝다고 합니다. Java가 C/C++보다 느린 이유 중 하나로 I/O가 거론되었고, 그것을 개선하려고 등장한 것이 NIO입니다.

Blocking과 Non-blocking, Stream 기반과 Channel 기반 등 차이는 있지만, 일반적으로 I/O가 자주 발생하는 경우에는 NIO가 더 좋은 성능을 기대할 수 있습니다. 그 밖의 장점은 다음과 같습니다.

- 스레드 블로킹이 적다
- 코드가 더 간결하다
- 복사, 이동, 읽기 옵션 지정이 쉽다
- 기본적으로 Buffer를 쓰므로 별도 Buffered 래핑이 덜 필요하다

저는 JVM 내부 구조를 깊이 설명할 정도로 자신 있지는 않습니다. 다만 코드가 더 간결해지는 건 분명한 장점입니다. 실제 코드에서 NIO를 어떻게 쓰는지 보겠습니다.

## File 에서 Path 로

NIO는 `File`보다 `Path`를 주로 씁니다. `Path`의 가장 큰 장점은 파일 경로를 디렉토리와 파일 이름으로 나눠서 다룰 수 있다는 점입니다.

예를 들어 경로가 여러 폴더로 나뉘어 있다고 해 보겠습니다.

```java
// 여러 디렉토리와 파일이 각각 문자열로 존재한다
String rootDirectory = "path";
String toDirectory = "to";
String fileName = "file.txt";
```

`File` 객체를 만들려면 문자열을 직접 이어 붙여야 합니다.

```java
// File 객체 생성
File file = new File(rootDirectory + "/" + toDirectory + "/" + fileName);
```

반면 `Path`는 여러 문자열을 순서대로 넘기면 됩니다.

```java
// Path 객체 생성
Path path = Paths.get(rootDirectory, toDirectory, fileName);
```

그래서 생성 자체는 `Path`가 더 편합니다. 꼭 `File`이 필요한 경우도 `Path`와 쉽게 변환할 수 있습니다.

```java
// Path를 File로
Path path = file.toPath();

// File을 Path로
File file = path.toFile();
```

`toURI()` 같은 기능도 있어서, 전반적으로 `File`보다 `Path`가 더 유연합니다.

## Files

기존 I/O에서 복사, 삭제 같은 작업은 `InputStream`, `OutputStream`, `Writer`, `Reader`를 조합해서 처리했습니다. NIO에서는 이런 작업을 주로 `Files` 클래스로 합니다. `Writer`, `Reader`, `InputStream`, `OutputStream`을 만드는 기능도 있어서 꽤 편합니다.

### 파일 복사

`Files.copy()`는 간단합니다.

```java
Files.copy(source, target);
```

옵션도 줄 수 있습니다.

```java
StandardCopyOption option = StandardCopyOption.COPY_ATTRIBUTES;
Files.copy(source, target, option);
```

실제 파일이 아니라 `InputStream`을 원본으로 써도 됩니다.

```java
Files.copy(sourceStream, target);
```

### 파일 삭제

삭제도 `Path` 하나만 넘기면 됩니다.

```java
Files.delete(path);
```

있을 때만 삭제하고 싶다면 `deleteIfExists()`를 씁니다.

```java
Files.deleteIfExists(path);
```

### 파일 이동

이동은 복사와 삭제를 합친 느낌이고, 이름 변경에도 쓸 수 있습니다.

```java
Files.move(path);

StandardCopyOption option = StandardCopyOption.REPLACE_EXISTING;
Files.move(path, option);
```

### 파일 쓰기

`Files.write()`로 직접 쓸 수 있습니다.

```java
Files.write(path, content);
```

인수로는 `byte[]`나 `List<String>` 등을 줄 수 있고, 옵션도 지정할 수 있습니다.

```java
StandardOpenOption option = StandardOpenOption.APPEND;
Files.write(path, content, option);
```

### 파일 읽기

읽기 역시 문자열이나 `byte[]` 형태로 나뉩니다.

```java
Stream<String> lines = Files.lines(path);
List<String> liness = Files.readAllLines(path);

byte[] bytes = Files.readAllBytes(path);
```

디렉토리도 다룰 수 있습니다.

```java
Stream<Path> files = Files.list(path);
```

### 기존 I/O와 함께 쓰기

`Files`의 일부 메서드는 기존 I/O와 함께 쓸 수 있습니다.

```java
InputStream is = Files.newInputStream(path);
BufferedReader br = Files.newBufferedReader(path);

OutputStream os = Files.newOutputStream(path);
BufferedWriter bw = Files.newBufferedWriter(path);
```

옵션도 줄 수 있습니다.

```java
StandardOpenOption option = StandardOpenOption.CREATE;
InputStream is = Files.newInputStream(path, option);
```

## 마지막으로

같은 기능이라도 굳이 새 API를 쓸 이유가 없어 보일 수 있습니다. 그래도 NIO는 옵션을 더 명확하게 표현해 주고, 코드도 줄여 주는 편이라 한 번 익혀 두면 꽤 편합니다. `File`을 완전히 버릴 필요는 없지만, `Path`와 `Files`를 중심으로 사고하는 방식은 분명 장점이 있습니다.

`Channel` 관련 API나 파일 속성, 심볼릭 링크, 디렉토리 여부, 같은 파일인지 확인하는 기능도 많으니, 파일 처리를 자주 다룬다면 한 번쯤은 정리해 둘 만합니다.
