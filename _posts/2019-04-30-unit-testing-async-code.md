---
layout: post
author: adam
title: Unit Testing RxJava or Coroutine Code With Constructor Injection
description: A walk through for unit testing asynchronous tools such as RxJava or Coroutines.
modified: 2019-04-30
published: true
tags: [rxjava, coroutines, testing]
categories: [android]
---

Putting aside the long lasting debate right now about whether you should use RxJava or coroutines for your asynchronous code on Android, both camps often hit the same problem. How do I write unit tests for this? 

Unit testing asynchronous code is tricky, because we may need to know how to properly test callback APIs, or perhaps we just want things to run instantly and not worry about thread changes. We may also be wondering how to handle not having a "main" thread in a junit test, unlike a connected test. This post will be focusing on handling that last one.

<!--more-->

# The problem

Let's start by first analyzing the problem. Let's say I have the following code to fetch Pokemon from an API:

```kotlin
open class PokemonRepository(
    private val api: PokemonAPI,
    private val disposables: CompositeDisposable
) {
    fun fetchPokemon() {
        val subscription = api.getPokemon()
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe {
                // Do something with the response
            }

        disposables.add(subscription)
    }
}
```

Now, if we were to call this method from a junit test, we'll get an `ExceptionInInitializerError` because we're unable to mock the `AndroidSchedulers.mainThread()` scheduler. The solution to this would be to use [Schedulers.trampoline()](http://reactivex.io/RxJava/javadoc/io/reactivex/schedulers/Schedulers.html#trampoline--), which you can think of as an instant scheduler. There's a little more to it than that, which you can learn about in the docs, but it solves our use case. 

# The Solution

Unfortunately, with the repository code we have above, we can't easily change our unit tests to use .trampoline(). So we need to move the schedulers somewhere we can modify, like the constructor of the repository:

```kotlin
open class PokemonRepository(
    private val api: PokemonAPI,
    private val disposables: CompositeDisposable,
    private val processScheduler: Scheduler = Schedulers.io(),
    private val observerScheduler: Scheduler = AndroidSchedulers.mainThread()
) {
    fun fetchPokemon() {
        val subscription = api.getPokemon()
            .subscribeOn(processScheduler)
            .observeOn(observerScheduler)
            .subscribe {
                // Do something with the response
            }

        disposables.add(subscription)
    }
}
```

With the wonderful help of default parameters in Kotlin, we can supply the defaults to be used by the app which means at the call site we only have to supply two parameters just like before. Now, though, we can change the schedulers used inside a unit test:

```kotlin
class PokemonRepositoryTest {
    private val mockAPI = mock(PokemonAPI::class.java)
    private val repository = PokemonRepository(
        mockAPI,
        CompositeDisposable(),
        Schedulers.trampoline(),
        Schedulers.trampoline()
    )

    // ...
}
```

That's it! All we need to do to unit test our RxJava code is move our schedulers into the constructor. If you've already moved on from RxJava to coroutines, or are contemplating it, the solution to that is very similar.

# The Coroutines Version

Let's consider we wrote some coroutines code like this:

```kotlin
class MainActivityViewModel(
    repository: PokemonRepository,
) : BaseObservableViewModel() {
    // ...

    private var job: Job? = null

    // ...

    init {
        job = CoroutineScope(Dispatchers.IO).launch {
            withContext(Dispatchers.Main) {
                startLoading()
            }

            val newState = try {
                val response = repository.getPokemon()
                // Handle success
            } catch (error: Throwable) {
                // Handle error
            }

            withContext(Dispatchers.Main) {
                // Post the new state to the UI
            }
        }
    }

    // ..

    override fun onCleared() {
        super.onCleared()
        job?.cancel()
    }
}
```

If we do this, we'll run into _the same problem_ as our original RxJava code. `Dispatchers.Main` is only configured when running on Android, not inside JUnit. Also, we don't _need_ the IO dispatcher, when we can just use `Dispatchers.Unconfined` for junit tests. Similar to RxJava, we could put them both as constructor parameters, but I decided to create a data class that would allow me to override any dispatcher:

```kotlin
data class DispatcherProvider(
    val IO: CoroutineDispatcher = Dispatchers.IO,
    val Main: CoroutineDispatcher = Dispatchers.Main,
    val Unconfined: CoroutineDispatcher = Dispatchers.Unconfined
)

class MainActivityViewModel(
    repository: PokemonRepository,
    dispatcherProvider: DispatcherProvider = DispatcherProvider()
) : BaseObservableViewModel() {
    // ...

    init {
        job = CoroutineScope(dispatcherProvider.IO).launch {
            withContext(dispatcherProvider.Main) {
                startLoading()
            }

            // ...

            withContext(dispatcherProvider.Main) {
                // Post the new state to the UI
            }
        }
    }

    // ...
}
```

Then, to call this in our junit tests, we can just override the dispatchers we want:

```kotlin
private val testProvider = DispatcherProvider(
    IO = Dispatchers.Unconfined, 
    Main = Dispatchers.Unconfined
)
```

That's it! Now we don't have to worry about the main looper being undefined for junit tests. 

I've kept this simple to highlight how we can modify the Schedulers/Dispatchers that are used in our unit tests. For a full code example check out the resources below, or let me know in the comments or [on Twitter](https://twitter.com/AdamMc331) if you have any questions.

# Resources

I've just recently switched the project in this post to use coroutines. However, I did [tag the last commit](https://github.com/AdamMc331/PokeDex/releases/tag/rxjava) with RxJava code if you'd like to see how that was used and unit tested.

To see how to use coroutines in a project and unit test them, you can view [this project](https://github.com/AdamMc331/PokeDex).