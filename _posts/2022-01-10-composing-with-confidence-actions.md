---
layout: post
author: adam
title: "Composing With Confidence: Actions"
modified: 2022-01-10
published: true
tags: [jetpack compose, testing]
categories: [android]
---

Now that we understanding how to find items on the screen, the next step is learning how to interract with them. We do this by performing actions that the user would preform - typing, clicking, gestures, and more. Let's dive in. 

<!--more-->

## Actions

Once we have a reference to a node, we can perform an action:

```kotlin
composeTestRule.onNodeWithText("LOGIN").performClick()
composeTestRule.onNodeWithText("Email").performTextInput("testy@mctestface.com")
```

Similar to finders, we can provide more specific gestures by using the `performGesture()` method:

```kotlin
composeTestRule.onNodeWithText("LOGIN").performGesture {
    longClick()
}

composeTestRule.onNodeWithTag("ProfileCard").performGesture {
    swipeRight()
}
```

Like the last section, you can reference the [testing cheatsheet](https://developer.android.com/jetpack/compose/testing-cheatsheet) for more. 

![Compose Testing Cheatsheet](https://developer.android.com/images/jetpack/compose/compose-testing-cheatsheet.png)

As mentioned in the last section, we also have the ability to create our own custom action implementations as well, but we'll save customization for later on. Next, we can proceed to look at running assertions. 