---
layout: post
title: "How to Debug and Fix Play Store Warning About Missing Native Debug Symbols"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/drafts" %}

## Introduction

When I recently uploaded my Android app [One Rep Max Tracker](https://github.com/molundb/one-rep-max-tracker) to the Google Play Console for the first time, I was met with this warning:

![screenshot]({{asset_path}}/google-play-console-no-native-debug-symbols-warning.png){:width="800px"}

In this article I would like to share what I learnt while researching and debugging this warning, in the hopes that it can help others in the same situation.

## Understanding the warning
First things first, let's understand what the warning means. I'm assuming you already know what an App Bundle is. But what are native code and debug symbols?

### Native code
The Native Development Kit (NDK) is a set of tools that allows you to use C and C++ code with Android. The best place to learn about it is the [official documentation](https://developer.android.com/ndk/guides). Most Android applications don't need C/C++ code, but it can be useful for cases where you need to either improve the performance in apps like games or physics simulations, or reuse C/C++ libraries.

> Native code = C/C++ code.

Seeing the warning in Play Store means that your application in some way contains C/C++ code.

### Native debug symbols
Native debug symbols are needed to be able to debug native code. From the official documentation: 

 > Native debug symbols in Android are essential files used during the debugging and profiling of native (C/C++) code within Android applications. These symbols provide valuable information about the native code, such as function names, variable names, and line numbers. This information is crucial for understanding the context and flow of the code, especially when diagnosing crashes or performance issues.

Now that we know what native code and debug symbols are, we're ready to start investigating the issue!

## Checking if the debug symbols are in the bundle
If you're building an android bundle, you should first check if the debug symbols are being packaged in the bundle. 

<!-- https://stackoverflow.com/a/69417189/2225594 -->

Generate a signed bundle with `Build -> Generate Signed App Bundle / APK` and press analyze in the popup that appears.

![screenshot]({{asset_path}}/signed-bundle-generated-popup.png){:width="800px"}

Check if it contains debug symbols. If it doesn't you will see the warning in Google Play Console when you upload the bundle.

![screenshot]({{asset_path}}/analyzing-bundle.png){:width="800px"}

Since you're getting the warning in Google Play Console I bet you won't find the debug symbols packaged in the bundle. In that case it's time for some detective work! 

## Investigating the issue

The first thing to figure out is where the native code is.

According to this fantastic [comment](https://issuetracker.google.com/u/0/issues/234737605#comment19), there are three possibilities for where the native code could be.

1. The app you are building has native source code
2. The app depends on an android library with native source code that is part of the project
3. The app depends on an android library with native source code that is not part of the project

### To figure out where your native code is coming from, perform the following checks
- Check if there are any files in `app/src/**/jniLibs`. If there are, it means the app has native source code.
- Check if there are any files in `app/build/intermediates/merged_native_libs/release/out/lib`. If there are, it means the app depends on an android library with native source code that is part of the project. 
- If there are no files in either of those two folders, it means the app depends on an android library with native source code that is not part of the project.

After you've identified where the native code is, check if the files have debug symbols with the `nm` command on mac/linux (`nm path/to/*.so`).
They need to have debug symbols for the AGP (Android Gradle Plugin) to automatically package them in the bundle. 

### No debug symbols

If the .so files don't have debug symbols - congratulations, you've found the issue! This was the case for me. To get more information I used the command `nm -gD` on the .so file resulted in the following output:

<!--  https://stackoverflow.com/a/34796/2225594-->

```
0000000000000d10 T Java_androidx_datastore_core_NativeSharedCounter_nativeCreateSharedCounter
0000000000000db0 T Java_androidx_datastore_core_NativeSharedCounter_nativeGetCounterValue
0000000000000dc0 T Java_androidx_datastore_core_NativeSharedCounter_nativeIncrementAndGetCounterValue
0000000000000cb0 T Java_androidx_datastore_core_NativeSharedCounter_nativeTruncateFile
0000000000000c60 T _Z16ThrowIoExceptionP7_JNIEnvPKc
0000000000000dd0 T _ZN9datastore12TruncateFileEi
0000000000000e40 T _ZN9datastore15GetCounterValueEPNSt6__ndk16atomicIjEE
0000000000000df0 T _ZN9datastore19CreateSharedCounterEiPPv
0000000000000e50 T _ZN9datastore27IncrementAndGetCounterValueEPNSt6__ndk16atomicIjEE
                 U __cxa_atexit@LIBC
                 U __cxa_finalize@LIBC
                 U __errno@LIBC
                 U __stack_chk_fail@LIBC
                 U ftruncate@LIBC
                 U mmap@LIBC
                 U strerror@LIBC
```

This made me realise my problem had to do with the datastore package. I searched for "native debug symbols" on google issue tracker and found the [issue](https://issuetracker.google.com/u/0/issues/342671895).

In my case the debugging session ended here. I now knew what the issue was, and it was out of my control to solve.

### If the .so files have debug symbols
If the .so files *have* debug symbols then they should be packaged in the bundle. If they're not you need to figure out why. Detective hat on again!

>By default, native code libraries are stripped in release builds of your app. This stripping consists of removing the symbol table and debugging information contained in any native libraries used by your app.

If you use Android Gradle plugin version 4.1 or later and are building an Android App Bundle, you can automatically include the native debug symbols file in it. https://developer.android.com/build/shrink-code#android_gradle_plugin_version_41_or_later

1. Install NDK and CMake https://developer.android.com/studio/projects/install-ndk
2. Add `android.buildTypes.release.ndk.debugSymbolLevel = { SYMBOL_TABLE | FULL }` 

```
buildTypes {  
    release {  
        ndk {  
            debugSymbolLevel = "FULL"  
        }  
    }
 }
```

If you're building an APK or using Android Gradle plugin version 4.0 or earlier (and other build systems) then follow the instructions in the article https://developer.android.com/build/shrink-code#android_gradle_plugin_version_41_or_later.

If it for some reason still does not work to automatically package the debug symbols in the bundle, I'd recommend posting an issue on google issue tracker. In the meantime you can upload the debug symbols manually.

### How to upload the debug symbols manually?
Go to `[YOUR_PROJECT]/app/buld/intermediates/merged_native_libs/release/mergeReleaseNativeLibs/out/lib`. 

Note the 4 folders that exist inside.
- arm64-v8a
- armeabi-v7a
- x86
- x86_64

Select these 4 folders and create a .zip file. The name doesn't matter.
Go to the Google Play Console and upload the zip file.


https://stackoverflow.com/a/68778908/2225594


## Conclusion
Native code is something most Android developers won't need to worry about, but when, for whatever reason, the warning appears in Google Play Console, it's good to know how to debug it, figure out why it's happening and in the best case fix it. I hope this article can be helpful in that endeavour.