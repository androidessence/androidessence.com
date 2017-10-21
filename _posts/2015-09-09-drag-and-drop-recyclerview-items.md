---
layout: post
author: adam
title: Drag And Drop RecyclerView Items
description: Tutorial on how to use the drag and drop feature on a RecyclerView.
modified: 2015-09-09
published: true
tags: [recyclerview, drag and drop]
categories: [android]
---

Before continuing this post, I recommend that you read my previous one on [Swipe To Dismiss RecyclerView Items]({{ site.baseurl }}{% link _posts/2015-09-09-swipe-to-dismiss-recyclerview-items.md %}) as this will build upon the `ItemTouchHelper` class discussed there. Once you've done that, come back to this short post and learn how to drag and drop RecyclerView items like this:

![Drag And Drop RecyclerView](/images/drag-and-drop.gif)

<!--more-->

# Swap Adapter Items

Before we can properly use the drag and drop feature, we need to add another method to our Adapter that swaps two items. You can use `Collections.swap() to do this easily on a List object:

```java
	public void swap(int firstPosition, int secondPosition) {
		Collections.swap(movies, firstPosition, secondPosition);
		notifyItemMoved(firstPosition, secondPosition);
	}
```

# Implement onMove

In our previous lesson, you'll recall we left our implementation of `onMove` blank. To make us of it, all we have to do is make sure it swaps our Adapter items:

```java
	@Override
	public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
	   mMovieAdapter.swap(viewHolder.getAdapterPosition(), target.getAdapterPosition());
	   return true;
	}
```

Once you've done those two things, in addition to all the necessary steps for swipe-to-dismiss, you're done. If you chose to only use the drag and drop feature, you can delet the `remove()` function from the adapter and the implementation of `onSwap()` inside the `ItemTouchHelper` class.

I hope this simple tutorial helps you advance your RecyclerView skills. I have updated the MovieList project on [GitHub](https://github.com/androidessence/SwipeToDismissSample) to include the code from this example.