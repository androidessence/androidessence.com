---
layout: post
author: adam
title: Leveraging The Robot Pattern For Espresso Tests
description: Demonstrates the benefits of the Robot pattern for automated testing in Android.
modified: 2017-07-04
published: true
tags: [espresso, testing, robot pattern]
categories: [android]
---

Espresso is a [testing framework for Android](https://developer.android.com/training/testing/ui-testing/espresso-testing.html) that allows developers to write automated tests for their applications. The benefit of automated testing is that you can write a test plan, and simply hit run and have all of the important features in your app tested effortlessly, and arguably more consistent and thorough than manual testing. There is no doubt that it is a lot faster.

However, one of the lesser known development patterns for automated testing is the robot pattern, which makes writing tests much easier while providing a painless way to update tests whenever your app changes. Let's take a deeper dive into what makes the robot pattern so powerful, and how to implement it in your next test suite.

<!--more-->

This tutorial is going to assume a basic knowledge of Android development. Exposure to Espresso already will help, but we'll do a quick run down of that first.

# Sample

First, let's take a look at an example of what we're building. This is a simple application that takes you to a new activity to enter a person's information and then displays that person's name in the original activity:

![Espresso](/images/espresso/sample.gif)

# Espresso Cheat Sheet

Below is a quick [cheat sheet](https://google.github.io/android-testing-support-library/docs/espresso/cheatsheet/) for using Espresso. We'll be using each of the categories you see below:

![Espresso Cheat Sheet](/images/espresso-cheatsheet.png)

If you're new to Espresso, here are the take aways from that cheat sheet:
 * ViewMatcher: This is something that describes a View that you want to interact with - whether that's the id, the text, the parent, or various other matchers in the above sheet.
 * ViewAction: This is an action you can perform on a View such as clicking it, typing text, and others.
 * ViewAssertion: This is used to assert something about the View like its position or whether it matches certain conditions.
 * onView(ViewMatcher): This method returns a ViewInteraction for a View that we can perform actions on or make assertions.
 * check(ViewAssertion): This method verifies that the ViewInteraction meets the given criteria.
 * perform(ViewAction): This method completes the supplied actions on the ViewInteraction.

# Test Adding A Person

Before we get into the robot pattern, it's important to understand the problem it solves. In the above application, we may want to automate a test that goes to the add person activity, enters their information, clicks submit, and verifies that the person was added to the list. We can do that like this:

```kotlin
    @Test
    fun addPersonSuccess() {
    	// Click the FAB
        onView(withId(R.id.fab)).perform(click())

        // Enter all their information and click submit
        onView(withId(R.id.first_name)).perform(clearText(), typeText(testPerson.firstName), closeSoftKeyboard())
        onView(withId(R.id.last_name)).perform(clearText(), typeText(testPerson.lastName), closeSoftKeyboard())
        onView(withId(R.id.age)).perform(clearText(), typeText(testPerson.age.toString()), closeSoftKeyboard())
        onView(withId(R.id.email_address)).perform(clearText(), typeText(testPerson.emailAddress), closeSoftKeyboard())
        onView(withId(R.id.submit)).perform(click())

        // Make sure we came back, check for item to be displayed
        onView(withId(R.id.fab)).check(matches(isDisplayed()))
        onView(withText(testPerson.fullName)).check(matches(isDisplayed()))
    }
```

Here is what our test is doing:
 1. Click the view with an id of "fab"
 2. Find a view with the id "first_name", clear the text, type in the first name, and close the keyboard. Repeat for each input.
 3. Find a view with the id "submit" and click it.
 4. Verify that the floating action button is displayed (this means we returned to the first activity).
 5. Verify that a view with our input person's name is displayed (this is the first row in the RecyclerView).

Now we can take this code, copy it, and modify it to test all error scenarios such as an empty input and verify that the error is displayed. One problem this poses is that you have many tests that are difficult to read (you start to see onView... all the time and may not easily spot differences), and another is that as you write more tests, you'll have more work to do if your view ever changes.

This downfall is demonstrated in a [presentation by Sam Edwards](https://www.youtube.com/watch?v=fhx_Ji5s3p4), where you can see that if your view changes, you need to go in and update every single test:

![Espresso](/images/espresso-no-robot.png)

# Robot Pattern

Now, imagine you had a robot you could use to perform each action for you. If your view ever changes, you wouldn't have to update each individual test anymore - you'd just need to update your robot. Our above diagram now looks something like this:

![Espresso](/images/espresso-robot.png)

Before we show the robot classes code, let's take a look at the implementation:

```kotlin
    @Test
    fun addPersonSuccess() {
        onView(withId(R.id.fab)).perform(click())

        AddPersonRobot()
                .firstName(testPerson.firstName)
                .lastName(testPerson.lastName)
                .age(testPerson.age)
                .emailAddress(testPerson.emailAddress)
                .submit()

        // Make sure we came back, check for item to be displayed
        onView(withId(R.id.fab)).check(matches(isDisplayed()))
        onView(withText(testPerson.fullName)).check(matches(isDisplayed()))
    }
```

As far as our test is concerned, we just gained two big benefits by using a robot:

 1. Increased readability - no more parsing the `onView(...)` code in your head, it's clear based on the method names what is happening.
 2. Abstracted the logic out of the test and only focus on the order. Our test no longer cares how first name is entered, it only cares that it's entered first, for example.

With this, we can write a robot once, and use it anywhere and it makes writing the actual tests more fun. Without it, I might not want to take the time to write four negative tests (one test with bad input for each of the four fields). However, with a Robot, writing those tests will take a lot less time. In fact, it's a very similar amount of code, which we discussed is already much less than the original:

```kotlin
    @Test
    fun checkFirstNameError() {
        onView(withId(R.id.fab)).perform(click())

        AddPersonRobot()
                .lastName(testPerson.lastName)
                .age(testPerson.age)
                .emailAddress(testPerson.emailAddress)
                .submit()
                .matchFirstNameError(rule.activity.getString(R.string.err_first_name_blank))
    }
```

As you can see above, our robot should use the [builder pattern](http://www.javaworld.com/article/2074938/core-java/too-many-parameters-in-java-methods-part-3-builder-pattern.html) which will allow us to chain calls together easily. Here is our `AddPersonRobot.kt` class:

```kotlin
    class AddPersonRobot {
        fun firstName(firstName: String): AddPersonRobot {
            onView(FIRST_NAME_MATCHER).perform(clearText(), typeText(firstName), ViewActions.closeSoftKeyboard())
            return this
        }

        fun lastName(lastName: String): AddPersonRobot {
            onView(LAST_NAME_MATCHER).perform(clearText(), typeText(lastName), ViewActions.closeSoftKeyboard())
            return this
        }

        // ...

        fun submit(): AddPersonRobot {
            onView(SUBMIT_MATCHER).perform(click())
            return this
        }

        fun matchFirstNameError(error: String): AddPersonRobot {
            onView(FIRST_NAME_MATCHER).check(matches(hasErrorText(error)))
            return this
        }

        fun matchLastNameError(error: String): AddPersonRobot {
            onView(LAST_NAME_MATCHER).check(matches(hasErrorText(error)))
            return this
        }

        // ...

        companion object {
            val FIRST_NAME_MATCHER: Matcher<View> = withId(R.id.first_name)
            val LAST_NAME_MATCHER: Matcher<View> = withId(R.id.last_name)
            // ...
        }
    }
```

I've left out some of the code for simplicity, but this demonstrates what I suggest doing for your robot class:

 1. Create static matchers for your views, so that if you ever change something about the view itself (such as its id) you only need to touch one spot.
 2. Create one method for each input, and one to check each error. You might have the urge to abstract this, but I like being able to keep each input separate. I also like the readability this gives me inside the test class itself.

That's all you need to implement the robot pattern for your tests. As linked above, Sam Edwards presentation from Droidcon 2016 goes into even deeper detail on how you can leverage this robot pattern for taking screenshots at each step of your tests. I hope you found this useful and this shows you that automated testing does not have to be as daunting as it sounds!

The full code for a sample application can be found on [GitHub](https://github.com/androidessence/EspressoSample).