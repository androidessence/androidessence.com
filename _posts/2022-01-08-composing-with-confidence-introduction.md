---
layout: post
author: adam
title: "Composing With Confidence: Introduction"
modified: 2022-01-08
published: true
tags: [jetpack compose, testing]
categories: [android]
---

Testing is a crucial part of any software development workflow. This helps us ensure our code maintains a certain level of quality, and that it behaves the way we expect it. Testing can protect us against future refactors and changes to our code, allowing us to make updates and ship them with confidence. 

Let's look at how we can create this confidence with the new Jetpack Compose UI framework. 

<!--more-->

There's a lot that goes into testing with Jetpack Compose, so we'll review them in a multi part series. In this first section, we'll cover what we need to do to be up and running with our Compose UI tests. Tooling is just as important as the test themselves, and deserves a dedicated conversation.

## Dependency

As is routine for us Android developers, we can start by adding our Gradle dependency:

```groovy
androidTestImplementation "androidx.compose.ui:ui-test-junit4:$rootProject.ext.versions.compose"
```

You can find the documentation for this dependency [here](https://developer.android.com/reference/kotlin/androidx/compose/ui/test/junit4/package-summary).

## Test Rule

Once you've added this dependency, we need to create a JUnit test rule inside our test classes. This will create the screen that we want to test.

There are two ways we can create a test rule, which depend on what we want to test. 

### Testing An Activity

If we want to create a specific Activity and test the Compose UI code inside of it, we use `createAndroidComposeRule()`. 

```kotlin
class MainActivityTest {
    
    @get:Rule
    val composeTestRule = createAndroidComposeRule<MainActivity>()
}
```

### Testing Individual Composables

If you don't care about your specific Activity, but just want something to test an individual composable, we can use the short and sweet `createComposeRule()`:

```kotlin
class PrimaryButtonTest {

    @get:Rule
    val composeTestRule = createComposeRule()
}
```

This is effectively the same as the previous method, but creates an instance of a `ComponentActivity`. An empty Activity that allows us to render Compose code, like this:

```kotlin
class PrimaryButtonTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun renderButton() {
        composeTestRule.setContent {
            PrimaryButton()
        }
    }
}
```

However, this alone won't work - when we run this, we'll get an error about `ComponentActivity` not being defined. To do so, we can create a new manifest file and put it in `app/src/debug/AndroidManifest.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <activity android:name="androidx.activity.ComponentActivity" />
    </application>
</manifest>
```

When we build our app, this will be merged into our debug AndroidManifest and now our UI test will be able to create an instance of this Activity. 

At this point, we can write tests and run them on a physical device using a task like `./gradlew connectedCheck`. Having an understanding of the setup required, we can move on to understanding how to put items on the screen and interract with them in the next section. 