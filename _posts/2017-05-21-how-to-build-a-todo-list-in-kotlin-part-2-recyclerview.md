---
layout: post
author: adam
title: How To Build A Todo List In Kotlin Part 2&#58; Implementing A RecyclerView
description: Discusses the building of a simple todo list application in Kotlin.
modified: 2017-05-21
published: true
tags: [kotlin, recyclerview]
categories: [android]
---

Following up on [part 2]({{ site.baseurl }}{% link _posts/2017-05-21-how-to-build-a-todo-list-in-kotlin-part-1-new-project.md %}) which demonstrates how to create your Android app and configure Kotlin, we'll begin building the heart and soul of a Todo List application - the list!

# Data Model

Let's begin by defining our model. We're going to create a simple class that has two fields, one for description and one for whether or not it's completed. Here is how this class will look in Kotlin:

```kotlin
	data class Task(var description: String, var completed: Boolean = false) : Serializable
```

You may not believe it, but that's all we need. Let's talk about what's happening here:

* A data class is a special class in Kotlin that provides you with default behaviors for all your Object methods like `toString()` `hashCode()` `equals()` and `copy()`. Read more [here](https://kotlinlang.org/docs/reference/data-classes.html).
* Kotlin allows for default constructors to be defined right with the class name.
* Kotlin allows for default parameters. So in this case, we have a constructor that can be used as `Task("Description")` and it will default to incomplete, or we can call it with `Task("Description", true)` to set the initial value of the completed boolean.
* We've had our class implement Serializable. In this simple app, we're just going to save the data to a text file instead of over complicating it with SQLite.

<!--more-->

# RecyclerView.Adapter

We can start out by first defining the XML layout for one of our list items. We'll simply use a TextView and a CheckBox to mark completion:

```xml
	<?xml version="1.0" encoding="utf-8"?>
	<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:layout_width="match_parent"
	    android:layout_height="wrap_content"
	    android:padding="8dp">

	    <TextView
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:textAppearance="?android:attr/textAppearanceMedium"
	        android:text="Medium Text"
	        android:id="@+id/task_description"
	        android:layout_alignParentTop="true"
	        android:layout_alignParentLeft="true"
	        android:layout_alignParentStart="true"
	        android:layout_toLeftOf="@+id/task_completed"
	        android:layout_toStartOf="@+id/task_completed"
	        android:textColor="@android:color/black"
	        android:layout_alignParentBottom="false" />

	    <CheckBox
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:id="@+id/task_completed"
	        android:layout_alignBottom="@+id/task_description"
	        android:layout_alignParentRight="true"
	        android:layout_alignParentEnd="true"
	        android:layout_alignParentTop="true" />
	</RelativeLayout>
```

Next, we need to define our `RecyclerView.Adapter` class. I like to start by building out the ViewHolder, so let's dissect that:

```kotlin
	class TaskAdapter(var tasks: MutableList<Task> = ArrayList()) {
	    
	    inner class TaskViewHolder(view: View?) : RecyclerView.ViewHolder(view) {
	        val descriptionTextView = view?.findViewById(R.id.task_description) as? TextView
	        val completedCheckBox = view?.findViewById(R.id.task_completed) as? CheckBox

	        fun bindTask(task: Task) {
	            descriptionTextView?.text = task.description
	            completedCheckBox?.isChecked = task.completed
	            
	            completedCheckBox?.setOnCheckedChangeListener { buttonView, isChecked -> 
	                tasks[adapterPosition].completed = isChecked
	            }
	        }
	    }
	}
```

Here we create an inner class that is a ViewHolder, it takes in a View that is passed as a parameter to the constructor of the super class as well. We defined our two UI elements, and wrote a bind method that takes in a task and displays it accordingly. We've also added an `OnCheckedChangeListener` that modifies the task at a given position. Implementing the rest of the adapter is pretty straight forward. Here's what the final results look like:

```kotlin
	class TaskAdapter(var tasks: MutableList<Task>) : RecyclerView.Adapter<TaskAdapter.TaskViewHolder>() {

	    override fun onCreateViewHolder(parent: ViewGroup?, viewType: Int): TaskViewHolder {
	        val context = parent?.context
	        val view = LayoutInflater.from(context)?.inflate(R.layout.list_item_task, parent, false)
	        return TaskViewHolder(view)
	    }

	    override fun onBindViewHolder(holder: TaskViewHolder?, position: Int) {
	        holder?.bindTask(tasks[position])
	    }

	    override fun getItemCount(): Int {
	        return tasks.size
	    }

	    inner class TaskViewHolder(view: View?) : RecyclerView.ViewHolder(view) {
	        val descriptionTextView = view?.findViewById(R.id.task_description) as TextView
	        val completedCheckBox = view?.findViewById(R.id.task_completed) as CheckBox

	        fun bindTask(task: Task) {
	            descriptionTextView.text = task.description
	            completedCheckBox.isChecked = task.completed

	            completedCheckBox.setOnCheckedChangeListener { buttonView, isChecked ->
	                tasks[adapterPosition].completed = isChecked
	            }
	        }
	    }
	}
```

Next we need to modify the `content_main.xml` file to include a RecyclerView:

```xml
	<?xml version="1.0" encoding="utf-8"?>
	<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:app="http://schemas.android.com/apk/res-auto"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    app:layout_behavior="@string/appbar_scrolling_view_behavior"
	    tools:context="com.adammcneilly.todolist.MainActivity"
	    tools:showIn="@layout/activity_main">

	    <android.support.v7.widget.RecyclerView
	        android:id="@+id/task_list"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent" />

	</android.support.constraint.ConstraintLayout>
```

Now in our `MainActivity.kt` file we can add the following in `onCreate()`:

```kotlin
	val recyclerView = findViewById(R.id.task_list) as RecyclerView
	val layoutManager = LinearLayoutManager(this)
	val adapter = TaskAdapter(getSampleTasks())
	recyclerView.layoutManager = layoutManager
	recyclerView.adapter = adapter
```

The `getSampleTasks()` method is a private method I've added just for testing:

```kotlin
	private fun getSampleTasks(): MutableList<Task> {
	    val task1 = Task("task1")
	    val task2 = Task("task2", true)

	    return mutableListOf(task1, task2)
	}
```

Note: In this context it makes sense to have the adapter defined right before the RecyclerView, but later in the tutorial you'll want to have it defined at the class level of your activity.

At this point, let's run our app and verify that our list appears. Following all of the above steps, this is what you can expect to see:

![AndroidEssence](/images/kotlin/todo-1.png)

Now that you have a working RecyclerView, you can move on to [part 3]({{ site.baseurl }}{% link _posts/2017-05-21-how-to-build-a-todo-list-in-kotlin-part-3-adding-items.md %}). If you've missed any code, you can find it on [GitHub](http://github.com/AdamMc331/todo-kotlin).