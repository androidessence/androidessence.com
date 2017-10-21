---
layout: post
author: adam
title: How To Make An Android UI Part 2&#58; Java Code
modified: 2015-12-02
published: true
tags: [ui]
categories: [android]
---

After reading [part one]({{ site.baseurl }}{% link _posts/2015-11-08-how-to-make-an-android-ui-part-1.md %}) which discusses what Views and ViewGroups are, as well as how to create them in XML, the next step is to incorporate them into your Android application. How you use the UI you wrote depends on what it’s used for. Let’s break down some of the key things:

<!--more-->

# Activities

Activities are a specific focus for the user. Nearly every Activity you create will display a window for the user, which is defined using the [setContentView()](http://developer.android.com/reference/android/app/Activity.html#setContentView(int)) function. There are a few overloads for this method, the most popular being one that takes an integer resource to your xml file. That is why you will often see the following code in a default template:

```java
	@Override
	protected void onCreate(Bundle savedInstanceState) {
	    super.onCreate(savedInstanceState);
	    setContentView(R.layout.activity_main);
	}
```

This call to `setContentView()` is required in order to see anything displayed in the Activity, and it must also be called before any interaction with the UI elements in the Activity, which we’ll talk about later.  Next, let’s discuss how Fragments get their layouts.

# Fragments

A fragment is just a part of your app’s UI that can be placed within an Activity. Fragments do not use the `setContentView()` method, but instead implement [onCreateView()](http://developer.android.com/intl/pt-br/reference/android/app/Fragment.html#onCreateView(android.view.LayoutInflater, android.view.ViewGroup, android.os.Bundle)) to initialize a Fragment’s User Interface. A [LayoutInflater](http://developer.android.com/reference/android/view/LayoutInflater.html) object is used to inflate the layout:

```java
	public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
	    View view = inflater.inflate(R.layout.fragment_accounts, container, false);
	    // Do actions with view if necessary
	    return view;
	}
```

Again, we’ll come back to what kinds of actions you can preform on Views a little later. For now, let’s try to really understand what’s happening behind the scenes in a LayoutInflater.

# LayoutInflater

The concept is rather simple, but important to understand what is going on. A LayoutInflater instantiates a layout XML file into its corresponding View objects. In other words, when you have your LinearLayout and children defined in XML, the inflater will create actually View objects for you that you can manipulate in code. This is your gateway into everything you will need to create a positive user experience.

# Accessing Views

If you ever need to manipulate view, whether you’re setting text, reading input, or handling clicks programatically, you’ll need to create an object in your code. To access that object, you can use the [View.findViewbyId()](http://developer.android.com/reference/android/view/View.html#findViewById(int)) method. There also exists an Activity.findViewById() method, which means you can just call the method directly inside an Activity:

```java
	// Inside activity
	protected void onCreate(Bundle savedInstanceState) {
	    super.onCreate(savedInstanceState);
	    setContentView(R.layout.activity_main);
	 
	    TextView myTextView = (TextView) findViewById(R.id.my_text_view);
	}
	 
	// Inside fragment
	public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
	    View view = inflater.inflate(R.layout.fragment_main, container, false);
	    TextView myTextView = (TextView) view.findViewById(R.id.my_text_view);
	    return view;
	}
```

Once you’ve learned how to create your XML files, inflate the views, and access specific elements, you have the base foundation required to create beautiful interactive interfaces. Unfortunately, there are too many different things you can do for me to discuss them all here, so I highly encourage you to look at the docs to understand the various attributes of UI elements and learn the different ways to style them and manipulate them. I hope this gave you a better understand of the User Interface.