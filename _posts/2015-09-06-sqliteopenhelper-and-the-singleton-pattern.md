---
layout: post
title: SQLiteOpenHelper and the Singleton Pattern
description: Discussing the Singleton Pattern and it's benefits when used with SQLiteOpenHelper.
modified: 2015-09-06
tags: [android, sqlite, singleton pattern]
categories: [android]
---

The [singleton design pattern](https://en.wikipedia.org/wiki/Singleton_pattern) is a design that limits the instantiation of a class to a single object. To quote Wikipedia:

> This is useful when exactly one object is needed to coordinate actions across the system.

A great example of this is the SQLiteOpenHelper, or any other data source object. Using more than one datasource to talk to an SQLite database on Android creates the opportunity for leaks in your SQLite connection, which we will discuss later. First, let’s talk about the proper way to use the SQLiteOpenHelper and the singleton pattern. This tutorial assumes you have already created the open helper class. If you do not have any experience with SQLite databases for Android, I recommend [Lars Vogel’s tutorial](http://www.vogella.com/tutorials/AndroidSQLite/article.html).

<!--more-->

# With a ContentProvider

If you are using a ContentProvider in your application, you can store your SQLiteOpenHelper as a class level variable:

```java
	public class MyProvider extends ContentProvider{ 
	   private MyHelper mOpenHelper;  
	 
	   @Override 
	   public boolean onCreate(){
	      mOpenHelper = new MyHelper(getContext()); 
	      return true; 
	   }
	}
```

By using the helper as a class level variable, we are assured that there is only one connection anytime the ContentProvider is used. In other words, inside the required methods such as query(), delete(), update(), you should not be creating a new connection. Instead, you just get the database using the open helper:

```java
	@Override
	public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs,
	String sortOrder) {
	   final SQLiteDatabase db = mOpenHelper.getReadableDatabase();
	 
	   //...
	}
```

Fortunately, with the ContentProvider, we don’t have to worry about closing the connection when we’re done. According to [Dianne Hackborn](https://groups.google.com/forum/#!msg/android-developers/NwDRpHUXt0U/jIam4Q8-cqQJ), an Android Framework Engineer:

> A content provider is created when its hosting process is created, and remains around for as long as the process does, so there is no need to close the database — it will get closed as part of the kernel cleaning up the process’s resources when the process is killed.

# Without a ContentProvider

If you are not using a ContentProvider, but writing your own data source class, you will likely need to handle the closing of the database yourself. Here is an example of a simple datasource class:

```java
	public class MyDataSource{ 
	   private MyOpenHelper mOpenHelper; 
	 
	   public MyDataSource(Context context){ 
	      mOpenHelper = new MyOpenHelper(context); 
	   }
	 
	    public void close() { 
	      mOpenHelper.close(); 
	   }
	   
	   public Cursor query(...){
	      final SQLiteDatabase db = mOpenHelper.getReadableDatabase();
	      //...
	    }
	}
```

Now that you’ve extended the OpenHelper to a class level variable, you can sleep happy knowing that there is only one connection used by the datasource, helping prevent any SQLite leaks that may occur. However, you can only sleep happy once you’ve verified that you’ve closed the database inside the activity that creates the datasource. I prefer to close the connection inside `onPause()`, as that is one of the earliest methods which can be called before your app closes.

# SQLite Leaks

Let’s say you don’t do this, and instead you create a new connection every time you query the database. Something like this:

```java
	public Cursor query(Context context, ...){ 
	   return new MyOpenHelper(context).query(...);
	}
```

You may find yourself with an error down the line that ‘an SQLite object for database was leaked’. A memory leak is when a program fails to properly release discarded memory, which can impact performance or even cause applications to crash. There are several questions on StackOverflow about SQLite leaks, including [this one](http://stackoverflow.com/questions/12801602/android-sqlite-leaked/12801889#12801889) which inspired this post. As Graham Borland’s answer points out, you should always make sure you are closing the Cursor after you make a query, but Ian Warwick was quick to point out that beyond closing the query, you need to make sure you are closing the connection as well. By using the singleton pattern for you SQLiteOpenHelper, you only have one connection to be responsible for, instead of one connection for every query you preform.

As long as you follow these simple steps, and properly maintain your singleton SQLiteOpenHelper, you should never have to worry about SQLite memory leaks in your Android apps.