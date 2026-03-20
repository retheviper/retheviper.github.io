---
title: "Kotlin으로 써 본 코드 4"
date: 2023-09-03
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
translationKey: "posts/kotlin-code-in-my-style-4"
---

오랜만에 다시 이어 쓰는 `Kotlin으로 써 본 코드` 시리즈 네 번째 글입니다.
## List 기준으로 List 정렬

어떤 `List`를 다른 `List`가 가진 순서 기준에 맞춰 정렬하고 싶을 때가 있습니다. 예를 들어 한 테이블이 다른 테이블의 ID 목록을 배열로 들고 있고, 그 배열이 곧 표시 순서를 의미하는 경우가 그렇습니다.
물론 쿼리에서 바로 정렬할 수도 있지만, 그 경우 정렬된 결과를 애플리케이션 쪽에서 그대로 캐시하기는 애매할 수 있습니다. 그래서 여기서는 코드에서 한 `List`를 다른 `List`의 순서를 기준으로 정렬하는 방법을 정리해 보겠습니다.
먼저 다음과 같은 데이터가 있다고 가정해 보겠습니다. 여러 이미지를 가지는 `ImageBox`가 있고, 그 안의 `imageOrder`에는 이미지 ID가 배열 형태로 들어 있어 표시 순서를 나타냅니다.
```kotlin
data class ImageBox(
    val id: Int,
    val title: String,
    val imageOrder: List<Int> // image ID 배열
)

data class Image(
    val id: Int,
    val url: String
)
```

여기서 DB로는 ImageBox 테이블과 Image 테이블이 있고 ImageBox 테이블의 imageOrder에는 Image 테이블의 ID가 배열로 들어 있다고 가정합니다. 그리고 앱에서는 ImageBox의 imageOrder 순서로 Image를 표시하고 싶습니다.
```kotlin
val imageBox = ImageBox(
    id = 1,
    title = "title",
    imageOrder = listOf(3, 1, 2, 5, 4)
)

val images = listOf(
    Image(id = 1, url = "url1"),
    Image(id = 2, url = "url2"),
    Image(id = 3, url = "url3"),
    Image(id = 4, url = "url4"),
    Image(id = 5, url = "url5")
)
```

이 경우 다음과 같이 정렬할 수 있습니다.
```kotlin
// imageBox의 imageOrder를 인덱스를 키로 하는 Map으로 변환
val indexMap = imageBox.imageOrder.mapIndexed { index, value -> value to index }.toMap()
// images를 imageOrder 기준으로 정렬
val sortedImages = images.sortedBy { indexMap[it.id] }
```

원리는 단순합니다. `imageOrder`를 `인덱스-값` Map으로 바꾸고, 그 값을 기준으로 정렬하면 됩니다. 이렇게 하면 쿼리 단계에서 정렬하지 않아도 원하는 순서를 코드에서 재현할 수 있습니다.
이 로직은 확장 함수로 빼 두면 다른 곳에서도 재사용하기 좋습니다. `another`에는 정렬 기준이 되는 목록을, `keySelector`에는 정렬할 요소에서 비교 키를 꺼내는 함수를 넘기면 됩니다.
```kotlin
fun <T, U> List<T>.sortedBy(another: List<U>, keySelector: (T) -> U): List<T> {
    val indexMap = another.mapIndexed { index, value -> value to index }.toMap()
    return this.sortedBy { indexMap[keySelector(it)] }
}
```

이 경우 다음과 같이 사용할 수 있습니다.
```kotlin
val sortedImages = images.sortedBy(imageBox.imageOrder) { it.id }
```

keySelector에서 it조차 생략하고 싶다면 다음과 같은 방법도 있습니다.
```kotlin
fun <T, U> List<T>.sortedBy(another: List<U>, keySelector: T.() -> U): List<T> {
    val indexMap = another.mapIndexed { index, value -> value to index }.toMap()
    return this.sortedBy { indexMap[keySelector(it)] }
}
```

이 경우 다음과 같이 사용할 수 있습니다.
```kotlin
val sortedImages = images.sortedBy(imageBox.imageOrder) { id }
```

## List 페이징

페이징은 보통 DB 쿼리 단계에서 처리하지만, 외부 API 응답처럼 이미 메모리 위에 올라온 `List`를 나눠야 할 때도 있습니다. 이럴 때는 확장 함수로 간단한 페이징 처리를 만들어 둘 수 있습니다.
```kotlin
fun <T> List<T>.paginated(offset: Int, limit: Int): List<T> {
    // 페이징이 필요 없으면 그대로 반환한다
    if (offset == 0 && limit >= size) {
        return this
    }
    // 페이징이 필요하면 offset과 limit을 기준으로 subList를 가져온다
    val fromIndex = (offset - 1) * limit
    return if (size <= fromIndex) {
        // 페이징 범위를 벗어나면 빈 목록을 반환한다
        emptyList()
    } else {
        // 페이징 범위 안이면 subList를 반환한다
        val toIndex = size.coerceAtMost(fromIndex + limit)
        subList(fromIndex, toIndex)
    }
}
```

여기서는 [coerceAtMost()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/coerce-at-most.html)를 사용해 `size`보다 큰 값이 들어와도 끝 인덱스가 넘치지 않도록 했습니다. 목적은 당연히 `IndexOutOfBoundsException`을 피하기 위해서입니다.
## Exposed

엄밀히 말하면 [Exposed](https://github.com/JetBrains/Exposed)는 ORM이지만, Kotlin 스타일로 정리해 두면 더 편해지는 부분도 있어서 이번에는 그것도 함께 다뤄 보겠습니다.
### Insert와 Update의 공통화

Exposed에서 `insert`와 `update`를 쓰는 일 자체는 어렵지 않지만, 둘 사이에 거의 같은 코드를 반복하게 되는 경우가 많습니다. 그래서 이 중복을 줄이는 방법을 생각해 보았습니다. 예를 들어 다음과 같은 코드가 있다고 해 봅시다.
```kotlin
fun insert(image: Image): Image {
    ImageTable.insert {
        this[id] = image.id
        this[url] = image.url
        this[title] = image.title
        this[description] = image.description
    }
    return image
}

fun update(image: Image): Image {
    ImageTable.update({ ImageTable.id eq image.id }) {
        this[url] = image.url
        this[title] = image.title
        this[description] = image.description
    }
    return image
}
```

이 경우 `id`를 제외하면 `url`, `title`, `description`을 채우는 로직은 사실상 같습니다. 테이블이 많아질수록 이런 중복도 함께 늘어납니다. 다행히 Exposed에서는 [UpdateBuilder](https://github.com/JetBrains/Exposed/blob/main/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/statements/UpdateBuilder.kt)를 [InsertStatement](https://github.com/JetBrains/Exposed/blob/main/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/statements/InsertStatement.kt)가 활용하는 구조라, 공통화하기가 비교적 쉽습니다.
```kotlin
private fun UpdateBuilder<Int>.setParametersFrom(image: Image) {
    this[url] = image.url
    this[title] = image.title
    this[description] = image.description
}
```

이러한 함수를 정의하면 Insert와 Update의 처리는 다음과 같이 쓸 수 있습니다.
```kotlin
fun insert(image: Image): Image {
    ImageTable.insert {
        this[id] = image.id
        it.setParametersFrom(image)
    }
    return image
}

fun update(image: Image): Image {
    ImageTable.update({ ImageTable.id eq image.id }) {
        it.setParametersFrom(image)
    }
    return image
}
```

이 패턴은 `batchInsert`에도 그대로 적용할 수 있습니다. [BatchInsertStatement](https://github.com/JetBrains/Exposed/blob/main/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/statements/BatchInsertStatement.kt) 역시 같은 계열이기 때문에 아래처럼 정리할 수 있습니다.
```kotlin
fun batchInsert(images: List<Image>): List<Image> {
    ImageTable.batchInsert(images) {
        this[id] = it.id
        this.setParametersFrom(it)
    }
    return images
}
```

### BatchUpsert 구현

Exposed에는 `batchInsert`는 있지만 `batchUpsert`는 없습니다. 물론 [ExposedPowerUtils](https://github.com/PerfectDreams/ExposedPowerUtils) 같은 라이브러리를 써도 되지만, 제 경우에는 그대로 쓰기 어려운 부분이 있었습니다.
해당 라이브러리는 특정 컬럼을 기준으로 중복 여부를 판단해, 있으면 `UPDATE`, 없으면 `INSERT`하는 방식입니다. 최종 SQL도 `INSERT ... ON CONFLICT DO UPDATE` 형태로 만들어집니다.
문제는 테이블의 ID를 애플리케이션 쪽 UUID로 관리하고 있고, `ON CONFLICT` 기준 컬럼이 FK일 때는 ID까지 함께 갱신될 수 있다는 점이었습니다. 그래서 기존 구현을 바탕으로 PK를 제외하고만 갱신하도록 바꿨습니다. 아래 코드는 그 핵심 부분입니다. (불필요한 인자는 생략했습니다)
```kotlin
fun <T : Table, E> T.batchUpsert(
    data: Collection<E>,
    vararg keys: Column<*> = (primaryKey ?: throw IllegalArgumentException("primary key is missing")).columns,
    body: T.(BatchInsertStatement, E) -> Unit
): BatchInsertOrUpdate {
    return BatchInsertOrUpdate(table = this, keys = keys).apply {
        data.forEach {
            addBatch()
            body(this, it)
        }
        execute(TransactionManager.current())
    }
}

class BatchInsertOrUpdate(
    table: Table,
    private vararg val keys: Column<*>,
) : BatchInsertStatement(table, false, false) {
    override fun prepareSQL(transaction: Transaction): String {
        val transactionManager = TransactionManager.current()
        val primaryKey = table.primaryKey?.columns?.toSet() // 여기서 primaryKey를 가져온다
        val updateSetter = ((table.columns - keys.toSet())
            .let { columns -> primaryKey?.let { columns - primaryKey } ?: columns }) // 여기서 primaryKey를 제외한다
            .joinToString { "${transactionManager.identity(it)} = EXCLUDED.${transactionManager.identity(it)}" }
        val onConflict =
            "ON CONFLICT (${keys.joinToString { transactionManager.identity(it) }}) DO UPDATE SET $updateSetter"
        return "${super.prepareSQL(transaction)} $onConflict"
    }
}
```

### 거래 분리

이번에는 리드 복제본을 도입하면서, 조회 계열 API에는 우선 리드 복제본을 사용하게 되었습니다. 다만 읽기 복제본은 갱신 쿼리를 받을 수 없으므로, 쓰기 쿼리는 기본 DB로 보내야 합니다.

또 API 구현에서는 예외를 던지기 전에 명시적으로 롤백하기 위해 별도의 함수를 정의해 두고 있었습니다. 그래서 같은 사용감으로 리드 레플리카에 접속하는 함수도 만들고 싶었습니다. 먼저 기존에 사용하던 함수는 다음과 같습니다.
```kotlin
fun <T> transactionWrapper(statement: Transaction.() -> T): T {
    return transaction {
        try {
            statement()
        } catch (ex: Exception) {
            TransactionManager.current().rollback()
            throw ex
        }
    }
}
```

Exposed에서 [`transaction`](https://github.com/JetBrains/Exposed/blob/master/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/transactions/ThreadLocalTransactionManager.kt)에 [`Database`](https://github.com/JetBrains/Exposed/blob/master/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/Database.kt)를 인수로 전달하면 쉽게 연결할 DB를 지정할 수 있습니다. 따라서 이를 사용해 읽기 복제본에 연결하는 함수를 다음과 같이 정의할 수 있습니다.
```kotlin
fun <T> readReplicaTransactionWrapper(statement: Transaction.() -> T): T {
    val readReplicaDatabase: Database = getKoinModule(named("readReplicaDatabase")))
    return transaction(readReplicaDatabase) { statement() }
}
```

여기서 `Database`는 [Koin](https://github.com/InsertKoinIO/koin)으로 주입하고 있으므로, 추가로 다음과 같은 설정을 넣었습니다.
```kotlin
// 기본 DB는 primary로 고정
val defaultDatabase: Database = getKoinModule(named("primaryDatabase"))
TransactionManager.defaultDatabase = defaultDatabase

// Top level function에서 Koin inject를 쓰기 위한 설정
fun <T> getKoinModule(qualifier: Qualifier? = null): T {
    return object : KoinComponent {
        val value: T by inject(qualifier)
    }.value
}
```

이제 `transaction`을 사용할 때 함수 이름만 바꾸면 기본 DB와 읽기 복제본을 나눠 쓸 수 있습니다.
## 마지막으로

이번 편에서는 조금 더 실무적인 Kotlin 패턴을 정리해 봤습니다. 최근에는 예전만큼 복잡한 개인 프로젝트를 자주 하지는 않아서 블로그 소재를 고르는 일이 쉽지 않지만, 그래도 가끔 이 시리즈를 참고해 주시는 분들이 있는 것 같아 가능한 한 이어 가 보려고 합니다.

한동안 업데이트가 뜸했지만, Compose Multiplatform도 1.5.0에 들어섰고 Java 21 출시도 가까웠던 시기라 조만간 그쪽 이야기도 다시 다뤄 보고 싶었습니다.
