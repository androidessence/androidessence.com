---
layout: post
author: adam
title: How To Build A Todo List In Kotlin Part 4&#58; Storing Items
description: Discusses the building of a simple todo list application in Kotlin.
modified: 2017-05-21
published: true
tags: [kotlin]
categories: [android]
---

If you've been following along with parts 1-3, you now have an (almost) working todo list application. The only thing we are missing is persisting the data. Running your Android app now and rotating your screen will show you that the items you add won't persist, and disappear anytime an activity is killed and recreated. This post will show you how to write the list to a text file and read from it.

<!--more-->

# Storage Methods

The first thing we need to do is implement a way to read and write tasks. We've already marked our tasks as serializable in part 2, so we can pass them into an ObjectOutputStream without a problem. I won't go too in depth into how the streams work, but here is the code required to read and write from the text file:

```
	object Storage {
	    private val LOG_TAG = Storage::class.java.simpleName
	    private val FILE_NAME = "todo_list.ser"

	    fun writeData(context: Context, tasks: List<Task>?) {
	        var fos: FileOutputStream? = null
	        var oos: ObjectOutputStream? = null

	        try {
	            // Open file and write list
	            fos = context.openFileOutput(FILE_NAME, Context.MODE_PRIVATE)
	            oos = ObjectOutputStream(fos)
	            oos.writeObject(tasks)
	        } catch (e: Exception) {
	            Log.e(LOG_TAG, "Could not write to file.")
	            e.printStackTrace()
	        } finally {
	            try {
	                oos?.close()
	                fos?.close()
	            } catch (e: Exception) {
	                Log.e(LOG_TAG, "Could not close the file.")
	                e.printStackTrace()
	            }

	        }
	    }

	    fun readData(context: Context): MutableList<Task>? {
	        var fis: FileInputStream? = null
	        var ois: ObjectInputStream? = null

	        var tasks: MutableList<Task>? = ArrayList()

	        try {
	            // Open file and read list
	            fis = context.openFileInput(FILE_NAME)
	            ois = ObjectInputStream(fis)

	            tasks = ois?.readObject() as? MutableList<Task>
	        } catch (e: Exception) {
	            Log.e(LOG_TAG, "Could not read from file.")
	            e.printStackTrace()
	        } finally {
	            try {
	                ois?.close()
	                fis?.close()
	            } catch (e: Exception) {
	                Log.e(LOG_TAG, "Could not close the file.")
	                e.printStackTrace()
	            }

	        }

	        return tasks
	    }
	}
```

Note that we use `object Storage` instead of `class Storage` at the top. This is because all of the methods are used without the need for any instantiation. You can read more about object declarations [here](https://kotlinlang.org/docs/reference/object-declarations.html).

The last thing we have to do is update our `MainActivity.kt` file to read from storage when it resumes, and writes to storage when it pauses. When the activity resumes, though, we only want to swap items when the list is empty. If the list isn't empty, this means the activity could be coming back from `AddTaskActivity` and if we just read from a file again, we'll overwrite that new item. Here is what this code looks like:

```kotlin
	override fun onResume() {
	    super.onResume()

	    val tasks = Storage.readData(this)

	    // We only want to set the tasks if the list is already empty.
	    if (tasks != null && (adapter?.tasks?.isEmpty() ?: true)) {
	        adapter?.tasks = tasks
	    }
	}

	override fun onPause() {
	    super.onPause()

	    Storage.writeData(this, adapter?.tasks)
	}
```

Note: When running this in Android Studio, you may notice that "tasks" is highlighted in onResume, stating "Smart cast to kotlin.collections.MutableList<com.package.Task>". The reason for this is because we never explicitly defined the type of "tasks", but it was inferred based on the return type of `readData()`. 

The only piece of syntax that may look weird to you is the elvis operator (?:). This operator tries to return the first argument, and if it's null it will return the value that comes after. So in this example, if adapter or its tasks are null, it will return true. Otherwise, it returns the value of the `.isEmpty()` call.

Run your app again, and you're all set! Congratulations, you've written your first todo list application from scratch in Kotlin. If you missed any code or want to look at the code for reference, you can find this project on [GitHub](https://github.com/AdamMc331/ToDo-Kotlin).