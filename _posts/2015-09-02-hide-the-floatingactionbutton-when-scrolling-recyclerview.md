---
layout: post
author: adam
title: Hide The FloatingActionButton When Scrolling A RecyclerView
description: Tutorial on how to show and hide a FloatingActionButton when scrolling a RecyclerView.
modified: 2015-09-02
published: true
tags: [recyclerview, floatingactionbutton]
categories: [android]
---

One of my favorite components introduced with Material Design is the [FloatingActionButton](https://www.google.com/design/spec/components/buttons-floating-action-button.html) (FAB). These buttons are great for emphasizing the primary action of an Activity, but quickly become a nuisance when displayed over a RecyclerView as they may block the bottom list item. To avoid this, we can hide the FloatingActionButton when scrolling a RecyclerView.

In today’s tutorial we will only be hiding the FAB when the RecyclerView is scrolled upward, and it will reappear on the next down scroll as shown here:

![FloatingActionButton](/images/fab_scroll.gif)

<!--more-->

# Add the support library dependency

As with any support library component, like the Toolbar in my last post, we need to add a dependency to our build.gradle file for the design library, and for the RecyclerView:

```groovy
	dependencies{
	   compile 'com.android.support:design:23.0.0'
	   compile 'com.android.support:recyclerview-v7:23:0:0'
	}
```

# Add the FAB and RecyclerView inside a CoordinatorLayout

In order to achieve this scroll animation, we must put the Views inside of a CoordinatorLayout. As the documentation states, a primary use of the [CoordinatorLayout](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.html) is:

> As a container for a specific interaction with one or more child views

Since the FAB is interacting with the RecyclerView (by hiding/showing depending on scroll) the CoordinatorLayout will help us with this process. Here is how the XML will look:

```xml
	<android.support.design.widget.CoordinatorLayout 
	   xmlns:android="http://schemas.android.com/apk/res/android" 
	   xmlns:app="http://schemas.android.com/apk/res-auto" 
	   android:layout_width="match_parent" 
	   android:layout_height="match_parent">  
	 
	   <android.support.v7.widget.RecyclerView 
	      android:id="@+id/recycler_view" 
	      android:layout_width="match_parent" 
	      android:layout_height="match_parent"/>  
	 
	   <android.support.design.widget.FloatingActionButton
	      android:id="@+id/fab"
	      android:layout_width="wrap_content" 
	      android:layout_height="wrap_content" 
	      android:layout_margin="16dp" 
	      app:layout_anchor="@+id/recycler_view" 
	      app:layout_anchorGravity="bottom|end" />
	 
	</android.support.design.widget.CoordinatorLayout>
```

Two attributes that may be new to you are `layout_anchor` and `layout_anchorGravity`. These attributes were [introduced with the CoordinatorLayout](http://android-developers.blogspot.com/2015/05/android-design-support-library.html) to place some views relative to others.

# Implement the FAB’s behavior

If you took a peak at the CoordinatorLayout documentation, you will notice the mentioned something about Behavior:

> By specifying [Behaviors](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html) for child views of a CoordinatorLayout you can provide many different interactions within a single parent and those views can also interact with one another.

There is a `FloatingActionButton.Behavior` class that we can extend to implement our own behavior. Start by creating the following class:

```java
	public class FABScrollBehavior extends FloatingActionButton.Behavior { }
```

In order for our FAB to react to the scroll of the RecyclerView, we can override `onStartNestedScroll()` and `onNestedScroll()`:

```java
	public class FABScrollBehavior extends FloatingActionButton.Behavior {  
	   @Override  
	   public void onNestedScroll(CoordinatorLayout coordinatorLayout, FloatingActionButton child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) { 
	      super.onNestedScroll(coordinatorLayout, child, target, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed); 
	      if(dyConsumed > 0 && child.getVisibility() == View.VISIBLE){ 
	         child.hide(); 
	      } else if(dyConsumed < 0 && child.getVisibility() == View.GONE){ 
	         child.show(); 
	      }  
	   }  
	 
	   @Override 
	   public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, FloatingActionButton child, View directTargetChild, View target, int nestedScrollAxes) { 
	      return nestedScrollAxes == ViewCompat.SCROLL_AXIS_VERTICAL; 
	   }
	}
```

This implementation is pretty straight forward. If the scroll is upward (dyConsumed > 0) and the child (the FAB here) is visible, hide it. If the opposite occurs, show it. Thankfully, we do not need to write any animations ourself! The FAB class contains methods for [showing](https://developer.android.com/reference/android/support/design/widget/FloatingActionButton.html#show()) and [hiding](https://developer.android.com/reference/android/support/design/widget/FloatingActionButton.html#hide()) the button.

We’ve also implemented onStartNestedScroll to show that we are only handling vertical scrolls.

# Attach the behavior to the FAB

Next we need to make sure the FAB behaves like we just implemented. This can be done using XML:

```xml
	<android.support.design.widget.FloatingActionButton 
	   android:layout_width="wrap_content" 
	   android:layout_height="wrap_content" 
	   android:layout_margin="16dp" 
	   app:layout_anchor="@+id/recycler_view" 
	   app:layout_anchorGravity="bottom|end" 
	   app:layout_behavior="com.adammcneilly.fabscrollsample.FABScrollBehavior"/>
```

While writing this tutorial for you, I learned the hard way that this is not enough. As it is you will get the following error:

> Caused by: java.lang.RuntimeException: Could not inflate Behavior subclass com.adammcneilly.fabscrollsample.FABScrollBehavior

To fix this, just make sure you’ve added a default constructor to your behavior class:

```java
	public FABScrollBehavior(Context context, AttributeSet attributeSet){
	   super();
	}
```

Once you’ve done that, you’ll be able to hide the FloatingActionButton when scrolling the RecyclerView. Great work!

The code for this tutorial can be found on [GitHub](https://github.com/androidessence/FABScrollTutorial).