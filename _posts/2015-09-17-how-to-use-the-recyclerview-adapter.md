---
layout: post
author: adam
title: How To Use The RecyclerView Adapter
description: This post explains how the RecyclerView.Adapter class is implemented.
modified: 2015-09-17
published: true
tags: [recyclerview, adapter]
categories: [android]
---

The [RecyclerView.Adapter](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.Adapter.html) class is used to bind a dataset to a RecyclerView to be displayed to a user. As I explained in another post, [RecyclerView Vs ListView]({{ site.baseurl }}{% link _posts/2015-09-07-recyclerview-vs-listview.md %}), the RecyclerView.Adapter forces the use of the ViewHolder pattern, which may be hard to understand when switching to a RecyclerView from a ListView. In this short post I am going to reference my MovieAdapter class from my [Swipe-To-Dismiss]({{ site.baseurl }}{% link _posts/2015-09-09-swipe-to-dismiss-recyclerview-items.md %}) example, and break it down to explain the required implementations and how to use the RecyclerView adapter.

<!--more-->

# The Adapter Class

Let's first take a look at the class header for the adapter:

```java
	public class MovieAdapter extends RecyclerView.Adapter<MovieAdapter.ViewHolder> { }
```

The class extends from RecyclerView.Adapter, which should be the obvious part here. However, you'll notice that the class requires a generic type that must extend from [RecyclerView.ViewHolder](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.ViewHolder.html).

# The RecyclerView.ViewHolder

The ViewHolder is a class that represents layout information for an item in the RecyclerView. The use of the ViewHolder is so that a new layout isn't inflated for each and every item, but the same layout is jsut reused and the attributes of an element (such as the text in a TextView) are repopulated for each item. This is a huge performance helper and enables for much smoother scrolling. Let's take a look at our ViewHolder:

```java
	public class ViewHolder extends RecyclerView.ViewHolder{
	   public final TextView movieNameTextView;
	 
	   public ViewHolder(View view){
	      super(view);
	      movieNameTextView = (TextView) view.findViewById(R.id.movie_name);
	   }
	 
	   public void bindMovie(Movie movie){
	      this.movieNameTextView.setText(movie.getName());
	   }
	}
```

The class, as I mentioned earlier, extends from RecyclerView.ViewHolder. In this case, we only have one element for each item that must be populated: the movie's name. Inside this class, you put all UI elements that will be manipulated at the class level, and you instantiate them inside of your constructor (which takes a View parameter).

I have added the `bindMovie()` method here as an option to make my program more readable. This method takes a Movie object, and populates all of the necessary UI elements for that movie. If you don't want to implement this method, you can handle this in `onBindViewHolder` explained later.

# The constructor

In a basic adapter, you will need one class level variable, which is your data set. In this example, we'll use a `List<>` object. You can use a simple constructor to instantiate these fields:

```java
	private List<Movie> movies;

	public MovieAdapter(List<Movie> movies) {
		this.movies = movies;
	}
```

# getItemCount()

Of the three methods required for implementation, `getItemCount()` is perhaps the easiest. All you have to do is return the number of items in your RecyclerView, which in this case is just the size of our data set:

```java
	@Override
	public int getItemCount() {
		return movies.size();
	}
```

If you used an array, you can just change this line to `return movies.length;`.

# onCreateViewHolder

When the Adapter creates its first item, onCreateViewHolder is called. This is what allows the Adapter to reuse a reference to a view instead of re-inflating it as I mentioned earlier. Typically this implementation will just inflate a view and return a ViewHolder object:

```java
	@Override
	public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
		Context context = parent.getContext();
		View view = LayoutInflater.from(context).inflate(R.layout.list_item_movie, parent, false);
		return new ViewHolder(view);
	}
```

You'll notice that there is a second parameter for viewType. This parameter is used if you are displaying different types of items inside your list, meaning you'll have to create a second ViewHolder and use this parameter to determine which of them to return.

# onBindViewHolder

The onBindViewHolder method is where the magic happens. This is where you get an item from your data set and bind it to the list item in your adapter, like this:

```java
	@Override
	public void onBindViewHolder(ViewHolder holder, int position) {
		Movie movie = movies.get(position);
		holder.bindMovie(movie);
	}
```

The position parameter represents the position of the item being bound at the time. You use this parameter to get the proper item from your data set. The holder is the ViewHolder your item will be bound to. So, in this method you can either call `holder.movieNameTextView.setText(movie.getName())` and all other declarations, or you can make use of the bindMovie method like I did.

Once you've added the ViewHolder and implemented these three required methods, you know how to use the RecyclerView adapter! This is a very powerful tool that will make your displays much more flexible down the line.