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

In this post, we'll go over how to unit test such a scenario, and take the opportunity to look at some additional options of unit testing with lint as well. 

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

`LintDetectorTest` is an abstract class that requires us to define the `Detector` that we're testing, and which `Issues` that are under test as well.

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
2. Provide all of the `TestFiles` that we want to test. This is where we mock up the XML, or potentially Java/Kotlin files that we want to test.
3. Provide any additional configuration of the task, if necessary. 
4. Run the task.
5. Perform any assertions on the `TestLintResult`.

# Success Test

Let's write a test that we expect to pass. To do this, we'll provide some mock XML that uses the `StudyGuideBottomNavigationView`. We'll run lint, and ensure there were no errors. Let's break down each step. 

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
<com.adammcneilly.androidstudyguide.ui.StudyGuideBottomNavigationView 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />
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
<com.adammcneilly.androidstudyguide.ui.StudyGuideBottomNavigationView 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />
"""
            )
        )
        .allowMissingSdk()
        .run()
        .expectClean()
}
```

# Exploring LintTestResult

Now that we have a formula for creating a lint test, let's highlight some of the helpful methods for validation inside `LintTestResult`. We've already seen `expectClean()`, which is useful for verifying that there are no lint errors. What do we do if we are testing code with violations, though?

## Validating Warning Counts

A quick, high level way to validate a lint result is to check the number of warnings. While it's not as thorough as checking that a specific warning was emitted, knowing that the count emitted was the count expected can provide confidence.

Every lint warning has a `Severity` associated with it. Most common are `Severity.WARNING` and `Severity.ERROR`, but we also have `INFORMATIONAL` and `IGNORE`. 

For warnings and errors, we have utility methods already:

```kotlin
lint()
    // ...
    .run()
    .expectWarningCount(0)
    .expectErrorCount(1)
```

If you want to check other `Severity` levels, you can pass them into the `expectCount` method:

```kotlin
lint()
    // ...
    .run()
    .expectCount(
        5,
        Severity.INFORMATIONAL
    )
```

## Validating Lint Output

If you're aware of the actual text that you expect to be output from a lint test, you can do a match against a string:

```kotlin
lint()
    // ...
    .run()
    .expect(
        """
res/layout/layout.xml:2: Error: This view must be replaced by a custom Study Guide implementation. [UnusedStudyGuideView]
<com.google.android.material.bottomnavigation.BottomNavigationView 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
1 errors, 0 warnings
    """
    )
```

If you want to get really fancy with this approach, there is a `.expectMatches()` method that allows you to pass in some RegEx, but I am not skilled enough to write about that one. 

Even more robust, the lint-tests artifacts provides a `TestResultChecker` interface that you can implement yourself, and use that to valid your lint outputs, if you need special validations on the output.

## Validating Fixes

In addition to verifying that your lint check outputs correctly, you may want to verify that any quick fixes you provide work as well. There are two quick approaches to validating that.

First, we can validate the diff result of a lint fix using the `.expectFixDiffs()` method:

```kotlin
lint()
    // ...
    .run()
    .expectFixDiffs(
        """
Fix for res/layout/layout.xml line 2: Replace with com.adammcneilly.androidstudyguide.ui.StudyGuideBottomNavigationView:
@@ -2 +2
- <com.google.android.material.bottomnavigation.BottomNavigationView
+ <com.adammcneilly.androidstudyguide.ui.StudyGuideBottomNavigationView
        """.trimIndent()
    )
```

Another approach is to supply a `TestFile` with what you expect your code to look like after a fix:

```kotlin
val initialFile = xml(
    "res/layout/layout.xml",
    """
<com.google.android.material.bottomnavigation.BottomNavigationView 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />
"""
)

val fixedFile = xml(
    "res/layout/layout.xml", 
    """
<com.adammcneilly.androidstudyguide.ui.StudyGuideBottomNavigationView 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />
"""
)

lint()
    .files(initialFile)
    .run()
    // If you only have one quick fix, we can pass null. Otherwise, pass the name
    // of this fix you want to validate. 
    .checkFix(fix = null, after = fixedFile)
```

# Recap

There's even more we can do with the lint-tests dependency, but this should be more than enough to get started. We looked at how to formulate a custom lint test, and the different types of validations we can run against those checks. This should be a great foundation to determine which validations are best for your application. If you'd like to see more sample unit tests, you can find them in [this sample project](https://github.com/AdamMc331/AndroidStudyGuide/blob/development/lint-checks/src/test/java/com/adammcneilly/studyguide/lint/UnusedStudyGuideViewDetectorTest.kt). 