---
layout: post
author: adam
title: How To Build A Todo List In Kotlin Part 3&#58; Adding Items
description: Discusses the building of a simple todo list application in Kotlin.
modified: 2017-05-21
published: true
tags: [kotlin]
categories: [android]
---

Following parts [1]({{ site.baseurl }}{% link _posts/2017-05-21-how-to-build-a-todo-list-in-kotlin-part-1-new-project.md %}) and [2]({{ site.baseurl }}{% link _posts/2017-05-21-how-to-build-a-todo-list-in-kotlin-part-2-recyclerview.md %}) you should have a working Android app in Kotlin that displays a list of Tasks to be completed and lets you mark them as complete. This segment is going to show you how to implement support for adding new items.

<!--more-->

# Create A New Kotlin Activity

First, we need to create an Activity for adding an account. We can create a new activity by selecting our project folder, right clicking and going to `New -> Kotlin Activity -> Empty Activity` in the context menu that pops up. I'm going to name it AddTaskActivity:

![AndroidEssence](/images/kotlin/new-activity-1.png)

![AndroidESsence](/images/kotlin/new-activity-2.png)

We'll just use a simple layout to avoid over-complicating the tutorial. I used the new `ConstraintLayout` and placed an EditText with a Button underneath:

```xml
	<?xml version="1.0" encoding="utf-8"?>
	<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:app="http://schemas.android.com/apk/res-auto"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    tools:context="com.adammcneilly.todolist.AddTaskActivity">

	    <EditText
	        android:id="@+id/task_description"
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:hint="@string/description"
	        android:inputType="text"
	        app:layout_constraintTop_toTopOf="parent" />

	    <Button
	        android:id="@+id/submit"
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:text="@string/submit"
	        app:layout_constraintTop_toBottomOf="@id/task_description" />
	</android.support.constraint.ConstraintLayout>
```

The string resources are defined as:

```xml
	<string name="description">Description</string>
	<string name="submit">Submit</string>
```

# Launch New Activity

Before we implement the logic in this new activity, let's make the corresponding changes in `MainActivity.kt` to launch it. There's four things we need to do:

1. Add an `addTask()` method to our TaskAdapter class.
2. Define a companion object in MainActivity which houses variables that work similar to static fields in Java. More information [here](https://kotlinlang.org/docs/reference/object-declarations.html#companion-objects).
3. Modify the onClickListener of the FloatingActionButton in MainActivity to start our AddTaskActivity with a result code.
4. Override onActivityResult in MainActivity to get the description that was passed from the other activity (this is coded later) and create a new task with it.

Here is what our `addTask()` method looks like:

```kotlin
	fun addTask(task: Task) {
	    tasks.add(task)
	    notifyDataSetChanged()
	}
```

And here is what the MainActivity looks like with our new code added:

```kotlin
	class MainActivity : AppCompatActivity() {

	    var adapter: TaskAdapter? = TaskAdapter()

	    override fun onCreate(savedInstanceState: Bundle?) {
	        // ...

	        val fab = findViewById(R.id.fab) as? FloatingActionButton
	        fab?.setOnClickListener { _ ->
	            val intent = Intent(this, AddTaskActivity::class.java)
	            startActivityForResult(intent, ADD_TASK_REQUEST)
	        }
	    }

	    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
	        if (requestCode == ADD_TASK_REQUEST && resultCode == Activity.RESULT_OK) {
	            val task = Task(data?.getStringExtra(DESCRIPTION_TEXT).orEmpty())
	            adapter?.addTask(task)
	        }
	    }

	    // ...

	    companion object {
	    	private val ADD_TASK_REQUEST = 0
	        val DESCRIPTION_TEXT = "description"
	    }
	}
```

If you use this as a break point, you'll see the new activity starts up, but nothing happens when you hit submit. We'll implement that now.

# Implement New Activity

In our activity, we'll have the following logic:

* When the user submits, check if the EditText is empty.
* If it's empty, show an error message.
* If it's not empty, pass the description back as an intent extra.

All of this code can be implemented in our `onCreate()` method:

```kotlin
	class AddTaskActivity : AppCompatActivity() {

	    override fun onCreate(savedInstanceState: Bundle?) {
	        super.onCreate(savedInstanceState)
	        setContentView(R.layout.activity_add_task)
	        
	        val description = findViewById(R.id.task_description) as? EditText
	        val submit = findViewById(R.id.submit) as? Button
	        
	        submit?.setOnClickListener { 
	            if (description?.text?.toString().isNullOrBlank()) {
	                description?.error = "Please enter a description"
	            } else {
	                val data = Intent()
	                data.putExtra(MainActivity.DESCRIPTION_TEXT, description?.text.toString())
	                setResult(Activity.RESULT_OK, data)
	                
	                finish()
	            }
	        }
	    }
	}
```

Running the app at this point you'll see that we can start a new activity, type in a description, and hit submit to see the new task in our list. That's it for part 3! Move on to the the [fourth part]({{ site.baseurl }}{% link _posts/2017-05-21-how-to-build-a-todo-list-in-kotlin-part-4-storing-items.md %}) to see how we can read and write to a text file and persist the information. If you've missed any code, you can find it on [GitHub](http://github.com/AdamMc331/todo-kotlin).