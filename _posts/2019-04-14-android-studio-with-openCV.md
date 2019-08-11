---
layout: post
title: "Getting OpenCV to work on Android"
date: 2019-04-14
post_id: 6
---

OpenCV (open computer vision) is a c++ project devoted to computer vision applications. For this weekend project, I was specifically looking into how it could be used in an Android application. To start off, I read through the OpenCV android tutorials located [here](https://docs.opencv.org/2.4/doc/tutorials/introduction/android_binary_package/android_dev_intro.html) and [here](https://docs.opencv.org/2.4/doc/tutorials/introduction/android_binary_package/O4A_SDK.html). Unfortunately, these tutorials referenced an older version of Android Studio... (nowadays, there's a fancy new Android studio that even has a dark mode!)... so I had to try my best to translate outdated instructions towards the new stuff. 

The first things I did were to download the most up-to-date version of Android Studio from google and to download the OpenCV Android SDK. Per the tutorials, I got the most up-to-date Android SDK, OpenCV 3.4.3, located on sourceforge (this turned out to be a big mistake). 

After waiting for Android Studio to finish downloading and installing, I immediately tried importing in one of the sample projects located in the OpenCV Android SDK files called 'face-detection.' It imported a module for the face detection application (openCVSamplefacedetection), as well as a module for the actual OpenCV library SDK (openCVLibrary343). I immediately got a ton of errors, which I will detail below.

The first error:

```
ERROR: Could not find com.android.tools.build:gradle:3.3.2.
```

gradle is some sort of configuration language used for managing project settings and build settings. My initial gradle file looked like this:

``` gradle
// Top-level build file where you can add configuration options common to all sub-projects/modules.
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.3.2'
    }
}

allprojects {
    repositories {
        jcenter()
    }
}
```

I did not know what gradle was prior to this project, so I did some blind googling around.. From what I understand, gradle doesn't know what to do until it looks through links referenced in the `repositories{}` block. Whatever version of gradle I needed wasn't included in the `jcenter()` repository, so I had to add the `google()` repo in both repository blocks and the errors went away.

The next error I got was: 

```
Minimum supported Gradle version is 4.10.1. Current version is 4.8.
```

This error included a "refactor" (i.e. automagically fix) option that I could click to fix it. What it did was go into the `gradle-wrapper.properties` file:

```
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-4.8-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

In that file, it references a distribution URL with a gradle version 4.8. When I clicked the refactor link, it changed that URL to reference `gradle-4.10.1-all.zip` instead.

Next, I got the following error:

```
ERROR: The minSdk version should not be declared in the android manifest file. You can move the version from the manifest to the defaultConfig in the build.gradle file.
Remove minSdkVersion and sync project
Affected Modules: openCVLibrary343, openCVSamplefacedetection
```

I think that build settings used to be declared in the manifest XML file, but then got replaced by this whole gradle thing. This was also another easy fix; it didn't have straight up refactor options, but included links to where the `minSdk` tags were defined. I went ahead and deleted those tags in the `AndroidManifest.xml` files on the openCVLibrary343 and openCVSamplefacedetection modules. `minSdkVersion` properties were already set in the gradle files for both modules.

I also got the warning `Configuration 'compile' is obsolete and has been replaced with 'implementation' and 'api'.` To make this warning go away, I just replaced `compile` commands in the `build.gradle` files to `implementation` commands instead.

At this point, my openCVSamplefacedetection module's `build.gradle` file looked like this:

``` gradle
apply plugin: 'com.android.application'

android {
    compileSdkVersion 11
    buildToolsVersion "28.0.3"

    defaultConfig {
        applicationId "org.opencv.samples.facedetect"
        minSdkVersion 8
        targetSdkVersion 8

        ndk {
            moduleName "detection_based_tracker"
        }
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.txt'
        }
    }
}

dependencies {
    implementation project(':openCVLibrary343')
}

```

Next, I got a bunch of Java compiler errors on the openCVLibrary343 module! Mostly due to references to `android.hardware.camera2`, which the compiler thought did not exist. Turns out that this was because I was trying to compile with SDK version 14, but camera2 wasn't introduced until SDK version 21. So I went ahead and got into openCVLibrary343's build.gradle file and changed the `compileSdkVersion` value to 21 (even if I compile with version 21, it should still be compatible with Android SDKs down to 8).

After all of that, I was left with an error that took me hours to resolve:

```
Error: Your project contains C++ files but it is not using a supported native build system.
Consider using CMake or ndk-build integration. For more information, go to:
 https://d.android.com/r/studio-ui/add-native-code.html
Alternatively, you can use the experimental plugin:
 https://developer.android.com/r/tools/experimental-plugin.html
```

First off, I had to download some C++ SDK tools using Tools->SDK Tools->SDK Tools, and downloading CMake, NDK, and LLDB (not sure if I ended up ever using LLDB). Then, I used the now-available "Link C++ project with Gradle" right-click option on my openCVSamplefacedetection module, and linked it to the `src\main\jni\Android.mk` file that was included in the sample. Did that fix the problem? Yes and no. I got this error:

```
Build command failed.
Error while executing process C:\Android\sdk\ndk-bundle\ndk-build.cmd with arguments {NDK_PROJECT_PATH=null APP_BUILD_SCRIPT=C:\Users\chkodama\StudioProjects\archive\opencv\OpenCV-android-sdk\face-detection\openCVSamplefacedetection\src\main\jni\Android.mk NDK_APPLICATION_MK=C:\Users\chkodama\StudioProjects\archive\opencv\OpenCV-android-sdk\face-detection\openCVSamplefacedetection\src\main\jni\Application.mk APP_ABI=armeabi-v7a NDK_ALL_ABIS=armeabi-v7a NDK_DEBUG=1 APP_PLATFORM=android-16 NDK_OUT=C:/Users/chkodama/StudioProjects/archive/opencv/OpenCV-android-sdk/face-detection/openCVSamplefacedetection/build/intermediates/ndkBuild/debug/obj NDK_LIBS_OUT=C:\Users\chkodama\StudioProjects\archive\opencv\OpenCV-android-sdk\face-detection\openCVSamplefacedetection\build\intermediates\ndkBuild\debug\lib APP_SHORT_COMMANDS=false LOCAL_SHORT_COMMANDS=false -B -n}

process_begin: CreateProcess(NULL, "", ...) failed.
C:/Android/sdk/ndk-bundle/build//../build/core/add-application.mk:178: *** Android NDK: APP_STL gnustl_static is no longer supported. Please switch to either c++_static or c++_shared. See https://developer.android.com/ndk/guides/cpp-support.html for more information.    .  Stop.
```

The key problem is near the bottom: `Android NDK: APP_STL gnustl_static is no longer supported. Please switch to either c++_static or c++_shared.` This setting is controlled by the `src/main/jni/Application.mk` file, which includes the line `APP_STL := gnustl_static`. Simply changing it to `c++_static` or `c++_shared` does not fix the issue (I still tried it, though) because the code is compiler-specific. 

From googling around, it almost seemed like I could use CMake instead of NDK and get it to work somehow... I floundered for several hours trying various stuff, and I could not get anything to work! This was partially because I was going about the problem the wrong way, but mostly because I have almost zero Android/C++ native support expertise and was relying solely on my googling prowess to solve this problem. After struggling for several hours, I realized it was 3AM and went to bed.

One key thing I learned in that struggle was some specifics about the face-detection sample I was trying to run. That project had a section with native c++ code to do face-detection, as well as another section that used ported java code to do face-detection (with the intent of displaying the speed difference between native and ported code). 

If I made another project that only used Java code, like the color-blob-detection one, I could get the project to build and work. I used the tutorial [here](https://android.jlelse.eu/a-beginners-guide-to-setting-up-opencv-android-library-on-android-studio-19794e220f3c) to do that. Note: there is a small flaw in that tutorial. In one of the later sections, it says to copy-paste some code into `MainActivity.java` and into a new `ColorBlobDetector.java`. When you do that, you have to make sure to replace `package com.mobymagic.opencvproject` at the top of both files to the actual package name of your application, and in `MainActivity.java`, you have to go to like 74 `setContentView(R.layout.color_blob_detection_surface_view);` and change `color_blob_detection_surface_view` to `MainActivityView` or whatever your XML file is called in your layouts folder.

So I pretty much narrowed down my problem into figuring out how to get specifically the native code part of the face-detection project working. 

After falling asleep at 3 AM and waking up the next day at 2PM, I re-attacked the problem. I wandered my way into the [OpenCV GitHub page](https://github.com/opencv/opencv), which included more up-to-date android code samples! That was nice and all, but I also needed a more up-to-date version of the SDK.. This wasn't directly hosted on github, but they provided some python scripts to create your own SDKs. So, I downloaded the entire OpenCV repository, went to `platforms\android\build_sdk.py`, and immediately tried just double-clicking it to see if it worked. That didn't work, I had to call it with command-line arguments. I tried to be fancy and run the script from WSL (windows subsytem for linux), but that had problems because it called windows executables by their filename (without the .exe extension), and WSL didn't know how to handle that, and it also used windows paths starting with `C:\`, and WSL didn't like that either.

Instead, I used powershell. In WSL, my (failed) command looked like 

``` shell
python ./build_sdk.py --sdk_path /mnt/c/Android/sdk /mnt/c/opencv_output/
```

In powershell, my command was (I don't have python on my PATH):

``` powershell
$env:LOCALAPPDATA\Programs\Python\Python36\python.exe .\build_sdk.py --sdk_path C:\Android\sdk\ C:\opencv_output\
```

This didn't work, I was getting `cannot concat bytes to str` errors.. Turns out the code was written for Python 2, so I installed that for all users and then ran

``` powershell
C:\Python27\python.exe .\build_sdk.py --sdk_path C:\Android\sdk\ C:\opencv_output\
```

And then it started churning. The first time I ran it, Windows gave me one of those "Windows Firewall has blocked some features of this app" dialog windows, and it messed up the script. I ran the script again and it worked fine. The script was nice enough to also re-generate the sample applications so that they would definitely be compatible with the SDKs that I had. 

After that, I opened up Android studio and imported the entire `samples` folder as a project. When i tried to build the `face-detection` project, it built on the first try, without any errors! Amazing! Same with loading the actual app!

(When I loaded it on to my Android device, the only catch was that it didn't enable camera permissions automatically, so I had to manually enable camera permission from the phone)

It took a while, but I finally got something working. Now, if I ever want to make some sort of image processing application with OpenCV, I will have a baseline to start working with right away.
