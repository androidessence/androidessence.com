---
layout: post
author: adam
title: Implementing Paging Library With Retrofit Part 1&#58; Data Source
description: Overview of the paging library and how to implement it with Retrofit.
modified: 2018-04-17
published: true
tags: [architecture components, paging]
categories: [android]
---

The Android [Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html) libraries were announced at Google I/O 2017. They were released to help developers "design robust, testable, and maintainable apps" as well as managing "UI component lifecycle and handling data persistence". Usually when discussing these components I feel developers will go on to talk about [Room](https://developer.android.com/topic/libraries/architecture/room.html), or [ViewModels](https://developer.android.com/topic/libraries/architecture/viewmodel.html), but today we're going to be discussing the Paging library.

The [Paging Library](https://developer.android.com/topic/libraries/architecture/paging.html) is used to help developers paginate data from a source to display it for the user. A common reason to use pagination is that you have a lot of data to show the user, but you don't want to slow down the device with a long network call or database request. We also don't want to show the user any more data than they need.

A seasoned Android developer who's reading this may start to worry about things like "how do I know when we've reached the end of a RecyclerView" or "how do I make sequential Retrofit calls when that happens" and many other concerns. None of this should be a concern - that's what the paging library is here to do.

<!--more-->

To demonstrate the paging library, we'll build a simple PokeDex app that displays a list of Pokemon, automatically loading more for us as we scroll:

![AndroidEssence](https://thepracticaldev.s3.amazonaws.com/i/am2687bnk1e3vzro9jlg.gif)

## Background

This post will not go over Retrofit directly, but you can view that in a [previous AndroidEssence](https://androidessence.com/android/getting-started-with-retrofit/) post. It will help to familiarize yourself with the [PokeApi](https://pokeapi.co/), specifically calling for a list of pokemon with some limit and offset, which we can do like this: `https://pokeapi.co/api/v2/pokemon/?limit=2&offset=0`. 

Please be aware that at the time of publishing the paging library is also in beta, so the API is subject to changes.

## DataSource

Diving into the paging library, the first class we want to understand is the [DataSource](https://developer.android.com/reference/android/arch/paging/DataSource.html). This class is used to supply data to a [PagedList](https://developer.android.com/reference/android/arch/paging/PagedList.html), which is simply a list implementation that loads pages/chunks of data from the DataSource. 

There are multiple implementations of DataSource, but for this example we want a [PageKeyedDataSource](https://developer.android.com/reference/android/arch/paging/PageKeyedDataSource.html). Notice how in our API call we can paginate through Pokemon by changing the `offset` parameter? Since that offset parameter works like a "page" number for which subset of Pokemon we want, and it's the only thing that changes in each call, PageKeyedDataSource is the one we want. 

Let's start by defining our DataSource, including any properties we will need and the retry mechanism. Note that the generic types are for the pagination key (in this example, an Int), and the type of object in the resulting list (in this example, a BasePokemon):

```kotlin
    /**
     * Paged data source for fetching a list of pokemon. A [PageKeyedDataSource] is helpful when the
     * data is fetched by a key such as a page number, or in this example an offset.
     */
    class PageKeyedBasePokemonDataSource(private val pokeAPI: PokeAPI, private val retryExecutor: Executor) : PageKeyedDataSource<Int, BasePokemon>() {

        // Keep a function reference for the retry event
        private var retry: (() -> Any)? = null

        // One network state is used while we are loading and paging, but the initial load is always
        // called first so we need to keep track of that state as well.
        val networkState = MutableLiveData<NetworkState>()
        val initialLoad = MutableLiveData<NetworkState>()

        fun retryAllFailed() {
            val prevRetry = retry
            retry = null

            prevRetry?.let {
                retryExecutor.execute {
                    it.invoke()
                }
            }
        }
    }
```

In the code above, `NetworkState` is just a sealed class representing the [different states](https://github.com/AdamMc331/PokeDex/blob/master/app/src/main/java/com/adammcneilly/pokedex/network/NetworkState.kt) the data source can have based on network requests. 

Next, we have three methods we need to override:

[loadBefore](https://developer.android.com/reference/android/arch/paging/PageKeyedDataSource.html#loadBefore(android.arch.paging.PageKeyedDataSource.LoadParams%3CKey%3E,%20android.arch.paging.PageKeyedDataSource.LoadCallback%3CKey,%20Value%3E)) is used to append data to the initial paged list. In this case, we don't have any data to prepend, so we leave it unused.

```kotlin
    override fun loadBefore(params: LoadParams<Int>, callback: LoadCallback<Int, BasePokemon>) {
        // Not implemented because we only append to our initial load.
    }
```

[loadInitial](https://developer.android.com/reference/android/arch/paging/PageKeyedDataSource.html#loadInitial(android.arch.paging.PageKeyedDataSource.LoadInitialParams%3CKey%3E,%20android.arch.paging.PageKeyedDataSource.LoadInitialCallback%3CKey,%20Value%3E)) is called first and fetches our initial data. Once we have our initial data, we'll parse out the next offset key from the API response, and notify a callback. If there's an error, we just post that NetworkState to the LiveData. This is what the method will look like, along with relevant helper functions:

```kotlin
    override fun loadInitial(params: LoadInitialParams<Int>, callback: LoadInitialCallback<Int, BasePokemon>) {
        val request = pokeAPI.getPokemonList(params.requestedLoadSize, 0)

        networkState.postValue(NetworkState.Loading())
        initialLoad.postValue(NetworkState.Loading())

        // This is triggered by a refresh, so we execute synchronously
        try {
            val response = request.execute()
            handleLoadInitialSuccess(response, callback)

        } catch (ioe: IOException) {
            handleLoadInitialError(ioe, params, callback)
        }
    }

    private fun handleLoadInitialSuccess(response: Response<PokemonListResponse>, callback: LoadInitialCallback<Int, BasePokemon>) {
        val items = getPokemonListFromResponse(response)
        val previousKey = 0
        val nextKey = getNextKeyFromResponse(response)
        retry = null

        networkState.postValue(NetworkState.Loaded())
        initialLoad.postValue(NetworkState.Loaded())

        callback.onResult(items, previousKey, nextKey)
    }

    private fun handleLoadInitialError(e: Throwable?, params: LoadInitialParams<Int>, callback: LoadInitialCallback<Int, BasePokemon>) {
        retry = {
            loadInitial(params, callback)
        }

        val error = NetworkState.Error(e?.message)
        networkState.postValue(error)
        initialLoad.postValue(error)
    }

    private fun getPokemonListFromResponse(response: Response<PokemonListResponse>): List<BasePokemon> {
        val data = response.body()
        return data?.results ?: ArrayList()
    }

    private fun getNextKeyFromResponse(response: Response<PokemonListResponse>): Int {
        val data = response.body()
        val uri = Uri.parse(data?.next)
        return uri.getQueryParameter("offset").toInt()
    }
```

As we scroll, we'll want to request and append more data to the list. This is done by the [loadAfter](https://developer.android.com/reference/android/arch/paging/PageKeyedDataSource.html#loadAfter(android.arch.paging.PageKeyedDataSource.LoadParams%3CKey%3E,%20android.arch.paging.PageKeyedDataSource.LoadCallback%3CKey,%20Value%3E)) method:

```kotlin
    override fun loadAfter(params: LoadParams<Int>, callback: LoadCallback<Int, BasePokemon>) {
        networkState.postValue(NetworkState.Loading())

        pokeAPI.getPokemonList(params.requestedLoadSize, params.key).enqueue(object : Callback<PokemonListResponse> {
            override fun onFailure(call: Call<PokemonListResponse>?, t: Throwable?) {
                handleLoadAfterError(t, params, callback)
            }

            override fun onResponse(call: Call<PokemonListResponse>?, response: Response<PokemonListResponse>?) {
                if (response?.isSuccessful == true) {
                    handleLoadAfterSuccess(response, callback)
                } else {
                    val error = Throwable("Error fetching Pokemon: ${response?.code()}")
                    handleLoadAfterError(error, params, callback)
                }
            }
        })
    }

    private fun handleLoadAfterSuccess(response: Response<PokemonListResponse>, callback: LoadCallback<Int, BasePokemon>) {
        val items = getPokemonListFromResponse(response)
        val nextKey = getNextKeyFromResponse(response)
        retry = null

        networkState.postValue(NetworkState.Loaded())
        callback.onResult(items, nextKey)
    }

    private fun handleLoadAfterError(e: Throwable?, params: LoadParams<Int>, callback: LoadCallback<Int, BasePokemon>) {
        retry = {
            loadAfter(params, callback)
        }

        val error = NetworkState.Error(e?.message)
        networkState.postValue(error)
    }
```

You can view the full data source complete with KDocs [here](https://github.com/AdamMc331/PokeDex/blob/master/app/src/main/java/com/adammcneilly/pokedex/basepokemon/PageKeyedBasePokemonDataSource.kt).

Once we've built our DataSource, we'll want to implement a [DataSource.Factory](https://developer.android.com/reference/android/arch/paging/DataSource.Factory.html) to create this source. We'll reference this later in our repository.

```kotlin
    class BasePokemonDataSourceFactory(private val pokeAPI: PokeAPI, private val retryExecutor: Executor) : DataSource.Factory<Int, BasePokemon>() {
        val sourceLiveData = MutableLiveData<PageKeyedBasePokemonDataSource>()

        override fun create(): DataSource<Int, BasePokemon> {
            val source = PageKeyedBasePokemonDataSource(pokeAPI, retryExecutor)
            sourceLiveData.postValue(source)
            return source
        }
    }
```

This post was broken up here as there was a lot of code and explanation required for the DataSource, so check out [part 2]({{ site.baseurl }}{% link _posts/2018-04-17-implementing-paging-library-with-retrofit-part-2.md %}) for information how to create your repository and implement this inside a ViewModel.