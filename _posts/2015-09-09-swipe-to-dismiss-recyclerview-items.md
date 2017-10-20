---
layout: post
author: adam
title: Swipe To Dismiss RecyclerView Items
description: A tutorial on using the swipe-to-dismiss feature to remove items from a list in Android.
modified: 2015-09-09
published: true
tags: [recyclerview, swipe to dismiss]
categories: [android]
---

In my last post I broke down the differences between the RecyclerView and a ListView. One of the benefits of the RecyclerView that I touched on was the [ItemTouchHelper](https://developer.android.com/reference/android/support/v7/widget/helper/ItemTouchHelper.html). This class is used to handle the Swipe-To-Dismiss and Drag-N-Drop behaviors of a RecyclerView. In this post I am going to teach you how to swipe to dismiss RecyclerView items using the sample application seen here:

![Swipe To Dismiss](/images/std-sample.gif)

<!--more-->

I am going to move forward with the assumption that you are familiar with a RecyclerView and have already added one into your application. If you haven't, you can check out [this link](https://developer.android.com/training/material/lists-cards.html). The sample list for this application will show a number of `Movie` objects, that have a single field for `movieName`.

# The Adapter

We will not change the adapter much to allow for swiping, but we should add the following method along with all of the other required implementations:

```java
	public void remove(int position) {
		movies.remove(position);
		notifyItemRemoved(position);
	}
```

This method will remove an item from our movie list, and notifies the adapter that an item has been removed. By notifying the adapter, we are not rebinding the data but simply altering the positions, as explained in the [documentation](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.Adapter.html#notifyItemRemoved(int)):

> This is a structural change event. Representations of other existing items in the data set are still considered up to date and will not be rebound, though their positions may be altered.

If you would like to see the entire Adapter class, please see [this file on GitHub](https://github.com/androidessence/SwipeToDismissSample/blob/master/app/src/main/java/androidessence/movielist/MovieAdapter.java).

# ItemTouchHelper.SimpleCallback

To handle the swiping we will create an [ItemTouchHelper.SimpleCallback](https://developer.android.com/reference/android/support/v7/widget/helper/ItemTouchHelper.SimpleCallback.html) class. This is a simple wrapper class using drag and swipe directions to handle those events. In this tutorial, we are only concerned with swiping. Here is our callback:

```java
	public class MovieTouchHelper extends ItemTouchHelper.SimpleCallback {
	   private MovieAdapter mMovieAdapter;  
	 
	   public MovieTouchHelper(MovieAdapter movieAdapter){ 
	      super(ItemTouchHelper.UP | ItemTouchHelper.DOWN, ItemTouchHelper.LEFT | ItemTouchHelper.RIGHT); 
	      this.mMovieAdapter = movieAdapter; 
	   }  
	 
	   @Override 
	   public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {  
	      //TODO: Not implemented here
	      return false;  
	   } 
	  
	   @Override 
	   public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) { 
	      //Remove item
	      mMovieAdapter.remove(viewHolder.getAdapterPosition()); 
	   }
	}
```

The required default constructor for the SimpleCallback class takes two parameters. The first is for the drag directions, the second for swiping directions. The two methods are required implementations. However, we are not implementing Drag-N-Drop for this example so we can leave the method empty. Inside the `onSwiped` method we remove the swiped item from the Adapter using the `remove()` method we added earlier.

# Add ItemTouchHelper to RecyclerView

Now that we’ve created our ItemTouchHelper class, we can easily attach it to our RecyclerView in just three lines of code. This code is found inside `onCreate()` of the activity:

```java
	ItemTouchHelper.Callback callback = new MovieTouchHelper(movieAdapter);
	ItemTouchHelper helper = new ItemTouchHelper(callback);
	helper.attachToRecyclerView(movieRecyclerView);
```

Now we’re done! You don’t have to add any animations, all of those are handled behind the scenes. I hope you enjoyed learning how to swipe to dismiss RecyclerView items. The full code for this sample application can be found on my [GitHub](https://github.com/androidessence/SwipeToDismissSample) page.