---
title: "실제 프로덕트에서 밟은 JPA / Hibernate 함정들"
date: 2026-05-15
translationKey: "posts/jpa-hibernate-pitfalls"
categories:
  - quarkus
image: "../../images/quarkus.webp"
tags:
  - kotlin
  - quarkus
  - jpa
  - hibernate
  - jooq
---

최근 어떤 업무 시스템의 업데이트 API에서 레이턴시 조사와 에러 대응을 할 일이 있었습니다. 구성은 Kotlin + Quarkus + Hibernate ORM이고, 원래는 Hibernate를 중심으로 만들어진 프로젝트입니다. 다만 N+1이나 업데이트 시의 암묵적인 동작이 무거워지면서 일부에는 jOOQ도 도입했고, 지금은 Hibernate와 jOOQ를 섞어서 운용하고 있는 상태입니다.

조사해 보니 비즈니스 로직 자체보다 JPA / Hibernate의 동작에서 오는 비용이 꽤 많았습니다. 물론 Hibernate가 나쁘다거나 JPA는 절대 쓰면 안 된다고 말하려는 것은 아닙니다. 다만 실제 프로덕트에서 업데이트 API의 p90이 초 단위가 되었을 때, 실제로 무엇이 일어나고 있었는지를 정리해 둘 가치는 있다고 생각했습니다.

참고로 예전에 이 블로그에 [「MyBatis보다 JPA를 써 보고 싶다」](../spring-data-jpa/)는 글을 쓴 적이 있습니다. 당시에는 SQL을 가능한 단순하게 두고 애플리케이션 쪽의 엔티티를 중심으로 다룰 수 있다는 점이 꽤 매력적으로 보였습니다. 지금도 그 생각 자체가 완전히 틀렸다고 보지는 않습니다. 다만 실제 프로덕트에서 업데이트 hot path를 따라가 보니, "SQL을 쓰지 않아도 된다"는 장점 뒤에 있는 암묵적인 SELECT, flush, refresh, lifecycle 비용을 무시할 수 없게 됐다는 것이 지금의 감각입니다.

이번 글에서는 구체적인 비즈니스 로직은 숨기고, 실제로 밟았던 함정과 그때 선택한 회피책을 써보려고 합니다.

## 응답 DTO 구성에서 lazy 프록시가 연쇄적으로 초기화된다

처음에 크게 영향을 주던 것은 업데이트 처리의 마지막에 엔티티를 응답 DTO로 변환하던 부분이었습니다.

PUT이나 POST의 마지막에 업데이트된 엔티티에 대해 `toResponseDto()` 같은 함수를 호출합니다. 그러면 그 안에서 `@ManyToOne(fetch = LAZY)`나 `@OneToMany`, `@ManyToMany`의 getter가 차례대로 호출됩니다. Hibernate 입장에서는 이것이 lazy 프록시를 초기화해야 한다는 의미가 됩니다.

결과적으로 CloudWatch Performance Insights의 Top SQL에는 비즈니스 로직 본체가 아니라 관련 엔티티 SELECT가 대량으로 나타났습니다. 게다가 자식 엔티티의 DTO 구성이 다시 자기 lazy 관계를 건드리기 때문에, 두 단계로 SELECT가 연쇄됩니다.

여기서 선택한 대응은 꽤 단순했습니다. 업데이트 API의 응답을 `{ id, version }` 같은 최소 DTO로 바꿨습니다. 상세 표시가 필요하다면 별도의 GET으로 다시 가져오는 계약으로 바꾸는 것입니다. getter 자체를 호출하지 않으면 lazy 초기화도 물리적으로 일어나지 않습니다.

## CascadeType.REFRESH로 관련 collection이 전부 읽힌다

다음으로 크게 영향을 준 것은 `CascadeType.REFRESH`였습니다.

공통 기반 repository에 영속화하고, flush하고, refresh하는 편의 메서드가 있었습니다. 이미지로는 다음과 같은 코드입니다.

```kotlin
entityManager.persist(entity)
entityManager.flush()
entityManager.refresh(entity)
```

이 자체는 optimistic lock이나 DB에서 결정되는 값을 다시 가져오고 싶을 때 편리합니다. 하지만 대상 엔티티의 관계에 `CascadeType.REFRESH`가 붙어 있으면 이야기가 달라집니다. `refresh()` 시 Hibernate는 대상 엔티티를 다시 로드하고, REFRESH cascade 대상인 lazy collection을 초기화한 뒤, 각 요소도 refresh합니다.

즉 "관련에도 refresh를 전파하고 싶다"고 한 곳에 적은 것뿐인데, 업데이트할 때마다 관련 collection의 읽기 비용을 지불하는 구조가 되어 있었습니다. 업무상 건드리지 않는 collection까지 SELECT되기 때문에 꽤 알아차리기 어려운 비용입니다.

대응으로는 기반 repository에 refresh하지 않는 형제 메서드를 만들었습니다. 성능에 민감한 경로에서는 그쪽을 사용합니다. 테스트에서는 Hibernate의 `Statistics`를 사용해 `getCollectionFetchCount()` 등으로 collection fetch가 늘지 않는지 확인할 수 있습니다.

## orphanRemoval과 clear + addAll은 전체 교체가 된다

`@OneToMany(orphanRemoval = true)` collection을 가진 aggregate에서도 문제가 있었습니다. 업데이트 시 다음과 같은 방식으로 다시 쓰고 있었습니다.

```kotlin
children.clear()
children.addAll(newChildren)
```

이렇게 쓰면 Hibernate는 기존 요소를 orphan으로 보고 DELETE하고, 새 요소를 INSERT합니다. 차이가 1행뿐이어도 기존 전 행 DELETE, 신규 전 행 INSERT가 됩니다. 업무적으로는 일부만 바뀐 것 같아도 DB상으로는 전체 교체입니다.

이 경로는 jOOQ로 차분 적용하도록 바꿨습니다. `INSERT ... ON CONFLICT`, `UPDATE`, `DELETE WHERE NOT IN (...)`처럼 필요한 차분만 SQL로 명시적으로 쓰는 방식입니다.

여기서 중요한 것은, GET에서 관련 그래프를 반환하기 위한 JPA mapping을 즉시 전부 버릴 필요는 없다는 점입니다. 읽기에서는 편리한 mapping을 남겨두고, 업데이트 경로만 SQL builder로 빼는 식의 구분도 현실적이었습니다.

## addAll만으로도 lazy collection이 초기화된다

비슷하지만 더 조용하게 밟은 문제가 `addAll` 단독 사용입니다.

히스토리성 자식 테이블에 append만 하고 있다고 생각했는데, 업데이트 경로에서 매번 기존 히스토리 전체가 SELECT되고 있었습니다. 그 안에는 JSON 컬럼도 포함되어 있었기 때문에 읽기 비용을 무시할 수 없었습니다.

원인은 부모 엔티티의 helper가 내부에서 collection에 `addAll`을 하고 있었기 때문입니다. Hibernate의 `PersistentBag`은 `addAll` 전에 기존 요소와의 동등성 비교를 해야 하므로 collection을 초기화합니다. 즉 `clear() + addAll()` 전체 교체뿐 아니라 append 용도의 `addAll` 단독도 기존 collection을 읽으러 갈 수 있습니다.

대응은 append 계열 자식 테이블에 쓸 때 collection을 전혀 건드리지 않고 jOOQ로 직접 INSERT하는 것이었습니다. 엔티티 helper를 호출하면 편리해 보이지만, 그 뒤에서 lazy collection이 초기화된다면 업데이트 경로에서는 꽤 비싼 편의성입니다.

## flush + refresh와 PreUpdate 재발화 위험

응답 DTO를 가볍게 만들어도 업데이트 API의 레이턴시가 남는 곳이 있었습니다. 여기서도 공통 기반의 `flush + refresh`가 영향을 주고 있었습니다.

단순한 round trip 비용도 있지만, 더 무서운 것은 lifecycle listener와의 조합입니다. 예를 들어 `@PreUpdate`에서 `updatedAt`을 덮어쓰는 설계라면, handler 안에서 `updatedAt`을 읽어 다른 컬럼에 동기화하는 단순한 구현은 타이밍 문제로 깨집니다.

또 jOOQ로 보조 컬럼을 업데이트한 뒤 그 값을 managed entity에 다시 써 넣으면 엔티티가 dirty 상태가 됩니다. 그대로 transaction commit으로 가면 다시 flush되고, `@PreUpdate`가 재발화되어 `updated_at`만 불필요하게 앞으로 가는 사고가 생깁니다.

여기서는 refresh하지 않는 경로를 쓰는 것에 더해, jOOQ로 업데이트한 값을 managed entity에 되돌려 쓰지 않는 방침으로 갔습니다. 보조 컬럼의 동기화는 SQL builder로 직접 쓰고 JPA lifecycle을 거치지 않습니다. in-memory 엔티티를 "깔끔하게 동기화하고 싶다"는 마음은 있지만, 그걸 건드려서 다른 업데이트가 발화된다면 동기화하지 않는 편이 안전합니다.

## Hibernate.initialize와 같은 aggregate의 중복 로드

응답을 가볍게 만든 뒤에는 `Hibernate.initialize(entity.someAssociation)` 같은 호출도 정리했습니다.

예전에는 응답 DTO 때문에 필요했을 수도 있지만, 최소 DTO로 바꾼 뒤에는 단순히 소비되지 않는 IO입니다. refresh 직후 ManyToMany를 한 번 더 SELECT하는 곳도 있었습니다.

게다가 히스토리 생성을 위해 엔티티를 로드하고, 그 뒤 공통 기반 업데이트 메서드에서도 같은 엔티티를 다시 로드하는 중복도 있었습니다. "로드된 엔티티를 넘기는" 메서드와 "ID를 넘기면 내부에서 다시 가져오는" 메서드가 섞여 있으면 이런 비용은 잘 보이지 않습니다.

이런 문제는 개별 `initialize`를 지우는 것만으로는 부족하고, 기반 메서드의 계약 자체를 정리할 필요가 있었습니다.

## 비관적 락의 round trip 자체가 무겁다

lazy 초기화나 refresh를 제거해도 업데이트 API의 p90이 아직 수백 ms에서 초 단위로 남는 경로가 있었습니다.

보니 공통 기반 repository의 "버전 포함 업데이트" 메서드가 기본적으로 다음 흐름을 타고 있었습니다.

1. `findById(id, LockModeType.PESSIMISTIC_WRITE)`로 `SELECT ... FOR UPDATE`
2. `persistAndFlush`
3. `refresh`

cascade도 lazy도 건드리지 않는 경우에도, 이 세 단계 round trip 자체가 지배적입니다. 특히 경합이 거의 없는 업데이트 경로에서는 "혹시 모르니" 넣은 비관적 락이 매번 왕복 1회 비용을 더하고 있을 뿐이었습니다.

여기서는 업데이트 semantics를 다시 보고, `@Version` 기반 optimistic lock으로 충분한 경로에서는 비관적 락을 제거했습니다. 기반 메서드도 "락을 건다 / 걸지 않는다", "refresh한다 / 하지 않는다"를 나눌 필요가 있습니다. 편리한 공통 기반이 암묵적으로 비용을 쌓고 있지 않은지는 꽤 의심해 보는 편이 좋다고 생각합니다.

## JSON 컬럼을 다른 경로로 쓰면 표현이 어긋난다

jOOQ로 우회할 때 다른 종류의 문제도 나왔습니다. `@JdbcTypeCode(SqlTypes.JSON)` JSON 컬럼을 Hibernate cascade가 아니라 SQL builder로 직접 쓰도록 바꾸자, DB상의 JSON 표현이 미묘하게 달라졌습니다.

예를 들면 property 순서, `null` 필드의 유무, naming rule 같은 것입니다. 뒤쪽에서 비교나 hash, 테스트를 하고 있으면 이 차이로 깨집니다.

원인은 Hibernate 경유 serialization에서는 Quarkus에서 DI된 Jackson `ObjectMapper`가 사용되고 있었지만, jOOQ 쪽에서는 직접 만든 `ObjectMapper`를 쓰고 있었기 때문입니다. `ObjectMapperCustomizer`가 적용되지 않았기 때문에 같은 객체라도 JSON 문자열이 일치하지 않았습니다.

대응으로는 DI된 `ObjectMapper`를 명시적으로 inject해서 Hibernate 쪽과 같은 설정으로 serialize하도록 했습니다. 그리고 양쪽 경로의 결과가 bit-for-bit로 일치하는지 테스트로 확인했습니다.

ORM의 컬럼 타입 변환을 SQL builder 쪽에 이식할 때는 SQL만 보면 같아 보입니다. 하지만 실제로는 serialization layer의 암묵적인 의존도 함께 이식해야 합니다. 이 문제는 `AttributeConverter` 계열 컬럼에서도 똑같이 일어날 수 있습니다.

## detach했다고 생각해도 persistence context에 관련 엔티티가 남는다

일괄 편집 API에서는 또 다른 종류의 사고도 있었습니다. 흐름은 먼저 대상 엔티티를 fetch join으로 로드하고, 업데이트 전 값을 보관한 뒤 jOOQ로 일괄 UPDATE하고, 업데이트 후 값을 다시 로드해 히스토리를 등록하는 방식이었습니다.

이때 업데이트 전 값을 보관하기 위해 각 엔티티에 `entityManager.detach(it)`을 호출하고 있었습니다. 개별 entity를 persistence context에서 빼면 이후 jOOQ 업데이트와 간섭하지 않을 것이라고 생각한 것입니다.

하지만 `detach()`는 지정한 엔티티만 분리합니다. fetch join으로 함께 로드된 관련 엔티티나 자식 collection 요소는 managed 상태로 persistence context에 남습니다. 이 상태에서 jOOQ가 DB를 직접 UPDATE하면 DB의 값과 managed entity의 값이 어긋납니다. 다음 쿼리 실행 시점이나 transaction commit 시점에 Hibernate의 auto-flush가 실행되면, 남아 있던 managed entity의 dirty checking이 DB의 최신 상태와 충돌합니다.

더 무서운 것은 업데이트 전 값이라고 생각하고 참조하던 것이 사실 Hibernate가 관리하는 같은 인스턴스였던 경우입니다. auto-flush나 이후 처리의 영향으로 히스토리에 넣을 before 값이 이미 업데이트 후 값이 되어 있는, 메모리상의 불일치도 생길 수 있습니다.

대응으로는 업데이트 전 값을 plain data class snapshot으로 복사해 보관하도록 바꿨습니다. 그리고 개별 `detach()`가 아니라 `entityManager.clear()`로 persistence context 전체를 비웠습니다. fetch join으로 같이 끌려온 관련 엔티티까지 한 번에 분리하고, 이후의 차분 판정과 히스토리 before 값은 snapshot만 보도록 했습니다.

ORM으로 읽은 것을 SQL builder로 업데이트한다면, "읽은 entity를 그대로 업무 로직의 값으로 믿지 않는다"는 점이 중요합니다. 필요한 값은 immutable snapshot으로 빼내고, persistence context는 빨리 끊어야 합니다. `detach()`는 cascade되지 않는다는 전제를 잊으면 꽤 위험합니다.

## Hibernate를 넣고 있는 한 Kotlin 빌드 설정에도 영향이 간다

마지막은 runtime이 아니라 빌드와 언어 버전 이야기입니다.

Kotlin에서 JPA 엔티티를 쓰려면 기본적으로 두 가지 제약이 있습니다.

- no-arg constructor가 필요하므로 `kotlin-noarg` 또는 `kotlin-jpa`가 필요
- proxy 생성을 위해 class를 `open`으로 만들어야 하므로 `kotlin-allopen`이 필요

이 플러그인들은 Kotlin 본체, Gradle plugin, serialization 같은 관련 플러그인과 강하게 묶여 있습니다. 결과적으로 Kotlin 버전 업데이트, K2 compiler 이행, kapt 제거 같은 주제가 JPA 플러그인 호환성 확인에 끌려갑니다.

Hibernate 자체를 직접 건드리지 않는 업데이트 작업까지 ORM의 존재에 의해 속도가 제한되는 셈입니다. 개인적으로는 ORM을 선택할 때 자주 놓치는 비용이라고 생각합니다.

## 경향을 정리하면

이번 문제들을 늘어놓으면 꽤 일관된 경향이 있습니다.

| 카테고리 | 핵심 메커니즘 | 회피책 |
|---|---|---|
| lazy 프록시 초기화 연쇄 | DTO 구성의 getter가 SELECT를 연쇄 발화한다 | 업데이트 응답을 최소 DTO로 한다 |
| cascade REFRESH 전체 전파 | `refresh()`가 관련 collection을 초기화한다 | refresh하지 않는 경로를 둔다 |
| orphanRemoval 전체 교체 | `clear() + addAll()`이 전 행 DELETE + INSERT가 된다 | jOOQ로 차분 적용한다 |
| collection 암묵 초기화 | `addAll`만으로도 `PersistentBag`이 초기화된다 | collection을 건드리지 않고 직접 쓴다 |
| flush + refresh | round trip 비용과 `@PreUpdate` 재발화가 생긴다 | refresh를 제거하고 lifecycle을 우회한다 |
| 비관적 락 | `SELECT ... FOR UPDATE` 왕복 1회가 항상 붙는다 | optimistic lock으로 충분한 경로에서 제거한다 |
| 중복 로드 | 같은 aggregate를 여러 계층에서 로드한다 | 메서드 계약을 정리한다 |
| JSON 직렬화 | ORM 경로와 SQL 경로에서 표현이 어긋난다 | DI된 `ObjectMapper`를 재사용한다 |
| persistence context와 auto-flush | `detach()`가 관련 entity를 분리하지 않고, managed entity가 이후 업데이트와 충돌한다 | snapshot으로 빼고 `entityManager.clear()`를 호출한다 |
| 빌드 계층으로의 파급 | JPA용 Kotlin 플러그인이 업데이트를 제한한다 | 신규에서는 ORM 자체를 빼는 선택지를 둔다 |

공통점은 코드상으로는 작아 보이는 조작이 Hibernate의 암묵적인 동작을 통해 큰 IO나 flush로 변환된다는 것입니다. getter, `refresh()`, `addAll()`, `clear()`, `initialize()`, `PESSIMISTIC_WRITE`, `detach()`. 하나하나는 편리하지만 업데이트 API의 hot path에 들어가면 꽤 무거워집니다.

## 신규 프로젝트에서는 어떻게 할 것인가

이번 경험을 바탕으로 한다면, 저는 신규 프로젝트에서는 Hibernate ORM을 처음부터 넣지 않고 SQL builder를 중심으로 만들 것 같습니다. 이 프로젝트에서는 jOOQ를 사용하고 있습니다.

예전의 제 입장에서 보면 꽤 큰 의견 변화입니다. 바뀐 이유는 JPA의 개념을 더 공부했기 때문이 아니라, 실제 운영에서 업데이트 API의 느린 지점을 추적하고 무엇이 언제 DB에 접근하는지를 하나씩 벗겨 봤기 때문입니다.

물론 기존 프로젝트에서 Hibernate를 한 번에 없애는 것은 현실적이지 않습니다. GET에서 관련 그래프를 반환하기 위한 mapping이 편리한 장면도 있습니다. 그래서 지금 프로젝트에서는 Hibernate와 jOOQ를 섞어 쓰면서, 특히 업데이트 hot path에서 Hibernate의 암묵적인 동작을 제거해 가는 방향을 취하고 있습니다.

다만 신규라면 처음부터 `@OneToMany`, `CascadeType.*`, `orphanRemoval`, `Hibernate.initialize`, `persistAndFlushRefresh`, `entityManager.detach` 같은 도구를 닫아 두는 편이 싸다고 느낍니다. 편리해 보이는 것일수록 실제로 SELECT를 몇 번 하는지, 언제 flush되는지, 어떤 listener가 실행되는지, 어떤 인스턴스가 managed인지 해석해야 합니다.

하나만 덧붙이면, Hibernate Validator는 ORM이 아닙니다. Bean Validation의 reference implementation이므로 도입해도 문제 없습니다. 피하고 싶은 것은 `quarkus-hibernate-orm` 같은 ORM 본체입니다. CI에서 설정 키를 grep한다면 `quarkus.hibernate-orm.*`만 reject 대상으로 삼는 식의 구분이 필요합니다.

## 마지막으로

이번에 밟은 문제들은 개별 버그라기보다 엔티티, aggregate 경계, lifecycle listener, 공통 기반 repository가 Hibernate의 암묵적인 동작과 얽힌 구조적 부채였습니다. 한 곳을 고쳐도 다른 곳에서 같은 패턴이 다시 나타납니다.

그래서 대응도 꽤 소박합니다. 업데이트 응답에서는 getter를 호출하지 않는다. refresh하지 않는다. collection을 건드리지 않는다. 비관적 락을 기본값으로 두지 않는다. SQL builder로 우회할 때는 serialization 설정까지 맞춘다. ORM으로 읽은 값은 snapshot으로 분리하고, 필요하다면 `entityManager.clear()`로 persistence context 자체를 버린다. 편리한 공통 메서드일수록 내부에서 무엇을 하는지 의심한다.

JPA에 숨어 있는 지뢰의 총량은 직접 밟아 보기 전까지 알기 어렵습니다. 이번 조사에서 그 비용을 꽤 실감했습니다.

그럼, 또 봐요!
