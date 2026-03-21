---
title: "Kotlin의 숨겨진 비용 3"
date: 2021-11-28
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-hidden-cost-3"
---

이번 글은 `Kotlin의 숨겨진 비용` 시리즈의 마지막 편입니다. 앞선 글들도 흥미로웠지만, 이번에는 Kotlin다운 기능에 더 가까이 들어가다 보니 언어 자체를 이해해야 보이는 부분이 많습니다.
주제는 `위임 프로퍼티`와 `range를 이용한 루프`입니다. 내용은 [Exploring Kotlin’s hidden costs - Part 3](https://bladecoder.medium.com/exploring-kotlins-hidden-costs-part-3-3bf6e0dbf0a4)를 바탕으로 핵심만 정리했습니다.
## 위임 프로퍼티

[위임 프로퍼티](https://kotlinlang.org/docs/delegated-properties.html)는 `getter`와 `setter` 동작을 별도 delegate 객체에 맡기는 [프로퍼티](https://kotlinlang.org/docs/properties.html)를 뜻합니다. 이를 이용하면 재사용 가능한 커스텀 프로퍼티 동작을 만들 수 있습니다.
```kotlin
class Example {
    var p: String by Delegate()
}
```

위임 객체는 프로퍼티 값을 읽고 쓸 수 있도록 `getValue()`와 `setValue()`를 구현해야 합니다. 이 함수들은 프로퍼티 메타데이터(이름 등)와 객체 인스턴스를 인자로 받습니다.
클래스에 위임 프로퍼티가 정의되면 컴파일러는 아래와 같은 코드를 만듭니다.
```java
public final class Example {
   @NotNull
   private final Delegate p$delegate = new Delegate();
   // 생성된 필드
   static final KProperty[] $$delegatedProperties = new KProperty[]{(KProperty)Reflection.mutableProperty1(new MutablePropertyReference1Impl(Reflection.getOrCreateKotlinClass(Example.class), "p", "getP()Ljava/lang/String;"))};

   @NotNull
   public final String getP() {
      return this.p$delegate.getValue(this, $$delegatedProperties[0]);
   }

   public final void setP(@NotNull String var1) {
      Intrinsics.checkParameterIsNotNull(var1, "<set-?>");
      this.p$delegate.setValue(this, $$delegatedProperties[0], var1);
   }
}
```

즉 클래스 안에 정적 메타데이터가 추가되고, 값 읽기와 쓰기 과정도 일반 프로퍼티보다 한 단계 더 복잡해집니다.
### 위임 인스턴스

앞선 예시에서는 프로퍼티 구현을 위해 새로운 위임 인스턴스가 생성됩니다. 위임이 상태를 가지는 경우라면 이런 구조가 자연스럽습니다. 예를 들면 로컬 캐시를 두는 경우가 그렇습니다.
```kotlin
class StringDelegate {
    private var cache: String? = null

    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        var result = cache
        if (result == null) {
            result = someOperation()
            cache = result
        }
        return result
    }
}
```

또 생성자에 추가 인자를 전달해야 하는 경우에도 새 위임 인스턴스가 필요합니다.
```kotlin
class Example {
    private val nameView by BindViewDelegate<TextView>(R.id.name)
}
```

반대로 상태가 없고, 전달받은 객체와 프로퍼티 이름만 사용하면 된다면 위임 클래스를 `object`로 만들어 싱글턴처럼 쓸 수 있습니다.
```kotlin
object FragmentDelegate {
    operator fun getValue(thisRef: Activity, property: KProperty<*>): Fragment? {
        return thisRef.fragmentManager.findFragmentByTag(property.name)
    }
}
```

기존 객체를 확장해 위임하는 방법도 있습니다. 즉 `getValue()`와 `setValue()`를 확장 함수로 정의할 수 있다는 뜻입니다. Kotlin 표준 라이브러리도 `Map`과 `MutableMap` 위임에서 이런 패턴을 사용합니다. 이 경우 프로퍼티 이름이 키 역할을 합니다.
하나의 클래스 안에서 여러 프로퍼티가 같은 위임 인스턴스를 공유해야 한다면, 생성자에서 그 인스턴스를 만들어 재사용하는 쪽이 낫습니다. Kotlin 1.1부터는 함수 안의 로컬 변수도 위임 속성으로 만들 수 있으므로, 필요한 시점까지 초기화를 늦추는 패턴도 가능합니다.
결국 클래스에 위임 프로퍼티를 하나 추가할 때마다 오버헤드와 메타데이터가 따라옵니다. 그래서 가능한 한 재사용 가능한 구조를 먼저 고민하는 편이 좋고, 위임해야 할 항목이 많다면 정말 이 방법이 최선인지 한 번 더 따져볼 필요가 있습니다.
### 제네릭 위임

위임 함수는 제네릭으로도 정의할 수 있습니다. 그래서 하나의 위임 클래스를 여러 타입의 프로퍼티에 공통으로 적용할 수도 있습니다.
```kotlin
private var maxDelay: Long by SharedPreferencesDelegate<Long>()
```

다만 이렇게 primitive 타입을 제네릭 위임으로 처리하면, 값 읽기와 쓰기 과정에서 `boxing`과 `unboxing`이 발생합니다. 프로퍼티가 non-null이어도 마찬가지입니다.
그래서 non-null primitive 프로퍼티를 위임할 때는 제네릭 정의를 남발하지 않는 편이 안전합니다.
### 표준 위임(`lazy()`)

Kotlin에는 [Delegates.notNull()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/not-null.html), [Delegates.observable()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/observable.html), [lazy()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/lazy.html) 같은 표준 위임 기능도 있습니다.
`lazy()`은 읽기 전용 위임 속성에 대한 함수입니다. 처음으로 로드가 발생하면 속성을 초기화할 때 lambda를 지정할 수 있습니다.
```kotlin
private val dateFormat: DateFormat by lazy {
    SimpleDateFormat("dd-MM-yyyy", Locale.getDefault())
}
```

이 방식은 실제로 값이 필요해질 때까지 비용 큰 초기화를 미룰 수 있어, 성능과 가독성 양쪽에서 꽤 유용합니다.
다만 `lazy()`는 `inline` 함수가 아니고, 넘긴 람다는 별도의 `Function` 클래스로 컴파일됩니다. 반환되는 위임 객체도 인라인되지 않으므로 이 비용은 알아 두는 편이 좋습니다.
또 `lazy()`에서 자주 놓치기 쉬운 부분이 `mode` 인자입니다. 이 값에 따라 실제 위임 구현이 달라집니다.
```kotlin
public fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)
public fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
        when (mode) {
            LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
            LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
            LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
        }
```

`mode`를 지정하지 않으면 기본적으로 `LazyThreadSafetyMode.SYNCHRONIZED`가 사용됩니다. 여러 스레드에서 안전하게 초기화되도록 하기 위한 선택이지만, 그만큼 `double-checked lock` 비용도 따라옵니다.
프로퍼티에 단일 스레드에서만 접근한다는 것이 분명하다면, 굳이 이 락 비용을 감수할 필요는 없습니다. 그럴 때는 `LazyThreadSafetyMode.NONE`을 사용할 수 있습니다.
```kotlin
val dateFormat: DateFormat by lazy(LazyThreadSafetyMode.NONE) {
    SimpleDateFormat("dd-MM-yyyy", Locale.getDefault())
}
```

## Ranges

[Ranges](https://kotlinlang.org/docs/ranges.html)를 사용하면 제한된 범위의 값 집합을 표현할 수 있습니다. `Comparable`을 구현한 타입이라면 대부분 범위를 만들 수 있고, 이 표현은 내부적으로 [ClosedRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-closed-range/)와 연결됩니다.
### 포함 테스트

range를 사용하면 특정 값이 범위 안에 있는지 `in`이나 `!in`으로 검사할 수 있습니다.
```kotlin
if (i in 1..10) {
    println(i)
}
```

range는 non-null primitive 타입(`Int`, `Long`, `Byte`, `Short`, `Float`, `Double`, `Char`)에 대해 최적화가 이루어지기 때문에, 컴파일 결과는 아래처럼 단순한 비교식이 됩니다.
```java
if(1 <= i && i <= 10) {
   System.out.println(i);
}
```

그래서 이 경우에는 별도 객체 할당이나 큰 오버헤드가 없습니다. 그렇다면 primitive가 아닌 타입은 어떨까요?
```kotlin
if (name in "Alfred".."Alicia") {
    println(name)
}
```

Kotlin 1.1.50 이전에는 컴파일시 `ClosedRange` 객체가 항상 생성되었습니다. 그러나 1.1.50부터는 다음과 같습니다.
```kotlin
if(name.compareTo("Alfred") >= 0) {
   if(name.compareTo("Alicia") <= 0) {
      System.out.println(name);
   }
}
```

range는 `when` 조건식에서도 쓸 수 있습니다. 이런 경우는 `if-else`보다 읽기 쉬운 편입니다.
```kotlin
val message = when (statusCode) {
    in 200..299 -> "OK"
    in 300..399 -> "Find it somewhere else"
    else -> "Oops"
}
```

다만 range를 프로퍼티나 다른 중간 구조에 담아 두고 사용하는 경우에는 비용이 생길 수 있습니다. 예를 들어 아래처럼 범위를 getter로 빼 두면 이야기가 달라집니다.
```kotlin
private val myRange get() = 1..10

fun rangeTest(i: Int) {
    if (i in myRange) {
        println(i)
    }
}
```

이 경우 컴파일하면 `IntRange` 객체가 추가됩니다.
```java
private final IntRange getMyRange() {
   return new IntRange(1, 10);
}

public final void rangeTest(int i) {
   if(this.getMyRange().contains(i)) {
      System.out.println(i);
   }
}
```

이 현상은 프로퍼티 getter를 `inline`처럼 보이게 만들어도 크게 달라지지 않습니다. 그래서 가능하면 range는 실제로 사용하는 위치에 직접 쓰는 편이 좋습니다. primitive가 아닌 객체를 다루는 경우에는 범위 객체를 재사용하는 쪽이 더 나을 수도 있습니다.
### for 루프

`Float`과 `Double`을 제외한 primitive 형의 범위를 루프로 사용하는 것도 좋은 선택입니다.
```kotlin
for (i in 1..10) {
    println(i)
}
```

컴파일된 결과에는 오버헤드가 발생하지 않습니다.
```java
int i = 1;
for(byte var2 = 11; i < var2; ++i) {
   System.out.println(i);
}
```

역순으로 루프하고 싶은 경우는 [downTo()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/down-to.html)를 사용할 수 있습니다.
```kotlin
for (i in 10 downTo 1) {
    println(i)
}
```

이 경우에도 오버헤드가 발생하지 않습니다.
```java
int i = 10;
byte var1 = 1;
while(true) {
   System.out.println(i);
   if(i == var1) {
      return;
   }
   --i;
}
```

[until](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/until.html)을 사용하여 특정 값 미만으로 루프하는 것이 좋습니다.
```kotlin
for (i in 0 until size) {
    println(i)
}
```

이전에는 약간의 비용이 들었지만 Kotlin 1.1.4 이상에서는 다음과 같은 코드가 생성됩니다.
```java
int i = 0;
for(int var2 = size; i < var2; ++i) {
   System.out.println(i);
}
```

다만, 그 외는 최적화가 그다지 효과가 없는 경우도 있습니다. [reversed()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/reversed.html)를 사용하는 예가 있다고 가정해 봅시다.
```kotlin
for (i in (1..10).reversed()) {
    println(i)
}
```

컴파일된 코드가 그다지 깨끗하다고는 말할 수 없습니다.
```kotlin
IntProgression var10000 = RangesKt.reversed((IntProgression)(new IntRange(1, 10)));
int i = var10000.getFirst();
int var3 = var10000.getLast();
int var4 = var10000.getStep();
if(var4 > 0) {
   if(i > var3) {
      return;
   }
} else if(i < var3) {
   return;
}

while(true) {
   System.out.println(i);
   if(i == var3) {
      return;
   }

   i += var4;
}
```

`IntRange` 오브젝트가 범위를 재정의하기 위해 생성되고 `IntProgression` 오브젝트가 요소를 역순으로 정렬하기 위해 생성됩니다.
`progression`을 만드는데 둘 이상의 함수를 사용하면 둘 이상의 객체를 만드는 것과 같은 오버헤드가 발생합니다.
위의 규칙은 [step()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/step.html)을 사용하는 경우도 같고 `step 1`을 지정해도 상황은 변하지 않습니다.
```kotlin
for (i in 1..10 step 2) {
    println(i)
}
```

또한 생성된 코드에서 마지막 값을 읽을 때 `IntProgression` 객체의 마지막 요소와 `step()`에 지정된 범위를 고려하여 추가 처리가 수행됩니다. 위의 샘플이라면 마지막 요소는 `9`입니다.
따라서 `for`을 이용한 루프를 할 때는 가능한 한 `..`, `downTo()`, `until()`을 이용하여 오버 헤드를 피하는 것이 좋습니다.
### forEach 루프

`for` 루프 대신 range에 대해 inline 확장 함수의 `forEach()`을 사용하는 경우에도 결과는 그다지 변하지 않습니다.
```kotlin
(1..10).forEach {
    println(it)
}
```

그러나 `forEach()`은 `Iterable`에 대해서만 최적화되지 않습니다. 즉, iterator를 생성해야 함을 의미합니다. 그래서 컴파일되면 다음과 같이 됩니다.
```java
Iterable $receiver$iv = (Iterable)(new IntRange(1, 10));
Iterator var1 = $receiver$iv.iterator();

while(var1.hasNext()) {
   int element$iv = ((IntIterator)var1).nextInt();
   System.out.println(element$iv);
}
```

이것은 이전 샘플보다 비용이 많이 듭니다. `IntRange` 오브젝트를 생성할 뿐만 아니라 `IntIterator` 오브젝트도 생성하고 있기 때문입니다. primitive가 아니라면 더 많은 비용이 듭니다.
그래서 range를 사용한 루프가 필요한 경우는 `forEach()`보다 `for` 루프를 사용하여 오버 헤드를 줄이는 것이 좋습니다.
### collection 인덱스 루프

Kotlin의 표준 라이브러리는 [indices](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/indices.html)라는 확장 속성으로 배열과 `Collection`의 INTEX를 제공합니다.
```kotlin
val list = listOf("A", "B", "C")
for (i in list.indices) {
    println(list[i])
}
```

`indices`의 컴파일된 결과는 좋은 최적화를 보여줍니다.
```java
List list = CollectionsKt.listOf(new String[]{"A", "B", "C"});
int i = 0;
for(int var2 = ((Collection)list).size(); i < var2; ++i) {
   Object var3 = list.get(i);
   System.out.println(var3);
}
```

`IntRange` 객체가 만들어지지 않았습니다. 그럼 직접 구현해 보면 어떻게 될까요.
```kotlin
inline val SparseArray<*>.indices: IntRange
    get() = 0 until size()

fun printValues(map: SparseArray<String>) {
    for (i in map.indices) {
        println(map.valueAt(i))
    }
}
```

확장 속성으로 정의하고 컴파일하면 덜 효율적인 코드가 있음을 알 수 있습니다. `IntRange` 객체가 만들어집니다.
```kotlin
public static final void printValues(@NotNull SparseArray map) {
   Intrinsics.checkParameterIsNotNull(map, "map");
   IntRange var10000 = RangesKt.until(0, map.size());
   int i = var10000.getFirst();
   int var2 = var10000.getLast();
   if(i <= var2) {
      while(true) {
         Object $receiver$iv = map.valueAt(i);
         System.out.println($receiver$iv);
         if(i == var2) {
            break;
         }
         ++i;
      }
   }
}
```

이 경우 대신 `until()`과 `for` 루프를 사용하는 것이 좋습니다.
```kotlin
fun printValues(map: SparseArray<String>) {
    for (i in 0 until map.size()) {
        println(map.valueAt(i))
    }
}
```

## 마지막으로

이번 편도 개인적으로는 꽤 공부가 됐습니다. 위임 프로퍼티는 평소 많이 쓰지 않았기 때문에 기본 동작을 다시 이해하는 데 도움이 됐고, range 역시 Java 습관대로 테스트 클래스 필드에 두고 여러 함수에서 재사용하던 방식이 생각보다 비용을 만들 수 있다는 점이 인상적이었습니다.
결국 중요한 것은 Kotlin이 제공하는 기능과 API를 "편리하다"는 이유만으로 쓰지 않고, 실제로 어떤 코드로 컴파일되는지까지 한 번쯤 확인해 보는 일이라고 생각합니다. [옥컴의 면도](https://ko.wikipedia.org/wiki/%E3%98%EA%EC%BB%B4%EC%9D%98_%EB%A9%B4%EB%8F%84)라는 말처럼, 가능하면 더 단순한 로직과 구조를 우선하는 편이 대체로 안전합니다. IntelliJ의 `Tools > Kotlin > Show Kotlin Bytecode` 메뉴를 활용하면 Java 코드로 어떻게 풀리는지도 쉽게 볼 수 있으니, 성능이 신경 쓰이는 부분은 직접 확인해 보는 습관이 도움이 됩니다.
이번 시리즈는 평소처럼 제 경험담을 쓰기보다는 요약과 정리에 가까웠지만, 개인적으로는 꽤 값진 공부였습니다. 다음에도 비슷하게 정리할 만한 주제가 있으면 이어서 다뤄 보겠습니다.
