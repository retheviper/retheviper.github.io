---
title: "다시 보는 파일 복사"
date: 2021-01-12
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
  - io
  - nio
translationKey: "posts/java-file-copy"
---

예전에 Java 1.7에서 들어온 NIO를 정리한 적이 있지만, Java의 파일 I/O는 여전히 손에 익지 않는 부분이 많습니다. 결국 좋은 코드를 쓰려면 먼저 지금 쓰는 방식에 어떤 문제가 있는지부터 인정해야 합니다. 그래서 이번에는 제가 그동안 별생각 없이 써 온 파일 복사 코드를 다시 돌아보고, 어떤 방식이 더 나은지 정리해 보려 합니다.

다만 파일 I/O 전체를 한 번에 다루기는 어렵기 때문에, 이번 글에서는 범위를 `Java 코드로 파일을 복사하는 경우`로 한정하겠습니다. 확인하고 싶은 질문은 다음과 같습니다.

1. InputStream과 OutputStream은 어떻게 만들 수 있습니까?
    1. FileInputStream과 FileOutputStream을 사용합니까?
    2. Files 메서드를 사용합니까?
2. 파일 복사는 어떤 방법을 사용하는 것이 좋은가?
    1. InputSteam에서 OutpuStream에 쓸까요?
        1. readAllBytes()
        2. transferTo()
    2. Files 메서드를 사용합니까?

이번 글에서는 이런 질문을 중심으로, 실제로 많이 쓰이는 파일 복사 코드를 하나씩 살펴보겠습니다.

## InputStream과 OutputStream 만들기

먼저 InputStream과 OutputStream부터 보겠습니다. 지금도 파일은 메모리에 한 번에 올리기보다 가능한 한 스트림으로 다루는 경우가 많습니다. 특히 Java가 많이 쓰이는 웹 애플리케이션에서는 Spring 같은 프레임워크도 업로드와 다운로드를 기본적으로 스트림 기반으로 처리합니다.

그런데 로컬 파일을 복사할 때 스트림을 만드는 방식은 Java 1.7 이후 기준으로 크게 두 가지입니다. 하나는 `File` 객체에서 `FileInputStream`과 `FileOutputStream`을 만드는 전통적인 방식이고, 다른 하나는 `Path`와 `Files` 클래스를 이용하는 방식입니다.

가능하면 앞으로는 NIO 스타일로 통일하고 싶지만, 정말 기존 방식까지 전부 바꿔야 하는지부터 먼저 확인해 보고 싶었습니다.

### 복사 방식

먼저 Java에서 `InputStream`과 `OutputStream`으로 파일을 복사하는 가장 익숙한 코드부터 보겠습니다.

저를 포함해 Java를 처음 배울 때 가장 먼저 접하게 되는 파일 복사 코드는 아마 아래와 같은 형태일 것입니다. 말하자면 가장 흔하고 전통적인 방식입니다.

```java
// byte[]를 사용하는 예
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

// BufferedStream을 사용하는 예
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

여기서 `FileInputStream`은 `Files.newInputStream`, `FileOutputStream`은 `Files.newOutputStream`으로 대체할 수 있습니다. 눈에 띄는 차이라면 전자는 `File`을 받고, 후자는 `Path`를 받는다는 점입니다. 다만 `File`과 `Path`는 서로 변환이 가능하므로, 이 차이만으로 어느 쪽이 절대적으로 낫다고 보기는 어렵습니다.

언뜻 보면 `Files` 클래스 쪽이 더 최신 API이니 성능도 더 좋을 것처럼 보입니다. 하지만 Java의 NIO가 언제나 기존 IO보다 빠른 것은 아닙니다. 실제로 파일 I/O에서는 NIO를 써도 결국 블로킹 방식으로 동작하기 때문에, 성능 차이가 크지 않다는 이야기도 자주 나옵니다.

그렇다면 특별히 문제만 없다면 기존 코드를 굳이 전부 NIO로 바꿀 필요는 없어 보이기도 합니다. 하지만 정말 그럴까요?

### FileInputStream 및 FileOutputStream 문제

하지만 꼭 그렇지만도 않습니다. `FileInputStream`과 `FileOutputStream`에는 성능과는 별개로 신경 써야 할 문제가 있습니다. 경우에 따라 GC와 얽혀 애플리케이션 전체가 오래 멈출 수 있다는 점입니다.

#### finalize()의 오버라이드 문제

GC 때문에 애플리케이션이 멈춘다는 말만 들으면, 결국 `close()`를 제대로 안 해서 생기는 문제 아닌가 싶습니다. 파일 I/O에서 스트림을 쓰면서 메모리가 쌓인다면 보통은 그런 경우가 많으니까요.

그런데 실제 문제는 `FileInputStream`과 `FileOutputStream` 구현 쪽에 있습니다. 이 두 클래스는 `finalize()`를 오버라이드하고 있었고, 이 때문에 제대로 `close()`했더라도 리소스 정리가 기대만큼 깔끔하지 않을 수 있다는 지적이 있었습니다. 자세한 배경은 [이 문서](https://dzone.com/articles/fileinputstream-fileoutputstream-considered-harmful)에 잘 정리돼 있고, Jenkins에서도 [관련 이슈](https://issues.jenkins.io/browse/JENKINS-42934)가 있었으며, OpenJDK에서도 [finalize() 제거 필요성](https://bugs.openjdk.java.net/browse/JDK-8212050)이 논의된 적이 있습니다.

JDK도 이 문제를 인식하고 있어서, `FileInputStream`과 `FileOutputStream`의 `finalize()`는 Java 9부터 `Deprecated`가 되었고 Java 10부터는 다른 구현으로 보완되었습니다. 다만 Java 1.7이나 1.8을 쓰는 경우에는 여전히 문제가 생길 수 있습니다.

그래서 가능하면 새 코드에서는 `FileInputStream`과 `FileOutputStream` 대신 `Files.newInputStream()`과 `Files.newOutputStream()`을 쓰는 편이 낫다고 봅니다. 오래된 습관일수록 무심코 남기기 쉬운 법이니까요.

## 파일 복사 방식 고르기

앞의 결론에 따라 스트림 생성은 NIO 방식으로 통일하겠습니다. 그러면 예제 코드는 다음처럼 바꿀 수 있습니다.

```java
// byte[]를 사용하는 예
private void copy(Path source, Path dest) throws IOException {
    InputStream is = null;
    OutputStream os = null;
    try {
        is = Files.newInputStream(source); // FileInputStream을 쓰지 않음
        os = Files.newOutputStream(dest); // FileOutputStream을 쓰지 않음
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

// BufferedStream을 사용하는 예
private void copy(Path source, Path dest) throws IOException {
    int buff = 8192;
    InputStream is = null;
    OutputStream os = null;
    try {
        is = new BufferedInputStream(Files.newInputStream(source), buff); // FileInputStream을 쓰지 않음
        os = new BufferedOutputStream(Files.newOutputStream(dest), buff); // FileOutputStream을 쓰지 않음
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

### try-with-resource

스트림은 마지막에 반드시 `close()`해야 합니다. 예전에는 이를 위해 `finally` 블록을 직접 작성하는 경우가 많았지만, 이 방식은 코드가 장황해지고 예외 처리도 번거롭습니다. 무엇보다 바쁜 와중에 `close()`를 빠뜨리기 쉽습니다.

그래서 Java 1.7부터는 `AutoCloseable`과 `try-with-resource`가 도입됐습니다. 위 예제도 `try-with-resource`를 쓰면 더 짧고 안전하게 바꿀 수 있습니다.

```java
// byte[]를 사용하는 예
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

// BufferedStream을 사용하는 예
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

이 경우에는 `try-with-resource`를 쓰지 않을 이유가 거의 없습니다. 특별한 사정이 없다면 기본값으로 삼는 편이 좋습니다.

### readAllBytes()

다음으로 볼 것은 버퍼와 `readAllBytes()`입니다. 위 예제에서는 `byte[]` 버퍼를 직접 쓰거나 `BufferedInputStream`과 `BufferedOutputStream`을 사용했는데, 이런 선택은 기본적으로 성능과 메모리 사용량의 균형 문제라고 보면 됩니다.

직관적으로는 버퍼가 클수록 빠를 것 같고, 실제로 어느 정도는 맞는 말입니다. 그렇다고 해서 "그럼 그냥 한 번에 전부 읽으면 되지 않을까?"라는 생각으로 바로 이어지면 조금 위험합니다.

Java 9부터는 `InputStream` 전체를 한 번에 읽어 `byte[]`로 돌려주는 `readAllBytes()`가 생겼습니다. 코드만 보면 꽤 편리하고, 작은 파일이라면 실제로 유용합니다.

문제는 이 메서드가 읽은 데이터를 통째로 메모리에 올린다는 점입니다. 예를 들어 여러 사용자가 동시에 대용량 파일을 업로드하는 웹 애플리케이션에서 `readAllBytes()`를 사용하면, 메모리가 금방 병목이 될 수 있습니다. 파일 하나만 볼 때는 빨라 보여도, 동시에 여러 요청이 들어오면 애플리케이션 전체에는 오히려 더 큰 부담이 됩니다.

그래서 일반적인 파일 복사, 특히 서버 애플리케이션 안에서의 복사라면 `readAllBytes()`는 가능한 한 피하는 편이 안전합니다. 작은 파일을 잠깐 다루는 유틸리티 정도가 아니라면 더욱 그렇습니다.

### transferTo()

Java 9에는 `InputStream`에서 `OutputStream`으로 데이터를 옮기는 `transferTo()`도 추가됐습니다. 이 메서드를 쓰면 파일 복사 코드가 훨씬 간단해집니다.

```java
private void copy(Path source, Path dest) throws IOException {
    try (InputStream is = Files.newInputStream(source);
        OutputStream os = Files.newOutputStream(dest)) {
        is.transferTo(os);
    }
}
```

구현을 보면 `transferTo()` 내부에서도 기본 크기 버퍼를 사용해 복사합니다. 즉, 기본 버퍼 크기인 `8192`바이트 정도를 쓸 생각이었다면 직접 버퍼를 선언하지 않아도 됩니다. 이 점이 꽤 편합니다. 아래는 관련 소스 코드입니다.

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

다만 여기서 한 가지 궁금해집니다. `transferTo()`를 쓸 때도 `BufferedInputStream`이나 `BufferedOutputStream`을 추가로 감싸는 편이 더 빠를까 하는 점입니다. 감으로는 그럴 듯하지만, 직접 재보지 않으면 알 수 없습니다. 그래서 10GB 정도의 파일을 만들어 아래 케이스로 간단한 벤치마크를 돌려 봤습니다.

- InputStream → OutputStream
- BufferedInputStream → OutputStream
- InputStream → BufferedIOutputStream
- BufferedInputStream → BufferedIOutputStream

그리고 코드는 다음과 같습니다.

```java
@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
public class StreamBufferTest {

    private Path source;

    private Path output = Path.of("/Users/retheviper/temp/benchmarkOutput");

    // 테스트용 파일을 만든다
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

그리고 벤치마크 결과는 다음과 같습니다.

```dos
Benchmark                          Mode  Cnt   Score   Error  Units
StreamBufferTest.noBuffer          avgt   25  13.055 ± 0.583   s/op
StreamBufferTest.withInputBuffer   avgt   25  13.302 ± 0.460   s/op
StreamBufferTest.withOutputBuffer  avgt   25  13.663 ± 0.535   s/op
StreamBufferTest.withBothBuffer    avgt   25  12.668 ± 0.355   s/op
```

예상대로 `transferTo()`를 사용하는 단순 파일 복사에서는 `BufferedInputStream`이나 `BufferedOutputStream`을 추가한다고 해서 성능이 크게 달라지지 않았습니다. 물론 더 복잡한 I/O 처리나 네트워크 스트림처럼 조건이 달라지면 이야기가 달라질 수 있습니다. 하지만 로컬 파일을 단순 복사하는 정도라면, `transferTo()`만으로도 충분하다고 봐도 무방해 보입니다.

### `Files.copy()`가 더 나은가

Java 1.7에서는 `Files.copy()`을 통해 다음 파일 복사가 가능합니다.

- InputStream → Path
- Path → OutputStream
- Path → Path

그리고 종종 "`Files.copy()`가 더 빠르다"는 이야기를 보게 됩니다. NIO 쪽이 내부적으로 네이티브 코드와 더 가깝게 붙어 있으니, `InputStream`에서 `OutputStream`으로 복사하는 것보다 `Path` 기반 복사가 유리할 수 있다는 주장입니다. 이게 사실이라면, 로컬 파일을 다룰 때는 `Path` 기반 복사를 우선 고려할 만합니다.

#### 소스 코드를 보면

이 부분은 직접 구현을 확인하는 편이 빠릅니다. 먼저 `InputStream → Path`와 `Path → OutputStream`의 소스 코드를 보면, 결국 내부에서 스트림을 열고 `transferTo()`를 써서 복사합니다. 아래 예시는 Java 11 기준입니다.

```java
// InputStream → Path
public static long copy(InputStream in, Path target, CopyOption... options) throws IOException {
    // 복사 외의 처리는 생략

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

// Path → OutputStream
public static long copy(Path source, OutputStream out) throws IOException {
    // ensure not null before opening file
    Objects.requireNonNull(out);

    try (InputStream in = newInputStream(source)) {
        return in.transferTo(out);
    }
}
```

반면 `Path → Path`는 구현이 다릅니다. 원본과 대상이 같은 파일 시스템에 있으면 [FileSystemProvider](https://docs.oracle.com/javase/7/docs/api/java/nio/file/spi/FileSystemProvider.html)를 사용하고, 그렇지 않으면 `CopyMoveHelper`를 사용합니다.

```java
// Path → Path
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

여기서 `CopyMoveHelper.copyToForeignTarget()`는 결국 `Files.copy(InputStream, Path)` 쪽으로 이어집니다. 결국 눈여겨볼 부분은 `같은 파일 시스템 안에서 Path → Path로 복사할 때`입니다. 이 경우에는 구현이 달라서 성능 차이가 생길 여지가 있습니다.

이 역시 실제로 재보는 편이 낫습니다. 물론 파일 시스템 종류에 따라 결과는 달라질 수 있으니 절대적인 결론으로 받아들이긴 어렵습니다. 다만 적어도 경향은 볼 수 있습니다. 다른 `Files.copy()` 오버로드는 사실상 `transferTo()`와 비슷하게 동작하므로, 여기서는 `InputStream → OutputStream`과 `Path → Path`만 비교했습니다. 그리고 차이를 더 보기 위해 앞선 실험보다 파일 크기를 더 키웠습니다.

```java
@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
public class StreamCopyTest {

    private Path source;

    private Path output = Path.of("/Users/youngbinkim/Downloads/benchmarkOutput");

    // 테스트용 파일을 만든다
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

그리고 벤치마크 결과는 다음과 같습니다.

```dos
Benchmark                      Mode  Cnt   Score   Error  Units
StreamCopyTest.streamToStream  avgt   25  12.128 ± 0.331   s/op
StreamCopyTest.pathToPath      avgt   25  12.257 ± 0.342   s/op
```

10GB 파일에서는 거의 오차 범위 안에 들어왔기 때문에, 파일 크기를 100GB로 키워 같은 벤치마크를 한 번 더 돌렸습니다. 결과는 다음과 같습니다.

```dos
Benchmark                      Mode  Cnt    Score   Error  Units
StreamCopyTest.streamToStream  avgt   25  160.046 ± 2.538   s/op
StreamCopyTest.pathToPath      avgt   25  153.505 ± 2.662   s/op
```

100GB 정도가 되니 그제야 차이가 눈에 들어왔고, 결과는 `Path → Path` 쪽이 조금 더 빠른 편이었습니다. 언젠가 멀티스레드 환경이나 OS 차이까지 포함해 더 넓게 재보면 재미있을 것 같지만, 일단 이번 실험 범위에서는 예상과 크게 다르지 않았습니다.

## 마지막으로

지금까지 본 코드와 벤치마크를 기준으로 정리하면, Java에서 파일을 복사할 때는 우선 아래 정도로 정리할 수 있겠습니다.

- Java 1.7 이상인 경우
  - `FileInputStream`·`FileOutputStream` 대신 `Files.newInputStream`·`Files.newOutputStream` 사용
  - `try-with-resource` 사용
  - 원본과 대상이 같은 파일 시스템 경로라면 `Files.copy(Path, Path)` 우선 검토
- Java 9 이상인 경우
  - 기본 버퍼 크기(`8192`) 정도면 `transferTo()` 사용
  - `transferTo()`를 쓸 때 `BufferedInputStream`·`BufferedOutputStream`은 필수가 아님

대부분의 엔터프라이즈 애플리케이션이 LTS인 Java 8이나 11을 쓴다고 가정하면, 실무에서도 위 기준은 꽤 그대로 적용할 수 있습니다.

이번 글은 "요즘 스타일"의 정답을 정하기보다, 익숙해서 무심코 써 오던 코드를 다시 검토해 보는 데 의미가 있었습니다. 그냥 예전부터 그렇게 써 왔다는 이유만으로 남겨 둔 코드도, 한 번쯤 구현과 근거를 직접 확인해 보면 생각보다 배울 점이 많습니다.

다음에도 이런 식으로 소스 코드와 간단한 벤치마크를 함께 보면서 정리해 보고 싶습니다.
