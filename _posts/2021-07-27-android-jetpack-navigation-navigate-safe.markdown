---
layout: post
title: "Navigate without crashes using Android Jetpack Navigation"
date: 2021-07-26 19:00:00 +0300
image: /assets/jetpack-safe-helmets.jpg
description: "Do not crash an app during navigation. Navigate safely."
postImage:
  src: jetpack-safe-helmets
  alt: Helmets makes navigation safe
twitterLink: https://twitter.com/VysotskiVadim/status/1420029690727866375
---

## Introduction

Android Jetpack navigation throws an exception if something is wrong.
It's either works or doesn't.
There's no middle state of partially working.
Sometimes it's useful, sometimes not.

I'm okay with unhandled exceptions during development or testing.
But it's not acceptable for production.
Imagine a user opens a settings screen and the app crashes.

I will share how I used Jetpack Navigation on my last project.
My application doesn't crash even if something goes wrong.

The given approach also saved us from the issues related to double navigation ```java.lang.IllegalArgumentException: navigation destination XXX is unknown to this NavController```.
I can quickly click a few times on the button, which triggers navigation, but the app navigates only one time.

## Safe navigation

### Error handling

To avoid crashes handle all exceptions from Jetpack Navigation library.
I use Jetpack navigation via wrapper functions, which handle all errors.

```kotlin
@SuppressLint("UnsafeNavigation")
fun NavController.navigateSafe(@IdRes action: Int, args: Bundle? = null): Boolean {
    return try {
        navigate(action, args)
        true
    } catch (t: Throwable) {
        Log.e(NAVIGATION_SAFE_TAG, "navigation error for action $action")
        false
    }
}
```

What should an app do in case of a navigation error?
You have two options: ignore errors or send them to the crash tracker.

I used to send errors as non-fatal crashes in crashlytics.
It didn't go well.
We got too many false-positive errors.
Every time user clicked simultaneously on two buttons the app sent not-fatal error ```java.lang.IllegalArgumentException: navigation destination XXX is unknown to this NavController```.
And there is no stable way to distinguish potentially fixable error from a junk.

I decided to ignore any errors during navigation.
I just log it and this is it.
If the app can't navigate - it does nothing.

### Always use navigateSafe

It's hard to remember that you and your teammates should prefer `navigateSafe` wrapper over default `navigate`.
Linter can remind us.

{% include image.html src="safe-jetpack-navigation-linter" alt="Linter rule highlight error" %}

Check out [a linter rule on the github](https://github.com/VysotskiVadim/jetpack-navigation-example/blob/master/lintrules/src/main/java/dev/vadzimv/jetpack/navigation/lintrules/UnsafeNavigationDetector.kt)
and check out [the article about linter rules](https://proandroiddev.com/implementing-your-first-android-lint-rule-6e572383b292) if you aren't familiar with custom linter rules.

It's a lot of code, but don't worry it's simple.

Step 1: tell the linter method name that you'd like to inspect.
```kotlin
override fun getApplicableMethodNames(): List<String> = listOf("navigate")
```
Step 2: report error if method `navigate` is member of `androidx.navigation.NavController`.
```kotlin
if (evaluator.isMemberInClass(method, "androidx.navigation.NavController")) {
    context.report(...)
}
```
### Prefer actions to directions

I always use actions instead of directions.
It helps handle double navigation.
Consider the following example.

You have a screen with 2 buttons.
`settingsButton` navigates the user to the settings screen.
`homeButton` navigates the user to the home screen.

Let's try navigating using destinations.
```kotlin
settingsButton.setOnClickListener {
    findNavController().navigateSafe(R.id.settingsFragment)
}
homeButton.setOnClickListener {
    findNavController().navigateSafe(R.id.homeFragment)
}
```

What does happen when a user clicks two buttons simultaneously?
The app navigates two times, to `settingsFragment` and then to `homeFragment`, or other way around.
I.e. the user has both screens in the back stack.

Now replace destinations by actions and try click 2 buttons simultaneously again.
```kotlin
settingsButton.setOnClickListener {
    findNavController().navigateSafe(R.id.action_navigation_notifications_to_settingsFragment)
}
homeButton.setOnClickListener {
    findNavController().navigateSafe(R.id.action_navigation_notifications_to_homeFragment)
}
```
The first of two simultaneous clicks causes navigation.
The second click causes `IllegalStateException` which is handled by `navigateSafe` wrapper.
I.e. the first navigation wins, no crashes.

## Summary

Think about prod users.
You implement your app for them.
Don't let it crash if some screen can't be open.

## Links

* [Post image](https://flic.kr/p/a2rJ6x)
* [Example project](https://github.com/VysotskiVadim/jetpack-navigation-example)