---
layout: post
author: adam
title: Unit Testing Custom Lint Checks
modified: 2021-03-01
published: true
tags: [lint]
categories: [android]
---

In our [previous post]({{ site.baseurl }}{% link _posts/2021-02-22-enforce-custom-views-with-lint.md %}) we looked at writing a custom lint check to enforce usages of a custom view instead of an Android framework implementation.

In this post, we'll go over how to unit test such a scenario, and take the opportunity to look at some additional benefits of unit testing with lint as well. 

<!--more-->

## Dependencies

Just to get it out of the way, you will need to add the `lint-tests` dependency on your lint module in the project:

```
dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"

    compileOnly "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
    compileOnly "com.android.tools.lint:lint-api:27.1.2"
    compileOnly "com.android.tools.lint:lint-checks:27.1.2"

    // ADD THESE!
    testImplementation "com.android.tools.lint:lint-tests:27.1.2"
    testImplementation "junit:junit:4.13.2"
}
```

# LintDetectorTest

When writing a test for our lint check, we want the test class to extend `LintDetectorTest`. This class provides some utilities for validating a `Detector` - which we wrote in the previous post. 

`LintDetectorTest` is an abstract class that requires us to define the `Detector` that we're testing, and which `Issue`s that are under test as well.

```kotlin
@RunWith(JUnit4::class)
class UnusedStudyGuideViewDetectorTest : LintDetectorTest() {
    override fun getDetector(): Detector {
        return UnusedStudyGuideViewDetector()
    }

    override fun getIssues(): MutableList<Issue> {
        return mutableListOf(
            UnusedStudyGuideViewDetector.ISSUE_UNUSED_STUDY_GUIDE_VIEW
        )
    }
}
```

# Test Cases

Before looking at code, let's review all steps that are required to writing a lint test. In our project, each one is going to follow the same flow:

1. Create a `TestLintTask`.
2. Provide all of the `TestFile`s that we want to test. This is where we mock up the XML, or potentially Java/Kotlin files that we want to test.
3. Provide any additional configuration of the task, if necessary. 
4. Run the task.
5. Perform any assertions on the `TestLintResult`.

# Success Test

Let's write a test that we expect to pass. To do this, we'll provide some mock XML that uses the `StudyGuideBottomNavigationView`. We'll run lint, and ensure there were no errors. To do this, we'll look at each step individually.

## Start TestLintTask

To create an instance of `TestLintTask`, we can call the `lint()` method:

```kotlin
@Test
fun passesWithStudyGuideBottomNavigationView() {
    var task = lint()

    // ...
}
```

Spoiler: You don't need to keep a reference to the task, we'll remove it in the last code sample, but it helps to understand as we split steps. 

## Provide TestFile

Next we need to create the file that we're going to run lint against. We do that by calling the appropriate method, `xml(fileName, fileContents)`. There's a kotlin and a java equivalent as well. Here is what that looks like for our test:

```kotlin
@Test
fun passesWithStudyGuideBottomNavigationView() {
    // ...

    val layoutFile = xml(
        "res/layout/layout.xml",
        """
<com.adammcneilly.androidstudyguide.ui.StudyGuideBottomNavigationView xmlns:android="http://schemas.android.com/apk/res/android"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
/>
"""
    )

    task = task.files(layoutFile)

    // ...
}
```

## Provide Additional Configurations

When running these tests for the first time, you may experience an `AssertionError`:

```
java.lang.AssertionError: This test requires an Android SDK: No SDK configured. 
If this test does not really need an SDK, set TestLintTask#allowMissingSdk(). Otherwise, make sure an SDK is available either by specifically pointing to one via TestLintTask#sdkHome(File), or configure $ANDROID_HOME in the environment
```

If you don't need the SDK installed, (which we don't in our test), we can just implement the test method the error described:

```kotlin
@Test
fun passesWithStudyGuideBottomNavigationView() {
    // ...

    task = task.allowMissingSdk()
    
    // ...
}
```

There are some other configurations you can supply here, such as requiring the compileSdkVersion to be installed, whether or not to allow compilation errors, setting a baseline file, and more. 

## Run And Perform Assertions

Once you've set up your task, we can run it and perform any assertions. Here, we want to ensure that no errors occured:

```kotlin
@Test
fun passesWithStudyGuideBottomNavigationView() {
    // ... 

    task.run()
        .expectClean()
}
```

## All Together

Let's throw all of that code together into one block: 

```kotlin
@Test
fun passesWithStudyGuideBottomNavigationView() {
    lint()
        .files(
            xml(
                "res/layout/layout.xml", """
<com.adammcneilly.androidstudyguide.ui.StudyGuideBottomNavigationView xmlns:android="http://schemas.android.com/apk/res/android"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
/>
"""
            )
        )
        .allowMissingSdk()
        .run()
        .expectClean()
}
```