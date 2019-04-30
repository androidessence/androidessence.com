---
layout: post
author: adam
title: Fingerprint Authentication Tutorial
description: Step by step guide to creating a fingerprint authentication flow for Android.
modified: 2018-01-02
published: true
tags: [fingerprint]
categories: [android, tutorial]
---

When Android released version 6.0 Marshmallow (yes, a little outdated at this point), a whole slew of new developer APIs came with it. One that I've personally enjoyed as a consumer is [fingerprint authentication](https://developer.android.com/about/versions/marshmallow/android-6.0.html#fingerprint-authentication). I skimmed over the official docs, and even through their [Fingerprint Dialog Sample](https://developer.android.com/samples/FingerprintDialog/index.html) but had a difficult time following what was going on.

Eventually, though, I was able to recreate the flow. This post is going to be a step by step guide to integrating your own fingerprint dialog in your Android application. 

<!--more-->

## Dialog View

When adding fingerprint authentication to your app, you have full control over how that flow should be designed. However, there is a [material design specification](https://material.io/guidelines/patterns/fingerprint.html#) for the fingerprint authentication flow. This page dicusses certain text that should be included, what the behavior looks like, and the icon that you should use. This is necessary because we, as developers, want to make sure our apps are providing a consistent flow compared to other apps doing similar work.

From the above specification, you'll notice that Google has supplied the red lines for creating a dialog:

![fingerprint Authentication Dialog](/images/fingerprint/redlines.png)

For the purposes of this tutorial, we won't be including a "use password" fallback, and will focus solely on the fingerprint authentication. Using the above red lines, I was able to create an XML file for the dialog using a ConstraintLayout. The following code is inside my `dialog_fingerprint.xml` file, which you can see on [GitHub](https://github.com/androidessence/FingerprintDialogSample/blob/master/app/src/main/res/layout/dialog_fingerprint.xml).

## Fingerprint Controller

Before writing out the actual dialog, we need to create a helper class that handles some of the fingerprint authentication flow. While much of this code could be written inside the DialogFragment class, it is better to abstract it out for separation of concerns, and only give the controller the info that it needs. Let's look at how we define our `FingerprintController.kt` class:

```kotlin
	class FingerprintController(
		private val fingerprintManager: FingerprintManagerCompat,
		private val callback: Callback,
		private val title: TextView,
		private val subtitle: TextView,
		private val errorText: TextView,
		private val icon: ImageView) : FingerprintManagerCompat.AuthenticationCallback() { 

		interface Callback {
			fun onAuthenticated()
			fun onError()
		}
	}
```

This looks overwhelming, so we can break it down by each bit:
 * [FingerprintManagerCompat](https://developer.android.com/reference/android/support/v4/hardware/fingerprint/FingerprintManagerCompat.html) class is a class used to connect with the fingerprint hardware. There is a [FingerprintManager](https://developer.android.com/reference/android/hardware/fingerprint/FingerprintManager.html) class, but as discussed in the beginning these APIs were only added in API 23 (Marshmallow), and so it is best to use the `Compat` class to better support older Android versions.
 * Callback is an interface that lets the view know when fingerprint authentication is completed.
 * title, subtitle, errorText, and icon are all views that may be modified throughout the fingerprint authentication flow, based on successful or unsuccessful attempts.
 * Our controller extends the [`AuthenticaitonCallback()`](https://developer.android.com/reference/android/support/v4/hardware/fingerprint/FingerprintManagerCompat.AuthenticationCallback.html) class which handles the various callback possibilities from the authentication attempts.

### Class Properties

In addition to all of the properties accepted via the constructor, there are some values that we need to create inside the class itself. We need:
 * A cancellation signal to alert us when authentication should be cancelled.
 * A boolean flag to know if cancellation occured by our controller or some other source.
 * A boolean flag to tell us if this device should support fingerprint authentication.
 * A runnable that resets the dialog to its initial state. We'll run this when the class is instantiated.

Here is what our class looks like now:

```kotlin
	class FingerprintController(...) : FingerprintManagerCompat.AuthenticationCallback() {
		/**
		 * Helper variable to get the context from one of the views. The view used is arbitrary. 
		 */
		private val context: Context
			get() = errorText.context

		private var cancellationSignal: CancellationSignal? = null
		private var selfCancelled = false

		private val isFingerprintAuthAvailable: Boolean
			get() = fingerprintManager.isHardwareDetected && fingerprintManager.hasEnrolledFingerprints()

		private val resetErrorTextRunnable: Runnable = Runnable {
			errorText.setTextColor(ContextCompat.getColor(context, R.color.hint_color))
			errorText.text = context.getString(R.string.touch_sensor)
			icon.setImageResource(R.drawable.ic_fingerprint_white_24dp)
		}

		init {
			errorText.post(resetErrorTextRunnable)
		}

		interface Callback { }
	}
```

### Listening For Authentication

Now that we've defined all of the fields required for our controller, we can tell it to start listening for authentication and to stop. In order to do so, we'll need a crypto object that we'll be authenticating via fingerprint, but more on that later. Our start listening method will check for hardware support, and if it's there, resets the cancellation signal and flag, and called the [authenticate](https://developer.android.com/reference/android/support/v4/hardware/fingerprint/FingerprintManagerCompat.html#authenticate(android.support.v4.hardware.fingerprint.FingerprintManagerCompat.CryptoObject,%20int,%20android.support.v4.os.CancellationSignal,%20android.support.v4.hardware.fingerprint.FingerprintManagerCompat.AuthenticationCallback,%20android.os.Handler)) method of the fingerprint manager. When we want to stop listening, we check if we have a cancellation signal by using Kotlin's `let` delgate, so that we only execute the block of code if the signal is not null. Here is what all of that looks like:

```kotlin
    fun startListening(cryptoObject: FingerprintManagerCompat.CryptoObject) {
        if (!isFingerprintAuthAvailable) return

        cancellationSignal = CancellationSignal()
        selfCancelled = false
        fingerprintManager.authenticate(cryptoObject, 0, cancellationSignal, this, null)
    }

    fun stopListening() {
        cancellationSignal?.let {
            selfCancelled = true
            it.cancel()
            cancellationSignal = null
        }
    }
```

You may notice that the code simply returns from the start listening method if there is no support for fingerprint. Later, when we add this dialog to our activity, we will want to do a check then as well so we don't even prompt the user for fingerprint if it's unnecessary. This will also come later.

### Handle Fingerprint Authentication Callbacks

As discussed in the first part of this section, our controller class extends `FingerprintManagerCompat.AuthenticationCallback`. There are four methods in this class that we will implement, and another utility method. These are:
 * onAuthenticationError - called when a fatal error for fingerprint authentication occurs.
 * onAuthenticationHelp - called when an error occurs, but it wasn't a fatal exception. This will be things like moving your finger too fast, or the sensor being dirty.
 * onAuthenticationFailed - called when a fingerprint is valid, but not one of the enrolled fingerprints on the device.
 * onAuthenticationSucceeded - called when a valid and enrolled fingerprint was scanned.
 * showError - this is a helper method we will use to display an error to the user. This error will be displayed for about a second and a half before resetting the view. 

 Sample code for each one may look like this. Everything below is inside our `FingerprintController.kt` file:

```kotlin
	private fun showError(text: CharSequence?) {
		icon.setImageResource(R.drawable.ic_error_white_24dp)
		errorText.text = text
		errorText.setTextColor(ContextCompat.getColor(errorText.context, R.color.warning_color))
		errorText.removeCallbacks(resetErrorTextRunnable)
		errorText.postDelayed(resetErrorTextRunnable, ERROR_TIMEOUT_MILLIS)
	}

	override fun onAuthenticationError(errMsgId: Int, errString: CharSequence?) {
		if (!selfCancelled) {
			showError(errString)
			icon.postDelayed({
				callback.onError()
			}, ERROR_TIMEOUT_MILLIS)
		}
	}

	override fun onAuthenticationSucceeded(result: FingerprintManagerCompat.AuthenticationResult?) {
		errorText.removeCallbacks(resetErrorTextRunnable)
		icon.setImageResource(R.drawable.ic_check_white_24dp)
		errorText.setTextColor(ContextCompat.getColor(errorText.context, R.color.success_color))
		errorText.text = errorText.context.getString(R.string.fingerprint_recognized)
		icon.postDelayed({
			callback.onAuthenticated()
		}, SUCCESS_DELAY_MILLIS)
	}

	override fun onAuthenticationHelp(helpMsgId: Int, helpString: CharSequence?) {
		showError(helpString)
	}

	override fun onAuthenticationFailed() {
		showError(errorText.context.getString(R.string.fingerprint_not_recognized))
	}

	companion object {
		private val ERROR_TIMEOUT_MILLIS = 1600L
		private val SUCCESS_DELAY_MILLIS = 1300L
	}
```

## Fingerprint Dialog

Once we've created our XML layout, and our FingerprintController class, we can begin to setup the dialog. Let's get the generalized code, not specific to new fingerprint functionality, out of the way first. We need:
 * A DialogFragment that implements our `FingerprintController.Callback` interface.
 * A reference to a `FingerprintController` for use inside the dialog.
 * A new instance method that creates this dialog with a specific title and subtitle, initialized once dialog is created.

To save space on the blog itself, I'll be omitting that code but you can find it [on GitHub](https://github.com/androidessence/FingerprintDialogSample/blob/master/app/src/main/java/com/androidessence/fingerprintdialogsample/FingerprintDialog.kt).

### Cryptography Components

So, admittedly, this is where I struggled myself to understand what was happening. I followed along with the [FingerprintDialog](https://github.com/googlesamples/android-FingerprintDialog/) samples from Google, and learned that you're required to have a `FingerprintManagerCompat.CryptoObject` to supply to the FingerprintManager. The purpose of this CryptoObject is to manage a symmetric key which gets assigned to the Android KeyStore, which can only be used once the user has actually authenticated with a fingerprint. That is how you know the user has biometrically authenticated themselves. You can learn more about this flow, and specifically how to connect it to a backend, in this [great talk by Ben Oberkfell from American Express](https://www.youtube.com/watch?v=_OiyXGTdR3Q). Let's lay out the steps of what we need to do, and follow it with the code required:
 * Create a class level variable for the crypto object, the Android keystore, and a key generator.
 * In the DialogFragment's `onCreate()` method, we will get an instance of the keystore and the generator.
 * Once we have those instances, we'll create a key by some default name to be stored in the keystore.
 * When we have our key, we can initialize our crypto object.
 * Last, we setup our controller to start listening in `onResume()` and stop in `onPause()`. 

Here is what our class looks like now, with code removed for simplicity:

```kotlin
	class FingerprintDialog : DialogFragment(), FingerprintController.Callback {

	    private var cryptoObject: FingerprintManagerCompat.CryptoObject? = null
	    private var keyStore: KeyStore? = null
	    private var keyGenerator: KeyGenerator? = null

	    override fun onCreate(savedInstanceState: Bundle?) {
	        super.onCreate(savedInstanceState)

	        try {
	            keyStore = KeyStore.getInstance("AndroidKeyStore")
	        } catch (...) {}

	        try {
	            keyGenerator = KeyGenerator
	                    .getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
	        } catch (...) {}

	        createKey(DEFAULT_KEY_NAME, false)

	        val defaultCipher: Cipher
	        try {
	            defaultCipher = Cipher.getInstance(KeyProperties.KEY_ALGORITHM_AES + "/"
	                    + KeyProperties.BLOCK_MODE_CBC + "/"
	                    + KeyProperties.ENCRYPTION_PADDING_PKCS7)
	        } catch (...) {}

	        if (initCipher(defaultCipher, DEFAULT_KEY_NAME)) {
	            cryptoObject = FingerprintManagerCompat.CryptoObject(defaultCipher)
	        }
	    }

	    override fun onResume() {
	        super.onResume()

	        dialog?.window?.setLayout(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)
	        cryptoObject?.let {
	            controller.startListening(it)
	        }
	    }

	    override fun onPause() {
	        super.onPause()
	        controller.stopListening()
	    }

	    /**
	     * Lifted code from the Google samples - https://github.com/googlesamples/android-FingerprintDialog/blob/master/kotlinApp/app/src/main/java/com/example/android/fingerprintdialog/MainActivity.kt
	     *
	     * Initialize the [Cipher] instance with the created key in the
	     * [.createKey] method.
	     *
	     * @param keyName the key name to init the cipher
	     * @return `true` if initialization is successful, `false` if the lock screen has
	     * been disabled or reset after the key was generated, or if a fingerprint got enrolled after
	     * the key was generated.
	     */
	    private fun initCipher(cipher: Cipher, keyName: String): Boolean {
	        try {
	            keyStore?.load(null)
	            val key = keyStore?.getKey(keyName, null) as SecretKey
	            cipher.init(Cipher.ENCRYPT_MODE, key)
	            return true
	        } catch (...) {}
	    }

	    /**
	     * Lifted code from the Google Samples - https://github.com/googlesamples/android-FingerprintDialog/blob/master/kotlinApp/app/src/main/java/com/example/android/fingerprintdialog/MainActivity.kt
	     *
	     * Creates a symmetric key in the Android Key Store which can only be used after the user has
	     * authenticated with fingerprint.
	     *
	     * @param keyName the name of the key to be created
	     * @param invalidatedByBiometricEnrollment if `false` is passed, the created key will not
	     * be invalidated even if a new fingerprint is enrolled.
	     * The default value is `true`, so passing
	     * `true` doesn't change the behavior
	     * (the key will be invalidated if a new fingerprint is
	     * enrolled.). Note that this parameter is only valid if
	     * the app works on Android N developer preview.
	     */
	    private fun createKey(keyName: String, invalidatedByBiometricEnrollment: Boolean) {
	        // The enrolling flow for fingerprint. This is where you ask the user to set up fingerprint
	        // for your flow. Use of keys is necessary if you need to know if the set of
	        // enrolled fingerprints has changed.
	        try {
	            keyStore?.load(null)
	            // Set the alias of the entry in Android KeyStore where the key will appear
	            // and the constrains (purposes) in the constructor of the Builder

	            val builder = KeyGenParameterSpec.Builder(keyName,
	                    KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
	                    .setBlockModes(KeyProperties.BLOCK_MODE_CBC)
	                    // Require the user to authenticate with a fingerprint to authorize every use
	                    // of the key
	                    .setUserAuthenticationRequired(true)
	                    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_PKCS7)

	            // This is a workaround to avoid crashes on devices whose API level is < 24
	            // because KeyGenParameterSpec.Builder#setInvalidatedByBiometricEnrollment is only
	            // visible on API level +24.
	            // Ideally there should be a compat library for KeyGenParameterSpec.Builder but
	            // which isn't available yet.
	            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
	                builder.setInvalidatedByBiometricEnrollment(invalidatedByBiometricEnrollment)
	            }
	            keyGenerator?.init(builder.build())
	            keyGenerator?.generateKey()
	        } catch (...) {}

	    }

	    companion object {
	        private val DEFAULT_KEY_NAME = "default_key"
	    }
	}
```

## Summary

Congrats! If you've made it this far, you made it through the toughest chunk of fingerprint authentication code. The only remaining step is to setup your [MainActivity](https://github.com/androidessence/FingerprintDialogSample/blob/master/app/src/main/java/com/androidessence/fingerprintdialogsample/MainActivity.kt) to launch this dialog.

As usual, you can find a full sample application for this post on [GitHub](https://github.com/androidessence/FingerprintDialogSample), where everything is documented. If you have questions or improvements, please leave them in the comments!