---
layout: post
author: adam
title: Implementing Paging Library With Retrofit Part 2&#58; Implementation
description: Overview of the paging library and how to implement it with Retrofit.
modified: 2018-04-17
published: true
tags: [architecture components, paging]
categories: [android]
---

In [part 1]({{ site.baseurl }}{% link _posts/2018-04-17-implementing-paging-library-with-retrofit-part-1.md %}) of this series we gave an overview of the paging library, and how to implement a [PageKeyedDataSource](https://developer.android.com/reference/android/arch/paging/PageKeyedDataSource.html). 

Now that we've created our DataSource, let's go over how to implement this. 

<!--more-->

## Repository

After creating our data source, we want to setup a repository to fetch data from this source. While you may be familiar with returning an Observable/LiveData/some other reactive component from your repository, we'll implement a `Listing` class for this example. The listing will be responsible for not only our data, but the state we are in and the refresh/retry methods:

```kotlin
	data class Listing<T>(
	        // the LiveData of paged lists for the UI to observe
	        val pagedList: LiveData<PagedList<T>>,
	        // represents the network request status to show to the user
	        val networkState: LiveData<NetworkState>
	        // retries any failed requests.
	        val retry: () -> Unit
	)
```

To implement a repository method to fetch pokemon, we'll first setup our data source, create a [PagedList](https://developer.android.com/reference/android/arch/paging/PagedList.html) of the data, and build out our Listing:

```kotlin
	class PokeRepository(
	        private val pokeAPI: PokeAPI,
	        private val retryExecutor: Executor
	) {
	    fun pokemonList(count: Int): Listing<BasePokemon> {
	        val sourceFactory = BasePokemonDataSourceFactory(pokeAPI, retryExecutor)

	        val livePagedList = LivePagedListBuilder(sourceFactory, count)
	                // provide custom executor for network requests, otherwise it will default to
	                // Arch Components' IO pool which is also used for disk access
	                .setFetchExecutor(retryExecutor)
	                .build()

	        return Listing(
	                pagedList = livePagedList,
	                networkState = Transformations.switchMap(sourceFactory.sourceLiveData, {
	                    it.networkState
	                }),
	                retry = { sourceFactory.sourceLiveData.value?.retryAllFailed() }
	        )
	    }
	}
```

If you're unfamiliar with it, `Transformations.switchMap` takes in a LiveData, watches for changes to it, applies a function to the new value, and applies the result of that function to the resulting LiveData from the switchMap. Let's explain the above example, because this comes up again later.

Consider this snippet (variables changed for clarity)

```kotlin
requestState = Transformations.switchMap(sourceFactory.sourceLiveData, {
    it.networkState
})
```

In this line, the switchMap monitors `sourceFactory.sourceLiveData`, and any time a value is published to `sourceLiveData`, it will grab the `networkState` from it, and publish that to the `requestState` LiveData.

## ViewModel

In this project, I used a `ViewModel` to fetch the data and expose it for the Fragment, but the same idea could be used in an MVP architecture too. This class is responsible for fetching data, and exposes a couple LiveData:

* One exposes the `PagedList<BasePokemon>` for our adapter.
* One exposes the `NetworkState` that we are in.

This class also maintains a reference to a retry function that can be called whenever a network request fails.

```kotlin
class BasePokemonViewModel(private val repository: PokeRepository) : ViewModel() {
    private val listingRequest = MutableLiveData<Listing<BasePokemon>>()
    val pokemon: LiveData<PagedList<BasePokemon>> = Transformations.switchMap(listingRequest, { it.pagedList })
    val networkState: LiveData<NetworkState> = Transformations.switchMap(listingRequest, { it.networkState })

    fun fetchPokemon() {
        if (listingRequest.value == null) {
            listingRequest.value = repository.pokemonList(20)
        }
    }

    fun retry() {
        val listing = listingRequest.value
        listing?.retry?.invoke()
    }
}
```

You'll notice we used the `switchMap` method again here. This means anytime a value is posted to `listingRequest`, our `pokemon` and `networkState` values will change to match the paged list and network state of the listing. 

## Adapter

We need to implement an adapter for our RecyclerView, but how do we implement one that knows when to request more data, and to handle this special data type called a `PagedList` that we implemented now?

Another benefit of the paging library is that you don't have to worry about this. You can just implement the [PagedListAdapter](https://developer.android.com/reference/android/arch/paging/PagedListAdapter.html) class that comes as part of the library. The implementation is similar to what you are likely already familiar with:

```kotlin
	class BasePokemonAdapter(
	        private val retryCallback: () -> Unit
	) : PagedListAdapter<BasePokemon, RecyclerView.ViewHolder>(POKEMON_COMPARATOR) {

	    private var networkState: NetworkState? = null

	    private val hasExtraRow: Boolean
	        get() = networkState != null && networkState !is NetworkState.Loaded

	    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
	        return when (viewType) {
	            VIEW_TYPE_NETWORK_STATE -> NetworkStateViewHolder.create(parent, retryCallback)
	            VIEW_TYPE_BASE_POKEMON -> BasePokemonViewHolder.create(parent)
	            else -> throw(IllegalArgumentException("View type $viewType is not supported."))
	        }
	    }

	    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
	        when (getItemViewType(position)) {
	            VIEW_TYPE_NETWORK_STATE -> (holder as NetworkStateViewHolder).bindNetworkState(networkState)
	            VIEW_TYPE_BASE_POKEMON -> (holder as BasePokemonViewHolder).bindPokemon(getItem(position))
	        }
	    }

	    override fun getItemCount(): Int {
	        return super.getItemCount() + if (hasExtraRow) 1 else 0
	    }

	    override fun getItemViewType(position: Int): Int {
	        return if(hasExtraRow && position == itemCount - 1) {
	            VIEW_TYPE_NETWORK_STATE
	        } else {
	            VIEW_TYPE_BASE_POKEMON
	        }
	    }

	    fun setNetworkState(networkState: NetworkState?) {
	        val previousState = this.networkState
	        val hadExtraRow = hasExtraRow
	        this.networkState = networkState
	        val hasExtraRow = hasExtraRow
	        if (hadExtraRow != hasExtraRow) {
	            if (hadExtraRow) {
	                notifyItemRemoved(super.getItemCount())
	            } else {
	                notifyItemInserted(super.getItemCount())
	            }
	        } else if (hasExtraRow && previousState != networkState) {
	            notifyItemChanged(itemCount - 1)
	        }
	    }

	    companion object {
	        const val VIEW_TYPE_NETWORK_STATE = R.layout.list_item_network_state
	        const val VIEW_TYPE_BASE_POKEMON = R.layout.list_item_base_pokemon

	        val POKEMON_COMPARATOR = object : DiffUtil.ItemCallback<BasePokemon>() {
	            override fun areContentsTheSame(oldItem: BasePokemon, newItem: BasePokemon): Boolean =
	                    oldItem == newItem

	            override fun areItemsTheSame(oldItem: BasePokemon, newItem: BasePokemon): Boolean =
	                    oldItem.name == newItem.name
	        }
	    }
	}
```

## Fragment

The last piece of functionality to implement is your view. This could be a fragment or an activity. There are two steps:

1. Use a `ViewModelProvider.Factory` to create your ViewModel. We need a factory because we want to create a ViewModel that has a constructor. This is not required if your ViewModel does not have any constructor parameters, you could instead do `ViewModelProviders.of(this).get(MyViewModel::class.java)`. 
2. Observe the LiveDatas exposed by the ViewModel, and use them to update our adapter.

```kotlin
class BasePokemonFragment : Fragment() {
    private lateinit var viewModel: BasePokemonViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        viewModel = ViewModelProviders.of(this, object : ViewModelProvider.Factory {
            override fun <T : ViewModel?> create(modelClass: Class<T>): T {
            	val api = App.getPokeAPI()
            	val executor = App.getDefaultExecutor()
                val repository = PokeRepository(api, executor)

                @Suppress("UNCHECKED_CAST")
                return BasePokemonViewModel(repository) as T
            }
        }).get(BasePokemonViewModel::class.java)
    }

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? =
            inflater.inflate(R.layout.fragment_pokemon_list, container, false)

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        initRecyclerView()

        viewModel.fetchPokemon()
    }

    private fun initRecyclerView() {
        rvPokemonList.layoutManager = LinearLayoutManager(context)

        val adapter = BasePokemonAdapter {
            viewModel.retry()
        }

        rvPokemonList.adapter = adapter

        viewModel.pokemon.observe(this, Observer<PagedList<BasePokemon>> {
            adapter.submitList(it)
        })

        viewModel.networkState.observe(this, Observer {
            adapter.setNetworkState(it)
        })
    }
}
```