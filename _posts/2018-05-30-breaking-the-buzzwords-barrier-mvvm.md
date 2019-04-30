---
layout: post
author: adam
title: Breaking the Buzzwords Barrier Part 1&#58; MVVM
description: Demonstrates how to architect an Android application using MVVM.
modified: 2018-05-30
published: true
tags: [architecture, mvvm]
categories: [android, tutorial]
---

When you're starting out with Android development, and even as an expert, you will hear about a lot of different architecture patterns. Anything from:

 - Model-View-Controller
 - Model-View-Presenter
 - Model-View-ViewModel
 - Model-View-Intent

It can be extremely hard to know which one to pick, what their differences are, and why they matter. I will tell you that even with my three years of Android experience at the point of writing this, I have trouble answering the first two questions. I can, however, explain why these architecture patterns matter - and it boils down to the idea of separation of concerns.

<!--more-->

# Separation Of Concerns

Let's take a look at a high level explanation of MVVM using a flow chart taken from [Wikipedia](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel):

![MVVM](/images/buzzwords/MVVM.png)

As you can see in this diagram, our application's logic is broken up into three parts:

 - View, which refers to your Activity or Fragment, who's sole responsibility should be to display data.
 - ViewModel, which is a class that manages the state of your view, interacts with the Model, and handles some relevant business logic.
 - Model, which refers to your data source. This is often a repository that interacts with a database or remote server, or a combination of the two.

Why should we separate our concerns this way? Well, without them, we'd be dumping all of this data into one class (our Fragment or Activity) perhaps with the exception of the networking code. This makes our application EXTREMELY difficult to refactor someday, if we chose to do so. It's difficult because everything is so tightly coupled together, and you'll spend more time than is necessary chasing down everything that needs to be updated.

When your application is broken up like the diagram, it becomes easier - want to swap out a database for actual network requests? No problem - just update your respository class, and you won't even have to tweak your View or ViewModel at all (assuming the actual model structure is the same). Want to change the way your view looks completely? You can go ahead and change up your Fragment without ever having to touch the ViewModel or underlying Model code. 

# Cash Caretaker Example

Now that we understand at a high level of how MVVM is broken down, let's look at a flow chart for the account package inside Cash Caretaker:

![MVVM](/images/buzzwords/cashcaretaker_mvvm.png)

We'll briefly break down each block, but go over each part in more detail in later posts:

 - Account refers to the simple POJO that contains all the information an account would need.
 - The AccountAdapter displays a list of accounts in a RecyclerView, so it uses the account class to know what to display.
 - The AccountFragment houses a RecyclerView, and displays the list as well as an empty state if necessary. 
 - The AccountViewModel requests data, but also exposes a state to the Fragment so it knows what information to display (a list, or an empty state).
 - The CC Repository is a middle man between the ViewModel and the Database. This is necessary because I may consider swapping out the database for a server, but the necessary calls and connection with the ViewModel shouldn't have to change.
 - The CC Database is the local SQLite database on the phone, which pulls all of the information that is saved between sessions. 

# Conclusion

I hope this was helpful in understanding the MVVM architecture a little better, and why it's important!

You can head on over to [part 2]({{ site.baseurl }}{% link _posts/2018-05-31-breaking-the-buzzwords-barrier-room-rx-repository.md %}) to learn more about the 'R' buzzwords.
