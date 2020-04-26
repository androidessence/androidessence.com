---
layout: post
author: adam
title: Room Relationship Recap
modified: 2020-04-27
published: true
tags: [databases]
categories: [android]
---

In this post, we're going to explore some advanced concepts of the [Room Persistance Library](https://developer.android.com/training/data-storage/room). Room is a great tool for storing complex data for your Android applications inside a SQLite database. As you begin to store more data in your applications, though, it can be difficult to determine how to organize all of it. 

We're going to demistify this, and break down everything you need to know about database relationships in the Room library. 

<!--more-->

---

This blog post is adapted from a YouTube video I made for [AsyncAndroid](https://twitter.com/AsyncAndroid).

<iframe width="560" height="315" src="https://www.youtube.com/embed/8-3GbQq8GMk" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

---

# The Problem

To start, let's look at a very common limitation we will encounter when working with a sqlite database. Here is a basic Room entity with three fields:

```kotlin
@Entity
data class Student(
    @PrimaryKey(autoGenerate = true)
    val studentId: Long = 0L,
    val firstName: String = "",
    val lastName: String = ""
)
```

Room will be able to take this entity, and successfully map it to a SQLite table with three columns:

![Android Essence](assets/rrr/student_table.png)

This works because Room is able to take primitive data types (numbers, strings, booleans) and map them to valid SQLite data types. We aren't able to store anything more complex than that in a single table. Let's try to make `Student` more complex and include an `Address` field:

```kotlin
@Entity
data class Student(
    @PrimaryKey(autoGenerate = true)
    val studentId: Long = 0L,
    val firstName: String = "",
    val lastName: String = "",
    val address: Address? = null
)

data class Address(
    val streetName: String = "",
    val streetNumber: String = ""
)
```

Room will not be able to convert this to a SQLite table, because Address is not a valid SQLite data type:

![Android Essence](assets/rrr/student_table_with_address.png)

This is the baseline for the rest of this post. While SQLite can't store complex data in a single table, our applications still need to store information about students such as addresses, and the class they're taking. The solution to storing this data can vary, so let's explore each of the options we have. 

# Embedded Properties

The previous example of storing a Student with their Address actually has a quick, one line solution, which is the [@Embedded](https://developer.android.com/reference/androidx/room/Embedded) annotation. This will tell the Room library to take each field from the `@Embedded` property, and map it to a column inside the entity we're creating. We can demonstrate this clearly by also specifying a `prefix` for the embedded columns. 

This means we can take this code:

```kotlin
@Entity
data class Student(
    @PrimaryKey(autoGenerate = true)
    val studentId: Long = 0L,
    val firstName: String = "",
    val lastName: String = "",
    @Embedded(prefix = "address_")
    val address: Address? = null
)

data class Address(
    val streetName: String = "",
    val streetNumber: String = ""
)
```

To create a database table with these columns:

![Android Essence](assets/rrr/student_table_with_address_mapped.png)

# Embedded Property Limitations

It's really great that we can get a one line solution to this problem with embedded properties, but it's important to understand their own limitations, too. Since these are just additional columns on our student table, we don't have a way to safely share this data across entities. Let's consider two students, Adam and John, who live at the same address:

| studentName | streetNumber | streetName  |
|-------------|--------------|-------------|
| Adam        | 111          | Wall Street |
| John        | 111          | Wall Street |

If I go into the database, it's possible that I only update one of the rows, causing the students to have different addresses:

| studentName | streetNumber | streetName  |
|-------------|--------------|-------------|
| Adam        | 111          | Wall Street |
| John        | 112          | Wall Street |

Looking at the database, I have no idea which one is right. This problem is something called an update anomaly. Now, if you aren't concerned about sharing information between rows, and you are just looking for a way to better structure your Room entities, then embedded properties are the perfect solution. 

If you need stronger data integrity, we need to move this information into a different entity, and create a link between the two. This connection between two entities is called an [entity relationship](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model). We're going to look at three types of relationships in this post. 

# One-To-Many Relationships

The first relationship to discuss is a one-to-many relationship. This is when you have exactly one instance of a parent entity, and zero or more instances of a child entity that relates to it. An example could be a `Student`, and any `Vehicle`s that they have registered to park at their university. Using our knowledge of creating Room entities, we may arrive on a solution kind of like this one:

```kotlin
/**
 * @property[ownerId] This property should map to a valid id in the [Student.kt] entity.
 */
@Entity
data class Vehicle(
    @PrimaryKey(autoGenerate = true)
    val vehicleId: Long = 0L,
    val ownerId: Long = 0L,
    val vehicleType: String = ""
)
```

This entity will compile, and we will be able to insert vehicles for valid students, query them, and all the functionality we want. There is a small problem here, though, which is that the two entities aren't actually linked at a database level. This means:

1. We can add vehicle entities with invalid student ids that don't actually exist. 
2. If we delete a student, the vehicles associated with them still exist in the database and we end up with [orphaned records](https://database.guide/what-is-an-orphaned-record/).

## Creating One-To-Many Relationships

To solve this problem, we need to define a foreign key relationship between the two tables. To create one of these, we need to specify three things:

1. The entity we want to reference.
2. The columns in the child entity that will be used to define this relationship.
3. The columns in the parent entity that should match the child columns to ensure the relationship.

Altogether, we will end up with this code:

```kotlin
@Entity(
    foreignKeys = [
        ForeignKey(
            entity = Student::class,
            childColumns = ["ownerId"],
            parentColumns = ["studentId"]
        )
    ]
)
data class Vehicle(
    @PrimaryKey(autoGenerate = true)
    val vehicleId: Long = 0L,
    val ownerId: Long = 0L,
    val vehicleType: String = ""
)
```

This means that we will be unable to insert into the `Vehicle` entity unless we use an ownerId that does exist inside the `Student` table already.

Optionally, we can add a fourth property which is what to do when a row is deleted from the Student table if there are associated Vehicle records. You can learn more about that [here](https://developer.android.com/reference/androidx/room/ForeignKey#onDelete()). 

## Querying One-To-Many Relationships

A common query with this structure is to request a student and the vehicles associated with them. This may seem like an intimidating challenge in the Room library, like you might have to learn join queries and write some complex SQL, but the Room library takes care of a lot of that for us. Let's try to build a query that returns the following response class:

```kotlin
data class StudentWithVehicles(
    val student: Student,
    val vehicles: List<Vehicle>
)
```

In order for Room to return this entity from a query, we need to supply two annotations.

1. The `@Embedded` annotation on student, so that Room will be able to map each column from the response to the Student entity, much like we saw with embedded properties. 
2. The `@Relation` annotaion on vehicles, so that Room will understand how the two entities are related. 

The resulting code is like this:

```kotlin
data class StudentWithVehicles(
    @Embedded
    val student: Student,
    @Relation(
        parentColumn = "studentId",
        entityColumn = "ownerId"
    )
    val vehicles: List<Vehicle>
)
```

Next, we can go into our DAO and write a query to fetch every student and their vehicles:

```kotlin
@Dao
interface UniversityDAO {
    @Query("SELECT * FROM Student")
    @Transaction
    fun fetchStudentsWithVehicles(): LiveData<List<StudentWithVehicles>>
}
```

You'll notice two annotations here as well:

1. The `@Query()` annotation which defines every student we want returned for the query.
2. The `@Transaction` annotation which is required because Room is actually going to run two queries behind the scenes - so they must run inside a database transaction.

## Room Behind The Scenes

It's rare that you'll ever dive into the generated code from a library like Room, but it may be helpful in scenarios like this one to understand what each of the annotations are actually doing. We can see all of the generated code for the `UniversityDAO_Impl.java` file in [this gist](https://gist.github.com/AdamMc331/311f1b84045af382132c8fa145be81cf), but let's just highlight what actually happens when we call `fetchStudentsWithVehicles()`. 

1. First, we run the SQL query specified in our `@Query()` annotation, which in this example is to request all students. Seen on [line 30](https://gist.github.com/AdamMc331/311f1b84045af382132c8fa145be81cf#file-universitydao_impl-java-L30).
2. Next, we get the index of all the relevant Student columns, thanks to the `@Embedded` annotation in our response object. Seen on [lines 39 to 43](https://gist.github.com/AdamMc331/311f1b84045af382132c8fa145be81cf#file-universitydao_impl-java-L39-L43).
3. Once we've requested all of the students, we request all vehicles that belong to one of those students. This is the second database query that's run. Seen on [line 132](https://gist.github.com/AdamMc331/311f1b84045af382132c8fa145be81cf#file-universitydao_impl-java-L132).
4. After running these queries, the Room library will then take the responses and map everything into the neat data class that we've defined and return a list of them. 

