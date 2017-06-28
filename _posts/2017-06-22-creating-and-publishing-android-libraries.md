---
layout: post
title: Creating And Publishing Android Libraries
description: Tutorial covering the process to create and publish an Android library to JCenter.
modified: 2017-06-20
published: false
tags: [jcenter, library]
categories: [android]
---

Publishing an Android library is one of the most rewarding things you can do. You can make your life, and potentially the lives of other developers, much easier by reducing code you find yourselves reusing many times. Or, perhaps, you found a really unique way to do something and you want to provide it to others in an easy to use fashion. Regardless of your motives, libraries are essential to software developers, and building and publishing them is not as complicated as it seems.

In this post I'm going to give a high level walk through of creating a library module and uploading it to bintray and further jcenter, so that other people can include your library in their project.

<!--more-->

# Creating the sample and lib modules.

Currently, AndroidStudio doesn't let you start a project with a library module. It always kicks off with an application module. This is a hassle, 