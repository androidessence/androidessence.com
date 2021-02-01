---
layout: post
author: adam
title: Storing Network Responses With Apollo HTTP Cache
modified: 2021-02-01
published: true
tags: [graphql]
categories: [android]
---

Caching is the practice of storing data that we requested previously so we can serve it faster in the future. This creates a better user experience by decreasing loading times. It also has long term benefits like reducing the number of network requests, to save on phone resources or potentially provide offline support. Today, we're going to discuss how to use the [HTTP cache](https://www.apollographql.com/docs/android/essentials/http-cache/) for the [Apollo Android SDK](https://github.com/apollographql/apollo-android).

<!--more-->

## Disclaimer

This post is written assuming some basic knowledge of the Apollo Android library and GraphQL. If you're unfamiliar with the topic, you can start with the [official documentation](https://www.apollographql.com/docs/android/essentials/get-started-kotlin/), or with this presentation from [Android Makers 2020](https://www.youtube.com/watch?v=np3JSkjisCk). 

You can find a sample project with all of the code from this blog post on [GitHub](https://github.com/AdamMc331/ApolloCaching). 

# HTTP Cache

From the [documentation](https://www.apollographql.com/docs/android/essentials/http-cache/), the HTTP cache is easier to set up but it does have some limitations. We'll start by understanding how it works, how to set it up ourselves, and what limitations we might run into. 

# How It Works

The HTTP cache stores data in a file directory on the device. With every GraphQL request we make, a unique key is created for that request. Then, upon response from our API, we store that response in a file with that unique key.

The next time our application makes that same request, Apollo will check our cache directory to see if a file with that key exists, and return the response that was stored, rather than requesting from the API again.

# Implementing HTTP Cache

To implement the HTTP cache, we need to do the following:

1. Create our cache directory. 
2. Set a max size of the directory. Once this max size is reached, Apollo will remove the oldest entries. 
3. Apply this cache in our Apollo Client builder.

In code, that would look like this:

```kotlin
val file = File(applicationContext.cacheDir, "apolloCache")

// 1 MB = 1 x 1024 x 1024
val numMegabytes = 1
val sizeInMegabytes = BYTES_PER_KILOBYTE * KILOBYTES_PER_MEGABYTE * numMegabytes

val cacheStore = DiskLruHttpCacheStore(file, sizeInMegabytes)

val httpCache = ApolloHttpCache(cacheStore)

val apolloClient = ApolloClient.builder()
    .httpCache(httpCache)
    .build()
```

# Querying From The Cache

Creating a cache and applying it to our ApolloClient instance isn't enough. We also need to tell the application when and how to use the cache, which we do through one of the [HttpCachePolicy](https://www.apollographql.com/docs/android/essentials/http-cache/#raw-http-response-cache) configurations. We have two options for setting a cache policy.

We can choose to set a default for our ApolloClient:

```kotlin
val apolloClient = ApolloClient.builder()
    .httpCache(httpCache)
    .defaultHttpCachePolicy(HttpCachePolicy.CACHE_FIRST)
    .build()
```

Or we can choose to set it per query:

```kotlin
val response = apolloClient.query(query)
    .toBuilder()
    .httpCachePolicy(HttpCachePolicy.CACHE_FIRST)
    .build()
    .await()
```

# Invalidating HTTP Cache

For the HTTP Cache, we have a few different ways we can expire or invalidate data.

1. We can set a time limit on the data, after which it is considered expired and not to be used.
2. We can expire a cache entry immediately after reading it.
3. We can clear the entire cache.

Here is the respective code for each option:

```
val oneHourPolicy = HttpCachePolicy.CACHE_FIRST.expireAfter(1, TimeUnit.HOURS)

val removeAfterRead = HttpCachePolicy.CACHE_FIRST.expireAfterRead()

// Clear manually
apolloClient.clearHttpCache()
```

# Debugging HTTP Cache

Once we've implemented an HTTP cache, it's important to understand if it's working the way we expect it to. There's a couple options for doing this. 

## Logging Cache Hits And Misses

When we create our HTTP Cache, we have a parameter on `ApolloHttpCache` to pass an instance of a `Logger`. We can use `ApolloAndroidLogger` to get detailed information. 

```kotlin
val file = // ...
val sizeInMegabytes = .. ,,,

val cacheStore = DiskLruHttpCacheStore(file, sizeInMegabytes)

val httpCache = ApolloHttpCache(cacheStore, ApolloAndroidLogger())
```

This class is from the [Apollo-Android-Support](https://github.com/apollographql/apollo-android/tree/main/apollo-android-support) artifact, and logs Apollo messages into the logcat for us. If we apply this logger, we can see as we run our app how the cache is affecting query results:

```
D/ApolloAndroidLogger: Cache first for request: ...
D/ApolloAndroidLogger: Cache HIT for request: ...

D/ApolloAndroidLogger: Cache first for request: ...
D/ApolloAndroidLogger: Cache MISS for request: ...

D/ApolloAndroidLogger: Cache first for request: ...
D/ApolloAndroidLogger: Cache HIT for request: ...
```

## Printing HTTP Cache

This method is a little more complex, as it requires having to understand how information is stored inside the HTTP Cache. I do not recommend using this tool often, but it can help provider some insights. Since the HTTP cache is just stored inside of a file directory, we have the ability to loop through that directory and print whatever is inside:

```kotlin
private fun printHttpCache() {
    val cacheDirectory = File(applicationContext.cacheDir, "apolloCache")
    cacheDirectory.listFiles()?.forEach { file ->
        Log.d("ApolloHttpCache", "HttpCacheFile: ${file.name}")
        Log.d("ApolloHttpCache", "HttpCacheFile: ${file.readText()}")
        Log.d("ApolloHttpCache", "-----")
    }
}
```

Using this, we can get a quick overview of what is inside our cache. Some files are quite large, but we can highlight one file where we can see that a CountryDetailQuery response was stored in:

```
D/ApolloHttpCache: HttpCacheFile: 61b4ca292f9e869f711db7612be3e67e.1
D/ApolloHttpCache: HttpCacheFile: {"data":{"country":{"__typename":"Country","code":"AU","name":"Australia","continent":{"__typename":"Continent","code":"OC","name":"Oceania"},"capital":"Canberra","emoji":"🇦🇺"}}}
D/ApolloHttpCache: -----
```

The file name, `61b4ca292f9e869f711db7612be3e67e.1`, is a reference to the cache key that Apollo generates for that request. The next time I try to search for Australia, Apollo looks to see if a file with that key exists already. 

Using the file directory itself is complicated, but it can provide some visual clues as to what is or is not inside our cache, if necessary. 

# Limitations

Way back in the beginning of this post, we discussed that the HTTP cache was quick to set up but had some limitations. Here is a list of some limitations that might impact your app.

## Stores Duplicate Data

If multiple operations request similar information, it is stored twice. For example, our CountryListQuery and CountryDetailQuery each request the same information. This is stored in two files - one for the list response, and one for the detail response. We have no way to link these two files together to avoid requesting information a second time. 

## Cannot Observe Changes

As a result of using the file system, there is no easy way for us to observe changes to the HTTP cache. If we delete or modify an existing entry, any screens that requested data are now out of date and need to make new requests to be in sync again. 

## Does Not Work With HTTP Mutations

The HTTP cache does not support caching of GraphQL mutations, only query requests.

# Recap

The Apollo HTTP cache is a great built in tool from Apollo that helps us store data to create better user experiences. Leveraging this cache decreases loading times and cuts down on network requests. It's quick to set up, but does have some potential limitations. If those limitations are fine in your projects, give it a shot! If you need support for observing changes and relating data in the cache, check out our next post on the [normalized cache](https://www.apollographql.com/docs/android/essentials/normalized-cache/). 
