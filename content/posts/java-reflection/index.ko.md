---
title: "Reflection과 Generic 활용하기"
date: 2019-07-30
categories:
  - java
image: "../../images/java.webp"
tags:
  - reflection
  - generic
  - java
translationKey: "posts/java-reflection"
---

이번 작업에서 배운 것은, 내 코드를 다른 사람이 라이브러리처럼 사용할 때 어떻게 설계하고 구현해야 하는가였습니다. 특정 기능을 하는 스크립트나, 최종 사용자가 쓰는 UI와 그 안의 콘텐츠를 처리하는 로직만 다뤄 온 저에게는 꽤 새로운 경험이었습니다. 지금까지는 제가 맡은 기능을 구현하고, 그걸 최적화하면 끝이었습니다. 하지만 라이브러리는 기본적으로 코드를 다룰 수 있는 사람이 쓰는 것이기 때문에 설계 자체가 다릅니다.

또 하나 어려웠던 점은 유연성을 확보하는 일이었습니다. 예를 들어 데이터를 받아 처리할 때, 이전에는 제가 만든 Bean에 데이터가 매핑된다는 전제를 두고 있었습니다. 그런데 지금의 일에서는 "어떤 Bean이 들어올지 모르니 그에 맞춰 처리해야 한다"는 요구가 있었습니다.

그래서 먼저 클래스나 인스턴스를 인수로 받는 방법부터 알아야 했습니다. 처음에는 `Object`를 그대로 쓰려고 했지만, 찾아보니 `Generic`을 쓰는 편이 더 적절해 보여 그 방식으로 정리했습니다.

그다음에는 Generic으로 받은 Bean을 어떻게 다룰지 고민했습니다. 제가 설계한 Bean만 쓰는 경우라면 필드의 데이터 타입도 알고 있고, `Getter`/`Setter`로 값을 주고받는 것도 쉽습니다. 하지만 내가 직접 만든 것이 아니라면 필드 타입도, 어떤 메서드가 있는지도 알기 어렵습니다. 그때 찾은 해답이 `Reflection`이었습니다.

이번 글에서는 이 두 가지를 이용해 어떻게 "내가 만들지 않은 Bean을 처리했는지" 정리해 보겠습니다.

## Generic

Generic은 말 그대로 "데이터 타입을 일반화"하는 것입니다. 총칭형이라고도 부릅니다. 클래스 안에서 사용할 타입을 외부에서 정할 때 쓰는 방식입니다. 이전에는 여러 데이터 타입이나 객체를 다뤄야 할 때 주로 `Object`를 사용했습니다. Java에서는 일부 Primitive를 제외하면 대부분의 데이터가 객체로 취급되기 때문에, 가장 상위 타입인 `Object`로 받아도 큰 문제는 없었습니다.

하지만 `Object`를 인수로 받으면 메서드 안에서 어떤 필드를 쓸 수 있는지 알기 어렵습니다. `Getter`나 `Setter`도 바로 호출할 수 없고, 결국 다시 직접 확인해야 합니다. 그래서 객체 자체를 받는 것만으로는 부족하고, 인스턴스 내부를 직접 들여다볼 필요가 있습니다.

여기서 Generic을 쓰면, `Object`와 비슷하게 어떤 인스턴스나 클래스든 받을 수 있으면서도 처음에 구체적인 타입이 정해지기 때문에 캐스팅이 줄어듭니다. 그만큼 코드가 안전해지고, 의도도 더 분명해집니다. 그리고 실제 구조를 모르는 대상에 대해서는 Reflection으로 클래스나 인스턴스 내부를 확인하면 메서드와 필드도 가져올 수 있습니다.

먼저 인스턴스를 인수로 받는 방법부터 보겠습니다. 인수에 `T`[^1]를 지정하면 Generic 타입의 인수를 받을 수 있습니다. 즉 인스턴스 자체를 인수로 넘기는 방식입니다. 다만 `T`를 쓸 때는 메서드 반환형 앞에 `<T>`를 함께 선언해야 합니다.

```java
// Bean 인스턴스를 받는 메서드
public <T> boolean isBean(T parameter) {
    // ... 어떤 처리
}

// 사용 예
BeanObject beanObject = new BeanObject();

if (isBean(beanObject)) {
    // ... 어떤 처리
}
```

List 안에도 Generic을 쓸 수 있습니다. 예를 들어 다음처럼 쓰면 어떤 타입이든 담을 수 있습니다.

```java
List<T> list = new ArrayList<>();
```

인스턴스가 아니라 클래스 자체를 Generic으로 받으려면 이렇게 작성합니다.

```java
// Bean 클래스를 받는 메서드
public boolean isBean(Class<?> parameter) {
    // ... 어떤 처리
}

// 상속도 가능하다(제한적)
public boolean isStringBean(Class<? extends String> parameter) {
    // ... 어떤 처리
}

// 사용 예
if (isBean(BeanObject.class)) {
    // ... 어떤 처리
}
```

이제 Generic 인수를 전달할 준비는 끝났습니다. 다음은 이렇게 받은 인수를 다루는 Reflection을 살펴보겠습니다.

## Reflection

클래스의 구체적인 타입을 몰라도 메서드, 생성자, 필드 등에 접근할 수 있게 해 주는 API를 Reflection이라고 합니다. 가져온 메서드, 생성자, 필드는 그대로 사용하거나 값을 읽고, 붙어 있는 어노테이션을 확인하는 등 일반적인 클래스에서 할 수 있는 작업을 대부분 수행할 수 있습니다.

이제 이 Reflection을 실제로 어떻게 쓰는지 코드로 보겠습니다. 먼저 인스턴스에서 클래스를 가져오고, 다시 그 클래스에서 생성자를 다루는 예시입니다.

```java
public <T> boolean isBean(T object) {

    // 인스턴스에서 클래스를 가져온다
    Class<?> objectClass = object.getClass();
    // 클래스에서 다시 인스턴스를 생성한다
    Object instance = objectClass.newInstance();

    // 클래스의 패키지명을 가져온다
    String packageName = objectClass.getPackage().getName();
    // 패키지를 포함한 클래스명을 가져온다
    String classNamePackageInvolved = objectClass.getName();
    // 클래스명만 가져온다
    String className = objectClass.getSimpleName();

    // public 생성자들을 배열로 가져온다
    Constructor[] constructors = objectClass.getConstructors();
    // 특정 public 생성자를 가져온다
    Constructor constructor = objectClass.getConstructor(parameter1, parameter2, ...);
    // 가져온 생성자로 인스턴스를 만든다
    Object instance2 = constructor.newInstance();
}
```

클래스 자체를 다룰 수 있으니, 내부 구조만 알면 새 인스턴스를 만들어 쓰는 것도 가능합니다. 다음은 필드를 가져오는 방법입니다.

## Field의 취득

```java
public boolean isBean(T object) {

    Class<?> objectClass = object.getClass();

    // public 필드들을 배열로 가져온다
    Field[] fields = objectClass.getFields();
    // 특정 public 필드를 가져온다
    Field field = objectClass.getField("필드명");
    // 모든 필드를 배열로 가져온다(public 이외도 포함)
    Field[] declaredFields = objectClass.getDeclaredFields();
    // 특정 필드를 가져온다(public 이외도 포함)
    Field declaredField = objectClass.getDeclaredField("필드명");

    // Field로 할 수 있는 일
    // 필드에 값을 설정한다
    field.set(object, parameter);
    // 필드 값을 가져온다
    String fieldValue = (String) field.get(object);
    // 필드의 어노테이션을 가져온다
    Annotation[] annotations = field.getAnnotations();
}
```

여기서 주의할 점은, 필드를 가져올 때는 클래스로부터 가져오지만 실제 값을 넣거나 읽을 때는 대상이 되는 인스턴스를 사용한다는 것입니다. 클래스는 설계도이고, 인스턴스는 그 설계도로 만든 결과물이라는 점이 더 분명해집니다. Reflection을 쓰는 장점이 이런 부분에서도 드러납니다.

다음은 메서드를 살펴보겠습니다.

## Method의 취득

메서드도 필드와 크게 다르지 않습니다. 클래스로부터 메서드를 가져오고, 그 메서드로 여러 작업을 할 수 있습니다. 예제 코드는 아래와 같습니다.

```java
public boolean isBean(T object) {

    Class<?> objectClass = object.getClass();

    // public 메서드들을 배열로 가져온다
    Method[] methods = objectClass.getMethods();
    // 특정 public 메서드를 가져온다
    Method method = objectClass.getMethod("메서드명", parameter1, parameter2, ...);
    // 모든 메서드를 배열로 가져온다(public 이외도 포함)
    Method[] declaredMethods = objectClass.getDeclaredMethods();
    // 특정 메서드를 가져온다(public 이외도 포함)
    Method declaredMethod = objectClass.getDeclaredMethod("메서드명", parameter1, parameter2, ...);

    // Method로 할 수 있는 일
    // 메서드 이름을 가져온다
    String methodName = method.getName();
    // 메서드를 실행한다
    Object methodInvoked = method.invoke(object);
    // 매개변수의 어노테이션을 가져온다
    Annotation[] parameterAnnotations = method.getParameterAnnotations();
    // 매개변수를 가져온다
    Parameter[] parameters = method.getParameters();
}
```

어노테이션은 클래스에서도, 필드에서도 가져올 수 있습니다. 메서드는 여기에 더해 매개변수의 어노테이션까지 확인할 수 있다는 점이 특징입니다. 물론 `Parameter` 클래스로 매개변수 이름을 가져오는 등 다른 작업도 가능합니다.

## 결론

결국 핵심은 "모르는 타입의 객체를 어떻게 안전하게 다룰 것인가"였습니다. 메서드나 생성자를 직접 조작하지 않더라도, 인수로 받은 인스턴스에서 필드를 꺼내 값을 `Object`로 받고, `instanceof`로 분기한 뒤 캐스팅해서 처리하는 정도만으로도 필요한 작업은 충분히 마무리할 수 있었습니다. 간단히 쓰면 아래와 비슷합니다.

```java
// 클래스를 모르는 Bean을 인수로 받는 메서드
public <T> void processSomething(T bean) {
    // 여러 타입의 값을 담을 변수
    String stringObject;
    Integer intObject;

    // 클래스와 필드를 가져온다
    Class<?> beanClass = bean.getClass();
    Field[] beanFields = beanClass.getDeclaredFields();

    // 각 필드를 순회하면서 값을 읽고, 타입에 따라 분기한다
    for (Field field : beanFields) {
        // private 필드에도 접근할 수 있도록 한다
        if (!field.canAccess(bean)) {
            field.setAccessible(true);
        }

        // 필드 값을 가져와 타입을 확인한다
        Object value = field.get(bean);
        if (value instanceof String) {
            stringObject = (String) value;
        } else if (value instanceof Integer) {
            intObject = (Integer) value;
        }
    }
}
```

어떤가요. 방법만 알면 생각보다 어렵지 않습니다. 더 응용하면 인스턴스의 일부 값만 수정해서 다시 돌려주는 식의 동작도 가능할 것 같습니다. 활용할 수 있는 범위가 꽤 넓습니다.

## 마지막으로

Reflection은 무턱대고 쓰면 복잡해 보이지만, 클래스와 인스턴스를 런타임에 다뤄야 하는 상황에서는 꽤 강력한 도구입니다. Generic과 함께 이해해 두면 라이브러리처럼 범용 코드를 만들 때 선택지가 훨씬 넓어집니다.

[^1]: Type을 뜻합니다. 다만 E(Element), K(Key), N(Number), V(Value)도 자주 쓰입니다.
