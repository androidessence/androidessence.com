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

## Downgrade Destructive Migrations

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

# The Data Focused Way

If you want to preserve your applications data while updating to a new schema, it requires a little more work. We're going to break down that process and help create a tool set for having the confidence to write and create migrations that are maintainable for developers, and keep your users happy without losing their on-device data.

## Project Set Up

Before creating the database migrations, there's some recommended project configurations. We need to do three things:

1. Update our build.gradle file to export the database schema as a JSON file. This will be used when we want to test our database migrations, as it will tell the Room library what the scema for the database is at a specific version. 
2. We need to modify our build.gradle file to make those schemas part of the source sets for the androidTest package, so that our database tests will be able to reference them. 
3. We need to add a dependency to the room-testing artifact to get access to classes that help us test database migrations. 

Here is what our app module's build.gradle file looks like after that, trimming it down to those three bullets:

```groovy
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                // Will export the schema to this directory during the "build" command.
                arguments = ["room.schemaLocation": "$projectDir/schemas".toString()]
            }
        }
    }
    
    // Tells our androidTest project where to find the schema
    sourceSets {
        androidTest.assets.srcDirs += files("$projectDir/schemas".toString())
    }
}

dependencies {
    // Brings in helpers for testing these migrations
    androidTestImplementation "androidx.room:room-testing:$room_version"
}
```

If we build our project after this, we'll notice a new class inside `$projectDir/schemas` called `1.json`, or whatever your current database version is. Here is a [sample](https://github.com/AdamMc331/mastering-room-migrations/blob/master/app/schemas/com.adammcneilly.masteringroommigrations.StudentDatabase/1.json) of what that looks like, but we'll revisit the importance of this file shortly. 

## Making The Change

Once we have done this project setup, and we've exported the json schema of our current database version, we can go ahead and change our database. Let's make the same change as earlier - add a new field on `Student` called `age` and update our database version to 2. If we rebuild now, we'll see a new file for `2.json` in the schema directory.

It's important to make sure we exported the original before making this change - having a file for each database version is what Room is going to use when we write tests for this, discussed in a later section. 

## Create The Migration

In order to support the migration from one version to another, we need to create our own implementation of the [Migration](https://developer.android.com/reference/kotlin/androidx/room/migration/Migration) abstract class:

```
/**
 * When Migrating from version 1 to version 2, we added the age property on the student
 * entity.
 */
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // ...
    }
}
```

Here are tips for creating these properties:

1. A common naming convention found in the Room documentation is `MIGRATION_$startVersion_$endVersion`. You can use whatever you are comfortable with, though.
2. The parent constructor takes an argument for the starting and ending version of the migration. Here, we're migrating from version 1 to version 2. 
3. Since `Migration` is an abstract class, we  need to override the one method called `migrate`, which gives us a reference to the database we're migrating. 
4. I recommend documenting your migrations with the changes that are relevant to it. This will help you and your team when reviewing these in the future. 

Inside of the `migrate` method, we have to write the actual SQL code for handling the migration. The code you put here will vary depending on what's changed - adding a field, removing a field, adding a new table, and more cases. At the end of this post you'll find some resources to help you with determining what the required statements are. 

For this particular change, we need to alter the `Student` table by adding a new column:

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE Student ADD COLUMN age INTEGER NOT NULL DEFAULT 0")
    }
}
```

Since `age` is non-null in the database, we need to supply a default which is going to be applied to all of the existing rows. If the new field were nullable, we could omit the `DEFAULT` property, if we wanted to.

## Add Migration To Database Builder

Now that we've defined the expected behavior for migrating from version 1 to version 2, we need to add this migration to our database builder, so that when we attempt to create a database for a phone that has our old application version, Room will know how to support this user:

```kotlin
fun createDatabase(appContext: Context): StudentDatabase {
    return Room.databaseBuilder(
        appContext,
        StudentDatabase::class.java,
        "student-database.db"
    )
        .addMigrations(MIGRATION_1_2)
        .allowMainThreadQueries()
        .build()
}
```

We're now safe to run our app without experiencing any of the crashes from the beginning of this post! 

