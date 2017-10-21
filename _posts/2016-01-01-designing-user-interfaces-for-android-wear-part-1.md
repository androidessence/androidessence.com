---
layout: post
author: adam
modified: 2016-01-01
title: Designing User Interfaces For Android Wear Part 1&#58; Lists
tags: [android wear]
categories: android
published: true
---

Rising in popularity after the latest Google I/O, [Android Wear](https://www.android.com/wear/) is a game changer for mobile development. Wearable technology has brought benefits from a quicker access to information to a more accurate monitoring of our physical health. Developing for this platform allows you to tap into those features that are not as readily available on mobile handhelds as well as offer a more immersive experience of your product by making it available on wearable devices.

As always, the [documentation](http://developer.android.com/intl/pt-br/wear/index.html) will offer the most thorough insight into what is available, but I’d like to discuss how the UI development differs and how you can get started.

<!--more-->

# WatchViewStub

The first class you should learn about from the Wear library is [WatchViewStub](http://developer.android.com/intl/pt-br/reference/android/support/wearable/view/WatchViewStub.html). This class is responsible for determining the screen shape of a Wear device. Currently there are two (or three) shapes of Wearable devices: square and round, with some round devices having an inset at the bottom. Assuming you’d like your Activity to appear differently for each device, your `activity_main.xml` file should have this component as its root view:

```xml
    <android.support.wearable.view.WatchViewStub 
    	xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools"
        android:id="@+id/watch_view_stub"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:rectLayout="@layout/rect_activity_main"
        app:roundLayout="@layout/round_activity_main"
        tools:context="com.androidessence.watchsample.MainActivity"
        tools:deviceIds="wear"/>
```

The main attributes to notice here are `app:rectLayout` and `app:roundLayout`. The layouts specified in each are the layouts that will appear on the device when the Activity loads, after the device shape has been determined.

Unlike an Activity on a mobile handset, referencing views is not as simple as calling `findViewById()` inside of `onCreate()`. You need to wait until the WatchViewStub has inflated the necessary layout. You do so by attached an `OnLayoutInflatedListener`:

```java
    final WatchViewStub stub = (WatchViewStub) findViewById(R.id.watch_view_stub);
    stub.setOnLayoutInflatedListener(new WatchViewStub.OnLayoutInflatedListener() {
        @Override
        public void onLayoutInflated(WatchViewStub stub) {
            // Handle views
        }
    });
```

If the same resource identifiers are used in both round and square layouts, your code can be the same here. If you need to do something special programatically based on screen shape, see [this article](http://developer.android.com/intl/pt-br/training/wearables/watch-faces/issues.html#ScreenShape) from the documentation.

# BoxInsetLayout

[This layout](http://developer.android.com/intl/pt-br/reference/android/support/wearable/view/BoxInsetLayout.html) is similar to a FrameLayout but is aware of the type of screen and is used to display something the same way on both round and square devices.

# WearableListView

Extending from the new RecyclerView, a WearableListView is simply a list component optimized for Wear devices. It is optimized by only showing three list items at a time, so the item height is programatically determined by the component. This is to allow for easier click handling of list items. It is also recommended that you create a special layout for each list item, so you can use an `OnCenterProximityListener` to provide a special implementation for views when they appear in the middle of the watch screen, as explained [here](http://developer.android.com/intl/pt-br/training/wearables/ui/lists.html#layout-impl). In this first segment on Wear UI, we’ll go through that in a little more detail, skipping over what we’ve already touched on.

# List Item

In this sample, we’ll use a WearableListItemLayout that extends from TextView, so we can just simply display some words on the screen for testing purposes:

```java
    public class WearableListItemLayout extends TextView implements WearableListView.OnCenterProximityListener{
     
        WearableListItemLayout(Context context) {
            this(context, null);
        }
     
        WearableListItemLayout(Context context, AttributeSet attributeSet) {
            this(context, attributeSet, 0);
        }
     
        WearableListItemLayout(Context context, AttributeSet attributeSet, int defStyle) {
            super(context, attributeSet, defStyle);
        }
     
        @Override
        public void onCenterPosition(boolean b) {
            // When centered, let's enlarge the text.
            // As of API 23, we don't need to pass context.
            if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                setTextAppearance(android.R.style.TextAppearance_Large);
            } else {
                setTextAppearance(getContext(), android.R.style.TextAppearance_Large);
            }
        }
     
        @Override
        public void onNonCenterPosition(boolean b) {
            // When leaving centered, set text backk to small.
            if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                setTextAppearance(android.R.style.TextAppearance_Small);
            } else {
                setTextAppearance(getContext(), android.R.style.TextAppearance_Small);
            }
        }
    }
```

As mentioned, we can implement the `OnCenterProximityListener` so that items in the center of our list at the time can be enlarged, as seen by this photo:

![WearList](/images/wear_list_sample.png)

We also implement the `onNonCenterPosition` method so that when an item moves out of the center location, it returns back to the appropriate text appearance.

Earlier I stated that the WearableListView class extends from RecyclerView, as a result the Adapter class is setup in a very similar fashion, so it has been omitted from this post. If you would like to see it, please check the code on [GitHub](https://github.com/androidessence/Android-Wear-Sample), which you can also run on your own Wear device or emulator to see the list in action.

More information on running your Wear app can be found [here](http://developer.android.com/intl/pt-br/training/wearables/apps/creating.html).