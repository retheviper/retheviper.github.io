---
title: "I want to use JPA rather than MyBatis"
date: 2020-07-10
categories: 
  - spring
image: "../../images/spring.webp"
tags:
  - database
  - jdbc
  - jpa
  - spring
---

Personally, I don't think it's a good idea to format data using the DB itself, such as queries or DB plug-ins, and I think it's better to limit SQL statements to the minimum level of CRUD when there is some kind of "processing" involved, and to implement an implementation in which all necessary work is handled by the app. The reason is that I think that method is better in terms of overall system performance, application independence from the DB, and ease of processing.

First, regarding performance, if the DB server and AP server use machines with the same physical performance, it is actually possible to improve performance with optimized queries. However, this is not always the case as the amount of data and requests to be processed increases. When the number of users increases in a web application, scaling out [^1] and scaling up [^2] must be considered, but when scaling out, apps that are highly dependent on the DB may cause problems. There is no difference in the operation of the application even if multiple AP servers are used, but when the number of DB servers increases, a new problem of data consistency arises. If you want to guarantee that the same data is stored in every DB, I think it is better to scale out the AP server than to scale out the DB server. So in this case, it would be better to leave the "processing" of the data to the app.

Next, regarding the independence of the application's DB, recently more and more systems are migrating from the expensive [Oralce](https://www.oracle.com/database/technologies) to the free [PostgreSQL](https://www.postgresql.org) and [MySQL](https://www.mysql.com). However, the more parts that depend on DB queries and plugins, the more difficult the migration may become. You need to consider everything from slightly different column data types to incompatible queries and functions. Therefore, it is better to make SQL statements as simple as possible to increase the number of transplants.

Finally, regarding ease of processing, in the case of RDBS, which is currently the mainstream database, the most important issue is how to thoroughly and efficiently store data. The table structure after normalization is not suitable for processing data. So in an app, you create and use entities as needed, but when you write SQL for those entities, you also have to draw complicated joins, wheres, etc. If an entity contains other entities, you must consider whether or not to join when writing a query.

For these reasons, I was wondering if there was a way to eliminate complex queries and develop applications as much as possible. At first, I used JDBC, and then I started using MyBatis, but even if I used MyBatis, I still needed to write queries for each case. (Plus, xml may be required.) Recently, due to telework, I have no time to commute, and after work I am studying and implementing my own applications, but I was wondering if there was an easier and more application-friendly DB API, and what I discovered was JPA.

## What is JPA?

JPA stands for [Java Persistence API](https://www.oracle.com/java/technologies/persistence-jsp.htm) and is a standard API provided by Java. However, even if it is a standard API, it is only defined as an Interface, so an implementation of this is required. [Hibernate](https://hibernate.org) is a famous implementation of JPA, and [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#reference) exists as a module that organizes it even more easily.

The introduction is quite long, but what I want to say in conclusion is that using Spring Data JPA makes development easier than using other libraries or frameworks. There were probably other people who were thinking the same thing as me when it came to SQL-centered development, and JPA makes it possible to work with DBs without touching SQL at all, such as automatically creating simple queries and automatically joining tables. Personally, I think it would be more efficient to use JPA than MyBatis, which is also commonly used as an ORM. Perhaps for that reason, overseas users seem to be overwhelmingly more likely to use JPA than other ORMs. So, if you are still using MyBatis, I would like you to try using JPA at least once.

Now, let's actually introduce the features and usage of JPA (Spring Data JPA) one by one.

## Features of JPA

First, let's talk about the features of JPA. I think it's more like an advantage of using JPA than a feature.

### automatic graph exploration

First, suppose we have a class like this.

```java
class Team {
    private long id;
    private String name;
}
```

And there are players on the team.

```java
class Player {
    private long id;
    private String name;
    private Team team;
}
```

If you want to query this player object from the DB using existing SQL, the following steps are required.

1. Select Team table
1. Embed the selected data in the Team object
1. Select Player table
1. Embed the selected data in the Player object
1. Set Team object to Player object

This step increases as the number of table joins increases. Also, the SQL will be even longer. For example, what if the Team class contains a class called Region as a field, and Region also contains a class called Country? You need to create a query that queries and joins each table, and you also need to map all the objects. If there are any omissions in this step, your code will not work as expected.

JPA automatically resolves such reference relationships. For example, you can do the following without doing anything:

```java
// Retrieve a Player object from the Repository
Player player = repository.findById(1);
// Retrieve the Team the Player belongs to
Team team = player.getTeam();
```

## Performance optimization

When you look at the part that automatically searches the graph, you might think that it is doing something pointless like querying the Team table every time you query the Player table, but in reality, due to lazy loading, Team issues another Select statement and queries the moment it executes `getTeam()`. Therefore, if you need information only about Player, you will only query that information.

Now, you may be wondering whether transactions may occur several times in some cases, but JPA seems to solve this problem by specifying cache, batch, and fetch types. These work as follows.

- Cache: The second and subsequent Selects for the same entity (if it is the same transaction) will query from the cache.
- Batch: Provides a batch method and implements the ability to transfer multiple queries with a single commit.
- Fetch type: You can choose between lazy loading or joining and querying from the beginning depending on the annotation.

### automatic query generation

Earlier we looked at an example of retrieving an object from a Repository, but this is possible because JPA automatically generates a query. By using the annotations and interfaces provided by JPA, you can use automatically generated queries.

In other words, by linking the classes used as entities and JPA to annotations and interfaces, there is no need to create queries and map them to objects each time. Of course, you can use your own queries, so there is no problem if you want to use automatically generated queries.

Generating this query is not limited to CRUD of entities. It also automatically creates DDL if any changes occur to the entity (if a field needs to be added). You can choose whether the automatically generated DDL is Drop-Create or Alter using the Hibernate settings in the application properties.

## How to use JPA

Now, let's take a look at how we can actually use JPA as code.

### Entity

In JPA, entities and their fields (table columns and reference relationships) can be easily defined using annotations. For example:

```java
// Mark as an Entity
@Entity
class Car {

    // Auto-incrementing PK
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    // Unique and non-null settings
    @Column(nullable = false, unique = true)
    private String name;

    // Many-to-one relationship with Company
    @ManyToOne
    @JoinColumn(name = "company_id")
    private Company company;

    // One-to-many relationship with Insurance, using lazy loading
    @OneToMany(fetch = FetchType.LAZY)
    @JoinColumn(name = "insurances_id")
    private List<Insurance> insurances;
}
```

By attaching getters and setters to Entity in this way, the Company associated with this class will be able to refer to table information such as Insurance when it is selected.

### Repository

JPA provides several interfaces called Repositories. Commonly used ones include [JpaRepository](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/JpaRepository.html), [CrudRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html), and [PagingAndSortingRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/PagingAndSortingRepository.html), and each of them basically provides different methods. For example, CrudRepository basically provides methods for CRUD (find() for Select, save() for Upsert, delete() for Delete, etc.), and PagingAndSortingRepository provides methods for paging (pagination), so it basically has all the commonly used methods. For example:

```java
// Count records
long count = repository.count();

// Fetch all records
Iterable<Car> cars = repository.findAll();

// Fetch a single record
Optional<Car> car1 = repository.findById(1);

// Delete a single record
repository.delete(car2);

// Save a record (upsert)
Car saved = repository.save(car3);
```

If you first define entities and create a Repository provided by JPA for each entity (extend), you will be able to issue queries using that Repository. Also, if the basically provided methods are not sufficient, you can write the [query method](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods) directly in the Repository according to the naming convention and the query will be automatically created, or you can directly write the required query with the [@Query](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.at-query) annotation.

```java
public interface CarRepository extends CrudRepository<Car, Long> {

    // Custom query method that searches by name
    boolean existsByName(String name);

    // When specifying a query in an annotation
    @Query("select c from Car c where c.name = ?1")
    Optional<Car> findByName(String name);
}
```

## else

In 2018, [Spring Data JDBC](https://spring.io/projects/spring-data-jdbc) was also announced. Compared to going through the JPA + Hibernate layer between Spring Data JPA and JDBC APIs, Spring Data JDBC seems to have an advantage in terms of performance by directly calling the JDBC API. It seems that other problems with JPA have been resolved by simplifying the implementation. For example, eliminating cache and lazy loading (I introduced this as an advantage of JPA...)

However, since Spring Data JDBC was only released in 2018, it seems that there are still many points to consider before migrating from JPA, such as not supporting features such as automatically generating queries using just the query method, and the naming conventions for tables and columns being different from JPA. This kind of thing is like the relationship between Node.js and Deno.

I don't know much about Spring Data JDBC myself, so I only looked at the sample code and tested it, but the structure seemed like someone who already had experience with Spring Data JPA could adapt it right away. Even though the package has changed, the usage of the annotations and interfaces themselves have not changed significantly, so there is no big difference in feeling. It was noticeable that there was no need to use annotations like `@Entity`. So it might be a good idea to try it out for personal projects. (I think there are still things that need to be verified at the enterprise level)

Also, with the introduction of [Webflux](https://docs.spring.io/spring-framework/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/web-reactive.htm), which allows reactive programming from Spring 5, it seems that [Spring R2DBC](https://spring.io/projects/spring-data-r2dbc), which also supports non-blocking, is also attracting attention. It seems that there is no need to introduce Webflux to an existing Spring project right away, so it may not be necessary to introduce Spring R2DBC right away, but there are certainly cases [^3] where performance can be improved by non-blocking, so it may be worth considering introducing it in some cases.

## lastly

In my case, I wanted to eliminate `XML` as much as possible and manage everything with just Java code while building an application with Spring, so I was attracted to it because it automatically generates queries, but when I actually used it, I found that it was easy to use (though I think it has its quirks), and that it was easy to fix even if the table changed midway through. It seems that JPA is more commonly used than other ORMs, as it is the standard in terms of occupancy, but if you are like me and have only used MyBatis, I think it's worth giving it a try.

In addition to JPA, as I introduced, there are various libraries for Spring Data alone, each with a different philosophy, so I feel like I still have a long way to go even if I only have a little understanding of one. However, there is a lot of knowledge that can be gained just by looking into the history and philosophy behind the creation of a single library, and all of it is so fresh that it doesn't go to waste, so I think it's meaningful. I hope that by gathering information like this, it will become valuable information for someone else. I will continue to research and post on my blog. See you soon!

[^1]: Increase the number of servers.
[^2]: To increase the machine power of the server.
[^3]: Cases where the thread pool is too small or the processing time for one request is too long, creating a bottleneck.
