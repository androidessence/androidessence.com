---
layout: post
author: adam
title: Mastering Room Database Migrations
modified: 2020-05-14
published: true
tags: [databases]
categories: [android]
---

In the last post, we demonstrated the different types of [database relationships in Room]({{ site.baseurl }}{% link _posts/2020-04-27-room-relationship-recap.md %}). Next, we're going to explore another niched concept of Room database management: database migrations. A migration is a way to handle moving from one version of a database to another as users update your application from the play store. 

<!--more-->

---

This post is adapted from a YouTube video I made for [AsyncAndroid](https://twitter.com/asyncandroid).

<iframe width="560" height="315" src="https://www.youtube.com/embed/tUGUNU6DPtk" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

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

```kotlin
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

## Testing Migrations

Sometimes it may feel like manually testing your application is easier than writing your own unit tests. When it comes to database migrations, though, the opposite is often true. It is very time consuming to install a version of your app with the old database version, compile your app with the new database, and install on device to make sure everything worked. Especially if it doesn't - we have to start that whole process over again. 

So let's review how we can write our own tests to validate these migrations. First, we can create a test class inside our `androidTest` directory. This class needs two things before we can write tests:

1. A reference to a SupportSQLiteDatabase, which we'll create for each test.
2. A JUnit rule for a [MigrationTestHelper](https://developer.android.com/reference/androidx/room/testing/MigrationTestHelper) from the Room library. 

Here is our initial code:

```kotlin
@RunWith(AndroidJUnit4::class)
class StudentDatabaseMigrationsTest {

    private lateinit var database: SupportSQLiteDatabase

    @JvmField
    @Rule
    val migrationTestHelper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        StudentDatabase::class.java.canonicalName,
        FrameworkSQLiteOpenHelperFactory()
    )
}
```

Once we have that, we can begin to test a migration. We'll look at testing the migration from version 1 to version 2 in a step-by-step manner. 

### Step 1: Create A Database At The Starting Version

We want to test migration from version 1 to 2. The first thing we should do is create a database that's using version 1 of our schema, and insert some test data that we want to work with. Note that we have to insert this test data manually, and cannot use the Room DAO interfaces, as those are expecting the latest database schema. 

```kotlin
@Test
fun migrate1to2() {
    database = migrationTestHelper.createDatabase("test-database", 1).apply {
        // Since we've created a database with version 1, we need to insert stuff manually, because
        // the  Room DAO interfaces are expecting the latest schema.

        // Version 1 of the database has id, firstName, and lastName.
        execSQL(
            """
            INSERT INTO Student VALUES (1, 'Adam',  'McNeilly') 
            """.trimIndent()
        )

        close()
    }
}
```

### Step 2: Migrate To End Version

Next, we need to have the database migrate from version 1 to 2, and validate the new schema. We also supply the migrations that should be run, in this case just `MIGRATION_1_2`. 

```kotlin
@Test
fun migrate1to2() {
    database = migrationTestHelper.createDatabase("test-database", 1).apply {
        // ...
    }

    database = migrationTestHelper.runMigrationsAndValidate("test-database", 2, true, MIGRATION_1_2)
}
```

At this point, if Room was unable to find a valid migration path, your test will fail. This is a good sanity check, but I think we can take this test a little further and be more thorough. 

### Step 3: Validate Data

Beyond validating the schema changes, we can also query the data from the database to ensure the data is what we expect. Earlier in the test, we inserted a dummy row with an id, first name, and last name. Let's manually query our new database and make sure we have all the same data, as well as a default age of 0 - which is the default we defined in our migration. 

```kotlin
@Test
fun migrate1to2() {
    database = migrationTestHelper.createDatabase(TEST_DB, 1).apply {
        /....
    }

    database = migrationTestHelper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2)

    val resultCursor = database.query("SELECT * FROM Student")

    assertTrue(resultCursor.moveToFirst())
    
    val ageColumnIndex = resultCursor.getColumnIndex("age")
    val nameColumnIndex = resultCursor.getColumnIndex("firstName")
    val lastNameColumnIndex = resultCursor.getColumnIndex("lastName")

    val ageFromDatabase = resultCursor.getInt(ageColumnIndex)
    val firstNameFromDatabase = resultCursor.getString(nameColumnIndex)
    val lastNameFromDatabase = resultCursor.getString(lastNameColumnIndex)

    assertEquals(0, ageFromDatabase)
    assertEquals("Adam", firstNameFromDatabase)
    assertEquals("McNeilly", lastNameFromDatabase)
}
```

### Step 4: Ship With Confidence

Now that you've written a database migration, and written a unit test that validates an application can update this schema and support the same data after this upgrade, you are able to ship your new app version with confidence. As you add more migrations, you can continue this approach to validate migrating from version 2 to 3, or versions 1 to 3, or whatever combination you find most important. 

# Common SQL Statements For Migrations

Understanding the process of creating and testing database migrations is the fundamental building block that is consistent no matter what changes are being made to your database. The only time the details about the change matters is when you are writing the migration itself:

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // What SQL statement really goes here?
        database.execSQL("")
    }
}
```

Room does a great job of abstracting away most of the SQL statements required for database interraction. As a result, it's easy to forget the SQL required for the migration statements. That's why I've built [this cheat sheet](https://github.com/AdamMc331/mastering-room-migrations/blob/master/README.md) alongside a sample project with database migrations to help you understand what to do, depending on the change you're making. 

# Recap

Database migrations can be intimidating due to the number of steps required to create, implement, and test them. Not to mention the use of manual SQL statements that Room historically helps us avoid. I hope this post was helpful in breaking down those steps, and providing you with the confidence to ship your own migrations. 

Being able to save device data after making changes to the database schema is a really great user experience. Users don't want to feel like they're starting over every time they update your application. 

If you have any questions, let me know in the comments! If you find a particularly unique database migration, let me know how to help, and feel free to submit PRs to the public cheat sheet above.
