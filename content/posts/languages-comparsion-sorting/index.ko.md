---
title: "다양한 언어로 정렬해 보기"
date: 2021-11-10
categories:
  - languages
image: "../../images/magic.webp"
tags:
  - kotlin
  - java
  - javascript
  - python
  - swift
  - go
translationKey: "posts/languages-comparsion-sorting"
---

요즘은 어떤 프로그래밍 언어를 선택해도 할 수 있는 일의 차이가 예전보다 크지 않습니다. 특히 Kotlin/JS나 Flutter처럼 여러 플랫폼을 한 번에 노리는 도구가 많아지면서, 언어 간 경계도 점점 옅어지고 있습니다.

그렇다고 해서 언어마다 차이가 없는 것은 아닙니다. 실제 업무에서는 사용 언어가 정해져 있는 경우가 많고, 익숙하지 않은 언어를 갑자기 읽거나 수정해야 할 일도 생깁니다. 그래서 여러 언어의 특징을 알고 있으면 꽤 도움이 됩니다. 다른 언어의 관점을 알면, 오히려 주로 쓰는 언어를 더 잘 이해하게 되는 경우도 많습니다.

이번 글에서는 그런 비교의 첫 주제로 "정렬"을 골랐습니다.

## JavaScript

JavaScript에서는 `Array.prototype.sort()`로 배열을 정렬할 수 있습니다.

```javascript
const a = [22, 1, 44, 300, 5000]
a.sort()
console.log(a)
// [1, 22, 300, 44, 5000]
```

문제는 숫자 배열도 기본적으로 문자열 기준으로 비교한다는 점입니다. 그래서 기대한 순서가 나오지 않을 수 있습니다.

```javascript
const a = [22, 1, 44, 300, 5000]
a.sort((a, b) => a - b)
console.log(a)
// [1, 22, 44, 300, 5000]
```

`sort()`에는 비교 함수를 넘길 수 있고, `0`보다 작으면 앞에, `0`이면 그대로, `0`보다 크면 뒤에 배치됩니다. Java의 `Comparator`와 거의 같은 느낌입니다.

원본 배열을 건드리지 않고 새 배열이 필요하다면 복사한 뒤 정렬하면 됩니다.

```javascript
const a = [22, 1, 44, 300, 5000]
const b = [...a].sort((a, b) => a - b)
console.log(b)
// [1, 22, 44, 300, 5000]
```

역순이 필요하면 `reverse()`를 쓰면 됩니다.

```javascript
const a = [22, 1, 44, 300, 5000]
a.reverse()
console.log(a)
// [5000, 300, 44, 1, 22]
```

## Java

Java에서는 `Comparator`를 중심으로 정렬 방법을 정합니다. 컬렉션과 배열에 따라 선택지도 많습니다.

가장 기본적인 방법은 `Collections.sort()`나 `Arrays.sort()`를 사용하는 것입니다.

```java
var a = new ArrayList<Integer>() {{ add(22); add(1); add(44); add(300); add(5000); }};
Collections.sort(a);
System.out.println(a);
// [1, 22, 44, 300, 5000]
```

`List.sort()`를 쓰면 비교 함수를 직접 넘길 수 있습니다.

```java
var a = new ArrayList<Integer>() {{ add(22); add(1); add(44); add(300); add(5000); }};
a.sort(Comparator.naturalOrder());
System.out.println(a);
// [1, 22, 44, 300, 5000]
```

내림차순은 `reverseOrder()`를 쓰면 됩니다.

```java
var a = new ArrayList<Integer>() {{ add(22); add(1); add(44); add(300); add(5000); }};
Collections.sort(a, Comparator.reverseOrder());
System.out.println(a);
// [5000, 300, 44, 22, 1]
```

원본을 바꾸지 않고 새 결과를 얻고 싶다면 `Stream.sorted()`를 사용할 수도 있습니다.

```java
var a = List.of(22, 1, 44, 300, 5000);
var b = a.stream().sorted().collect(Collectors.toList());
System.out.println(b);
// [1, 22, 44, 300, 5000]
```

DTO처럼 복잡한 객체를 정렬할 때는 `Comparable`보다 `Comparator`가 더 자주 쓰입니다. 정렬 기준이 코드에 분명하게 드러나고, 조건이 바뀌어도 클래스 자체를 고치지 않아도 되기 때문입니다.

## Kotlin

Kotlin은 정렬 관련 API가 꽤 다양합니다. 원본을 바꾸는지, 새 컬렉션을 만드는지, 그리고 정렬 기준이 자연 정렬인지 사용자 정의인지로 나눠 보면 이해가 쉽습니다.

| 종류 | 결과 | 함수 | 비고 |
|---|---|---|---|
| Natural | 호출한 컬렉션 | `sort()` | 오름차순 |
|  |  | `sortDescending()` | 내림차순 |
|  |  | `reverse()` | 순서 반전 |
|  | Array | `sortedArray()` | 오름차순 |
|  |  | `sortedArrayDescending()` | 내림차순 |
|  |  | `reversedArray()` | 순서 반전 |
|  | List | `sorted()` | 오름차순 |
|  |  | `sortedDescending()` | 내림차순 |
|  |  | `asReversed()` | 순서 반전 |
| Custom | 호출한 컬렉션 | `sortBy()` | selector 필요 |
|  |  | `sortByDescending()` | selector 필요 |
|  | List | `sortedBy()` | selector 필요 |
|  |  | `sortedByDescending()` | selector 필요 |
|  | Array | `sortedArrayWith()` | `Comparator` 필요 |
|  | List | `sortedWith()` | `Comparator` 필요 |

예를 들어 새 `List`를 만들고 싶다면 다음처럼 쓰면 됩니다.

```kotlin
val a = listOf(22, 1, 44, 300, 5000)
val b = a.sorted()
val c = a.sortedDescending()

println(b)
// [1, 22, 44, 300, 5000]
println(c)
// [5000, 300, 44, 22, 1]
```

객체 리스트를 정렬할 때는 `sortedBy`나 `sortedByDescending`이 편합니다.

```kotlin
data class Data(val number: Int)

val a = listOf(Data(22), Data(1), Data(44), Data(300), Data(5000))
val b = a.sortedBy { it.number }
val c = a.sortedByDescending { it.number }
```

복잡한 기준이 필요하면 Java처럼 `Comparator`를 직접 쓰면 됩니다.

## Swift

Swift는 원본을 정렬할지, 새 컬렉션을 만들지에 따라 API가 나뉩니다.

```swift
var a = [22, 1, 44, 300, 5000]
a.sort()
print(a)
// [1, 22, 44, 300, 5000]
```

새 배열이 필요하면 `sorted()`를 쓰면 됩니다.

```swift
let a = [22, 1, 44, 300, 5000]
let b = a.sorted()
print(b)
// [1, 22, 44, 300, 5000]
```

Swift의 특징은 정렬 기준을 Boolean 반환 클로저로 넘긴다는 점입니다. JavaScript나 Java/Kotlin처럼 숫자 비교 결과를 돌려주는 방식과는 조금 다릅니다.

```swift
let students: Set = ["Kofi", "Abena", "Peter", "Kweku", "Akosua"]
let descendingStudents = students.sorted(by: >)
print(descendingStudents)
// ["Peter", "Kweku", "Kofi", "Akosua", "Abena"]
```

구조체 필드를 기준으로 정렬할 수도 있습니다.

```swift
struct Data { var number = 0 }

let datas = [Data(number: 1), Data(number: 3), Data(number: 4), Data(number: 2)]
let descending = datas.sorted { $0.number > $1.number }
```

## Go

Go에는 제네릭이 없어서인지 `sort` 패키지에 자료형별 함수가 따로 준비되어 있습니다.

- `sort.Float64s`
- `sort.Ints`
- `sort.Strings`

정수 배열은 다음처럼 정렬할 수 있습니다.

```go
a := []int{22, 1, 44, 300, 5000}
sort.Ints(a)
fmt.Println(a) // [1 22 44 300 5000]
```

구조체 슬라이스는 `sort.Slice`를 사용합니다.

```go
people := []struct {
  Name string
  Age  int
}{
  {"Gopher", 7},
  {"Alice", 55},
  {"Vera", 24},
  {"Bob", 75},
}
sort.Slice(people, func(i, j int) bool { return people[i].Name < people[j].Name })
fmt.Println(people)
```

Go에는 안정 정렬용으로 `sort.SliceStable()`도 따로 있습니다. 같은 값끼리의 원래 순서를 유지해야 한다면 이쪽을 사용하면 됩니다.

```go
people := []struct {
  Name string
  Age  int
}{
  {"Alice", 25},
  {"Elizabeth", 75},
  {"Alice", 75},
  {"Bob", 75},
  {"Alice", 75},
  {"Bob", 25},
  {"Colin", 25},
  {"Elizabeth", 25},
}

sort.SliceStable(people, func(i, j int) bool { return people[i].Age < people[j].Age })
```

## Python

Python에서는 `list.sort()`와 `sorted()`를 자주 사용합니다. 전자는 원본을 바꾸고, 후자는 새 리스트를 만듭니다.

```python
a = [22, 1, 44, 300, 5000]
a.sort()
print(a)
# [1, 22, 44, 300, 5000]
```

```python
a = [22, 1, 44, 300, 5000]
b = sorted(a)
print(b)
# [1, 22, 44, 300, 5000]
```

`key`와 `reverse`를 지정하면 정렬 기준도 쉽게 바꿀 수 있습니다.

```python
class Data:
    def __init__(self, number):
        self.number = number

    def __repr__(self):
        return repr(self.number)


datas = [Data(1), Data(3), Data(2), Data(4)]
datas.sort(key=lambda data: data.number)
sorted(datas, key=lambda data: data.number, reverse=True)
```

## 번외: Stable sort

여기서 비교한 언어들이 안정 정렬을 어떻게 다루는지도 같이 정리해 봤습니다.

| 언어 | stable | non-stable | 비고 |
|---|---|---|---|
| Go | O | O | 함수에 따라 선택 가능 |
| Java | O | O | Stream은 non-stable일 수 있음 |
| JavaScript | O | O | 브라우저 버전에 따라 다름 |
| Python | O | X | |
| Kotlin | O | X | Sequence도 안정 정렬 |
| Swift | X | O | stable을 보장하지 않음 |

대부분의 언어가 안정 정렬을 지원하지만, 세부 구현은 조금씩 다릅니다. Java는 `Stream` 쪽에서 주의가 필요하고, JavaScript는 브라우저 버전에 따라 차이가 있을 수 있습니다. Swift는 안정 정렬을 보장하지 않는 편입니다. 반대로 Python과 Kotlin은 안정 정렬을 기본으로 생각해도 됩니다.

## 마지막으로

이번에는 여러 언어의 정렬 API를 비교해 봤습니다. 정렬은 단순한 기능처럼 보이지만, 언어마다 설계 방향과 표준 라이브러리 철학이 꽤 잘 드러납니다.

이런 비교를 해 보면 각 언어의 API가 왜 그런 모양인지 조금 더 이해할 수 있어서 재미있습니다. 다음에도 같은 방식으로 비슷한 기능을 여러 언어에서 나란히 비교해 볼 생각입니다.
