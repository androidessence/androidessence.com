---
layout: post
author: adam
title: Maintain Library Versions Across Modules
description: How to use gradle to keep consistent library versioning across modules in your project.
modfied: 2017-05-05
published: true
tags: [gradle, library]
---

It is tough enough maintaing your app by updating the support library version numbers every time a new version is out, let alone factoring in any other third party libraries you may use. This is especially painful if you have multiple modules, as you have to update the version in each `build.gradle` file. Thankfully, we can make use of the project level gradle file to make this more maintainable.

<!--more-->

# Define Variables In Project Gradle File

In your root `build.gradle` file for the project, you can define some [ExtraPropertyExtensions](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html) that will be reused for various modules. That would look something like this:

```groovy
	allprojects {
	    repositories {
	        jcenter()
	    }

	    ext {
	        //Android
	        androidBuildToolsVersion = "25.0.2"
	        androidMinSdkVersion = 16
	        androidTargetSdkVersion = 25
	        androidCompileSdkVersion = 25

	        //Libraries
	        supportLibraryVersion = "25.3.1"
	        playServicesVersion = "10.2.1"
	    }
	}
```

Now we've got some reusable variables that we want to be consistent across our various modules.

# Define Global Configuration Reference In Module

In the `build.gradle` file of each module, inside the "android" block, we'll need to define a reference to the global configuration we just created. We can do so with the following single line of code:

```groovy
	android {
	    def globalConfiguration = rootProject.extensions.getByName("ext")

	    ...
	}
```

With that, there are two ways you can use these properties. When trying to reference them individually, for properties such as build tools or compile sdk version, you can reference them using the property key:

```groovy
android {
	...
	compileSdkVersion globalConfiguration["androidCompileSdkVersion"]
	buildToolsVersion globalConfiguration["androidBuildToolsVersion"]
	...
}
```

If you want to include version numbers as part of your compile statements, you can simply use gradle's string interpolation:

```groovy
	compile "com.android.support:appcompat-v7:${supportLibraryVersion}"
```

That's it! That one simple little trick is all you need to have consistent library versions across modules. Some notes on further studying this:
* A gist of all above code can be found [here](https://gist.github.com/AdamMc331/a1dd4e8503c0cf86e0165c5f14c308ba).
* A project that goes even deeper on this topic and reusing compile statements (beyond just build numbers), can be found [here](https://github.com/android10/Android-CleanArchitecture). This came up as a result of a Facebook discussion I was a part of.