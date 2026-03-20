---
title: "Kotlin에서 일본 달력 다루기"
date: 2021-12-05
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-japanese-era"
---

장표나 문서를 만들다 보면 가끔 일본 달력을 다뤄야 할 때가 있습니다. 예를 들어 연호를 표기하거나, 서기가 아니라 일본식 연도를 함께 보여 줘야 하는 경우입니다. Kotlin(JVM)에서는 서기 날짜를 다루는 일은 `Date`나 `LocalDate`만으로도 충분하지만, 일본 달력은 자주 쓰는 기능이 아니다 보니 막상 필요할 때 방법이 바로 떠오르지 않을 때가 많습니다. 그래서 이번에는 Kotlin에서 일본 달력을 다루는 방법을 간단히 정리해 보겠습니다.

## JapaneseEra/JapaneseDate

Java는 1.8부터 일본 달력 날짜를 다루는 [JapaneseDate](https://docs.oracle.com/javase/ko/11/docs/api/java.base/java/time/chrono/JapaneseDate.html)와, 연호를 다루는 [JapaneseEra](https://docs.oracle.com/javase/ko/11/docs/api/java.base/java/time/chrono/JapaneseEra.html)를 제공합니다. 그래서 `JapaneseDate` 인스턴스를 만든 뒤 여기서 `JapaneseEra`를 꺼내면 연호 정보를 쉽게 얻을 수 있습니다. 사용법은 다음과 같습니다.

```kotlin
// 현재 날짜의 JapaneseDate를 가져온다
val japaneseDate = JapaneseDate.now()

// JapaneseEra를 가져온다
val japaneseEra = japaneseDate.era
```

`JapaneseDate`의 경우 `LocalDate`과 마찬가지로 [ChronoLocalDate](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/chrono/ChronoLocalDate.html)를 상속하고 있으므로 인스턴스를 만드는 방법은 그렇습니다. 그래서 다음과 같은 것도 가능합니다.

```kotlin
// LocalDate를 JapaneseDate로 변환한다
val japaneseDateFromLocalDate = JapaneseDate.from(LocalDate.now())

// 특정 날짜를 지정해 JapaneseDate를 만든다
val japaneseDateFromSpecificDate = JapaneseDate.of(2000, 12, 31)
```

## 원호를 일본어로 표기

일본 달력을 다룰 때 하고 싶은 일은 크게 두 가지라고 생각합니다. 하나는 연호를 문자열로 다루는 일이고, 다른 하나는 연도를 숫자로 다루는 일입니다. 먼저 연호를 문자열로 얻는 방법부터 보겠습니다.

앞에서 본 것처럼 먼저 `JapaneseDate`를 만들고, 여기서 `JapaneseEra`를 꺼냅니다. 그다음 `JapaneseEra.getDisplayName()`에 [TextStyle](https://docs.oracle.com/javase/ko/8/docs/api/java/time/format/TextStyle.html)과 [Locale](https://docs.oracle.com/javase/ko/11/docs/api/java.base/java/util/Locale.html)을 넘기면 문자열 형태의 연호를 얻을 수 있습니다. 앞의 값은 표기 스타일이고, 뒤의 값은 언어 설정입니다.

`TextStyle`에는 아래와 같은 값이 있습니다. 언어 설정에 따라 결과는 꽤 달라질 수 있지만, 일본어 기준으로는 `FULL`과 `NARROW` 정도만 알아도 대부분 충분합니다.

| 상수 | 출력 예시 |
|---|---|
| `FULL` | 쇼와 |
| `FULL_STANDALONE` | 쇼와 |
| `NARROW` | S |
| `NARROW_STANDALONE` | S |
| `SHORT` | 쇼와 |
| `SHORT_STANDALONE` | 쇼와 |

`Locale`은 `Locale.JAPAN`과 `Locale.JAPANESE` 모두 비슷하게 동작합니다. 다만 내부적으로는 아래처럼 설정이 조금 다르기 때문에, 가능하면 `Locale.JAPAN`을 쓰는 편이 더 명확합니다.

| Locale | 생성되는 BaseLocale 설정 |
|---|---|
| `JAPAN` | `language = ja, region = JP` |
| `JAPANESE` | `language = ja` |

아래는 실제로 연호 문자열을 얻는 예입니다.

```kotlin
val today = JapaneseDate.now()
val era = today.era

// 연호를 한자로 가져온다
val eraName = era.getDisplayName(TextStyle.FULL, Locale.JAPAN) // 레이와
```

연호뿐 아니라 연도까지 함께 표기하고 싶을 때는 [DateTimeFormatter](https://docs.oracle.com/javase/ko/11/docs/api/java.base/java/time/format/DateTimeFormatter.html)를 쓰면 됩니다. `JapaneseDate` 역시 `ChronoLocalDate` 계열이기 때문에 이런 포맷팅이 가능합니다.

```kotlin
// 날짜를 일본어로 표기한다
val formatter = DateTimeFormatter.ofPattern("Gy년", Locale.JAPAN)
val todayString = formatter.format(JapaneseDate.now()) // 레이와 3년
```

만약 Java 8 이전 환경처럼 `LocalDate`나 `JapaneseDate`를 쓸 수 없고 `java.util.Date`만 써야 한다면, 아래 같은 방식으로 연호와 연도를 얻을 수 있습니다.

```kotlin
val format = SimpleDateFormat("Gy년", Locale("Ja", "JP", "JP"))
val year = format.format(Date()) // 레이와 3년
```

`java.util.Date`를 쓸 때는 `Locale`에 세 번째 인수인 `variant`까지 지정해야 해서, 미리 정의된 상수를 그대로 쓰기는 어렵습니다.

또한 `Locale.ENGLISH` 등으로 설정하면 `JapaneseDate`를 사용하는 경우에도 얻은 결과는 `AD2021년12월5일`입니다.

### 합자로 표기

연호를 유니코드 합자 형태로 쓰고 싶을 때도 있습니다. 그럴 때는 아래처럼 매핑 테이블을 하나 두고 꺼내 쓰는 편이 단순합니다. 확장 함수로 감싸도 괜찮습니다.

```kotlin
val eraUnicodeMap = mapOf(
    JapaneseEra.MEIJI to "\u337e", // ㍾
    JapaneseEra.TAISHO to "\u337d", // ㍽
    JapaneseEra.SHOWA to "\u337c", // ㍼
    JapaneseEra.HEISEI to "\u337b", // ㍻
    JapaneseEra.REIWA to "\u32ff" // ㋿
)

val era = JapaneseDate.now().era
// 연호를 합자 형태로 가져온다
val eraUnicode = eraUnicodeMap[era] // ㋿
```

위 예제에서는 `JapaneseEra`가 enum이므로 그대로 키로 썼지만, 숫자 값을 기준으로 처리하는 방법도 가능합니다. 각 값은 아래와 같습니다.

| JapaneseEra | 숫자 |
|---|---|
| MEIJI | -1 |
| TAISHO | 0 |
| SHOWA | 1 |
| HEISEI | 2 |
| REIWA | 3 |

예를 들어 2021년이나 2022년 3월은 레이와 3년이지만, 그렇다고 `JapaneseEra.REIWA.value`가 연도를 뜻하는 것은 아닙니다. 실제 연도 정보는 `JapaneseDate` 쪽에 있으니 헷갈리지 않는 편이 좋습니다.

## 연도를 숫자로 표시

`JapaneseEra`는 어디까지나 연호를 나타내는 enum이므로, 날짜 자체의 정보는 들고 있지 않습니다. 즉 여기서 확인할 수 있는 것은 해당 날짜가 속한 연호뿐입니다.

그래서 숫자 형태의 연도를 얻으려면 [ChronoField](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/temporal/ChronoField.html)를 `JapaneseDate.get()`에 넘겨서 가져와야 합니다.

```kotlin
val today = JapaneseDate.of(2010, 12, 31) // 헤이세이 22년

// 연도를 Int로 가져온다
val year = today.get(ChronoField.YEAR) // 2010
val yearOfHeisei = today.get(ChronoField.YEAR_OF_ERA) // 22
```

이 방식이 필요한 이유는 `JapaneseDate`가 `LocalDate`처럼 `year` 프로퍼티를 바로 노출하지 않기 때문입니다. 내부적으로는 `LocalDate`와 `yearOfEra`를 따로 가지고 있어서, `get(ChronoField.YEAR_OF_ERA)`를 호출해야 비로소 연호 기준 연도를 얻을 수 있습니다. Kotlin이라면 필요할 때 확장 함수를 만들어 감싸는 것도 괜찮겠습니다.

반대로 날짜 객체가 `LocalDate`라면 `ChronoField.YEAR_OF_ERA`를 줘도 우리가 기대하는 일본식 연도가 나오지 않습니다. 일본 달력이 필요하다면 먼저 `JapaneseDate`를 쓰고 있는지부터 확인해야 합니다.

### 연도를 두 자리 문자로 표시

엄밀히는 일본 달력 자체의 기능은 아니지만, 연도를 꺼내 쓸 때 앞에 `0`을 붙인 두 자리 문자열로 맞추고 싶을 때도 있습니다. `JapaneseDate`에서 꺼낸 연도는 `Int`이므로, 1~9는 한 자리 숫자로 나옵니다. 이를 `01`~`09`처럼 맞추고 싶다면 아래 방법을 쓸 수 있습니다.

#### DecimalFormat 사용

하나는 Java API인 [DecimalFormat](https://docs.oracle.com/javase/ko/11/docs/api/java.base/java/text/DecimalFormat.html)을 쓰는 방법입니다. 포맷을 눈에 보이게 지정할 수 있어서 개인적으로는 이쪽을 더 자주 씁니다.

```kotlin
val today = JapaneseDate.now() // 레이와 3년

// 숫자를 표시하기 위한 포맷을 지정한다
val decimalFormat = DecimalFormat("00")
val year = decimalFormat.format(today) // 03
```

#### String.format 사용

또 다른 방법은 Kotlin 표준 라이브러리의 [String.format()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/format.html)을 쓰는 것입니다. 단순한 문자열 포맷만 필요하다면 이쪽도 충분합니다.

```kotlin
val today = JapaneseDate.now() // 레이와 3년

// 숫자를 표시하기 위한 포맷을 지정한다
val year = "%02d".format(today) // 03
```

## 번외: kotlinx-datetime

Kotlin에는 원래 날짜와 시간을 다루는 표준 API가 없었지만, 2020년부터 [kotlinx-datetime](https://github.com/Kotlin/kotlinx-datetime)을 제공하고 있습니다. 덕분에 Kotlin/JS나 Kotlin/Native처럼 JVM이 아닌 환경에서도 날짜를 다룰 수 있는 공식 API가 생겼습니다. 다만 몇 가지는 미리 생각해 둘 필요가 있습니다.

### Pre-release 단계

`kotlinx-datetime`은 당시 기준으로 아직 초기 단계였고, 기능 폭도 `java.time`에 비해 훨씬 좁았습니다. 연호 계산처럼 특수한 달력 처리도 바로 지원하지 않기 때문에, 최소한의 날짜 API 정도로 보는 편이 맞았습니다.

### 멀티 플랫폼용

Kotlin 표준 라이브러리나 `kotlinx` 계열 라이브러리는 멀티플랫폼을 전제로 만들어지기 때문에, 플랫폼이 달라도 같은 방식으로 쓸 수 있다는 장점이 있습니다. 반대로 말하면 JVM만 사용하는 프로젝트에서는 그 장점이 크게 와닿지 않을 수도 있습니다. 실제로 `kotlinx-datetime`의 JVM 구현은 내부적으로 `java.time`에 의존합니다.

또 플랫폼별 구현 차이 때문에, 어느 한쪽에서는 잘 되는데 다른 쪽에서는 예상과 다르게 동작하는 상황이 생길 수도 있습니다.

## java.time 우려

`JapaneseEra`에서는 메이지 이전(경응 등)의 원호는 사용할 수 없지만, 아마 그 이유는 일본 달력으로 그레고리우스 달력이 사용된 것은 메이지에서였다는 역사적인 배경이 있는 것은 아닐까 생각합니다. 또한 `JapaneseDate`에서도 메이지 6년(기원 1873년 1월 1일) 이전의 날짜를 지정하면 다음과 같이 예외가 발생합니다.

```shell
Exception in thread "main" java.time.DateTimeException: JapaneseDate before Meiji 6 is not supported
	at java.base/java.time.chrono.JapaneseDate.<init>(JapaneseDate.java:333)
	at java.base/java.time.chrono.JapaneseDate.of(JapaneseDate.java:257)
```

그래서 단순히 장표를 만드는 등의 경우가 아니라 역사적인 연구를 위한 날짜 계산에서는 여기서 소개한 방법은 사용할 수 없는 경우도 있을까 생각합니다.

또, JDK의 버전 등의 문제가 있기 때문에, `JapaneseEra.REIWA`의 취득을 할 수 없어, 에러가 되는 경우가 있으므로 주의할 필요가 있습니다. 이 경우에도 `value`의 값의 취득은 문제 없기 때문에, 조금 가독성은 저하하면서 분기등의 판정에 정수를 그대로 사용하는 것은 피하는 편이 좋을 것 같습니다. (정확한 이유는 모르겠지만…)

## 마지막으로

이번 글은 흥미가 생겨 조사한 내용을 정리한 것이기도 하지만, 실제 업무에서 필요한 처리와도 맞닿아 있어서 꽤 유용한 정리였습니다. Kotlin 쪽에서는 이런 로직을 어떤 확장 함수로 감싸면 더 편하게 쓸 수 있을지도 함께 고민해 볼 만했습니다.

또 Java API 쪽은 [Java 버전별 개원(신원호) 대응 정리](https://qiita.com/yamadamn/items/56e7370bae2ceaec55d5)도 잘 정리돼 있어서, 더 깊게 보고 싶다면 함께 읽어 볼 만합니다.
