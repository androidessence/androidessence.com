---
layout: post
author: adam
title: Breaking the Buzzwords Barrier Part 2&#58; Rx, Room, and Repository
description: Demonstrates how to use RxJava with Room and the Repository pattern.
modified: 2018-05-31
published: true
tags: [architecture, rxjava, room, databases]
categories: [android, tutorial]
---

In [part 1]({{ site.baseurl }}{% link _posts/2018-05-30-breaking-the-buzzwords-barrier-mvvm.md %}) we discussed how we were going to architect the various components of our application. Now it's time to build them. To understand what we should build first, we should revisit the diagram:

![MVVM](/images/buzzwords/cashcaretaker_mvvm.png)

I would start with three spots:

1. The account
2. The database
3. The repository

A good rule of thumb to remember this, is that these nodes don't depend on anything else just yet (well, the repository depends on the database, but that was included). I can't build my ViewModel until I have my repository, and so on. 

Let's start with persistence.

<!--more-->

# Room Persistence Library

I've already written once about the [room persistence library]({{ site.baseurl }}{% link _posts/2017-06-10-getting-started-with-room-persistence-library.md %}) which was announced at Google I/O 2017. Feel free to skim that over for some details, but I'm going to revisit a lot of it here, as I've learned much more about Room/Kotlin over the last year, and I think this will be more relevant.

## Purpose

I mentioned in the [intro post]({{ site.baseurl }}{% link _posts/2018-05-29-breaking-the-buzzwords-barrier.md %}) that I would go over the necessity of each buzzword. Room was created as [an abstraction over SQLite](https://developer.android.com/topic/libraries/architecture/room) to make working with a SQLite database on Android even easier for developers. The ability has always existed - but previously we had to write all of the table schema ourselves, handle upgrades (now simpler with [migrations](https://developer.android.com/training/data-storage/room/migrating-db-versions)), and write our own queries and read from cursor objects, and much more I've probably forgotten about. 

As a result of all of this verbose, boilerplate, and often repetitive code, the developers over at Google sought out to make this easier for us, and they've nailed it. I won't be comparing Room to previous options, but trust me that what you're about to see is smoother than it was.

## Entity

The first thing we need to get started with Room is an [Entity](https://developer.android.com/training/data-storage/room/defining-data). To create an entity in room, you just have to annotate the class. What this will do is create a table for your entity, and each property you have (that you don't ignore) will be created as a column. 

So if I wanted a table called `Account`, with a column for `name` and `balance`, I would write an entity like this:

```kotlin
	@Entity(indices = [(Index("name"))])
	data class Account(
	        @PrimaryKey(autoGenerate = false) var name: String = "",
	        var balance: Double = 0.0
	)
```

If you chose to [export schemas](https://developer.android.com/reference/android/arch/persistence/room/Database#exportschema) in room, you can see a detailed overview of what this class will actually do with regards to the database:

```json
	{
		"tableName": "Account",
		"createSql": "CREATE TABLE IF NOT EXISTS `${TABLE_NAME}` (`name` TEXT NOT NULL, `balance` REAL NOT NULL, PRIMARY KEY(`name`))",
		"fields": [
		  {
		    "fieldPath": "name",
		    "columnName": "name",
		    "affinity": "TEXT",
		    "notNull": true
		  },
		  {
		    "fieldPath": "balance",
		    "columnName": "balance",
		    "affinity": "REAL",
		    "notNull": true
		  }
		],
		"primaryKey": {
		  "columnNames": [
		    "name"
		  ],
		  "autoGenerate": false
		},
		"indices": [
		  {
		    "name": "index_Account_name",
		    "unique": false,
		    "columnNames": [
		      "name"
		    ],
		    "createSql": "CREATE  INDEX `index_Account_name` ON `${TABLE_NAME}` (`name`)"
		  }
		],
		"foreignKeys": []
	}
```

An important thing to note, is that the class name is what is automatically used as the table name, unless annotated otherwise. In Cash Caretaker, we naturally want a class called a `Transaction`, representing a financial one, but we can't have a `Transaction` table as that's a reserved SQLite keyword. So in some cases you want to add the table name to your annotation (this example also shows how you can have foreign keys as well):

```kotlin
	@Parcelize
	@Entity(tableName = "transactionTable", foreignKeys = [(ForeignKey(entity = Account::class, parentColumns = arrayOf("name"), childColumns = arrayOf("accountName"), onDelete = ForeignKey.CASCADE))])
	data class Transaction(
	        var accountName: String = "",
	        var description: String = "",
	        var amount: Double = 0.0,
	        var withdrawal: Boolean = true,
	        var date: Date = Date(),
	        @PrimaryKey(autoGenerate = true) var id: Long = 0
	) : Parcelable
```

## Database Access Object

Once we've created our entity, we need to create a Database Access Object (DAO). Our DAO is what we use to write our insert/update/query functionality.

Fortunately, we have annotations for Insert/Update/Delete already, but if you want to write a special query (even if it's for deleting, not necessarily a select query), you can do that as well. Below is an example of some options, but you can find more info [in the official documentation](https://developer.android.com/training/data-storage/room/accessing-data):

```kotlin
	@Dao
	interface AccountDAO {
	    @Query("SELECT * FROM account ORDER BY name")
	    fun getAll(): Flowable<List<Account>>

	    @Insert
	    fun insert(account: Account): Long

	    @Delete
	    fun delete(account: Account): Int

	    @Query("DELETE FROM account")
	    fun deleteAll(): Int
	}
```

Other than the two special cases, we didn't have to write out long insert/delete queries, or even worry about reading back from the cursor. We just annotate our interface with `@Dao`, and each method with the appropriate database action. If you're interested in studying the generate code for this DAO, I've put an example [into a GitHub gist](https://gist.github.com/AdamMc331/150908d1be78fec8d638542502e61839).

## Database

Once you've created your entity and DAO, the only thing left is the actual database. This is only a few short steps as well:

1. Define your database as an abstract class extending from `RoomDatabase`.
2. Annotate your DAOs, version number, and any [converters](https://developer.android.com/reference/android/arch/persistence/room/TypeConverter) you may have. These are used to convert special types like longs to datetimes. 
3. Create an abstract function for each of your DAOs.

This will leave you with something like this:

```kotlin
	@Database(entities = [(Account::class), (Transaction::class)], version = 1)
	@TypeConverters(Converters::class)
	abstract class CCDatabase : RoomDatabase() {
	    abstract fun accountDao(): AccountDAO
	    abstract fun transactionDao(): TransactionDAO

	    companion object {
	        private var INSTANCE: CCDatabase? = null

	        fun getInMemoryDatabase(context: Context): CCDatabase {
	            if (INSTANCE == null) {
	                INSTANCE = Room.databaseBuilder(context,
	                        CCDatabase::class.java, "cashcaretaker.db")
	                        .build()
	            }

	            return INSTANCE!!
	        }
	    }
	}
```

### Callback

An optional feature you may want to add is a `RoomDatabase.Callback`. This [class](https://developer.android.com/reference/android/arch/persistence/room/RoomDatabase.Callback) allows you to get a callback each time the database is updated or created. You may use this to define triggers for your database such as updating account balance whenever a transaction is added. That will look a little something like this:

```kotlin
	fun getInMemoryDatabase(context: Context): CCDatabase {
	    if (INSTANCE == null) {
	        INSTANCE = Room.databaseBuilder(context,
	                CCDatabase::class.java, "cashcaretaker.db")
	                .addCallback(CALLBACK)
	                .build()
	    }

	    return INSTANCE!!
	}

	val CALLBACK = object : RoomDatabase.Callback() {
	    override fun onCreate(db: SupportSQLiteDatabase) {
	        super.onCreate(db)

	        db.execSQL(
	                "CREATE TRIGGER update_balance_for_withdrawal " +
	                        "AFTER INSERT ON transactionTable " +
	                        "WHEN new.withdrawal " +
	                        "BEGIN " +
	                        "UPDATE account " +
	                        "SET balance = balance - new.amount " +
	                        "WHERE name = new.accountName; END;")

	        ...
	}
```

# RxJava

There are countless blog posts and tutorial on the internet about RxJava. Rightfully so, as this is a really complex topic and one with a high learning curve. I really recommend [RxJava With Kotlin In Baby Steps](https://www.youtube.com/watch?v=YPf6AYDaYf8) by Annyce Davis for getting started. 

## Purpose

[RxJava](https://github.com/ReactiveX/RxJava) is an implementation of [Reactive Extensions](http://reactivex.io/), a pattern of APIs used for asynchronous programming in a number of languages. It is used by many Android developers to be able to run operations asynchronously, without managing the thread handling ourselves. Use cases for asynchronous programming include long running operations like downloading files, calling a network, or reading and writing to a database. Streams can also be used to notify different components when something happens, which you'll see later on in the ViewModel section. As I said, though, we also want to do our database operations asynchronously (in fact, Room requires this unless you specify otherwise), so it is important to understand RxJava here.

I won't be going into a deep dive of Rx, but instead linked to one helpful resource as well as the official docs. However, there are a few classes that will appear throughout the Cash Caretaker codebase that are worth highlighting.

## Flowable

You may have noticed above that our select query returned a `Flowable<List<Account>>`. A [Flowable](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html) is a stream that you can subscribe to, and observe each item that is emitted by that stream. 

In this database example, our query will give us a Flowable that exists a `List<Account>` when the query returns, but them subsequently each time the data changes. This is great! We only have to subscribe once, and then be notified each time the data changes, until the subscription is killed. 

## Single

A single is similar to a Flowable, in that it is something that you can observe, but it will only emit a single item (get it?) and then complete. This is used a couple times in Cash Caretaker to run an asynchronous operation. You can use [Single.fromCallable](http://reactivex.io/RxJava/javadoc/io/reactivex/Single.html#fromCallable-java.util.concurrent.Callable-) to run a function that you pass in, and the result of the function will be emitted. We'll see this example in our repository too. 

## Subject

A subject is an interesting class in RxJava because it is both an Observer (can subscribe to something and react to change), but also an Observable (meaning it can be subscribed to, and admit new items). We use Subjects in Cash Caretaker later on to handle being notified when a view is clicked, but also when accounts are interacted with. You'll see this in the Fragment/ViewModel section, but was worth discussing with other RxJava classes.

To learn more about the various subjects, I love [this post](https://blog.mindorks.com/understanding-rxjava-subject-publish-replay-behavior-and-async-subject-224d663d452f) by Amit Shekhar.

# Repository Pattern

I feel this is another common buzzword that appears often in Android development discussions. I learned more about this pattern from [an article by Hannes Dorfmann](http://hannesdorfmann.com/android/evolution-of-the-repository-pattern) which explains the history and evolution of the pattern, and I found it very helpful.

## Purpose

The TL;DR purpose of the repository pattern is abstracting *what* information is required from the *how* it is collected. Let's consider the Cash Caretaker example. When we build a repository, it should run the same 4 account operations that are in our DAO:

1. Select all accounts
2. Insert an account
3. Delete a specific account
4. Delete all accounts

Now, a reasonable question is "why not just have our ViewModel that we eventually write interract with our DAO directly?" We could! But the downfall is, what if I ever want to change the application to read/write data from a server instead of a local database? Or perhaps both? To make that change - it would require modifying the way the ViewModel works.

If we use a repository as a middle man, we don't have to update our ViewModel when we change our underlying data source. We only have to change the repository itself. This is the same separation of concerns discussion we had in part 1. Our ViewModel only cares about what information it needs access to, it does not care how that information is collected. Our repository serves as that interface.

For those of you coming from an MVP background, it works like an [interactor](https://stackoverflow.com/a/47452682/3131147).

## Implementation

For the purpose of this app, the repository will behave in a number of cases like a proxy straight through to the database - but it can also be used for some intermediate mapping where necessary! Let's look at our account examples:

```kotlin
	open class CCRepository(private val database: CCDatabase) {
	    private val accountDAO: AccountDAO = database.accountDao()

	    fun getAllAccounts(): Flowable<DataViewState> = accountDAO.getAll()
	            .map {
	                if (it.isEmpty()) {
	                    DataViewState.Empty()
	                } else {
	                    DataViewState.Success(it)
	                }
	            }

	    fun deleteAccount(account: Account): Int = accountDAO.delete(account)

	    fun insertAccount(account: Account): Long = accountDAO.insert(account)

	    ...
	}
```

In the case of deleting and inserting an account, we just return whatever the database wants. However, when I query for accounts, I don't want to return the list of accounts, but rather a state that my ViewModel can be in.

We'll learn more about this in the next post, but there's room for discussion on whether or not this mapping logic should be inside my repository or the ViewModel itself. I think it is okay here in the repository, but some could argue that your repository should only care about fetching data, and nothing to do with a state. 

If you're unfamiliar with the RxJava map operator, you can learn more [here](http://reactivex.io/documentation/operators/map.html). 

# Conclusion

I hope this helped you understand a few 'R' buzzwords - Room/RxJava/Repositories. Still confused? Unsure what their purposes are? Let me know in the comments and I'll be sure to clarify!

Now you can head over to [part 3]({{ site.baseurl }}{% link _posts/2018-06-01-breaking-the-buzzwords-barrier-viewmodel.md %}), which will go over the ViewModel, and the differences between an Android ViewModel and the ViewModel we talk about in MVVM. 