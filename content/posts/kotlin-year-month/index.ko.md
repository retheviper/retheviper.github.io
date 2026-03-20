---
title: "연월을 다뤄 보자"
date: 2021-04-27
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - spring
translationKey: "posts/kotlin-year-month"
---

Kotlin에서는 `java.time` 패키지를 통해 날짜와 시간을 다룰 수 있습니다. 예를 들면 [LocalDateTime](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/LocalDateTime.html)이나 [LocalDate](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/LocalDate.html) 같은 타입이 대표적입니다. 서버 코드에서는 이런 클래스를 이용해 DB에 날짜를 저장하거나, 토큰 만료 시간을 계산하는 작업을 흔하게 처리합니다. [Period](https://docs.oracle.com/javase/8/docs/api/java/time/Period.html)나 [Duration](https://docs.oracle.com/javase/8/docs/api/java/time/Duration.html)처럼 기간을 표현하는 타입도 이미 잘 갖춰져 있습니다.

그런데 실제 업무에서는 "날짜"가 아니라 "연월"만 다루고 싶은 경우도 자주 있습니다. 예를 들어 거래 내역을 조회하면서 "2021년 2월부터 4월까지"처럼 월 단위로 범위를 지정하는 경우가 그렇습니다. 이런 상황에서 굳이 일자나 시간까지 포함한 타입을 쓰면 모델이 불필요하게 무거워지고, 처리 과정에서 실수가 생길 여지도 커집니다.

이번 글에서는 이런 연월 데이터를 Kotlin과 Java 환경에서 어떻게 다룰 수 있는지 정리해 보겠습니다.

## 연월을 년과 월로

연월을 다룬다는 것은 결국 필요할 때 `년`과 `월` 두 값으로 나누어 쓸 수 있어야 한다는 뜻이기도 합니다. 여기서는 연월을 분리해서 다루는 방법을 두 가지로 나눠 보겠습니다.

### YearMonth로

`LocalDate`와 `LocalDateTime`은 기본적으로 [ISO-8601](https://www.iso.org/iso-8601-date-and-time-format.html) 형식의 날짜를 다룹니다. 물론 [DateTimeFormatter](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/time/format/DateTimeFormatter.html)로 다른 형식을 지정할 수도 있지만, 기본 전제는 표준 날짜 형식이라고 보면 됩니다.

이 말은 곧 Spring에서 REST API를 만들 때, 요청 값이 ISO-8601 형식을 지키고 있으면 `LocalDate`나 `LocalDateTime`으로 비교적 자연스럽게 바인딩할 수 있다는 뜻입니다. 예를 들어 아래와 같은 JSON 요청이 있다고 해 봅시다.

```json
{
    "id": "1",
    "date": "2021-04-01"
}
```

Spring 측에서는 다음과 같은 코드로 요청의 date를 `LocalDate`로 변환할 수 있습니다.

```kotlin
// 요청 본문
data class DateRequest(val id: Int, val date: LocalDate)

// 컨트롤러
@PostMapping("/date")
fun date(@RequestBody request: DateRequest) {
    // ...
}
```

같은 방식으로 [YearMonth](https://docs.oracle.com/javase/ko/8/docs/api/java/time/YearMonth.html)를 사용하면 연월도 다룰 수 있습니다. 예를 들어 요청이 아래처럼 들어온다고 해 보겠습니다.

```json
{
    "id": "1",
    "yearMonth": "2021-04"
}
```

이때 `yearMonth` 필드를 `YearMonth` 타입으로 받으면 됩니다.

```kotlin
// 요청 본문
data class YearMonthRequest(val id: Int, val yearMonth: YearMonth)

// 컨트롤러
@PostMapping("/year-month")
fun yearMonth(@RequestBody request: YearMonthRequest) {
    // ...
}
```

`YearMonth`의 장점은 `LocalDate`나 `LocalDateTime`과 같은 `java.time` 계열에 속해 있어 서로 자연스럽게 변환할 수 있다는 점입니다. 예를 들면 다음과 같습니다.

```kotlin
>>> val yearMonth = YearMonth.now() // 현재 연월을 가져온다
>>> println(yearMonth)
2021-04
>>> val localDate = yearMonth.atDay(1) // 연월에 날짜를 지정해 LocalDate로 만든다
>>> println(localDate)
2021-04-01
```

또 `YearMonth`에는 날짜 계산에 유용한 메서드가 꽤 많습니다. 단순히 `YYYYMM` 같은 숫자 대용으로 쓰는 것보다, 연월을 실제 도메인 타입으로 다뤄야 할 때 더 편합니다. 예를 들면 아래 같은 기능을 바로 쓸 수 있습니다.

```kotlin
>>> val yearMonth = YearMonth.of(2021, 5)
>>> println(yearMonth)
2021-05
>>> println(yearMonth.getYear()) // 연도를 가져온다
2021
>>> println(yearMonth.getMonth()) // 월(Enum)을 가져온다
MAY
>>> println(yearMonth.getMonthValue()) // 월(숫자)을 가져온다
5
>>> println(yearMonth.isLeapYear()) // 윤년인지 여부
false
>>> println(yearMonth.atEndOfMonth()) // 해당 월의 마지막 날(LocalDate)
2021-05-31
```

### 숫자로

가장 깔끔한 방법은 `YearMonth`를 직접 받는 것이지만, 상황에 따라서는 `Int`로 처리해야 하는 경우도 있습니다. 예를 들어 요청이 아래처럼 `202104` 형태로 들어온다면 말입니다.

```json
{
    "id": "1",
    "yearMonth": 202104
}
```

애초에 `year`와 `month`가 따로 온다면 가장 단순하겠지만, 이렇게 하나의 숫자로 묶여 들어오면 직접 연도와 월을 분리해야 합니다. 이럴 때는 간단한 확장 함수를 만들 수 있습니다.

```kotlin
// 연도를 추출한다
fun Int.extractYear(): Int = this / 100
// 월을 추출한다
fun Int.extractMonth(): Int = this % 100
```

실제 코드를 움직여 보면 제대로 의도대로 움직이는 것을 확인할 수 있습니다.

```kotlin
>>> fun Int.extractYear(): Int = this / 100
>>> 202104.extractYear()
res4: kotlin.Int = 2021
>>> fun Int.extractMonth(): Int = this % 100
>>> 202104.extractMonth()
res6: kotlin.Int = 4
```

다만 이렇게 받으면 값이 정말 `YYYYMM` 형식인지 보장되지 않는다는 문제가 있습니다. 결국 숫자 하나일 뿐이니, 잘못된 값이 들어와도 컴파일 단계에서는 막을 수 없습니다.

그래서 정규식을 이용한 검증을 함께 두는 편이 안전합니다. 예를 들어 다음처럼 처리할 수 있습니다.

```kotlin
fun Int.toYearMonth(): Pair<Int, Int> =
    if (Regex("^(19|20)\\d{2}(0[1-9]|1[012])").matches(this.toString()))
        this / 100 to this % 100
    else
        throw IllegalArgumentException("cannot convert")
```

위 함수는 다음처럼 사용할 수 있습니다.

```kotlin
>>> val (year, month) = 202104.toYearMonth()
>>> println(year)
2021
>>> println(month)
4
```

앞 예시에서는 연도와 월을 두 개의 `Int`로 나누기 위해 `Pair`를 반환했지만, 경우에 따라서는 곧바로 `YearMonth`로 바꾸는 쪽이 더 나을 수도 있습니다.

```kotlin
fun Int.toYearMonth(): YearMonth =
    if (Regex("^(19|20)\\d{2}(0[1-9]|1[012])").matches(this.toString()))
        YearMonth(this / 100, this % 100)
    else
        throw IllegalArgumentException("cannot convert")
```

## 년과 달을 연월로

이번에는 반대로 `년`과 `월`을 합쳐 연월로 만드는 경우를 보겠습니다. 즉 두 개의 `Int` 값을 하나의 `YYYYMM` 형태로 합치는 방법입니다. 여기서는 `YearMonth`를 활용하는 방법과 문자열로 처리하는 방법을 각각 살펴보겠습니다.

### YearMonth에서

먼저 `YearMonth`를 쓰는 경우입니다. 연도와 월을 인자로 넘겨 객체를 만든 뒤, 문자열에서 `-`를 제거하고 `Int`로 바꾸면 됩니다. `YearMonth`는 기본적으로 `2021-04` 같은 형식으로 표현되기 때문에, 이 한 단계가 필요합니다.

```kotlin
fun toYearMonth(year: Int, month: Int): Int = 
    YearMonth.of(year, month).toString().replace("-", "").toInt()
```

### 문자열로

문자열로 처리할 수도 있습니다. 다만 월은 1자리일 수 있으므로, 단순히 문자열을 이어 붙이면 `20214`처럼 잘못된 값이 됩니다. 그래서 [String templates](https://kotlinlang.org/docs/basic-types.html#string-templates)와 `padStart()`를 함께 써서 월을 두 자리로 맞춰 줍니다.

```kotlin
fun toYearMonth(year: Int, month: Int): Int = "${year}${month.toString().padStart(2, '0')}".toInt()
```

이 함수는 인자가 두 개라서, 취향에 따라 `infix` 형태로 정의할 수도 있습니다.

```kotlin
>>> infix fun Int.toYearMonthWith(month: Int): Int = "${this}${month.toString().padStart(2, '0')}".toInt()
>>> 2021 toYearMonthWith 5
res10: kotlin.Int = 202105
```

## 마지막으로

코드 자체는 아주 어렵지 않지만, 막상 연월을 깔끔하게 다루려면 어떤 타입을 써야 할지 잠깐 고민하게 되는 경우가 있습니다. 저도 이번에 `YearMonth`를 다시 살펴보면서, 단순한 숫자보다 도메인에 맞는 타입을 쓰는 편이 훨씬 명확하다는 점을 새삼 느꼈습니다.

Kotlin이나 Java에서 연월 데이터를 다룰 일이 있다면, 상황에 따라 `YearMonth`와 숫자 표현을 적절히 골라 쓰는 데 이 글이 조금이라도 도움이 되면 좋겠습니다.
