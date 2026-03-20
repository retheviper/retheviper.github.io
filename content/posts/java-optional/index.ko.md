---
title: "널 체크 지옥에서 벗어나기"
date: 2020-01-05
categories: 
  - java
image: "../../images/java.webp"
tags:
  - optional
  - java
translationKey: "posts/java-optional"
---

Java로 애플리케이션을 만들다 보면 가장 자주 마주치는 예외는 결국 `NullPointerException`일 것입니다. 처음 프로그래밍을 배우는 사람에게는 `null`과 빈 값의 차이부터 헷갈리기 쉽고, `null`을 이해한 뒤에도 예상치 못한 곳에서 NPE를 만나 고생하는 경우가 많습니다. 어떤 의미에서는 Java 개발이 NPE와의 싸움처럼 느껴지기도 합니다.

그래서 이번에는 NPE를 줄이는 방법을 차근차근 살펴보겠습니다. `null`을 안전하게 다루는 방식으로 `Optional`을 보려 합니다.

## 널 체크만으로 충분한가

어떤 학생의 이름을 가져오는 메서드가 있다고 가정해 보겠습니다. 데이터 객체를 인수로 받아서 학교, 학년, 반, 학생 정보를 차례로 따라가 마지막에 학생 이름을 문자열로 돌려주는 함수입니다. 코드로 쓰면 대략 이런 형태입니다.

```java
public String getStudentName(Data data) {
    School school = data.getSchool();
    Grade grade = school.getGrade();
    ClassName className = grade.getClassName();
    Student student = className.getStudent();
    return student.getName();
}
```

조금 더 줄이면 이렇게도 쓸 수 있습니다.

```java
public String getStudentName(Data data) {
    return data.getSchool().getGrade().getClassName().getStudent().getName();
}
```

의도만 보면 꽤 단순한 코드입니다. 하지만 실제로는 어디서든 예외가 날 수 있습니다. 학생 이름이 `null`일 수도 있고, 학생이나 반, 학년, 학교, 혹은 인수 자체가 `null`일 수도 있습니다. 이런 식의 체인은 아주 쉽게 NPE를 만듭니다.

대안은 각 단계에서 `null`을 확인하고, `null`이 아니면 다음 단계로 진행하는 방식입니다. `null`일 때는 적절한 기본값을 돌려주거나 별도 처리를 하면 됩니다. 하지만 이렇게 만들면 코드가 점점 길어집니다.

```java
public String getStudentName(Data data) {
    if (data != null) {
        School school = data.getSchool();
        if (school != null) {
            Grade grade = school.getGrade();
            if (grade != null) {
                ClassRoom classRoom = grade.getClassRoom();
                if (classRoom != null) {
                    Student student = classRoom.getStudent();
                    if (student != null) {
                        String studentName = student.getName();
                        if (studentName != null) {
                            return studentName;
                        }
                    }
                }
            }
        }
    }
    return "Sato"; // Default value
}
```

이 코드는 중첩이 너무 깊어서 읽기 어렵습니다. 한 단계라도 빠지면 문제가 생길 수 있고, 수정하기도 쉽지 않습니다. 메서드의 목적은 단순히 학생 이름을 얻는 것이었는데, 이제는 `null` 체크가 로직을 가려 버립니다.

중첩을 줄이려고 `if`를 밖으로 빼도 결과는 크게 달라지지 않습니다.

```java
public String getStudentName(Data data) {
    if (data == null) {
        return "Sato"; // Default value
    }
    
    School school = data.getSchool();
    if (school == null) {
        return "Sato"; // Default value
    }

    Grade grade = school.getGrade();
    if (grade == null) {
        return "Sato"; // Default value
    }
    
    ClassRoom classRoom = grade.getClassRoom();
    if (classRoom == null) {
        return "Sato"; // Default value
    }

    Student student = classRoom.getStudent();
    if (student == null) {
        return "Sato"; // Default value
    }

    String studentName = student.getName();
    if (studentName == null) {
        return "Sato"; // Default value
    }

    return studentName;
}
```

이 방식은 중첩은 줄었지만, 대신 `return`이 너무 많아집니다. 결국 읽기 쉬운 코드라고 보기도 어렵습니다.

이럴 때 Java가 제공하는 도구가 바로 이번 주제인 `Optional`입니다.

## Optional 도입

요즘 언어들은 `null` 때문에 생기는 문제를 아예 줄이도록 설계하는 경우가 많습니다. 처음부터 `null`을 허용하지 않거나, `null`을 다루는 전용 API를 제공하는 식입니다. Kotlin이나 Swift가 그런 계열이고, Java도 이런 흐름에 맞춰 Java 8에서 `Optional`을 도입했습니다.

`Optional`은 `null`이 될 수 있는 값을 감싸서 다루는 방법입니다. 함수형 언어의 영향을 받은 설계라고 볼 수 있습니다. 기존 Java 스타일과는 꽤 다르기 때문에 처음에는 낯설게 느껴질 수 있습니다. 하지만 그만큼 `null` 관련 처리를 메서드 체인으로 표현할 수 있어, 코드를 더 의도적으로 보이게 할 수 있습니다.

그렇다면 이런 낯선 API를 왜 써야 할까요. 먼저 `Optional`이 어떤 특징을 갖는지 보겠습니다.

### 사용하기 쉬움

`map()`이나 `filter()` 같은 메서드는 Collection이나 Stream과 비슷합니다. Lambda도 함께 쓰기 때문에, 이쪽에 익숙하다면 금방 적응할 수 있습니다.

Optional을 잘 쓰려면 메서드 체이닝과 Lambda에 익숙해져야 합니다. 이전 글의 [함수형 인터페이스](../java-functional-interface)도 참고할 만합니다.

### 봐도 알 수 있음

Optional은 값을 감싸서 `null` 가능성을 드러내는 API입니다. 즉, 반환값만 봐도 "이 값은 비어 있을 수 있다"는 점을 알 수 있습니다. 그 자체로 코드의 의도가 더 분명해집니다.

### 가독성 향상

원래 목적은 `null` 체크지만, 이를 코드로 잘 표현하면 오히려 더 읽기 쉬워집니다. 중첩된 객체를 따라갈 때도 `Optional`을 연결해 처리할 수 있어서, 조건문을 여러 겹 쓰는 것보다 훨씬 단순합니다.

## Optional에서 널 체크를 바꿔 봅시다

이제 실제 코드로 `Optional`이 어떻게 `null` 체크를 대신하는지 보겠습니다. 앞서의 코드는 이렇게 바꿀 수 있습니다.

```java
public String getStudentName(Data data) {
    return Optional.ofNullable(data)
                    .map(Data::getSchool)
                    .map(School::getGrade)
                    .map(Grade::getClassRoom)
                    .map(ClassRoom::getStudent)
                    .map(Student::getName)
                    .orElse("Sato");
}
```

처음 보면 `Optional`이 정확히 어떤 역할을 하는지 조금 낯설 수 있지만, 코드 양이 크게 줄고 흐름이 훨씬 명확해집니다. `null` 체크를 생략한 것이 아니라, 표현 방식을 바꾼 것입니다. 이것이 `Optional`의 핵심입니다.

## Optional 메서드

`Optional`은 `java.util.Optional<T>`를 통해 값을 감싸고, 값이 있는지 없는지에 따라 여러 방식으로 처리할 수 있게 해줍니다. 자주 쓰는 메서드를 하나씩 보겠습니다.

### empty()

비어 있는 Optional을 만듭니다. 이름 그대로 내부에 값이 없는 상태입니다.

```java
Optional<String> emptyName = Optional.empty(); // 값이 없는 Optional
```

### get()

Optional에 담긴 값을 꺼낼 때 사용합니다.

```java
String value = "Sato";
Optional<String> name = Optional.of(value);

System.out.println(name.get()); // Sato
```

### of(T value)

값을 감싼 Optional을 만듭니다. 단, `null`을 넣으면 안 됩니다.

```java
String value = null;
Optional<String> name = Optional.of(value);

name.get(); // NullPointerException
```

### ofNullable(T value)

`of()`와 비슷하지만, 인수가 `null`이면 빈 Optional을 만듭니다.

```java
String value = null;
Optional<String> name = Optional.ofNullable(value);

name.orElse("Sato"); // "Sato"
```

### map(Function<? super T, ? extends U> mapper)

Collection이나 Stream의 `map()`과 비슷합니다. 중첩된 값을 안전하게 꺼내고 싶을 때 특히 유용합니다. `map()`의 결과는 다시 Optional로 감싸집니다.

```java
String name = "Sato";
Student sato = new Student(name);

Optional<Student> student = Optional.ofNullable(sato);
String nameOfSato = student.map(Student::getName).get(); // Optional<Student> -> Optional<String>
```

여기서 쓰인 `::` 표현은 Method Reference입니다. Lambda보다 짧게 쓸 수 있고, 대상 메서드가 분명할 때 사용하면 좋습니다.

```java
// 인수를 표준 출력하는 Lambda의 일반적인 작성법
Consumer<String> print = name -> System.out.println(name);

// Method Reference로 바꾼 형태
Consumer<String> print = System.out::print;

// 인스턴스 생성
Supplier<String> supplier = String::new;
```

### filter(Predicate<? super T> predicate)

`filter()`도 Collection이나 Stream에 익숙하다면 쉽게 이해할 수 있습니다. 조건에 맞을 때만 값을 남기고 싶을 때 씁니다.

```java
// 전통적인 패턴
public String getSato(Student student) {
    String name = student.getName();
    if (name != null && name.equals("Sato")) {
        return name;
    }
}

// filter
public String getSato(Student student) {
    return Optional.ofNullable(student)
            .filter(s -> s.getName().equals("Sato"))
            .map(Student::getName)
            .get();
}
```

Optional은 요소가 하나뿐이므로, 조건이 false가 되면 뒤의 메서드는 더 이상 실행되지 않습니다.

### isPresent()

값이 들어 있는지 확인하는 메서드입니다. 값이 있으면 `true`, 없으면 `false`를 반환합니다.

```java
String name = "Sato";
Optional<String> studentName = Optional.ofNullable(name);
studentName.isPresent(); // true
```

### ifPresent(Consumer<? super T> consumer)

값이 있을 때만 동작을 실행하고 싶을 때 씁니다.

```java
Optional<String> name = Optional.ofNullable(student.getName());
name.ifPresent(n -> System.out.println(n));
```

### orElse(T other)

값이 없을 때 기본값을 돌려줍니다. 따라서 별도의 `get()`를 쓰지 않아도 됩니다.

```java
String defaultName = "Sato";
Optional<String> name = Optional.ofNullable(student.getName());
String result = name.orElse(defaultName); // student.getName()이 null이면 defaultName이 된다
```

### orElseGet(Supplier<? extends T> other)

값이 없을 때만 Lambda를 실행해서 기본값을 만듭니다. 기본값 생성 비용이 있다면 이쪽이 더 낫습니다.

```java
Optional<String> name = Optional.ofNullable(student.getName());
String result = name.orElseGet(() -> student.getNumber() + "번 학생의 이름이 없습니다"); // student.getName()이 null이면 Lambda를 실행한다
```

### orElseThrow(Supplier<? extends X> exceptionSupplier)

값이 없을 때 예외를 던집니다.

```java
Optional<String> name = Optional.ofNullable(student.getName());
String result = name.orElseThrow(BusinessException::new);
```

## Optional에서 주의해야 할 사항

Optional은 편리하지만, 모든 `null` 처리를 전부 Optional로 바꿀 필요는 없습니다. 어떤 경우에는 오히려 단순한 `null` 체크가 더 낫습니다. 여기서는 사용할 때 주의할 점만 간단히 짚어 보겠습니다.

### 성능을 의식하자

Optional은 값을 감싸는 객체이므로, 무조건 쓰면 성능 비용이 생깁니다. 단순한 `null` 체크를 대체할 목적이라면 굳이 사용할 이유가 없습니다.

또한 `orElse()`는 값이 있어도 인수 평가가 일어나므로, 기본값 생성 비용이 큰 경우에는 `orElseGet()`이 더 적합합니다. `orElseGet()`은 필요한 경우에만 실행되므로 더 Lazy합니다.

다만 항상 `orElseGet()`이 정답은 아닙니다. 기본값이 이미 정적으로 준비되어 있다면 `orElse()`가 더 단순할 수 있습니다. 상황에 맞게 선택해야 합니다.

```java
// 좋지 않은 예(null이 아닐 때 버려지는 인스턴스)
public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return Optional.ofNullable(student).orElse(Student::new);
}

// 좋은 예
public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return Optional.ofNullable(student).orElseGet(Student::new);
}
```

기본값이 항상 같은 경우라면 단순한 `null` 체크가 더 읽기 쉽습니다.

```java
// 좋지 않은 예(항상 같은 기본값이 정해져 있는 경우)
private static Student defaultStudent;

public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return Optional.ofNullable(student).orElse(defaultStudent);
}

// 좋은 예
public Student getStudent(String name) {
    Student student = this.repository.getStudent(name);
    return student != null ? student : defaultStudent;
}
```

### `isPresent()`와 `get()`의 조합은 피하자

`isPresent()`로 확인한 뒤 `get()`으로 꺼내는 방식은 결국 평범한 `null` 체크와 다르지 않습니다. 기본값이 필요하면 `orElseGet()`을, 예외가 필요하면 `orElseThrow()`를 쓰는 편이 낫습니다.

```java
// 좋지 않은 예
public String getStudent(String name) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));

    if (student.isPresent()) { // (value != null) 쪽이 낫다
        return student.get();
    } else {
        throw new NullPointerException();
    }
}

// 좋은 예
public String getStudent(String name) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));
    return student.orElseThrow(NullPointerException::new);
}
```

값이 있을 때만 처리하고 싶다면 `ifPresent()`를 쓰면 됩니다.

```java
// 좋지 않은 예
public void adjustScore(String name, int score) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));
    
    if (student.isPresent()) {
        student.get().setScore(score);
    }
}

// 좋은 예
public void adjustScore(String name, int score) {
    Optional<Student> student = Optional.ofNullable(this.repository.getStudent(name));
    student.ifPresent(s -> s.setScore(score));
}
```

### 필드에는 쓰지 말자

Optional은 필드용으로 쓰는 것을 전제로 하지 않는 경우가 많습니다. `Serializable`과도 잘 맞지 않기 때문에 DTO의 필드에 Optional을 두면 오히려 불편해집니다.

```java
// 좋지 않은 예
@Data
public class Student implements Serializable{

    private Optional<String> name; // 직렬화할 수 없다
}

// 좋은 예
@Data
public class Student implements Serializable{

    private String name; // 직렬화할 수 있다
}
```

### 인수로 쓰지 말자

메서드나 생성자의 인수로 Optional을 쓰면, 호출하는 쪽도 매번 Optional을 만들어야 합니다. 그 결과 코드가 오히려 복잡해집니다. 인수는 보통 타입으로 받고, 내부에서 `null`을 처리하는 편이 낫습니다.

```java
// 좋지 않은 예
public class Student {

    private String name;

    public Student(Optional<String> name) { // 인스턴스를 만들 때마다 Optional도 필요하다
        this.name = name.orElseThrow(NullPointerException::new); // Optional로 null 체크와 대입이 필요하다
    }
}

// 좋은 예
public class Student {

    private String name;

    public Student(String name) {
        this.name = name; // 기대한 대로 처리된다
    }
}
```

### Collection 안에서는 쓰지 말자

Collection의 요소는 보통 여러 개이므로, 각 요소를 Optional로 감싸는 것은 불필요한 비용이 됩니다. 요소 추가나 조회 때마다 매번 Optional을 거치게 되니 불편하기도 합니다.

```java
// 좋지 않은 예
List<Optional<String>> names = new ArrayList<>();
names.add(Optional.ofNullable(name1)); // 요소를 추가할 때마다 래핑이 필요하다

// 좋은 예
List<String> names = new ArrayList<>();
names.add(name1);
```

### Collection은 Collection답게

Collection을 반환하는 메서드라면, `null` 대신 빈 Collection을 돌려주는 편이 보통 더 낫습니다. `Collections.emptyList()`나 `Collections.emptyMap()`을 쓰면 됩니다. Spring Data JPA처럼 애초에 빈 리스트를 돌려주는 경우도 있으니, 굳이 Optional로 한 번 더 감쌀 필요는 없습니다.

```java
// 좋지 않은 예
public Optional<List<Student>> listStudent() {
    List<Student> students = this.repository.listStudent();
    return Optional.ofNullable(students);
}

// 좋은 예
public List<Student> listStudent() {
    List<Student> students = this.repository.listStudent();
    return students != null ? students : Collections.emptyList();
}
```

### int/long/double은 전용 Optional을 쓰자

Optional에는 primitive 타입 전용 변형도 있습니다. `int`, `long`, `double`이라면 `OptionalInt`, `OptionalLong`, `OptionalDouble`을 쓰는 편이 낫습니다.

```java
// 좋지 않은 예
Optional<Integer> count = Optional.of(100);
int countNum = count.get();

// 좋은 예
OptionalInt count = OptionalInt.of(100);
int countNum = count.getAsInt();
```

## 마지막으로

실무에서는 아직도 Java 8 이상으로 올리는 과정에 있는 프로젝트가 적지 않습니다. 그렇다면 Java 개발자로서는 Java 8의 핵심 API에 익숙해져 두는 편이 분명 도움이 됩니다. 그런 의미에서 `Function`과 `Optional`은 꼭 알아 둘 만한 API입니다. Java가 오래 살아남은 이유 중 하나도 결국 쓰기 쉬운 생태계를 계속 넓혀 왔기 때문이니, 이런 편의 기능을 외면할 이유는 없습니다.

Java는 오래된 언어이지만 최근에는 함수형 스타일 같은 흐름도 적극적으로 받아들이고 있습니다. 앞으로 얼마나 더 진화할지, 또 얼마나 오래 중심 언어로 남을지는 지켜볼 문제겠지만, 적어도 Java 8 이후의 변화는 한 번쯤 정리해 둘 가치가 충분합니다.

[^1]: List, Set, Map처럼 여러 요소를 담는 객체를 가리킵니다.
[^2]: 파일 입출력용 InputStream/OutputStream이 아니라, Collection의 요소를 하나씩 순회하면서 특정 메서드(주로 Lambda)를 실행할 수 있게 해 주는 Java 8의 API입니다.
[^3]: 반환값이 자기 자신이어서 여러 메서드를 연속으로 연결해 쓸 수 있는 구조입니다. 빌더 패턴은 대표적인 메서드 체인의 예입니다.
[^4]: 프로그래밍에서 Lazy란, 어떤 처리가 항상 일어나는 것이 아니라 호출될 때 처음 실행되는 구조를 의미합니다. 필요할 때만 처리하므로 불필요한 연산을 줄일 수 있습니다.
