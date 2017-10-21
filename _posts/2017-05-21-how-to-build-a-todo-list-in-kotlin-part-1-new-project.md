---
layout: post
author: adam
title: How To Build A Todo List In Kotlin Part 1&#58; Creating A New Project
description: Discusses the building of a simple todo list application in Kotlin.
modified: 2017-05-21
published: true
tags: [kotlin]
categories: [android]
---

This blog post is going to discuss creating a project from scratch in Kotlin. We will build a sample Todo List application that will not be very complicated, but covers enough to show the benefits of Kotlin. Part 1 will discuss creating a new project and configuring Kotlin. If you're familiar with that, copy the `MainActivity.kt` code and skip to part 2 to begin building the app.

This tutorial assumes a basic knowledge of programming, and some familiarity with Android. If you have any questions please leave them in the comments, and I will udpate these posts with a deeper explanation.

<!--more-->

I'll be writing this using Android Studio 2.3.2 which does not come with Kotlin out of the box, so we'll discuss how to get the Kotlin plugin. In AS 3.0, you'll have to option to include Kotlin support right in the new project wizard.

First, just create a new project using the wizard in Android Studio. I am building mine using a "Basic Activity" from the picker so that I can have a FloatingActionButton. Check out the below video:

![AndroidEssence](/images/kotlin/NewProjectWizard.gif)

Once we have that, we also need to make sure to have the Kotlin plugin installed. You can do this by going to `Android Studio -> Preferences -> Plugins` and searching for Kotlin. Once you install and restart AndroidStudio, you can easily configure Kotlin by going to `Tools -> Kotlin -> Configure Kotlin In Project` in the menu:

![AndroidEssence](/images/kotlin/configure_kotlin.png)

When it asks how to configure, simply select `Android with Gradle`, chose whether you want your entire project or a single module, the language version, and click OK. Android Studio will make the necessary changes in the appropriate `build.gradle` files, and all you have to do is resync the project and you're good to go.

Now, you have support for Kotlin in your project, but if you look inside the source files everything is still Java. Configuring Kotlin in your project does not convert it for you, but that is still easy to do. On a Mac you can do `CMD + ALT + SHIFT + K` to convert, or select `Code -> Convert Java File To Kotlin File` in the menu:

![AndroidEssence](/images/kotlin/convert_to_kotlin.png)

And that is all you need to get off the ground in Kotlin. You should run the HelloWorld application we've built to make sure that there weren't mistakes along the way. Let's wrap up part 1 by dissecting our new `MainActivity.kt` file, and how it's different from Java:

```kotlin
	package com.adammcneilly.todolist

	import android.os.Bundle
	import android.support.design.widget.FloatingActionButton
	import android.support.design.widget.Snackbar
	import android.support.v7.app.AppCompatActivity
	import android.support.v7.widget.Toolbar
	import android.view.View
	import android.view.Menu
	import android.view.MenuItem

	class MainActivity : AppCompatActivity() {

	    override fun onCreate(savedInstanceState: Bundle?) {
	        super.onCreate(savedInstanceState)
	        setContentView(R.layout.activity_main)
	        val toolbar = findViewById(R.id.toolbar) as Toolbar
	        setSupportActionBar(toolbar)

	        val fab = findViewById(R.id.fab) as FloatingActionButton
	        fab.setOnClickListener { view ->
	            Snackbar.make(view, "Replace with your own action", Snackbar.LENGTH_LONG)
	                    .setAction("Action", null).show()
	        }
	    }

	    override fun onCreateOptionsMenu(menu: Menu): Boolean {
	        // Inflate the menu; this adds items to the action bar if it is present.
	        menuInflater.inflate(R.menu.menu_main, menu)
	        return true
	    }

	    override fun onOptionsItemSelected(item: MenuItem): Boolean {
	        // Handle action bar item clicks here. The action bar will
	        // automatically handle clicks on the Home/Up button, so long
	        // as you specify a parent activity in AndroidManifest.xml.
	        val id = item.itemId


	        if (id == R.id.action_settings) {
	            return true
	        }

	        return super.onOptionsItemSelected(item)
	    }
	}
```

There's a lot here, and I don't want this to be an ultra-detailed tutorial of the Kotlin language, but let's highlight a few things:

1. No `extends` keyword. Kotlin uses `:` to represent extending a class or implementing an interface.
2. Method declaration changes - instead of `accessModifier returnType methodName(params) { }` like Java, Kotlin syntax is `accessModifier fun methodName(params): returnType`.
3. Method param changes - instead of `(Type variableName)` like Java has, Kotlin syntax is `(variableName: Type)`
4. Optionals! Kotlin has nullability in its type system, meaning you can specify a variable as nullable or not. If you want a type to be nullable, you had a `?` after the type. Notice this change in `savedInstanceState: Bundle?`. More on this in [the official Kotlin docs](https://kotlinlang.org/docs/reference/null-safety.html).
5. Lambdas - Similar to the new Java 8, Kotlin has support for lambdas which can be used in place of an anonymous class, like the click listener above. Note: This still generates an anonymous class in your byte code, but simplifies the work for the developer.
6. No semi-colons!

Now as great as this is, there's a few ways we can change the above code. Specifically:

1. While Kotlin has nullability in the type system and shouldn't throw NPEs, it's not impossible. The `as` keyword forces a cast, and if one of our views fails to be cast correctly we could be in trouble. We could fix that by using `as?` instead, but we'll keep it because if our views can't be cast properly, I would rather fail than hide the error.
2. Once we do that, the inferred type for `fab` and `toolbar` will be `FloatingActionButton?` and `Toolbar?`, respectfully. Since they're optional, we'll have to add a safe operator where we set the click listener. See the Kotlin docs linked above for more.
3. Using `when` inside `onOptionsItemSelected`. The `when` keyword in Kotlin replaces the `switch` statement in Java.

Once we make those changes and optimize imports, here is what our `MainActivity.kt` file should look like:

```kotlin
	package com.adammcneilly.todolist

	import android.os.Bundle
	import android.support.design.widget.FloatingActionButton
	import android.support.design.widget.Snackbar
	import android.support.v7.app.AppCompatActivity
	import android.support.v7.widget.Toolbar
	import android.view.Menu
	import android.view.MenuItem

	class MainActivity : AppCompatActivity() {

	    override fun onCreate(savedInstanceState: Bundle?) {
	        super.onCreate(savedInstanceState)
	        setContentView(R.layout.activity_main)
	        
	        val toolbar = findViewById(R.id.toolbar) as Toolbar
	        setSupportActionBar(toolbar)

	        val fab = findViewById(R.id.fab) as FloatingActionButton
	        fab.setOnClickListener { view ->
	            Snackbar.make(view, "Replace with your own action", Snackbar.LENGTH_LONG)
	                    .setAction("Action", null).show()
	        }
	    }

	    override fun onCreateOptionsMenu(menu: Menu): Boolean {
	        // Inflate the menu; this adds items to the action bar if it is present.
	        menuInflater.inflate(R.menu.menu_main, menu)
	        return true
	    }

	    override fun onOptionsItemSelected(item: MenuItem): Boolean {
	        // Handle action bar item clicks here. The action bar will
	        // automatically handle clicks on the Home/Up button, so long
	        // as you specify a parent activity in AndroidManifest.xml.
	        when (item.itemId) {
	            R.id.action_settings -> return true
	            else -> return super.onOptionsItemSelected(item)
	        }
	    }
	}
```

A note on the `as` keyword:
* `as` is referred to as the unsafe cast operator in Kotlin. If the cast fails, it can throw an exception. However, Kotlin provides an `as?` operator, which is a safe operator. If the cast fails, it simply returns null. You can read more on them [here](https://kotlinlang.org/docs/reference/typecasts.html#unsafe-cast-operator).

You may have also noticed that converting to Kotlin was a whole 10 lines shorter in this example. Kotlin is much more concise than Java, thus making your code easier to maintain.

Now that you've been introduced to including Kotlin in your project, let's move on in [part 2]({{ site.baseurl }}{% link _posts/2017-05-21-how-to-build-a-todo-list-in-kotlin-part-2-recyclerview.md %}) to begin building our TodoList. If you've missed any code, you can find it on [GitHub](http://github.com/AdamMc331/todo-kotlin).