---
layout: post
author: adam
title: Breaking the Buzzwords Barrier Part 3&#58; ViewModel
description: Explains what a ViewModel means in both MVVM and Android contexts. 
modified: 2018-06-01
published: true
tags: [architecture]
categories: [android, tutorial]
---

So far we've covered four big buzzwords used in our application:

1. Model-View-ViewModel
2. Room
3. RxJava
4. Repository Pattern

Now, we should circle back to the beginning. Following our diagram outlined in the [previous parts]({{ site.baseurl }}{% link _posts/2018-05-31-breaking-the-buzzwords-barrier-room-rx-repository.md %}), the next component we can begin to work on is our AccountViewModel:

![Android Essence](/images/buzzwords/architecture_viewmodel.png)

Naturally, this may bring up some confusion. We already discussed ViewModels in [part 1]({{ site.baseurl }}{% link _posts/2018-05-30-breaking-the-buzzwords-barrier-mvvm.md %}). Well, depending on context, we may not be referring to the same thing.

<!--more-->

# Architecture

When we refer to the MVVM architecture, the ViewModel here just refers to a specific component in your application's architecture. It does not have anything to do with Android or life cycles at this point. To quote the [Wikipedia article](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel):

> The view model is an abstraction of the view exposing public properties and commands. Instead of the controller of the MVC pattern, or the presenter of the MVP pattern, MVVM has a binder. In the view model, the binder mediates communication between the view and the data binder. The view model has been described as a state of the data in the model.

Up to this point, a ViewModel is nothing more than a component that binds data to your view. We can do this a number of ways, including data binding which we will discuss in a later post, but we'll leave it at that for now. An important take away from this discussion is that MVVM is not specific to Android - a ViewModel could be a class inside an iOS or even a desktop application, if you chose to architect it as such. 

# Android

So then what does a ViewModel mean in terms of Android discussions? Every time the [architecture components](https://developer.android.com/topic/libraries/architecture/) come up, it seems without fail that we begin talking about [ViewModels](https://developer.android.com/topic/libraries/architecture/viewmodel).

This is a new class added with the architecture components in 2017 to manage data in a lifecycle conscious way. For those of you who have been doing Android for a long time, you've experienced the struggles of activities being killed on rotation, and repulling for data, or learning the appropriate way to save state yourself. It's a hassle, and something every single Android developer has had to deal with.

However, just like they did with Room, the Google developers sought out to make this easier for everybody, and provided this class to do so.

> Architecture Components provides ViewModel helper class for the UI controller that is responsible for preparing data for the UI. ViewModel objects are automatically retained during configuration changes so that data they hold is immediately available to the next activity or fragment instance.

So in this context, when we refer to the architecture component, we're simply talking about a class that can maintain data throughout orientation changes. While this might sound counterintuitive, the Android class for ViewModel is completely separate from the MVVM concept, and can be mutually exclusive. You can use an MVVM architecture without this class, and you can use this class without an MVVM architecture.

# Overlap

Despite the ability to be mutually exclusive, there is often overlap in this discussion. The reason for that goes back to the purpose of a ViewModel in MVVM. As stated earlier, the ViewModel is simply an abstraction of the view that exposes certain properties. These properties could be a list of data to display, or the current state of your view.

An Android ViewModel is a way to persist data across orientation changes. What kind of data would you want to persist? A list of data you're displaying, or the current state of your view. Thus, they often go hand in hand because the ViewModel class in Android was made to handle many of the concerns faced by a ViewModel in MVVM. 

# Implementation

As I go over these buzzwords, I also want to show the implementation where it's relevant. I've trimmed this a little bit from the actual class in Cash Caretaker, but enough to demonstrate a couple key points:

1. The ViewModel manages the current state of the view, not our Fragment. It's exposed via a [BehaviorSubject](), so the Fragment can respond to it. 
2. The ViewModel interracts with the Repository we made in the last section to fetch data. It also checks to make sure it's not pulling data redundently.
3. The ViewModel also manages a [CompositeDisposable](http://reactivex.io/RxJava/javadoc/io/reactivex/disposables/CompositeDisposable.html) which holds our network subscription, and clears the network subscription when our ViewModel is cleared. This is to avoid memory leaks if there's a long running request.

If you don't understand the RxJava code below, head back to [part 2]({{ site.baseurl }}{% link _posts/2018-05-31-breaking-the-buzzwords-barrier-room-rx-repository.md %}) for some notes and links to external resources.

```kotlin
	class AccountViewModel(private val repository: CCRepository) : ViewModel() {
	    private val compositeDisposable = CompositeDisposable()
	    val state: BehaviorSubject<DataViewState> = BehaviorSubject.create()

	    fun fetchAccounts() {
	        if (state.value !is DataViewState.Success<*>) {
	            Timber.d("Loading Accounts")
	            postState(DataViewState.Loading())

	            val subscription = repository
	                    .getAllAccounts()
	                    .subscribeOn(Schedulers.io())
	                    .observeOn(AndroidSchedulers.mainThread())
	                    .subscribe(
	                            this::postState,
	                            Timber::e
	                    )

	            compositeDisposable.add(subscription)
	        }
	    }

	    private fun postState(newState: DataViewState) {
	        state.onNext(newState)
	        notifyChange()
	    }

	    override fun onCleared() {
	        compositeDisposable.dispose()
	    }
	}
```

Because this survives orientation changes, and because we do a state check before pulling accounts, we can save ourselves from unnecessarily requesting data every time the user rotates their phone, and instead only when the view (and ViewModel) are created for the first time. 

## Factory

Instantiating your ViewModel from inside your activity or fragment can be pretty easy:

```kotlin
val viewModel = ViewModelProviders.of(this).get(MyViewModel::class.java)
```

However, you may notice in the above example, we don't have any way to call a constructor. Unfortunately, there aren't any examples in the documentation, but you can do this using a [ViewModelProvider.Factory](https://developer.android.com/reference/android/arch/lifecycle/ViewModelProvider.Factory). There is an [explanation by Mohit Sharma](https://android.jlelse.eu/android-viewmodel-with-custom-arguments-d0ff0fba29e1) that is quick and to the point.

You can create an instance of this factory using an anonymous class, create your ViewModel with the constructor, and then return that. Here is an example from Cash Caretaker:

```kotlin
val viewModelFactory: ViewModelProvider.Factory by lazy {
    object : ViewModelProvider.Factory {
        override fun <T : ViewModel?> create(modelClass: Class<T>): T {
            val database = CCDatabase.getInMemoryDatabase(context!!)
            val repository = CCRepository(database)

            return AccountViewModel(repository) as T
        }
    }
}

val viewModel = ViewModelProviders.of(this, viewModelFactory).get(AccountViewModel::class.java)
```

# Conclusion

I hope this demonstrates clearly what a ViewModel means in terms of the software architecture, as well as what the ViewModel class in Android does for us. Congratulations on making it through! You can now cross off another intimidating buzzword, and have a discussion about why this is truly relevant. 

Keep an eye out for part 4 coming soon!