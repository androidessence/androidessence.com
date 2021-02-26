---
layout: post
author: adam
title: Enforcing Custom View Usage With Android Lint
modified: 2021-02-22
published: true
tags: [lint]
categories: [android]
---

Sometimes an Android project will have to implement a [custom view](https://developer.android.com/guide/topics/ui/custom-components) that is an extension of an existing Android view. We may do this for style purposes, or to implement additional logic, or any number of customization purposes. 

This solution brings a new problem for our codebase - how do we enforce that other developers use our custom view, instead of the Android framework view? We can solve this problem by writing our own [Android lint](https://developer.android.com/studio/write/lint) check. 

<!--more-->

## Lint Resources

The focus of this blog post is about a lint check to enforce custom views. We won't talk about the setup of writing a custom lint check, and each individual method in a ton of detail. If you've never written a custom lint check, please check out the following resources which were immensely helpful in making this post a reality. 

* [Write Custom Android Lint Rule - Layout Files - Varun Barad](https://varunbarad.com/blog/write-custom-android-lint-rule-layout-files)
* [Android Lint Framework — An Introduction - Saurabh Mishra](https://proandroiddev.com/android-lint-framework-an-introduction-36139deedf8b)
* [Android Lint Checks Demo - Alex Lockwood](https://github.com/alexjlockwood/android-lint-checks-demo)

# The Problem

For this post, let's imagine a scenario where our application was using a [BottomNavigationView](https://developer.android.com/reference/com/google/android/material/bottomnavigation/BottomNavigationView). You may create your own subclass of this view, in order to provide some custom styling or behavior. To demonstrate that, I created a [StudyGuideBottomNavigationView](https://github.com/AdamMc331/AndroidStudyGuide/blob/development/app/src/main/java/com/adammcneilly/androidstudyguide/ui/StudyGuideBottomNavigationView.kt) in this project I've been building on [Twitch](https://www.twitch.tv/adammc331).  

In a production application, we may have several custom views. We could create a `StudyGuideCheckBox`, or a `StudyGuideTextInputLayout`, and more. Eventually, it can be hard to keep track of which views should be replaced by our own custom view - especially as new developers join and they may be entirely unaware that a custom view implementation exists. We want to ensure that these custom views are used, though, so that we can keep a consistent implementation across our product.

Let's get started. 

# Implementing The Detector

Keeping in mind that we want this lint check to be scalable, no matter how many views we have, we should ensure that our detector isn't tightly coupled to one specific view. Let's call it an `UnusedStudyGuideViewDetector`:

```kotlin
class UnusedStudyGuideViewDetector : LayoutDetector() {
    // ...
}
```

A `LayoutDetector` is a custom type of `ResourceXmlDetector` that ensures this lint check is only run against layout XML files.  

# Defining The Issue

With our `Detector`, we also need to define an `Issue`, which represents the potential bug that lint is looking for. This is where we define severity, priority, and a description:

```kotlin
@JvmStatic
internal val ISSUE_UNUSED_STUDY_GUIDE_VIEW = Issue.create(
    id = "UnusedStudyGuideView",
    briefDescription = "Replace Material Design Components With Study Guide Custom Views",
    explanation = "This view must be replaced by a custom Study Guide implementation.",
    category = Category.CUSTOM_LINT_CHECKS,
    priority = 10,
    severity = Severity.ERROR,
    implementation = Implementation(
        UnusedStudyGuideViewDetector::class.java,
        Scope.RESOURCE_FILE_SCOPE
    )
)
```

# Defining Applicable Elements

Inside of our `LayoutDetector`, we can tell lint which XML elements are applicable to this check. In our use case, this should be all of the Android or Material UI components that we want to avoid:

```kotlin
class UnusedStudyGuideViewDetector : LayoutDetector() {

    override fun getApplicableElements(): Collection<String> {
        return listOf(
            "com.google.android.material.bottomnavigation.BottomNavigationView"
        )
    }
}
```

# Visiting Elements

Once we've defined the applicable elements, we'll get a callback each time lint finds one. This happens in the `visitElement` method. In our project, we don't need to do any special checks against the element - just knowing that the element exists, means we should report an error. 

```kotlin
class UnusedStudyGuideViewDetector : LayoutDetector() {

    override fun visitElement(context: XmlContext, element: Element) {
        context.report(
            issue = ISSUE_UNUSED_STUDY_GUIDE_VIEW,
            location = context.getNameLocation(element),
            message = ISSUE_UNUSED_STUDY_GUIDE_VIEW.getExplanation(TextFormat.TEXT),
        )
    }
}
```

# Result

As a result, we'll see a highlight in Android Studio wherever the Material BottomNavigationView is used:

![](assets/lint/StudyGuideLintError.png)

We will also see this in the output of the `./gradlew lint` command:

```
  /AndroidStudioProjects/AndroidStudyGuide/app/src/main/res/layout/content_main.xml:19: Error: This view must be replaced by a custom Study Guide implementation. [UnusedStudyGuideView]
      <com.google.android.material.bottomnavigation.BottomNavigationView
       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

# Supporting Quick Fixes

Often times if a lint warning can be corrected automatically, the IDE will offer a way to do so. In our codebase, we know the package name of the view that we want the developer to use, so let's implement that utility.

To support a quick fix, we supply `quickfixData` to the `context.report` method when we visit an element:

```kotlin
override fun visitElement(context: XmlContext, element: Element) {
    context.report(
        issue = ISSUE_UNUSED_STUDY_GUIDE_VIEW,
        location = context.getNameLocation(element),
        message = ISSUE_UNUSED_STUDY_GUIDE_VIEW.getExplanation(TextFormat.TEXT),
        quickfixData = LintFix.create()
            .replace()
            // Put the text we're looking to replace
            .text("com.google.android.material.bottomnavigation.BottomNavigationView")
            // Put the text we want to replace it with. 
            .with("com.adammcneilly.androidstudyguide.ui.StudyGuideBottomNavigationView")
            .build()
    )
}
```

Now our IDE will offer the chance to replace the line altogether:

![](assets/lint/StudyGuideLintErrorSuggestion.png)

# Supporting Additional View Checks

So far, our `UnusedStudyGuideViewDetector` is pretty tightly coupled to the `BottomNavigationView` case. Even if we updated `getApplicableElements` to consider more views, we'd run into an issue with the quickFixData - how do we determine what element we're looking at, and what to replace it with? 

To make this scalable, we can create a `Map` of views that we want to avoid, and what they should be replaced with. If we leverage String constants, we can create a readable explanation:

```kotlin
class UnusedStudyGuideViewDetector : LayoutDetector() {

    companion object {
        private const val MATERIAL_BOTTOM_NAVIGATION_VIEW =
            "com.google.android.material.bottomnavigation.BottomNavigationView"

        private const val STUDY_GUIDE_BOTTOM_NAVIGATION_VIEW =
            "com.adammcneilly.androidstudyguide.ui.StudyGuideBottomNavigationView"

        private val VIEW_REPLACEMENT_MAP = mapOf(
            MATERIAL_BOTTOM_NAVIGATION_VIEW to STUDY_GUIDE_BOTTOM_NAVIGATION_VIEW
        )
    }
}
```

Next, we can update `getApplicableElements` to only consider the keys of this map:

```kotlin
override fun getApplicableElements(): Collection<String> {
    return VIEW_REPLACEMENT_MAP.keys
}
```

Lastly, we can update `visitElement` to get the element name that was found, and use the map to look up its replacement:

```kotlin
override fun visitElement(context: XmlContext, element: Element) {
    val foundViewName = element.nodeName
    val suggestedName = VIEW_REPLACEMENT_MAP[foundViewName]

    context.report(
        issue = ISSUE_UNUSED_STUDY_GUIDE_VIEW,
        location = context.getNameLocation(element),
        message = ISSUE_UNUSED_STUDY_GUIDE_VIEW.getExplanation(TextFormat.TEXT),
        quickfixData = LintFix.create()
            .replace()
            .text(foundViewName)
            .with(suggestedName)
            .build()
    )
}
```

Now, the next time we implement a new custom view, all we need to do is add a new entry to `VIEW_REPLACEMENT_MAP` and our lint check will support that, too! 

# Recap

Lint is a very powerful tool to help ensure code quality and consistency throughout an application. There's helpful APIs to allow developers to write their own custom checks, to ensure our own consistency is enforced when necessary.

If you'd like to see this lint check in action, you can find it in the `lint-checks` module of this [sample application](https://github.com/AdamMc331/AndroidStudyGuide/blob/development/lint-checks/src/main/java/com/adammcneilly/studyguide/lint/UnusedStudyGuideViewDetector.kt). 
