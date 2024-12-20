---
layout: post
author: adam
title: "Wrapping Android XR For KMP"
modified: 2024-12-19
published: true
tags: [xr, kmp]
categories: [android]
---

Recently I've been finding myself immersed in [Compose Multiplatform](https://github.com/JetBrains/compose-multiplatform), a way to build cross platform mobile applications with Jetpack Compose UI. Of course, when Android announced their extended reality framework [Android XR](https://www.android.com/xr/) this week, I initially worried I wouldn't be able to try it out in my newest side projects.

However, one of the biggest benefits of KMP is how seamless it is to provide platform specific functionality for your shared code. So let's do that with Android XR. 

<!--more-->

---

## Dependency Setup

With KMP, we can have [source set specific dependencies](https://kotlinlang.org/docs/multiplatform-add-dependencies.html#library-used-in-specific-source-sets), so the XR dependencies are only applied to the Android target of our shared module:

```kotlin
// shared/build.gradle.kts
sourceSets {
    androidMain.dependencies {
        implementation(libs.androidx.xr.compose)
        implementation(libs.androidx.xr.compose.material3)
        implementation(libs.androidx.xr.scenecore)
    }
}
``` 

## Common Functionality

When we want platform specific functionality, like XR features, we start by defining what the common functionality is. In our XR case for now, we want the following behaviors:

1. A way to request home space mode (when apps are sharing space with each other in Android XR)
2. A way to request full space mode (when an app is full focus and only visible app in Android XR)
3. A way to determine if spacial ui is enabled (in practice, this means if the app is in full space mode)

A great way to model this is via an interface, which allows us to provide platform specific implementations. On Android, we'll use the XR dependencies. On iOS, we'll do nothing for now (though you can begin to see how this might expand to [AVP](https://developer.apple.com/visionos/) support).

```kotlin
interface XRSession {
	/**
	 * This getter has to be composable, because on the Android side 
	 * we'll reference `LocalSpatialCapabilities` Composition local.
	 * 
	 * https://developer.android.com/develop/xr/jetpack-xr-sdk/check-spatial-capabilities
	 */ 
    val isSpatialUiEnabled: Boolean
        @Composable get

    fun requestHomeSpaceMode()

    fun requestFullSpaceMode()
}
```

## Android Implementation

We can implement our own `AndroidXRSession` that is a wrapper around a scenecore Session entity. 

```kotlin
class AndroidXRSession(
    val xrSession: Session, // from the scenecore dependency
) : XRSession {
    override val isSpatialUiEnabled: Boolean
        @Composable get() = LocalSpatialCapabilities.current.isSpatialUiEnabled

    override fun requestHomeSpaceMode() {
        xrSession.requestHomeSpaceMode()
    }

    override fun requestFullSpaceMode() {
        xrSession.requestFullSpaceMode()
    }
}
```

## Expect/Actual

Now that we've defined our interface and a platform specific implementation, we need a way to reference our `XRSession` in common code. We can do that by writing [expect/actual declarations](https://kotlinlang.org/docs/multiplatform-expect-actual.html).

From each platform, we expect a function to give us an XRSession. The return type here is nullable for two reasons:

1. For now, we're ignoring XR on iOS. 
2. Even on Android, we might not have an XRSession, if we're running on a phone for example.

```kotlin
// XRSession.kt
@Composable
expect fun currentXRSession(): XRSession?
```

On iOS, we can be straight forward and return null:

```kotlin
// XRSession.ios.kt
@Composable
actual fun currentXRSession(): XRSession? {
    return null
}
```

On Android, we check if we have a `LocalSession` and if so, map that to our implementation from above:

```kotlin
// XRSession.android.kt
@Composable
actual fun currentXRSession(): XRSession? {
    return LocalSession.current?.let(::AndroidXRSession)
}
```

## CompositionLocal

Similar to how [Jetpack Compose for XR](https://developer.android.com/develop/xr/jetpack-xr-sdk/develop-ui) framework works, we can supply our own XRSession using CompositionLocals. 

We define the CompositionLocal with a default value of null:

```kotlin
val LocalXRSession = compositionLocalOf<XRSession?> {
    null
}
```

Then, in our root Composable function, we can provide the current session we get from our expect/actual functions:

```kotlin
@Composable
fun MyApp() {
    CompositionLocalProvider(
        LocalXRSession provides currentXRSession(),
    ) {
    	// App content
    }
}
```

## Further Details

From here, any time we want to use our XRSession, we can call `LocalXRSession.current` and use it as if we were following the Android XR docs directly. For example, we can create a `SpatialModeSwitchFAB` to switch between home and full space mode, entirely in our common code:

```kotlin
@Composable
fun SpatialModeSwitchFAB(
    modifier: Modifier = Modifier,
) {
    val xrSession = LocalXRSession.current ?: return

    val hasSpatialUi = xrSession.isSpatialUiEnabled

    FloatingActionButton(
        onClick = {
            if (hasSpatialUi) {
                xrSession.requestHomeSpaceMode()
            } else {
                xrSession.requestFullSpaceMode()
            }
        },
    ) {
    	// ...
    }
}
```

And now, our entirely Compose Multiplatform application has the ability to switch between home and full space mode on an Android XR emulator:

![](assets/xr/XRSwitch.gif)

For more Android XR and other general Android content, make sure you're following me over on [Twitch](https://twitch.tv/adammc) and [YouTube](https://youtube.com/adammcneilly)!