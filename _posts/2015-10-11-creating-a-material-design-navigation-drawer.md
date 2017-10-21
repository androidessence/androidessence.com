---
layout: post
author: adam
title: Creating A Material Design Navigation Drawer
description: An understanding of the Navigation Activity template from Android Studio.
modified: 2015-10-11
published: true
tags: [material design, navigation drawer]
categories: [android]
---

For those of you who haven’t upgraded to Android Studio 1.4 yet, you may not have realized there is a new Navigation Drawer template in the New Project menu. While templates are great, they don’t always give you the full story. Modifying the Navigation View can be tricky if you don’t know where to look, and you may not understand why certain things are written as they are. This post is going to walk you through the process of creating your own Navigation View, and discussing some of the differences between this and the previous model.

<!--more-->

# The XML

The first thing that’s changed is that we no longer need a fragment object to build our Navigation Drawer in the XML. We can use a [NavigationView](https://developer.android.com/reference/android/support/design/widget/NavigationView.html), which according to docs:

> Represents a standard navigation menu for application. The menu contents can be populated by a menu resource file.

```xml
	<?xml version="1.0" encoding="utf-8"?>
	<android.support.v4.widget.DrawerLayout
	    xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:app="http://schemas.android.com/apk/res-auto"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:id="@+id/drawer_layout"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:fitsSystemWindows="true"
	    tools:openDrawer="start">
	 
	    <include
	        layout="@layout/app_bar_main"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"/>
	 
	    <android.support.design.widget.NavigationView
	        android:id="@+id/nav_view"
	        android:layout_width="wrap_content"
	        android:layout_height="match_parent"
	        android:layout_gravity="start"
	        android:fitsSystemWindows="true"
	        app:headerLayout="@layout/nav_header_main"
	        app:menu="@menu/activity_main_drawer"/>
	 
	</android.support.v4.widget.DrawerLayout>
```

The NavigationView should still belong inside a DrawerLayout, as we did previously. There are two attributes that may be new to you: headerLayout and menu. These attributes represent the green portion of the image above, and the menu below it, respectively. We’ll look at each of those in more depth next.

# The headerLayout

The header layout is rather simple; It’s just a LinearLayout. However, it does have a few attributes to keep it up to spec. These include, but are not limited to:

* 16dp padding
* 160dp height
* 16dp padding above the ImageView
* 16dp padding between the top TextView and the ImageView

The XML looks like this:

```xml
	<?xml version="1.0" encoding="utf-8"?>
	<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
		android:layout_width="match_parent"
		android:layout_height="@dimen/nav_header_height"
		android:background="@drawable/side_nav_bar"
		android:paddingBottom="@dimen/activity_vertical_margin"
		android:paddingLeft="@dimen/activity_horizontal_margin"
		android:paddingRight="@dimen/activity_horizontal_margin"
		android:paddingTop="@dimen/activity_vertical_margin"
		android:theme="@style/ThemeOverlay.AppCompat.Dark"
		android:orientation="vertical"
		android:gravity="bottom">
	 
	    <ImageView
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:paddingTop="@dimen/nav_header_vertical_spacing"
	        android:src="@android:drawable/sym_def_app_icon"
	        android:id="@+id/imageView"/>
	 
	    <TextView
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:paddingTop="@dimen/nav_header_vertical_spacing"
	        android:text="Android Studio"
	        android:textAppearance="@style/TextAppearance.AppCompat.Body1"/>
	 
	    <TextView
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:text="android.studio@android.com"
	        android:id="@+id/textView"/>
	 
	</LinearLayout>
```

The background to the view is a gradient that was left out for simplicity.

# The menu

There are two cool ways we can handle menus inside the NavigationDrawer. The first is to add a <group> tag that includes menu items and their icons. This is to handle the first four items from import to tools:

```xml
	<?xml version="1.0" encoding="utf-8"?>
	<menu xmlns:android="http://schemas.android.com/apk/res/android">
	 
	    <group android:checkableBehavior="single">
	        <item
	            android:id="@+id/nav_camara"
	            android:icon="@android:drawable/ic_menu_camera"
	            android:title="Import"/>
	        <item
	            android:id="@+id/nav_gallery"
	            android:icon="@android:drawable/ic_menu_gallery"
	            android:title="Gallery"/>
	        <item
	            android:id="@+id/nav_slideshow"
	            android:icon="@android:drawable/ic_menu_slideshow"
	            android:title="Slideshow"/>
	        <item
	            android:id="@+id/nav_manage"
	            android:icon="@android:drawable/ic_menu_manage"
	            android:title="Tools"/>
	    </group>
	 
	    <item android:title="Communicate">
	        <menu>
	            <item
	                android:id="@+id/nav_share"
	                android:icon="@android:drawable/ic_menu_share"
	                android:title="Share"/>
	            <item
	                android:id="@+id/nav_send"
	                android:icon="@android:drawable/ic_menu_send"
	                android:title="Send"/>
	        </menu>
	    </item>
	 
	</menu>
```

The second option is to use <item> with a menu inside. That is how we get the label ‘Communicate’ at the bottom of the Navigation View and separate these items from the rest.

# The MainActivity

A personal favorite about the new NavigationView is, as I mentioned earlier, the lack of need for a fragment class. The XML above is all we need to build our NavigationView, and we can interact with it as necessary inside our Activity. All we need to do is implement OnNavigationItemSelectedListener:

```java
	public class MainActivity extends AppCompatActivity
	        implements NavigationView.OnNavigationItemSelectedListener {
	}
```

This interface only requires one implementation to handle the navigation item that was selected:

```java
	@SuppressWarnings("StatementWithEmptyBody")
	@Override
	public boolean onNavigationItemSelected(MenuItem item) {
	    // Handle navigation view item clicks here.
	    int id = item.getItemId();
	 
	    if (id == R.id.nav_camara) {
	        // Handle the camera action
	    } else if (id == R.id.nav_gallery) {
	 
	    } else if (id == R.id.nav_slideshow) {
	 
	    } else if (id == R.id.nav_manage) {
	 
	    } else if (id == R.id.nav_share) {
	 
	    } else if (id == R.id.nav_send) {
	 
	    }
	 
	    DrawerLayout drawer = (DrawerLayout) findViewById(R.id.drawer_layout);
	    drawer.closeDrawer(GravityCompat.START);
	    return true;
	}
```

The last thing you need to do is set the interface on your NavigationView inside onCreate():

```java
	NavigationView navigationView = (NavigationView) findViewById(R.id.nav_view);
	navigationView.setNavigationItemSelectedListener(this);
```

That’s it! You’ve now successfully created a Material Design NavigationView! As you go on to build your own and customize them, make sure to check with the [design guidelines](https://www.google.com/design/spec/patterns/navigation-drawer.html) for proper specifications.

# Additional Notes

Because items in the navigation header belong to the NavigationView, this also means they belong to the Activity. Therefore, nothing fancy is required to access these elements, you can do it right inside the activity by calling findViewById(R.id.imageView). This is the ImageView item inside nav_header_main.xml.

No code was added to Github for this post as all code was sampled using the template from AndroidStudio 1.4.