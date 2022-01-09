---
layout: post
author: adam
title: "Composing With Confidence: Finders And Matchers"
modified: 2022-01-09
published: true
tags: [jetpack compose, testing]
categories: [android]
---

In our previous section, we looked at the setup required to build and run tests for our Jetpack Compose code. Now that our tooling is configured, we can begin the basics of our tests. Let's start by understanding how to find components on the screen. 

<!--more-->

## Test Tags

We can start by going over the quickest way to reference a component. We do that through the `testTag()` modifier. Consider the following code:

```kotlin
Button(
    modifier = Modifier.testTag("PrimaryButton"),
)
```

To reference this node in our test, we can use the `onNodeWithTag()` function:

```kotlin
composeTestRule.onNodeWithTag("PrimaryButton").performClick()`
```

This concept is what we'll use throughout our tests - create a way to identify a component, and then use a function to find the component with that reference. Let's talk about what's really happening here. 

## Finders

Finders are methods for obtaining references to a component on the screen. By default, the Compose testing library provides a handful of helper methods for common properties, for example:

```kotlin
val labelNode = composeTestRule.onNodeWithText("LabelText")
val imageNode = composeTestRule.onNodeWithContentDescription("ContentDescription")
val primaryButton = composeTestRule.onNodeWithTag("PrimaryButton")
```

We also have symmetry in case we want all nodes that match a condition:

```kotlin
val allLabelNodes = composeTestRule.onAllNodesWithText("LabelText")
```

## Matchers

Under the hood, each of these finder methods is using some kind of `Matcher` to determine how to reference a node. If we want to find a node with text ourselves, we would write it like this:

```kotlin
val labelNode = composeTestRule.onNode(hasText("LabelText"))
```

We can replace `hasText()` with any number of built-in matchers, such as:

```kotlin
hasText()
hasClickAction()
isFocused()
isOn()
// ...
```

If we need to match multiple conditions, we can leverage the fact that `SemanticsMatcher` has implemented infix functions for `and` and `or`. Consider the following:

```kotlin
val hasTextAndIsClickable = composeTestRule.onNode(
    hasText("ButtonText") and hasClickAction(),
)

val hasThisOrThatText = composeTestRule.onNode(
    hasText("This") or hasText("That"),
)
```

To see a larger list of what's included, you can reference the [Compose testing cheatsheet](https://developer.android.com/jetpack/compose/testing-cheatsheet). 

![Compose Testing Cheatsheet](https://developer.android.com/images/jetpack/compose/compose-testing-cheatsheet.png)

If necessary, we can write our own custom finders and matchers, but we'll look at customization in a future session. For now, we'll move on to talk about actions. 