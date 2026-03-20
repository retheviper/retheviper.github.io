---
title: "문자열 패턴 검증하기"
date: 2020-11-22
categories:
  - java
image: "../../images/java.webp"
tags:
  - java
  - pattern
  - regular expression
translationKey: "posts/java-string-pattern-validator"
---

보통 애플리케이션은 업무 요구사항이나 보안 관점에서 고려할 점이 많습니다. 그래서 기능을 만들 때는 단순히 "동작한다"는 것만으로는 부족하고, 어떤 조건에서만 허용할지 같은 제한이 필요합니다.

이번 글도 그런 업무 요구에서 나온 이야기입니다. 제가 관련된 이슈에서는 EC2에서 실행되는 Spring Boot 기반 앱을 만들고 있었고, 파일 데이터와 업로드 경로를 받아 S3에 올리는 기능을 담당하고 있었습니다.

단순히 업로드 경로와 데이터가 있으면 되는 기능은 어렵지 않습니다. Spring에는 [ResourceLoader](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/ResourceLoader.html)도 있고, [AWS SDK](https://aws.amazon.com/ko/sdk-for-java)로도 쉽게 구현할 수 있습니다. 사실 이 기능은 업로드 경로와 파일만 있으면 되는 수준이라 구현이라고 할 것도 없습니다.

하지만 이 기능이 호출될 때 전달된 업로드 경로가 "올바른 경로"인지 확인해야 했습니다. 업무상 S3에 저장할 때 정해진 경로 규칙이 있었고, 전달받은 경로가 규정된 패턴과 일치하는지 먼저 검사해야 했습니다.

그럼 이런 검증 기능은 무엇으로 만드는 게 좋을까요? 여러 방법이 있겠지만, 여기서는 제가 직접 구현했던 방식을 기준으로 정리해 보겠습니다.

## 문자열 패턴은 정규식으로

파일의 업로드 경로는 문자열이어야 하고, 특정 패턴을 따라야 합니다. 문자열이 패턴과 맞는지는 [정규 표현식](https://ko.wikipedia.org/wiki/%E6%AD%A3%E8%A6%8F%E8%A1%A8%E7%8F%BE)으로 확인합니다. 즉, 미리 정규식을 정의해 두고 전달받은 값이 그것과 일치하는지 검사하면 됩니다.

Java에서는 문자열 패턴을 확인하는 방법이 몇 가지 있습니다.

1. `Pattern.matches()` 사용
1. `Pattern`에서 `Matcher`를 얻어 사용
1. `Pattern`에서 `Predicate`를 얻어 사용
1. `String.matches()` 사용

예시는 아래와 같습니다.

```java
// 정규식 예시
String patternRegex = "^[0-9]*$";
// 검사할 문자열
String value = "123456789";

// Pattern.matches()
boolean patternMatches = Pattern.matches(patternRegex, value);

// Matcher 사용
Pattern pattern = Pattern.compile(patternRegex);
Matcher matcher = pattern.matcher(value);
boolean matcherFind = matcher.find(); // 부분 일치
boolean matcherMatches = matcher.matches(); // 완전 일치

// Predicate 사용
Pattern pattern = Pattern.compile(patternRegex);
boolean matcherFind = pattern.asPredicate().test(value); // 부분 일치
boolean matcherMatches = pattern.asMatchPredicate().test(value); // 완전 일치

// String.matches()
boolean stringMatches = value.matches(patternRegex);
```

부분 일치가 필요하다면 `Matcher`나 `Predicate`를 쓰면 됩니다. 그런데 완전 일치가 필요하다면 무엇이 더 나을까요? 같은 결과라면 성능이 좋은 쪽을 고르고 싶습니다.

## 같다면 성능으로

문자열이 정규식과 일치하는지 판단하는 여러 방법 중 무엇이 가장 빠른지 측정하고 싶었습니다. 단순한 스톱워치 방식도 가능하지만, 좀 더 정확하게 비교하려고 OpenJDK의 [JMH](https://openjdk.java.net/projects/code-tools/jmh)를 사용했습니다. 워밍업도 포함해 측정해 주니 편리합니다.

실제로 쓴 벤치마크 코드는 아래와 같습니다.

```java
public class RegexTest {

    private static final String PATTERN_REGEX = "^[0-9]*$";

    private static final DecimalFormat DECIMAL_FORMAT = new DecimalFormat("0000000");

    private static final Pattern PATTERN = Pattern.compile(PATTERN_REGEX);

    private static final Predicate PREDICATE = PATTERN.asPredicate();

    private static final Predicate MATCH_PREDICATE = PATTERN.asMatchPredicate();

    private static final List<String> VALUES = IntStream.rangeClosed(0, 9999999)
            .mapToObj(DECIMAL_FORMAT::format)
            .collect(Collectors.toList());

    @Benchmark
    public void patternMatches(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(Pattern.matches(PATTERN_REGEX, value));
        }
    }

    @Benchmark
    public void matcherFind(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(PATTERN.matcher(value).find());
        }
    }

    @Benchmark
    public void matcherMatches(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(PATTERN.matcher(value).matches());
        }
    }

    @Benchmark
    public void predicate(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(PREDICATE.test(value));
        }
    }

    @Benchmark
    public void matchPredicate(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(MATCH_PREDICATE.test(value));
        }
    }

    @Benchmark
    public void stringMatches(Blackhole bh) {
        for (String value : VALUES) {
            bh.consume(value.matches(PATTERN_REGEX));
        }
    }
}
```

측정 결과는 다음과 같았습니다.

```dos
Benchmark                  Mode  Cnt  Score   Error  Units
RegexTest.patternMatches  thrpt   25  0.591 ± 0.013  ops/s
RegexTest.matcherFind     thrpt   25  1.525 ± 0.022  ops/s
RegexTest.matcherMatches  thrpt   25  1.481 ± 0.030  ops/s
RegexTest.predicate       thrpt   25  2.050 ± 0.182  ops/s
RegexTest.matchPredicate  thrpt   25  1.733 ± 0.236  ops/s
RegexTest.stringMatches   thrpt   25  0.609 ± 0.005  ops/s
```

성능만 보면 `Matcher`나 `Predicate`가 유리했습니다. 특히 `Predicate`가 가장 좋았지만, `Pattern.asPredicate()`는 Java 8, `Pattern.asMatchPredicate()`는 Java 11에서 들어왔으니 JDK 버전에 맞춰 골라야 합니다.

왜 이런 차이가 나는지도 봤습니다. `Pattern.matches()`와 `String.matches()`는 호출할 때마다 `Pattern`과 `Matcher`를 새로 만들기 때문에 느립니다.

```java
// Pattern.matches
public static boolean matches(String regex, CharSequence input) {
    Pattern p = Pattern.compile(regex);
    Matcher m = p.matcher(input);
    return m.matches();
}
```

```java
// String.matches
public boolean matches(String regex) {
    return Pattern.matches(regex, this);
}
```

반면 `Matcher`나 `Predicate`는 미리 인스턴스를 만들어 두기 때문에 상대적으로 빠릅니다.

## 실제 Validator 만들기

성능상 `Matcher`와 `Predicate`가 유리하다는 걸 확인했으니, 이제 이를 이용해 업로드 경로를 검증하는 Validator를 만들면 됩니다. 이번에는 Java 11을 사용했으므로 `Predicate`를 골랐습니다.

패턴이 여러 개이니 미리 리스트로 보관하고, 검사할 때는 리스트를 순회하면서 하나라도 맞는지 확인하면 됩니다.

```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class StorageValidator {

    /**
     * 허용된 경로 패턴
     */
    private static final List<Predicate<String>> PATH_PATTERN_MATCHERS = List.of(
            createMatcher(
                    "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"),
            createMatcher(
                    "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail_backup\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"));

    /**
     * 주어진 문자열이 유효한 업로드 경로인지 판정한다.
     *
     * @param path 판정 대상 문자열
     * @return 판정 결과
     */
    public static boolean isValidUploadPath(String path) {
        return PATH_PATTERN_MATCHERS.stream().anyMatch(predicate -> predicate.test(path));
    }

    /**
     * 정규식에서 Predicate 기반 패턴 매처를 만든다.
     *
     * @param pattern 정규식
     * @return 생성된 패턴 매처
     */
    private static Predicate<String> createMatcher(String pattern) {
        return Pattern.compile(pattern).asMatchPredicate();
    }
}
```

이제 전달받은 경로가 예상 패턴과 맞는지 판정할 수 있습니다.

### 번외: Kotlin으로 바꾸면

내용과 직접 관련은 없지만, 흥미로워서 Java로 만든 Validator를 Kotlin으로 바꿔 봤습니다. IntelliJ에는 Java 코드를 Kotlin으로 바꿔 주는 기능이 있어서 어렵지 않습니다.

`static final` 필드는 Kotlin에서 `companion object`로 옮기게 됩니다. 또 `stream()` 대신 컬렉션에서 바로 쓸 수 있는 `any`가 있고, `List.of()`는 `listOf()`로 바꾸면 됩니다. 자동 변환은 여기까지는 안 해 줘서 직접 손봤습니다.

```kotlin
class StorageValidator {

    companion object {
        private val PATH_PATTERN_UPLOAD = listOf(
            Pattern.compile(
                "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"
            ).asMatchPredicate(),
            Pattern.compile(
                "\\/contents\\/images\\/\\d{0,4}\\/(19[0-9]{2}|20[0-9]{2})(0[0-9]|1[0-2])\\/thumbnail_backup\\.(?:bmp|jpg|jpeg|gif|png|BMP|JPG|JPEG|GIF|PNG)$"
            ).asMatchPredicate()
        )
    }

    fun isValidUploadPath(path: String): Boolean {
        return PATH_PATTERN_UPLOAD.any { predicate -> predicate.test(path) }
    }
}
```

## 마지막으로

정규 표현식으로 문자열이 패턴과 일치하는지 검사하는 기능 자체는 그리 어렵지 않습니다. 오히려 정규식 문법을 읽고 다듬는 쪽이 더 어렵습니다. 다만 요즘은 [RegExr](https://regexr.com), [RegEx Testing](https://www.regextester.com), [regex101](https://regex101.com) 같은 도구가 있어서 입력하면서 바로 확인할 수 있습니다.

코드 자체는 짧고 단순했지만, 어떤 API를 고를지와 성능 차이를 같이 생각해 볼 수 있어서 꽤 흥미로운 작업이었습니다. 이런 종류의 검증은 자주 등장하니 한 번 정리해 두면 다음에도 바로 응용할 수 있습니다.
