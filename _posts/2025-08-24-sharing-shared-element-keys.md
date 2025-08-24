---
layout: post
author: adam
title: "Sharing Shared Element Keys"
modified: 2025-08-24
published: true
tags: [compose]
categories: [android]
---

[Shared Element Transitions](https://developer.android.com/develop/ui/compose/animation/shared-elements) are a fantastic way to add visual flair to your applications. They allow us to animate content between two separate screens. One complication is sharing keys for these components across different screens in a consistent manner. Let's explore one simple option. 

<!--more-->

## The Problem

To clarify the background, let's understand how we create shared element transitions in Compose. Imagine a screen showing a list of articles, with photos. When you click on the article card, we want that picture to grow or change into the shape that the image appears on the article detail page. 

To do that, we can leverage the following code snippet, on both the list and the detail page:

```kotlin
with(sharedTransitionScope) {
    AsyncImage(
        model = "article_image_url",
        modifier = Modifier
            .sharedElement(
                rememberSharedContentState(key = "article_image_$articleId"),
                animatedVisibilityScope = animatedVisibilityScope,
            ),
    )
    // ...
}
```

A couple key points:

1. The shared element key clarifies this is an article's image, and we include the article's ID in the string because there are multiple articles being displayed on the first screen. 
2. Since the key being used is a string, we run the risk of making typos between screens, or changing one and forgetting to change the other. 

A helpful way to solve this problem, while maintaining an explanatory shared element key name, is through a DisplayModel.

## UI Models/Display Models

A common practice among some apps is to create a specific data class used to define the UI representation. I personally call them DisplayModels. DisplayModels are an extension of our domain objects, but manipulated to represent what is displayed visually to a user. In the context of an article, we might show a special string representation of the title and author combined; or maybe you want to format a date for a specific UI context.

Consider the following domain class:

```kotlin
data class Article(
    val id: String,
    val title: String,
    val imageUrl: String,
    val url: String,
    val summary: String,
    val publishedAtUtc: Instant,
    val authors: List<Author>,
)
```

To support those UI representations, we can define a DisplayModel accordingly, and have a special constructor to map fields from an `Article`:

```kotlin
data class ArticleDisplayModel(
    val id: String,
    val title: AnnotatedString,
    val publishedAt: String,
    val image: ImageModel,
    val url: String,
    val summary: String,
) {
    constructor(
        article: Article,
    ) : this(
        id = article.id,
        title = buildAnnotatedString {
            withStyle(style = SpanStyle(fontWeight = FontWeight.Bold)) {
                append(article.title)
            }
            append(" | ")

            val authorNames = article.authors.joinToString(transform = Author::name)
            append(authorNames)
        },
        publishedAt = article.publishedAtUtc.format(publishedDateFormat),
        image = ImageModel.Remote(article.imageUrl),
        url = article.url,
        summary = article.summary,
    )
}
```

This approach may feel like boilerplate, but it's a good example of separation of concerns, keeping your domain/data layer separate from the UI representation. It also helps with testability, allowing you to create simple UI models for smaller components instead of creating a full complex version of your domain models to test your UI.

## Shared Element Keys

Since we have classes dedicated to the UI representation of a specific domain model, this is also the place to define anything that would impact their animations. So, we can take those hard coded keys from our early shared element transition examples, and put them on the DisplayModel:

```kotlin
data class ArticleDisplayModel(
	val id: String,
    // ...
) {
	val imageSharedElementKey = "article_image_$id"
    
    // ...
}
``` 

## Usage

Now, the call sites for the shared element transitions can leverage this consistent property, since the DisplayModel is used in both:

```kotlin
// ArticleSummaryCard.kt
fun ArticleSummaryCard(article: ArticleDisplayModel) {
	// ...
	ArticleImage(
		modifier = Modifier
			.sharedElement(
                rememberSharedContentState(key = article.imageSharedElementKey),
            ),
	)
}

// ArticleDetailContent.kt
fun ArticleDetailContent(article: ArticleDisplayModel) {
	// ...
	ArticleImage(
		modifier = Modifier
			.sharedElement(
                rememberSharedContentState(key = article.imageSharedElementKey),
            ),
	)
}
```

## Recap

While you technically don't need the idea of DisplayModels to leverage this approach, and can simply just add these computed properties onto any other class, this is a simple approach I've really come to enjoy. 
