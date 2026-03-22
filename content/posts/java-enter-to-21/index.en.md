---
title: "What's New in Java 21"
date: 2023-09-18
translationKey: "posts/java-enter-to-21"
categories: 
  - java
image: "../../images/java.webp"
tags:
  - java
  - kotlin
---
It's been about two years since Java 17 was released, and Java 21 is being released this month. Depending on the project, there are still plenty of cases where older versions such as 1.7 remain in use, but since 21 is a new LTS release, it may be worth considering for new projects. So in this post, I will give a rough summary of what changed in Java 21.

This article talks about changes from Java 17, so please refer to [previous post](../java-enter-to-17/) for information about changes from Java 11 to 17.

## Language specs

### String Templates (Preview)

Languages like Kotlin have a feature called String Interpolation. This is a feature that allows you to embed variables within strings. For example, if you have variables x and y, when you want to embed them in a String, you can write them in Kotlin as follows.

```kotlin
val s = "$x plus $y equals ${x + y}"
```

To achieve this in Java, you would write it as follows.

```java
// String concatenation
String s = x + " plus " + y + " equals " + (x + y);

// StringBuilder
String s = new StringBuilder()
                 .append(x)
                 .append(" plus ")
                 .append(y)
                 .append(" equals ")
                 .append(x + y)
                 .toString();

// String.format
String s = String.format("%2$d plus %1$d equals %3$d", x, y, x + y);
String t = "%2$d plus %1$d equals %3$d".formatted(x, y, x + y);

// MessageFormat
MessageFormat mf = new MessageFormat("{0} plus {1} equals {2}");
String s = mf.format(x, y, x + y);
```

All of these methods are verbose and have poor readability. Therefore, Java 21 added a feature called String Templates. This is a feature that allows you to embed variables within strings. So, it is now possible to create Strings in Java in a much simpler way.

However, string interpolation also comes with concerns such as SQL injection, so Java takes a different approach here. Rather than embedding variables directly into a string, it first creates a [template](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/lang/StringTemplate.html) and then uses that to produce the final string. The code looks like this.

```java
// Using STR
String name = "Joan";
String info = STR."My name is \{name}"; // My name is Joan

// Using RAW
String name = "Joan";
StringTemplate st = RAW."My name is \{name}";
String info = STR.process(st); // My name is Joan
```

[STR](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/lang/StringTemplate.html#STR) and [RAW](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/lang/StringTemplate.html#RAW) first create an instance of StringTemplate, but this StringTemplate instance has a field called fragments and an array called values. fragments is an array of strings in which variables are replaced with empty strings, and values ​​is an array of variable values. Therefore, you can get not only the string that is the result of embedding variables, but also the value of the actually given variable.

```java
int x = 10, y = 20;
StringTemplate st = RAW."\{x} plus \{y} equals \{x + y}";
String s = st.toString(); // StringTemplate{ fragments = [ "", " plus ", " equals ", "" ], values = [10, 20, 30] }
```

Also, StringTemplate has an Interface called Processor, which allows you to implement your own as a Functional Interface.

```java
// Processor Interface
public interface StringTemplate {
    @FunctionalInterface
    public interface Processor<R, E extends Throwable> {
        R process(StringTemplate st) throws E;
    }
}

var INTER = StringTemplate.Processor.of(StringTemplate::interpolate);
String s = INTER."\{x} plus \{y} equals \{x + y}";
```

It's still in Preview, so the way it's used may change in the future, but it's a pretty interesting approach, so it's a feature I'd like to keep an eye on for future trends.

### Sequenced Collections

In the case of Java Collections, depending on the type, there are various ways to write it to get the last element, and it becomes redundant. For example, if you take the first element and the last element, it will be as follows depending on the type of Colleciton.

```java
// List
var firstOnList = list.get(0);
var lastOnList = list.get(list.size() - 1);

// Deque
var firstOnDeque = deque.getFirst();
var lastOnDeque = deque.getLast();

// SortedSet
var firstOnSortedSet = sortedSet.first();
var lastOnSortedSet = sortedSet.last();

// LinkedHashSet
var firstOnLinkedHashSet = linkedHashSet.iterator().next();
var lastOnLinkedHashSet = linkedHashSet.stream().reduce((first, second) -> second).orElse(null);
```

Also, if you want to reverse the order of the loop, the code may become redundant or difficult to use. For example, if you look at the code below, you will see that although they are trying to do the same thing, the code is completely different.
  
```java
// NavigableSet with descendingSet
for (var e: navigableSet.descendingSet()) {
    process(e);
}

// Deque with reverse Iterator
for (var it = deque.descendingIterator(); it.hasNext(); ) {
    var e = it.next();
    process(e);
}

// List with reverse ListIterator
for (var it = list.listIterator(list.size()); it.hasPrevious(); ) {
    var e = it.previous();
    process(e);
}
```

Also, depending on the implementation class, there are cases where a Collection that preserves the order of elements is downgraded to one that does not. For example, wrapping a LinkedHashSet in Collections::unmodifiableSet will cause the LinkedHashSet to lose its order.

Therefore, Java 21 adds Interfaces called [SequencedCollection](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/util/SequencedCollection.html) and [SequencedSet](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/util/SequencedSet.html) to solve the above problem. These Interfaces provide methods such as:

```java
interface SequencedCollection<E> extends Collection<E> {
    // new method
    SequencedCollection<E> reversed();
    // methods promoted from Deque
    void addFirst(E);
    void addLast(E);
    E getFirst();
    E getLast();
    E removeFirst();
    E removeLast();
}

interface SequencedSet<E> extends Set<E>, SequencedCollection<E> {
    SequencedSet<E> reversed();    // covariant override
}
```

Also, an Interface called [SequencedMap](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/util/SequencedMap.html) has been added to Map, and it provides the following methods.

```java
interface SequencedMap<K,V> extends Map<K,V> {
    // new methods
    SequencedMap<K,V> reversed();
    SequencedSet<K> sequencedKeySet();
    SequencedCollection<V> sequencedValues();
    SequencedSet<Entry<K,V>> sequencedEntrySet();
    V putFirst(K, V);
    V putLast(K, V);
    // methods promoted from NavigableMap
    Entry<K, V> firstEntry();
    Entry<K, V> lastEntry();
    Entry<K, V> pollFirstEntry();
    Entry<K, V> pollLastEntry();
}
```

With the addition of these new interfaces, the inheritance relationships across the Collection hierarchy have changed as follows.

![Sequenced Collections](SequencedCollectionDiagram20220216.webp)
*Source: OpenJDK - [JEP 431: Sequenced Collections](https://openjdk.org/jeps/431)*

I think downcasting may occur due to inheritance relationships, but in the case of List, it inherits from SequencedCollection, so you can use the new method as is.

### Generational ZGC

ZGC is Garbage Collector introduced in Java 11, but Java 21 adds a feature called Generational ZGC. This will divide the ZGC heap into Young Generation and Old Generation to improve ZGC performance. This makes it possible to perform Young Generation GC more frequently, shorten the Young Generation GC time, and reduce memory and CPU overhead.

To use Generational ZGC, specify the startup options as follows.

```bash
java -XX:+UseZGC -XX:+ZGenerational
```

However, regarding the new GC, the [official document](https://openjdk.org/jeps/439) mostly discusses design and implementation details, while from an application engineer's standpoint the practical question is how much performance benefit it actually provides. For that, I think [this article](https://timefold.ai/blog/2023/java-21-performance/) is useful when evaluating Generational ZGC in practice. Its conclusion is that ParallelGC still performs best overall. Of course, that depends on the machine spec, especially available memory, so it is worth testing in your own environment.

### Record Patterns

Java 16 introduces a feature called [Pattern Matching](https://openjdk.org/jeps/394), which makes it easier to write code that performs type checking with instanceOf and then performs a cast. For example:

```java
// Prior to Java 16
if (obj instanceof String) {
    String s = (String)obj;
    ... use s ...
}

// As of Java 16
if (obj instanceof String s) {
    ... use s ...
}
```

In Java 21, this Pattern Matching can now also be applied to [Record](https://openjdk.org/jeps/395), which was also introduced in Java 16. For example, the code below.

```java
// As of Java 16
record Point(int x, int y) {}

static void printSum(Object obj) {
    if (obj instanceof Point p) {
        int x = p.x();
        int y = p.y();
        System.out.println(x+y);
    }
}

// As of Java 21
static void printSum(Object obj) {
    if (obj instanceof Point(int x, int y)) {
        System.out.println(x+y);
    }
}
```

It can also be applied to nested Records. For example, the following code is also possible.

```java
static void printXCoordOfUpperLeftPointWithPatterns(Rectangle r) {
    if (r instanceof Rectangle(ColoredPoint(Point(var x, var y), var c),
                               var lr)) {
        System.out.println("Upper-left corner: " + x);
    }
}
```

### Pattern Matching for switch

Pattern matching improvements also apply to `switch`. For example, as shown below, you can now handle `null` directly in `switch`, use type patterns, and add guarded conditions with `when`.

```java
static void testStringEnhanced(String response) {
    switch (response) {
        case null -> { }
        case "y", "Y" -> {
            System.out.println("You got it");
        }
        case "n", "N" -> {
            System.out.println("Shame");
        }
        case String s
        when s.equalsIgnoreCase("YES") -> {
            System.out.println("You got it");
        }
        case String s
        when s.equalsIgnoreCase("NO") -> {
            System.out.println("Shame");
        }
        case String s -> {
            System.out.println("Sorry?");
        }
    }
}
```

This improvement has also been applied to Enums. For example, the following code is now possible.

```java
static void exhaustiveSwitchWithBetterEnumSupport(CardClassification c) {
    switch (c) {
        case Suit.CLUBS -> {
            System.out.println("It's clubs");
        }
        case Suit.DIAMONDS -> {
            System.out.println("It's diamonds");
        }
        case Suit.HEARTS -> {
            System.out.println("It's hearts");
        }
        case Suit.SPADES -> {
            System.out.println("It's spades");
        }
        case Tarot t -> {
            System.out.println("It's a tarot");
        }
    }
}
```

Although it has not yet been applied to primitive types, it seems that this will be improved in the future, so I hope to see it in the next version.

### Foreign Functions and Memory Access API (Third Preview)

A feature introduced in Java 19 that adds APIs that allow access to code and data outside of the Java runtime. This is available as a Third Preview in Java 21 and will allow you to call C and C++ code from Java. For example, you can call a C library with the code below.

```java
// 1. Find foreign function on the C library path
Linker linker          = Linker.nativeLinker();
SymbolLookup stdlib    = linker.defaultLookup();
MethodHandle radixsort = linker.downcallHandle(stdlib.find("radixsort"), ...);
// 2. Allocate on-heap memory to store four strings
String[] javaStrings = { "mouse", "cat", "dog", "car" };
// 3. Use try-with-resources to manage the lifetime of off-heap memory
try (Arena offHeap = Arena.ofConfined()) {
    // 4. Allocate a region of off-heap memory to store four pointers
    MemorySegment pointers
        = offHeap.allocateArray(ValueLayout.ADDRESS, javaStrings.length);
    // 5. Copy the strings from on-heap to off-heap
    for (int i = 0; i < javaStrings.length; i++) {
        MemorySegment cString = offHeap.allocateUtf8String(javaStrings[i]);
        pointers.setAtIndex(ValueLayout.ADDRESS, i, cString);
    }
    // 6. Sort the off-heap data by calling the foreign function
    radixsort.invoke(pointers, javaStrings.length, MemorySegment.NULL, '\0');
    // 7. Copy the (reordered) strings from off-heap to on-heap
    for (int i = 0; i < javaStrings.length; i++) {
        MemorySegment cString = pointers.getAtIndex(ValueLayout.ADDRESS, i);
        javaStrings[i] = cString.getUtf8String(0);
    }
} // 8. All off-heap memory is deallocated here
assert Arrays.equals(javaStrings,
                     new String[] {"car", "cat", "dog", "mouse"});  // true
```

Nowadays, when using Java to reference libraries and applications created in other languages, most people use the Wrapper or Runtime, but I think that using this will make it easier to call libraries in other languages, reducing the size of applications and improving performance. However, since it is still in Preview, it may change in the future, and since it directly accesses memory, there is a possibility of memory leaks, so I think you need to be careful when using it.

## Unnamed Patterns and Variables (Preview)

Variables that are not used during processing can now be expressed as `_`. So, you can write code like the following.

```java
// Loop
int acc = 0;
for (Order _ : orders) {
    if (acc < LIMIT) { 
        ... acc++ ...
    }
}

// Multiple assignment
Queue<Integer> q = ... // x1, y1, z1, x2, y2, z2, ...
while (q.size() >= 3) {
    var x = q.remove();
    var _ = q.remove();
    var _ = q.remove(); 
    ... new Point(x, 0) ...
}

// Catch block
String s = ...
try { 
    int i = Integer.parseInt(s);
    ... i ...
} catch (NumberFormatException _) { 
    System.out.println("Bad number: " + s);
}

// try-with-resources
try (var _ = ScopedContext.acquire()) {
    ... no use of acquired resource ...
}

// Lambda
stream.collect(Collectors.toMap(String::toUpperCase, _ -> "NODATA"))
```

## Virtual threads

This feature has been in development for a long time under the name Project Loom. In my personal opinion, I think this is the feature that is attracting the most attention in Java 21. In existing multi-threaded programming, there was a physical constraint on the number of threads that could be created, but the virtual thread introduced this time uses the OS's threads to be further divided into smaller parts, so it is characterized by the ability to handle more threads at the same time.The usage is not much different from existing physical threads, so you can use it with code like the following.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
}  // executor.close() is called implicitly, and waits
```

Since virtual threads do not have a 1:1 correspondence with actual OS threads, there is little need to limit the number of threads by creating a ThreadPool as in the existing method. Even the official website says that it is not recommended to use Pool.

In fact, according to [an article about implementing and experimenting with Dispatcher to use Java's virtual threads in Kotlin](https://kt.academy/article/dispatcher-loom), even a machine with 30 threads can generate 1 million
You can see that we can create virtual threads to do processing. For server-side applications written in the JVM language, the traditional threading model required limiting the number of threads, so I think the introduction of virtual threads will make it possible to process more requests at the same time.

## Unnamed Classes and Instance Main Methods (Preview)

Functions can now be defined at the top level. So, for the traditional Hello World sample, you can write code like the following.

```java
// Prior to Java 21
public class HelloWorld { 
    public static void main(String[] args) { 
        System.out.println("Hello, World!");
    }
}

// As of Java 21
void main() {
    System.out.println("Hello, World!");
}
```

Top-level functions and fields are also treated as members of the Unnamed Class, so code like the one below will work without any problems.

```java
// Method
String greeting() { return "Hello, World!"; }

void main() {
    System.out.println(greeting());
}

// Field
String greeting = "Hello, World!";

void main() {
    System.out.println(greeting);
}
```

Also, an Unnamed Class with a main function can be executed as follows.

```java
new Object() {
    // the unnamed class's body
}.main();
```

## Scoped Values (Preview)

In web applications, it is common for each request to be assigned a thread to ensure that it is executed within a consistent context. However, if you treat the context as an object, you will need to pass it as an argument to the function that is executed if it already exists. For example, the code below.

```java
@Override
void handle(Request request, Response response, FrameworkContext context) {
    ...
    var userInfo = readUserInfo(context);
    ...
}

private UserInfo readUserInfo(FrameworkContext context) {
    return (UserInfo)framework.readKey("userInfo", context);
}
```

Alternatively, you can use [ThreadLocal](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/lang/ThreadLocal.html) and write it as follows:

```java
public class Framework {
    private final Application application;
    public Framework(Application app) { this.application = app; }
    
    private final static ThreadLocal<FrameworkContext> CONTEXT 
                       = new ThreadLocal<>();

    void serve(Request request, Response response) {
        var context = createContext(request);
        CONTEXT.set(context);
        Application.handle(request, response);
    }

    public PersistedObject readKey(String key) {
        var context = CONTEXT.get();
        var db = getDBConnection(context);
        db.readKey(key);
    }
}
```

However, there are various problems when using ThreadLocal. First, the value of ThreadLocal itself is changed. Then, it is necessary to delete ThreadLocal values ​​that are no longer needed, which creates overhead.

Therefore, in Java 21, a class called [ScopedValue](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/lang/ScopedValue.html) was added. Using this, you can set the value per thread as follows.

```java
class Framework {
    private final static ScopedValue<FrameworkContext> CONTEXT 
                        = ScopedValue.newInstance();

    void serve(Request request, Response response) {
        var context = createContext(request);
        ScopedValue.where(CONTEXT, context)
                   .run(() -> Application.handle(request, response));
    }
    
    public PersistedObject readKey(String key) {
        var context = CONTEXT.get();
        var db = getDBConnection(context);
        db.readKey(key);
    }
}
```

ScopedVlaue doesn't have a setter, but that doesn't mean you can't give it other values. A different approach than ThreadLocal allows you to pass a specific value and execute the run() function.

```java
private static final ScopedValue<String> X = ScopedValue.newInstance();

void foo() {
    ScopedValue.where(X, "hello").run(() -> bar());
}

void bar() {
    System.out.println(X.get()); // prints hello
    ScopedValue.where(X, "goodbye").run(() -> baz());
    System.out.println(X.get()); // prints hello
}

void baz() {
    System.out.println(X.get()); // prints goodbye
}
```

This ScopedValue does not retain its value only while the thread is running, so it can be used more safely than ThreadLocal.

## Vector API (Sixth Incubator)

Unlike [Vector](https://docs.oracle.com/javase/8/docs/api/java/util/Vector.html), which handles arrays in the Java 1.0 era, a Vector API for calculating numbers (matrix) has been added. Basically, you will be able to do the following:

```java
// Prior to Java 21
void scalarComputation(float[] a, float[] b, float[] c) {
   for (int i = 0; i < a.length; i++) {
        c[i] = (a[i] * a[i] + b[i] * b[i]) * -1.0f;
   }
}

// As of Java 21
static final VectorSpecies<Float> SPECIES = FloatVector.SPECIES_PREFERRED;

void vectorComputation(float[] a, float[] b, float[] c) {
    int i = 0;
    int upperBound = SPECIES.loopBound(a.length);
    for (; i < upperBound; i += SPECIES.length()) {
        // FloatVector va, vb, vc;
        var va = FloatVector.fromArray(SPECIES, a, i);
        var vb = FloatVector.fromArray(SPECIES, b, i);
        var vc = va.mul(va)
                   .add(vb.mul(vb))
                   .neg();
        vc.intoArray(c, i);
    }
    for (; i < a.length; i++) {
        c[i] = (a[i] * a[i] + b[i] * b[i]) * -1.0f;
    }
}
```

I don't think it is used much in general web applications, but if you are writing a process that requires such calculations, it may be useful because it is faster and can be parallelized than traditional code.

## Deprecate the Windows 32-bit x86 Port for Removal

Since the Windows x86-32 port will eventually end, the first thing to do is to make it Deprecated. If Virtual Thread is an applicable OS, the performance improvement will not be as expected, and it is said to be a response to the fact that support for Windows 10, the last Windows compatible with 32 bits, will end in October 2025.

## Prepare to Disallow the Dynamic Loading of Agents

Dynamic loading with the Java agent also allows for changes to the running application. However, such features can also compromise the integrity of the application. To prevent such problems, we will disallow dynamic loading in the future and print a warning first in Java 21. Messages like the following may be output.

```text
WARNING: A {Java,JVM TI} agent has been loaded dynamically (file:/u/bob/agent.jar)
WARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning
WARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information
WARNING: Dynamic loading of agents will be disallowed by default in a future release
```

To avoid such warnings, you need to specify the option `-XX:+EnableDynamicAgentLoading` when running the application.Tools for monitoring applications such as [Datadog](https://www.datadoghq.com/) and [JMX](https://docs.oracle.com/cd/F25597_01/document/products/wls/docs90/jmxinst/understanding.html) may depend on this kind of functionality, so the implementation method may change when using future versions.

## Key Encapsulation Mechanism API

Compatible with the latest encryption algorithms. There is talk that existing encryption algorithms will no longer work on quantum computers, so I think it was introduced as a response (even the official version says ` Post-Quantum Cryptography standardization process`).

It supports functions such as generating public/private key pairs using new algorithms, encapsulation, and decapsulation. The usage is as follows.

```java
// Receiver side
KeyPairGenerator g = KeyPairGenerator.getInstance("ABC");
KeyPair kp = g.generateKeyPair();
publishKey(kp.getPublic());

// Sender side
KEM kemS = KEM.getInstance("ABC-KEM");
PublicKey pkR = retrieveKey();
ABCKEMParameterSpec specS = new ABCKEMParameterSpec(...);
KEM.Encapsulator e = kemS.newEncapsulator(pkR, specS, null);
KEM.Encapsulated enc = e.encapsulate();
SecretKey secS = enc.key();
sendBytes(enc.encapsulation());
sendBytes(enc.params());

// Receiver side
byte[] em = receiveBytes();
byte[] params = receiveBytes();
KEM kemR = KEM.getInstance("ABC-KEM");
AlgorithmParameters algParams = AlgorithmParameters.getInstance("ABC-KEM");
algParams.init(params);
ABCKEMParameterSpec specR = algParams.getParameterSpec(ABCKEMParameterSpec.class);
KEM.Decapsulator d = kemR.newDecapsulator(kp.getPrivate(), specR);
SecretKey secR = d.decapsulate(em);
```

## Structured Concurrency (Preview)

This is an API to make parallel processing easier. A unit of work performed by multiple threads can be treated as a single task.

For example, suppose you have the following code. This is a function that retrieves user and order data in different threads and returns the results.

```java
Response handle() throws ExecutionException, InterruptedException {
    Future<String>  user  = esvc.submit(() -> findUser());
    Future<Integer> order = esvc.submit(() -> fetchOrder());
    String theUser  = user.get();   // Join findUser
    int    theOrder = order.get();  // Join fetchOrder
    return new Response(theUser, theOrder);
}
```

The above code may cause the following problems:

- Even if an exception occurs in findUser(), fetchOrder() will be executed and waste resources.
- If the thread running handle() is interrupted, findUser() and fetchOrder() will remain running
- If findUser() takes too long to execute, it will wait for fetchOrder() even if it fails (resulting in failure)

The fact that these problems are mentioned means that the new API can solve them. The new API solves the above problem with code like the one below.

```java
Response handle() throws ExecutionException, InterruptedException {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Supplier<String>  user  = scope.fork(() -> findUser());
        Supplier<Integer> order = scope.fork(() -> fetchOrder());

        scope.join()            // Join both subtasks
             .throwIfFailed();  // ... and propagate errors

        // Here, both subtasks have succeeded, so compose their results
        return new Response(user.get(), order.get());
    }
}
```

When processing using [StructuredTaskScope](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/util/concurrent/StructuredTaskScope.html), there are the following advantages.

- If either findUser() or ffectOrder() fails, the remaining processing will be canceled.
- findUser() and fetchOrder() are canceled if the thread running handle() is interrupted
- Processes can be clearly understood

## API

This new API is well explained in the language specifications, and you can filter and check what has been added for each version using the new Javadoc, so I will only paste the [Javadoc link](https://download.java.net/java/early_access/jdk21/docs/api/new-list.html) here.

## Finally

What did you think? I hardly ever write apps in Java anymore, mostly in Kotlin, and new APIs don't affect my code that much, but I still get excited when a new version of Java is released. In particular, APIs such as Virtual Thread can be used in Kotlin, and it would be great to think that the performance of middleware created in Java such as Tomcat and Netty could be further improved by utilizing this. The other APIs that will be added have a different approach from Kotlin, so I thought it would be a great learning experience.

I'm currently using Java 17 at work, but I would like to use Java 21 as soon as it becomes available. Especially since Kotlin 2.0 will be released next year, I would like to take advantage of Java's new features to further improve Kotlin's build and performance.

See you soon!
