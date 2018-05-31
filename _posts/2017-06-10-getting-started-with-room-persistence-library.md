---
layout: post
author: adam
title: Getting Started With Room Persistence Library
description: Teaches how to build a todo list using Room Persistence Library
modified: 2017-06-10
published: true
tags: [kotlin, room, sqlite]
categories: [android]
---

This year at Google I/O, the Android team announced [Android Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html) a combination of new, helpful libraries for Android development. One that particularly interested me was the [Room Persistence Library](https://developer.android.com/topic/libraries/architecture/room.html), which is an abstraction layer of SQLite designed to make database access and creation a lot easier. Right off the bat it reminded me of [Realm](https://realm.io/products/realm-mobile-database/), which I learned about at [Droidcon NYC](https://www.youtube.com/watch?v=QT7XD1hifkU) and really admired, so I decided to dive in and build a todo list using Room & RxJava.

<!--more--> 

## Update

I am leaving this information for legacy sake, but about a year after this was published I wrote [again]({{ site.baseurl }}{% link _posts/2018-05-31-breaking-the-buzzwords-barrier-room-rx-repository.md %}) on the topic, including some new info and a little nice Kotlin syntax.

# Project Setup

First, add the following dependencies to your app's `build.gradle` file. The Kotlin dependency is optional, but it's the language I'll be using for this tutorial.

```groovy
	compile "android.arch.persistence.room:runtime:$roomLibraryVersion"
	compile "android.arch.persistence.room:rxjava2:$roomLibraryVersion"
	compile "io.reactivex.rxjava2:rxjava:$rxJavaVersion"
	compile "io.reactivex.rxjava2:rxandroid:$rxAndroidVersion"
	annotationProcessor "android.arch.persistence.room:compiler:$roomLibraryVersion"
	kapt "android.arch.persistence.room:compiler:$roomLibraryVersion"
```

If you're not using Kotlin, you also don't need the `kapt` line at the end. That's for Kotlin annotation processing. Here are all of the version numbers used in this tutorial:

```groovy
	roomLibraryVersion = "1.0.0-alpha1"
	rxJavaVersion = "2.0.6"
	rxAndroidVersion = "2.0.1"
```

You can get more information or the latest versions [here](https://developer.android.com/topic/libraries/architecture/adding-components.html).

# Task Entity

An [Entity](https://developer.android.com/topic/libraries/architecture/room.html#entities) is a class that represents a database row. In this application, we'll have a table of `Task` objects that the user has to complete, so we can annotate our class with the `@Entity` annotation. We can also use annotations like `@PrimaryKey` to define which property should be the primary key, and even autogenerate one if necessary:

```kotlin
	@Entity
	class Task() {
	    @PrimaryKey(autoGenerate = true) var id: Int = 0
	    var description: String = ""
	    var completed: Boolean = false

	    constructor(description: String, completed: Boolean = false): this() {
	        this.description = description
	        this.completed = completed
	    }
	}
```

# Task DAO

A [DAO](https://developer.android.com/topic/libraries/architecture/room.html#daos), or Database Access Object is an interface used to abstract access to the database. This is where you put all of your CRUD (Create, Read, Update, Delete) methods. Below is the code for our DAO, but please check the documentation for additional information:

```kotlin
	@Dao
	interface TaskDAO {
	    @Query("SELECT * FROM task")
	    fun getAll(): Flowable<List<Task>>

	    @Query("SELECT * FROM task WHERE completed = :arg0")
	    fun getTasksByCompletion(complete: Boolean): Flowable<List<Task>>

	    @Insert
	    fun insertAll(vararg tasks: Task)

	    @Update
	    fun update(task: Task)

	    @Delete
	    fun delete(task: Task)
	}
```

Note: The `getTasksByCompletion()` and `delete()` methods aren't actually used in this sample, but were added just for educational purposes.

From the above methods, the Query methods can use the [Room RxJava2](https://developer.android.com/topic/libraries/architecture/room.html#daos-query-rxjava) integration to return `Flowable` objects. A `Flowable` is an RxJava component you can read about [here](https://github.com/ReactiveX/RxJava/blob/2.x/DESIGN.md#flowable).

# AppDatabase

The last thing we need to create is our [Database](https://developer.android.com/reference/android/arch/persistence/room/Database.html) class. This is an abstract class extending from [RoomDatabase](https://developer.android.com/reference/android/arch/persistence/room/RoomDatabase.html) which defines the entities used in this database, it's version, and is the primary access point for the database. 

In this example, I've also decided to use the class with the Singleton Pattern to get a database instance for the app to use:

```kotlin
	@Database(entities = arrayOf(Task::class), version = 2)
	abstract class AppDatabase : RoomDatabase() {
	    abstract fun taskDao(): TaskDAO

	    companion object {
	        private var INSTANCE: AppDatabase? = null
	            private set

	        fun getInMemoryDatabase(context: Context): AppDatabase {
	            if (INSTANCE == null) {
	                INSTANCE = Room.databaseBuilder(context,
	                        AppDatabase::class.java, "todo-list")
	                        .build()
	            }

	            return INSTANCE!!
	        }
	    }
	}
```

For every DAO you create, you'll need a corresponding abstract method for that DAO inside of this class. If you want to learn more about database migrations, which aren't covered in this post, you can read about them [here](https://developer.android.com/topic/libraries/architecture/room.html#db-migration).

# Accessing The Database - Notes

Now that we've created our AppDatabase, we can call `AppDatabase.getInMemoryDatabase(context).taskDao()...` to perform database operations. However, Room does not allow you to access the database on the main thread as it could produce Application Not Responding (ANR) errors. Your choices are to move the code to a separate thread yourself, or if you insist you can use the [allowOnMainThread()](https://developer.android.com/reference/android/arch/persistence/room/RoomDatabase.Builder.html#allowMainThreadQueries()) method in your builder.

I've also decided to make use of Kotlin's extension methods, to create an extension method on a Context to easily access our database from an activity or view context seen later in the post:

```kotlin
	fun Context.taskDao(): TaskDAO {
	    return AppDatabase.getInMemoryDatabase(this).taskDao()
	}
```

# Query

Now that we have our database setup, let's first go over our implementation of the query calls. Since our queries return RxJava Flowables, we'll use that object to subscribe on a new thread, observe on the main thread, and update the tasks in our `RecyclerView.Adapter` when it's done, like this:

```kotlin
	taskDao().getAll()
			.subscribeOn(Schedulers.newThread())
			.observeOn(AndroidSchedulers.mainThread())
			.subscribe({ adapter.tasks = it })
```

For the full Activity and Adapter code, please see [GitHub](https://github.com/androidessence/todo-room).

# Insert

To insert into the database, I've used an RxJava [Single](https://github.com/ReactiveX/RxJava/blob/2.x/DESIGN.md#single) to run this action asynchronously:

```kotlin
	Single.fromCallable { taskDao().insertAll(task) }
			.subscribeOn(Schedulers.newThread())
			.subscribe()
```

# Update

This was a little tricky. I tried using a Single just like the last example, but I couldn't quite get it to work. You can read more about the question and solution on [StackOverflow](https://stackoverflow.com/questions/44477568/calling-an-rxjava-single-in-kotlin-lambda), but here is the code inside of the ViewHolder:

```kotlin
	class TaskViewHolder(view: View?, taskAdapter: TaskAdapter) : RecyclerView.ViewHolder(view) {
	    val adapter: WeakReference<TaskAdapter> = WeakReference(taskAdapter)
	    val descriptionTextView = view?.findViewById(R.id.task_description) as? TextView
	    val completedCheckBox = view?.findViewById(R.id.task_completed) as? CheckBox

	    private lateinit var emitter: ObservableEmitter<Task>
	    private val disposable: Disposable = Observable.create(ObservableOnSubscribe<Task> { e -> emitter = e })
	            .subscribeOn(Schedulers.newThread())
	            .observeOn(Schedulers.newThread())
	            .subscribe({ itemView.context.taskDao().update(it) })

	    fun bindTask(task: Task) {
	        descriptionTextView?.text = task.description
	        completedCheckBox?.isChecked = task.completed

	        completedCheckBox?.setOnCheckedChangeListener { _, isChecked ->
	            adapter.get()?.tasks?.get(adapterPosition)?.completed = isChecked
	            emitter.onNext(adapter.get()?.tasks?.get(adapterPosition))
	        }
	    }
	}
```

That is all of the Room related code for this tutorial! You can find all of this [on GitHub](https://github.com/androidessence/todo-room), but those of you familiar with SQLite on Android can already see how much simpler this was. We didn't have to write any table contracts, define individual columns, write a pesky SQLiteOpenHelper, or any of that boilerplate work. 

