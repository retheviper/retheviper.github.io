---
title: "Exposed에서 OneToMany를 매핑하는 방법"
date: 2021-07-26
categories: 
  - exposed
image: "../../images/exposed.webp"
tags:
  - kotlin
  - exposed
  - map
translationKey: "posts/exposed-mapping-record-to-object"
---

DB에서 1:N 관계는 흔합니다. 예를 들어 쇼핑몰에서 회원이 배송지를 여러 개 등록할 수 있다면, 회원 테이블에 배송지 컬럼을 계속 늘리는 것보다 배송지 테이블을 분리하는 편이 훨씬 자연스럽고 안전합니다. 이렇게 나누면 배송지 테이블은 회원 테이블과 N:1 관계를 갖게 됩니다.

하지만 저장 구조가 우선인 DB와, 그 데이터를 사용하기 좋게 가공해야 하는 애플리케이션은 요구사항이 다릅니다. 예를 들어 하나의 회원 레코드에 여러 배송지 레코드가 연결되어 있다면, SQL 결과는 보통 아래처럼 펼쳐진 형태가 됩니다.

```text
|-----------|-------------|-----------------|
| member.id | member.name | mailing.address |
|-----------|-------------|-----------------|
|         1 |        John |           Tokyo |
|         1 |        John |        New York |
|         1 |        John |         Beijing |
|         2 |     Simpson |           Osaka |
|         2 |     Simpson |          Nagoya |
|-----------|-------------|-----------------|
```

하지만 애플리케이션에서 이런 펼쳐진 형태를 그대로 다루는 경우는 많지 않습니다. Kotlin이라면 보통 한 회원 객체 안에 배송지 목록을 넣는 식으로 표현하게 됩니다.

```kotlin
data class Member(
    val id: Int,
    val name: String,
    val mailingAdress: List<String>
)
```

이런 객체는 대개 REST API 응답 JSON으로 그대로 사용하게 됩니다. 앞의 데이터를 JSON으로 바꾸면 대략 이런 모습이 됩니다.

```json
{
    "members": [
        { 
            "id": 1,
            "name": "John",
            "mailingAddress": [
                "Tokyo",
                "New York",
                "Beijing"
            ]
        },
        {
            "id": 2,
            "name": "Simpson",
            "mailingAddress": [
                "Osaka",
                "Nagoya"
            ]            
        }
    ] 
}
```

문제는 JSON 변환이 아니라, DB에서 가져온 레코드를 이런 객체 구조로 어떻게 매핑하느냐입니다. JPA 같은 ORM은 관계를 어노테이션으로 선언해 두면 어느 정도 자동으로 처리해 주지만, [jOOQ](https://www.jooq.org), [Querydsl](https://querydsl.com), [Exposed](https://github.com/JetBrains/Exposed), [Ktorm](https://www.ktorm.org)처럼 DSL로 SQL을 직접 다루는 방식에서는 매핑을 직접 써야 합니다. 그리고 조회 결과는 결국 행 단위 데이터이기 때문에, 이를 어떻게 효율적으로 묶을지가 고민거리가 됩니다.

이번 글에서는 Exposed DSL로 조회한 One-to-Many 결과를 Kotlin 객체로 매핑하는 방법을 몇 가지 비교해 보겠습니다.

## 표별로 선택

가장 단순한 방법은 테이블별로 따로 조회한 뒤 애플리케이션에서 합치는 것입니다. 쿼리 의도가 명확하다는 장점은 있지만, 효율 면에서는 아쉬움이 남습니다. 예를 들면 다음과 같습니다.

```kotlin
transaction {
    // 먼저 Member 테이블을 조회해 객체로 매핑한다
    val member = Member.select { Member.id eq id }
        .first()
        .let {
            MemberDto(
                id = row[Member.id].value,
                name = row[Member.name],
                role = listOf(row[Mailing.role])
            )
        }

    // Mailing 테이블을 조회해 리스트로 만든다
    val mailingAddress = Mailing
            .select { Mailing.memberId eq member.id }
            .map { it[Mailing.address] }

    // 객체를 복사하고 배송지 데이터를 채운다
    member.copy(mailingAddress = mailingAddress)
}
```

이 방식은 이해하기는 쉽지만, 트랜잭션과 쿼리 수 관점에서는 좋지 않습니다. Exposed의 `transaction` 블록 안에 넣을 수는 있어도, 사실상 한 번에 가져올 수 있는 데이터를 여러 번 조회하게 되기 때문입니다. 지금은 Member와 Mailing 두 테이블뿐이라 단순하지만, 1:N 관계 테이블이 더 늘어나면 쿼리 수도 함께 늘어납니다. 조회 대상 회원이 여러 명이면 상황은 더 나빠집니다.

게다가 객체를 한 번 만든 뒤 다시 `copy()`로 조합하는 것도 비용이 있습니다. 대상 레코드가 많아질수록 쿼리뿐 아니라 객체 생성 수도 함께 늘어나므로, 큰 규모에서는 비효율이 눈에 띌 수 있습니다.

## join하여 매핑하기

관련 데이터를 여러 테이블에 걸쳐 얻으려면 역시 `join`이 효율적입니다. 이 경우 먼저 발행되는 쿼리의 수는 개별 테이블에 대해 Select할 때에 비해 극적으로 줄어듭니다. 알고리즘에서 자주 사용되는 표현의 [Big O 기법](https://vmm.dev/ko/cci/cci-0.md)로 표현하면, 전자는 `O(N^2)`이며, 이것은 `O(1)`라고 표현할 수 있을 것입니다.

그렇다면 `join`으로 한 번에 가져오는 쪽이 더 낫다는 결론은 자연스럽습니다. 문제는 그다음입니다. 조인 결과에는 회원 정보가 배송지 수만큼 반복되기 때문에, 이 중복을 애플리케이션 쪽에서 다시 정리해야 합니다.

여기서는 그 방법을 세 가지로 나눠 보겠습니다.

### reduce

우선은 쿼리의 결과로서 취득한 행을, 각각 객체에 매핑한 후, `reduce`로 정리하는 방법입니다. 예를 들면 다음과 같습니다.

```kotlin
transaction {
    Member.leftJoin(Mailing)
        .select { (Member.id eq id) and (Mailing.memberId eq Member.id) }
        .map {
            // 일단 객체로 매핑한다
            MemberDto(
                id = it[Member.id].value,
                name = it[Member.name],
                mailingAddress = listOf(it[Mailing.address])
            )
        }.reduce { acc, memberDto ->
            // 객체를 하나로 합친다 (mailingAddress는 누적)
            acc.copy(
                mailingAddress = acc.mailingAddress + memberDto.mailingAddress
            )
        }
}
```

이 방식의 가장 큰 문제는 행 수만큼 객체 인스턴스가 먼저 만들어진다는 점입니다. 조회 대상 회원이 한 명뿐이어도 배송지가 많으면 그만큼 `MemberDto`가 생기고, `reduce` 과정에서 다시 `copy()`가 일어나므로 객체 생성 수는 더 늘어납니다.

또 회원을 여러 명 가져오는 경우에는 모든 결과를 하나의 객체로 합쳐 버릴 위험도 있습니다. 결국 이 방법은 단건 조회처럼 아주 제한된 상황에서만 안전하게 쓸 수 있습니다.

### groupBy

조회한 레코드를 한 번 `Map`으로 묶는 방법도 있습니다. Kotlin의 `groupBy`를 사용하면 key와 value 변환 기준을 지정해 한 key 아래에 여러 값을 모을 수 있습니다. 여기서는 key를 회원 객체, value를 배송지로 두면 비교적 자연스럽게 정리할 수 있습니다.

```kotlin
transaction {
    Member.leftJoin(Mailing)
        .select { (Member.id eq id) and (Mailing.memberId eq Member.id) }
        // key는 Member 객체, value에는 Mailing 레코드를 모은다
        .groupBy({
            MemberDto(
                id = it[Member.id].value,
                name = it[Member.name],
            )
        }, { it[Mailing.address] })
        // key 객체에 Mailing 레코드를 넣는다
        .map { (key, value) ->
            key.copy(mailingAddress = value)
        }
}
```

이 방식은 앞선 방법들의 문제를 꽤 줄여 줍니다. 다만 `groupBy`도 결국 각 행에 대해 람다를 실행해야 하므로, 객체 생성 비용이 얼마나 드는지는 한 번 확인해 볼 필요가 있습니다. 그래서 구현을 잠깐 들여다보면 다음과 같습니다.

```kotlin
public inline fun <T, K, V> Iterable<T>.groupBy(keySelector: (T) -> K, valueTransform: (T) -> V): Map<K, List<V>> {
    return groupByTo(LinkedHashMap<K, MutableList<V>>(), keySelector, valueTransform)
}
```

`groupBy`의 구현에서는 `groupByTo`이라는 함수에 자신의 인수와 만들어지는 Map의 인스턴스를 건네주고 있을 뿐이지요. 그럼 `groupByTo`의 내용을 살펴 보겠습니다.

```kotlin
public inline fun <T, K, V, M : MutableMap<in K, MutableList<V>>> Iterable<T>.groupByTo(destination: M, keySelector: (T) -> K, valueTransform: (T) -> V): M {
    for (element in this) {
        val key = keySelector(element)
        val list = destination.getOrPut(key) { ArrayList<V>() }
        list.add(valueTransform(element))
    }
    return destination
}
```

여기서 확인할 수 있는 것은, 결국 원본 컬렉션의 요소 수만큼 `keySelector`와 `valueTransform`이 실행된다는 점입니다. `reduce`처럼 전부 하나로 합쳐지는 문제는 줄어들지만, 여전히 중간 객체가 많이 생길 수 있다는 한계는 남습니다. 그래서 마지막으로 다른 방법을 보겠습니다.

### Map

마지막 방법은, 조인 결과를 다시 `Map`으로 만드는 대신 처음부터 보조 `Map`을 하나 두고 거기에 누적하는 방식입니다. `Map.compute()`를 사용하면 key가 없을 때 새 값을 만들고, 이미 있으면 기존 값을 바꾸는 로직을 한곳에 모을 수 있습니다.

즉 회원 ID를 key로 두고, 값이 없으면 `MemberDto`를 새로 만들고, 값이 있으면 그 객체에 배송지 정보를 누적하는 식입니다. 루프가 끝난 뒤에는 `Map`의 value만 꺼내면 됩니다.

```kotlin
// 객체를 합치기 위한 Map (key는 Member.id)
val helperMap = mutableMapOf<Int, MemberDto>()

transaction {
    Member.leftJoin(Mailing)
        .select {
            (Member.id eq id) and (Mailing.memberId eq Mailing.id)
        }
        .forEach {
            helperMap.compute(it[Member.id].value) { key, value ->
                // value가 null이 아니면 복사하면서 mailingAddress를 누적한다
                value?.copy(
                    mailingAddress = value.mailingAddress + it[Mailing.address]
                // value가 null이면 새 인스턴스를 만든다
                ) ?: MemberDto(
                    id = key,
                    name = it[Member.name],
                    mailingAddress = listOf(it[Mailing.address])
                )
            }
        }.let {
            // value를 List로 변환
            helperMap.map { it.value }
        }
}
```

이 방법은 중복 정리와 객체 생성 수 절감 사이에서 가장 균형이 좋습니다. 물론 `mailingAddress`를 누적할 때마다 `copy()`가 일어나는 비용은 남아 있지만, 앞선 방식들보다 불필요한 중간 객체는 훨씬 적습니다.

다만 여기서 쓰는 `Map`은 반드시 메서드 내부에서만 생성해야 합니다. 필드로 빼 두면 데이터 무결성이나 메모리 사용량에 예상 밖의 영향을 줄 수 있습니다.

## 마지막으로

DSL로 쿼리를 직접 작성하면 JPA 계열 ORM에서 자주 거론되는 N+1 문제를 비교적 의식적으로 피할 수 있습니다. 대신 객체 매핑을 직접 설계해야 한다는 부담이 생깁니다. 저도 쿼리를 직접 쓰는 일이 늘 즐겁지는 않지만, 필요한 쿼리만 정확히 제어할 수 있다는 점은 분명한 장점입니다. 반대로 단순한 구조라면 ORM이 매핑까지 자동으로 처리해 주는 쪽이 더 편할 때도 있습니다.

그리고 여기서 다룬 문제는 DB에만 한정되지 않습니다. 다른 API 응답처럼 중복된 데이터를 받아, 애플리케이션에서 원하는 구조로 다시 묶어야 하는 상황은 어디서든 생길 수 있습니다. 지금 기준으로는 `Map`을 이용한 방식이 가장 균형이 좋아 보이지만, 상황에 따라 더 나은 방법이 있을 수도 있습니다.
