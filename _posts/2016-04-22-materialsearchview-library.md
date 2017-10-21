---
layout: post
author: adam
modified: 2016-04-22
title: MaterialSearchView Library
published: true
tags: [material design, library]
categories: android
description: MaterialSearchView is a library that simulates the material design search from Google.
---

Android has a built in SearchView component but it doesn't feel much like Material Design. This has even been asked on [StackOverflow](http://stackoverflow.com/questions/27556623/creating-a-searchview-that-looks-like-the-material-design-guidelines) because developers are having trouble recreating the same MaterialSearchView that appears in many of Google's applications. However, thanks to one of my good friends Maurício, you can now implement this great component in your own projects!

<!--more-->

The [MaterialSearchView](https://github.com/Mauker1/MaterialSearchView) library that he wrote provides a Material Design overlay for your screen that includes an EditText for searching, along with a ListView to show some items, typically recent or suggested searches. It supports both text input and voice input searches. Let's take a look at the sample:

![MaterialSearchView](/images/msv-sample.gif)

You'll notice three really great things about this sample:

1. A beautiful open and close animation.
2. The ListView is prepopulated with recent searches.
3. The ListView automatically updates to show the filtered and suggested searches based on user input.

The MaterialSearchView is supported with a min SDK of 14. For information on how to include it in your next project, as well as more details on the functionality of the library, please see the linked repository above. 