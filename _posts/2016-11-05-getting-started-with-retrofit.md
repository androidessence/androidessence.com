---
layout: post
author: adam
modified: 2018-10-13
title: Getting Started With Retrofit In Android
published: true
tags: [retrofit, networking]
description: Retrofit is an HTTP client for networking in Android.
categories: Android
---

As a new or intermediate Android developer, getting started with [Retrofit](http://square.github.io/retrofit), a popular HTTP client, can seem pretty daunting. However, using this library is not as scary as it sounds. Here are four easy steps to creating your first Retrofit application!

<!--more-->

# Gradle Dependencies

First things first, include the following dependencies in your build.gradle file. These are for the retrofit library, as well as a GSON converter which is responsible for converting the JSON response from the server into your plain old Java object (POJO).

```groovy
    implementation 'com.squareup.retrofit2:retrofit:2.4.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.4.0'
```

# Define Your Model(s)

Next, we need to define the models that will be returned from the server. In this example, we will be returning a list of people that are presidents. Each person is represented in the following way:

```java
    public class Person {
    	private String id;
    	private String firstName;
    	private String lastName;
    }
```

It's important to note that the variable names match up with the JSON representation:

```json
    {
    	"id": 1,
    	"firstName": "George",
    	"lastName": "Washington"
    }
```

If you don't want to use the same variable names as the JSON keys, you can use the `SerializedName` annotation:

```java
    public class Person {
    	private String id;

    	@SerializedName("firstName")
    	private String first;

    	@SerializedName("lastName")
    	private String last;
    }
```

Since our project will be taking back a list of presidents, not just a single person, we should have a POJO for that as well:

```java
    public class Presidents {
    	public List<Person> presidents;
    }
```

Which has the corresponding JSON response:

```json
    {
    	"presidents": [
    		...
    	]
    }
```

# Create A Service Interface

Next, we need to create an interface that is responsible for communicating with our API, which will look like this:

```java
    public interface PersonService {
     
        @GET("people/presidents")
        Call<Presidents> getPresidents();
     
        @GET("people/{id}")
        Call<Person> getPerson(@Path("id") String id);
     
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("https://raw.githubusercontent.com/androidessence/RetrofitSample/master/")
                .addConverterFactory(GsonConverterFactory.create())
                .build();
    }
```

At the bottom we create a Retrofit object which has the base URL of our API, and add the GSOn converter mentioned in the beginning. There is also a method for each call we can make to our API. In the first one we have a call that returns a presidents object, and the annotation is what defines that this is a GET request, and provides the additional path beyond the base url.

**Note:** `Call` is a common class name. Depending on your dependencies (sorry), you may need to be careful about what class you import here. Make sure it is `retrofit2.Call`.

# Implement The Interface

Now that we've defined our model, and our API calls, we can implement them in code in roughly three steps:

1. Instantiate our Retrofit service.
2. Instantiate the call we want to make.
3. Implement a callback for that call to be handled when the HTTP request is completed.

Here is what the code for that looks like. Note that a Retrofit callback has two methods: `onSuccess()` and `onFailure()`. In this case we just log the failure, but replace our ListView adapter in onSuccess:

```java
    private void getPresidents() {
        PersonService service = PersonService.retrofit.create(PersonService.class);
     
        Call<Presidents> call = service.getPresidents();
     
        call.enqueue(new Callback<Presidents>() {
            @Override
            public void onResponse(Call<Presidents> call, Response<Presidents> response) {
                Log.v(LOG_TAG, "Received response: " + response.toString());
                adapter.swapPresidents(response.body());
            }
     
            @Override
            public void onFailure(Call<Presidents> call, Throwable t) {
                Log.v(LOG_TAG, t.getMessage());
            }
        });
    }
```

**Note:** `Call` is a common class name. Depending on your dependencies (sorry), you may need to be careful about what class you import here. Make sure it is `retrofit2.Call`.

And that's it! Just like that you are getting started with Retrofit. The full sample code for this project can be found on [GitHub](https://github.com/androidessence/RetrofitSample).