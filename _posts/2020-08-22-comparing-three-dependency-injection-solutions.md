---
layout: post
author: adam
title: Comparing Three Dependency Injection Solutions
modified: 2020-08-22
published: true
tags: [dependency injection]
categories: [android]
---

During a recent live stream on my [Twitch](https://www.twitch.tv/adammc331) channel, we explored three different solutions to dependency injection on Android. A do it yourself approach, Koin, and Dagger Hilt. Let's revisit them side by side, and look at the nuances between them, so we can determine which solution we want to use in our own applications.

<!--more-->

---

If you'd like to watch the video where we explore the different solutions for the same application, you can find that here:

<iframe width="560" height="315" src="https://www.youtube.com/embed/9E82aKkiRqo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

---

# What Is Dependency Injection?

Dependency injection is a concept that can be really intimidating. It's two long words, thrown around with a number of other buzzwords and libraries, and can be paired with some complicated code samples. Let's try to break down dependency injection first. To do that, we'll look at the following constructor for a ViewModel:

```kotlin
class ArticleListViewModel(
    private val articleRepository: ArticleRepository
) : ViewModel() {
```

This ViewModel has a private variable and constructor paramter for an `ArticleRepository`. This means that the repository is a `dependency` required by the ViewModel. We do this for a couple reasons:

1. By supplying an implementation of the interface to the ViewModel, we have clear separation of concerns. The ViewModel knows how to interact with the repository, but it doesn't need to know anything about how to create it. 
2. This also allows us to write better unit tests for the ViewModel, as we can supply our own fake or mock repository in our test cases. 

Much like any other context where we would use the word, `injection` refers to how we supply this `dependency` to the ViewModel. We can do that through constructor injection (seen above), or field injection (create a public setter, to set a dependency after the component has been initialized). 

This is the minimum required work for dependency injection. If your app is doing stuff like this already, you are off to a great start. The concept of dependency injection can be simplified to "passing things in through the constructor". 

# Dependency Injection Frameworks

If dependency injection is just that, then why do we have complicated dependency injection libraries for Android? Well, these libraries go on to answer some additional concerns, after we have our base understanding of dependency injection:

1. Who is responsible for creating dependencies? If my ViewModel is created inside a Fragment, should the Fragment be responsible for creating dependencies? This doesn't seem right, because the Fragment is just a view, it should only care about view things. 
2. If we want to share a dependency across multiple ViewModels or other components, who should be responsible for creating the dependency then? We could put it into the Activity, or the Application class, but for similar reasons, that's not necessarily their responsibility. 
3. How do we handle the scope of dependencies? Who is responsible for cleaning up dependencies when they're no longer needed? Who should hang on to dependencies if we want to ensure they outlive our Fragments or Activities? 

As we build larger apps with more screens, more dependencies, and more classes, these questions will become increasingly important. We don't want our Fragment/Activity classes to be bloated with dependency management code. We don't want our Application class storing a reference to shared dependencies. 

# Do It Yourself Approach

For a deep dive into this type of approach, check out [DIY Dependency Injection with Kotlin by Sam Edwards](https://www.youtube.com/watch?v=ucZnYS7LmGU). 

Examining a DIY approach to dependency injection may help explain some of the later approaches, so we'll look at that first. Let's start by creating a dependency graph. A dependency graph is a grouping of related dependencies - and this may contain sub graphs. An example could be:

1. A base dependency graph for the application.
2. A sub graph that contains all of our repositories. 
3. A sub graph that contains all of the ViewModel factories. 

## Defining Graphs

We can define each graph as an interface:

```kotlin
// This graph creates any data repositories for the app. 
interface RepositoryGraph {
    fun articleRepository(): ArticleRepository
}

// This graph creates any ViewModel factories for the app. 
interface ViewModelFactoryGraph {
    fun articleListViewModelFactory(): ViewModelProvider.Factory
}

// This is the base graph, with all of the sub graphs for the application.
interface AppGraph {
    val repositoryGraph: RepositoryGraph
    val viewModelFactoryGraph: ViewModelFactoryGraph
}
```

## Implementing Graphs

To implement one of these dependency graphs, we can create a class that builds the dependencies. Try to use a descriptive name! For example, if your repositories are all talking to a remote service, we could do something like this:

```kotlin
class RemoteRepositoryGraph : RepositoryGraph {
    override fun articleRepository(): ArticleRepository {
        return RetrofitArticleRepository()
    }
}
```

It's okay to have one dependency graph depend on another:

```kotlin
class BaseViewModelFactoryGraph(
    private val repositoryGraph: RepositoryGraph
) : ViewModelFactoryGraph {
    override fun articleListViewModel(): ViewModelProvider.Factory {
        val repository = repositoryGraph.articleRepository()
        // Create the ViewModelFactory here
    }
}
```

Last, we can create the base instance:

```kotlin
class BaseAppGraph : AppGraph {
    override val repositoryGraph = RemoteRepositoryGraph()

    override val viewModelFactoryGraph = BaseViewModelFactoryGraph(repositoryGraph)
}
```

## Exposing Dependency Graphs

To expose the graph throughout the application, we can do so inside our Application class:

```kotlin
class StudyGuideApp : Application() {
    val dependencyGraph: AppGraph = BaseAppGraph()
}
```

This means we can trim down the code inside our Fragment that was creating the ViewModel:

```kotlin
class ArticleListFragment : Fragment() {
    private fun createViewModel() {
        val viewModelFactory = (requireContext().applicationContext as StudyGuideApp)
            .dependencyGraph
            .viewModelFactoryGraph
            .articleListViewModelFactory()

        val viewModelProvider = ViewModelProviders(this, viewModelFactory)
        viewModel = viewModelProvider.get(ArticleListViewModel::class.java)
    }
}
```

What's really great about the code above is that our Fragment class no longer has any knowledge of how to create dependencies. It is only responsible for requesting what it needs. We can do that by getting a reference to the application, and pulling the dependency from the graph, and that's it. Any logic around what that dependency really is, how it's created, etc, is managed elsewhere. 

## Pros And Cons

There are some benefits to a DIY approach:

1. All of the dependency management code is managed by us. There's no black box like when you rely on a third party library.
2. Our app size may be smaller, because we didn't have to import any new dependencies. 
3. We have actual references to dependencies, so we can leverage the IDE to command/control click into dependencies and see how they're actually created.
4. This can help with onboarding people who may not be familiar with a specific library. 

It's important to consider some pitfalls, too:

1. This approach is very verbose. It's a lot of additional code just to maintain your graphs, and to reference dependencies from them. 
2. As your app scales, or as dependencies change, this approach can have a lot of cascading effects throughout the codebase, that may be tedious to refactor. 
3. Handling scoping of dependencies is something you will have to manage yourself, in addition to what we've already seen. 

You can see a pull request of this approach [here](https://github.com/AdamMc331/AndroidStudyGuide/pull/25/files). 

Neither of those lists are exhaustive, but they can give you some insight into choosing the right approach for your project. Let's look at two more popular approaches and compare them. 

# Koin

The next approach we'll look at is [Koin](https://insert-koin.io/), a dependency injection framework for Kotlin. 

Conceptually, Koin works very similar to the previous DIY approach. Similar to creating dependency graphs, we'll create something in Koin called a [module](https://doc.insert-koin.io/#/koin-core/modules). Then, everywhere a dependency is used, we can have Koin look it up for us, kind of like how we had to look up dependencies from our base application graph. 

## How It Works

Before we look at code, let's discuss conceptually how Koin works. We write the code to tell Koin how to create any dependencies used inside our application. Then, whenever we need these dependencies, Koin will look them up at runtime. As we'll see below, this means that whenever we need a dependency, we can just call `get()` and Koin will do the rest.

This is similar to the last example, where we just got an instance of our application and requested the dependency. With Koin, it's even simpler, as we don't need to create that reference to the application class.

## Creating Modules

Let's create modules that match the two graphs from the last example:

```kotlin
val remoteRepositoryModule = module {
    // A single creates a singleton to be used by Koin.
    // To get a new instance each time, use `factory`.
    single<ArticleRepository> {
        RetrofitArticleService()
    }
}

val viewModelModule = module {
    viewModel {
        ArticleListViewModel(repository = get())
    }
}
```

Let's take a look at this line: `ArticleListViewModel(repository = get())`. 

By calling `get()`, this is how we look up dependencies in Koin. This will happen at runtime, and assumes we've defined all of the necessary dependencies in a module. If we don't do this, we will get runtime errors. 

Koin does provide a [checkModules](https://doc.insert-koin.io/#/koin-test/checkmodules_plugin?id=the-junit-checkmodules-test) test method that we can use to validate our modules, though.

## Starting Koin

To have Koin start managing all of our dependencies in order for us to look them up, we can initialize it inside our Application class with all of the modules:

```kotlin
class StudyGuideApp : Application() {
    override fun onCreate() {
        super.onCreate()

        startKoin {
            androidContext(this@StudyGuideApp)
            modules(remoteRepositoryModule)
            modules(viewModelModule)
        }
    }
}
```

## Looking Up Dependencies

When it comes time to look up dependencies, we can do this a few different ways:

```kotlin
class ArticleListFragment : Fragment() {
    // Looks up dependency immediately
    val articleRepository: ArticleRepository = get()

    // Koin will lazily inject this dependency, when we need it
    val lazyArticleRepository: ArticleRepository by inject()

    // Using Koin-ViewModel library, we can have it create our ViewModels
    // and remove all of the factory boilerplate
    val viewModel: ArticleListViewModel by viewModel()
}
```

## Thoughts

You can see a pull request of this approach [here](https://github.com/AdamMc331/AndroidStudyGuide/pull/26/files).

Despite the conceptual similarities, there's some noteworthy differences between Koin and the DIY approach:

1. We're using a third party library, so this means we don't have 100% ownership of the code, and this could have an impact on app size. 
2. We have less code overall, since Koin is the one that manages the dependencies, we don't need to define all of the graphs and connect them to the places that actually consume dependencies. 
3. Managing [scoping](https://doc.insert-koin.io/#/koin-core/scopes?id=what-is-a-scope) is supported by the library. 
4. Dependencies are looked up at runtime, meaning we don't have compile time validation, but we can leverage Koin's test artifact to ship with confidence. 

# Dagger Hilt

The third and final dependency injection solution we'll discuss is [Hilt](https://developer.android.com/training/dependency-injection/hilt-android). Hilt is a tool built on top of [Dagger](https://dagger.dev/) to improve dependency injection on Android. It is also Google's recommended solution for dependency injection. 

## How It Works

At a high level, Hilt takes a different approach to dependency management than Koin. While Koin looks up dependencies at runtime, Hilt uses annotation processing to validate all of our dependency management at compile time. 

## Creating Modules

A common theme across all approaches is the idea that we take related dependencies and group them. Hilt is no exception. Let's look at a Hilt module, and then we'll talk about each piece:

```kotlin
@Module
@InstallIn(ActivityRetainedComponent::class)
abstract class RemoteRepositoryModule {

    @Binds
    abstract fun bindArticleRepository(
        androidEssenceArticleService: AndroidEssenceArticleService
    ): ArticleRepository
}
```

1. A Hilt [module](https://developer.android.com/training/dependency-injection/hilt-android#hilt-modules) tells Hilt how to create instances of certain dependencies. 
2. Here, we use the [Binds](https://developer.android.com/training/dependency-injection/hilt-android#inject-interfaces) annotation which tells Hilt that whenever we need the `ArticleRepository` interface, we should supply the `AndroidEssenceArticleService` implementation. 
3. We annotate the module with @InstallIn so that Hilt knows which [components](https://developer.android.com/training/dependency-injection/hilt-android#generated-components) will be using this module. Here, our dependencies are supplied to a ViewModel, which is why we use `ActivityRetainedComponent`. 

NOTE: Not demonstrated here is the [provides](https://developer.android.com/training/dependency-injection/hilt-android#inject-provides) annotation which is used if you're creating an instance of something from a third party, like a Retrofit API. You can see an example of that [here](https://github.com/AdamMc331/AndroidStudyGuide/blob/development/app/src/main/java/com/adammcneilly/androidstudyguide/di/RetrofitModule.kt).  

## Consuming Dependencies

Once you've created a module and Hilt knows how to create dependencies, there's two ways we can consume them. 

First, we can have them injected into the constructor of another class, by annotating the constructor with `@Inject`:

```kotlin
class ArticleListDataManager @Inject constructor(
    private val articleRepository: ArticleRepository
) {
```

Or we can have it injected into the field of a class, as long as that class is marked with `@AndroidEntryPoint` annotation:

```kotlin
@AndroidEntryPoint
class ArticleListFragment : Fragment() {

    @Inject
    lateinit var articleRepository: ArticleRepository
}
```

## Jetpack Integrations

Similar to Koin, Hilt also has a [ViewModel extension](https://developer.android.com/training/dependency-injection/hilt-jetpack#viewmodels) which allows us to inject components straight into ViewModels:

```kotlin
class ArticleListViewModel @ViewModelInject constructor(
    private val articleRepository: ArticleRepository
) : ViewModel() {
```

Then we can skip all of the ViewModelProvider.Factory boilerplate in our Fragment:

```kotlin
@AndroidEntryPoint
class ArticleListFragment : Fragment() {

    private val viewModel: ArticleListViewModel by viewModels()
}
```

## Thoughts

You can find the pull request example for Hilt [here](https://github.com/AdamMc331/AndroidStudyGuide/pull/27/files). I also recommend this [YouTube playlist](https://www.youtube.com/watch?v=zTpM2olXCok&list=PLgCYzUzKIBE_MUlyvbCiOWsfq0nFgGXQ9) from Mitch Tabian to learn more.  

Some takeaways from looking at Hilt:

1. We have compile time validation of our dependencies, so we know early if something isn't right.
2. There's a lot of annotations, which can be confusing. The annotation processing can also have impacts on build time as it scales up. 
3. Scoping is also handled [by Hilt](https://developer.android.com/training/dependency-injection/hilt-android#component-scopes).
4. We have a little more code involved than Koin, but still less than the DIY approach. As our dependencies change, the number of places that our code must change is limited as well, which is different from the DIY approach. 

# Recap

While none of these sections were a true deep dive into any of the dependency injection frameworks, this should provide a quick introduction into what each one looks like, and serve as a quick side-by-side glance at the different approaches. 

If none of these approaches stand out to you, there's even more you can explore - like [Kodein](https://github.com/Kodein-Framework/Kodein-DI) or [Toothpick](https://github.com/stephanenicolas/toothpick). 

Like any other technology, which one is best for your project depends on a number of factors. Is your company strongly against third party libraries? You can roll your own approach. Are you looking for a DI framework that has a quick learning curve and setup? Koin is great for that. Would you prefer a more type safe approach that comes with compile time validation? Hilt is the one for you. 