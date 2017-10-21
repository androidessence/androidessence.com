---
layout: post
author: adam
title: Developing And Publishing Android Libraries
description: Step-by-step guide to developing an Android library and publishing it to JCenter.
modified: 2017-10-19
published: true
tags: [library]
categories: [android]
---

Third party libraries are a huge part of mobile app development. Popular tools like Retrofit, RxJava, Picasso, and many others prevent Android developers from reinventing the wheel everytime we need to do something over the network, asynchronously, or loading images. 

However, developing and publishing these libraries can be intimidating to many people. I think this occurs for a number of reasons - worry about keeping up with maintenance, being outshined, or sometimes thinking no one would use your code. I have many thoughts on those ideas, but will save them for another blog post. In this one, we'll just go over a step by step guide to creating and publishing a library to JCenter.

<!--more-->

# Creating the library module

## App Module

Unfortunately, something that Android Studio doesn't support is creating a library module right out of the gate. What this means is that you have to create your application module first. The way I prefer to do this is by going through the new project wizard as usual, but I tweak my package name by adding `.sample` at the end. 

![AndroidEssence](/images/library/sample_package.png)

Then you can complete the setup wizard as usual with all of your necessary preferences.

## Library Module

Now that you have a working project with an app module, you can right click on the project structure and select "New -> Module" to create your library module. This will have the same package name as your sample app, but without the `.sample` at the end of the package. The flow is demonstrated by the following screenshots:

![AndroidEssence](/images/library/new_module.png)

![AndroidEssence](/images/library/library_module.png)

![AndroidEssence](/images/library/library_package.png)

# Referencing the library from the app

To use your library code within your application, all you have to do is modify your dependencies in your application's `build.gradle` file:

```groovy
	compile project(":lib")
```

Now you can use whatever models/views/utility functions you wrote in your library and use them in your app, so you can provide some sample uses to your future users.

# Build library

I originally learned about publishing through a [post by my friend Eric](http://room-15.github.io/blog/2015/11/05/How-to-publish-a-library-to-jcenter/), but will relay the main points to you here. 

## Create Bintray account

Bintray is a software distribution service that you can use to push up your libraries. You can create an account [here](https://bintray.com/) and use GitHub authentication if you prefer.

## Library configuration

In your library module's `build.gradle` file, you'll need to add some code. First, at the top and underneat the plugin line, you define your library information. Here is an example from one of mine:

```groovy
	ext {
	    PUBLISH_GROUP_ID = 'com.adammcneilly'
	    PUBLISH_ARTIFACT_ID = 'recyclerviewutils'
	    PUBLISH_VERSION = '2.0.2'
	}
```

An explanation for each portion:
* Group Id is your package name.
* Artifact Id is the library's name.
* Publish Version is your current version. 

To provide a familiar example, to import the above library I would write:

```groovy
	compile 'com.adammcneilly:recyclerviewutils:2.0.2'
```

Also, at the bottom of the `build.gradle` file you'll need to include this line:

```groovy
	apply from: 'https://raw.githubusercontent.com/blundell/release-android-library/master/android-release-aar.gradle'
```

## Project configuration

Once you've added the necessary code to your library's build file, we need to modify the `build.gradle` file at the project level as well by adding the following dependencies:

```groovy
	dependencies {
	    classpath 'com.android.tools.build:gradle:2.3.3'
	    classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.2'
	    classpath 'com.github.dcendents:android-maven-plugin:1.2'
	}
```

Note that your gradle version number may change from the time of writing this. The first line will likely already be in the file from when Android Studio generated the project.

## Bintray API setup

Next you'll need to modify your `local.properties` file to include your Bintray information:

```groovy
	bintray.user=bintrayUserName
	bintray.apikey=abc123ApiKey
```

Your username is whatever you created your account with. To find your API key, login to bintray, click on your avatar at the top right and select edit profile. You will have a tab on the left menu for API Key and can find it there:

![AndroidEssence](/images/library/api_key.png)

## Generate release zip 

Last, you need to generate the zip file that will be uploaded to Bintray. You can do that by going into the terminal in Android Studio and running:

```bash
	./gradlew clean build generateRelease
```

Now you'll find in your lib/build folder a file with a name of `release-x.y.z.zip` which contains everything you'll need to upload.

## Bintray Upload

Once you've generated the zip file, the remaining steps are just a tedious upload:

1. Sign in to Bintray.
2. Select your maven repository.
3. Click the 'Add New Package' button.
4. Enter the required info and click submit.
5. Select this package, and on the right side you'll have a button to add a new version.
![AndroidEssence](/images/library/version.png)
6. Enter all the additional info and hit submit.
7. Select that version, and use the file uploader on the right. Make sure to select "explode this archive".
![AndroidEssence](/images/library/files.png)
![AndroidEssence](/images/library/explode.png)
8. You're done! People can now access your projects from your maven repo. If you want to link it to Jcenter, select your package and you'll see a "Link To Jcenter" option at the bottom right.
![AndroidEssence](/images/library/jcenter.png)
