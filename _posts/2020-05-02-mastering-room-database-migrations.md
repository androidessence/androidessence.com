---
layout: post
author: adam
title: Mastering Room Database Migrations
modified: 2020-05-02
published: true
tags: [databases]
categories: [android]
---

In the last post, we demonstrated the different types of [database relationships in Room]({{ site.baseurl }}{% link _posts/2020-04-27-room-relationship-recap.md %}). Next, we're going to explore another niched concept of Room database management: database migrations. A migration is a way to handle moving from one version of a database to another as users update your application from the play store. 

<!--more-->

# Starting Point

Since we're exploring a niched topic within Room databases, I recommend at least reviewing the [getting started](https://developer.android.com/training/data-storage/room) documentation for Room. You don't need to be an expert, but understanding the initial annotations is important. 

For this post, we're going to look at an application that has a Room database with one single entity, representing a Student:

```kotlin
@Entity
data class Student(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0L,
    val firstName: String = "",
    val lastName: String = ""
)
```

There's a DAO for inserting and fetching students:

```kotlin
@Dao
interface StudentDAO {
    @Insert
    fun insertStudent(student: Student): Long

    @Query("SELECT * FROM Student")
    fun fetchStudents(): List<Student>
}
```

And a database class with a helper method to create a database reference:

```kotlin
@Database(entities = [Student::class], version = 1)
abstract class StudentDatabase : RoomDatabase() {
    abstract fun studentDAO(): StudentDAO

    companion object {
        fun createDatabase(appContext: Context): StudentDatabase {
            return Room.databaseBuilder(
                appContext,
                StudentDatabase::class.java,
                "student-database.db"
            )
                .allowMainThreadQueries()
                .build()
        }
    }
}
```

In this sample we've allowed main thread queries to keep the focus on the database work itself. It is highly recommended that you do not do database operations on the main thread in production.

# What Happens If I Don't Migrate?

Let's start by looking at a couple common errors that will happen if you change the database used by your applications. First, we can try to add a new field onto Student:

```kotlin
@Entity
data class Student(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0L,
    val firstName: String = "",
    val lastName: String = "",
    val age: Int = 0
)
```

If we run our application right now, we'll get the following error:

```
Caused by: java.lang.IllegalStateException: Room cannot verify the data integrity. Looks like you've changed schema but forgot to update the version number. You can simply fix this by increasing the version number.
```

When Room says that it cannot verify the data integrity, what it means is that the schema of the database we're trying to create does not match the schema of the database that's already on the device. In this case, our new database schema includes a column that wasn't there before. 

Thankfully, we've got a helpful error message, we just need to increase the database version number!

```kotlin
@Database(entities = [Student::class], version = 2)
abstract class StudentDatabase : RoomDatabase() {
    abstract fun studentDAO(): StudentDAO

    // ...
}
```

Unfortunately, the app will crash again with this new error:

```
Caused by: java.lang.IllegalStateException: A migration from 1 to 2 was required but not found. Please provide the necessary Migration path via RoomDatabase.Builder.addMigration(Migration ...) or allow for destructive migrations via one of the RoomDatabase.Builder.fallbackToDestructiveMigration* methods.
```

Room now recognizes that we are using a different version of the database than what was previously there, but Room doesn't have the capabilities to understand how to migrate from the previous version to the new one. The error message hints at a couple options, and this post is going to review each of them.

# The Easy Way Out

First, we'll look at what might be considered the easy way out - destructive migrations. Ultimately, migrations are only important if you want to keep data on your device even after the user updates your application to a new database. For apps that work purely offline, this is very important. You may decide, though, that you don't care if a user's database is cleared out upon upgrade. Maybe the database is just a caching mechanism, and the user experience won't really be hindered by this. 

If that's the case, we can just supply a call to `fallbackToDestructiveMigration()` when we create the database:

```kotlin
fun createDatabase(appContext: Context): StudentDatabase {
    return Room.databaseBuilder(
        appContext,
        StudentDatabase::class.java,
        "student-database.db"
    )
        .allowMainThreadQueries()
        .fallbackToDestructiveMigration()
        .build()
}
```

With this approach, we still need to update the database version each time the schema changes, but if Room comes across a migration that it can't support, it will fall back by destroying the entire database and recreating it with the new schema. This is certainly the shortest solution to database migrations, but may not be the best user experience. Discuss with your team if you think this approach is worth the tradeoffs. 

## Specific Destructive Migrations

Maybe your application supports migrations in some cases, but from other versions you're willing to sacrifice a destructive migration. If that's the case, you can use the [fallbackToDestructiveMigrationsFrom](https://developer.android.com/reference/kotlin/androidx/room/RoomDatabase.Builder#fallbacktodestructivemigrationfrom) method. 

```kotlin
fun createDatabase(appContext: Context): StudentDatabase {
    return Room.databaseBuilder(
        appContext,
        StudentDatabase::class.java,
        "student-database.db"
    )
        .allowMainThreadQueries()
        .fallbackToDestructiveMigrationsFrom(1, 2)
        .build()
}
```

# Downgrade Destructive Migrations

It's possible that a database version is being downgraded. We may see this happen often in development, if developer "A" is working on a new database change, but developer "B" is not. Switching between these branches will cause a migration error like we saw earlier. We could resolve this case by only allowing destructive migrations if we notice the database version is being downgraded:

```kotlin
fun createDatabase(appContext: Context): StudentDatabase {
    return Room.databaseBuilder(
        appContext,
        StudentDatabase::class.java,
        "student-database.db"
    )
        .allowMainThreadQueries()
        .fallbackToDestructiveMigrationOnDowngrade()
        .build()
}
```

This approach could make development smoother, while still ensuring that you handle migrations when upgrading the database, which is the flow your users are likely to encounter. 