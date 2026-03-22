---
title: "Reverse grouping of data in Kotlin"
date: 2022-01-29
translationKey: "posts/kotlin-reverse-groupping"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
---
There are cases where the way data is structured in the database differs greatly from the way it is ultimately used in the application. This is especially true when features are added later. Of course, some of this comes from the difference between databases, which need to think about storing data efficiently through normalization and similar techniques, and apps, which need to think about how to process and use data. But I also think it happens because, as apps continue to evolve, the places where the same data is used and the way it is expressed also change.

That's why I would like to introduce a method that can be used in such cases. It's not an algorithm, and I'm sure there are more efficient methods, but I think it can be used in many different places if applied.

## Scenario

For example, suppose we have the following scenario:

1. Employees are assigned to two departments, A and B.
2. The dates on which an employee is assigned to each department are different.

In this case, there may be various points of view when creating data, but if you were to create data based on the date of assignment to a department, you would probably have a list of the type of department, the date of assignment, and the employees who were assigned on that date. If you were to express it as Kotlin code, it would look like this:

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

Let's assume there are three employees, each assigned to department A and department B on different dates. The data is as follows.

| Employee number | Assigned to department A | Assigned to department B |
|---|---|---|
| 1 | January 1st | January 1st |
| 2 | January 1st | February 1st |
| 3 | February 1st | February 1st |

If you take the above data and create an actual list of the department data, it would look like the following.

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

However, how can I process this data based on employees and the date they were assigned to each department? It has two parts: the employee number and the date the employee was assigned to the department. For example, expressed in code as follows.

```kotlin
data class JoinedDates(
    val employerId: Int,
    val departmentA: LocalDate,
    val departmentB: LocalDate
)
```

In other words, what we want to do is to convert the `departments` from earlier into the following data.

```kotlin
[
  JoinedDates(employerId=1, departmentA=2022-01-01, departmentB=2022-01-01), 
  JoinedDates(employerId=2, departmentA=2022-01-01, departmentB=2022-02-01), 
  JoinedDates(employerId=3, departmentA=2022-02-01, departmentB=2022-02-01)
]
```

This is a situation where the standards for data alignment are turned upside down, making it difficult to know what to do. This time, I would like to introduce my method to solve this problem.

## Logic

If we consider Department as the standard, there will be multiple Employer data, but our requirement is to reverse this and create multiple Departments based on Employer. If so, I think the key points of possible logic are as follows.

1. Group by Employer ID
1. For each Employer, have an array that divides Department by Type.

First, you need to go into the list of Employers you are nesting and extract their IDs. You don't want to duplicate this ID, so it's a good idea to use it as the Map Key.

After that, it would be a good idea to perform the following processing depending on whether the Employer ID exists as a key in that Map.

1. If Key does not exist, insert the new Department type and its date.
2. If Key exists, extract its value and add Department type and date

Therefore, you will need to convert the Department List to a Map and then convert it to a JoinedDates List. The above branch can be achieved by using [compute()](https://docs.oracle.com/javase/8/docs/api/java/util/Map.html#compute-K-java.util.function.BiFunction-), so I think it would be a good idea to think about the shape of the Map as intermediate data.

In my case, Map is easier to retrieve data, so the final process was as follows.

```kotlin
fun List<Department>.toJoinedDates(): List<JoinedDates> {
    // Intermediate data
    val tempMap = mutableMapOf<Int, Map<DepartmentType, LocalDate>>()

    this.forEach { department ->
        // Pair of Department type and its date
        val departmentJoined = department.departmentType to department.date

        department.employers.forEach { employer ->

            // If the Employer ID exists as a key, add to it; otherwise create a new Map entry
            tempMap.compute(employer.id) { _, value ->
                value?.let { value + departmentJoined } ?: mapOf(departmentJoined)
            }
        }
    }

    // Convert the intermediate data into a List of JoinedDates and return it
    return tempMap.map { (id, department) ->
        JoinedDates(
            employerId = id,
            departmentA = department.getValue(DepartmentType.A),
            departmentB = department.getValue(DepartmentType.B)
        )
    }
}
```

## Common logic

I thought that if I separated part of the above logic as a class using generics, I could reuse it in various similar cases, so I wrote the code below.

```kotlin
class Aggregator<T, K, V, R> {
    private val tempMap = mutableMapOf<T, Map<K, V>>()

    // Add data
    fun add(key: T, value: Pair<K, V>) {
        tempMap.compute(key) { _, existingValue ->
            existingValue?.let { existingValue + value } ?: mapOf(value)
        }
    }

    // Retrieve as a specified List
    fun getList(transfer: (T, Map<K, V>) -> R): List<R> {
        return tempMap.map { transfer(it.key, it.value) }
    }
}
```

In this case, you can use it as follows.

```kotlin
val aggregator = Aggregator<Int, DepartmentType, LocalDate, JoinedDates>()

// Add data
departments.forEach { a ->
    a.employers.forEach { b ->
        aggregator.add(
            key = b.id,
            value = a.departmentType to a.date
        )
    }
}

// Get the result as a List
val joinedDates = aggregator.getList { id, joinedDate ->
    JoinedDates(
        employerId = id,
        departmentA = joinedDate.getValue(DepartmentType.A),
        departmentB = joinedDate.getValue(DepartmentType.B)
    )
}
```

Although it is versatile, the disadvantages may be that it increases the number of calling code and that it is difficult to understand without proper KDoc or comments because the meaning and intention of the specified type is not clear. However, the important thing is the type of intermediate data and the branching process using `compute()`, so I think it would be possible to extract that part well and apply it elsewhere.

## Finally

When using Kotlin on the server side, I think it is normal to treat data as `List` in most cases, but in some cases using `Map` may be a good choice when writing logic. In particular, in addition to the `compute()` introduced this time, functions such as [getOrPut()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/get-or-put.html) and [getOrDefault()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-map/get-or-default.html) are useful and can be used in many situations. I have introduced something similar to this process in [previous post](../exposed-mapping-record-to-object/), so if you are interested, please refer to that as well.

I think there are many aspects of standard libraries provided by programming languages ​​that are easy to overlook, but if you pay attention to the functions that often appear in the documentation or auto-completion lists, you may find that what you need suddenly appears like this. I'm still a novice who has only been using Kotlin for about a year, so I'd be happy if I continue to make new discoveries.

See you soon!
