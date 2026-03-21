---
title: "Kotlin에서 기준을 뒤집어 그룹화하기"
date: 2022-01-29
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
translationKey: "posts/kotlin-reverse-groupping"
---

DB에 저장된 데이터 구조와, 애플리케이션에서 실제로 쓰고 싶은 데이터 구조가 크게 다른 경우는 흔합니다. 특히 기능이 늘어나고 화면이나 API 요구사항이 바뀌기 시작하면, 같은 데이터를 전혀 다른 기준으로 다시 묶어야 할 때가 자주 생깁니다. DB는 정규화와 저장 효율을 우선하지만, 애플리케이션은 조회와 가공 편의성이 더 중요하기 때문입니다.
이번 글에서는 그런 상황에서 써 볼 수 있는 데이터 재구성 방법을 소개합니다. 거창한 알고리즘이라기보다는, 기준이 뒤집힌 데이터를 Kotlin 컬렉션과 `Map`으로 다시 묶는 작은 패턴에 가깝습니다. 응용 범위는 꽤 넓습니다.
## 시나리오

예를 들어 다음과 같은 시나리오가 있다고 가정합니다.
1. 직원은 A, B라는 두 부서에 배속됩니다.
2. 각 부서에 배속되는 날짜는 서로 다릅니다.

이 데이터를 표현하는 방법은 여러 가지가 있겠지만, 우선 "부서 배치일"을 기준으로 보면 부서 종류와 날짜, 그리고 그 날짜에 배치된 직원 목록을 묶는 형태가 자연스럽습니다. Kotlin 코드로 쓰면 다음과 같습니다.
```kotlin
enum class DepartmentType { A, B }

data class Department(
    val departmentType: DepartmentType,
    val date: LocalDate,
    val employers: List<Employer>
) {
    data class Employer(
        val id: Int
    )
}
```

여기서 사원의 3명이 있고, 각각 부서 A와 부서 B에 배속된 날짜가 다른 경우가 있다고 합시다. 데이터는 다음과 같습니다.
| 사번 | 부서 A 배정 | 부서 B 배정 |
|---|---|---|
| 1 | 1월 1일 | 1월 1일 |
| 2 | 1월 1일 | 2월 1일 |
| 3 | 2월 1일 | 2월 1일 |

위 표를 기준으로 실제 데이터를 만들면 대략 이런 리스트가 됩니다.
```kotlin
val departments = listOf(
    Department(
        departmentType = DepartmentType.A,
        date = LocalDate.of(2022, 1, 1),
        employers = listOf(
            Department.Employer(id = 1),
            Department.Employer(id = 2)
        ),
    ),
    Department(
        departmentType = DepartmentType.A,
        date = LocalDate.of(2022, 2, 1),
        employers = listOf(
            Department.Employer(id = 3)
        ),
    ),
    Department(
        departmentType = DepartmentType.B,
        date = LocalDate.of(2022, 1, 1),
        employers = listOf(
            Department.Employer(id = 1)
        ),
    ),
    Department(
        departmentType = DepartmentType.B,
        date = LocalDate.of(2022, 2, 1),
        employers = listOf(
            Department.Employer(id = 2),
            Department.Employer(id = 3),
        )
    )
)
```

그런데 이번에 필요한 것은 그 반대 방향의 데이터입니다. 즉 직원을 기준으로, 각 부서에 언제 배치됐는지를 모아 보고 싶은 상황입니다. 코드로 표현하면 다음과 같습니다.
```kotlin
data class JoinedDates(
    val employerId: Int,
    val departmentA: LocalDate,
    val departmentB: LocalDate
)
```

결국 목표는 앞의 `departments` 리스트를 아래 같은 형태로 바꾸는 것입니다.
```kotlin
[
  JoinedDates(employerId=1, departmentA=2022-01-01, departmentB=2022-01-01), 
  JoinedDates(employerId=2, departmentA=2022-01-01, departmentB=2022-02-01), 
  JoinedDates(employerId=3, departmentA=2022-02-01, departmentB=2022-02-01)
]
```

기준이 `부서 -> 직원`에서 `직원 -> 부서`로 바뀌기 때문에, 중간에 데이터를 다시 모으는 단계가 필요합니다. 여기서는 제가 사용한 방법을 소개하겠습니다.
## 로직

`Department` 기준에서는 하나의 부서 항목 안에 여러 직원이 들어 있습니다. 이번 요건은 이 구조를 뒤집어, 각 직원이 여러 부서 정보를 갖는 형태로 재구성하는 것입니다. 핵심은 아래 두 단계입니다.
1. Employer의 ID별로 정리
1. Employer마다 Department를 Type별로 나눈 배열을 갖게 한다

먼저 중첩된 `Employer` 목록을 순회하면서 ID를 꺼내야 합니다. 이 ID는 중복되면 안 되므로 `Map`의 키로 두는 편이 적절합니다.
그다음에는 해당 ID가 이미 `Map`에 있는지에 따라 아래처럼 처리하면 됩니다.
1. Key가 존재하지 않는 경우는, 새롭게 Department의 타입과 그 일자를 insert
2. Key가 존재하는 경우는, 그 value를 꺼내 Department의 타입과 일자를 추가

즉, `Department` 리스트를 먼저 중간 `Map`으로 변환하고, 마지막에 다시 `JoinedDates` 리스트로 바꾸는 흐름입니다. 위 분기는 [compute()](https://docs.oracle.com/javase/8/docs/api/java/util/Map.html#compute-K-java.util.function.BiFunction-)를 쓰면 비교적 깔끔하게 처리할 수 있습니다. 저는 중간 데이터를 `Map<직원 ID, Map<부서 타입, 날짜>>` 형태로 잡았고, 최종 코드는 아래와 같았습니다.
```kotlin
fun List<Department>.toJoinedDates(): List<JoinedDates> {
    // 중간 데이터
    val tempMap = mutableMapOf<Int, Map<DepartmentType, LocalDate>>()

    this.forEach { department ->
        // Department 타입과 해당 날짜의 Pair
        val departmentJoined = department.departmentType to department.date

        department.employers.forEach { employer ->

            // Employer ID가 키로 있으면 더하고, 없으면 Map에 추가
            tempMap.compute(employer.id) { _, value ->
                value?.let { value + departmentJoined } ?: mapOf(departmentJoined)
            }
        }
    }

    // 중간 데이터를 JoinedDates의 List로 바꿔 반환
    return tempMap.map { (id, department) ->
        JoinedDates(
            employerId = id,
            departmentA = department.getValue(DepartmentType.A),
            departmentB = department.getValue(DepartmentType.B)
        )
    }
}
```

## 공통 로직화

이 로직은 비슷한 문제에도 재사용할 수 있을 것 같아서, 일부를 제네릭 클래스로 분리한 버전도 만들어 봤습니다.
```kotlin
class Aggregator<T, K, V, R> {
    private val tempMap = mutableMapOf<T, Map<K, V>>()

    // 데이터 추가
    fun add(key: T, value: Pair<K, V>) {
        tempMap.compute(key) { _, existingValue ->
            existingValue?.let { existingValue + value } ?: mapOf(value)
        }
    }

    // 지정한 List로 반환
    fun getList(transfer: (T, Map<K, V>) -> R): List<R> {
        return tempMap.map { transfer(it.key, it.value) }
    }
}
```

이 경우에는 다음과 같이 사용할 수 있습니다.
```kotlin
val aggregator = Aggregator<Int, DepartmentType, LocalDate, JoinedDates>()

// 데이터 추가
departments.forEach { a ->
    a.employers.forEach { b ->
        aggregator.add(
            key = b.id,
            value = a.departmentType to a.date
        )
    }
}

// List 결과를 가져온다
val joinedDates = aggregator.getList { id, joinedDate ->
    JoinedDates(
        employerId = id,
        departmentA = joinedDate.getValue(DepartmentType.A),
        departmentB = joinedDate.getValue(DepartmentType.B)
    )
}
```

범용성은 올라가지만, 호출부가 길어지고 타입 파라미터만 봐서는 의도가 바로 드러나지 않는다는 단점도 있습니다. 그래서 이런 형태로 일반화할 때는 KDoc이나 간단한 설명이 함께 있는 편이 좋습니다. 그래도 핵심은 결국 중간 `Map` 구조와 `compute()`를 이용한 갱신 패턴이므로, 그 부분만 이해해 두면 다른 상황에도 충분히 응용할 수 있습니다.
## 마지막으로

서버 사이드 Kotlin에서는 데이터를 `List`로 다루는 경우가 많지만, 이런 재구성 작업에서는 `Map`이 훨씬 잘 맞을 때도 있습니다. 특히 이번에 쓴 `compute()` 외에도 [getOrPut()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/get-or-put.html), [getOrDefault()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-map/get-or-default.html) 같은 함수는 비슷한 상황에서 꽤 유용합니다. 이 글과 비슷한 문제는 [이전 글](../exposed-mapping-record-to-object)에서도 한 번 다룬 적이 있으니, 관심이 있다면 함께 보면 좋겠습니다.

표준 라이브러리에는 평소에는 잘 눈에 띄지 않다가도, 막상 필요한 순간에 큰 도움이 되는 함수가 꽤 많습니다. 문서나 IDE 자동완성에 보이는 함수들을 그냥 지나치지 않고 한 번씩 살펴보면, 이런 식으로 문제를 푸는 실마리를 얻는 경우가 많습니다.
