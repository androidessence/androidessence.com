---
layout: post
author: adam
title: "Five Essential Developer Experience Concepts For Android"
modified: 2021-09-10
published: true
tags: [twitch, toa]
categories: [android]
---

In the most recent [live stream](https://www.youtube.com/watch?v=ePpbpLyYI1w) for Tasks Of Affirmation, we looked at creating a good developer experience. It's important to consider developers at the start of any new project, so we can ensure that over the lifetime of a project anyone who contributes can understand how to contribute, match any guidelines that a team has, and get up and running quickly. We also want to ensure that our codebase maintains a certain quality of formatting and a lack of code smells - and putting these checks in place before you even start prevents them from ever being introduced at all. 

Let's look at five developer experience concepts I consider essential to every project.

<!--more-->

During the live stream, we went over the following concepts to use in the TOA application:

* GitHub Actions
* Danger
* Ktlint
* Detekt
* Git Hooks

The live stream link will have video markers to each of these sections, where you can see the implementation in more detail. For this blog post, we're just going to highlight what each of these concepts are, and why they were necessary for the [TOA application](https://github.com/adammc331/toa).

## GitHub Actions

GitHub Actions is a way to [automate workflows](https://github.com/features/actions) for your GitHub projects. The possibilities are endless - verifying unit tests pass, running lint checks, publishing to the play store, sharing internal builds, generating documentation - any process that you want to automate can be done via GitHub Actions. 

You don't need to use Actions - there are many CI tools out there such as Bitrise, Circle CI, Jenkins, and others. We've opted to use GitHub Actions for this project because it is quick and easy to implement straight from GitHub, where this project is hosted anyways. 

### Setup

To create an Action, we need to add a new directory `.github/workflows` in our project. Inside of that, we can add a YAML file that configures the action. This can be edited via the GitHub UI which gives additional tooling for editing them. You can also feel free to use the [TOA Actions](https://github.com/AdamMc331/TOA/tree/development/.github/workflows) for inspiration.

## Danger

Danger is a tool for automating [common pull request chores](https://danger.systems/ruby). There are versions of Danger in multiple languages, but I prefer the Ruby gem. At first this may sound like the same thing as a GitHub Action, but it's a little more specific. Danger will run on Actions (or any other CI tool) to perform checks related to a Pull Request itself. An example of common Danger checks:

* Does this PR have labels?
* Does this PR have a description?
* Did we assign a reviewer?
* Is it too long? 
* Did we modify the CHANGELOG.md file?

Any meta information about a git diff or PR that you can think of, can be validated with Danger. 

### Setup

I would defer to the official [Danger](https://danger.systems/guides/getting_started.html) documentation for getting started, as it can vary a bit based on where you are running it and what source control tooling you have. 

For inspiration on some Danger checks, you can reference the [TOA Dangerfile](https://github.com/AdamMc331/TOA/blob/development/Dangerfile).

## Ktlint

Ktlint is a static analysis tool from Pinterest to [prevent bikeshedding](https://github.com/pinterest/ktlint). It has the ability to format your code and verify your code is formatted to match the Kotlin style guides. It is customizable and you can turn off certain rulesets if you wish, but I tend to use the out of the box configuration. 

### Setup

For Android, I prefer to use [this Gradle plugin](https://github.com/jlleitschuh/ktlint-gradle) to set up Ktlint in my Android applications. For an example of how to quickly integrate the plugin, you can check out [this pull request](https://github.com/AdamMc331/TOA/pull/30/files). 

## Detekt

A similar tool that provides a lot of benefit is [Detekt](https://github.com/detekt/detekt). This is another static analysis tool but rather than just checking for formatting, it checks for other common code smells such as:

* Magic numbers
* Functions that are too long
* Classes that have too many functions
* Functions that have too many parameters
* And more!

### Setup

To setup Detekt, you'll need to import the Gradle plugin, but also generate a config file that allows you to customize all of the different Detekt rules. You can actually generate this by running `./gradlew detektGenerateConfig` once you've added the plugin.

For an example of what that config file looks like, you can view it [here](https://github.com/AdamMc331/TOA/blob/development/config/detekt/detekt.yml). You can see a comparison in [this pull request](https://github.com/AdamMc331/TOA/pull/31) for the full integration.  

## Git Hooks

The last concept we discussed in this stream was git hooks. Hooks are a way to connect a script with a certain git action, like committing or pushing code. In my projects, I like to do the following checks:

1. Before committing code, format all of my changed Kotlin files. 
2. Before pushing code, ensure that we don't have any code smells. 

To create a git hook, you need to store it in the `.git/hooks` directory and name the file with the relevant hook like `pre-commit` and `pre-push`, respectively. However, this alone isn't super helpful because it doesn't allow you to easily share hooks between teammates.

Thankfully, I learned from a great friend and fellow GDE Sebastiano Poggi how to do this in [his amazing blog post](https://blog.sebastiano.dev/ooga-chaka-git-hooks-to-enforce-code-quality/), which I recommend reading for the details. In summary:

1. Add your git hooks to your repo, in files like `pre-commit.sh` and `pre-push.sh`. 
2. Write a gradle task that copies those files into the `.git/hooks` directory.
3. Write a gradle task to modify those files so that they are executable.
4. Have any new teammates on your project run these gradle tasks and now they can ensure their code is formatted, too!

Here is the relevant [PR](https://github.com/AdamMc331/TOA/pull/32) from the TOA stream. 

# Recap

Of course this list is not exhaustive, nor are all five of these essential to every team. They are, however, five tools that I've come to trust and find incredibly helpful in my projects. I hope that they've inspired some of you as well, and that you can find benefit adding these or similar tools to your project. If you have any other inspirations, let me know on Twitter, or join me live on [Twitch](https://twitch.tv/adammc331) every Wednesday! 
