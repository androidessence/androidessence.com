---
layout: post
author: adam
title: Material Design Specs Library
description: Material Design Specs library contains all of your material design color palettes.
modified: 2015-12-13
published: true
tags: [material design, library]
categories: android
---

Along with the [RecyclerViewCursorAdapter library](http://androidessence.com/recyclerview-cursoradapter-library/) that was released earlier this week, I have now released my second open source Android library. In collaboration with my good friend [Maurício](https://github.com/Mauker1), we have built a library for including the Material Design Specs in your Android application. Currently, the library has the full color palette along with some helper methods, and some elevation resources to give the proper elevation to your components. The source code, as well as instructions for including the library can be found on [GitHub](https://github.com/androidessence/MaterialDesignSpecs), so go there to check it out and give us a star!

<!--more-->

# Colors

The convention for accessing the color resources is like so:

```xml
	R.color.mds_colorName_colorLevel
```

where colorName and colorLevel are replaced with the values you want. There are 19 color names:

* red
* pink
* purple
* deeppurple
* indigo
* blue
* lightblue
* cyan
* teal
* green
* lightgreen
* lime
* yellow
* amber
* orange
* deeporange
* brown
* grey
* bluegrey

And there are 14 color levels:

* 50
* 100
* 200
* 300
* 400
* 500
* 600
* 700
* 800
* 900
* A100
* A200
* A400
* A700

The ones prefixed with an A represent accent shades. Please note that brown, grey, and bluegray do not have accent shades with them. With that, an example of a color may be `R.color.mds_red_500` or `R.color.mds_pink_A200`.

Available helper methods include:

* `getColorsByName()`
* `getAccentColorsByName()`
* `getColorsByLevel()`
* `getRandomColor()`
* `getRandomNonAccentColor()`
* `getRandomAccentColor()`
* `getRandomColorByLevel()`
* `getRandomColorByName()`
* `getRandomColorNonRepeating()`
* `getAllColors()`

Each of these methods will throw an [IllegalAccessException](http://developer.android.com/intl/pt-br/reference/java/lang/IllegalAccessException.html) if any of the resources are unable to be accessed. For passing in the color name or color level, you can make use of the available static strings, such as:

```java
	MaterialPalettes.RED
	MaterialPalettes.LEVEL_500
```

# Elevation

Currently the only supported elevation values are those found from the [elevation specs](https://www.google.com/design/spec/what-is-material/elevation-shadows.html#elevation-shadows-elevation-android) that follow convention:

```xml
	R.dimen.mds_elevation_componentName
```

where componentNAme can be replaced by "dialog" or "appbar". With its current functionality, there is no need for programatic helper methods for using elevation values. This may change as more specs are included in the future.

We hope this gives you everything you need for moving forward and incorporating Material Design in your applications. If you feel like our library is missing anything, let us know in the comments or submit an issue on GitHub!