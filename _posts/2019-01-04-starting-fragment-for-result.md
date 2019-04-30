---
layout: post
author: adam
title: Showing A Fragment For A Result
description: An explanation on how fragments can be used similar to startActivityForResult in Android.
modified: 2019-01-04
published: true
tags: [fragments]
categories: [android]
---

A number of developers preach a single activity architecture on Android, which is something I've been trying to move forward to as well. In the process, though, I ran into one tricky problem. I don't have something like `startActivityResult` for fragments. If you're unfamiliar, `startActivityForResult` is a method that allows you to launch an activity with a specific request code, and when that activity finishes, your first activity will get a callback in `onActivityResult` and can do stuff with it. 

This post is going to walk through how we can achieve that same affect using fragments. 

<!--more-->

## Example

To walk through this concept, we're going to write a small application with two fragments. One is our `DisplayNameFragment` which displays a message like "Your name is: Adam", with a button that will navigate to a new fragment, `SetNameFragment`, where you can enter a name in an EditText, hit a finish button, and it will return back to the previous fragment and display whatever you put in the input. This is what it will look like:

![Android Essence](/images/FragmentResultsSample.gif)

## Target Fragments

The key to this is using a target fragment. A [target fragment](https://developer.android.com/reference/android/app/Fragment.html#setTargetFragment(android.app.Fragment,%20int)) is used when one fragment is being started from another, and when the first one wants to get a result back. I had used this concept in dialog fragments quite often, but didn't realize the same could be applied to other fragments as well.

## Starting The Fragment

The first step is starting our `SetNameFragment` from within `DisplayNameFragment`. We can do that by putting the following code in the button's click listener:

```kotlin
  private fun launchSetNameFragment() {
      val newFragment = SetNameFragment()
      val tag = SetNameFragment::class.java.simpleName

      newFragment.setTargetFragment(this, SET_NAME_REQUEST_CODE)
      (activity as? MainActivity)?.replace(newFragment, tag)
  }
```

The `MainActivity.replace()` method above will replace the existing fragment inside a container with the new one that was passed to it. The fact that we use replace is important, and we'll talk about that in a second. 

## Returning The Result

Appropriately named, the fragment passed into `setTargetFragment()` can be retrieved in `getTargetFragment()` of the new one. So, inside our `SetNameFragment`, we can pass back the result by putting the following code inside the button listener:

```
  private fun returnWithName() {
      val name = name_input.text.toString()

      (targetFragment as? DisplayNameFragment)?.setName(name)
      activity?.supportFragmentManager?.popBackStackImmediate()
  }
```

In this example, we're _expecting_ that the target fragment is an instance of DisplayNameFragment. My first instinct was to write a basic/generic way of handling this, but I've decided not to over engineer it. 

When a name is entered, we pass that result back to the `DisplayNameFragment` by calling `setName()`, and then telling the activity to pop this fragment off the top, which will then reshow our `DisplayNameFragment`. 

## Handling The Result

Before I show you the code for the `setName()` method above, it's necessary to provide some background on fragment backstacks. How you use your backstack will determine how you can handle the results. Specifically, it can depend on whether you use an `add` or `replace` transaction:

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

I highly recommend [this answer](https://stackoverflow.com/questions/18634207/difference-between-add-replace-and-addtobackstack/48106957#48106957) on Stack Overflow to see the lifecycle differences.

### Using Add

If you're using an add transaction, the fragment is still visible on screen (just behind the previous one), so you're free to manipulate the view directly when handling the result:

```kotlin
  fun setName(name: String) {
      name_textview.text = getString(R.string.your_name_is).format(name)
  }
```

### Using Replace

If you're using replace, though, the above code will crash. That is because the `DisplayNameFragment` view was destroyed so that it can be replaced by the new one. The above code would be trying to access a view that didn't exist, and so it would crash. To handle this, we can set the name in a class level variable, and read from it in `onResume()` to update our TextView:

```kotlin
  class DisplayNameFragment : Fragment() {

      private var name: String = ""

      override fun onResume() {
          super.onResume()

          name_textview.text = getString(R.string.your_name_is).format(name)
      }

      fun setName(name: String) {
          this.name = name
      }

      // ...
  }
```

This code wouldn't even work in the add case, because `onResume()` won't be called on a back press.

While it's a little more code to do it this way, I still prefer using replace transactions, but it's up to you and your needs. Just be aware that what you chose will affect how you pass information between fragments.

## Conclusion

TL;DR You can use a target fragment to pass data between fragments in the same way you would use `startActivityForResult()` and `onActivityResult()`. You just need to be aware of how `add` and `replace` transactions affect the fragment lifecycles, and how to handle the result in each case. 

If you'd like to see the full sample, you can find the project on [GitHub](https://github.com/AdamMc331/FragmentResults).