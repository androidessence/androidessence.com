---
layout: post
author: adam
title: "Composing With Confidence: Assertions"
modified: 2022-01-11
published: true
tags: [jetpack compose, testing]
categories: [android]
---

In the previous parts, we looked at finders, matchers, and actions. Now that we know how to reference a component on the screen, and interact with it, we can start making assertions about what's on the screen. 

<!--more-->

## Assertions

Let's reference our helpful cheat sheet again. We can find our assertions on the right:

![Compose Testing Cheatsheet](https://developer.android.com/images/jetpack/compose/compose-testing-cheatsheet.png)

We have, of course, our standard `assert()` method where we can supply any matcher:

```kotlin
composeTestRule.onNodeWithTag("PrimaryButton").assert(
    hasText("Button Text")
)
```

Compose offers a number of out of the box assertions as well, for common use cases.

There are assertions about the display of a component:

```kotlin
node.assertIsDisplayed()
node.assertIsNotDisplayed()
node.assertExists()
node.assertDoesNotExist()
```

We can make assertions about values of a component:

```kotlin
node.assertTextEquals()
node.assertTextContains()
node.assertValueEquals()
```

For interactive components, like checkboxes and radio buttons, we can make assertions about their state as well:

```kotlin
node.assertIsOn()
node.assertIsOff()
node.assertIsSelected()
node.assertIsNotSelected()
```

Like the other functions, we can write our own custom assertions - but we'll dive into that later. A spoiler, though, is it's not quite the assertion that needs to be customized. We would need to write a `matcher` that we can supply to the base `assert()` function. We'll look at that soon. 

Next, let's take all this information and use it to write our first test, starting with an individual composable. 