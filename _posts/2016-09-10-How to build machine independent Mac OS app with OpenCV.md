---
layout: post
title: How to build machine independent Mac OS app with OpenCV
description: "A brief solution to build a Mac OS app, which utilizes OpenCV, that can run in any machine."
modified: 2016-11-18
tags: [MacOS OpenCV]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

This blog briefly records how I built a machine-independent Mac OS app which uses OpenCV.

If you want to build an iOS app with OpenCV, you won’t even think about this question: environment dependency. The main reason is that OpenCV provides a framework, which you can easily download and import to your app project. Not to mention that cocoapod supports OpenCV. But if you think you just need to make the same amount of effort for a Mac app, you are wrong.

I had one project that required to process the camera feed in a mac app. I wrote some POC scripts in python, which is fairly easy. But when it comes to Xcode for mac, trouble begins.


## Step 1: Install OpenCV

It should be fairly simple to install OpenCV as OSX is just an Unix-like system. I forgot why it took me a while to do that (could not find the right instructions). Just do followings:

1. download source code from [here](http://opencv.org/downloads.html) (any version you like), then inside the source code folder:

2. run the following commands in sequence:
```
$ mkdir build
$ cd build
$ cmake ..
$ make
$ make install
```
(*Alert*: this will cause problem in the following steps. I just want to repeat what I did step by step.)


## Step 2: Link OpenCV in Xcode

Now inside `/build/lib`, you can see a bunch of dynamic libraries like `libopencv_calib3d.3.1.0.dylib`.
Inside the Xcode project setting, inside **Linked Frameworks** and **Libraries**, add the libraries you need.

Then, in **Build Setting – Search Paths**, add `/usr/local/include` to **Header Search Path**, and add the `absolute_path/to/opencv/build/lib` to **Library Search Path**.
Now you should be able to `#include "opencv2/opencv.hpp"` in your .h file.

There is no tutorial/example online (as far as I know) that guides you how to access each frame and process them. Thanks to that I have done similar things for iOS, after a couple of hours of debug, I got it working. 
[I have a demo project for that.](https://xiomarg.github.com)


## Step 3: Make the app machine-independent

I forgot this draft for a few days and forgot what I wanted to add previously… anyway, here comes the core part of this article. If you just send this app to another computer and run it, it will fail. If you see the crash log, you will find something like *“unable to locate /Users/clarke/Documents/opencv-2.4.12/build/lib/libopencv_calib3d.2.4.dylib (compatibility version 2.4.0, current version 2.4.12)“.*

I tried many solutions, like copy all .dylib files to project folder, add them to **Build Phases – Copy Bundle Resources**, and removed every absolute link in project setting. Still got the same error.

Though I struggled on this for a whole weekend, when solved, it’s quite simple.
First of all, the opencv libraries were built as dynamic. Even though I copied them in project setting, they still reference to that absolute location in my computer. So we need to build static libraries. Just a little change in the opencv installation step:

```
$ mkdir build
$ cd build
$ cmake -DBUILD_SHARED_LIBS=OFF ..
$ make
$ make install
```
Then instead of a bunch of `.dylib` files. you get `.a` files.
Next, in Xcode project setting, embed these `.a` files instead of `.dylib` files.
Problem solved? BOOM!
<img src="{{ site.url }}/images/build-indenpendent-opencv-mac-app.png" alt="">

If you search *Undefined symbols* for `architecture x86_64: "_gzclose", reference from ....`, the first few answers will talk about the Xcode c++ compiler. In **Build Setting – Apple LLVM 7.1 – language C++**, change c++ compiler library to **compiler default**. Unfortunately, it doesn’t help.

The truth is, these libraries inside the `/opencv-x.x.x/build/lib/` have dependencies on the third party libraries inside `/opencv-x.x.x/build/3rdparty/lib/`.
Then embed these libraries into your project setting as well. Finally you got a machine-independent OpenCV app.


I didn’t elaborate every details here. So if you have any question, just comment.