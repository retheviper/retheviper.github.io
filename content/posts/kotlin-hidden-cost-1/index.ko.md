---
title: "Kotlin의 숨겨진 비용 1"
date: 2021-11-14
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
  - java
translationKey: "posts/kotlin-hidden-cost-1"
---

Kotlin의 큰 장점 가운데 하나는 역시 생산성입니다. 문법이 간결하고 여러 문법적 편의가 잘 갖춰져 있어서, 같은 일을 하더라도 Java보다 짧고 읽기 쉬운 코드가 나오는 경우가 많습니다. 다만 이런 편의성 뒤에는 성능이나 바이트코드 관점에서의 비용이 숨어 있을 수 있습니다.

이번 글에서는 그런 지점을 다룬 [Exploring Kotlin’s hidden costs - Part 1](https://bladecoder.medium.com/exploring-kotlins-hidden-costs-part-1-fbb9935d9b62)을 바탕으로 내용을 정리해 보겠습니다. 완전한 번역이라기보다는 핵심을 추려 소개하는 요약에 가깝습니다.
다만 원문은 2017년에 쓰였고, 당시 Kotlin 버전은 1.1이었습니다. 지금은 컴파일러도 많이 발전했기 때문에 세부 결과는 달라졌을 수 있습니다. 그래도 어떤 지점에서 비용이 생기는지 이해하는 데는 여전히 참고할 가치가 있습니다. 특히 글에 나온 바이트코드를 최신 Kotlin이 생성하는 결과와 직접 비교해 보면 더 흥미로울 것 같습니다.
이 시리즈는 `Part 1`부터 `Part 3`까지 이어지는데, 이번 글에서는 먼저 `Lambda 표현식`과 `companion object` 부분을 보겠습니다.
## 고차 함수와 람다 표현식

예를 들어 다음 같은 함수를 정의해 둔다고 해 봅시다. 전달받은 함수를 DB 트랜잭션 안에서 실행하고, 그 결과 행 수를 돌려주는 형태입니다.
```kotlin
fun transaction(db: Database, body: (Database) -> Int): Int {
    db.beginTransaction()
    try {
        val result = body(db)
        db.setTransactionSuccessful()
        return result
    } finally {
        db.endTransaction()
    }
}
```

위의 함수는 Lambda를 전달하여 다음과 같이 사용할 수 있습니다.
```kotlin
val deletedRows = transaction(db) {
    it.delete(“Customers”, null, null)
}
```

Kotlin은 Java 1.6 JVM과도 호환되지만, Java 1.6 자체는 lambda를 지원하지 않습니다. 그래서 Kotlin은 호환성을 유지하기 위해 lambda나 익명 함수를 내부적으로 `Function` 객체로 바꿔 처리합니다.
### Function 객체

그럼, 실제로 컴파일된 Lambda(body)가 Java의 코드로서는 어떻게 되어 있는지를 살펴 보겠습니다. (여기에서는 Intellij/Android Studio의 `Show Kotlin Bytecode`의 `Decompile` 기능을 사용하고 있습니다)
```java
class MyClass$myMethod$1 implements Function1 {
   // 생성된 메서드
   // 브리지 메서드
   public Object invoke(Object var1) {
      return Integer.valueOf(this.invoke((Database)var1));
   }

   public final int invoke(@NotNull Database it) {
      Intrinsics.checkParameterIsNotNull(it, "it");
      return it.delete("Customers", null, null);
   }
}
```

즉 lambda나 익명 함수를 쓰면, 컴파일 결과에는 보조 메서드와 클래스가 추가로 생깁니다. 그리고 이때 만들어지는 `Function` 객체 인스턴스는 상황에 따라 생성 방식이 달라집니다.
- value capture가 있는 경우: 호출할 때마다 `Function` 인스턴스가 생성되어 GC 대상이 됩니다.
- value capture가 없는 경우: `Function`이 싱글턴처럼 한 번만 생성되어 재사용됩니다.

이전 코드에서는 value capture가 없으므로 Lambda 호출자는 다음과 같은 코드로 컴파일됩니다.
```java
this.transaction(db, (Function1)MyClass$myMethod$1.INSTANCE);
```

반대로 value capture가 있는 고차 함수를 반복 호출하면, 이 객체 생성 비용과 GC 비용이 누적될 수 있습니다.
### Boxing 오버헤드

Java 8 이후에는 primitive 타입용 함수형 인터페이스를 여러 개 제공해 boxing/unboxing을 줄이려 합니다. 하지만 Kotlin이 생성하는 `Function` 인터페이스는 기본적으로 generic 기반입니다.
```kotlin
/** 인자를 하나 받는 함수 */
public interface Function1<in P1, out R> : Function<R> {
    /** 인자를 받아 함수를 실행한다 */
    public operator fun invoke(p1: P1): R
}
```

즉 고차 함수에 넘긴 lambda가 primitive 타입을 파라미터나 반환값으로 다룰 때는 boxing/unboxing이 일어날 수 있습니다. 앞서 본 바이트코드에서도 반환값이 `Integer`로 boxing되는 것을 확인할 수 있었습니다. 호출 횟수가 적다면 크게 신경 쓰지 않아도 되지만, 반복 횟수가 많다면 성능에 영향을 줄 수 있습니다.
### 인라인 함수

다행히 Kotlin에는 `inline` 키워드가 있습니다. 이를 사용하면 고차 함수 내용을 호출 지점에 직접 펼쳐 넣는 방식으로 컴파일할 수 있습니다. 그래서 다음과 같은 비용을 줄일 수 있습니다.
- Function 객체의 인스턴스가 생성되지 않음
- primitive 타입을 다룰 때 boxing/unboxing이 줄어듭니다.
- 메서드 수가 불필요하게 늘어나지 않습니다.
- 함수 호출 오버헤드를 줄일 수 있습니다.

인라인된 경우의 코드를 확인해 봅시다. `transaction` 함수가 사라지고 `db.delete`을 직접 호출하는 것을 볼 수 있습니다. 또, 반환값의 `result`도 Wrapper 클래스로부터 primitive 타입이 되어 있는 것을 알 수 있습니다.
```java
db.beginTransaction();
try {
   int result$iv = db.delete("Customers", null, null);
   db.setTransactionSuccessful();
} finally {
   db.endTransaction();
}
```

다만 `inline`에도 주의할 점은 있습니다.
- 재귀 구조와는 잘 맞지 않습니다.
- public inline 함수는 접근 가능한 멤버 제약이 있습니다.
- 호출 지점마다 코드가 펼쳐지므로 바이트코드가 커질 수 있습니다.

가능하면 고차 함수는 `inline`으로 처리하고, 너무 긴 로직이 들어간다면 그 안에서 다시 일반 함수로 분리하는 편이 균형이 좋습니다. 특히 성능이 중요한 경로라면 어떤 함수에 `inline`을 붙일지 의식적으로 선택할 필요가 있습니다.
## Companion object

Kotlin에서는 Java처럼 정적 필드나 메서드를 직접 선언하지 않고, 대신 `companion object`를 사용합니다.
### 클래스의 private 필드를 companion object에서 액세스

다음과 같은 예가 있다고 가정합시다.
```kotlin
class MyClass private constructor() {

    private var hello = 0

    companion object {
        fun newInstance() = MyClass()
    }
}
```

위 코드가 컴파일되면 `companion object`는 싱글턴 클래스가 됩니다. 그래서 외부 형태인 companion object가 클래스의 private 필드에 접근할 수 있도록, 컴파일러가 보조 `getter`와 `setter`를 추가로 만듭니다. companion object는 이 메서드를 통해 필드에 접근합니다.
```text
ALOAD 1
INVOKESTATIC be/myapplication/MyClass.access$getHello$p (Lbe/myapplication/MyClass;)I
ISTORE 2
```

Java라면 package-private 같은 방식으로 우회할 수 있었겠지만, Kotlin에는 같은 개념이 없습니다. `public`이나 `internal`을 써도 접근 메서드가 생기는 경우가 있고, 이 메서드는 static이 아니라 instance 메서드이기 때문에 비용이 더 큽니다. 그래서 단순히 최적화만을 위해 접근 제한자를 억지로 바꾸는 것은 좋은 선택이 아닐 수 있습니다. companion object에서 같은 필드에 자주 접근한다면 캐시 같은 다른 방법을 고민하는 편이 낫습니다.
### Companion object 상수에 액세스

Kotlin에서 클래스의 정적 상수는 `companion object`에 정의하는 것이 일반적입니다.
```kotlin
class MyClass {

    companion object {
        private val TAG = "TAG"
    }

    fun helloWorld() {
        println(TAG)
    }
}
```

겉보기 간단하고 좋은 코드이지만, Kotlin 1.2.40 이전의 경우라면 꽤 뒤의 코드는 더러워지고 있습니다.
#### Kotlin 1.2.40 이하의 경우

`companion object`에 정의된 private 상수에 액세스하는 경우 위와 같은 것(`getter` 사용)이 발생합니다.
```text
GETSTATIC be/myapplication/MyClass.Companion : Lbe/myapplication/MyClass$Companion;
INVOKESTATIC be/myapplication/MyClass$Companion.access$getTAG$p (Lbe/myapplication/MyClass$Companion;)Ljava/lang/String;
ASTORE 1
```

문제는 이것만이 아닙니다. 생성된 메서드는 실제 값을 반환하지 않고 instance 메서드로 생성된 `getter`을 호출합니다.
```text
ALOAD 0
INVOKESPECIAL be/myapplication/MyClass$Companion.getTAG ()Ljava/lang/String;
ARETURN
```

상수가 `public`이면 직접 액세스할 수 있지만, 여전히 `getter` 메서드를 통해 값에 접근하게 됩니다.
그리고 상수 값을 저장하기 위해 Kotlin 컴파일러는 `companion object`가 아니라 클래스 쪽에 `private static final` 필드를 생성합니다. 게다가 `companion object`에서 이 필드에 접근하기 위해 또 다른 메서드도 생성됩니다.
```text
INVOKESTATIC be/myapplication/MyClass.access$getTAG$cp()Ljava/lang/String; 
ARETURN
```

이렇게 한참 우회한 뒤에야 값을 읽게 됩니다.
```text
GETSTATIC be/myapplication/MyClass.TAG : Ljava/lang/String; 
ARETURN
```

요약하면 Kotlin 1.2.40 이전 버전을 사용하는 경우 다음과 같이 보입니다.
- `companion object`에서 정적 메서드 호출
  - `companion object`에서 instance 메서드 호출
  - 클래스의 static 메서드 호출
      - static 필드에서 값 읽기

이것을 Java의 코드로 표현하면 다음과 같습니다.
```java
public final class MyClass {
    private static final String TAG = "TAG";
    public static final Companion companion = new Companion();

    // 생성된 메서드
    public static final String access$getTAG$cp() {
        return TAG;
    }

    public static final class Companion {
        private final String getTAG() {
            return MyClass.access$getTAG$cp();
        }

        // 생성된 메서드
        public static final String access$getTAG$p(Companion c) {
            return c.getTAG();
        }
    }

    public final void helloWorld() {
        System.out.println(Companion.access$getTAG$p(companion));
    }
}
```

보다 비용이 낮은 Bytecode를 생성하는 것도 가능하지만 간단하지는 않습니다.
먼저 `const` 키워드를 사용하여 컴파일 타임 상수를 정의하여 메소드 호출을 없앨 수 있습니다. 그러나 Kotlin은 primitive 또는 String에 대해서만 가능한 방법입니다.
```kotlin
class MyClass {

    companion object {
        private const val TAG = "TAG"
    }

    fun helloWorld() {
        println(TAG)
    }
}
```

또는 `@JvmField`을 사용하여 Java 접근 방식을 취하는 방법을 생각해 볼 수 있습니다. 이렇게 하면 `getter`이나 `setter`이 생성되지 않고 필드에 직접 액세스할 수 있습니다. 다만, `@Jvm`계의 어노테이션은 Java와의 호환성을 위한 것이므로 이것이 과연 좋은 방법인가 어떤가를 생각하는 것이 좋을 것입니다. 그리고 `public` 필드만 가능한 방법입니다.
Android 개발의 경우라면 `Parcelable` 객체를 직접 구현하는 경우에만 유효한 방법으로 보입니다. 예를 들면 다음과 같습니다.
```kotlin
class MyClass() : Parcelable {

    companion object {
        @JvmField
        val CREATOR = creator { MyClass(it) }
    }

    private constructor(parcel: Parcel) : this()

    override fun writeToParcel(dest: Parcel, flags: Int) {}

    override fun describeContents() = 0
}
```

마지막 방법으로는 [ProGuard](https://developer.android.com/studio/build/shrink-code)나 R8과 같은 툴을 사용해 Bytecode의 최적화를 노리는 방법이 있을 것입니다.
#### Kotlin 1.2.40 이상의 경우

Kotlin 1.2.40부터는 `companion object`에 정의된 값이 메인 클래스에 저장된다는 점은 그대로지만, 메서드를 따로 만들고 호출하지 않아도 직접 접근할 수 있게 되었습니다. 이를 Java 코드로 표현하면 다음과 같습니다.
```java
public final class MyClass {
    private static final String TAG = "TAG";
    public static final Companion companion = new Companion();

    public static final class Companion {
    }

    public final void helloWorld() {
        System.out.println(TAG);
    }
}
```

또 위와 같이 `companion object`에 메서드가 하나도 없으면, ProGuard나 R8 같은 도구를 사용할 때 클래스 자체가 제거되면서 최적화됩니다.
다만 `companion object`에 정의된 메서드는 여전히 약간의 비용이 듭니다. 필드가 메인 클래스 쪽에 저장되기 때문에, `companion object`에 정의된 private 필드에 접근하려면 여전히 생성된 메서드를 거쳐야 합니다.
## 마지막으로

이번 글은 직접 실험한 결과라기보다 기존 분석 글을 따라가며 정리한 내용이지만, 그래도 꽤 배울 점이 많았습니다. 특히 IntelliJ가 언제 `inline`을 권하는지, 그리고 `companion object`가 단순한 문법 설탕만은 아니라는 점이 더 분명하게 보였습니다.

지금은 Kotlin 컴파일러도 많이 발전해서 원문 당시와 결과가 완전히 같지는 않을 수 있습니다. 그래도 "편한 문법이 어떤 코드로 바뀌는가"를 이해하는 출발점으로는 충분히 흥미로운 주제라고 생각합니다. 다음 편에서는 나머지 숨겨진 비용도 이어서 정리해 보겠습니다.
