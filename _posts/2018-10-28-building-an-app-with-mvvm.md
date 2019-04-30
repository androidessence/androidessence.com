---
layout: post
author: adam
title: Building An Application With MVVM
description: Demonstrates how to build an Android application with MVVM architecture.
modified: 2018-10-28
published: true
tags: [architecture, mvvm]
categories: [android]
---

This is the second post in what will be an ongoing series to demonstrate a few different architecture patterns that are used for Android development. You can find the code for each of them, often appearing before the blog posts, by following [this repo](https://github.com/AdamMc331/todo-monorepo). Give it a star!

In our [previous post]({{ site.baseurl }}{% link _posts/2018-10-20-building-an-app-with-mvp.md %}) we discussed the MVP architecture for building an app. This time, we're going to check out MVVM. 

<!--more-->

# MVVM

MVVM stands for Model-View-ViewModel. The name, just like any other architecture, comes from the components involved. I'll break them down just like the last post (in fact, Model and View have been copy and pasted):

## Model

Model, in this context, doesn't necessarily refer to your model classes that you write. I prefer to think of it as the data source for your application. This could be a database, a remote server, or even just dummy data that you supply. Often, you will see developers put the code for this in some kind of `Repository` class. That is what we will refer to as our model.

## View

The view is the component responsible for any UI work. This includes displaying data, and handling click events. It does _not_ include the business logic for those click events, which I will clarify in the next paragraph. In many cases, you will have an activity or a fragment that represents the view. 

## ViewModel

A ViewModel is a tricky buzzword. This is because it could refer to two things. There's the ViewModel in this context, specifically referring to the architecture pattern, and there's the ViewModel that refers to the [Android Architecture Component](https://developer.android.com/topic/libraries/architecture/viewmodel). They do go hand in hand, though, and I hope I can explain why. 

When we speak about the MVVM architecture, the ViewModel is the component responsible for maintaining state, interacting with the model, and any relevant business logic. 

The Android class is directly related to that. It's a class that maintains state throughout orientation, which is historically a pain in the ass on Android. So, if you're using MVVM architecture, it helps to have a class that can do that.

# Communication Flow

Now that we have the components defined, let's dive into the MVVM communication flow. We'll go a step further and compare it to MVP, our last example.

![](/images/mvvm.png)

This image came from [Wikipedia](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel).

If you followed the last post, you may feel like the communication pattern is very similar to MVP. Notably, the model and the view never talk to each other, in either case. That's the biggest benefit. *The difference between MVVM and MVP lies in the communication flow. MVP has two-way communication, but MVVM uses unidirectional data flow.*

I recommend this nice article by [David Street](https://www.exclamationlabs.com/blog/the-case-for-unidirectional-data-flow/) to understand the benefits of unidirectional data flow. The TL;DR being that it is predictable, and has a lack of side effects, when communication only flows in one direction.

Now, let's dive into building our todo-list app with this architecture.

# Model

Our model between the two projects doesn't actually change here. In the last example it had to extend from our contract class, but here, we can just have a standalone repository to use with dummy data.

```kotlin
    open class TaskRepository {
        open fun getItems(): List<BaseTask> {
            return listOf(
                    BaseTask("Sample task 1"),
                    BaseTask("Sample task 2"),
                    BaseTask("Sample task 3"),
                    BaseTask("Sample task 4"),
                    BaseTask("Sample task 5"),
                    BaseTask("Sample task 6"),
                    BaseTask("Sample task 7"),
                    BaseTask("Sample task 8"),
                    BaseTask("Sample task 9"),
                    BaseTask("Sample task 10")
            )
        }
    }
```

# ViewModel

Before we show our code for the ViewModel, I want to explain the responsibilities it needs to have. Most importantly: unlike the presenter in MVP, the ViewModel should have absolutely no reference to the view. So, how will the view know when to do something? The ViewModel will expose that information through [LiveData](https://developer.android.com/topic/libraries/architecture/livedata), that the view can subscribe to. With that said, we need the following:

1. Our ViewModel will have a reference to our `TaskRepository` to fetch tasks.
2. Those tasks will be exposed via LiveData.
3. When we return from adding a task, the ViewModel should retrieve the task and expose the new one via livedata so the view can update the adapter.
4. When the add button is clicked, the ViewModel should expose via LiveData some way for the View to know that it must navigate to the add task screen.

Given all of that, we end up with the following ViewModel code:

```kotlin
    class TaskListViewModel(private val repository: TaskRepository) : ViewModel() {
        val tasks = MutableLiveData<List<BaseTask>>()
        val newTask = MutableLiveData<BaseTask>()
        val navigationAction = MutableLiveData<NavigationAction>()

        fun getTasks() {
            if (tasks.value == null) {
                tasks.value = repository.getItems()
            }
        }

        fun returnedFromAddTask(data: Intent?) {
            val description = data?.getStringExtra(AddTaskActivity.DESCRIPTION_KEY).orEmpty()
            val taskFromIntent = BaseTask(description)
            newTask.value = taskFromIntent
        }

        fun addButtonClicked() {
            navigationAction.value = NavigationAction.ADD_TASK
        }
    }
```

Again, the key thing to note here is that the ViewModel has no ties whatsoever to the view. The nice thing about this now is that multiple views could reference an instance of this viewmodel, if we needed to.

Side note: `NavigationAction` is just an enum I made so that our ViewModel doesn't have to handle each possible navigation route, but just emit a single action that the view can listen for and handle accordingly. We'll see that next.

# View

Similar to the last example, our view just refers to the activity or fragment. Before we look at the code, I'll highlight a couple things:

1. ViewModels that have a constructor must be created with a [ViewModelProvider.Factory](https://developer.android.com/reference/android/arch/lifecycle/ViewModelProvider.Factory).
2. Once the ViewModel is created, the view will observe the three LiveData we created and react accordingly.

```kotlin
    class TaskListActivity : BaseTaskListActivity() {
        private val adapter = BaseTaskAdapter()
        private lateinit var viewModel: TaskListViewModel

        private val viewModelFactory = object : ViewModelProvider.Factory {
            override fun <T : ViewModel?> create(modelClass: Class<T>): T {
                val repository = TaskRepository()
                val viewModel = TaskListViewModel(repository)

                @Suppress("UNCHECKED_CAST")
                return viewModel as T
            }
        }

        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)

            setupViewModel()
            initializeRecyclerView()
            initializeFAB()

            viewModel.getTasks()
        }

        override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
            super.onActivityResult(requestCode, resultCode, data)

            if (requestCode == ADD_TASK_REQUEST && resultCode == Activity.RESULT_OK) {
                viewModel.returnedFromAddTask(data)
            }
        }

        private fun setupViewModel() {
            viewModel = ViewModelProviders.of(this, viewModelFactory).get(TaskListViewModel::class.java)

            viewModel.tasks.observe(this, Observer {
                it?.let(adapter::tasks::set)
            })

            viewModel.newTask.observe(this, Observer {
                it?.let { task ->
                    adapter.tasks += task
                }
            })

            viewModel.navigationAction.observe(this, Observer {
                when (it) {
                    NavigationAction.ADD_TASK -> navigateToAddTask()
                }
            })
        }

        private fun initializeRecyclerView() {
            taskList.adapter = adapter
            taskList.layoutManager = LinearLayoutManager(this)
        }

        private fun initializeFAB() {
            fab.setOnClickListener {
                viewModel.addButtonClicked()
            }
        }

        private fun navigateToAddTask() {
            val intent = Intent(this, AddTaskActivity::class.java)
            startActivityForResult(intent, ADD_TASK_REQUEST)
        }

        companion object {
            private const val ADD_TASK_REQUEST = 0
        }
    }
```

# More

To see the complete code for this MVVM example, you can find it on [GitHub](https://github.com/AdamMc331/todo-monorepo/tree/master/todo-mvvm)!