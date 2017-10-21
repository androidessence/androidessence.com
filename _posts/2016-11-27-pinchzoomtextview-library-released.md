---
layout: post
author: adam
modified: 2016-11-27
title: PinchZoomTextView Library Released
published: true
tags: [views]
description: PinchZoomTextView is a library that allows you to increase/decrease the font size with a two finger gesture.
categories: Android
---

I was recently asked if there was a way to increase/decrease the font size of a TextView by pinching on it, just as you can with many iamges. Well, it turns out that's quite possible!

All you have to do is override the `onTouchEvent()` method of your view, which is [exactly what I do](https://github.com/androidessence/PinchZoomTextView/blob/master/lib/src/main/java/com/androidessence/pinchzoomtextview/PinchZoomTextView.java#L67-L86) in my latest library, which allows you to pinch your screen to zoom in and out of a TextView. Check out the sample gif below:

<!--more-->

![Sample Gif](/images/pztv-sample.gif)

Interested in using this in your next project? You can get more details by viewing the library on [GitHub](https://github.com/androidessence/PinchZoomTextView).