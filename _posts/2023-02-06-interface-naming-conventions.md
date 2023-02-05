---
layout: post
author: adam
title: "Interface Naming Conventions"
modified: 2023-02-06
published: true
tags: [architecture, practices]
categories: [android]
---

Many engineers will tell you that one of the most complicated responsibilities of our job is naming things. Variables, classes, functions, everything we write requires conscious thought. 

A special case among these are interfaces. This is because we not only have to name an interface, but we need to decide how to name the implementations as well.

<!--more-->

---

## Basic Convention

Traditionally, and by this I mean in the textbooks I read in college, interfaces and their implementations shared the same naming convention. I've seen this surfaced in one of two ways.

1. Using an `IInterface` and `Implementation` convention:

```kotlin
interface IBookRepository {
    // ...
}

class BookRepository {
    // ...
}
```

2. Using an `Impl` suffix:

```kotlin
interface BookRepository {
    // ...
}

class BookRepositoryImpl {
    // ...
}
```

## Problems

This approach, while it does clearly separate the convention between an interface and its implementation, has a couple inherent problems.

1. It assumes one implementation per interface. If we write a `BookRepository` and `BookRepositoryImpl`, but need to add a second, different `BookRepository` what do we call it? `BookRepositoryImpl2`? 
2. Similarly, the `Impl` suffix doesn't provide any information about how the implementation operates. Is it pulling local data? Remote data? What is the data source used by `BookRepositoryImpl`? All of these questions require active investigation and diving into the code to understand and come back with a confident answer. 

## Alternatives

To avoid these problems, we can consider a number of alternative naming conventions for our interfaces and their implementations. As always, we have multiple different solutions, that you may choose to stick to one, mix and match, or change based on your situation. I've decided to highlight a few different approaches I have tried myself, and have heard used by others. Have different ideas? Let me know in the comments!

### Naming With Data Source

If our application has a `BookRepository` data source interface, the implementation can be named based on the data source used to request books. Some examples may look like this:

```kotlin
interface BookRepository

class GoogleBooksBookRepository : BookRepository

class OpenLibraryBooksRepository : BookRepository

class NewYorkTimesBookRepository : BookRepository
```

### Naming With Situation

Sometimes, our implementation might always be specific to the same data source/dependency. We still may want to leverage interfaces because it helps with testing, or some other situation. In these moments, we can consider naming our interface based on the situation it is being used in. For example, consider we include some interface for Crashlytics initialization. In our production app, we can name it accordingly.

```kotlin
interface CrashlyticsInitializer

class ProdCrashlyticsInitializer : CrashlyticsInitializer
```

Why is this better than `CrashlyticsInitializerImpl`? I believe this is better because it begins to set a precedent for other situations we might use a `CrashlyticsInitializer`. Situations like our device tests, where we can have a `TestCrashlyticsInitializer` or a special debug flavor that includes a `DebugCrashlyticsInitializer`. 

### Other Considerations

The above suggestions are my personal preference, but there are other situations you may want to consider. By prefixing the implementations, searching for them in the IDE becomes slightly less straight forward. When searching in Android Studio, for example, if we start typing `BookRepository`, any files that are named `BookRepositoryImpl` get included in the same search. For that reason, you may consider taking the above suggestions, and combining them with the traditional suffix, like this:

```kotlin
interface BookRepository

class BookRepositoryGoogleBooksImpl : BookRepository

class BookRepositoryOpenLibraryImpl : BookRepository

class BookRepositoryNewYorkTimesImpl : BookRepository
```

## Recap

At the end of the day, the naming convention you and your team use is a personal choice. By using interfaces, we're making a good choice toward scalable applications. The naming convention is mostly cosmetic, but I hope you've seen that it can serve a purpose for discoverability and quickly understanding the codebase by reading class names. What I believe matters most is that you take the time to think about it. Your conversation may ultimately end with "well, there's no dependency or situation specific usage here, so we just want to fall back on the `Impl` suffix. That's okay! Not all suggestions apply to all situations, but hopefully this post helps you find ways to provide more clarity in class names.
