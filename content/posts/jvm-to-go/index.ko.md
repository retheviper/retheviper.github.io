---
title: "JVM 언어 경험자가 Go를 건드릴 때의 함정"
date: 2022-04-17
categories:
  - go
image: "../../images/go.webp"
tags:
  - java
  - kotlin
  - go
translationKey: "posts/jvm-to-go"
---

Java와 Python, 그리고 조금의 JavaScript를 다루다가, 전직한 뒤 Go와 Kotlin을 본격적으로 만진 지 1년쯤 됐습니다. 요즘 언어는 서로 비슷해진 부분이 많아서 하나를 알면 다른 언어도 금방 익숙해질 것 같지만, 막상 써 보면 언어마다 생각보다 다른 점이 많습니다.

특히 예전에 익숙했던 언어의 습관을 그대로 들고 가면, 예상과 다른 결과를 만나기 쉽습니다. "이 언어에서는 이렇게 동작했으니 저 언어도 같겠지"라는 관성이나, "이건 아마 이런 사양일 것"이라는 추측이 문제를 만듭니다. 실제로 저도 그 함정에 꽤 여러 번 빠졌습니다.

그래서 이번에는 Java/Kotlin 배경의 엔지니어가 Go를 쓸 때 주의하면 좋을 만한 점을 몇 가지 적어 보겠습니다.

## 시간

Go에서는 시간을 다루는 표준 라이브러리로 `time` 패키지를 사용합니다.

```go
// 현재 시각을 얻기
now := time.Now()
// 특정 시각을 생성하기
someDay := time.Date(2022, 4, 1, 12, 30, 0, 0, time.UTC)
```

Java/Kotlin 쪽에서는 `java.time` 패키지가 대응합니다. Go와 가장 다른 점은, 시간을 더 세분화된 타입으로 나눠서 다룬다는 점입니다.

```java
// 연도
Year year = Year.now();
// 연월
YearMonth yearMonth = YearMonth.now();
// 연월일
LocalDate date = LocalDate.now();
// 시각
LocalDateTime time = LocalDateTime.now();
```

문제는 이렇게 타입이 다르다는 것만이 아닙니다. 같은 "한 달 전"을 계산해도 결과가 다를 수 있습니다.

```go
func getOneMonthBefore(t *time.Time) time.Time {
    // 한 달 전으로 이동
    return t.AddDate(0, -1, 0)
}

date := time.Date(2022, 3, 31, 0, 0, 0, 0, time.UTC)
oneMonthBefore := getOneMonthBefore(&date) // 2022-03-03
```

위 코드에서 기대한 값은 보통 `2022-02-28`일 텐데, 실제 결과는 `2022-03-03`이 됩니다. 이유는 Go의 `AddDate`가 존재하지 않는 날짜를 만나면 월말 보정 대신 날짜를 밀어 올리는 방식으로 계산하기 때문입니다.

반대로 Java/Kotlin의 `LocalDate`는 이런 경우 월말로 보정해 `2022-02-28`을 돌려줍니다. 그래서 JVM 언어의 습관으로 Go 코드를 작성하면 예상치 못한 결과를 만들 수 있습니다.

만약 Go에서도 월말 보정을 직접 하고 싶다면, 별도의 계산을 추가해서 처리해야 합니다.

```go
date := date.AddDate(2022, 3, 0, 0, 0, 0, 0, time.UTC) // 3월 0일은 2월 말일로 보정된다
```

## 맵

Go에서 변수 선언은 크게 두 가지 형태를 자주 봅니다.

```go
// 타입만 선언
var intSlice []int
// 초기화와 함께 선언
stringSlice := make([]string, 10)
```

문제는 선언 방식에 따라 값을 추가할 때 다르게 동작할 수 있다는 점입니다. slice는 `var`로 선언해도 `append`가 잘 동작합니다.

```go
var intSlice []int
// slice에 값 추가
intSlice = append(intSlice, 1) // [1]
```

하지만 map은 다릅니다. `var`로 선언만 해 둔 map은 nil이라서 바로 값을 넣을 수 없습니다.

```go
var stringMap map[string]string
stringMap["A"] = "a" // panic: assignment to entry in nil map
```

이 차이는 Go를 처음 접할 때 꽤 헷갈립니다. slice는 nil이어도 append가 되는데 map은 안 되기 때문입니다. Java나 Kotlin에서 `Map`을 생각하던 습관으로 보면 더 헷갈릴 수 있습니다.

## switch

Go의 `switch`는 모양은 Java와 비슷하지만 동작 방식은 Kotlin 쪽과 더 가깝습니다.

Java에서는 `break`를 직접 쓰지 않으면 아래 case로 계속 흘러갑니다.

```java
int i = 1;
switch (i) {
    case 0:
       System.out.println("zero");
    case 1:
       System.out.println("one");
    case 2:
       System.out.println("two");
    default:
       System.out.println("else");
}
```

실행 결과는 다음과 같습니다.

```text
one
two
else
```

Kotlin의 `when`은 기본적으로 한 분기만 실행하고 끝납니다.

```kotlin
val i = 1
when (i) {
    0 -> println("zero")
    1 -> println("one")
    2 -> println("two")
    else -> println("else")
}
```

결과는 다음과 같습니다.

```text
one
```

Go도 기본적으로는 Kotlin처럼 한 분기만 실행합니다.

```go
i := 1
switch i {
case 0:
    fmt.Println("zero")
case 1:
    fmt.Println("one")
case 2:
    fmt.Println("two")
default:
    fmt.Println("else")
}
```

즉, Java처럼 아래 case까지 계속 실행하고 싶다면 `fallthrough`를 직접 써야 합니다.

```go
i := 1
switch i {
case 0:
    fmt.Println("zero")
    fallthrough
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("two")
    fallthrough
default:
    fmt.Println("else")
}
```

Java 습관이 남아 있으면 Go에서도 자동으로 흘러갈 거라고 착각하기 쉬우니 주의가 필요합니다.

## if

Go는 `if` 조건이 논리적으로 이상하면 컴파일 단계에서 잡아 줍니다. 예를 들어 다음 코드를 보겠습니다.

```go
type Role int

const (
    SystemAdmin = 1
    Operator    = 2
    Developer   = 3
)

type User struct {
    Name string
    Role Role
}

// SystemAdmin이나 Developer가 아니면 에러를 돌려준다
func checkRunnableUser(u User) error {
    if u.Role != SystemAdmin && u.Role != Developer {
        return errors.New("user is not runnable")
    }
    return nil
}
```

이 코드는 `&&`를 써야 할 자리에 `||`를 쓰면 조건이 항상 참이 되어 버립니다. Go에서는 이런 실수를 컴파일러가 꽤 잘 잡아 줍니다.

Java에서는 이런 논리 오류를 런타임까지 놓치는 경우가 있는데, Go는 조건식 자체가 잘못되면 더 빨리 알려 주는 편입니다. 다만 JVM 언어에 익숙한 상태에서 Go를 쓰면, 조건식을 써 놓고도 "이상한데?"를 바로 못 느끼는 경우가 있습니다.

## Range loop

Go에는 전통적인 `for` 문 외에도 `range`를 이용한 반복이 있습니다.

```go
var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

for i, v := range pow {
    fmt.Printf("2**%d = %d\n", i, v)
}
```

Kotlin으로 쓰면 거의 같은 모양이 됩니다.

```kotlin
val pow = listOf(1, 2, 4, 8, 16, 32, 64, 128)

for ((i, v) in pow.withIndex()) {
    println("2**$i = $v")
}
```

그런데 Go에서 포인터를 함께 쓰면 주의할 점이 있습니다.

```go
var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

// 새 slice에 pow의 참조를 저장
var ppow []*int
for _, v := range pow {
    ppow = append(ppow, &v)
}

// ppow의 값을 출력
for i, v := range ppow {
    fmt.Printf("2**%d = %d\n", i, *v)
}
```

이 코드는 기대와 달리 모든 포인터가 같은 값을 가리키게 됩니다. `range` 루프에서 반복 변수 `v`는 매 회차 재사용되기 때문에, 주소를 그대로 저장하면 같은 메모리를 참조하게 되기 때문입니다.

이 문제를 피하려면 루프 안에서 값을 다시 복사하거나, 인덱스를 기준으로 직접 참조해야 합니다.

```go
var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

var ppow []*int
for _, v := range pow {
    v := v
    ppow = append(ppow, &v)
}

for i, v := range ppow {
    fmt.Printf("2**%d = %d\n", i, *v)
}
```

## 마지막으로

몇 가지 예를 들었지만, 저도 아직 Go를 깊게 안다고 하긴 어렵습니다. 앞으로도 이런 종류의 실수를 더 만나게 될 가능성은 충분합니다.

그래도 다른 언어를 배울 때는, 익숙한 언어의 습관을 그대로 가져오지 않는 것이 중요하다고 느꼈습니다. 배경 지식은 분명 도움이 되지만, 동시에 편견이 되기도 합니다. Go에만 해당하는 이야기는 아니고, 새로운 언어를 만질 때는 늘 같은 함정을 조심해야 한다는 뜻입니다.
