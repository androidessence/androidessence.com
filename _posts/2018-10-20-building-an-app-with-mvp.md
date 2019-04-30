---
layout: post
author: adam
title: Building An Application With MVP
description: Demonstrates how to build an Android application with MVP architecture.
modified: 2018-10-20
published: true
tags: [architecture, mvp]
categories: [android, tutorial]
---

This is the first post in what will be an ongoing series to demonstrate a few different architecture patterns that are used for Android development. You can find the code for each of them, often appearing before the blog posts, by following [this repo](https://github.com/AdamMc331/todo-monorepo). Give it a star!

The first architecture pattern we're going to walk through is MVP.

<!--more-->

# MVP

MVP stands for Model View Presenter. The name comes from each of the three components that are involved in this architecture pattern. A good place to start is understand what each of those components mean:

## Model

Model, in this context, doesn't necessarily refer to your model classes that you write. I prefer to think of it as the data source for your application. This could be a database, a remote server, or even just dummy data that you supply. Often, you will see developers put the code for this in some kind of `Repository` class. That is what we will refer to as our model.

## View

The view is the component responsible for any UI work. This includes displaying data, and handling click events. It does _not_ include the business logic for those click events, which I will clarify in the next paragraph. In many cases, you will have an activity or a fragment that represents the view. 

## Presenter

The presenter is responsible for most business logic in your application. That is everything from "what do we do when the view first loads" to "what do we do when a button is clicked". The presenter is notified of these events from the view, performs some actions, and tells the view what to do. We'll see this explained more in a second.

# Communication Flow 

To understand the communication flow a little better, let's look at a short flow chart:

![](/assets/mvp/mvp-diagram.png)

Having explained each component, let's break down the communication:

* There is one way communication between the model and presenter. Usually this communication is pretty quick: the presenter asks for data, the model hands it back. Note: Some people may make this two-way communication, but we won't do that here.
* There is two way communication with the view and the presenter. Usually this is action driven: the view says "this thing happened" and the presenter responds to it, and then tells the view "display this data" or "move to this screen" as possible actions. 

The biggest benefit of all of that is recognizing we have a clear separation of concerns here. Notice how the model and the view never talk to each other? That's because they don't need to. They shouldn't be that tightly coupled together.

This separation also makes our app easily testable, which we'll see with the code later.

# Contract Class

This concept may be different depending on which developers you talk to, but I'll be very opinionated here and say that I like having a contract class. The purpose of this class is to define the required behavior of each component. We can do this through interfaces.

Let's say we wanted to create a TODO list, and start by just showing an activity with a list of tasks on it. If we're building this with MVP, we may want a Model/View/Presenter with the following responsibilities:

```kotlin
class TaskListContract {

    interface View {
        fun showTasks(tasks: List<BaseTask>)
        fun navigateToAddTask()
    }

    interface Presenter {
        fun addButtonClicked()
        fun viewCreated()
        fun viewDestroyed()
    }

    interface Model {
        fun getTasks(): List<BaseTask>
    }
}
```

This breaks down the responsibilites of each component pretty clearly:

* The view, at a minimum, can display a list of tasks and navigate to a new screen.
* The presenter needs to handle any actions that happen in the view, such as the view being created/destroyed, and when a button is clicked.
* The model's only responsibility is fetching our tasks.

Now that we have each component, let's go over writing each one.

# Model

For our model, we'll create a repository class like I mentioned earlier. This is the easiest component because we'll just have it return dummy data:

```kotlin
class TaskRepository : TaskListContract.Model {
    override fun getTasks(): List<BaseTask> {
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

The key thing to notice is that the model has no reference to the presenter. This goes back to what I said earlier about their being one-way communication. 

# View

I'm going to cut out some of the boilerplate code here, to highlight the MVP components of this:

```kotlin
class TaskListActivity : BaseTaskListActivity(), TaskListContract.View {
    private val presenter = TaskListPresenter(this, TaskRepository())

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        initializeFAB()

        presenter.viewCreated()
    }

    private fun initializeFAB() {
        fab.setOnClickListener {
            presenter.addButtonClicked()
        }
    }

    override fun navigateToAddTask() {
        val intent = Intent(this, AddTaskActivity::class.java)
        startActivityForResult(intent, ADD_TASK_REQUEST)
    }

    override fun showTasks(tasks: List<BaseTask>) {
        taskAdapter.tasks = tasks
    }

    override fun onDestroy() {
        presenter.viewDestroyed()
        super.onDestroy()
    }
}
```

The key takeaways from the above:

* Our view maintains a reference to the presenter. This is required for the two-way communication: our presenter also has a reference to the view which we're about to see.
* Our view overrides the required methods of the contract.
* Our view doesn't do any flashy business logic or condition statements here. Those should generally be avoided inside the view. Sometimes it is difficult when you deal with intents and view related classes, which you'll see in the sample code for this on GitHub.
* Related to the above, the view never tells the presenter what to do. It only tells the presenter about certain events (created/destroyed/button clicked) and the presenter determines how to respond.

Notice, while our view does supply the model to the presenter, there's still no communication between the two. 

# Presenter

Our presenter class in this example is pretty short:

```kotlin
class TaskListPresenter(
        private var view: TaskListContract.View?,
        private val model: TaskListContract.Model
) : TaskListContract.Presenter {

    override fun addButtonClicked() {
        view?.navigateToAddTask()
    }

    private fun getTasks() {
        val tasks = model.getTasks()
        view?.showTasks(tasks)
    }

    override fun viewCreated() {
        getTasks()
    }

    override fun viewDestroyed() {
        view = null
    }
}
```

The key takeaways:

* The presenter has a reference to the model and view so it can communicate with them. 
* The model and view are passed in via the constructor. This allows for easy unit testing, which we'll see later. It also abides by this separation of concerns pattern, clarifying the presenter doesn't care about the implementation of the view and model, but just knowing that it must have the behavior defined by each in the contract class.
* It's a good idea to null out the view when it's destroyed, so we don't try to update it after the fact. If our presenter performs long running actions, this is important.

# More

For a full sample of the MVP pattern on multiple features, check out the sample code in [GitHub](https://github.com/AdamMc331/todo-monorepo/tree/master/todo-mvp).
