---
title: "How to map OneToMany with Exposed"
date: 2021-07-26
categories: 
  - exposed
image: "../../images/exposed.webp"
tags:
  - kotlin
  - exposed
  - map
---
A 1:N relationship is not uncommon for DB tables. For example, if you register as a member on an e-commerce site and want to be able to set multiple shipping addresses, it would be more reasonable and safe to separate the shipping address table and manage it separately, rather than adding a shipping address column to the member information table. The separate delivery destination table will generally have an N:1 relationship with the member information table.

However, the situation is different between a DB that prioritizes how to hold data and an application that processes and formats that data. For example, as shown above, if there can be multiple shipping destination records for one member information record, the data should be expressed in the following form when expressed in SQL.

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

However, applications rarely handle data in this way. In Kotlin, I think it is common to express that multiple records are included in one record as follows.

```kotlin
data class Member(
    val id: Int,
    val name: String,
    val mailingAdress: List<String>
)
```

Usually, objects like this are converted into JSON and used as responses to REST APIs. So, if you convert the previous record to JSON, it should look like the following.

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

The problem here is that converting objects to JSON is not that difficult (there are already libraries like [Jackson](https://github.com/FasterXML/jackson), [Gson](https://github.com/google/gson), [Kotlin Serialization](https://github.com/Kotlin/kotlinx.serialization))

So, in such a case, how should we map the records retrieved from the DB to objects? When using an ORM such as JPA, mapping to records is automatically done by appropriately using fields and annotations that express relationships between tables in the class, but [jOOQ](https://www.jooq.org), [Querydsl](https://querydsl.com), [Exposed](https://github.com/JetBrains/Exposed), When writing SQL using DSL with an ORM like [Ktorm](https://www.ktorm.org), you need to manually map the data. Since the obtained data is in the form of an array of rows, it is a bit difficult to figure out how to map it (efficiently).

So, this time I would like to talk about how to map One to Many records obtained using Exposed's DSL to objects in code.

## Select by tableThe easiest way is to select each individual table when retrieving records and then map them. Since the query is issued against individual tables, it also has the advantage of being clearer to write. For example, you can do something like:

```kotlin
transaction {
    // First, select from the Member table and map the rows to objects
    val member = Member.select { Member.id eq id }
        .first()
        .let {
            MemberDto(
                id = row[Member.id].value,
                name = row[Member.name],
                role = listOf(row[Mailing.role])
            )
        }

    // Select from the Mailing table and collect the rows into a list
    val mailingAddress = Mailing
            .select { Mailing.memberId eq member.id }
            .map { it[Mailing.address] }

    // Create a copy of the object and map the mailing address data into it
    member.copy(mailingAddress = mailingAddress)
}
```

Although this is the simplest method and is easy to understand in terms of code, it is not a good method from a transaction perspective. In Exposed, although it is possible to control the transaction unit by wrapping it in a `transaction` block, there is a problem that multiple queries can be issued in one go. Here, each time the Member table is queried, the Mailing table is also queried, so only one query is added, but if the number of tables that have a 1:N relationship with the Member table increases, the number of queries will increase. This time, the code is for one record, but the more records in the Member table to be queried, the more queries will be issued.

Also, it is not very efficient to create an instance of an object and then go to the trouble of copying it. This is also the same problem as the number of queries increasing, so the more records that are queried, the more instances of objects will be generated. Therefore, it can be said that the code does not consider performance or efficiency at all.

## join and map

`join` is probably more efficient for retrieving related data across multiple tables. First of all, the number of queries issued will be dramatically reduced compared to when selecting from individual tables. If expressed in [Big O notation](https://vmm.dev/cci/cci-0.md), which is often used in algorithms, the former is `O(N^2)`, and this should be expressed as `O(1)`.

If so, I understand that it would be ideal to use `join` as a query when acquiring data, but the problem is how to process the data acquired in this way. As I mentioned earlier, this is because some of the acquired data is duplicated. And you won't know if this is a duplicate until you run the query and check the results.

There are three possible methods here, so I'll introduce them one by one.

## reduce

The first method is to map each row obtained as a result of a query to an object, and then combine them with `reduce`. For example:

```kotlin
transaction {
    Member.leftJoin(Mailing)
        .select { (Member.id eq id) and (Mailing.memberId eq Member.id) }
        .map {
            // First, map the row to an object
            MemberDto(
                id = it[Member.id].value,
                name = it[Member.name],
                mailingAddress = listOf(it[Mailing.address])
            )
        }.reduce { acc, memberDto ->
            // Aggregate the objects into one result (mailingAddress is accumulated)
            acc.copy(
                mailingAddress = acc.mailingAddress + memberDto.mailingAddress
            )
        }
}
```

A possible problem with this approach is that first, instances of the object are created for the number of rows. There is only one Member record that we are trying to retrieve with this query, but the more Mailing records that are linked to that record, the more records there will be, and the more objects that will be generated. Also, since objects are copied not only in mapping but also in `reduce`, it is likely that objects for the number of rows are generated. The number of instances of the object is equal to the number of rows x2.

Another problem is that when you retrieve multiple Member records, they are all combined into one object. Therefore, this method can only be applied when retrieving a single record.

## groupBy

What happens if you convert the obtained records to a Map? Kotlin's Collection has a method called `groupBy`, and if you specify the key and value mapping method, a value in `List` format will be generated for one key. Since it is a Map, it is a good idea to map the Member object using the key and collect the Mailing records as the value. If the key is the same, it will be overwritten, so there should be no problem even if there are multiple records of the Member you want to retrieve. The code looks like this:

```kotlin
transaction {
    Member.leftJoin(Mailing)
        .select { (Member.id eq id) and (Mailing.memberId eq Member.id) }
        // The key maps the Member object, and the value aggregates Mailing records
        .groupBy({
            MemberDto(
                id = it[Member.id].value,
                name = it[Member.name],
            )
        }, { it[Mailing.address] })
        // Assign the Mailing records to the object held as the key
        .map { (key, value) ->
            key.copy(mailingAddress = value)
        }
}
```

This approach seems able to solve many of the same problems as the other methods I looked at. What concerns me is that the argument to `groupBy` is a lambda. Passing a function as an argument means that function is executed during iteration, so it seems possible that, just as with `reduce`, the same number of instances could be created. So let's look at the implementation of `groupBy`. The relevant code is shown below.

```kotlin
public inline fun <T, K, V> Iterable<T>.groupBy(keySelector: (T) -> K, valueTransform: (T) -> V): Map<K, List<V>> {
    return groupByTo(LinkedHashMap<K, MutableList<V>>(), keySelector, valueTransform)
}
```

In the implementation of `groupBy`, we simply pass our arguments and the instance of the Map to be created to a function called `groupByTo`. Now, let's take a closer look at the contents of `groupByTo`.

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

What we do know here is that we are still executing `keySelector` and `valueTransform` for the number of elements in the first Collection. Since it will be changed to a Map, unlike in `reduce`, no matter how many Member records there are, it will not be consolidated into one, but there is still the problem that multiple instances of the object will be created. So let's look for another method.

## MapThe last thing to consider is to declare an external Map and use it instead of collecting the selected rows in a Map. Map has a function called `compute`, which allows you to specify what to do with the key passed as an argument (what value to create and insert). For example, if a value does not exist for a specified key, it will be added as an element, and if it does exist, it will be possible to change the value. So, I feel that if you use this properly, you can solve the instance generation problem.

Let's first declare a Map that has nothing to do with transactions, and then execute `compute` on the selected data. In `compute`, if the specified key (Member id, etc.) is not in the Map, create a Member instance, and if it already exists, add Mailing data to that object. And when the loop is finished, it would be a good idea to get only the value of Map.

The above can be expressed in code as follows.

```kotlin
        // Map used to aggregate objects (key is Member.id)
val helperMap = mutableMapOf<Int, MemberDto>()

transaction {
    Member.leftJoin(Mailing)
        .select {
            (Member.id eq id) and (Mailing.memberId eq Mailing.id)
        }
        .forEach {
            helperMap.compute(it[Member.id].value) { key, value ->
                // If the value is not null, copy it and accumulate mailingAddress
                value?.copy(
                    mailingAddress = value.mailingAddress + it[Mailing.address]
                // If the value is null, create a new instance
                ) ?: MemberDto(
                    id = key,
                    name = it[Member.name],
                    mailingAddress = listOf(it[Mailing.address])
                )
            }
        }.let {
            // Convert the values into a List
            helperMap.map { it.value }
        }
}
```

What do you think? With this, I think I was able to avoid duplicate data and minimize instance creation. Of course, there is the problem that copying occurs every time you add mailingAddress, but I think this can be avoided by creating a dedicated setter.

One thing to keep in mind is that declaring the Map used here as a field will affect data integrity and the application's memory usage. Therefore, it is necessary to ensure that Map instances are created only within methods.

## Finally

When creating queries directly using a DSL, you can avoid problems like N+1 (related tables are always joined), which is a problem with ORMs like JPA, but the disadvantage is that you also have to write mappings to objects directly. Personally, I don't enjoy writing queries, but I think DSLs are better in that you can manage queries as code and write only the queries you need. Depending on the table structure and processing, it would be easier if the ORM could perform queries and mapping on its own.

However, in addition to the problem of how to obtain data with ORM, the problem of ``how to process duplicate data into different data'' is not necessarily limited to the case of obtaining records from a DB (for example, such data can also be obtained as a result of reading other APIs), so it seems necessary to consider various methods. At the moment, it seems like the best way is to use Map, but there may be other, more efficient methods.

See you soon!
