---
layout: post
author: adam
title: RichTextView Library Released
modified: 2016-04-03
description: RichTextView provides SpanBuilder utils for a single TextView.
published: true
tags: [views, TextView, library]
categories: android
---

As a form of ultimate procrastination this weekend, I decided to spend the last two days developing a RichTextView library.

This weekend I built the RichTextView (the naming convention comes from the [RichTextBox](https://msdn.microsoft.com/en-us/library/system.windows.controls.richtextbox(v=vs.110).aspx) C# class) which allows the user to format different parts ofa. TextView in different ways. For example, if I wanted to display a string but only bold a portion of it, I could achieve that with this class.

<!--more-->

Formatting a TextView like this is not unheard of, as you can do things such as [inject HTML](http://stackoverflow.com/a/13350726/3131147) into a TextView or use a [SpannableString](http://developer.android.com/intl/es/reference/android/text/SpannableString.html) (which this library uses under the hood). However, I aimed to simplify this process and make it much easier for the user.

Here is a sample of the library in action:

![RichTextView](/images/rtv-sample.png)

If you'd like to learn more, such as the available methods and how to include this into your next project, please check out the library on [GitHub](https://github.com/androidessence/RichTextView).