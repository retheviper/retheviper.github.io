---
title: "Kotlin에서 ZIP 압축하기"
date: 2021-04-14
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-zip-files"
---

서버 기능을 만들다 보면 파일 다운로드를 구현해야 할 때가 있습니다. 단순히 스토리지에 있는 파일 하나를 그대로 내려주는 것만이 아니라, 경우에 따라서는 파일을 생성해 응답에 담거나 여러 파일을 한 번에 전달해야 할 수도 있습니다.
파일 하나만 내려보내는 건 비교적 단순하지만, 여러 파일을 함께 보내려면 이야기가 조금 복잡해집니다. 이럴 때는 보통 ZIP으로 묶어 보내는 쪽이 자연스럽습니다. Java에는 기본적으로 [ZipOutputStream](https://docs.oracle.com/javase/ko/8/docs/api/java/util/zip/ZipOutputStream.html)이 있으니, 파일을 엔트리로 추가하고 출력만 하면 기본 구조는 완성됩니다.
문제는 실제 요구사항이 그보다 조금 더 복잡하다는 점입니다. 디렉터리가 중첩돼 있을 수도 있고, 원래 구조를 유지한 채 압축해야 할 수도 있으며, 특정 파일은 제외해야 할 수도 있습니다. 가능하면 파일 하나든 디렉터리든 같은 메소드로 처리하고 싶기도 합니다. 그래서 이번 글에서는 이런 요구를 만족하는 간단한 ZIP 압축 코드를 Java에서 Kotlin으로 옮기면서 정리해 보겠습니다.
이번에 소개하는 코드는 Baeldung의 [Java로 Zip을 압축하는 방법](https://www.baeldung.com/java-compress-and-uncompress)에 관한 기사에 소개되어 있는 것을 베이스로 하고 있습니다.
## 자바

먼저 Java 코드를 살펴 보겠습니다. 위의 기사에는 다음과 같은 코드가 소개되어 있습니다.
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

`zipFile` 메소드를 보면, 인수의 `fileToZip`에 ZIP로 압축하고 싶은 파일이나 디렉토리의 패스를 지정해, `fileName`에는 파일 혹은 디렉토리명, `zipOut`에는 압축 후의 ZIP의 파일명을 지정하게 되어 있습니다.
그리고 구현으로서는, 지정한 파일이나 디렉토리에 `hidden` 속성이 있는 경우는 압축하지 않고, 압축원의 파일이 디렉토리인 경우는 안의 파일을 전부 ZIP에 포함시키는 처리가 포함되어 있군요. 대상의 파일과 디렉토리를 모두 엔트리에 추가한 후에는, 압축원을 읽어 ZipOutputStream에 기입하는 처리가 되어 있습니다. 이것을 Kotlin의 코드로 바꾸자.
## Kotlin 코드로 바꾸기

Java 코드를 Kotlin 코드로 바꾸는 것은 그리 어렵지 않습니다. IntelliJ에는 Java 코드를 붙여 넣으면 자동으로 Kotlin 코드로 변환해 주는 기능도 있습니다. 다만 그것만으로 충분하지는 않습니다. 쉽게 변환할 수 있다고 해도, 그것이 정말로 Kotlin다운 코드라고는 말할 수 없는 경우가 있습니다.
그리고, 처리 자체도 보다 단순하게, 혹은 읽기 쉬운 코드로 하는 방법도 있을 것입니다. 위의 Java 코드를 먼저 Kotlin으로 바꾸고, 여러가지 개선하고 싶은 곳을 포함해 바꾸어 갑니다.
### Kotlin 같은 코드로 바꾸기

Intellij 2021.1을 기준으로 Java 코드를 그대로 붙여 넣으면 다음과 같은 코드로 자동 변환됩니다.
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

여기서 더 Kotlin 같은 코드로 바꾸고 싶은 부분은 `InputStream`이나 `OutputStream`의 사용법입니다. Java에서도 `try-with-resource`가 있고, Kotlin에는 `use()`이 있으므로 그쪽을 사용하는 것이 `close`보다 좋은 생각이 듭니다.
또한 `if`은 `when`로 바꾸거나 `for`을 `forEach()`로 바꾸는 등의 작업을 더 명확하게 할 수있는 것처럼 보입니다. 개인적으로는 스코프를 굳이 나누는 쪽이 책임이 명확해져, 처리를 쫓을 때에 혼란하지 않기 때문에 가능한 한 스코프 함수나 Collection 전용의 오퍼레이션을 적극적으로 사용해 처리의 단위를 나눌 수 있는 곳은 제대로 나누고 싶습니다. Java의 방법을 매우 취급으로는 전혀 문제가 없습니다만, 모처럼이므로 Kotlin만이 가능한 코드를 그리고 싶은 것입니다.
굳이 IOException을 던진다는 표시를 해 둔다는 것도, 런타임시의 예외의 처리를 강제하고 있지 않는 Kotlin에는 적합하지 않은 것이 아닌가 하는 생각도 하기 때문에, 어노테이션은 삭제하는 것으로 합니다.
### IO에서 NIO로 변경

NIO는 이전 글들에서도 몇 차례 언급했지만, 서버처럼 호출이 잦고 동시 실행 수가 많은 환경에서는 적극적으로 고려할 만합니다. 또 Java 1.8 이후 추가된 메서드 중에는 꽤 편리한 것들이 많아서, IO를 NIO로 바꾸는 것만으로도 코드 양을 많이 줄일 수 있습니다.
특히 디렉터리를 대상으로 할 때는, 자식 요소를 순회하는 데 NIO의 `Files`가 제공하는 기능이 꽤 강력합니다. 그래서 이번에는 그 기능을 적극 활용하기로 했습니다.
### 서명 변경

위 메소드는 인수를 세 개 받지만, 실제로 꼭 필요한 것은 압축 대상 경로와 결과 ZIP 경로뿐입니다. `ZipOutputStream`을 호출부에서 넘기게 할 이유는 크지 않고, 오히려 사용할 때마다 준비해야 해서 번거롭습니다. 또 엔트리 추가는 메소드 안에서 하면서 스트림 생성과 종료는 바깥에 맡기는 구조도 역할이 어색합니다. 그래서 `ZipOutputStream`의 생성과 사용은 메소드 안에서 모두 끝나도록 바꾸기로 했습니다.
이렇게 하면 호출부에서의 사용법도 더 단순해지고, 압축 원본을 읽는 `InputStream`은 내부에서 닫으면서 인수로 받은 `OutputStream`은 외부에서 닫아야 하는 식의 어색한 구조도 피할 수 있습니다.
### 재귀를 없애기

압축원의 패스가 디렉토리인 경우는, 한층 더 중첩된 디렉토리나 파일도 정리해 압축하기 위해서 재귀를 사용하게 되어 있습니다. 재귀는 알고리즘으로서는 중요하지만, 처리가 전부 끝날 때까지 메모리에 전 데이터와 처리를 채워 두기 때문에 처리의 효율이라고 하는 면에서는 그다지 좋지 않은 경우도 있네요. 하고 싶은 것은 단순히 `hidden`속성을 가지는 파일이나 디렉토리를 제외하는 것, 그리고 그 이외의 파일이나 디렉토리는 모두 ZipOutputStream의 엔트리에 넣고 싶다고 하는 단순한 일입니다.
다행히 NIO를 사용하여 디렉토리의 자식 요소를 전부 취득할 수 있고, 취득한 자식 요소는 `Stream<Path>`로서 취득할 수 있으므로, `filter()`나 `forEach()`와 같은 메소드를 사용할 수 있습니다. 이것으로 충분히, 재귀를 사용하지 않고 목적을 달성할 수 있을 것 같네요.
## 완성된 코드

이상을 반영하여 수정한 코드는 다음과 같습니다.
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

간단히 설명하면, `object`로 선언해 어디서든 쓸 수 있는 유틸리티 형태로 만들었고, 메소드 시그니처도 더 단순하게 정리했습니다. `source`에는 압축할 파일이나 디렉터리를, `target`에는 결과 ZIP 파일 경로를 전달합니다. `ZipOutputStream`은 메소드 안에서 만들고 `use()`로 자동 종료되도록 했습니다.
핵심은 `Files.walk()`로 하위 요소를 모두 가져오고, `filter()`로 hidden 파일을 제외한 뒤 순회하는 부분입니다. 디렉터리라면 `/`를 붙인 `ZipEntry`를 추가하고, 파일이라면 엔트리를 만든 뒤 내용을 그대로 복사합니다. 결과적으로 원래 Java 코드보다 훨씬 짧고 읽기 쉬운 형태가 됩니다.
## 마지막으로

Kotlin다운 코드라고 말했지만 위의 코드는 어디까지나 `Kotlin/JVM`에서만 유효합니다. `Kotlin/Native`, `Kotlin/JS` 등에서 사용하려면 다른 방법을 찾아야 합니다. 또한 `Files.walk()`는 Java 1.8에서 추가된 메서드이므로, 1.7 이하라면 `Files.walkFileTree()`나 그 이전 방식의 NIO 이외의 방법을 사용해야 합니다.
적어도 `Kotlin/JVM`(Java 1.8 이상) 환경에서는 꽤 실용적인 방법이라고 생각합니다. 물론 더 다듬을 여지는 있겠지만, Java API를 Kotlin 스타일로 다시 정리해 보는 작업 자체도 충분히 의미가 있습니다.
