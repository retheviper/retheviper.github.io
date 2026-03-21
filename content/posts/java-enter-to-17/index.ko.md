---
title: "Java 17에서 무엇이 바뀌었나"
date: 2021-09-25
categories:
  - java
image: "../../images/java.webp"
tags:
  - java
  - kotlin
translationKey: "posts/java-enter-to-17"
---

이번 달에 새로운 LTS 버전인 Java 17이 나왔습니다. 아직도 Java 8을 쓰는 프로젝트가 적지 않지만, Java 8 지원은 2022년, Java 11 지원은 2023년까지였기 때문에 결국은 Java 17로 넘어갈 준비를 해야 하는 시점이라고 볼 수 있습니다. 특히 Java 9부터 모듈 시스템이 들어오면서 8에서 바로 올라가는 일은 꽤 부담스러웠지만, 11에서 17로 가는 일은 상대적으로 덜 까다롭다고 알려져 있습니다. 그래서 지금이라도 Java 17에서 무엇이 달라졌는지 한 번 정리해 둘 가치는 충분합니다.

이 시점에는 [Eclipse Temurin](https://adoptium.net)(구 AdoptOpenJDK), [Zulu](https://www.azul.com/downloads/) 같은 주요 JDK 배포판도 대부분 Java 17 지원을 마쳤거나 진행 중이었습니다. 게다가 [Oracle JDK 17도 무료로 제공](https://www.itmedia.co.jp/news/articles/2109/15/news147.html)되기 시작했으니, 선택지 자체도 더 넓어졌습니다.

여기에 Google과 Oracle의 소송까지 Google 쪽 승리로 마무리되면서, 장기적으로는 Android 쪽에서도 Java 17을 다룰 가능성이 커졌습니다. 당장 바로 바뀌는 일은 아니겠지만, Spring 진영만 봐도 2022년 [Spring 6의 기준선이 Java 17](https://spring.io/blog/2021/09/02/a-java-17-and-jakarta-ee-9-baseline-for-spring-framework)이 되는 만큼, 기존에 Java 11을 쓰던 현장도 지원 주기를 고려해 17로 옮겨 갈 가능성이 높아 보입니다.

그래서 이번 글에서는 Java 17에서 달라진 점을 크게 두 갈래로 나눠 보겠습니다. 하나는 새로운 문법과 작성 방식처럼 언어 사양 차원에서 바뀐 부분이고, 다른 하나는 새로 추가된 API입니다. 프로젝트에 따라서는 Java 8에서 곧바로 17로 올라가는 경우도 있겠지만, 9~11 구간의 변화는 이미 따로 다룬 적이 있으니 여기서는 11~17 사이의 변화에 집중하겠습니다.

## 언어 사양

### macOS 렌더링 파이프라인 개선 (17)

macOS에서는 오랫동안 Swing 같은 Java 2D 렌더링에 [OpenGL](https://www.opengl.org/)을 사용해 왔습니다. 하지만 Apple이 새로운 [Metal framework](https://developer.apple.com/metal/)를 도입하면서, macOS 10.14부터 OpenGL은 사실상 더 이상 권장되지 않는 API가 되었습니다.

그래서 Java 쪽에서도 Metal 기반 렌더링을 위한 [Project Lanai](https://openjdk.java.net/projects/lanai/)가 진행되었고, Java 17부터는 [New macOS Rendering Pipeline](https://openjdk.java.net/jeps/382) 형태로 반영됐습니다. Java에서 GUI를 많이 쓰지 않는다고 생각하기 쉽지만, IntelliJ 같은 Java 기반 IDE도 이 영향을 받습니다. 다만 IntelliJ는 기본적으로 [JetBrains Runtime](https://confluence.jetbrains.com/display/JBR/JetBrains+Runtime)을 사용하므로, 실제 체감은 해당 런타임의 대응 상황도 함께 봐야 합니다.

### Apple Silicon 대응 (17)

Java 17부터는 M1 같은 [Apple Silicon 기반 Mac을 정식 지원](https://openjdk.java.net/jeps/391)합니다. 물론 [Zulu](https://www.azul.com/downloads/)처럼 그보다 먼저 자체 대응한 JDK도 있었지만, OpenJDK 기준 지원이 들어가면서 [Eclipse Temurin](https://adoptium.net/)이나 [Microsoft Build of OpenJDK](https://www.microsoft.com/openjdk) 같은 파생 배포판도 ARM Mac에서 훨씬 자연스럽게 쓸 수 있게 됐습니다.

### Record (17)

Java 14에서 Preview로 들어왔던 `Record`는 17에서 정식 기능이 됐습니다. 필드를 기반으로 생성자, 접근자, `toString()`, `hashCode()`, `equals()` 같은 코드를 자동으로 만들어 주는 기능입니다. 처음에는 Lombok의 [@Data](https://projectlombok.org/features/Data)에 가깝다고 생각했는데, 실제 성격은 [@Value](https://projectlombok.org/features/Value) 쪽에 더 가깝습니다. 값은 생성 시점에만 넣고 이후에는 바꾸지 않는 구조이기 때문입니다. 이런 점에서는 Kotlin의 `data class`와도 꽤 비슷하게 느껴집니다.

```java
// Record 정의
record MyRecord(String name, int number) {}

// 인스턴스 생성
MyRecord myRecord = new MyRecord("my record", 1);

// 필드 가져오기
String myRecordsName = myRecord.name();
int myRecordsNumber = myRecord.number();
```

다만 Kotlin에는 [Named Arguments](https://kotlinlang.org/docs/functions.html#named-arguments)가 있지만 Java에는 아직 비슷한 기능이 없습니다. 그래서 `Record` 필드가 너무 많아지면 생성자 인자만 보고는 무엇이 무엇인지 헷갈릴 수 있습니다. 이런 경우에는 래퍼 클래스를 두거나, 상황에 따라서는 `setter`가 있는 DTO나 빌더 패턴을 쓰는 편이 더 읽기 좋을 수도 있습니다.

또 `Record`는 접근자 이름이 일반적인 `getXxx()`가 아니라 필드명 그대로라는 점도 특징입니다. 자동 생성 생성자에 검증 로직을 넣는 방식도 일반 클래스와는 조금 다릅니다.

```java
record MyRecord(String name, int number) {
    // 생성자에 검증을 붙이는 예
    public MyRecord {
        if (name.isBlank()) {
            throw new IllegalArgumentException();
        }
    }
}
```

그 외에 `Record`로 정의해도 실제로는 `Class`이 만들어지므로 다음과 같은 것도 가능합니다.

- 생성자 추가
- 접근자 메서드 오버라이드
- 내부 클래스로 `Record` 정의
- 인터페이스 구현

또, `Reflection`에서도 클래스가 `Record`일지 어떨지를 판정하는 [isRecord](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Class.html#isRecord())도 추가되고 있습니다.

### 텍스트 블록 (15)

Java에서는 오랫동안 HTML이나 JSON, SQL 등을 리터럴로 사용하기 위해서는 이스케이프와 문자열 결합 등을 사용해야했습니다. 이것은 너무 가독성이라는 면에서 좋지 않아, 코드의 수정도 어렵게 되는 문제가 있었습니다. 예를 들어, HTML을 표현한다면 다음과 같은 일을 하고 있었을까 생각합니다.

```java
String html = "<html>\n" +
              "    <body>\n" +
              "        <h1>This is Java's new Text block!</h1>\n" +
              "    </body>\n" +
              "</html>\n";

String query = "SELECT \"EMP_ID\", \"LAST_NAME\" FROM \"EMPLOYEE_TB\"\n" +
               "WHERE \"CITY\" = 'INDIANAPOLIS'\n" +
               "ORDER BY \"EMP_ID\", \"LAST_NAME\";\n";
```

다행히 15부터 [Text Blocks](https://openjdk.java.net/jeps/378)가 도입되어 다른 언어와 같이 간단하고 가독성이 높은 문자열을 정의할 수 있게 되었습니다. 이것을 사용하면 이스케이프를 의식하지 않아도 되므로 여러 줄이 아니어도 다양한 분야에서 유효 활용할 수 있을 것 같네요. `Text Blocks`을 사용하여 위의 코드를 변경하면 다음과 같습니다.

```java
String html = """
              <html>
                  <body>
                      <h1>This is Java's new Text block!</h1>
                  </body>
              </html>
              """;

String query = """
               SELECT "EMP_ID", "LAST_NAME" FROM "EMPLOYEE_TB"
               WHERE "CITY" = 'INDIANAPOLIS'
               ORDER BY "EMP_ID", "LAST_NAME";
               """;
```

Kotlin에서는 똑같은 글쓰기로 똑같이 할 수 있기 때문에 여기에서는 할애합니다.

### Sealed Class (17)

JDK 15에서 Preview로 들어왔던 [sealed classes](https://openjdk.java.net/jeps/409)도 17에서 정식 기능이 됐습니다. `class`나 `interface`를 `sealed`로 선언하면, 이를 상속하거나 구현할 수 있는 타입을 제한할 수 있습니다. 라이브러리처럼 외부에서 임의로 확장되길 원하지 않는 계층을 만들 때 특히 유용합니다. 또 이런 구조는 앞으로 `switch`와도 더 잘 연결될 여지가 있습니다. 예를 들어 허용된 하위 타입을 컴파일러가 더 엄격하게 검사할 수 있게 되는 식입니다. 아래는 `permits` 키워드로 허용 타입을 지정하는 예입니다.

```java
public abstract sealed class Shape permits Circle, Rectangle, Square, WeirdShape { }
```

Kotlin에도 [Sealed Classes](https://kotlinlang.org/docs/sealed-classes.html)가 있습니다. 다만 `sealed interface`는 Kotlin 1.5 이후에 가능하고, Java처럼 `permits`로 허용 대상을 일일이 나열하는 방식은 아닙니다. 대신 같은 모듈이나 패키지 제약을 통해 계층을 제한합니다. 그래서 체감상 Kotlin 쪽 문법이 조금 더 간단합니다.

```kotlin
sealed interface Error

sealed class IOError(): Error

class FileReadError(val f: File): IOError()
class DatabaseError(val source: DataSource): IOError()

object RuntimeError : Error
```

Java에서는 `Record`와 마찬가지로, 해당 클래스가 `sealed`인지 확인할 수 있는 [isSealed](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Class.html#isSealed())도 함께 추가됐습니다.

### Switch Expressions (14)

Java 12부터 Preview로 들어온 [Switch Expressions](https://openjdk.java.net/jeps/361)는 14에서 정식 기능이 됐습니다. 기존 `switch`를 개선해 다음과 같은 점이 달라졌습니다.

- 여러 `case`를 한 줄에 함께 묶을 수 있습니다.
- `case` 처리를 화살표 문법으로 더 간단하게 쓸 수 있습니다.
- `switch` 자체를 표현식으로 써서 값을 바로 반환할 수 있습니다.

예를 들어 `day`라는 enum 값을 보고 `int` 값을 돌려주는 메서드를 만든다고 해 봅시다. 기존 방식은 아래와 같습니다.

```java
int numLetters;
switch (day) {
    case MONDAY:
    case FRIDAY:
    case SUNDAY:
        numLetters = 6;
        break;
    case TUESDAY:
        numLetters = 7;
        break;
    case THURSDAY:
    case SATURDAY:
        numLetters = 8;
        break;
    case WEDNESDAY:
        numLetters = 9;
        break;
    default:
        throw new IllegalStateException("Wat: " + day);
}
```

위의 처리는 새로운 `switch`에서는 다음과 같이 쓸 수 있습니다.

```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY                -> 7;
    case THURSDAY, SATURDAY     -> 8;
    case WEDNESDAY              -> 9;
};
```

Kotlin이라면 다음과 같이 될 것입니다.

```kotlin
val numLetters = when (day) {
    Day.MONDAY, Day.FRIDAY, Day.SUNDAY -> 6
    Day.TUESDAY -> 7
    Day.THURSDAY, Day.SATURDAY -> 8
    Day.WEDNESDAY -> 9
}
```

또 Kotlin의 `when`은 인자 없이 조건문처럼도 쓸 수 있어서 분기 표현이 더 유연합니다. 개인적으로는 아직도 사용감 자체는 Java의 `switch`보다 낫다고 느낍니다. 다만 Java도 버전이 올라가면서 뒤에서 볼 기능들이 계속 추가되고 있으니, 앞으로는 격차가 더 줄어들 수 있겠습니다.

### Pattern Matching for instanceof (16) / switch (17)

Java 14부터는 [Pattern Matching for instanceof](https://openjdk.java.net/jeps/394)가 도입됐고, 16에서 정식 기능이 됐습니다. 이전에는 `instanceof`로 타입을 확인한 뒤 다시 직접 캐스팅해야 했습니다.

```java
static String formatter(Object o) {
    String formatted = "unknown";
    if (o instanceof Integer) {
        formatted = String.format("int %d", (Integer) i);
    }
    // ...
}
```

이 방식은 중복이 많고, 실수하면 예외 원인이 되기도 합니다. 그래서 `Pattern Matching`이 들어오면서 캐스팅을 조건식 안으로 끌어올릴 수 있게 됐습니다. `instanceof` 뒤에 변수명을 함께 선언해 두면, 해당 블록 안에서는 이미 캐스팅된 값처럼 바로 사용할 수 있습니다.

```java
static String formatter(Object o) {
    String formatted = "unknown";
    if (o instanceof Integer i) {
        formatted = String.format("int %d", i);
    } else if (o instanceof Long l) {
        formatted = String.format("long %d", l);
    } else if (o instanceof Double d) {
        formatted = String.format("double %f", d);
    } else if (o instanceof String s) {
        formatted = String.format("String %s", s);
    }
    return formatted;
}
```

게다가 17부터는 Preview 기능으로 [Pattern Matching for switch](https://openjdk.java.net/jeps/406)도 들어왔습니다. 이 기능을 쓰면 `instanceof` 연쇄 없이도 `switch`로 더 간단하게 표현할 수 있습니다. 앞서 본 `Switch Expressions`와 조합하면 위 코드를 아래처럼 줄일 수 있습니다.

```java
static String formatterPatternSwitch(Object o) {
    return switch (o) {
        case Integer i -> String.format("int %d", i);
        case Long l    -> String.format("long %d", l);
        case Double d  -> String.format("double %f", d);
        case String s  -> String.format("String %s", s);
        default        -> o.toString();
    };
}
```

### Packaging Tool (16)

[Packaging Tool](https://openjdk.java.net/jeps/392)은 실행 가능한 패키지를 만드는 기능입니다. Java runtime, 라이브러리, 그리고 각 OS에 맞는 실행 파일을 하나의 패키지로 묶어 줍니다. 즉 Java runtime까지 함께 포함할 수 있으므로, OS에 설치된 Java 버전과 무관하게 실행하고 싶을 때 유용합니다. 여러 앱이 서로 다른 Java 버전을 써야 하는 경우에도 꽤 편리합니다.

## API

Java 17부터는 API 문서에서 새로 추가된 API만 따로 볼 수 있는 탭이 생겼습니다. 지금은 11 이후에 추가된 항목만 정리돼 있지만, 앞으로 새로운 LTS 버전이 나오면 그 이후 변경 사항도 같은 방식으로 확인할 수 있을 것 같습니다. 새로운 API 목록은 [여기](https://download.java.net/java/early_access/jdk17/docs/api/new-list.html)에서 볼 수 있습니다.

여기에서 모든 API의 상세까지 찾는 것은 어렵다고 생각하므로, 개인적으로 흥미롭다고 생각한 것을 일부 소개하고 싶습니다.

### @Serial (14)

`java.io` 패키지에는 [Serial](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Serial.html) 어노테이션이 추가됐습니다. [Serializable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Serializable.html)을 구현한 클래스에서, 직렬화 관련 멤버를 명시적으로 표시하는 데 쓰입니다. 느낌상 직렬화 버전의 `@Override` 비슷하게 이해해도 크게 틀리지는 않습니다. 예를 들면 아래와 같습니다.

```java
class SerializableClass implements Serializable {

    @Serial
    private static final ObjectStreamField[] serialPersistentFields;

    @Serial
    private static final long serialVersionUID;

    @Serial
    private void writeObject(ObjectOutputStream stream) throws IOException {}

    @Serial
    private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException {}

    @Serial
    private void readObjectNoData() throws ObjectStreamException {}
    
    @Serial
    Object writeReplace() throws ObjectStreamException {}

    @Serial
    Object readResolve() throws ObjectStreamException {}
}
```

이 어노테이션을 붙이면 컴파일 타임에 오류를 잡아낼 수 있다는 점도 특징입니다. 예를 들어 아래와 같은 클래스 멤버에 사용하면 컴파일 에러가 납니다.

- `Serializable`을 구현하지 않는 클래스
- enum처럼 일반 직렬화 흐름이 적용되지 않는 클래스
- [Externalizable](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/Externalizable.html)을 구현하는 클래스

이런 어노테이션이 들어오면 Jackson이나 Gson 같은 라이브러리 쪽 구현에도 간접적인 영향이 생길 수 있겠다는 생각이 듭니다.

### 문자열

같은 문자열을 다루더라도 Java와 Kotlin은 표면적인 사용감이 조금 다르기 때문에, 여기서는 새로 추가된 Java API와 Kotlin에서 비슷한 처리를 할 때 떠올릴 수 있는 방법을 함께 보겠습니다.

#### formatted (15)

Java에서는 예전부터 `String.format()`으로 문자열 포맷을 만들 수 있었습니다. 경우에 따라 단순 `+` 연결보다 읽기 좋고 관리하기 쉬워서 자주 쓰이던 방식입니다.

```java
String name = "formatted string";

// 15 이전
String formattedString = String.format("this is %s", name);

// 15 이후
String newFormattedString = "this is %s".formatted(name);
```

Kotlin이라면 [String.format](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/format.html)과 [String Templates](https://kotlinlang.org/docs/basic-syntax.html#string-templates)를 떠올릴 수 있습니다.

```kotlin
val name = "formatted string"

// Format
val formattedString = "this is %s".format(name)

// String Template
val templateString = "this is $name"
```

#### indent (12)

[indent](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html#indent(int))는 문자열 앞에 지정한 만큼 공백을 더하거나, 음수를 넘겨 공백을 줄이는 메서드입니다.

```java
String nonIndent = "A";
// 들여쓰기 추가
String indented10 = nonIndent.indent(10); // "          A"
// 들여쓰기 제거
String indented5 = indented10.indent(-5); // "     A"
```

Kotlin에서는 들여쓰기를 추가하는 [prependIndent](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/prepend-indent.html)나, 들여쓰기를 바꾸는 [replaceIndent](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/replace-indent.html)를 사용할 수 있습니다. 이쪽은 넘기는 인자가 문자열이라 Java의 `indent()`와는 사용감이 조금 다릅니다.

```kotlin
val nonIndent = "A"
// 들여쓰기 추가
val prepended = nonIndent.prependIndent("     ") // "     A"
// 들여쓰기를 대체(없으면 추가)
val replaced = prepended.replaceIndent("|||||") // "|||||A"
```

#### stripIndent (15)

`Text Block`처럼 여러 줄 문자열을 다룰 때는, 코드 가독성을 위해 넣어 둔 들여쓰기가 실제 데이터에는 방해가 될 수 있습니다. 이럴 때 들여쓰기를 정리하는 메서드가 [stripIndent](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html#stripIndent())입니다.

Kotlin에서는 [trimIndent](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/trim-indent.html)가 같은 역할을 하고 있습니다.

#### 변환 (12)

문자열에 [Function](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/function/Function.html)을 적용하는 간단한 API입니다. `replace()`만으로는 표현하기 애매한 변환이 필요할 때 쓸 만합니다. 구현도 꽤 단순합니다.

```java
public <R> R transform(Function<? super String, ? extends R> f) {
    return f.apply(this);
}
```

Kotlin에서는 문자열에도 `map`, `filter`, `reduce` 같은 고차 함수를 적용할 수 있으니 비슷한 흐름으로 생각할 수 있습니다. 아니면 아래처럼 확장 함수를 하나 정의해도 됩니다.

```kotlin
fun <R> String.transform(f: (String) -> R): R = f(this)
```

#### translateEscapes (15)

이스케이프된 일부 문자를 실제 리터럴 문자로 바꿔 주는 기능입니다. 이 부분은 설명보다 예제를 보는 편이 더 빠릅니다.

```java
String string = "this\\nis\\nmutli\\nline";
String escapeTranslated = string.translateEscapes() // "this\nis\nmutli\nline"
```

예전에는 `Matcher`와 정규식을 조합한 직접 구현이나 별도 라이브러리에 기대야 하는 경우가 많았기 때문에, 이런 기능이 기본 API로 들어온 점은 꽤 반갑습니다. 변환되는 이스케이프 문자는 아래와 같습니다.

| Escape | Name | Translation |
|---|---|---|
| `\b` | backspace | U+0008 |
| `\t` | horizontal tab | U+0009 |
| `\n` | line feed | U+000A |
| `\f` | form feed | U+000C |
| `\r` | carriage return | U+000D |
| `\s` | space | U+0020 |
| `\"` | double quote | U+0022 |
| `\'` | single quote | U+0027 |
| `\\` | backslash | U+005C |
| `\0 - \377` | octal escape | code point equivalents |
| `\<line-terminator>` | continuation | discard |

Kotlin에서는 비슷한 API가 없기 때문에 필요하다면 독자적인 처리를 쓰는 것이 좋을 것 같습니다. (라이브러리는 모르고…)

### Map.Entry.copyOf (17)

`Map.Entry`의 복사본을 만듭니다. 이렇게 만든 엔트리는 원래 맵과 더 이상 연결되지 않은 별도 데이터입니다. [공식 문서](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.Entry.html)에는 아래 같은 예가 나옵니다.

```java
var entries = map.entrySet().stream().map(Map.Entry::copyOf).toList();
```

참고로 `Map` 자체의 복사는 Java 10부터 추가된 [copyOf](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.html#copyOf(java.util.Map))로 할 수 있습니다.

```java
var copiedMap = Map.copyOf(map);
```

Kotlin이라면 `Entry`의 복사본은 다음과 같이 할 수 있습니다. 형식은 `List<MutableMap.MutableEntry<K, V>>`입니다.

```kotlin
// Map.Entry를 사용하는 경우
val entriesJava = map.entries.map { Map.Entry.copyOf(it) }

// Kotlin의 Map.Entry를 사용하는 경우
val entriesKotlin = map.entries.toSet()
```

또한 Kotlin에서 `Map`을 복사하는 방법은 다음과 같습니다.

```kotlin
val copiedMap = map.toMap()
```

### 스트림

#### mapMulti (16)

Java 16부터 Stream에는 [mapMulti](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html#mapMulti(java.util.function.BiConsumer))가 추가됐습니다. 기본적으로는 "한 요소를 1:N으로 펼쳐 결과 Stream에 넣는다"는 점에서 [flatMap](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html#flatMap(java.util.function.Function))과 비슷하지만, 아래 같은 경우에는 `flatMap`보다 더 잘 맞는다고 합니다.

- 결과 요소를 일부만 골라 내고 싶을 때
- 각 요소를 `Stream`으로 바꾸는 과정이 번거로운 경우

먼저 중첩된 컬렉션에 `flatMap`을 쓰는 경우를 생각해 보겠습니다. 일부 요소만 남기려면 우선 모든 요소를 펼친 뒤 `filter()`로 조건에 맞는 것만 다시 골라야 합니다. 이 과정에서 각 요소 그룹마다 `Stream` 인스턴스를 만들어야 하고, 중첩 구조를 어떻게 `Stream`으로 펼칠지도 직접 정의해야 합니다.

문제는 `Stream` 인스턴스를 계속 만들어야 해서 오버헤드가 생기고, 요소 모양이 제각각이면 이를 `Stream`으로 바꾸는 코드도 꽤 번거로워진다는 점입니다. 예를 들어 아래 같은 리스트가 있다고 해 봅시다.

```java
List<Object> numbers = List.of(List.of(1, 2L), 3, List.of(4, 5L, 6), List.of(7L), 8L);
```

이 목록에서 `Integer`만 추려 새 리스트를 만든다면 어떻게 될까요? `flatMap`을 쓰면 대략 아래처럼 작성하게 됩니다.

```java
List<Integer> integers = list.stream()
        .flatMap( // 요소를 Stream으로 변환한다
                it -> {
                    if (it instanceof Iterable<?> l) {
                        return StreamSupport.stream(l.spliterator(), false);
                    } else {
                        return Stream.of(it);
                    }
                })
        .filter(it -> it instanceof Integer) // Integer만 남긴다
        .map(it -> (Integer) it) // Object에서 Integer로 캐스팅
        .toList();
```

같은 작업을 `mapMulti`로 처리하면 아래처럼 됩니다. 구조가 훨씬 단순합니다.

```java
class MultiMapper {
    static void expandIterable(Object e, Consumer<Integer> c) {
        if (e instanceof Iterable<?> i) {
            i.forEach(ie -> expandIterable(ie, c));
        } else if (e instanceof Integer i) {
            c.accept(i);
        }
    }
}

List<Integer> integers = list.stream().mapMulti(MultiMapper::expandIterable).toList();
```

또 [mapMultiToInt](https://docs.oracle.com/en/java/javase/17/docs/api/new-list.html#:~:text=java.util.stream.Stream.mapMultiToInt(BiConsumer%3C%3F%20super%20T%2C%20%3F%20super%20IntConsumer%3E)), [mapMultiToLong](https://docs.oracle.com/en/java/javase/17/docs/api/new-list.html#:~:text=java.util.stream.Stream.mapMultiToLong(BiConsumer%3C%3F%20super%20T%2C%20%3F%20super%20LongConsumer%3E)), [mapMultiToDouble](https://docs.oracle.com/en/java/javase/17/docs/api/new-list.html#:~:text=java.util.stream.Stream.mapMultiToDouble(BiConsumer%3C%3F%20super%20T%2C%20%3F%20super%20DoubleConsumer%3E))도 함께 추가됐기 때문에, 숫자를 다룬다면 이쪽이 더 잘 맞습니다. 위 예제를 `mapMultiToInt`로 바꾸면 아래와 같습니다.

```java
class MultiMapper {
    static void expandIterable(Object e, IntConsumer c) {
        if (e instanceof Iterable<?> i) {
            i.forEach(ie -> expandIterable(ie, c));
        } else if (e instanceof Integer i) {
            c.accept(i);
        }
    }
}

List<Integer> integers = list.stream().mapMultiToInt(MultiMapper::expandIterable).boxed().toList();
```

`mapMultiToInt`의 반환값은 [IntStream](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/IntStream.html)이므로, `Stream<Integer>`가 필요하다면 [boxed()](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/IntStream.html#boxed())를 호출해야 합니다. `Consumer`가 `IntConsumer`로 바뀐다는 점도 차이입니다.

Kotlin은 애초에 `flatMap`을 Java Stream과 같은 방식으로 다루지 않으므로, 이 처리는 다른 관점에서 보는 편이 낫습니다. 다행히 Kotlin 컬렉션 API도 꽤 풍부해서 비슷한 문제를 푸는 일 자체는 어렵지 않습니다. 예를 들어 객체 타입을 기준으로 요소를 모으고 싶다면 아래처럼 쓸 수 있습니다.

```kotlin
val list = listOf(listOf("A", 'B'), "C", setOf("D", 'E', "F"), listOf('G'), 'H')

val result: List<String> = list.flatMap {
    if (it is Iterable<*>) {
        it.filterIsInstance<String>()
    } else {
        listOf(it).filterIsInstance<String>()
    }
} // [A, C, D, F]
```

다만 Java에서 `List.of(1, 2L)`를 만들면 `1`은 int, `2L`는 long으로 다루지만, Kotlin에서 `listOf(1, 2L)`를 쓰면 전체가 `List<Long>`로 추론됩니다. 그래서 예제를 옮겨 볼 때는 타입 추론 차이를 주의해야 합니다.

```kotlin
val list = listOf(listOf(1, 2L), 3, setOf(4, 5L, 6), listOf(7L), 8L)

val result = list.flatMap {
    if (it is Iterable<*>) {
        it.filterIsInstance<Int>()
    } else {
        listOf(it).filterIsInstance<Int>()
    }
} // [3]
```

#### toList(16)

자주 쓰는 종단 연산인 "List로 모은다"를 더 짧게 쓸 수 있게 만든 메서드라고 보면 됩니다. 처리 결과로 생성되는 List는 `Unmodifiable`입니다.

```java
List<String> list = List.of("a", "B", "c", "D");

// 구 방식
List<String> upper = list.stream().map(String::toUpperCase).collect(Collectors.toUnmodifiableList());

// 새 방식
List<String> lower = list.stream().map(String::toLowerCase).toList();
```

Kotlin에서는 보통 Collection에 고차 함수를 적용한 결과를 그대로 다루게 되지만, 필요하면 `stream`으로 바꿔 쓸 수도 있습니다. 상황에 따라서는 이런 선택지도 나쁘지 않아 보입니다.

### Collectors.teeing (12)

Collectors에 두 개의 `Collector`을 결합하는 [teeing] lectors.html#teeing(java.util.stream.Collector,java.util.stream.Collector,java.util.function.BiFunction))이라는 메서드가 추가되었습니다. 덧붙여서 `Tee`은 2개의 수도관을 접속해 하나로 해 주는 「T자 피팅」의 의미를 가지는 것 같습니다. 인수에 2개의 `Collector`과 그것을 결합하는 처리의 `BiFunction`을 지정하는 형태로 되어 있습니다.

예를 들면 다음과 같은 `Stream`이 있다고 가정해 봅시다.

```java
record Member(String name, boolean enabled) {}

/**
* [
*    Member[name=Member1, enabled=false],
*    Member[name=Member2, enabled=true],
*    Member[name=Member3, enabled=false],
*    Member[name=Member4, enabled=true],
* ]
*/
Stream<Member> members = IntStream.rangeClosed(1, 4).mapToObj(it -> new Member("Member" + it, it % 2 == 0));
```

이것을 `teeing`을 사용하여 `Member`의 `enabled`을 기준으로 두 개의 List로 나누면 다음과 같이됩니다.

```java
/**
* [
*    [
*       Member[name=Member2, enabled=true],
*       Member[name=Member4, enabled=true]
*    ],
*
*    [
*       Member[name=Member1, enabled=false],
*       Member[name=Member3, enabled=false]
*    ]
* ]
*/
List<List<Member>> result = members.collect(
        Collectors.teeing(
                Collectors.filtering(
                        Member::enabled,
                        Collectors.toList()
                ),
                Collectors.filtering(
                        Predicate.not(Member::enabled),
                        Collectors.toList()
                ),
                (list1, list2) -> List.of(list1, list2)
        )
);
```

Kotlin에서는 원래 `collect`가 필요 없기 때문에, 보통은 `Collection`의 고차 함수를 그대로 사용하는 쪽이 더 자연스럽습니다. Java에서도 그런 편이 더 읽기 쉬운 경우가 많습니다.

## 마지막으로

모든 변경 사항을 빠짐없이 정리하기는 어려워서 눈에 띄는 변화만 골라 소개했지만, 그래도 분량이 꽤 많았습니다. 그만큼 Java 17은 Java 11보다 더 현대적인 언어에 가까워진 버전이라고 볼 수 있고, Java를 쓰고 있는 프로젝트라면 충분히 도입을 검토할 만합니다. 또 Java 15 이후에는 G1GC 개선으로 [성능 향상도 있었다](https://www.optaplanner.org/blog/2021/01/26/HowMuchFasterIsJava15.html)고 알려져 있어서, 성능 측면에서도 기대해 볼 만합니다.

Kotlin을 주로 사용하더라도 JVM 위에서 동작하는 이상 이런 개선의 이점을 함께 누릴 수 있습니다. 그래서 API 변화뿐 아니라 런타임 개선까지 같이 본다면 Java 17 도입 가치는 더 분명해 보입니다. 다음 LTS에서는 또 어떤 변화가 나올지도 계속 지켜볼 만합니다.
