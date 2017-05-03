---
layout: post
title: RecyclerView VS ListView
description: Understanding what the difference is between a RecyclerView and a ListView.
modified: 2015-09-07
published: true
tags: [recyclerview, listview]
categories: [android]
---

Introduced in API 21 (Android 5.0), along with other MaterialDesign components, was the [RecyclerView](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.html) widget. This widget is a more flexible version of the [ListView](http://developer.android.com/reference/android/widget/ListView.html), which to many developers was their go-to [AdapterView](http://developer.android.com/reference/android/widget/AdapterView.html) for applications.

It is important to note here, though, that the RecyclerView does not extend from the AdapterView class. Intuitively, it sounds like it would as an AdapterView is defined in the docs as:

> An AdapterView is a view whose children are determined by an [Adapter](http://developer.android.com/reference/android/widget/Adapter.html).

In the AdapterView design, the Adapter is the component responsible for mapping content from the data source to the Views of an AdapterView which are displayed to the user. Through the use of Adapters and LayoutManagers, the RecyclerView is able to break up responsibilities of the layout and create opportunities for more unique designs, which are explained in further detail later. Before we discuss the opportunities given to us by the RecyclerView, let’s go over some of the benefits of the ListView (in comparison to the RecyclerView), and where it fell short.

<!--more-->

# ListView Benefits

1. OnItemClickListeners

If you’ve ever used an AdapterView such as the ListView, you’ve probably needed to handle the event when the user clicks on an item. This was trivial with the ListView:

```java
	mListView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
	   @Override 
	   public void onItemClick(AdapterView<?> parent, View view, int position, long id) {  
	      //TODO:
	   } 
	});
```

However, the RecyclerView does not have nested classes like OnItemClickListener, so handling these events is a little harder. I will give an example later on.

2. CursorAdapters

I am only speculating, but I expect there are many times when developers want to use a ListView or a RecyclerView to display information from a database. This can be done rather simply with a [CursorAdapter](http://developer.android.com/reference/android/widget/CursorAdapter.html), an Adapter class designed to expose data from a Cursor (an object to represent database data) to a ListView. Used with a LoaderManager this class makes displaying database data very easy. Currently, there is no CursorAdapter class used for the RecyclerView, though SO user [skyfishjy](http://stackoverflow.com/users/1833505/skyfishjy) has an awesome class on his [GitHub](https://gist.github.com/skyfishjy/443b7448f59be978bc59).

3. Dividers

With the ListView, you could set a [divider](http://developer.android.com/reference/android/widget/ListView.html#attr_android:divider) drawable to appear between each item, which can be as simple as a thin line to represent a separation. This could be done easily in XML for the ListViews:

```xml
	<ListView
	   android:layout_width="match_parent"
	   android:layout_height="match_parent"
	   android:divider="#FFFFFF"
	   android:dividerHeight="1dp"/>
```

However, with the RecyclerView you need to implement an ItemDecorator to achieve this effect.

The benefits of the ListView are not listed to only these three items, but they are three things that I believe could be achieved much easier using a ListView.

# Pitfalls of the AdapterView

Despite the benefits above which provide easy implementation for some basic requirements, the AdapterView did bring some limitations to developers. One of the more important issues being that the AdapterView was in charge of handling the items’ animations, touch interactions, and decorations, it became much harder to scale and expand the behavior of the AdapterView. For an example of this, see the section below about handling clicks in the RecyclerView. As mentioned previously, the AdapterView also lacked the requirement for the ViewHolder pattern, which is great for making lists scroll smoother. By making this optional, developers may create lists that don’t scroll as smoothly as they could, and can lead to misunderstandings regarding the AdapterView because it may be unclear when views are being created or recycled in the list.

Having discussed the pros and cons of the AdapterViews, let’s take a look at how the RecyclerView improved upon or hindered those behaviors.

# RecyclerView benefits

1. More explicit click listeners

As explained in the [The Busy Coder’s Guide to Android Development](https://commonsware.com/Android/), the RecyclerView, for the most part, detaches itself from being in control of input other than scrolling. The individual views inside of it are responsible for handling click listeners. Mark Murphy explains the benefits very well:

> Clickable widgets, like a RatingBar, in a ListView row had long been in conflict with click events on rows themselves. Getting rows that can be clicked, with row contents that can also be clicked, gets a bit tricky at times. With RecyclerView, you are in more explicit control over how this sort of thing gets handled… because you are the one setting up all of the on-click handling logic.

You are probably asking yourself how you are supposed to handle all of the on-click logic. Well, the best place to handle interactions with a particular View would be inside of your ViewHolder. For example, if you just want to handle the click of a particular RecyclerView item, just have the ViewHolder implement `View.OnClickListener` and set it to the view itself, like this:

```java
	public class MyViewHolder extends RecyclerView.ViewHolder 
	      implements View.OnClickListener{
	   public final TextView textView; 
	 
	   public MyViewHolder(View view){
	      textView = (TextView) view.findViewById(R.id.text_view);
	      view.setOnClickListener(this);
	   } 
	 
	   @Override
	   public void onClick(View v){
	      // Clicked on item 
	      Toast.makeText(mContext, "Clicked on position: " + getAdapterPosition(), Toast.LENGTH_SHORT).show();
	   }
	}
```

2. LayoutManagers

Unlike the ListView where the design of the layout was pre-set, like the vertical scrolling only of a ListView or the two-dimensional scrolling [GridView](http://developer.android.com/reference/android/widget/GridView.html), the RecyclerView is not responsible for handling how the data is displayed to the user. Instead, that is handled by the [LayoutManager](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.LayoutManager.html) which can be used for vertical scrolling, horizontal scrolling, staggered grids, and much more.

3. Item animators and item decorators

As mentioned above, the ListView itself was responsible for adding dividers between views. However, developers may want to freedom to create something more than a simple divider. For that, we can implement our own [ItemDecoration](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.ItemDecoration.html) class for all sorts of boundaries on items.

For a look at some of the other benefits brought by the RecyclerView, please review the [documentation](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.html) again and take a look at all of the nested classes that are available.

That was my breakdown of the RecyclerView vs ListView and their differences. I really look forward to the comments on this post, as there is so much more to the topic than is seen here.