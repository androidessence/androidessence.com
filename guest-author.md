---
layout: page
title: Become An Android Essence Author
share: true
---

# Write For The Community!

Android Essence is proud to anounce we are expanding to support multiple authors! When I launched this site in fall of 2015, it started as my own personal wordpress blog. As time went on and the tutorials and posts gained traction, I realized this was a website that was creating a positive impact on the community and the people around me.

Which is why I want to open it up to giving others an opportunity to put their own content out there! If you've ever wanted to contribute to the Android community - whether it's a tutorial, or a discussion on a tough bug you found - I want Android Essence to be your outlet for that. 

I've been proud to keep Android Essence ad free since the beginning. Sharing content has never been about making money for me - it's about driving the community forward. If you're interested in being part of that drive, read on to find out how you can contribute.

## Repository

The Android Essence website is built in [Jekyll](https://jekyllrb.com/), a static website generator, and the repository can be found on [GitHub](https://github.com/androidessence/androidessence.com). If you're unfamiliar with Jekyll, I encourage you to explore it before contributing. A guide on building the website locally is included in the testing section.

Before you can contribute, you'll need to [fork the repository](https://help.github.com/articles/fork-a-repo/) to your own GitHub account.

## Adding An Author Profile

Within the repository, in the `_data` directory we have an `authors.yml` file. In order to reference your author from the post, you'll need to create a record here. The format is as follows:

```markdown
	adam:
		name: Adam McNeilly
		site: https://www.adammcneilly.com
		avatar: images/avatars/adam.jpg
		bio: "Software Engineer and Android development enthusiast."
		github: AdamMc331
		twitter: AdamMc331
```

If you don't want to supply a GitHub or Twitter handle you don't have to - the follow buttons will simply be excluded. All that is required is that you replace `adam:` with your name, and replace all of the necessary attributes with your own information.

## Writing A Post

All of the posts are saved within the `_posts` directory. I'll break down the format here, but I encourage you to look at previous examples.

The file name for each post should have the following format:

> `YYYY-MM-DD-title-with-dashes.md`

Within each post, you'll have the [front matter](https://jekyllrb.com/docs/frontmatter/) followed by your content, like this:

```markdown
	---
	layout: post
	author: adam
	title: My post title.
	description: My Post Description.
	modified: 2017-10-19
	published: true
	tags: [tutorial]
	categories: [android]
	---

	This is my blog post. Everything I write in the beginning here will appear on the home page wherever this post is in the list.

	<!--more-->

	Everything after that special comment tag won't appear on the home page. It will appear on the post itself, but on the home page you will see a "Read More..." button where that comment sits.
```

Some notes about each post and the front matter
 * The layout must be "post".
 * The author field is whatever your author name is in `_data/authors.yml`.
 * Tags are searchable on the Android Essence website.
 * Categories were part of this jekyll theme, not currently used, but I encourage adding them if that ever changes.
 * The `<!--more-->` comment is case sensitive. It must match that exactly for the read more button to appear.
 * The "published" attribute determines whether or not your post appears on the main page.

## Testing

Once you've added yourself as an author, and written your post, it's time to test.

First, go into the `_config.yml` file and update the url to point to your localhost, instead of the production site, by uncommenting this first line and commenting the second:

```markdown
	# url: http://localhost:4000
	url: https://androidessence.com
```

I recommend following the [Jekyll Quick Start](https://jekyllrb.com/docs/quickstart/) guide to install everything, then navigate to the website repo and run the command `bundle exec jekyll serve`. From there you'll be able to navigate to `http://localhost:4000` in your browser and view the site. Here, you should follow each step:

1. Proofread your post
2. Verify that your post on the home page shows the correct author name, and only shows a snippet of your post.
3. At the bottom of your post, verify the "About The Author" section looks and behaves as you expected it to.

## Pull Requests

Once you've done all that, commit your code and [submit a pull request](https://help.github.com/articles/about-pull-requests/) to the main repository. From there I will be reviewing all new content, and merging it myself. I will also handle updating the website once posts are merged. If you have specific concerns about a post going live on a certain date, or any other considerations, please leave them in your PR comments.

## Updates

You can find all status updates related to the guest author feature on the [official blog post]({{ site.baseurl }}{% link _posts/2017-10-21-guest-authors-wanted.md %}) for it. 
