---
title: "Go를 써 본 첫인상"
date: 2021-03-21
categories: 
  - go
image: "../../images/go.webp"
tags:
  - go
  - java
  - kotlin
  - exception
translationKey: "posts/go-first-impression"
---

이 블로그에서는 꽤 갑작스러운 주제일 수 있지만, 이직한 뒤 업무상 Go를 조금 만지게 됐습니다. 예전부터 Go나 Rust를 한 번쯤 제대로 접해 보고 싶다고 생각하긴 했지만, 막상 한 번도 다뤄 본 적 없는 언어로 작성된 애플리케이션을 수정해야 한다고 생각하면 아무래도 조금 겁이 납니다. 그래서 최소한 Go가 어떤 언어인지는 알아 두는 편이 낫겠다고 느꼈고, 이번 글에서는 Go를 잠깐 써 보며 느낀 점을 Java 프로그래머의 관점에서 정리해 보려고 합니다.

Go의 특징을 이야기할 때 GC가 있다거나 VM이 없다는 점을 많이 언급하곤 합니다. 하지만 실제로 코드를 쓰는 입장에서 그런 차이는 체감이 아주 직접적이지는 않았습니다. "VM이 없으니 어느 정도는 빠르겠지" 정도의 인상은 있어도, 그보다 더 크게 느껴지는 차이는 따로 있었습니다.

실제 업무에서 중요한 건, 결국 그 언어로 코드를 어떻게 써야 하느냐입니다. 루프나 조건문은 어떻게 작성하는지, 지금까지 익숙했던 습관대로 코드를 써도 괜찮은지, 어디를 조심해야 하는지 같은 부분이 더 신경 쓰이게 됩니다. 이번 글은 그런 관점에서, 정말 조금만 Go를 만져 본 사람의 첫인상 정도로 봐 주시면 좋겠습니다.

## 사고방식부터 바꿔야 할지도 모른다

Go를 조금 만져 보면서 가장 먼저 느낀 것은, 아주 기본적인 부분부터 Java와는 접근 방식이 꽤 다를 수 있다는 점이었습니다. 저는 Java 말고도 Python, JavaScript, TypeScript, Kotlin 정도는 접해 봤는데, JavaScript와 TypeScript는 Java의 감각을 어느 정도 유지한 채로도 쓸 수 있었고, Kotlin 역시 기본적으로는 Java를 더 간결하게 만든 언어라는 느낌이 강했습니다. Python은 문법적으로 많이 다르지만, 오히려 "하고 싶은 걸 문법에 크게 막히지 않고 쓸 수 있다"는 쪽에 가까워서 문법 차이 자체가 크게 부담되지는 않았습니다.

그런데 Go는 조금 다르게 느껴졌습니다. 단순히 문법만 조금 다른 것이 아니라, 기능과 철학 자체가 Java와 꽤 다르기 때문입니다. 이런 차이는 결국 "Java 코드를 조금 바꾼 수준"으로는 좋은 Go 코드를 쓰기 어렵다는 뜻이기도 합니다. 그래서 어쩌면 문법보다 먼저 사고방식을 바꿔야 하는 언어일지도 모르겠다는 생각이 들었습니다.

## 비슷해 보이지만 꽤 다르다

우선 눈에 띄는 건 문법입니다. 물론 큰 틀은 흔히 말하는 C 계열 언어와 크게 다르지 않습니다. 하지만 Java와 비교하면, 구조적인 부분 외에도 세세한 지점에서 체감 차이가 꽤 큽니다.

예를 들어 `:=` 같은 문법이 있고, `if` 조건식에 괄호를 쓰지 않으며, `import`를 문자열로 적고, 클래스나 `public` / `private` 같은 키워드가 아예 없습니다. 그래서 단순히 타이핑 감각만 달라지는 것이 아니라, 패키지 구조나 애플리케이션 설계 방식까지 Java나 Kotlin과는 다른 접근이 필요할 수 있겠다는 생각이 들 정도였습니다.

예를 들어 다음처럼 숫자가 짝수인지 홀수인지 판별하는 Java 코드가 있다고 해 보겠습니다.

```java
package simple.math;

public Class Calculator {

    public void judge(int number) {
        boolean result = number %2 == 0;
        if (result) {
            System.out.println(number + "는 짝수입니다");
        } else {
            System.out.println(number + "는 홀수입니다");
        }
    }
}
```

이걸 Go로 바꾸면 대략 이런 느낌이 됩니다.

```go
package simple.math

import "fmt"

func Judge(number int) {
    condition := number % 2 == 0
    if condition {
        fmt.Println(number, "는 짝수입니다")
    } else {
        fmt.Println(number,"는 홀수입니다")
    }
}
```

겉보기에는 크게 다르지 않아 보여도, 실제로는 신경 써야 할 차이가 많습니다. IDE가 아무리 좋아도 언어 사양 자체를 모르면 올바른 방향으로 도와주기 어렵기 때문입니다.

예를 들어 import가 여러 개면 이렇게 써야 합니다.

```go
import (
    "fmt"
    "math"
)
```

Python처럼 alias를 자유롭게 붙이는 감각과도 조금 다르고, Java처럼 빌드 도구를 통해 의존성을 다루는 감각과도 다릅니다. Go는 언어와 패키지 관리가 더 강하게 연결돼 있어서, 아래처럼 GitHub 패키지를 직접 import하는 흐름도 자연스럽습니다.

```go
import "github.com/gin-gonic/gin"
```

또 `:=`는 함수 내부에서만 쓸 수 있기 때문에, 패키지 레벨에서는 `var`를 써야 한다는 점도 이해해야 합니다. 반대로 함수 매개변수는 `var` 없이 타입만 적습니다. Java처럼 언제나 타입을 명시하는 언어에 익숙하다 보면 이런 차이는 생각보다 자주 걸립니다.

즉, Go의 기본 작법을 이해하지 않은 채 Java 감각으로만 코드를 쓰려 하면 꽤 힘들 수 있습니다.

## 대문자에는 의미가 있다

회사마다 세부 규칙은 다르겠지만, 제가 지금까지 주로 봐 온 규칙은 대체로 이랬습니다.

- 클래스와 인터페이스는 PascalCase
- 필드, 메서드, 변수, 인수는 camelCase

Python을 쓸 때는 snake_case, URL은 kebab-case 같은 변형이 있었지만, 기본적으로는 사람이 정한 스타일 규칙이라는 인식이 강했습니다. 즉, 지키는 편이 좋지만, 그 자체가 언어 기능은 아니라는 뜻입니다.

그런데 Go에서는 이름의 첫 글자가 대문자인지 소문자인지가 실제 의미를 가집니다. 정확히 말하면, `public` / `private` 역할을 이 규칙이 대신합니다. 다른 패키지에서 참조 가능한 이름은 대문자로 시작하고, 그렇지 않은 것은 소문자로 시작합니다.

예를 들어 [A Tour of Go](https://tour.golang.org)의 예제를 보면 다음과 같습니다.

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    fmt.Println(math.pi)
}
```

겉보기엔 문제없어 보이지만, 실행하면 아래와 같은 에러가 납니다.

```bash
./prog.go:9:14: cannot refer to unexported name math.pi
```

즉, 외부에서 참조할 수 없는 이름이라는 뜻입니다. 그래서 올바른 코드는 이렇게 바뀌어야 합니다.

```go
func main() {
    fmt.Println(math.Pi)
}
```

이처럼 대문자로 시작해 외부에서 참조 가능한 이름을 [Exported Names](https://go-tour-jp.appspot.com/basics/3)라고 합니다. Go에는 클래스가 없기 때문에, 패키지를 import한 뒤 그 안에서 export된 항목만 사용하게 됩니다. 클래스 인스턴스를 만들고 내부 메서드를 호출하는 Java식 감각과는 확실히 다르게 느껴집니다.

## 포인터의 존재

프로그래밍 언어에서 포인터가 있는지 없는지는 생각보다 코드 감각에 큰 영향을 줍니다. Java나 Kotlin처럼 포인터를 직접 다루지 않는 언어에 익숙한 입장에서는, 이 차이만으로도 꽤 다른 세계처럼 보입니다.

물론 Go는 GC가 있기 때문에 C나 C++처럼 메모리 관리를 완전히 직접 해야 하는 것은 아닙니다. 그래도 포인터를 언어 차원에서 직접 다룰 수 있다는 사실 자체가 체감에 영향을 줍니다.

Java에서도 `public static`이나 Spring의 DI, Kotlin의 `companion object`처럼 전역에 가까운 접근을 제공하는 구조는 있습니다. 하지만 그런 것들은 대개 상수이거나, 동일한 동작을 공유하는 객체처럼 쓰이는 경우가 많습니다. 포인터처럼 값을 직접 가리키고, 그 값을 바꾸는 것까지 자연스럽게 염두에 두는 구조와는 꽤 다릅니다.

아직 저는 포인터를 본격적으로 다루는 언어 경험이 많지 않고, Go에서도 포인터를 적극적으로 활용하는 코드를 충분히 써 본 것은 아닙니다. 그래서 이 부분은 깊게 말하기보다, "포인터가 없는 언어에 익숙한 사람에게는 익숙해지기까지 시간이 꽤 걸릴 수 있겠다" 정도의 인상으로 남았습니다.

## 예외 처리가 독특하다

Go 코드를 처음 볼 때 가장 눈에 들어오는 부분 중 하나는 예외 처리 방식입니다. 제가 경험한 Java, Python, JavaScript, TypeScript, Kotlin은 모두 `try-catch` 계열 문법을 가지고 있었습니다. 언어마다 세부 차이는 있어도, 기본적으로 예외가 날 수 있는 코드를 블록으로 감싸고 처리한다는 발상은 비슷합니다.

예를 들어 Java에서는 이런 식으로 쓰는 것이 일반적입니다.

```java
public static void main(String[] args) {
    int result = 0;
    try {
        result = divide(1, 0);
    } catch (Exception e) {
        if (e instanceof ArithmeticException) {
            System.out.println("0으로 나눌 수 없습니다");
            return;
        }
    }
    System.out.println(result);
}

private static int divide(int numerator, int denominator) {
    return numerator / denominator;
}
```

하지만 Go에는 이런 방식이 없습니다. 대신 함수가 "기대한 값"과 "발생한 에러"를 함께 반환하고, 호출한 쪽에서 `err != nil`인지 확인해 처리하는 방식이 일반적입니다.

같은 코드를 Go 스타일로 바꾸면 대략 이런 형태가 됩니다.

```go
func main() {
    result, err := divide(1, 0)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(result)
}

func divide(numerator int32, denominator int32) (int32, error) {
    if (denominator != 0) {
        return numerator / denominator, nil
    } else {
        return 0, errors.New("0으로 나눌 수 없습니다")
    }
}
```

이 방식의 장점은, 에러가 어디서 발생할 수 있는지가 더 명확하게 드러난다는 점입니다. `try-catch` 블록이 너무 넓어지면 어떤 줄이 실제 예외 지점인지 흐려질 때가 있는데, Go에서는 적어도 함수 호출 직후에 확인한다는 패턴이 분명합니다.

그런 점에서는 "에러 처리와 로직을 더 명확히 분리할 수 있다"는 장점이 있다고 생각합니다.

다만 반대로, 함수를 호출할 때마다 비슷한 에러 체크를 반복하게 되는 경우가 많다는 점은 조금 낯설었습니다. 예를 들어 아래 같은 코드는 Go에서 자주 보게 되는데, 솔직히 처음에는 꽤 장황하게 느껴졌습니다.

```go
// 여러 함수를 호출해 처리하는 함수
func doSomething() (string, error) {
    // 함수 1 호출
    result1, err1 := someFunction1()
    // 함수 1에서 에러가 발생하면 반환
    if err1 != nil {
        return "", err
    }
    // 함수 1이 성공하면 함수 2 호출
    result2, err2 := someFunction2(result1)
    // 함수 2에서 에러가 발생하면 반환
    if err2 != nil {
        return "", err
    }
    // 함수 2가 성공하면 함수 3 호출
    result3, err3 := someFunction3(result2)
    // 함수 3에서 에러가 발생하면 반환
    if err3 != nil {
        return "", err
    }
    // 함수 3이 성공하면 함수 4 호출
    result4, err4 := someFunction4(result3)
    // 함수 4에서 에러가 발생하면 반환
    if err4 != nil {
        return "", err
    }
    // ...계속
}
```

물론 Go 쪽에서는 이런 반복도 포함해 하나의 명확한 스타일이라고 볼 수 있겠지만, Java에 익숙한 사람 입장에서는 꽤 특이하게 느껴질 수 있습니다.

## 컴파일러가 꽤 엄격하다

컴파일 에러는 보통 IDE가 잘 알려 주기 때문에 크게 신경 쓰지 않게 되기 쉽습니다. 그런데 Go를 만지다 보면 의외로 강하게 느껴지는 것이 컴파일러의 엄격함입니다.

저는 평소 `jshell` 같은 인터랙티브 도구로 작은 코드를 자주 검증해 보곤 하는데, Go는 그런 흐름이 조금 다릅니다. 그래서 Vim으로 작성한 코드를 터미널에서 바로 돌려 보거나, [The Go Playground](https://play.golang.org/)를 쓰곤 했습니다. 그런데 이런 환경에서는 IDE만큼 친절한 지원이 없어서, 컴파일러의 엄격함이 더 잘 드러납니다.

예를 들어 아래 코드를 보겠습니다.

```go
package main

import (
    "fmt"
)

func main() {
}
```

이걸 실행하면 다음과 같은 에러가 납니다.

```bash
./prog.go:4:2: imported and not used: "fmt"
```

이번에는 이런 코드입니다.

```go
package main

import (
    "fmt"
)

func main() {
    var result string
    fmt.Println("this is test program")
}
```

이 경우에는 다음 에러가 나옵니다.

```bash
./prog.go:8:9: result declared but not used
```

즉, 사용하지 않는 import나 변수는 그냥 경고가 아니라 곧바로 에러가 됩니다. 이런 점은 다른 언어보다 훨씬 엄격하다고 느껴졌습니다.

그래서 플러그인 없는 Vim 같은 환경에서 작업할 때는 특히 더 조심해야 합니다. IDE를 쓰더라도 자동 정리나 lint 설정을 잘 맞춰 두지 않으면 꽤 귀찮게 느껴질 수 있습니다.

## 마지막으로

이 외에도 세세한 차이는 훨씬 더 많겠지만, 지금 시점에서 이야기할 수 있는 첫인상은 이 정도입니다. 사실 여기서 적은 Go의 특징들도 결국은 "익숙해지면 별것 아닐 수 있는 것들"일지도 모릅니다. 다만 문제는, 이미 다른 언어에 충분히 익숙해져 있는 사람에게는 바로 그 "익숙해지는 과정"이 생각보다 어렵다는 점이겠죠.

사람의 언어로 비유하면, 같은 계열 언어를 쓰는 사람이 비슷한 언어를 배우는 것은 비교적 쉽다고들 합니다. 반대로 어휘, 문자, 문장 구조가 모두 다른 언어는 훨씬 어렵게 느껴집니다. 프로그래밍 언어도 결국은 사람이 이해하고 쓰는 언어라는 점에서 크게 다르지 않다고 생각합니다. 그래서 새 언어가 내가 이미 익숙한 언어와 닮아 있을수록 배우기 쉽고, 그렇지 않을수록 더 어렵게 느껴지는 것이 아닐까 싶습니다.

그런 의미에서 Java에서 Go로 넘어가는 일은, 얼핏 보면 쉬울 것 같지만 막상 해 보면 꽤 어렵게 느껴질 수도 있습니다.

물론 세상에는 "Java보다 Go가 더 단순하고 편했다"고 느끼는 사람도 많이 있을 것입니다. 어쩌면 제가 아직 충분히 익숙해지지 못한 것뿐일지도 모르겠습니다.
