---
title: "Writing It in Kotlin, Part 4"
date: 2023-09-03
translationKey: "posts/kotlin-code-in-my-style-4"
categories: 
  - kotlin
image: "../../images/kotlin.webp"
tags:
  - kotlin
---
I haven't written a series about writing in Kotlin for a while, so this is the fourth one.

## Sort One List by Another List

As a data structure, you may want to sort a List by List. For example, when a certain table has the ID of another table. If the IDs of this table are expressed as an array to represent the sort order, you would want to sort based on this array.

Of course, you can also sort by query, but the disadvantage of sorting by query is that you cannot cache the sorted results. So, I tried to think of a way to sort List by List.

First, suppose we have the following data. Assume that there is data called ImageBox that has multiple images, and that ImageBox's imageOrder contains the IDs of the images as an array.

```kotlin
data class ImageBox(
    val id: Int,
    val title: String,
    val imageOrder: List<Int> // Array of image IDs
)

data class Image(
    val id: Int,
    val url: String
)
```

Here, assume that the DB has an ImageBox table and an Image table, and the imageOrder of the ImageBox table contains the ID of the Image table as an array. In the app, we want to display images in the order of ImageBox's imageOrder.

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

In this case, you can sort as follows.

```kotlin
// Convert imageOrder into a map keyed by image ID, with the index as the value
val indexMap = imageBox.imageOrder.mapIndexed { index, value -> value to index }.toMap()
// Sort images based on imageOrder
val sortedImages = images.sortedBy { indexMap[it.id] }
```

The principle is simple, just convert the imageOrder array to a Map of Index and value, and sort based on that value. With this method, you can sort in the order of imageOrder without having to sort by query.

Also, by defining this type of sort as an extension function as shown below, you can reuse it in other places. Pass an array that represents the sort order to another, and pass a function that extracts the value that will be the basis for the sort order from the elements of the array you want to sort.

```kotlin
fun <T, U> List<T>.sortedBy(another: List<U>, keySelector: (T) -> U): List<T> {
    val indexMap = another.mapIndexed { index, value -> value to index }.toMap()
    return this.sortedBy { indexMap[keySelector(it)] }
}
```

In this case, you can use it like this:

```kotlin
val sortedImages = images.sortedBy(imageBox.imageOrder) { it.id }
```

If you want to omit even it in keySelector, there is also the following method.

```kotlin
fun <T, U> List<T>.sortedBy(another: List<U>, keySelector: T.() -> U): List<T> {
    val indexMap = another.mapIndexed { index, value -> value to index }.toMap()
    return this.sortedBy { indexMap[keySelector(it)] }
}
```

In this case, you can use it like this:

```kotlin
val sortedImages = images.sortedBy(imageBox.imageOrder) { id }
```

## List Paging

Paging is often done using a DB query, but there may be times when you want to process it only with code instead of a query. For example, when sending a request to an external API. In this case, the challenge is how to implement paging.

In this case, you can implement List paging by defining an extension function as shown below.

```kotlin
fun <T> List<T>.paginated(offset: Int, limit: Int): List<T> {
    // Return as-is when paging is not needed
    if (offest == 0 && limit < size) {
        return this
    }
    // Otherwise, derive a subList from offset and limit
    val fromIndex = (offset - 1) * limit
    return if (size <= fromIndex) {
        // Return an empty list if the page is out of range
        emptyList()
    } else {
        // Return the subList when the page is in range
        val toIndex = size.coerceAtMost(fromIndex + limit)
        subList(fromIndex, toIndex)
    }
}
```

For details, we use [coerceAtMost()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/coerce-at-most.html) to return size if a value larger than size is passed. This is of course to prevent IndexOutOfBoundsException.

## Exposed

Strictly speaking, [Exposed](https://github.com/JetBrains/Exposed) is an ORM, but there are some parts that can be even more convenient depending on how you write it in Kotlin, so I would like to introduce it this time as well.

### Sharing Insert and Update Logic

It is quite intuitive to write Inserts and Updates with Exposed, but in many cases you will end up writing the same processing for Inserts and Updates. Therefore, I thought about making Insert and Update common. For example, suppose you have the following code.

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

In this case, Insert and Update are just writing the same process in terms of setting values such as url and title other than ID. The more tables you have, the more code like this will be needed. Fortunately, Exposed is implemented in such a way that [UpdateBuilder](https://github.com/JetBrains/Exposed/blob/main/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/statements/UpdateBuilder.kt) for Update is inherited by [InsertStatement](https://github.com/JetBrains/Exposed/blob/main/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/statements/InsertStatement.kt) for Insert, so it can be shared as shown below.

```kotlin
private fun UpdateBuilder<Int>.setParametersFrom(image: Image) {
    this[url] = image.url
    this[title] = image.title
    this[description] = image.description
}
```

By defining such a function, the Insert and Update processing can be written as follows.

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

This is the same for batchInsert. [BatchInsertStatement](https://github.com/JetBrains/Exposed/blob/main/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/statements/BatchInsertStatement.kt) also inherits InsertStatement, so it can be shared as follows. This will allow you to write much cleaner code.

```kotlin
fun batchInsert(images: List<Image>): List<Image> {
    ImageTable.batchInsert(images) {
        this[id] = it.id
        this.setParametersFrom(it)
    }
    return images
}
```

### Implementing BatchUpsert

Exposed has `batchInsert`, but it does not provide `batchUpsert`. You can still implement batch upsert yourself by combining the query with application-side logic. Of course, there are also libraries such as [ExposedPowerUtils](https://github.com/PerfectDreams/ExposedPowerUtils), so using one of those is also an option.

However, there was a slight problem when using that library. First, the implementation of this library specifies a specific column and performs an update process based on that column if data already exists, and an insert process if it does not exist. The final code will first execute Insert, and if the specified column is duplicated, it will execute Update with `ON CONFLICT`.

Here, the table ID is a UUID generated on the application side, and if the column used as the basis for ON CONFLICT is FK, there was a problem that the ID would also be updated. So, based on the previous library implementation, I changed the implementation to update excluding PK. That is below. (Some unused arguments are omitted)

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
        val primaryKey = table.primaryKey?.columns?.toSet() // Get the primary key here
        val updateSetter = ((table.columns - keys.toSet())
            .let { columns -> primaryKey?.let { columns - primaryKey } ?: columns }) // Exclude the primary key here
            .joinToString { "${transactionManager.identity(it)} = EXCLUDED.${transactionManager.identity(it)}" }
        val onConflict =
            "ON CONFLICT (${keys.joinToString { transactionManager.identity(it) }}) DO UPDATE SET $updateSetter"
        return "${super.prepareSQL(transaction)} $onConflict"
    }
}
```

### Separate Transactions

This time, we decided to introduce a read replica, and for the time being we decided to use the read replica for GET-based APIs. However, read replicas do not accept update queries, so update queries must be sent to the primary.

As for the API implementation, a separate function is defined to explicitly rollback before throwing an exception, and I wanted to create something that could be used in a similar way to connect to a read replica. First, the existing functions used are as follows.

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

In Exposed, you can easily specify which DB to connect to by passing [Database](https://github.com/JetBrains/Exposed/blob/7c8d8a20e2efef5d0282ca6b703a433ecbf4a607/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/Database.kt) to the argument of [transaction](https://github.com/JetBrains/Exposed/blob/7c8d8a20e2efef5d0282ca6b703a433ecbf4a607/exposed-core/src/main/kotlin/org/jetbrains/exposed/sql/transactions/ThreadLocalTransactionManager.kt#L156). So, you can use this to define a function to connect to a read replica as follows.

```kotlin
fun <T> readReplicaTransactionWrapper(statement: Transaction.() -> T): T {
    val readReplicaDatabase: Database = getKoinModule(named("readReplicaDatabase")))
    return transaction(readReplicaDatabase) { statement() }
}
```

Since `Database` here is injected using [Koin](https://github.com/InsertKoinIO/koin), the following settings are additionally added.

```kotlin
// Fix the default database to the primary
val defaultDatabase: Database = getKoinModule(named("primaryDatabase"))
TransactionManager.defaultDatabase = defaultDatabase

// Allow Koin injection from a top-level function
fun <T> getKoinModule(qualifier: Qualifier? = null): T {
    return object : KoinComponent {
        val value: T by inject(qualifier)
    }.value
}
```

Now you can connect to the primary and read replica by simply changing the function name each time you use transaction.

## Finally

What did you think? This time too, I introduced some small tricks of Kotlin. Lately, I haven't been writing much complicated logic, and I haven't been doing much private development, so it's hard to come up with topics for my blog. It seems that some people refer to the articles on this blog from time to time, so I will try my best to update them more frequently.

I haven't updated my blog for a while, but Compose Multiplatform is now 1.5.0, and Java 21 will be released soon, so I'd like to update articles about them again soon.

See you soon!
