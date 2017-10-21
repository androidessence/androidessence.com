---
layout: post
author: adam
modified: 2017-04-14
title: Creating Android App Shortcuts
published: true
tags: [app-shortcuts, nougat]
description: Shortcuts are special actions that appear when you long press on a launcher icon. This post teaches how to implement them.
categories: Android
---

Android app shortcuts are a special feature added in Android 7.1 (API 25). Shortcuts are special actions that appear when you long press on a launcher icon, that will trigger a specified intent. With this post you will learn how to create both static and dynamic app shortcuts. The only pre-requisite is that your app is targeting API 25 or greater.

#### Static Shortcuts

Static app shortcuts are shortcuts that are defined via XML. This means they are created at compile time, and will be consistent as long as your app is on the user's device. Common use cases for this would be to launch a regularly used activity in your app that is not the launcher activity.

<!--more-->

Let's first look at how we can define a shortcut in XML:

```xml
  <?xml version="1.0" encoding="utf-8"?>
  <shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
      <shortcut
          android:enabled="true"
          android:icon="@mipmap/ic_launcher"
          android:shortcutId="static_shortcut"
          android:shortcutLongLabel="@string/static_shortcut_long_label"
          android:shortcutShortLabel="@string/static_shortcut_short_label">
          <intent
              android:action="android.intent.action.VIEW"
              android:targetClass="com.androidessence.appshortcuts.MainActivity"
              android:targetPackage="com.androidessence.appshortcuts" />
      </shortcut>
  </shortcuts>
```

The above snippet is in a file called shortcuts.xml inside the xml-v25 folder. The property names make this pretty straight forward, but here is a quick run down:

 - enabled - Whether or not your shortcut can be used.
 - icon - The icon that appears on your shortcut.
 - shortcutId - The id used to reference your shortcut in code.
 - Long/Short label - A label that appears on your shortcut. Which label is shown is dependent on the screen size of the device being used.
 - intent - The intent that should be launched when your shortcut is clicked. In this case, just launching our MainActivity.

In order for this shortcut to appear after a long press, though, we need to add some meta-data to our launcher activity. Find your default activity in your AndroidManifest.xml file, and add this information:

```xml
  <meta-data
     android:name="android.app.shortcuts"
     android:resource="@xml/shortcuts" />
```

And that's it! You can run the app at this point and you should see a static shortcut appear.

#### Dynamic Shortcuts

Dynamic app shortcuts are added via Java code when your launcher activity is created. As a result of this, they can be changed, disabled, and customized for the user. A use case for dynamic app shortcuts would be a messaging app - you could create a shortcut for each of your user's top 5 contacts, for example, that open the messaging thread with the contact that was clicked.

Creating a dynamic shortcut is very straightforward, while you may need to tweak this to create shortcuts specific to your app. To start, just use the [ShortcutInfo.Builder](https://developer.android.com/reference/android/content/pm/ShortcutInfo.html) class to create a shortcut like we did in XML:

```java
  Intent googleIntent = new Intent(Intent.ACTION_VIEW, Uri.parse("https://www.google.com"));

  ShortcutInfo shortcut = new ShortcutInfo.Builder(this, "dynamic_shortcut")
     .setShortLabel(getString(R.string.dynamic_shortcut_short_label))
     .setLongLabel(getString(R.string.dynamic_shortcut_long_label))
     .setIcon(Icon.createWithResource(this, R.mipmap.ic_launcher))
     .setIntent(googleIntent)
     .build();
```

Then, we can get our [ShortcutManager](https://developer.android.com/reference/android/content/pm/ShortcutManager.html) and add the shortcut:

```java
  ShortcutManager shortcutManager = getSystemService(ShortcutManager.class);
  shortcutManager.addDynamicShortcuts(Collections.singletonList(shortcut));
```

Now you've created a dynamic app shortcut, too! If you run the app again, you will see this shortcut when you long press on the launcher icon as well. One thing to note, is that the icon we just made was inside the onCreate() method of our activity. That means it won't appear right after install, but it will appear after the user opens the app for the first time.

#### Continued Development

This was just a quick starting guide on how to create shortcuts, but you may have more questions. How do I change them? How do I disable them? What happens if a user pins a shortcut? Can I track analytics on shortcuts?

Many of those questions can be answered in the [official documentation](https://developer.android.com/guide/topics/ui/shortcuts.html) for App Shortcuts.

[Caren Chang](https://twitter.com/calren24) gave an excellent presentation on app shortcuts at Droidcon Boston, but the video is not up at the time of posting this. I will try to update the blog post when it is.

The sample application used for this blog post can be found on [GitHub](https://github.com/androidessence/App-Shortcuts).