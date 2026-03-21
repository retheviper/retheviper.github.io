---
title: "Kotlin의 숨겨진 비용 2"
date: 2021-11-21
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-hidden-cost-2"
---

이번에도 Kotlin이 편의성을 제공하는 대신 어디에서 비용이 생기는지 살펴보겠습니다. 요즘은 예전만큼 이런 차이에 민감하지 않아도 되는 경우가 많지만, 컴파일 결과가 어떻게 달라지는지 알고 코드를 쓰는 건 여전히 의미가 있습니다.

지난 글에서는 고차 함수, Lambda, `companion object`를 다뤘습니다. 이번 글에서는 로컬 함수, null 안전성, `vararg`가 어떤 식으로 추가 비용을 만들 수 있는지 정리합니다. 이 글은 [Exploring Kotlin’s hidden costs - Part 2](https://bladecoder.medium.com/exploring-kotlins-hidden-costs-part-2-324a4a50b70)의 내용을 바탕으로 정리한 글입니다.
## 로컬 함수

함수 안에 정의한 함수를 로컬 함수라고 합니다. 로컬 함수는 자신을 감싸는 바깥 함수의 스코프에 접근할 수 있습니다. 아래 예제에서는 `sumSquare`가 `someMath`의 파라미터 `a`를 그대로 사용합니다.
```kotlin
fun someMath(a: Int): Int {
    fun sumSquare(b: Int) = (a + b) * (a + b)

    return sumSquare(1) + sumSquare(2)
}
```

로컬 함수는 Lambda와 비슷해 보이지만 제약도 있습니다. 로컬 함수 자체도, 로컬 함수를 포함한 바깥 함수도 `inline`으로 만들 수 없기 때문에 함수 호출 비용을 없앨 수 없습니다.
또 컴파일 결과를 보면 로컬 함수는 `Function` 객체 형태로 바뀝니다. 그래서 지난 글에서 다룬 "인라인되지 않은 Lambda"와 비슷한 비용 구조를 가집니다. 위 코드를 Java로 풀어 보면 다음과 같은 형태가 됩니다.
```java
public static final int someMath(final int a) {
   Function1 sumSquare$ = new Function1(1) {
      // 생성된 메서드
      // 브리지 메서드
      public Object invoke(Object var1) {
         return Integer.valueOf(this.invoke(((Number)var1).intValue()));
      }

      public final int invoke(int b) {
         return (a + b) * (a + b);
      }
   };
   return sumSquare$.invoke(1) + sumSquare$.invoke(2);
}
```

다만 Lambda와 비교했을 때 조금 나은 점도 있습니다. 호출하는 쪽에서 함수 인스턴스의 실제 타입을 알 수 있기 때문에, 제네릭 인터페이스를 거치는 대신 생성된 익명 클래스의 메서드를 직접 호출합니다. 그래서 바깥 함수에서 로컬 함수를 호출할 때 `casting`이나 `boxing`이 추가되지 않습니다. 실제 바이트코드를 보면 다음과 같습니다.
```text
ALOAD 1
ICONST_1
INVOKEVIRTUAL be/myapplication/MyClassKt$someMath$1.invoke (I)I
ALOAD 1
ICONST_2
INVOKEVIRTUAL be/myapplication/MyClassKt$someMath$1.invoke (I)I
IADD
IRETURN
```

메서드는 두 번 호출되지만, 인수와 반환값이 모두 `int`라서 `boxing`과 `unboxing`은 발생하지 않습니다.
그래도 여전히 함수 객체를 만들어야 한다는 비용은 남습니다. 이 점은 캡처를 없애는 방식으로 조금 줄일 수 있습니다.
```kotlin
fun someMath(a: Int): Int {
    fun sumSquare(a: Int, b: Int) = (a + b) * (a + b)

    return sumSquare(a, 1) + sumSquare(a, 2)
}
```

이렇게 바꾸면 `Function` 객체 인스턴스를 재사용할 수 있습니다. 결국 로컬 함수는 바깥 함수의 변수에 접근할 수 있다는 장점이 있지만, 그 대가로 추가 클래스와 함수 객체 생성 비용이 생길 수 있습니다. 그래서 가능하다면 캡처하지 않는 형태로 바꾸는 편이 낫습니다.
## Null 안전

Kotlin의 대표적인 장점 중 하나는 null이 될 수 있는 타입과 아닌 타입을 명확하게 구분한다는 점입니다. 덕분에 런타임에서 예상치 못한 `NullPointerException`을 줄일 수 있습니다.
### Non-null 파라미터의 런타임에서의 체크

예를 들어 다음과 같은 함수가 있다고 가정합니다.
```kotlin
fun sayHello(who: String) {
    println("Hello $who")
}
```

이것은 Java 코드에서 다음과 같습니다.
```java
public static final void sayHello(@NotNull String who) {
   Intrinsics.checkParameterIsNotNull(who, "who");
   String var1 = "Hello " + who;
   System.out.println(var1);
}
```

`@NotNull` 어노테이션이 붙어서 Java 쪽에도 null을 넘기면 안 된다는 의도를 전달합니다.
하지만 어노테이션만으로는 호출자에게 null 안전성을 강제할 수 없기 때문에, 실제 생성 코드에서는 정적 메서드 호출로 파라미터를 한 번 더 검사합니다. 이 검사는 문제가 있을 때 `IllegalArgumentException`을 던집니다.
public 함수에는 이런 `Intrinsics.checkParameterIsNotNull()` 검사가 항상 들어가지만, private 함수에는 보통 추가되지 않습니다. 같은 Kotlin 코드 안에서는 컴파일러가 null 안전성을 더 강하게 보장할 수 있기 때문입니다.
이 검사의 성능 영향은 대개 무시할 수준이지만, 아주 민감한 환경에서는 줄이고 싶을 수도 있습니다. 그런 경우 컴파일러 옵션에 `-Xno-param-assertions`을 추가하거나 [ProGuard](https://www.guardsquare.com/proguard) 규칙에 다음 설정을 넣어 런타임 null 검사를 제거할 수 있습니다.
```java
-assumenosideeffects class kotlin.jvm.internal.Intrinsics {
    static void checkParameterIsNotNull(java.lang.Object, java.lang.String);
}
```

위의 규칙을 추가하려면 Android ProGuard의 Optimization 설정이 사용 설정되어 있는지 확인해야 합니다. 이 설정은 기본적으로 비활성화되어 있습니다.
### Nullable primitive 형

먼저 기억할 점은 nullable로 선언한 primitive 타입은 항상 Java의 `int`, `float` 대신 `Integer`, `Float` 같은 boxed reference 타입으로 다뤄진다는 것입니다. 그래서 그만큼 추가 비용이 생깁니다.
[autoboxing](http://docs.oracle.com/javase/8/docs/technotes/guides/language/autoboxing.html)에 익숙하다면 이해하기 쉽습니다. Java에서는 `Integer`와 `int`를 비교적 느슨하게 넘나들 수 있지만, Kotlin은 nullable과 non-null을 명확하게 구분하므로 어느 쪽을 선택하는지가 더 분명하게 드러납니다. 가능한 경우 non-null 타입을 쓰는 편이 가독성과 성능 양쪽에서 유리합니다.
```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}

fun add(a: Int?, b: Int?): Int {
    return (a ?: 0) + (b ?: 0)
}
```

따라서 가능한 한 코드의 가독성과 성능을 고려하여 non-null을 선택하는 것이 좋습니다.
### 배열

Kotlin에는 다음 세 가지 배열이 있습니다.
- `IntArray`, `FloatArray` 같은: primitive 타입의 배열. `int[]`, `float[]`과 같은 타입으로 컴파일된다.
- `Array<T>`:non-null 객체의 형태가 지정된 배열. primitive에 대해 @MASK_1@@이 발생할 수 있습니다.
- `Array<T?>`:nullable 객체의 형태가 지정된 배열. 명확하게 `boxing`이 일어난다.

non-null primitive 배열이 필요하다면 가능하면 `Array<Int>`보다 `IntArray`를 사용하는 편이 좋습니다.
## Varargs

Kotlin에서도 [가변 길이 인수](https://kotlinlang.org/docs/functions.html#variable-number-of-arguments-varargs)를 정의할 수 있습니다. 문법은 Java와 조금 다르지만 개념은 비슷합니다.
```kotlin
fun printDouble(vararg values: Int) {
    values.forEach { println(it * 2) }
}
```

Java와 마찬가지로 `vararg`도 컴파일되면 결국 지정한 타입의 배열이 됩니다. 위 함수는 크게 세 가지 방식으로 호출할 수 있습니다.
### 여러 매개변수 전달

```kotlin
printDouble(1, 2, 3)
```

Kotlin 컴파일러는 이를 새 배열을 만들고 초기화하는 코드로 바꿉니다. 이 점은 Java와 크게 다르지 않습니다.
```kotlin
printDouble(new int[]{1, 2, 3});
```

즉 호출할 때마다 새 배열을 만드는 비용이 들어갑니다. 다만 Java에서도 비슷한 비용이 생깁니다.
### 배열 전달

Java에서는 배열을 그대로 넘길 수 있지만, Kotlin에서는 `spread operator`를 사용해야 합니다.
```kotlin
val values = intArrayOf(1, 2, 3)
printDouble(*values)
```

Java에서는 배열 참조가 `as-is`로 함수에 전달되며 새 배열 할당이 발생하지 않습니다. 그러나 Kotlin의 `spread operator`은 다음을 수행합니다.
```java
int[] values = new int[]{1, 2, 3};
printDouble(Arrays.copyOf(values, values.length));
```

배열 복사본을 함수에 전달하기 때문에, 호출자 쪽 배열을 건드리지 않는다는 점에서는 더 안전하다고 볼 수 있습니다. 대신 그만큼 메모리를 더 사용합니다.
### 배열과 다른 인수를 혼합하여 전달

`spread operator`의 장점은 배열과 다른 인수를 섞어서 전달할 수 있다는 점입니다.
```kotlin
val values = intArrayOf(1, 2, 3)
printDouble(0, *values, 42)
```

이 경우 컴파일 결과는 조금 더 복잡합니다.
```java
int[] values = new int[]{1, 2, 3};
IntSpreadBuilder var10000 = new IntSpreadBuilder(3);
var10000.add(0);
var10000.addSpread(values);
var10000.add(42);
printDouble(var10000.toArray());
```

새 배열을 만드는 것뿐 아니라, 임시 빌더 객체를 통해 최종 배열 크기까지 계산합니다. 그래서 단순히 배열을 넘길 때보다 비용이 더 큽니다.
호출 빈도가 높고 성능이 중요한 코드라면, 가능하면 `vararg` 대신 실제 배열을 파라미터로 받는 쪽이 더 낫습니다.
## 마지막으로

이번 글에서 특히 흥미로웠던 부분은, 평소 편하게 쓰던 기능들이 컴파일 이후에는 생각보다 다른 모습으로 바뀐다는 점이었습니다. 로컬 함수는 스코프를 좁히는 데 유용하지만 캡처 비용을 동반할 수 있고, nullable primitive는 자연스럽게 boxing 비용으로 이어집니다.

또 `vararg`는 문법상 간단해 보여도 호출 방식에 따라 배열 복사나 임시 객체 생성이 뒤따를 수 있다는 점이 인상적이었습니다. 평소 자주 쓰지 않는 기능일수록 이런 차이를 더 쉽게 잊게 되는데, 그래서 이런 정리는 한 번쯤 기억해 둘 만하다고 생각합니다.
