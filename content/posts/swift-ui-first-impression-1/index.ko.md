---
title: "SwiftUI를 써 본 첫인상 1"
date: 2022-07-31
categories: 
  - swift
image: "../../images/swift.webp"
tags:
  - swift
  - swiftui
  - gui
  - kotlin
  - compose
translationKey: "posts/swift-ui-first-impression-1"
---

지금까지의 커리어를 돌아보면 업무 경험은 거의 백엔드에만 몰려 있고, 화면 쪽 구현에는 거의 관여하지 않았습니다. 하지만 독립형 앱을 만들려면 웹, 모바일, 데스크톱을 가리지 않고 결국 화면이 필요합니다. 언젠가는 화면 쪽도 직접 다룰 수 있어야 하지 않을까 하고 늘 생각해 왔습니다.

화면을 만든다고 해도, 어떤 분야의 엔지니어로 커리어를 쌓을지, 어떤 회사에서 일하고 싶은지, 익숙한 언어가 무엇인지 등 따져 볼 요소는 꽤 많습니다. 저는 Kotlin에 익숙한 데다 웹, 모바일, 데스크톱을 모두 다룰 수 있다는 점에서 [Jetpack Compose](https://www.jetbrains.com/lp/compose-mpp)에 끌렸고, 평소 Mac과 iPhone, iPad를 자주 쓰는 편이라 Kotlin 경험을 살려 Swift에 들어가 보기 좋겠다는 생각에서 [SwiftUI](https://developer.apple.com/xcode/swiftui/)도 함께 공부해 보고 싶었습니다.

언어와 프레임워크를 정했다면 이제 직접 만져 볼 차례입니다. [공식 튜토리얼](https://developer.apple.com/tutorials/swiftui)이 꽤 잘 갖춰져 있어서, 우선 그 흐름을 따라가며 Swift와 SwiftUI에서 인상적이었던 점을 정리해 보려고 합니다. 모바일 앱 구현을 업무로 해 본 적은 없어서 내용이 다소 거칠 수는 있지만, 저처럼 Kotlin 배경에서 Swift를 시작해 보려는 분이나 백엔드만 하다가 GUI를 처음 접하는 분, 혹은 Kotlin과 Swift 중 어느 쪽에 먼저 관심을 가져야 할지 고민하는 분께는 참고가 되면 좋겠습니다.
## Swift

우선은 언어 자체부터 보겠습니다. Kotlin과 Swift가 비슷하다는 이야기는 자주 듣지만, 실제로 어디가 비슷한지는 직접 만져 보기 전까지는 잘 와닿지 않습니다. 비슷하다는 말은 결국 공통점이 있다는 뜻인데, 무엇을 기준으로 보느냐에 따라 떠오르는 공통점도 달라지기 때문입니다.

예를 들면 언어 설계 측면에서 둘 다 객체지향적이고 함수형 요소를 갖고 있으며, GC가 있다는 점을 들 수 있습니다. 또는 문법이나 키워드의 감각이 비슷하다고 말할 수도 있습니다. 세세하게 보면 세미콜론을 굳이 쓰지 않아도 된다는 점도 공통점으로 꼽을 수 있겠네요.

그래서 이 글에서는 공식 튜토리얼을 따라가며, 제가 직접 느낀 감각을 바탕으로 Kotlin에 비해 Swift가 어떤 언어였는지 정리해 보려고 합니다.
### Kotlin과 비슷한 것

그럼 먼저 Kotlin과 닮았다고 느낀 부분부터 보겠습니다. 물론 어디까지나 체감에 기반한 이야기라, 엄밀히 따지면 사양이 다른 부분도 많습니다. 여기서의 기준은 "Kotlin에서 하던 일을 얼마나 비슷한 감각으로 옮길 수 있느냐" 정도라고 봐 주시면 됩니다. 개인적인 인상이라는 점도 함께요.
#### Computed properties

첫째, 속성 이야기에서입니다. Kotlin에서는 data class를 정의할 때 다음 두 가지 방법으로 속성을 정의할 수 있습니다.
```kotlin
data class Student(
    val age: Int
) {
    val isAdult: Boolean
        get() = age >= 18
}
```

여기서 `age`는 인스턴스를 만들 때 정해지는 단순한 값이고, `isAdult`는 `age`가 18 이상인지 계산한 결과를 반환하는 프로퍼티입니다. 이런 계산형 프로퍼티는 Swift에서도 [Computed Properties](https://docs.swift.org/swift-book/LanguageGuide/Properties.html#ID259)로 정의할 수 있었습니다. 형태는 대략 다음과 같습니다.
```swift
struct Student {
    var age: Int

    var isAdult: Bool {
        get { return age >= 18 }
    }
}
```

이 정도만 봐도 "Kotlin과 Swift가 비슷하다"는 말이 왜 나오는지는 감이 옵니다. 계산이 필요한 프로퍼티를 다루는 방식도 그렇고, 타입을 정의하는 방식도 그렇고, 키워드는 조금 달라도 전체적인 읽는 감각이 꽤 비슷합니다.

다만 차이점도 분명합니다. Kotlin의 `data class`에 대응해 Swift는 Go나 Rust처럼 `struct`를 자연스럽게 사용합니다. 물론 Swift에도 `class`가 있으니 목적에 따라 골라 쓰게 됩니다. 이 점은 Kotlin에서 `data class`와 `class`를 구분해 쓰는 감각과도 어딘가 닮아 있습니다.
#### 확장

다음은 확장입니다. Kotlin에서는 객체 바깥에 메서드나 프로퍼티를 정의할 수 있습니다. 이것을 확장 함수, 확장 프로퍼티라고 부르며 다음처럼 쓸 수 있습니다.
```kotlin
val Student.isUnderAge: Boolean
    get() = age < 18
```

예전에 이 블로그에서도 언급한 [Effective Kotlin](https://www.amazon.co.jp/dp/B08WXCRVD2/ref=dp-kindle-redirect?_encoding=UTF8&btkr=1)에서 소개한 활용법인데, 유스케이스나 도메인에 따라 다른 처리가 필요할 때는 클래스 안에 모든 메서드와 프로퍼티를 넣기보다 이런 확장을 별도 패키지에 두어 접근 범위를 나누는 편이 낫습니다.

Swift에서도 비슷한 일을 할 수는 있지만, 쓰는 방식은 조금 다릅니다. 같은 프로퍼티를 Swift로 옮기면 다음과 같습니다.
```swift
extension Student {
  var isUnderAge: Bool {
    get { return age < 18 }
  }
}
```

Swift에는 [Extension](https://docs.swift.org/swift-book/LanguageGuide/Extensions.html)이라는 별도 키워드가 있어서, 마치 새로 `class`나 `struct`를 정의하는 것처럼 함수와 프로퍼티를 덧붙일 수 있습니다. 개인적으로는 Rust의 [메서드](https://doc.rust-jp.rs/book-ja/ch05-03-method-syntax.html)와도 비슷한 느낌이었고, 하나의 `extension` 안에 모아 둘 수 있어서 Kotlin보다 더 정돈된 인상을 받았습니다. Kotlin은 같은 객체에 대한 확장이 여러 개로 흩어지면 조금 지저분해 보일 때가 있거든요.
#### String Interpolation

Java를 포함한 많은 언어에서는 문자열 안에 다른 값을 넣으려면 `format()`을 쓰거나 문자열로 변환해 이어 붙이는 경우가 많습니다. Kotlin도 물론 그렇게 할 수 있지만, [String template](https://kotlinlang.org/docs/basic-types.html#string-templates)이 있어서 문자열 안에 다른 값을 훨씬 편하게 넣을 수 있습니다. 예를 들면 다음과 같습니다.
```kotlin
val world = "World"
println("Hello, $world")
```

Swift에도 [String Interpolation](https://docs.swift.org/swift-book/LanguageGuide/StringsAndCharacters.html#ID292)이 있어서 같은 일을 할 수 있습니다. 표기 방식은 조금 다르지만 기능적으로는 거의 같습니다.
```swift
let world = "World"
print("Hello, \(world)!")
```

#### Arguments

Kotlin에서는 함수 매개변수에 기본값을 둘 수 있어서, 오버로드를 쉽게 흉내 낼 수 있고 매개변수가 빠진 경우도 자연스럽게 처리할 수 있습니다.
```kotlin
// times에 지정한 횟수만큼 string을 출력한다
fun printHello(string: String, times: Int = 1) {
    repeat(times) {
        println("Hello, $string")
    }
}

printHello("world") // times를 지정하지 않아도 함수를 호출할 수 있다
```

또, 어떤 매개변수에 값을 넣는지 분명히 하고 싶거나, 정의된 순서와 상관없이 값을 넘기고 싶을 때는 [Named Arguments](https://kotlinlang.org/docs/functions.html#named-arguments)를 사용할 수 있습니다. 예를 들어 [joinToString()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/join-to-string.html)에는 `separator`, `limit`, `truncated` 등 여러 매개변수가 있지만 기본값이 있어서 Named Arguments로 다음처럼 쓸 수 있습니다.
```kotlin
listOf("A", "B", "C", "D").joinToString(prefix = "[", postfix = "]")
```

Kotlin에서 Named Argument는 선택 사항입니다. Java처럼 함수 정의 순서대로 값을 넘겨도 문제없습니다. 반대로 Swift에서는 이 부분이 기본 동작에 가깝습니다. `struct`의 인스턴스를 만들거나 함수를 호출할 때, 기본적으로 이름을 붙여서 전달해야 합니다. Swift에서는 이것을 [Argument Label](https://docs.swift.org/swift-book/LanguageGuide/Functions.html#ID166)이라고 부릅니다.
```swift
func printHello(string: String) {
    print("Hello \(string)!")
}

printHello(string: "world") // Function Argument Label로 string을 지정한다
```

하지만 이것도 Kotlin처럼 기본값을 지정할 수 있어서, 그 경우에는 매개변수를 생략할 수 있습니다.
```swift
func printHello(string: String, times: Int = 1) {
    var count = 1
    repeat { // Kotlin의 do-while과 비슷한 것
        print("Hello \(string)!")
        count += 1
    } while (count <= times)
}

printHello(string: "world") // times를 생략했다
```

또 밑줄을 쓰면 Argument Label을 생략할 수도 있습니다.
```swift
func printHello(_ string: String, times: Int = 1) {
    var count = 1
    repeat {
        print("Hello \(string)!")
        count += 1
    } while (count <= times)
}

printHello("world") // string을 생략했다
```

정의하는 쪽에서 보면 그다지 비슷하지 않아 보이지만, 호출하는 쪽에서는 꽤 비슷한 감각으로 코드를 쓸 수 있다는 점이 흥미롭습니다.
#### Range

Kotlin에서는 [rangeTo()](https://kotlinlang.org/docs/ranges.html#:~:text=values%20using%20the-,rangeTo(),-function%20from%20the)를 써서 숫자 범위를 쉽게 정의할 수 있습니다. 이 함수는 [operator](https://kotlinlang.org/docs/keyword-reference.html#operators-and-special-symbols)로 정의되어 있어서 `..`로 자연스럽게 사용할 수 있습니다. 이렇게 만든 Range는 최소값과 최대값을 확인하거나 List로 바꾸는 등 여러 용도로 쓸 수 있습니다.
```kotlin
// Range 정의
val min = 10
val max = 20
val range = min..max

// 최소값과 최대값을 얻는다
println(range.start) // 10
println(range.endInclusive) // 20

// Range를 List로 바꾼다
val intList = range.toList() // [10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
```

Swift에서도 [Range Operator](https://docs.swift.org/swift-book/LanguageGuide/BasicOperators.html#ID73)로 범위를 정의할 수 있습니다. 형태는 `...`로 비슷하고, 점 하나가 더 많다는 점만 다릅니다. 최소값과 최대값도 이름만 다를 뿐 프로퍼티처럼 얻을 수 있습니다.
```swift
// Range 정의
let min = 10
let max = 20
let range = min...max

// 최소값과 최대값을 얻는다
print(range.lowerBound) // 10
print(range.upperBound) // 20

// Range를 Array로 바꾼다
let array = Array(range) // [10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
```

다만, 위 코드만 보면 잘 드러나지 않지만 Range 구현은 두 언어에서 조금 다릅니다. Kotlin에서는 `rangeTo()`의 반환값이 값의 타입에 맞춰 `IntRange`나 `LongRange`처럼 결정되고, 최소값과 최대값도 그 타입을 그대로 따라갑니다.

반면 여기서 사용한 `min...max`는 Swift에서는 [ClosedRange](https://developer.apple.com/documentation/swift/closedrange)가 됩니다. 이 예제에서는 `lowerBound`, `upperBound`도 결국 `Int`지만, Kotlin의 `IntRange`처럼 전용 타입 이름을 두기보다 Swift는 `ClosedRange<Int>` 같은 제네릭 타입으로 표현합니다.
### Swift만의 것

지금까지는 Kotlin 사용자 입장에서 Swift를 얼마나 익숙한 감각으로 다룰 수 있는지에 초점을 맞췄습니다. 여기부터는 조금 다르다고 느낀 부분을 정리해 보겠습니다.
#### 메서드 속성 호출에서 생략

Kotlin에서는 [apply()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html)처럼 대상이 분명할 때 `this`를 생략할 수 있습니다. 예시는 다음과 같습니다.
```kotlin
data class Student(
    val name: String,
    var age: Int = 0
)

val studentA = Student(name = "A").apply { age = 18 } // Student(name=A, age=18)
```

이런 식으로 `this`를 쓰는 경우나 대상 import가 분명한 몇몇 경우를 빼면, Kotlin은 기본적으로 `Class.method()`처럼 어느 클래스의 멤버를 호출하는지 드러내는 편입니다.

Swift는 여기서 조금 더 느슨합니다. 컴파일러가 문맥을 보고 대상을 분명히 알 수 있으면 `.method()`처럼 앞부분을 생략할 수 있습니다. 아래는 SwiftUI 튜토리얼의 예시 일부인데, `filter`가 `FilterCategory` enum이기 때문에 삼항 연산자 안에서 `.all`처럼 쓰이고 있습니다.
```swift
struct LandmarkList: View {
    @State private var filter = FilterCategory.all

    enum FilterCategory: String, CaseIterable, Identifiable {
        case all = "ALL"
        case lakes = "Lakes"
        case rivers = "Rivers"
        case mountains = "Mountains"
        
        var id: FilterCategory { self }
    }

    var title: String {
        let title = filter == .all ? "Landmarks" : filter.rawValue
        return showFavoritesOnly ? "Favorite \(title)" : title
    }
}
```

#### Protocol

Swift에는 [Protocol](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html)이 있고, Java나 Kotlin의 인터페이스와 거의 같은 감각으로 사용할 수 있습니다. 여기까지만 보면 큰 차이가 없어 보이지만, 실제로 `struct`, `class`, `enum`을 정의할 때는 필요에 따라 protocol을 채택(adopt)해야 한다는 점이 체감되는 차이였습니다.

예를 들어 Kotlin에서 `data class` 하나를 정의하면 다음과 같은 멤버가 자동으로 붙습니다.
- equals()
- hashCode()
- toString()
- componentN()
- copy()

반면 Swift의 `struct`, `class`, `enum`에는 이런 멤버가 기본으로 붙지 않습니다. 그래서 필요한 기능이 있으면 그에 맞는 protocol을 채택하고 직접 구현해야 합니다. 예를 들어 해시값이 필요하면 [Hashable](https://developer.apple.com/documentation/swift/hashable), JSON 변환이 필요하면 [Codable](https://developer.apple.com/documentation/swift/codable), 리스트 순회가 필요하면 [Identifiable](https://developer.apple.com/documentation/swift/identifiable), enum의 모든 case를 순회하고 싶으면 [CaseIterable](https://developer.apple.com/documentation/swift/caseiterable), 같음을 비교하려면 [Equatable](https://developer.apple.com/documentation/swift/equatable)를 쓰는 식입니다.

Java나 Kotlin에서도 필요에 따라 interface나 annotation을 써야 하긴 하지만, Swift는 Kotlin에서 당연하게 쓰던 기능이 `struct`나 `class`를 정의한 시점에는 아직 준비되어 있지 않을 수 있어서 이 부분을 의식하고 있어야 합니다.
#### some

Swift에는 조금 독특하게 느껴지는 키워드가 하나 있습니다. 설명을 위해 먼저 다음과 같은 protocol과 struct가 있다고 해 보겠습니다.
```swift
protocol Something {
    func define()
}

struct GoodThing: Something {
    func define() {
        print("It's good thing!")
    }
}
```

이런 코드가 있으면 변수 타입 선언이나 함수 반환값에 조금 독특한 키워드인 `some`을 쓸 수 있습니다. 실제 사용 예는 다음과 같습니다.
```swift
var good: some Something = GoodThing()

func returnSomething() -> some Something {
    return GoodThing()
}
```

이것만 보면 `some`이 정확히 무슨 뜻인지 바로 감이 오지는 않습니다. Kotlin 쪽 개념으로 옮기면 `<T extends Something>`과 비슷하다고 볼 수 있습니다. Java나 Kotlin을 써 본 사람이라면 느낌을 쉽게 잡을 수 있을 겁니다.

즉 `some`은 특정 protocol을 만족하는 어떤 인스턴스를 가리킵니다. Swift에서는 protocol을 만족하는 객체라도 타입을 직접 protocol로 드러내지 못하는 경우가 있는데, 그럴 때 `some`을 써서 문제를 피해 갑니다. Java나 Kotlin에서 interface를 구체 구현과 분리해서 다루는 느낌과도 비슷합니다. 덕분에 SwiftUI에서는 [View](https://developer.apple.com/documentation/swiftui/view)를 만족하는 값을 화면 구성 요소로 자연스럽게 다룰 수 있습니다.

다만 개념적으로 interface와 비슷하다고 해도 실제로 코드를 쓰는 감각은 꽤 다르니, 이 부분은 주의가 필요합니다.
#### Compiler Control Statements

Swift에는 [Compiler Control Statements](https://docs.swift.org/swift-book/ReferenceManual/Statements.html#ID538)라는 문법이 있어서 컴파일 시점의 동작을 조건부로 바꿀 수 있습니다. SwiftUI 튜토리얼에서도 OS별로 다른 기능을 넣기 위해 이 기능을 사용합니다. 예시는 다음과 같습니다.
```swift
// watchOS에서 실행할 때는 알림을 사용한다
#if os(watchOS)
WKNotificationScene(controller: NotificationController.self, category: "LandmarkNear")
#endif

// macOS에서 실행할 때는 설정을 사용한다
#if os(macOS)
Settings {
    LandmarkSettings()
}
#endif
```

Kotlin도 Android 쪽에서는 비슷한 설정이 필요할 수 있겠지만, 백엔드만 해 온 입장에서는 코드로 컴파일러 동작을 조절하는 경우가 거의 없어서 꽤 신선하게 느껴졌습니다.
## 마지막으로

원래는 SwiftUI까지 함께 이야기하려 했는데, Swift만으로도 분량이 꽤 길어졌습니다. 그래서 SwiftUI 이야기는 다음 글에서 이어서 다루려고 합니다. 그래도 Swift만으로도 흥미로운 부분이 많아서, 튜토리얼을 따라가며 여러 경험을 해 본 선택은 괜찮았다고 생각합니다.

또 Kotlin과 Swift는 확실히 닮은 점이 있어서, 한쪽 경험이 있으면 다른 쪽으로 들어가는 진입장벽도 꽤 낮아지는 것 같습니다. 외국어 학습에서 말하는 배경지식의 효과와도 비슷한 느낌입니다. 그런 의미에서도 Kotlin을 꾸준히 다뤄 온 경험이 꽤 도움이 됐습니다.
