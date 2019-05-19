---
layout: post
author: adam
title: Repository Pattern&#58; Properly Organizing Your Data Layer
description: An explanation of the repository pattern and why we need it.
modified: 2019-05-20
published: true
tags: [architecture]
categories: [android]
---

How to properly architect your application is a concern we as developers constantly face. There's unfortunately no one size fits all answer to it, and sometimes we don't even know where to begin. I've learned along my Android journey that the answer can also vary depending on what portion of your app you're trying to organize. Of course, you might say, [it depends](https://handstandsam.com/2019/03/10/it-depends-is-the-answer-to-your-android-question/).

When it comes to your data layer, though, there are some really good tips on how to write clean, maintainable code. One of them is the Repository Pattern, and I'd like to provide a quick walk through of what it is and why it's important. 

<!--more-->

# TL;DR

The repository pattern is a way to organize your code such that your ViewModel or Presenter class doesn't need to care about where your data comes from. It only cares about how to request data and what it gets back. 

# A Bad Example

Let's first look at what happens to our code when we _don't_ implement the repository pattern. Let's say I have a detail page for a Pokemon (yes, I love using Pokemon examples), and I want to fetch the data from a retrofit service. I might end up with a ViewModel looking something like this:

```kotlin
class DetailActivityViewModel(
    private val pokemonAPI: PokemonAPI
) : BaseObservableViewModel() {
    ...

    init {
        job = CoroutineScope(dispatcherProvider.IO).launch {
            ...

            val pokemon = pokemonAPI.getPokemonDetailAsync(pokemonName).await()

            ...
        }
    }

    ...
}
```

While this may not look awful, it actually poses an interesting limitation. What if you were asked to fetch from a GraphQL API instead? Or even a local database? What if you wanted a mix, or to A/B test multiple approaches? 

Depending on which one of those you chose, this gets really ugly. First and foremost, you'd have to add the relevant properties to your ViewModel, and then your ViewModel class eventually becomes bloated with data layer work that really doesn't belong there anymore. 

If you've ever found yourself in this spot, even if you haven't yet hit this limitation (some people only use a Retrofit API and that's fine), you may want to consider helping your future self with the repository pattern.

# Repository Interface

Going back up to the TL;DR, your ViewModel shouldn't care where the information comes from. In many programming problems where we don't care about the implementation of something, we can put the contract of what we do care about in an interface.

We can start there by defining our interface for what our data fetching behavior should be:

```kotlin
interface PokemonRepository {
    suspend fun getPokemon(): PokemonResponse
    suspend fun getPokemonDetail(pokemonName: String): Pokemon
}
```

Once we've defined that, we should update our ViewModel to use this interface:

```kotlin
class DetailActivityViewModel(
    private val repository: PokemonRepository
) : BaseObservableViewModel() {
    ...

    init {
        job = CoroutineScope(dispatcherProvider.IO).launch {
            ...

            val pokemon = repository.getPokemonDetail(pokemonName)

            ...
        }
    }

    ...
}
```

Now we're in a good place to take control of where our data comes from, and update it as needed without worrying about updating our ViewModel.

# The Implementation

If you were someone who was only using one data source, like a Retrofit API, you only have to worry about creating one implementation, which can be done with some heavy copy and paste. In the Pokedex example, we converted our implementation to this:

```kotlin
open class PokemonRetrofitService(
    private val api: PokemonAPI
): PokemonRepository {
    override suspend fun getPokemon(): PokemonResponse {
        return api.getPokemonAsync().await()
    }

    override suspend fun getPokemonDetail(pokemonName: String): Pokemon {
        return api.getPokemonDetailAsync(pokemonName).await()
    }
}
```

Now, we can just update our call site for creating the ViewModel to use this implementation:

```kotlin
private val viewModelFactory = object : ViewModelProvider.Factory {
    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        val pokemonAPI = ...
        val repository = PokemonRetrofitService(pokemonAPI)

        return DetailActivityViewModel(repository) as T
    }
}
```

# Multiple Implementation Example

This pattern is especially important if you want to consider multiple implementations for fetching data. A common example may be that you want to also fetch data from a local database, in addition to your network service.

Let's consider an example where we have an explicit offline mode that fetches data from a database. If your code is already using the repository pattern, we don't need to worry about updating the ViewModel.

All we would need to do is create our `DatabasePokemonService` and then we can conditionally pass that into the ViewModel:

```kotlin
private val viewModelFactory = object : ViewModelProvider.Factory {
    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        val repository = getPokemonRepository()

        return DetailActivityViewModel(repository) as T
    }
}

private fun getPokemonRepository(): PokemonRepository {
    if (offlineMode()) {
        return DatabasePokemonService()
    } else {
        return RetrofitPokemonService()
    }
}
```

# Testing Benefits

When you move your data fetching code into an interface, you also make unit testing a lot easier! Our unit tests for the ViewModel don't have to worry about the data implementation either, we can just use Mockito to mock the interface and stub test data that way:

```kotlin
class DetailActivityViewModelTest {
    ...

    private val mockRepository = mock(PokemonRepository::class.java)

    @Test
    fun loadData() {
        val testPokemon = Pokemon(name = "Adam", types = listOf(TypeSlot(type = Type("grass"))))
        
        whenever(mockRepository.getPokemonDetail(anyString())).thenReturn(testPokemon)

        ...
    }

    ...
}
```

Now we don't have to worry about creating a room database in our unit tests, or a retrofit service. Dealing with an interface can help avoid all of those struggles. 

# Resources

I hope you found this helpful in understanding how to properly organize your data fetching code. If you want to see the repository pattern in action (although I don't do this A/B testing scenario), you should check out this [Pokedex project](https://github.com/AdamMc331/PokeDex) on GitHub. 

If you like analyzing the code directly, you can see the repository pattern implemented in [this pull request](https://github.com/AdamMc331/PokeDex/pull/20).