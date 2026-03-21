---
title: "데이터 클래스 목록을 CSV로 쓰기"
date: 2022-08-27
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
  - csv
translationKey: "posts/kotlin-write-csv"
---

앱을 쓰면 DB에 저장한 형태와는 다른 형태로 데이터를 읽거나 출력하는 경우가 많습니다. 대표적으로, 많은 백엔드 앱에서 채용하고 있는 REST API가 그렇네요. API의 입력값과 반환값은, DB에 보존되고 있는 형태와는 일치하지 않는 경우가 많아, 필요(요청을 송신해 온 측의 편리함)에 맞추어 적절히 파라미터를 받아, 응답을 돌려주게 되어 있습니다. 그리고 때와 때로는 인간이 쉽게 읽을 수 있는 형태로 데이터를 정리할 필요가 있습니다. 그러한 경우는 Excel 파일이나 CSV, PDF라고 하는 여러가지 것을 상정할 수 있네요.

이번 글에서는 사람이 읽기 쉬운 파일 형식 중에서도 CSV에 집중해, Kotlin에서 데이터 클래스 목록을 CSV로 내보내는 방법을 정리해 보겠습니다.

## Kotlin에서 CSV 처리

Kotlin에서 CSV를 읽고 쓰는 라이브러리로는 [kotlin-csv](https://github.com/doyaaaaaken/kotlin-csv/)가 있습니다. JVM뿐 아니라 Kotlin/JS에서도 쓸 수 있고, 기본적인 CSV 처리 자체는 꽤 편합니다. 또 [kotlin-grass](https://github.com/blackmo18/kotlin-grass) 같은 라이브러리를 조합하면 CSV 데이터를 data class 목록으로 매핑하는 일도 조금 더 수월해집니다. 읽기 옵션이나 커스텀 매핑도 어느 정도 제공해서, 전반적으로 쓰기 괜찮은 라이브러리입니다.

그러나 사실 `kotlin-csv`을 사용할 때 문제가 하나 있습니다. 앞서 언급했듯이 CSV에 데이터 출력 자체는 가능하지만 읽을 때 데이터 클래스에 매핑하려면 다른 라이브러리가 필요했듯이 데이터 클래스 목록을 작성하려면 추가 처리가 필요합니다. 이것은 kotlin-csv를 쓰는 방법이 다음과 같기 때문입니다.

```kotlin
fun writeAll(rows: List<List<Any?>>, targetFile: File, append: Boolean = false) {
    open(targetFile, append) { writeRows(rows) }
}
```

여기서 `rows`는 쓰기에 사용하는 데이터인데, 타입이 `List<List<Any?>>`이므로 내부의 `List` 하나가 CSV의 한 행을 뜻하고, 그 행들을 다시 `List`로 모은 것이 CSV 전체가 됩니다. 즉 data class 리스트를 쓰려면 각 인스턴스의 필드 값을 한 행짜리 `List`로 바꾸고, 그것들을 행 목록으로 모아야 합니다. 또한 CSV에는 보통 헤더가 있으므로, 첫 번째 행에는 헤더만 담은 `List`도 따로 준비해야 합니다.

보기에는 복잡해 보이지만 [reflection](https://kotlinlang.org/docs/reflection.html)을 이용하면 data class의 필드명과 그 값을 얻을 수 있으므로, data class의 List를 이 메서드에 맞는 형태로 바꿀 수 있습니다. 아래에서는 헤더를 만드는 단계와 data class 값을 행 데이터로 바꾸는 단계를 나누어 설명하겠습니다.

### 데이터 클래스에서 헤더 만들기

먼저 헤더를 만듭니다. 헤더를 만들려면 data class에서 필드를 가져오고, 그 이름만 뽑아 쓰면 됩니다. `id`라는 필드가 있다면 헤더도 그대로 `id`가 되는 식입니다. 필드명과 다른 이름을 붙이고 싶다면 어노테이션을 활용하는 방법도 있지만, 우선은 필드명을 그대로 사용하는 방법부터 보겠습니다.

Kotlin의 data class에서 필드를 얻는 방법에는 세 가지가 있습니다. 우선은 [KClass.members](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-class/members.html)가 있습니다. 다만, 이것이라면 메소드를 포함해, 모든 멤버를 가져오게 됩니다. 다음과 같습니다.

```kotlin
data class Data(val id: Int, val name: String)

val members = Data::class.members
println(members)
// [val Line_2.Data.id: kotlin.Int, val Line_2.Data.name: kotlin.String, fun Line_2.Data.component1(): kotlin.Int, fun Line_2.Data.component2(): kotlin.String, fun Line_2.Data.copy(kotlin.Int, kotlin.String): Line_2.Data, fun Line_2.Data.equals(kotlin.Any?): kotlin.Boolean, fun Line_2.Data.hashCode(): kotlin.Int, fun Line_2.Data.toString(): kotlin.String]
```

필드명이 나오기 때문에 이것도 쓸 수는 있지만, data class에는 기본적으로 `equals()`, `hashCode()`, `copy()`, `toString()`, `componentN()` 같은 메서드도 함께 포함됩니다. 따라서 이런 항목들은 따로 걸러 내야 합니다. 예를 들면 다음과 같습니다.

```kotlin
val memberProperties = Data::class.members.filterNot { it.name.contains("component") || it.name == "copy" || it.name == "equals" || it.name == "hashCode" || it.name == "toString" }

println(memberProperties)
// [val Line_2.Data.id: kotlin.Int, val Line_2.Data.name: kotlin.String]
```

그러나 필터를 사용하지 않고도 쉽게 필드만 추출할 수 있는 방법도 있습니다. [memberProperties](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect.full/member-properties.html)를 사용하는 것입니다.

```kotlin
val memberProperties = Data::class.memberProperties

println(memberProperties)
// [val Line_2.Data.id: kotlin.Int, val Line_2.Data.name: kotlin.String]
```

다만 이 방법에도 문제가 있습니다. 가져온 필드 순서가 data class에 선언한 순서가 아니라 알파벳 순서가 될 수 있다는 점입니다. 아래 예제에서는 `name`, `age` 순서로 선언했는데 결과는 `age`, `name` 순서로 나옵니다.

```kotlin
data class Person(val name: String, val age: Int)

val memberProperties = Person::class.memberProperties
println(memberProperties) // name, age 순서가 되지 않는다
// [val Line_11.Person.age: kotlin.Int, val Line_11.Person.name: kotlin.String]
```

그래서 필드를 선언한 순서대로 가져오고 싶다면 data class 생성자를 이용하는 편이 낫습니다. data class는 기본 생성자 파라미터 순서가 곧 필드 선언 순서와 연결되어 있기 때문입니다.

```kotlin
val parameters = Person::class.primaryConstructor!!.parameters.mapNotNull { it.name }

println(parameters) // [name, age]
```

조금 우회적인 방식이긴 하지만, 필요한 필드 이름은 이 방법으로 안정적으로 가져올 수 있습니다. 다음은 어노테이션으로 헤더명을 바꾸는 경우를 보겠습니다.

#### 어노테이션을 사용하는 경우

필드 이름을 그대로 헤더로 쓰고 싶지 않다면 어노테이션을 활용할 수 있습니다. 문자열 값을 가진 어노테이션을 정의해 두고, 헤더를 만들 때 그 값을 읽어 오는 방식입니다. 예를 들면 아래와 같습니다.

```kotlin
@Target(AnnotationTarget.PROPERTY)
annotation class CsvHeaderName(val value: String)
```

어노테이션은 다음과 같이 데이터 클래스에서 사용됩니다.

```kotlin
data class Person(
    @CsvHeaderName("이름")
    val name: String,
    @CsvHeaderName("나이")
    val age: Int
)
```

이제 이 data class에서 어노테이션을 읽어 오면 됩니다. `memberProperties`로 필드를 가져온 뒤, 각 필드의 어노테이션 목록에서 `CsvHeaderName`만 골라 값을 쓰면 됩니다. 어노테이션이 없으면 필드명을 그대로 사용하면 됩니다.

```kotlin
val datas = listOf(Person("John", 20))

val headers = datas.first()!!::class.memberProperties.map { property ->
            val name = property.annotations.filterIsInstance<CsvHeaderName>().firstOrNull() // 어노테이션이 없는 경우도 있다
            name?.value ?: property.name // 어노테이션이 없으면 필드명을 사용한다
        }

println(headers) // [나이, 이름]
```

`primaryConstructor` 기준으로 필드 순서를 맞췄다면 방법도 크게 다르지 않습니다. 생성자 파라미터를 기준으로 돌면서 이름이 일치하는 필드를 찾아 같은 방식으로 처리하면 됩니다.

```kotlin
val fieldNames = datas.first()::class.primaryConstructor!!.parameters.mapNotNull { it.name }

val headers = fieldNames.mapNotNull { name ->
    // 파라미터와 일치하는 필드를 대상으로 처리한다
    datas.first()::class.memberProperties.find { it.name == name }?.let { property ->
        val headerName = property.annotations.filterIsInstance<CsvHeaderName>().firstOrNull()
        headerName?.value ?: property.name
    }
}
```

이제 헤더 데이터는 준비됐습니다. 다음은 각 data class 인스턴스를 실제 CSV 행 형태의 `List`로 바꾸면 됩니다.

### 데이터 클래스를 List로 변경

헤더를 만들 때와 마찬가지로, data class를 `List`로 바꾸는 과정도 크게 다르지 않습니다. 이번에는 필드 이름이 아니라 실제 필드 값을 읽어 와야 한다는 점만 다릅니다. 여기서는 생성자 파라미터 순서를 기준으로 처리하는 예를 보겠습니다.

Kotlin reflection으로 필드 값을 가져오는 방식은 Java 쪽과 크게 다르지 않습니다. 해당 필드에 data class 인스턴스를 넘겨 호출하면 됩니다. 다만 `null`은 따로 처리해야 합니다. `null` 때문에 열 하나가 통째로 빠지면 CSV 열 순서가 어긋날 수 있기 때문입니다. 그래서 빈 문자열 같은 기본값으로 길이를 맞춰 주는 편이 안전합니다.

```kotlin
datas.map { d ->
    fieldNames.mapNotNull { name ->
        d::class.memberProperties.find { it.name == name }?.let { field ->
            field.call(d) ?: "" // 필드 값을 가져오고 null이면 빈 문자열로 둔다
            } 
        }
    }
```

다만 여기서 시간이나 날짜를 다룬다면 포맷터를 함께 쓰고 싶어질 수 있습니다. 예를 들어 앱 안에서는 `LocalTime`으로 들고 있지만 CSV에서는 `HH:mm`으로 출력하고 싶거나, `LocalDate`를 `yy/MM/dd`로 내보내고 싶은 경우입니다. 포맷 자체는 [DateTimeFormatter](https://docs.oracle.com/javase/ko/8/docs/api/java/time/format/DateTimeFormatter.html)를 쓰면 되는데, 문제는 가져온 필드가 어떤 타입인지 판별하는 부분입니다.

Reflection으로 가져온 필드는 [KProperty1](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-property1/) 형태입니다. 여기서 관건은 원래 타입을 어떻게 판별하느냐인데, 이 클래스는 [KCallable](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-callable/)을 구현하고 있고, [returnType](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-callable/return-type.html)을 통해 [KType](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-type/)을 얻을 수 있습니다. 이 값을 기준으로 분기하면 됩니다.

다만 `KType`은 Kotlin 쪽 타입 표현이라 `LocalDate`, `LocalTime` 같은 Java 타입과 바로 비교하기는 조금 애매합니다. 그래도 Java 클래스도 Kotlin이 이해할 수 있는 `KType`으로 바꿔 비교할 수 있습니다.

```kotlin
val localDateKType: KType = LocalDate::class.createType()
```

이제 남은 일은 분기해서 적절한 포맷터를 적용하는 것뿐입니다.

```kotlin
datas.map { d ->
        fieldNames.mapNotNull { name ->
            d::class.memberProperties.find { it.name == name }?.let { field ->
                field.call(d)?.let {
                    when (field.returnType) { // 타입에 따른 분기
                        LocalDate::class.createType() -> dateFormatter.format(it as LocalDate)
                        LocalTime::class.createType() -> timeFormatter.format(it as LocalTime)
                        LocalDateTime::class.createType() -> dateTimeFormatter.format(it as LocalDateTime)
                        else -> it
                    }
                } ?: ""
            }
        }
    }
```

다만 여기서 하나 더 주의할 점이 있습니다. nullable 필드의 `KType`은 non-null 타입과 다르게 취급됩니다. 즉 위 코드만으로는 아래 같은 data class에서 포맷터가 제대로 동작하지 않습니다.

```kotlin
// 어떤 필드도 포맷되지 않는다
data class DateTimes(
    val date: LocalDate?,
    val time: LocalTime?,
    val dateTime: LocalDateTime?
)
```

이 경우에는 `KType`을 만들 때 nullable 여부를 함께 지정하면 해결할 수 있습니다. 즉 분기에서 nullable 타입까지 같이 비교해 주면 됩니다.

```kotlin
datas.map { d ->
        fieldNames.mapNotNull { name ->
            d::class.memberProperties.find { it.name == name }?.let { field ->
                field.call(d)?.let {
                    when (field.returnType) { // nullable 여부에 따라 분기하여 포맷한다
                        LocalDate::class.createType(), LocalDate::class.createType(nullable = true) -> dateFormatter.format(it as LocalDate)
                        LocalTime::class.createType(), LocalTime::class.createType(nullable = true) -> timeFormatter.format(it as LocalTime)
                        LocalDateTime::class.createType(), LocalDateTime::class.createType(nullable = true) -> dateTimeFormatter.format(it as LocalDateTime)
                        else -> it
                    }
                } ?: ""
            }
        }
    }
```

마지막으로 헤더와 data class에서 뽑은 값 목록을 합쳐 `kotlin-csv`의 `writeAll()`에 넘기면 됩니다. 값 쪽은 이미 `List<List<Any>>` 형태지만, 헤더는 `List<String>`이므로 한 번 더 리스트로 감싸 주면 됩니다.

```kotlin
// 헤더
val headers = // 생략
// 실제 데이터
val rows = datas.map { /** 생략 */ }

csvWriter().writeAll(
            rows = listOf(headers) + rows,
            targetFile = targetFile
        )
```

이제 헤더가 먼저 행에 기록되고 다음 행에서 데이터 클래스의 필드에 저장된 실제 값이 출력됩니다.

## 마지막으로

이번에는 "Kotlin이니 Kotlin 라이브러리를 써 보자"는 가벼운 마음으로 시작했는데, 막상 원하는 형태와는 조금 달라서 꽤 헤맸습니다. 다행히 예전에 Java에서 [Apache POI](https://poi.apache.org/)로 비슷한 기능을 직접 만들어 본 경험이 있어서 그때 감각이 꽤 도움이 됐습니다. 당시에는 꽤 고생했지만, 지금 돌아보면 그런 경험이 있었기에 이번에도 대응할 수 있었던 것 같습니다.

위 코드들은 간단한 라이브러리 형태로도 묶어 보았기 때문에, 기회가 되면 다른 곳에서도 다시 써 보고 싶습니다. 조금 더 다듬으면 나중에 Maven Repository 같은 곳에 공개하는 것도 가능할 것 같습니다.
