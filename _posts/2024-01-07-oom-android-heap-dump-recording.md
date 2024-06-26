---
layout: post
title: "OutOfMemoryError on Android: heap dump recording"
date: 2024-01-07 12:00:00 +0100
image: /assets/green-robot-looking-for-a-leak.jpg
description: "Learn how to record a Java heap dump at the moment of OutOfMemoryError."
postImage:
  src: green-robot-looking-for-a-leak
  alt: A green robot which is looking for a leak
twitterLink: https://twitter.com/VysotskiVadim/status/1743957018275119489
linkedInLink: https://www.linkedin.com/posts/vadzim-vysotski-454b01210_outofmemoryerror-on-android-heap-dump-recording-activity-7149723307653709824-ORIS
---

## Introduction

```
java.lang.OutOfMemoryError: Failed to allocate a 32 byte allocation with 195576 free bytes and 190KB until OOM, target footprint 268435456, growth limit 268435456; giving up on allocation because <1% of heap free after GC.
```

Common debugging tools like logs and exception stack traces are often insufficient when dealing with an `OutOfMemoryError`.
The stack trace of the error merely indicates an allocation that occurred when the memory was already saturated, yet it fails to illuminate why the memory was at capacity at that particular moment.
Logs typically lack information regarding allocations and objects managed by the Garbage Collector. How can we effectively pinpoint the root cause of an `OutOfMemoryError`?

This article explains how to collect data essential to finding the cause of `OutOfMemoryError`: a heap dump at the moment of the last allocation, which failed with `OutOfMemoryError`.

I learned this technique while optimizing Java memory usage in [Mapbox Navigation SDK v2](https://github.com/mapbox/mapbox-navigation-android).
It was particularly useful when `OutOfMemoryError` occurred only in the customer's environment, and I wasn't able to reproduce them locally.

## Heap Dump recording

The most useful information for `OutOfMemoryError` troubleshooting is a Java heap dump. You can explore which objects consume the memory, how much is consumed by each object, and what stops the Garbage Collector from collecting them.

[The official guide](https://developer.android.com/studio/profile/memory-profiler) provides instructions on collecting heap dumps from Android Studio.
Simply launch the profiler and click "Record".
The challenge lies in determining when to record.
The heap dump should be taken when you observe a significant memory leak for it to be valuable.
The best time to record is when `OutOfMemoryError` occurs, a task not feasible for a human to perform manually.

Automating the recording of Java heap dumps is possible through the following Android API:
[UncaughtExceptionHandler](https://developer.android.com/reference/java/lang/Thread.UncaughtExceptionHandler)
and
[dumpHprofData](https://developer.android.com/reference/android/os/Debug#dumpHprofData(java.lang.String)).

### When to record a heap dump

Wait for the `OutOfMemoryError` in [UncaughtExceptionHandler](https://developer.android.com/reference/java/lang/Thread.UncaughtExceptionHandler), which is called when an unhandled exception happens:

```kotlin
Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
    if (throwable is OutOfMemoryError) {
        // record heap dump here
    }
    Log.e(LOG_TAG, "Unhandled exception", throwable)
    System.exit(1);
}
```

Third-party libraries like Firebase Crashlytics listen for unhandled exceptions using the same mechanism.
They could override the default uncaught exception handler.
Be careful if you use them. Find a guide that explains how to set a custom `UncaughtExceptionHandler` together with your library.


### How to record a heap dump

Call [dumpHprofData](https://developer.android.com/reference/android/os/Debug#dumpHprofData(java.lang.String)) passing the location where the heap dump should be recorded:


```kotlin
val heapDumpName = context
    .filesDir
    .absolutePath + "/error-heap-dump-${Date().time}.hprof"
Debug.dumpHprofData(heapDumpName)
```

### Final solution

```kotlin
private const val LOG_TAG = "OOM-HEAP-RECORDER"
private const val HEAP_DUMP_PREFIX = "error-heap-dump"

fun recordHeapDumpOnOOM(context: Context) {
    val heapDumpName = context
        .filesDir
        .absolutePath + "/$HEAP_DUMP_PREFIX-${Date().time}.hprof"
    val heapDumpCompletedErrorMessage = "heap dump recording completed: $heapDumpName"
    Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
        if (throwable is OutOfMemoryError) {
            Log.e(LOG_TAG, "unhandled exception, recording heap dump")
            Debug.dumpHprofData(heapDumpName)
            Log.e(LOG_TAG, heapDumpCompletedErrorMessage)
        }
        Log.e(LOG_TAG, "Unhandled exception", throwable)
        System.exit(1);
    }
}
```

The `recordHeapDumpOnOOM` initialises the handler and should be called once.
[Application#onCreate](https://developer.android.com/reference/android/app/Application#onCreate()) is a good place for that.

You can see `recordHeapDumpOnOOM` in action in the [Example app](https://github.com/VysotskiVadim/android-oom).

## Exploring heap dump

Heap dumps will be recorded in internal application storage.
I usually download them using [Device Explorer from Android Studio](https://developer.android.com/studio/debug/device-file-explorer).

Convert the downloaded Heap Dump from Android to Java format using `hprof-conv` from the Android SDK platform tools.
Refer to the [documentation](https://android.googlesource.com/platform/frameworks/base/+/f6d03e5/docs/html/tools/help/hprof-conv.jd) or check [Stack Overflow](https://stackoverflow.com/a/60205272) to learn more about it.


You have two tools available to open a Java Heap Dump:
1. [Android Studio Profiler](https://developer.android.com/studio/profile/memory-profiler#import-hprof). You already have it installed.
*Cannot recommend it. As of 2024, Android Studio was crashing until I allowed the IDE to use 5GB of memory. Even with 5GB of memory, it didn't work responsively.*
2. [MAT](https://eclipse.dev/mat/). More advanced compared to AS Profiler.
*I always use it for heap dump analysis. Works responsively with 2GB of memory available. As of 2024, it provides more capabilities for Java heap dump analysis than Android Studio Profiler.*


Explore the heap dump using one of the tools.
The memory state at the moment of `OutOfMemoryError` is the best possible clue to identify the root cause of the error.
