---
title: "두 개의 List 합치기"
date: 2020-07-26
categories:
  - java
image: "../../images/java.webp"
tags:
  - stream
  - java
translationKey: "posts/java-compare-and-merge-lists"
---

자주 보는 사이트에서 이런 질문을 본 적이 있습니다. `List` 두 개를 중복 없이 하나로 합치는 방법이었습니다. SQL이라면 쉽게 끝날 문제지만, 쿼리를 바꿀 수 없거나 여러 API의 반환값을 직접 다뤄야 하면 결국 코드로 합쳐야 합니다. 저도 비슷한 상황은 처음이라 흥미가 생겨 직접 여러 방법을 시험해 봤습니다.

## 문제 코드

질문한 분이 하려는 일은 `List<Map<String, Object>>` 두 개를 비교해서, Map의 요소가 겹치면 하나의 List로 합치는 것입니다. 중복 기준은 Map의 key와 value였습니다.

현재 코드는 있지만 중복 검사 로직이 너무 복잡해졌고, 성능도 좋지 않은 상태였습니다. 처음 올려진 코드는 아래와 같았습니다.

```java
List<Object> list1 = ...
List<Object> list2 = ...

for (Object object : list1) {
    Map<String, String> map = (HashMap<String, String>) object;
    String a1 = map.get("a");
    for (Object object2 : list2) {
        Map<String, String> map2 = (HashMap<String, String>) object2;
        String a2 = map2.get("a");
        if (a1.equals(a2)) {
            Set<String> keys = map2.keySet();
            for (String key : keys) {
                String v = map2.get(key);
                map.put(key, v);
            }
        }
    }
}
```

3중 루프라서 같은 값을 하나씩 계속 확인하고 있습니다. 실제 코드인지는 모르겠지만, key가 리터럴이라면 전체 key를 도는 루프도 굳이 하나 더 들어가면 안 맞아 보입니다. 이렇게 하면 List 크기만큼 반복하면서 부담이 커집니다. 그래서 더 짧고 단순하게 바꿔 보려 했습니다.

## Stream 사용

먼저 Stream으로 해결할 수 있는지 살펴봤습니다. 여러 List를 합치는 방식은 꽤 있습니다.

### 결합과 중복 제거

`Stream.concat()`으로 두 스트림을 합치고, `distinct()`로 중복을 제거할 수 있습니다.

```java
// 결합할 List
List<String> list1 = List.of("a", "b", "d", "e");
List<String> list2 = List.of("b", "c", "d", "f");

// 결합과 중복 제거
List<String> concat = Stream.concat(list1.stream(), list2.stream()).distinct().collect(Collectors.toList());
```

`Stream.concat()`은 인자가 두 개뿐이라 3개 이상을 합칠 때는 다른 방법이 필요합니다. `distinct()`는 `equals()`가 제대로 정의돼 있다면 어떤 객체든 중복 제거가 가능합니다. 그래서 Lombok의 `@Data`처럼 `equals()`가 잘 준비된 클래스에도 쓸 수 있습니다.

### 한계

하지만 질문처럼 List 안의 Map 요소를 비교하려면 이 방법만으로는 부족합니다. Map 전체를 `equals()`로 비교하게 되기 때문에, 내부 요소 하나하나를 기준으로 검사하지는 못합니다. 그래서 단순히 두 List를 이어 붙이는 결과만 나오게 됩니다.

## for 루프 사용

이번에는 질문자의 코드를 조금 더 효율적으로 바꿔 보겠습니다. 조건이 맞을 때만 `put()`을 한 번 실행하면 되니, Stream보다 for 루프가 더 알맞았습니다.

먼저 `putAll()`을 써서 반복을 줄입니다.

```java
for (Object object : list1) {
    Map<String, String> map = (HashMap<String, String>) object;
    String a1 = map.get("a");
    for (Object object2 : list2) {
        Map<String, String> map2 = (HashMap<String, String>) object2;
        String a2 = map2.get("a");
        if (a1.equals(a2)) {
            map.putAll(map2);
            continue;
        }
    }
}
```

그 다음에는 key를 리터럴이 아니라 `Entry` 기준으로 비교하도록 바꿉니다. `Entry`는 `Set`으로 얻을 수 있으니 `contains()`로 비교할 수 있습니다.

```java
for (Object object : list1) {
    Map<String, String> map = (HashMap<String, String>) object;
    for (Object object2 : list2) {
        Map<String, String> map2 = (HashMap<String, String>) object2;
        for (Map.Entry<String, String> entry : map2.entrySet()) {
            if (map.entrySet().contains(entry)) {
                map.putAll(map2);
                continue;
            }
        }
    }
}
```

형 변환도 루프 밖에서 미리 해 두면 조금 더 깔끔합니다.

```java
List<Map<String, String>> convertedList1 = (List<Map<String, String>>) list1;
List<Map<String, String>> convertedList2 = (List<Map<String, String>>) list2;

for (Map<String, String> map : convertedList1) {
    for (Map<String, String> map2 : convertedList2) {
        for (Map.Entry<String, String> entry : map2.entrySet()) {
            if (map.entrySet().contains(entry)) {
                map.putAll(map2);
                continue;
            }
        }
    }
}
```

필요에 따라 key 정렬 같은 후처리가 더 필요할 수도 있지만, 일단 요구사항은 충족하는 형태입니다.

## 조건이 다를 때

질문자의 코드에서 추정할 수 있는 부분인데, List의 인덱스를 기준으로 비교하는 조건이라면 더 단순해질 수 있습니다. 같은 인덱스의 Map끼리 비교하는 방식입니다.

```java
for (Map<String, String> map : list1) {
    for (Map.Entry<String, String> entry : map.entrySet()) {
        Map<String, String> map2 = list2.get(list1.indexOf(map));
        if (map2.entrySet().contains(entry)) {
            map.putAll(map2);
            continue;
        }
    }
}
```

다만 이 방식은 두 List의 크기가 같다는 전제가 필요합니다.

## 마무리

이런 질문은 평소에 잘 마주치지 않던 조건을 한 번 더 생각해 보게 해서 좋습니다. 조사하는 과정에서 평소 잘 안 쓰던 메서드나 API를 다시 보게 되고, 제가 쓰던 코드도 자연스럽게 돌아보게 됩니다.

key가 아니라 value만 비교한다든지, 요소의 특정 필드만 중복 체크한다든지 하는 식으로 얼마든지 변형할 수 있습니다. 결국 중요한 것은 중복 기준을 먼저 분명히 정한 다음, 그 기준에 맞는 자료구조와 루프를 고르는 일이라고 생각합니다.
