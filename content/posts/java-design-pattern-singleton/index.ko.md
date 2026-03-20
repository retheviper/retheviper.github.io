---
title: "디자인 패턴: 싱글턴"
date: 2019-08-19
categories:
  - java
image: "../../images/java.webp"
tags:
  - design pattern
  - java
translationKey: "posts/java-design-pattern-singleton"
---

예전부터 PC를 쓰면서 늘 문제였던 것은 메모리였던 기억이 있습니다. 제가 처음 PC를 접한 것은 아버지가 일하실 때 쓰던 기기였는데, 당시 OS는 DOS였고 게임을 하려면 매번 메모리 설정을 바꿔야 했습니다. 그때는 그게 불편하다고 느끼기보다, 그냥 게임만 실행되면 좋다고 생각했습니다.

하지만 시간이 지나 대학에서 발표 자료를 만들다 보니, 메모리가 충분하지 않으면 멀티태스킹이 꽤 힘들다는 걸 실감했습니다. 요즘은 성능 업그레이드의 체감이 가장 큰 부품으로 SSD를 많이 말하지만, 그건 CPU와 메모리를 어느 정도 안정적으로 확보할 수 있는 시대가 됐기 때문이라고 생각합니다. 예전에는 메모리가 부족하면 그저 느리다는 말밖에 할 수 없었습니다.

프로그램을 만드는 입장이 되니 메모리 문제는 더 현실적으로 다가왔습니다. 하나의 시스템을 구축하고 여러 사용자가 그 시스템을 쓰게 되면, 제한된 자원인 메모리가 모자랄 가능성은 지금도 있습니다. 오브젝트를 만들 때마다 남는 메모리는 계속 줄어들기 때문입니다.

그렇다면 메모리를 아끼는 방법은 결국 불필요한 객체 생성을 줄이는 것입니다. 그런 목적에 맞는 패턴이 이미 있었는데, 그게 이번 글의 주제인 Singleton 패턴입니다.

## Singleton 패턴이란

Singleton 패턴은 애플리케이션 안에서 인스턴스를 한 번만 만들고, 종료될 때까지 계속 사용하는 클래스를 만들기 위한 디자인 패턴입니다. JavaBean처럼 서로 다른 상태를 가진 인스턴스를 여러 개 만드는 경우와 달리, 이 패턴은 인스턴스가 하나뿐이므로 상태를 들고 있는 필드는 더 조심해서 다뤄야 합니다. 대신 어디서든 접근 가능한 공통 처리나 공용 자원 관리에 자주 쓰입니다.

이 패턴의 장점은 역시 메모리입니다. 전역 변수도 어디서나 접근할 수 있어서 비슷해 보이지만, 전역 변수는 필요 없는데도 계속 메모리에 남아 있을 수 있습니다. Singleton은 필요할 때 만들고, 필요 없으면 아예 만들지 않는 식으로 조절할 수 있습니다. 그래서 메모리를 조금 더 아낄 수 있습니다.

실무에서는 주로 유틸리티 클래스에 가까운 형태로 Singleton을 쓰는 경우가 많았습니다. 같은 처리를 반복하는데 매번 인스턴스를 새로 만들면 메모리도 아깝고 코드도 불필요하게 길어집니다. 데이터를 담는 JavaBean과 처리를 맡는 Singleton으로 역할을 나누면 코드도 단순해지고 메모리 사용도 줄일 수 있습니다.

## 전통적인 Singleton

이제 Singleton 클래스를 어떻게 만드는지 보겠습니다. Singleton도 구현 방식이 여러 가지지만, 먼저 고전적인 형태부터 살펴보겠습니다.

목적은 인스턴스를 하나로 제한하는 것이므로, 외부에서 이미 만들어진 인스턴스에 접근은 하되 마음대로 새로 만들 수는 없게 해야 합니다. 그러려면 생성자 접근을 막아야 합니다.

```java
// 클래스는 public으로 열어 외부에서 접근 가능하게 한다
public class SingletonClass {

    // 생성자는 private으로 막는다
    private SingletonClass() {}
}
```

하지만 이것만으로는 부족합니다. 어디선가 인스턴스를 만들어야 하고, 그 시점도 외부에서 제어할 수 있어야 합니다. 그래서 private 생성자에 접근할 수 있는 정적 메서드를 제공합니다.

```java
public class SingletonClass {

    // 인스턴스를 보관할 정적 필드
    private static SingletonClass uniqueInstance;

    private SingletonClass() {}

    // 인스턴스를 돌려주는 메서드
    public static SingletonClass getInstance() {
        if (uniqueInstance == null) {
            uniqueInstance = new SingletonClass();
        }
        return uniqueInstance;
    }

    public void doSomething() {
        // 일반 메서드
    }
}
```

먼저 static 필드에 인스턴스를 저장합니다. 외부에서는 `getInstance()`로만 가져오게 됩니다. 선언만 해 두고 이 시점에 바로 만들지 않는 것은 전역 변수와 구분하기 위해서입니다.

그 다음 인스턴스가 없을 때만 접근 가능한 static 메서드를 만듭니다. 이 메서드에서 Singleton 인스턴스를 돌려줍니다. 인스턴스가 없으면 그때만 새로 만들면 됩니다.

```java
// 인스턴스 가져오기
SingletonClass singletonInstance = SingletonClass.getInstance();

// 인스턴스 메서드 사용
singletonInstance.doSomething();
```

이제 어디서나 같은 인스턴스를 쓰는 Singleton 클래스가 만들어졌습니다.

## 전통적인 Singleton의 문제

멀티스레드를 고려하지 않아도 되는 환경이라면 괜찮지만, 현실의 프로그램은 그렇지 않습니다. 특히 여러 사용자가 같은 시스템을 동시에 쓸 수 있는 서비스에서는 같은 클래스를 여러 스레드가 요청할 수 있습니다.

클래스 내부가 복잡하거나 인스턴스 생성이 오래 걸리면, 거의 동시에 요청된 두 스레드가 각자 인스턴스를 만들려는 상황이 생길 수 있습니다. 그러면 Singleton 설계가 깨지고 예기치 못한 예외가 발생할 수 있습니다.

이 문제를 해결하는 방법은 몇 가지 있지만, 각각 단점이 있습니다.

## 멀티스레드 대응

스레드 세이프한 Singleton을 만드는 대표적인 방법은 다음과 같습니다.

1. 인스턴스 생성을 동기화한다
2. Double-Checked Locking을 쓴다
3. JVM 클래스 로더에 맡긴다

먼저 가장 단순한 방법은 `getInstance()`에 `synchronized`를 붙이는 것입니다.

```java
public class SingletonClass {

    private static SingletonClass uniqueInstance;

    private SingletonClass() {}

    // 인스턴스를 제공하는 메서드를 동기화한다
    public static synchronized SingletonClass getInstance() {
        if (uniqueInstance == null) {
            uniqueInstance = new SingletonClass();
        }
        return uniqueInstance;
    }
}
```

다만 `synchronized`는 성능이 문제입니다. 경우에 따라 처리 속도가 크게 느려질 수 있어서, 성능이 중요한 멀티스레드 환경에서는 그리 좋지 않습니다.

다음 방법은 Double-Checked Locking입니다. 인스턴스가 없을 때만 동기화합니다.

```java
public class SingletonClass {

    // 안정성을 위해 volatile 사용
    private volatile static SingletonClass uniqueInstance;

    private SingletonClass() {}

    // 두 번 확인하는 방식
    public static SingletonClass getInstance() {
        if (uniqueInstance == null) {
            synchronized (SingletonClass.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new SingletonClass();
                }
            }
        }
        return uniqueInstance;
    }
}
```

`volatile`을 쓰는 이유는 값이 CPU 캐시 메모리에만 머무르는 상황을 막기 위해서입니다. 데이터를 시스템 메모리까지 올려 두면 인스턴스가 생성됐는지 여부를 더 안정적으로 판단할 수 있습니다. 그래도 한 번은 동기화 비용을 치러야 한다는 점은 남습니다.

마지막 방법은 JVM이 시작될 때 인스턴스를 바로 만드는 방식입니다.

```java
public class SingletonClass {

    // 필드에서 바로 인스턴스를 만든다
    private static SingletonClass uniqueInstance = new SingletonClass();

    private SingletonClass() {}

    // 체크할 필요가 없어졌다
    public static SingletonClass getInstance() {
        return uniqueInstance;
    }
}
```

이 방식은 클래스가 로드될 때 JVM이 바로 인스턴스를 만들기 때문에 어떤 스레드에서도 정적 필드 초기화 문제를 일으키지 않습니다. 다만 필요하지 않아도 메모리에 계속 남는다는 단점이 있습니다. 전역 변수처럼 보일 수 있지만, 인스턴스가 하나뿐이라는 점이 차이입니다. 애초에 전역 변수는 어디에 뭐가 들어 있는지 파악하기 어려운 경우가 많으니, 남발하지 않는 편이 좋습니다.

## 전부 static으로 두면 안 되는가

물론 그런 방법도 있습니다. 하지만 초기화 과정이 매우 단순할 때만 유효합니다. 클래스 구조가 단순하고 메서드가 외부에서 받은 데이터를 처리해서 돌려주는 역할뿐이라면 가능하겠지만, 기능 확장을 생각하면 좋은 해법은 아닙니다.

## 마지막으로

Singleton 패턴은 널리 쓰이고, 분명 매력적인 설계 방식입니다. 하지만 멀티스레드 문제를 피하기 위한 고민이 필요하고, 인스턴스가 하나뿐이라 필드 처리에도 신경 써야 합니다. 한 스레드에서 넣은 데이터를 다른 스레드가 건드리면 문제가 생길 수 있기 때문입니다.

OOP 원칙인 "한 클래스는 하나의 책임만 가진다"는 관점에서도 Singleton은 조금 애매합니다. 처리 담당과 인스턴스 관리라는 두 책임을 동시에 가지기 때문입니다. 생성자가 `private`이라 하위 클래스를 만들기 어렵다는 점도 한계입니다.

그래도 인스턴스를 하나만 유지해야 하는 요구가 분명하다면 Singleton은 여전히 유효한 선택지입니다. 중요한 건 습관적으로 쓰는 것이 아니라, 정말 하나만 존재해야 하는 객체인지 먼저 판단하는 일이라고 생각합니다.
