---
layout: post
title: "How to Debug and Fix Google Play Console Warning About Missing Native Debug Symbols"
---

{% assign date = page.date | date: "%F" %}
{% assign filename = page.id | split: "/" | last %}
{% assign asset_path = "/assets/images/blog/" | append: date | append: "/" | append: filename %}

## Introduction

When I recently uploaded my Android app [One Rep Max Tracker](https://github.com/molundb/one-rep-max-tracker) to the Google Play Console for the first time, I was met with this warning:

![screenshot]({{asset_path}}/google-play-console-no-native-debug-symbols-warning.png){:width="800px"}

In this article I would like to share what I learnt while researching and debugging this warning, in the hopes that it can help others in the same situation.

## Understanding the warning
First things first, let's understand what the warning means. I'm assuming you already know what an [App Bundle](https://developer.android.com/guide/app-bundle) is. But what are native code and debug symbols?

### Native code
The Native Development Kit (NDK) is a set of tools that allows you to use C and C++ code with Android. The best place to learn about it is the [official documentation](https://developer.android.com/ndk/guides). Most Android applications don't need C/C++ code, but it can be useful for cases where you need to either improve the performance in apps like games or physics simulations, or reuse C/C++ libraries.

Seeing the warning in Play Store means that your application contains C/C++ code.

### Native debug symbols
Native debug symbols are needed to be able to debug native code. From the official documentation: 

 > Native debug symbols in Android are essential files used during the debugging and profiling of native (C/C++) code within Android applications. These symbols provide valuable information about the native code, such as function names, variable names, and line numbers. This information is crucial for understanding the context and flow of the code, especially when diagnosing crashes or performance issues.

Now that we know what native code and debug symbols are, we're ready to start investigating the issue!

## Check if the debug symbols are in the bundle
If you're building an android bundle, you should first check if the debug symbols are being packaged in the bundle. 

<!-- https://stackoverflow.com/a/69417189/2225594 -->

Generate a signed bundle with `Build -> Generate Signed App Bundle / APK` and press analyze in the popup that appears.

![screenshot]({{asset_path}}/signed-bundle-generated-popup.png){:width="800px"}

Check if it contains debug symbols. If it doesn't you will see the warning in Google Play Console when you upload the bundle.

![screenshot]({{asset_path}}/analyzing-bundle.png){:width="800px"}

Since you're getting the warning in Google Play Console I bet you won't find the debug symbols packaged in the bundle. In that case it's time for some detective work to figure out the issue! 

## Investigating the issue

The first thing to figure out is where the native code is. According to this [comment](https://issuetracker.google.com/u/0/issues/234737605#comment19), there are three possible locations for the native code.

- Location 1: The app you are building has native source code.
- Location 2: The app depends on an android library with native source code that is part of the project.
- Location 3: The app depends on an android library with native source code that is not part of the project.

To figure out where your native code is coming from, perform the following checks after generating a signed bundle:
- Check if there are any files in `app/src/**/jniLibs`. If there are, it means the app has native source code (location 1).
- Check if there are any files in `app/build/intermediates/merged_native_libs/release/out/lib`. If there are, it means the app depends on an android library with native source code that is part of the project (location 2). 
- If there are no files in either of those two folders, it means the app depends on an android library with native source code that is not part of the project (location 3).

After you've identified where the native code is, check if the files have debug symbols with the `nm` command on mac/linux (`nm path/to/*.so`).

![screenshot]({{asset_path}}/nm-terminal-command.png){:width="1200px"}

The files need to have debug symbols for the AGP (Android Gradle Plugin) to automatically package the debug symbols in the bundle. 

### If the .so files don't have debug symbols

If the .so files don't have debug symbols - congratulations, you've found the issue!

This was the case for me. To get more information about the file I used the command `nm -gD` and got the following output:

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

In my case the debugging session ended here. I knew what the issue was, and could solve it by using a non-buggy version of the `datastore-preferences` library.

### If the .so files have debug symbols
If the .so files *have* debug symbols then they should be packaged in the bundle. If they're not we need to figure out why.

<!-- >By default, native code libraries are stripped in release builds of your app. This stripping consists of removing the symbol table and debugging information contained in any native libraries used by your app. -->

If you use Android Gradle plugin version 4.1 or later and are building an Android App Bundle, you can [automatically include](https://developer.android.com/build/shrink-code#android_gradle_plugin_version_41_or_later) the native debug symbols file in it. 

1. Install [NDK and CMake](https://developer.android.com/studio/projects/install-ndk)
2. Add `android.buildTypes.release.ndk.debugSymbolLevel = { SYMBOL_TABLE | FULL }` to your app's build.gradle file

```
android {
    buildTypes {  
        release {  
            ndk {  
                debugSymbolLevel = "FULL"  
            }  
        }
    }
}
```

If you're building an APK or using Android Gradle plugin version 4.0 or earlier (and other build systems) then follow the instructions in the [official documentation](https://developer.android.com/build/shrink-code#android_gradle_plugin_version_41_or_later).

If the debug symbols are still not automatically packaged in the bundle after doing that, I'd recommend posting an issue on [google issue tracker](https://issuetracker.google.com/). In the meantime you can upload the debug symbols manually.

### Upload debug symbols manually
Go to `[YOUR_PROJECT]/app/buld/intermediates/merged_native_libs/release/mergeReleaseNativeLibs/out/lib` and note the 4 folders that exist inside.

![screenshot]({{asset_path}}/merged-native-libs-folder.png){:width="800px"}

Select these 4 folders and create a .zip file. The name doesn't matter.

![screenshot]({{asset_path}}/zip-folders.png){:width="800px"}

Finally, go to the Google Play Console and upload the zip file.

![screenshot]({{asset_path}}/upload-native-debug-symbols.png){:width="800px"}

## Conclusion
Native code is something most Android developers won't need to worry about, but when the warning appears in Google Play Console, it's good to know how to understand why it's happening, debug it, and in the best case fix it. I hope this article can help with that.