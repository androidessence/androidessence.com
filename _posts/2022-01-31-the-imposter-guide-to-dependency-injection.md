---
layout: post
author: adam
title: "The Imposter's Guide To Dependency Injection"
modified: 2022-01-31
published: true
tags: [dependency injection]
categories: [android]
---

Dependency Injection is one of the hottest topics in Android and software development in general. It's also a topic that can provide a lot of anxiety and create imposter syndrome for developers. 

In this post, we'll take incremental steps toward understanding DI, why we need it, and how to implement it inside our applications.

<!--more-->

If interested, you can find this content in video form on YouTube:

<iframe width="560" height="315" src="https://www.youtube.com/embed/Nr_njiLsjcM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

---

# Code Without Dependency Injection

To start, let's review problems we might face if we write code without dependency injection. Consider the following code, where we have a ProfileViewModel class, and we want to track an event every time the user views a profile. This way we can see how common the screen is compared to others. 

```kotlin
class ProfileViewModel() {

	fun onProfileLoaded() {
		Firebase.analytics.logEvent("viewed_profile")
	}
}
```

This code doesn't look like it causes any problems at first glance. In fact, if you look at the documentation for Firebase Analytics and similar tools, it will suggest writing code like this. So don't be ashamed if your codebase looks something like this. For code like this, there are two specific problems I'd lke to highlight: 

1. Our code relies so heavily on Firebase Analytics, that if we wanted to change to another vendor we'd have to update every ViewModel in the application. 
2. We will have difficulty writing tests for this piece of code. 

# Testing Difficulties

A good unit test to write for this piece of code is one that verifies when `onProfileLoaded` is called, that we also track the correct analytics event. If we start writing this test, though, we'll recognize a problem very quickly. How can we verify a call was made to Firebase? 

```kotlin
class ProfileViewModelTest {

    @Test
    fun verifyEventTracked() {
        val viewModel = ProfileViewModel()
        viewModel.onProfileLoaded()

        // No Ability To Verify Event Tracked
    }
```

Not only that, this particular test will just crash. We'll get a RuntimeException that our main looper is not mocked, because of some work that's happening inside the Firebase Analytics library:

```
Caused by: java.lang.RuntimeException: Method getMainLooper in android.os.Looper not mocked.
```

# Dependency Injection

Our code has a _dependency_ on Firebase Analytics. We can help solve this testing problem by _injecting_ it to the constructor:

```kotlin
class ProfileViewModel(
    private val analytics: FirebaseAnalytics = Firebase.analytics,
) {

    fun onProfileLoaded() {
        analytics.logEvent("viewed_profile")
    }
}
```

This approach is often referred to as "Constructor Injection", where dependencies are supplied via the constructor of a class. This is the core concept of dependency injection. Kelly Shuster summed it up well in her Tweet:

{% twitter 1020363593060093952 %}

DI has a complicated name for a concept that can be reduced down to "passing stuff in." 

# Updating Our Test

Now that we have a way to supply our dependency, we can update our test accordingly to provide a fake implementation of Firebase Analytics, that we can then use to verify the proper event was tracked:

```kotlin
class ProfileViewModelTest {

    @Test
    fun verifyEventTracked() {
        val mockAnalytics = FakeFirebaseAnalytics()
        val viewModel = ProfileViewModel(mockAnalytics)
        viewModel.onProfileLoaded()

        mockAnalytics.verifyEventLogged("viewed_profile")
    }
}
```

However, when we do something like this we should be considering the other problem discussed earlier. Despite injecting this dependency, we still have a strong reliance on Firebase Analytics. We should avoid this, so we can support the ability to provide a different analytics service in the future.

# Wrapping Dependencies

We could create our own interface that defines an AnalyticsTracker, and a Firebase implementation of it. In the future, we can create other implementations for other services.

```kotlin
interface AnalyticsTracker {
    fun trackEvent(eventName: String)
}

class FirebaseAnalyticsTracker : AnalyticsTracker {
    override fun trackEvent(eventName: String) {
        Firebase.analytics.logEvent(eventName)
    }
}
```

Now we can have our ViewModel depend on this interface instead:

```kotlin
class ProfileViewModel(
    private val analytics: AnalyticsTracker = FirebaseAnalyticsTracker(),
)
```

Up until this point in the blog post, you've gained an understanding of the core concept of dependency injection, why we need it, and how to implement it for a given class. Having said that, let's address a new question:

> Why does dependency injection seem so much more complicated than passing stuff in?

There are a few advanced topics of dependency injection, starting with the idea of sharing dependencies. We can continue with our analytics tracking example. Analytics are everywhere, so we want to be able to provide one instance of an analytics tracker to be used by each screen. We also want to create a setup so that we can make one change to have every screen updated accordingly. 

# Dependency Injection Recipe

To achieve this goal, we can follow three steps.

1. Create a container to hold the dependencies used in our application.
2. Modify our `Application` class to be the host of those dependencies.
3. Update our individual screens to request those dependencies. 

To implement the first step, we can create a class called `AppDependencies`. Notice how our analytics tracker is typed to the interface from earlier, but will return a Firebase implementation:

```kotlin
class AppDependencies {
    val analyticsTracker: AnalyticsTracker
        get() = FirebaseAnalyticsTracker()
}
```

Next, we can create a reference to this dependency container inside our `Application` class:

```kotlin
class MyApp : Application() {
    val appDependencies = AppDependencies()
}
```

Lastly, we can get a reference to our Application Context from within an Activity or Fragment to request the dependencies:

```kotlin
class ProfileActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val appDependencies = (this.application as MyApp).appDependencies
        val analyticsTracker = appDependencies.analyticsTracker
        val profileViewModel = ProfileViewModel(analyticsTracker)
    }
}
```

You can learn more about manual dependency injection from [this Android guide](https://developer.android.com/training/dependency-injection/manual).

# Dependency Injection Libraries

Every few months a flamewar emerges on Android Twitter about which dependency injection library people should use. This only contributes to the confusion and anxiety people face when discussing dependency injection, which is why discussing them has been saved for the end of this post. DI libraries exist to help reduce some of the boilerplate code of a manual approach, but they actually follow the same recipe: create a group of dependencies, store them in your Application class, create a mechanism for requesting those dependencies. 

Let's look briefly at two dependency injection libraries and how they achieve this. 

## Hilt

[Hilt](https://developer.android.com/training/dependency-injection/hilt-android) is the official recommendation from Google for a dependency injection library. It manages dependencies through annotation processing. I recommend viewing the official docs to learn more, but let's review how Hilt uses our DI recipe. 

First, we can create a `Module` in Hilt that defines a group of dependencies. Here is an example module - don't be scared about all the annotations. The key point here is that we have a way to create an instance of `AnalyticsTracker` of type `FirebaseAnalyticsTracker`:

```kotlin
@Module
@InstallIn(ActivityComponent::class)
abstract class AnalyticsModule {

  @Binds
  abstract fun bindAnalyticsTracker(
    analyticsTracker: FirebaseAnalyticsTracker
  ): AnalyticsTracker
}
```

Next, we can setup our Application class. Hilt makes this incredibly easy, we just need to use the `@HiltAndroidApp` annotation:

```kotlin
@HiltAndroidApp
class MyApp : Application()
```

Last, we need a mechanism for requesting dependencies. With Hilt, we just use the `@Inject` annotation:

```kotlin
@AndroidEntryPoint
class ProfileActivity : Activity() {

    @Inject 
    lateinit var analytics: AnalyticsTracker
}
```

## Koin

Another common DI library is [Koin](https://insert-koin.io/docs/quickstart/android/). While the syntax and technical implementations are different, we once again see the same recipe used.

Create a collection of dependencies:

```kotlin
val analyticsModule = module {
    single<AnalyticsTracker> {
        FirebaseAnalyticsTracker()
    }
}
```

Store them in your Application class: 

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        startKoin {
            modules(analyticsModule)
        }
    }
}
```

Request them in your Activity:

```kotlin
class ProfileActivity : Activity() {

    val analytics: AnalyticsTracker by inject()
}
```

We're not going to dive into the nuanced differences between Hilt and Koin today. Instead, I just wanted to provide an overview of the libraries and show that they follow the same steps as writing our own manual dependency injection. 

# Recap

The dependency injection library you choose (or don't) matters so much less than why we need dependency injection in the first place. In this post, we reviewed how dependency injection enables better testing in our codebase, the ability to swap out dependencies for any reason in the future, and provides us with a centralized location to manage all of our applications dependencies. 

Did this help ease some of your anxiety around this complicated topic? Let me know in the comments, as well as any other imposter-syndrome-inducing topics we should look at in the future. 