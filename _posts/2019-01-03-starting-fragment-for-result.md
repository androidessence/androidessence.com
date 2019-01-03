---
layout: post
author: adam
title: Showing A Fragment For A Result
description: An explanation on how fragments can be used similar to startActivityForResult in Android.
modified: 2019-01-03
published: true
tags: [Android Essence]
categories: [android, tutorial, fragments]
---

A number of developers preach a single activity architecture on Android, which is something I've been trying to move forward to as well. In the process, though, I ran into one tricky problem. I don't have something like `startActivityResult` for fragments. If you're unfamiliar, `startActivityForResult` is a method that allows you to launch an activity with a specific request code, and when that activity finishes, your first activity will get a callback in `onActivityResult` and can do stuff with it. 

This post is going to walk through how we can achieve that same affect using fragments. 

<!--more-->

## Background

Before explaining how to pass results back from fragments, it's necessary to provide some background on fragment backstacks. It's actually quite important: how you handle your backstack will determine how you can handle the results. Specifically, it can depend on whether you use an `add` or `replace` transaction:

```kotlin
// Adds a fragment on top of whatever might already be in the container.
supportFragmentManager.beginTransaction()
   .add(R.id.container, newFragment, tag)
   .addToBackstack(tag)
   .commit()

// Replaces whatever is in the container with the new fragment.
supportFragmentManager.beginTransaction()
   .replace(R.id.container, newFragment, tag)
   .addToBackstack(tag)
   .commit()
```

In the `add` case, a new fragment is just placed on top of the container. Nothing happens to the previous fragment. No lifecycle events such as `onPause()` or even `onDestroyView()` are called. 

In the `replace` case, any existing fragments are removed from the container, and a new one is shown. This means the previous fragment runs through a couple lifecycle methods, including `onPause()` and `onDestroyView()`. It won't go all the way to `onDestroy()`, though, the fragment still exists.

When we get to the handling results section, I will explain why this distinction was important.

## Target Fragments

With that information in mind, let's talk about how we can get a result from one fragment to another. The key to this is using a target fragment. A [target fragment](https://developer.android.com/reference/android/app/Fragment.html#setTargetFragment(android.app.Fragment,%20int)) is used when one fragment is being started from another, and when the first one wants to get a result back. I had used this concept in dialog fragments quite often, but didn't realize the same could be applied to other fragments as well.

Consider I'm in a fragment responsible for displaying a name, `DisplayNameFragment`, and I want to launch `SetNameFragment` which shows the user an input to set the name that's displayed. We can set the target fragment and replace in the parent view like this:

```kotlin
val newFragment = SetNameFragment()
val tag = SetNameFragment::class.java.simpleName

newFragment.setTargetFragment(this, SET_NAME_REQUEST_CODE)
(activity as? MainActivity)?.replace(newFragment, tag)
```

Aside from the third line, this should look pretty similar to how you're showing fragments already. Now, how do we return the result?

Inside our second fragment, we have a button that will close down the fragment and send the name back to our target fragment. We can do so like this:

```kotlin
val name = name_input.text.toString()

(targetFragment as? DisplayNameFragment)?.setName(name)
activity?.supportFragmentManager?.popBackStackImmediate()
```

## Handling the result

