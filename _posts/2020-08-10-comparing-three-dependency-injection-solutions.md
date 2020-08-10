---
layout: post
author: adam
title: Comparing Three Dependency Injection Solutions
modified: 2020-08-10
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
2. If I want to share a dependency across multiple ViewModels or other components, who should be responsible for creating the dependency then? We could put it into the Activity, or the Application class, but for similar reasons, that's not necessarily their responsibility. 
3. How do we handle the scope of dependencies? Who is responsible for cleaning up dependencies when they're no longer needed? Who should hang on to dependencies if we want to ensure they outlive our Fragments or Activities? 

As we build larger apps with more screens, more dependencies, and more classes, these questions will become increasingly important. We don't want our Fragment/Activity classes to be bloated with dependency management code. We don't want our Application class storing a reference to shared dependencies. 



