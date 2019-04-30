---
layout: post
author: adam
title: Breaking the Buzzwords Barrier
description: Series of posts going over a few of the Android development buzzwords, and how to integrate them into your application.
modified: 2018-05-30
published: true
tags: [architecture]
categories: [android, tutorial]
---

MVVM? Retrofit? RxJava? Data binding? Architecture components? LiveData? Kotlin?

Right now, these buzzwords are heard all over the Android community. Every podcast/blog/conference talk is referencing one of these. Which can be **very** intimidating to new developers. Which one should I learn first? Do I need all of them? What are these things even used for?

The purpose of this series is to break all of that down, and show that none of these buzzwords are truly that scary. We'll go over an application I've published called [CashCaretaker](https://play.google.com/store/apps/details?id=com.androidessence.cashcaretaker) which is a simple finance tracker with all data stored locally on the device. It uses all of the buzzwords I mentioned further up, and we can go through them step by step.

You can checkout a simple gif of the project here:

<!--more-->

![Android Essence](/images/buzzwords/sample.gif)

You can find all the code for this project on [GitHub](https://github.com/adammc331/cashcaretaker).

Now, let's dive into [part 1]({{ site.baseurl }}{% link _posts/2018-05-30-breaking-the-buzzwords-barrier-mvvm.md %}) and talk about the architecture, first.