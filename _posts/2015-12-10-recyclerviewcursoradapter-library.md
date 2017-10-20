---
layout: post
author: adam
title: RecyclerViewCursorAdapter Library
description: RecyclerViewCursorAdapter is a libary that brings the CursorAdapter to the Recyclerview.
modified: 2015-12-10
tags: [recyclerview, adapter, library]
categories: [android]
published: true
---

Today I released my first library which is for a RecyclerviewCursorAdapter.

Using a ListView to display database data becomes a lot easier when you use a CursorAdapter combined with a CursorLoader to display data from your ContentProvider. The main benefit of [CursorLoader](http://developer.android.com/intl/pt-br/reference/android/content/CursorLoader.html) is explained in the docs:

<!--more-->

> This class implements the [Loader](http://developer.android.com/intl/pt-br/reference/android/content/Loader.html) protocol in a standard way for querying cursors, building on [AsyncTaskLoader](http://developer.android.com/reference/android/content/AsyncTaskLoader.html) to perform the cursor query on a background thread so that it does not block the application’s UI.

The CursorAdapter helped by efficiently clearing the reference to the Cursor that it held, so the responsibility didn't fall on the developer.

When ListView was replaced by RecyclerView, it came with a different type of Adapter with no CursorAdapter implementation. After being inspired by a StackOverflow [answer](http://stackoverflow.com/a/27732748/3131147) to see how a CursorAdapter could be used as the underlying data source of a RecyclerView.Adapter class, I quickly abstracted the idea into a library. You can find this library with samples on [GitHub](https://github.com/androidessence/RecyclerViewCursorAdapter). The README there will explain how to include this library in your project and get started using the CursorAdapter with your RecyclerView!