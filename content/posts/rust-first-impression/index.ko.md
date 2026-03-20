---
title: "Rust를 써 본 첫인상"
date: 2022-03-27
categories: 
  - rust
image: "../../images/rust.webp"
tags:
  - rust
  - python
  - kotlin
  - java
  - go
translationKey: "posts/rust-first-impression"
---

Rust를 공부해 보고 싶다고 생각한 건 대략 2년 전쯤이었습니다. 그때는 주로 Java와 Python을 다루고 있었는데, 성능이 아주 중요한 영역에서는 이 두 언어만으로는 한계가 있지 않을까 하는 생각이 들었습니다. 그래서 네이티브로 컴파일되는 언어를 한 번은 제대로 접해 볼 필요가 있다고 느꼈고, 가능하면 GC가 없고 포인터를 직접 다루는 언어로 애플리케이션을 써 보면, 주력 언어라고 할 수 있는 Java에 대한 이해도 더 깊어지지 않을까 기대했습니다.

그 후보로 떠올린 것이 Go와 Rust였습니다. 다만 Go는 세간의 평가와는 별개로, 제 입장에서 보면 제가 추구하는 방향과는 약간 어긋나는 부분이 있다고 느꼈습니다. 특히 이직 후 Go와 Kotlin을 병행해서 만지다 보니, 좋든 나쁘든 제가 언어에서 무엇을 중요하게 보는지 조금 더 분명해졌습니다.

그래서 슬슬 다음 후보였던 Rust를 본격적으로 접해 보고 싶다는 생각이 들었습니다. 이것 역시 세간의 평가를 그대로 따르기보다, 실제로 제게 맞는지 직접 확인해 보고 싶었습니다. 요즘은 여러 언어를 두루 다루는 [Polyglot Programmer](https://medium.com/@guestposts_92864/what-is-a-polyglot-programmer-and-why-you-should-become-one-e5629bf720c2)의 시대라고도 하고, 많은 언어가 서로의 장점을 흡수하면서 점점 비슷해졌다는 말도 있습니다. 그래도 저는 어디까지나 "내게 맞는 언어가 무엇인가"를 확인해 보고 싶다는 마음으로 Rust를 보기 시작했습니다.

그래서 이번에는 [The Rust Programming Language](https://doc.rust-jp.rs/book-ja/title-page.html)를 읽으며 흥미로웠던 부분을, 지금까지 제가 경험한 다른 언어들과 비교해 보면서 가볍게 정리해 보려고 합니다. 문서가 길고 저도 아직 이해가 깊지 않기 때문에, 우선은 일부만 다루겠습니다.

## Loop

Rust에는 전통적인 `for`, `while` 외에도 조건을 따로 적지 않는 무한 루프용 `loop`가 있습니다. 특정 조건에서만 `break`로 빠져나오면 됩니다.

```rust
loop {
    // do something
}
```

다른 언어에서는 보통 `while(true)` 같은 형태가 많습니다. 예를 들어 Python은 아래처럼 씁니다.

```python
while true:
  # do something
```

Kotlin이나 Java도 비슷합니다.

```java
while(true) {
    // do something
}
```

Kotlin에서는 확장 함수가 있으니 `loop` 비슷한 걸 만들 수 있지 않을까 생각해 봤는데, 그렇게 하면 컴파일러 입장에서 그 코드를 "루프"로 인식하지 못합니다. 그래서 내부에서 `break`를 쓰면 컴파일 에러가 납니다.

```kotlin
fun test() {
  loop {
    break // 컴파일 에러
  }
}

fun loop(doSomething: () -> Unit) {
  while(true) {
    doSomething()
  }
}
```

Go는 조건식 없는 `for`를 써서 간단하게 무한 루프를 만듭니다.

```go
for {
    // do something
}
```

개인적으로는 `while(true)`나 조건 없는 `for`는 결국 관습에 가까운 문법이라고 느껴집니다. 그런 의미에서 Rust처럼 `loop`라는 키워드를 따로 제공하는 편이 코드 의도를 더 직관적으로 드러내는 것 같았습니다. 사소해 보일 수도 있지만, 한 번 언어 사양으로 굳어지면 쉽게 바꾸기 어려운 부분이니 이런 키워드 선택도 언어 설계에서 꽤 중요하다고 생각합니다.

## Array

Rust에서 배열의 일부를 잘라낼 때는 보통 이렇게 씁니다. 참조(`&`)를 사용하고, 잘라낸 결과는 별도의 데이터 타입인 `slice`로 다뤄집니다.

```rust
let arr = [0, 1, 2, 3, 4];
let slice = &arr[1..3];

println!("{:?}", slice); // [1, 2]
```

Python도 느낌은 꽤 비슷합니다. 다만 잘라낸 결과가 여전히 `list`라는 점은 Rust와 다릅니다.

```python
list = [0, 1, 2, 3, 4]
slice = list[1:3]
print(slice) # [1, 2]
```

Kotlin에서는 [List](https://kotlinlang.org/docs/collections-overview.html#list)에 [Range](https://kotlinlang.org/docs/ranges.html)를 넘겨 비슷한 동작을 만들 수 있습니다.

```kotlin
val list = listOf(0, 1, 2, 3, 4)
val subList = list.slice((1..2))
println(subList) // [1, 2]
val subListUntil = list.slice((1 until 3))
println(subListUntil) // [1, 2]
```

여기서 조금 아쉬운 점은 Kotlin의 `Range` 문법이 익숙하지 않으면 범위가 한눈에 들어오지 않는다는 것입니다. 다행히 IntelliJ IDEA 2021.3부터는 [범위 힌트](https://blog.jetbrains.com/idea/2021/10/intellij-idea-2021-3-eap-5/#inline_hints_for_ranges)를 보여 주기 때문에, 이전 버전을 쓰고 있다면 업데이트할 만합니다.

Java에서는 [List.subList()](https://docs.oracle.com/javase/jp/8/docs/api/java/util/List.html#subList-int-int-)로 비슷하게 처리할 수 있습니다.

```java
List<Integer> list = List.of(0, 1, 2, 3, 4);
List<Integer> subList = list.subList(1, 3);
System.out.println(subList); // [1, 2]
```

Go도 Python과 거의 같은 방식으로 쓸 수 있습니다. 그리고 잘라낸 뷰를 `slice`라고 부른다는 점은 Rust와 비슷합니다. 다만 Go의 slice는 array와 달리 가변 길이입니다.

```go
arr := []int{0, 1, 2, 3, 4}
slice := arr[1:3]
fmt.Println(slice) // [1 2]
```

## Immutability

Rust에서 변수 선언은 기본적으로 `let` 하나이고, 기본값은 불변입니다. 물론 `mut`를 붙이면 값을 다시 대입할 수 있습니다.

```rust
fn main() {
    let mut x = 5;
    println!("The value of x is: {}", x); // The value of x is: 5
    x = 6;
    println!("The value of x is: {}", x); // The value of x is: 6
}
```

다른 언어들도 대개 변수의 가변성 여부를 키워드로 표현하거나, 반대로 기본값을 가변으로 두고 필요할 때만 불변 키워드를 붙이는 식으로 처리합니다. 그런데 Rust는 기본값이 immutable이라는 점이 꽤 인상적이었습니다. GC가 없는 언어로서 메모리 안전성을 확보하기 위한 설계가 이런 부분에서도 드러난다고 볼 수 있겠죠.

Python은 선언과 재대입을 엄격히 구분하지 않습니다.

```python
x = 5
print("The value of x is: {}".format(x)) # The value of x is: 5
x = 6
print("The value of x is: {}".format(x)) # The value of x is: 6
```

Kotlin은 `val`과 `var`로 나눕니다.

```kotlin
val x = 5
x = 5 // 컴파일 에러
var y = 6
y = 7 // OK
```

Java는 Rust와는 반대에 가깝습니다. `final`을 붙이지 않으면 기본적으로 재대입이 가능합니다.

```java
int x = 5
x = 6 // OK
final int y = 6
y = 7 // 컴파일 에러
```

Go는 immutable 변수를 직접 만드는 방식이 거의 없어서 재대입이 자유롭습니다. 그래서 오히려 Java의 `final` 같은 키워드가 있었으면 좋겠다는 생각도 들었습니다.

```go
x := 5
x = 6
fmt.Println(x) // 6
```

## Shadowing

이 부분은 예상하지 못했던 지점이었는데, Rust 문서에서는 변수 shadowing을 하나의 기능으로 소개하고 있습니다. `mut`가 있으면 충분하지 않나 싶기도 하지만, 변수를 불변으로 유지하면서 새로운 타입이나 값으로 다시 정의하고 싶을 때 유용하다고 설명합니다.

Rust에서는 이렇게 쓸 수 있습니다.

```rust
fn main() {
    let x = 5;
    let x = x + 1; // 6
    let x = x * 2; // 12

    println!("The value of x is: {}", x); // The value of x is: 12
}
```

Python도 겉모양만 보면 비슷한 코드를 자연스럽게 쓸 수 있습니다.

```python
x = 5
x = x + 1
x = x * 2
print("The value of x is: {}".format(x)) # The value of x is: 12
```

Kotlin에서는 shadowing이 일부 상황에서만 가능합니다. 예를 들면 함수 인자와 지역 변수 이름이 겹치는 경우입니다.

```kotlin
fun shadow(value: Int) {
  val value = value + 1 // Name shadowed: value
  println(value) // val 쪽이 출력됨
}
```

Java는 이런 식의 shadowing을 허용하지 않습니다. 다만 아래처럼 필드와 매개변수 이름이 겹치는 구조는 가능합니다.

```java
class Clazz {
  private int value = 0;

  public void setValue(int value) {
    this.value = value;
  }
}
```

Go는 조금 다릅니다. 아래 예시에서는 `x`를 두 번 선언하지만, 서로 다른 스코프에 있기 때문에 가능합니다.

```go
x := 0
fmt.Println("Before the decision block, x:", x) // Before the decision block, x: 0

if true {
  x := 1
  x++
}
fmt.Println("After the decision block, x:", x) // After the decision block, x: 0
```

같은 코드라도 `if` 안에서 `:=` 대신 `=`를 쓰면 의미가 완전히 달라집니다.

```go
x := 0
fmt.Println("Before the decision block, x:", x) // Before the decision block, x: 0

if true {
  x = 1
  x++
}
fmt.Println("After the decision block, x:", x) // After the decision block, x: 2
```

다른 언어들은 보통 shadowing을 피하도록 유도하는 경우가 많은데, Rust는 이를 꽤 적극적인 기능으로 다루고 있다는 점이 흥미로웠습니다. 이 역시 뒤에서 말할 ownership과 연결되는 개념처럼 보였습니다.

## Ownership

다른 언어와 비교했을 때 Rust를 가장 Rust답게 만드는 개념은 역시 ownership이 아닐까 싶습니다. 지금까지 저는 GC 없는 언어를 본격적으로 다뤄 본 적이 없어서, 이 개념이 특히 흥미로웠습니다.

예를 들어 Kotlin/Native는 [참조 카운팅](https://blog.jetbrains.com/kotlin/2021/05/kotlin-native-memory-management-update/#kn-gc)을 사용한다고 알려져 있습니다. Java, Python, Go는 GC가 돌면서 더 이상 참조되지 않는 객체의 메모리를 해제합니다.

반면 Rust는 정수, 부동소수점, 불리언, 문자 같은 scalar type을 제외한 참조형에 대해 ownership 규칙을 적용합니다. 기본적으로는 값이 스코프를 벗어나면 해제되고, 사용 방식에 따라 소유권이 이동합니다. Java의 primitive / reference 구분이 떠오르기도 하지만, Rust는 메모리 해제가 훨씬 더 적극적이고 엄격하게 이뤄지는 느낌입니다.

기본적으로는 "한 번 사용한 값을 아무 생각 없이 다시 쓰는 것"이 허용되지 않을 수 있다고 생각하는 편이 더 맞는 것 같습니다. 이 ownership에서 특히 인상적이었던 부분이 `move`였습니다.

### Move

move는 ownership과 직접 연결되는 개념입니다. 변수와 실제 데이터가 어떻게 연결되는지 보여 주는 좋은 예이기도 합니다.

아래 코드는 얼핏 보면 아무 문제 없어 보입니다.

```rust
let s1 = "hello";
let s2 = s1;

println!("s1 = {}, s2 = {}", s1, s2); // s1 = hello, s2 = hello
```

하지만 `String literal` 대신 `String`을 사용하면 상황이 달라집니다.

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

이 코드는 컴파일 단계에서 에러가 납니다.

```shell
error[E0382]: use of moved value: `s1`
 --> src/main.rs:4:27
  |
3 |     let s2 = s1;
  |         -- value moved here
4 |     println!("{}, world!", s1);
  |                            ^^ value used here after move
  |
  = note: move occurs because `s1` has type `std::string::String`,
which does not implement the `Copy` trait
```

즉, `s1`의 소유권이 `s2`로 이동했기 때문에 `s1`을 더 이상 사용할 수 없다는 뜻입니다. 둘 다 같은 데이터를 가지게 하고 싶다면 명시적으로 복사해야 합니다.

```rust
let s1 = String::from("hello");
let s2 = s1.clone();
println!("s1 = {}, s2 = {}", s1, s2); // s1 = hello, s2 = hello
```

문서 설명을 보면, Rust에서는 스코프를 벗어날 때 메모리를 해제하는데 여러 변수가 같은 포인터를 공유하면 이중 해제 위험이 생길 수 있기 때문에 이런 규칙을 둔 것으로 보입니다. 또한 `String literal`과 달리 `String`은 immutable이 아니기 때문에, 한쪽 변경이 다른 쪽까지 퍼지는 상황을 예방하는 의미도 있는 것 같습니다.

이런 대입이 문제가 될 수 있는 다른 언어도 있습니다. 예를 들어 Python에서는 두 변수가 같은 참조를 공유하기 때문에 재대입 시 둘 다 영향받는 상황을 쉽게 만들 수 있습니다.

```python
s1 = "hello"
s2 = s1
print("s1 = {}, s2 = {}".format(s1, s1)) # s1 = hello, s2 = hello

s1 = "world"
print("s1 = {}, s2 = {}".format(s1, s1)) # s1 = world, s2 = world
```

Java는 `String`을 immutable하게 다루기 때문에, `s1`을 다시 대입해도 `s2`는 영향을 받지 않습니다. Kotlin도 JVM 위에서는 비슷하게 동작합니다.

```kotlin
var s1 = "hello"
val s2 = s1
println("s1 = $s1, s2 = $s2") // s1 = hello, s2 = hello

s1 = "world"
println("s1 = $s1, s2 = $s2") // s1 = world, s2 = hello
```

Java도 마찬가지입니다.

```java
var s1 = "hello";
var s2 = s1;
System.out.println(String.format("s1 = %s, s2 = %s", s1, s2)); // s1 = hello, s2 = hello

s1 = "world";
System.out.println(String.format("s1 = %s, s2 = %s", s1, s2)); // s1 = world, s2 = hello
```

Go도 변수 자체를 immutable하게 만들 수는 없지만, 이런 재대입 때문에 다른 값이 덩달아 바뀌는 상황은 비교적 안전하게 처리됩니다.

```go
s1 := "hello"
s2 := s1
fmt.Println(fmt.Sprintf("s1 = %s, s2 = %s", s1, s2)) // s1 = hello, s2 = hello


s1 = "world"
fmt.Println(fmt.Sprintf("s1 = %s, s2 = %s", s1, s2)) // s1 = world, s2 = hello
```

Rust에서 명시적으로 복사하지 않으면 데이터가 이동해 버린다는 점은 분명 코딩하면서 계속 의식해야 하는 부분입니다. 다만 이 문제를 컴파일 타임에 바로 확인할 수 있다는 점, 그리고 다른 언어에서 무심코 하던 위험한 습관을 교정해 줄 수 있다는 점을 생각하면 꽤 좋은 설계처럼 느껴졌습니다.

## Closure

Rust에서는 closure를 `|val| val + x` 같은 형태로 작성합니다. 다른 언어로 치면 lambda에 가까운 개념이죠. 문법은 조금 독특하지만, 타입을 생략할 수 있다는 점은 꽤 편리해 보였습니다. 물론 타입을 명시적으로 적는 것도 가능합니다.

```rust
fn main() {
    // i32 인수가 필요한 경우
    let closure_annotated = |i: i32| -> i32 { i + 1 };
    let closure_inferred  = |i     |          i + 1  ;

    let i = 1;
    println!("closure_annotated: {}", closure_annotated(i)); // closure_annotated: 2
    println!("closure_inferred: {}", closure_inferred(i)); // closure_inferred: 2

    // 인수가 없는 경우
    let one = || 1;
    println!("closure returning one: {}", one()); // closure returning one: 1
}
```

Python은 아래처럼 쓸 수 있습니다. 함수 안에 함수를 정의할 수도 있지만, 간단한 경우라면 lambda가 훨씬 가볍습니다.

```python
closure = lambda x : x + 1
print(closure(1)) // 2
```

물론 [타입 힌트](https://docs.python.org/3/library/typing.html)를 붙일 수는 있지만, Rust처럼 컴파일 타임에서 강하게 확인하는 흐름과는 결이 다릅니다.

Kotlin도 간단히 정의할 수는 있지만, 적어도 인수 타입은 써 주거나 변수 타입을 함께 적어야 합니다.

```kotlin
val closure = { x: Int -> x + 1 }
println(closure(1)) // 2
```

Java는 메서드 안에 메서드를 정의할 수 없기 때문에, 1.8부터 도입된 `Functional Interface`를 사용해야 합니다. `var`가 도입된 뒤에도 이쪽은 여전히 제약이 꽤 많습니다.

```java
Function<Integer, Integer> closure = i -> i + 1;
System.out.println(closure.apply(1)); // 2
```

Go도 함수 안에 함수를 둘 수는 있지만, 다른 언어의 lambda처럼 아주 간단한 표기라기보다는 익명 함수를 변수에 넣는 느낌이 더 강합니다.

```go
closure := func(x int) int {
  return x + 1
}
fmt.Println(closure(1)) // 2
```

그리고 Rust의 closure에서 또 흥미로운 점은, closure를 인수로 받는 함수를 정의할 때의 문법입니다. generic과 `where`를 활용해서 closure 타입 제약을 표현합니다.

```rust
// F라는 closure를 인수로 받는 함수
fn apply_to_3<F>(f: F) -> i32 where
    // F는 i32를 받아 i32를 반환하는 closure
    F: Fn(i32) -> i32 {
    f(3)
}

fn main() {
    let double = |x| 2 * x;
    println!("3 doubled: {}", apply_to_3(double)); // 3 doubled: 6
}
```

다른 언어들은 "closure를 인수로 받는다"는 상황에서도 표기 방식이 크게 달라지지 않는 경우가 많은데, Rust는 이 부분도 상당히 자기 색이 강하다고 느꼈습니다.

## 마지막으로

아직 문서의 절반도 채 읽지 못했고, 실제로 무언가 제대로 된 애플리케이션을 만들어 본 것도 아니기 때문에 이번 글만으로 Rust를 충분히 이해했다고 말할 수는 없습니다. 그래도 오랜만에 다른 언어를 배우면서 꽤 흥미로운 지점이 많아서, 우선은 첫인상을 정리해 보고 싶었습니다.

Rust 컴파일러가 굉장히 우수하고, 컴파일러가 안내하는 방향대로만 코드를 짜도 배울 점이 많다는 이야기를 자주 들었습니다. 실제로 언어 설계도 상당히 잘 되어 있다는 인상을 받았기 때문에, 앞으로도 계속 공부하면서 느낀 점, 배운 점, 흥미로웠던 점을 블로그에 정리해 보고 싶습니다.

이번 글 하나로 끝내고 싶지는 않으니, 올해는 Rust도 조금 더 꾸준히 파고 들어가 보고 싶습니다.
