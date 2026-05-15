---
title: "JPA / Hibernate Pitfalls I Hit in a Real Product"
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

Recently I had a chance to investigate latency and handle errors in update APIs for a business system. The stack is Kotlin + Quarkus + Hibernate ORM, and the project originally started with Hibernate at the center. However, N+1 queries and implicit behavior during updates became expensive, so we introduced jOOQ in some places. At the moment, Hibernate and jOOQ coexist in the same project.

When I looked into the issue, a large part of the cost was not in the business logic itself, but in JPA / Hibernate behavior. I am not trying to say that Hibernate is bad, or that JPA must never be used. Still, when the p90 latency of update APIs reaches seconds in a real product, it is worth writing down what was actually happening.

For context, I once wrote a post on this blog titled [I want to use JPA rather than MyBatis](../spring-data-jpa/). At that time, I found it very attractive that SQL could be kept simple and the application could be built around entities. I still do not think that idea was completely wrong. But after tracing hot update paths in a real product, I can no longer ignore the hidden SELECTs, flushes, refreshes, and lifecycle costs behind the promise of "not having to write SQL."

In this post, I will hide the concrete business logic and focus on the pitfalls we actually hit and the workarounds we chose.

## Lazy proxies initialized while building response DTOs

The first major issue was converting the entity into a response DTO at the end of an update operation.

At the end of a PUT or POST request, we called something like `toResponseDto()` on the updated entity. Inside that function, getters for `@ManyToOne(fetch = LAZY)`, `@OneToMany`, and `@ManyToMany` were called one after another. From Hibernate's point of view, that means lazy proxies need to be initialized.

As a result, CloudWatch Performance Insights showed a large number of SELECTs for related entities in Top SQL, rather than queries for the business logic itself. It was even worse because DTO construction for child entities touched their own lazy relationships, creating a second layer of SELECT chains.

The fix here was simple. For update APIs, we changed the response to a minimal DTO such as `{ id, version }`. If the client needs details, it can call a separate GET endpoint. If the getters are never called, lazy initialization cannot happen in the first place.

## CascadeType.REFRESH loads every related collection

The next expensive behavior came from `CascadeType.REFRESH`.

There was a convenient method in a shared base repository that persisted, flushed, and refreshed the entity. Conceptually, it looked like this.

```kotlin
entityManager.persist(entity)
entityManager.flush()
entityManager.refresh(entity)
```

This is useful when you want to handle optimistic locking or reload values decided by the database. But if the target entity has relationships with `CascadeType.REFRESH`, `refresh()` changes its meaning. During refresh, Hibernate reloads the target entity, initializes lazy collections that are REFRESH cascade targets, and refreshes each element.

In other words, writing "propagate refresh to related entities" in one place made every update pay the read cost for related collections. Collections that were not touched by the business logic were still SELECTed, which made this cost hard to notice.

The workaround was to add sibling repository methods that do not refresh. Performance-critical paths use those instead. In tests, Hibernate `Statistics` can be used to assert that collection fetches do not increase, for example with `getCollectionFetchCount()`.

## orphanRemoval plus clear + addAll becomes full replacement

We also had a problem with aggregates that had `@OneToMany(orphanRemoval = true)` collections. On update, the code wrote the collection back like this.

```kotlin
children.clear()
children.addAll(newChildren)
```

With this code, Hibernate treats the existing elements as orphans and deletes them, then inserts the new elements. Even if only one row changed, the database sees it as DELETE all existing rows, then INSERT all new rows. From the business point of view it looks like a small diff, but in the database it becomes full replacement.

We replaced this path with diff application through jOOQ. That means explicit SQL such as `INSERT ... ON CONFLICT`, `UPDATE`, and `DELETE WHERE NOT IN (...)`, only for the actual changes.

One important point is that you do not necessarily have to throw away all JPA mappings used for GET responses. It was realistic to keep useful mappings for read paths while moving only the update path to a SQL builder.

## addAll alone can initialize a lazy collection

A similar but quieter issue was `addAll` by itself.

We thought we were only appending to a history-like child table, but every update path SELECTed all existing history rows first. Some of those rows included JSON columns, so the read cost was not trivial.

The cause was a helper method on the parent entity that internally called `addAll` on the collection. Hibernate's `PersistentBag` needs to compare with existing elements before `addAll`, so it initializes the collection. This means not only the full replacement pattern of `clear() + addAll()`, but also `addAll` by itself for append use cases can trigger a read of the existing collection.

The fix was to avoid touching the collection entirely when appending to child tables, and to insert directly with jOOQ. Calling an entity helper looks convenient, but if it initializes a lazy collection behind the scenes, that convenience becomes expensive on update paths.

## flush + refresh and the risk of PreUpdate firing again

Even after making response DTOs lighter, some update APIs still had visible latency. The shared `flush + refresh` behavior was still part of the problem.

The plain round trip cost matters, but the more dangerous part is the interaction with lifecycle listeners. For example, if `@PreUpdate` overwrites `updatedAt`, a naive implementation that reads `updatedAt` inside the handler and copies it to another column can break because the timing is wrong.

Another risk appears when jOOQ updates a side column and that value is written back to the managed entity. The entity becomes dirty. When the transaction commits, Hibernate flushes again, `@PreUpdate` fires again, and only `updated_at` moves forward unexpectedly.

Here, in addition to using the no-refresh path, we decided not to write jOOQ-updated values back into the managed entity. Side-column synchronization is done directly through the SQL builder, outside JPA lifecycle hooks. It is tempting to keep the in-memory entity "cleanly synchronized", but if touching it triggers another update, not synchronizing it is safer.

## Hibernate.initialize and loading the same aggregate twice

After reducing the response, we also reviewed calls such as `Hibernate.initialize(entity.someAssociation)`.

They may have been necessary for the old response DTO, but after switching to a minimal DTO, they became IO that nobody consumed. In some places, a ManyToMany association was SELECTed again immediately after refresh.

There were also duplicated loads, such as loading an entity to generate history, then loading the same entity again inside a shared update method. When some base methods accept a loaded entity and others accept an ID and load internally, these costs become hard to see.

This kind of issue cannot be fixed only by deleting individual `initialize` calls. The contract of the base methods also needs to be clarified.

## The round trip for pessimistic locking is itself expensive

Even after removing lazy initialization and refresh, some update APIs still had p90 latency in the hundreds of milliseconds or even seconds.

The shared repository method for "versioned updates" was doing the following by default.

1. `findById(id, LockModeType.PESSIMISTIC_WRITE)` with `SELECT ... FOR UPDATE`
2. `persistAndFlush`
3. `refresh`

Even when no cascade or lazy relationship was touched, these three round trips themselves dominated the cost. Especially on update paths where conflicts are rare, the pessimistic lock added "just in case" was simply adding one round trip every time.

Here we reviewed the update semantics and removed pessimistic locking from paths where `@Version`-based optimistic locking was enough. Base methods also need to be split by whether they lock and whether they refresh. It is worth being suspicious of convenient shared base methods that may be accumulating hidden costs.

## JSON columns can change representation when written through another path

Moving some writes to jOOQ also revealed a different kind of problem. When we stopped writing a `@JdbcTypeCode(SqlTypes.JSON)` column through Hibernate cascade and wrote it directly through the SQL builder, the JSON representation in the database changed slightly.

For example, property order, whether `null` fields are included, and naming rules can differ. If later comparison, hashing, or tests depend on the representation, this breaks things.

The cause was that Hibernate serialization used the Jackson `ObjectMapper` injected by Quarkus, while the jOOQ path created its own `ObjectMapper`. Since `ObjectMapperCustomizer` was not applied there, the same object did not produce the same JSON string.

The fix was to explicitly inject the configured `ObjectMapper` and serialize with the same settings as Hibernate. We also added tests to confirm that both paths produce bit-for-bit identical output.

When moving ORM column conversion to a SQL builder, the SQL may look the same. In reality, the implicit dependency on the serialization layer must be moved as well. The same issue can happen with columns using `AttributeConverter`.

## Related entities remain managed even after detach

Another kind of issue appeared in a bulk edit API. The flow was to load target entities with fetch joins, keep the values before the update, run a bulk UPDATE with jOOQ, reload the updated values, and then insert history records.

To keep the values before the update, the code called `entityManager.detach(it)` for each entity. The assumption was that detaching each entity from the persistence context would prevent it from interfering with later jOOQ updates.

But `detach()` only detaches the specified entity. Related entities and child collection elements loaded through fetch joins can remain managed in the persistence context. If jOOQ directly updates the database in that state, the database values and the managed entity values diverge. When Hibernate auto-flushes before the next query or at transaction commit, dirty checking on the remaining managed entities can collide with the latest database state.

An even more subtle problem appears when the value you thought was "before update" is actually the same instance still managed by Hibernate. Because of auto-flush or later processing, the before value inserted into history can already have become the after value in memory.

The fix was to copy the pre-update values into plain data class snapshots. Then, instead of individual `detach()` calls, we used `entityManager.clear()` to clear the whole persistence context. That also detaches related entities brought in by fetch joins. From that point on, diff calculation and history before values refer only to the snapshots.

If you read with ORM and then update with a SQL builder, it is important not to trust the loaded entity itself as a business value. Copy the required values into immutable snapshots, and cut off the persistence context early. Forgetting that `detach()` does not cascade is risky.

## As long as Hibernate is present, Kotlin build settings are affected

The last point is not about runtime behavior, but about build and language versions.

When writing JPA entities in Kotlin, there are basically two constraints.

- A no-arg constructor is required, so `kotlin-noarg` or `kotlin-jpa` is needed
- Classes need to be `open` for proxy generation, so `kotlin-allopen` is needed

These plugins are tightly tied to Kotlin itself, the Gradle plugin, and related plugins such as serialization. As a result, Kotlin version upgrades, migration to the K2 compiler, and removal of kapt get dragged into compatibility checks for JPA-related plugins.

Even update work that does not directly touch Hibernate can be slowed down by the presence of ORM. Personally, I think this cost is easy to miss when choosing an ORM.

## The pattern

When I put these issues side by side, the pattern is fairly consistent.

| Category | Core mechanism | Workaround |
|---|---|---|
| Lazy proxy chain | Getters during DTO construction trigger SELECT chains | Return minimal DTOs from update APIs |
| Cascade REFRESH propagation | `refresh()` initializes related collections | Add no-refresh paths |
| Full replacement by orphanRemoval | `clear() + addAll()` becomes DELETE all + INSERT all | Apply diffs with jOOQ |
| Implicit collection initialization | `addAll` alone initializes `PersistentBag` | Write directly without touching collections |
| flush + refresh | Round trips and `@PreUpdate` firing again | Remove refresh and bypass lifecycle hooks |
| Pessimistic locking | `SELECT ... FOR UPDATE` always adds a round trip | Remove it where optimistic locking is enough |
| Duplicate loads | The same aggregate is loaded in multiple layers | Clarify method contracts |
| JSON serialization | ORM and SQL paths produce different representations | Reuse the injected `ObjectMapper` |
| Persistence context and auto-flush | `detach()` does not detach related entities, and managed entities collide with later updates | Copy to snapshots and call `entityManager.clear()` |
| Build-layer impact | JPA Kotlin plugins slow down upgrades | Consider avoiding ORM in new projects |

The common theme is that operations that look small in code are converted by Hibernate's implicit behavior into large IO or flush costs. Getters, `refresh()`, `addAll()`, `clear()`, `initialize()`, `PESSIMISTIC_WRITE`, and `detach()` are each convenient on their own, but they become expensive when placed on hot update paths.

## What I would do in a new project

Based on this experience, I would avoid adding Hibernate ORM from the beginning in a new project and build around a SQL builder instead. In this project, that SQL builder is jOOQ.

This is a fairly large change in opinion from my earlier point of view. What changed my mind was not learning more JPA concepts, but tracing slow update APIs in production and peeling back, one by one, what was accessing the database and when.

Of course, removing Hibernate from an existing project all at once is usually not realistic. Mappings can still be useful when returning related graphs from GET APIs. So in the current project, Hibernate and jOOQ coexist, and the main direction is to remove Hibernate's implicit behavior from hot update paths.

For a new project, though, it feels cheaper to close off tools such as `@OneToMany`, `CascadeType.*`, `orphanRemoval`, `Hibernate.initialize`, `persistAndFlushRefresh`, and `entityManager.detach` from the start. The more convenient something looks, the more you eventually need to understand how many SELECTs it emits, when it flushes, which listeners run, and which instances are managed.

One small note: Hibernate Validator is not ORM. It is the reference implementation of Bean Validation, so using it is fine. What I want to avoid is the ORM itself, such as `quarkus-hibernate-orm`. If you grep configuration keys in CI, only `quarkus.hibernate-orm.*` should be rejected.

## Closing

The problems we hit were not isolated bugs. They were structural debt created by entities, aggregate boundaries, lifecycle listeners, and shared base repositories interacting with Hibernate's implicit behavior. Fixing one place does not prevent the same pattern from appearing somewhere else.

So the countermeasures are quite plain. Do not call getters in update responses. Do not refresh. Do not touch collections. Do not make pessimistic locking the default. When moving to a SQL builder, align serialization settings too. Values read through ORM should be copied into snapshots, and when needed, the whole persistence context should be discarded with `entityManager.clear()`. The more convenient a shared method is, the more carefully you should check what it does internally.

The total number of landmines hidden in JPA is hard to know until you step on them. This investigation made that cost very real to me.

See you!
