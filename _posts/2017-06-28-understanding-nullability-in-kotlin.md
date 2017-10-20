---
layout: post
author: adam
title: Understanding Nullability In Kotlin
description: Explaining the benefits of nullability within Kotlin's type system.
modified: 2017-06-28
published: true
tags: [kotlin]
categories: [android]
---

Every Java programmer has faced the dreaded `NullPointerException` at some point in their life. Sometimes it's your fault, sometimes it's a pesky race condition. Regardless, it's a head ache and generally leads to a ton of `if (myVariable != null) { }` conditions all over your code. However, the latest craze [Kotlin](https://kotlinlang.org) can help with that too. Kotlin introduced [null safety](https://kotlinlang.org/docs/reference/null-safety.html) into its type system, with the potential of removing all NPEs. 

This post is both going to review the official docs linked above, as well as provide some common tips and tricks to work with the nullability - something that is new in this language for many Java programmers.

<!--more-->

# Nullable and Non-Nullable Types

First, let's explain what we mean when we keep saying Kotlin has nullability in the type system. In Java, we know that an object can either have a value or be null. As discussed in the first paragraph, this can lead to trouble. In Kotlin, however, we have the ability to say a type will never be null. We do that by simplify specifying the type as is. If we want to declare a type as nullable, we append a question mark at the end. Here is an example:

```kotlin
	// String is a non-nullable type, so if we tried to assign null to it, there would be a compilation error
	val a: String = "test"
	a = null // Compilation error

	// String? is a nullable type, so if we tried to assign null to it, it would accept it
	val b: String? = "test"
	b = null // Okay
```

Note: If you are working with Java interrop, there's still a risk of the first option being able to throw an NPE. We'll discuss that later under platform types.

# Checking For Nullability

There are a few ways to check and handle nullability on a type. Let's discuss each of them using the `String?` example from above, where we want to use the strings length to control flow.

## Explicit Checks

Just like Java, we could explicitly check for null:

```kotlin
	val b: String? = "test"

	if (b != null && b.length > 0) {
		// Do something with the string
	} else {
		// String is null, handle error
	}
```

## Safe Operator

If we don't want to deal with the verbosity of the above code, we can use the safe operator (`?.`) provided by Kotlin. This operator will either return the value if non-null, or null if it was unable to be read. Here is what the code would look like with and without the safe operator:

```kotlin
	val length: Int? = if (b != null) b.length else null
	val length: Int? = b?.length
```

As you can see, the safe operator makes the code a lot more concise. This becomes even more useful when you have a number of nested objects, and you are able to chain multiple safe operator calls together.

## Elvis Operator

In Java, we may also be used to doing an explicit check for null and returning another value if it's not found. For example, you may have written code like this in your lifetime:

```java
	return (myValue != null) ? myValue : -1;
```

Kotlin provides an elvis operator (`?:`) which works similar to the safe operator. However, instead of returning null, it'll return the default value that you supply it. The above Java code would be written in Kotlin as:

```kotlin
	return myValue ?: -1
```

Using that for the example further up regarding string length:

```kotlin
	val length: Int = b?.length ?: 0
	if (length > 0) doSomething() else fail()
```

## Safe Class Casting

Similar to the safe operator for types, we can use the `as?` operator to safely cast objects. This will just return null if the cast is unsuccessful:

```kotlin
	val intVal: Int? = myValue as? Int
```

## Do Not Enter

If, for some reason, you insist on having a `NullPointerException`, you can use the [!!](https://kotlinlang.org/docs/reference/null-safety.html#the--operator) operator that will throw an NPE if you do try to access a null object.

# Platform Types

I mentioned above that even if you declare a type as non-nullable, you could have an NPE if it was interopping with Java code. Well, [as the Kotlin docs state](https://kotlinlang.org/docs/reference/java-interop.html#null-safety-and-platform-types), any object in Java could be null, so carrying over its type safety in Kotlin is impractical. For that reason, any object returned by a Java library is referred to as a "Platform Type", for these types null checks are relaxed as nullability cannot be guaranteed.

However, if the Java library uses the `@Nullable` or `@NotNull` annotations, the Kotlin compiler will be able to infer the type safety to use.

# Things To Look For

After reading through this, don't just go through all of your Kotlin code and make everything nullable thinking that you've avoided all NPEs and you're done. There are a couple things you should consider about a type before making it optional:

1. Does it make sense for this type to ever be null? If you expect that you'll always have the value, then make it non-nullable and save yourself from unnecessary safe and elvis operator calls.
2. Does this value come from, or ever get set by, an external Java library? If your answer to that is yes, you may be better off using an optional. If the external library doesn't guarantee safety, your code shouldn't either.

## Lazy Delegated Property

You should also be aware of some of Kotlin's [delegated properties](https://kotlinlang.org/docs/reference/delegated-properties.html), especially the `lazy()` method. Before we talk about what it does, let's explain the problem we're trying to solve. Consider the following fragment code:

```kotlin
	class MyFragment : Fragment() {
		private var manager: MyAPIManager? = null

		@Override
		public void onCreate(savedInstanceState: Bundle?) {
			super.onCreate(savedInstanceState)

			manager = MyAPIManager(context)
			manager.authorize()
		}
	}
```

In this case, we define a manager object at the class level. We have to give it an initial value, and we're forced to give it null. That's because we don't have access to the context at this point, and so we have no choice but to give it a value inside of our onCreate method. There are two consequences of this approach, despite how logical it may seem:

1. We're forced to declare manager as nullable, even though we *know* that it won't be null once it gets assigned in onCreate.
2. We have to label it a `var` (making it mutable) so we can assign it inside onCreate, even though we really know that it should never be reassigned to anything else.

Solution? [The lazy property](https://kotlinlang.org/docs/reference/delegated-properties.html#lazy). This property takes in a lambda, which is executed the first time the property is accessed. After that, it will return the value that was assigned to it. This way we can declare the property as immutable, and non-null, so that as long as the fragment is created before we access it the first time, we'll avoid the two problems mentioned above. Here is what the code looks like now:

```kotlin
	class MyFragment : Fragment() {
		private val manager: MyAPIManager by lazy {
			MyAPIManager(context)
		}

		@Override
		public void onCreate(savedInstanceState: Bundle?) {
			super.onCreate(savedInstanceState)

			manager.authorize()
		}
	}
```

## Lateinit Property

Similar to lazy, we can use the [lateinit](https://kotlinlang.org/docs/reference/properties.html#late-initialized-properties) keyword to define properties that will be initliazed outside of the constructor. The syntax looks similar to lazy, but you still need to initialize it at some point:

```kotlin
    class MyFragment : Fragment() {
		lateinit var manager: MyAPIManager

		@Override
		public void onCreate(savedInstanceState: Bundle?) {
			super.onCreate(savedInstanceState)

                        manager = MyAPIManager(context)
			manager.authorize()
		}
	}
```

This way, we can still declare the variable as non-nullable, and assign it *late*r when the time is right. However, we still are unable to access this field before it's assigned. If we do, though, an exception is thrown explaining that the field hasn't been initialized yet, which is slightly different from an NPE.


I hope you found this a useful guide to Kotlin's nullability, and that you are able to see the benefits it provides, when to use it, and how to use it to your advantage. Have you run into any snags using nullability in Kotlin before, or know of other tips beyond the lazy/lateinit properties? Let us know in the comments!